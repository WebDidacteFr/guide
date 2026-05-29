# CLAUDE.md — Installer Proxmox VE 9.x avec chiffrement de disque (LUKS)

> Donne ce fichier à ton agent IA (Claude Code, Codex, etc.). Il décrit **comment transformer
> une machine en hyperviseur Proxmox VE 9.x chiffré de bout en bout**, en utilisant la méthode
> « Debian d'abord, Proxmox ensuite » (l'installeur officiel de Proxmox ne propose pas LUKS).
>
> ⚠️ **À lire avant de commencer (toi, l'humain) :** cette procédure **efface entièrement le
> disque cible** et modifie le réseau de la machine. Sauvegarde tes données. Garde un **accès
> console physique / KVM / IPMI** : il sera indispensable (déverrouillage du disque à chaque
> démarrage, et filet de sécurité si le réseau est mal configuré).

---

## 0. Rôle de l'agent

Tu es un agent qui aide à installer Proxmox VE 9.x **chiffré**. Le travail se fait en deux phases :

- **Phase A — Installation de Debian + LUKS** : elle se déroule **au clavier sur la console**
  (installeur graphique Debian). **Tu ne peux pas la faire à distance** : tu guides l'humain
  pas à pas, il clique. (Section 3.)
- **Phase B — Conversion en Proxmox + réseau + vérification** : **tu la réalises en autonomie
  par SSH (root)**, une fois la Phase A terminée. C'est ton vrai périmètre. (Sections 4 à 8.)

---

## 1. Règles de sécurité (NON négociables)

1. **Jamais** d'opération destructive sur un disque (`mkfs`, `dd`, `cryptsetup luksFormat`,
   `wipefs`, partitionnement) sans **confirmation explicite** de l'humain ET vérification que
   c'est **le bon disque** (`lsblk`, taille, modèle).
2. **Confirme la machine cible** (hostname + IP) avant toute modification. Ne suppose jamais.
3. **Réseau = danger de te couper toi-même.** Avant de modifier `/etc/network/interfaces` :
   - sauvegarde l'ancien fichier ;
   - vérifie que l'humain a un **accès console** (sinon une erreur = machine injoignable) ;
   - applique avec un **filet anti-lockout** (voir §6).
4. **La passphrase LUKS se tape à chaque démarrage sur la console.** Tu ne peux pas la saisir
   à distance (sauf déverrouillage distant configuré — voir §9). Préviens l'humain : chaque
   reboot nécessite sa présence à la console, ou un accès KVM/IPMI.
5. **N'expose jamais** l'interface web Proxmox (port 8006) directement sur Internet sans
   pare-feu + 2FA + accès VPN. Recommande un accès LAN ou VPN.
6. **Montre les commandes destructives ou réseau avant de les exécuter** si l'humain le demande.

---

## 2. Variables à demander à l'humain (avant de commencer la Phase B)

Demande et note ces valeurs. **Ne les invente pas.**

| Variable | Exemple | Description |
|---|---|---|
| `SSH_TARGET` | `root@192.168.1.50` | Accès SSH root à la machine (après Phase A) |
| `NIC` | `eno1`, `enp3s0`, `ens18` | Nom de l'interface réseau physique (via `ip -br link`) |
| `STATIC_IP` | `192.168.1.50/24` | IP statique souhaitée pour l'hôte Proxmox (CIDR) |
| `GATEWAY` | `192.168.1.1` | Passerelle du réseau |
| `DNS` | `192.168.1.1` ou `1.1.1.1` | Serveur DNS |
| `HOSTNAME` | `pve` | Nom d'hôte court |
| `FQDN` | `pve.maison.lan` | Nom complet (hostname + domaine) |
| `CONSOLE_OK` | oui/non | L'humain a-t-il un accès console/KVM/IPMI ? (obligatoire) |

Si `CONSOLE_OK` = non → **arrête-toi** et explique qu'il faut un accès console avant de continuer.

---

## 3. Phase A — Installer Debian 13 + LUKS (guider l'humain)

L'humain démarre la machine sur une clé **Debian 13 « Trixie » netinst** (vérifier le SHA256 sur
cdimage.debian.org) et suit l'installeur graphique. Donne-lui ces consignes :

1. **Graphical install** (valider vite : le menu a un compte à rebours).
2. Langue/pays/clavier au choix. ⚠️ **Le clavier choisi sert aussi à taper la passphrase LUKS
   au démarrage** : choisir le clavier réellement utilisé sur la machine.
3. Réseau : laisser en DHCP pour l'install (on figera une IP statique en Phase B).
4. Hostname = `HOSTNAME`. Domaine = la partie domaine de `FQDN` (ou vide).
5. Mot de passe **root** fort (obligatoire pour Proxmox). Créer aussi un utilisateur.
6. **Partitionnement → « Assisté - utiliser un disque entier et configurer le LVM chiffré »**
   (= LUKS). Choisir le bon disque. Schéma : **« Tout dans une seule partition »**.
7. Écrire les changements. **Choisir une passphrase de chiffrement forte** (elle sera demandée
   à chaque boot) — la noter en lieu sûr. (L'effacement aléatoire de la partition peut être
   sauté sur un disque neuf.)
8. Sélection des logiciels (tasksel) : **décocher tout environnement de bureau**, **cocher
   « serveur SSH »** + « utilitaires usuels du système ». Rien d'autre.
9. Installer GRUB sur le disque système.
10. Redémarrer, **saisir la passphrase LUKS** au prompt, vérifier que Debian démarre et que SSH
    répond. Donner à l'agent l'accès `SSH_TARGET` (clé SSH recommandée).

> Quand l'humain confirme « Debian chiffré démarre, SSH OK », passe à la Phase B.

---

## 4. Phase B — Pré-vérifications (agent, par SSH)

```bash
# Confirmer l'OS, l'accès root, l'interface et l'accès Internet
. /etc/os-release; echo "OS=$PRETTY_NAME"      # doit être Debian 13 (trixie)
id                                             # doit être root
ip -br link                                    # repérer/confirmer $NIC
ip -4 -br addr                                 # IP actuelle (DHCP)
ping -c1 deb.debian.org                        # Internet OK ?
lsblk                                          # vue des disques (info)
```
Si l'OS n'est pas Debian 13 ou pas root → **stop**, explique.

## 5. Phase B — Conversion en Proxmox VE

```bash
# 5.1 — Le hostname DOIT résoudre vers une IP réelle (pas 127.0.1.1), sinon Proxmox refuse.
cp /etc/hosts /etc/hosts.bak
# Remplace la ligne 127.0.1.1 par l'IP statique cible + le FQDN :
#   <IP sans /CIDR>   <FQDN> <HOSTNAME>
# Exemple : 192.168.1.50   pve.maison.lan pve
hostname --ip-address    # doit renvoyer l'IP réelle (pas 127.x)

# 5.2 — Dépôt Proxmox VE (Trixie) + clé GPG, format deb822 officiel
cat > /etc/apt/sources.list.d/pve-install-repo.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
wget https://enterprise.proxmox.com/debian/proxmox-archive-keyring-trixie.gpg \
  -O /usr/share/keyrings/proxmox-archive-keyring.gpg
apt update     # doit récupérer le dépôt 'pve' sans erreur GPG

# 5.3 — Mise à niveau + noyau Proxmox
DEBIAN_FRONTEND=noninteractive apt -y -o Dpkg::Options::=--force-confold full-upgrade
apt -y install proxmox-default-kernel
```

**Reboot #1 sur le noyau Proxmox** → l'humain saisit la passphrase LUKS à la console.
Reconnecte-toi en SSH, vérifie : `uname -r` doit contenir `-pve`.

```bash
# 5.4 — Paquets Proxmox (postfix préconfiguré pour éviter l'interactif)
echo "postfix postfix/main_mailer_type select Local only" | debconf-set-selections
echo "postfix postfix/mailname string $FQDN" | debconf-set-selections
DEBIAN_FRONTEND=noninteractive apt -y -o Dpkg::Options::=--force-confold \
  install proxmox-ve postfix open-iscsi chrony

# 5.5 — Retirer le noyau Debian (ÉTAPE OFFICIELLE Proxmox) + os-prober
# ⚠️ `linux-image-amd64` n'est que le MÉTA-paquet. Il faut AUSSI retirer l'image versionnée
# déjà installée (série 6.12.x sur Trixie), sinon le noyau Debian reste sur le disque ET dans
# le menu GRUB — on veut ne booter QUE sur le noyau Proxmox (-pve).
dpkg -l 'linux-image-*' | grep '^ii'        # repérer les noyaux installés (garder l'image *-pve)
apt -y remove os-prober linux-image-amd64 'linux-image-6.12*'   # adapter 6.12* à ta série Debian
update-grub                                  # GRUB ne doit plus lister QUE le noyau -pve
pveversion       # doit afficher pve-manager/9.x
```

## 6. Phase B — Réseau : IP statique sur bridge `vmbr0` (ÉTAPE CRITIQUE)

> **Pourquoi c'est obligatoire :** (1) Proxmox a besoin d'un **bridge `vmbr0`** pour donner du
> réseau aux futures VM ; (2) `proxmox-ve` installe `ifupdown2` qui peut ajouter une
> configuration **IPv6 auto** ⇒ `dhclient -6` peut **bloquer le démarrage** (service réseau qui
> ne se termine jamais). Une **IP statique sans DHCP** règle les deux problèmes.

```bash
cp /etc/network/interfaces /etc/network/interfaces.bak
```
Écris `/etc/network/interfaces` ainsi (adapte `NIC`, `STATIC_IP`, `GATEWAY`) :
```
auto lo
iface lo inet loopback

iface <NIC> inet manual

auto vmbr0
iface vmbr0 inet static
        address <STATIC_IP>          # ex: 192.168.1.50/24
        gateway <GATEWAY>            # ex: 192.168.1.1
        bridge-ports <NIC>
        bridge-stp off
        bridge-fd 0
```
Mets aussi le DNS : `echo "nameserver <DNS>" > /etc/resolv.conf`.

**Filet anti-lockout** (si tu appliques à chaud sans reboot immédiat) :
```bash
# révoque la nouvelle conf dans 3 min SAUF si l'humain confirme que ça marche
( sleep 180 && cp /etc/network/interfaces.bak /etc/network/interfaces && ifreload -a ) &
REVERT_PID=$!
ifreload -a            # applique la nouvelle conf (ifupdown2)
# vérifie l'accès, puis si OK : kill $REVERT_PID  (annule la révocation)
```
> Si tu n'es pas sûr, **ne fais pas `ifreload` à chaud** : applique la conf et demande à
> l'humain de rebooter (il a la console). En cas de souci, il restaure `interfaces.bak` au clavier.

## 7. Phase B — Reboot final + vérification

**Reboot #2** → l'humain saisit la passphrase LUKS. Puis vérifie :
```bash
systemctl is-active pveproxy pvedaemon pve-cluster sshd   # tous 'active'
ip -4 -br addr show vmbr0                                  # doit porter STATIC_IP
ss -tln | grep :8006                                       # interface web en écoute
```
Donne à l'humain l'URL : **`https://<IP de l'hôte>:8006`** — connexion : utilisateur `root`,
**Realm = « Linux PAM standard authentication »**, mot de passe root défini en Phase A.

## 8. Diagnostic (si un démarrage bloque)

```bash
systemctl --failed
journalctl -b -p err --no-pager | tail -50
journalctl -b -u networking --no-pager           # souvent la cause (réseau)
```
**Hang typique après conversion** : `networking.service` reste « activating » à cause de
`dhclient -6`/IPv6 auto → applique la config statique `vmbr0` de §6 (c'est LE correctif).

## 9. Pour aller plus loin (optionnel)

- **Déverrouillage LUKS à distance** (éviter d'être à la console à chaque boot) :
  installer `dropbear-initramfs`, y déposer une clé SSH, configurer une IP dans l'initramfs,
  puis `ssh root@<ip> -p <port>` au boot pour saisir la passphrase. (Procédure avancée.)
- **Sécurité** : pare-feu Proxmox, 2FA sur le compte root, accès via VPN (WireGuard/Tailscale)
  plutôt que d'exposer 8006.
- **Sauvegarde** : noter la passphrase LUKS hors-ligne ; sans elle, **les données sont
  irrécupérables**.

---

## Rollback (annuler)

- `/etc/hosts.bak`, `/etc/network/interfaces.bak` → restaurer en cas d'erreur.
- La conversion Proxmox n'est pas « désinstallable » proprement : si tout part de travers,
  réinstaller Debian+LUKS (Phase A) est plus sûr que de bricoler.

## Checklist finale
- [ ] Debian 13 + LUKS installé, démarre et déverrouille
- [ ] `hostname --ip-address` = IP réelle
- [ ] Dépôt PVE ajouté, `apt update` sans erreur
- [ ] Noyau `-pve` actif après reboot #1
- [ ] `proxmox-ve` installé (`pveversion` = 9.x), noyau Debian retiré
- [ ] **IP statique sur `vmbr0`** (pas de DHCP)
- [ ] Reboot #2 OK, services `active`, web UI `:8006` accessible
