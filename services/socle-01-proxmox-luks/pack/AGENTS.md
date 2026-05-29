# AGENTS.md — Installer Proxmox VE 9.x avec chiffrement de disque (LUKS)

> Fichier d'instructions pour un agent IA de codage (Codex, Claude Code, Cursor, Aider, etc.).
> Objectif : **transformer une machine en hyperviseur Proxmox VE 9.x chiffré de bout en bout**
> via la méthode « Debian d'abord, Proxmox ensuite » (l'installeur Proxmox ne propose pas LUKS).
>
> ⚠️ **Humain, à lire avant de lancer l'agent :** cette procédure **efface tout le disque cible**
> et reconfigure le réseau. Sauvegarde tes données et garde un **accès console / KVM / IPMI**
> (indispensable : déverrouillage du disque à chaque démarrage + secours si le réseau casse).

---

## Périmètre de l'agent

Le travail se fait en **deux phases** :

- **Phase A — Debian + LUKS** : au **clavier sur la console** (installeur graphique Debian).
  **Non automatisable à distance** → l'agent *guide* l'humain pas à pas (section 3).
- **Phase B — Conversion Proxmox + réseau + vérification** : l'agent l'exécute **seul, par SSH
  (root)**, une fois la Phase A faite. C'est son vrai périmètre (sections 4 à 8).

## Règles de sécurité (impératives)

1. **Aucune** opération disque destructive (`mkfs`, `dd`, `cryptsetup luksFormat`, `wipefs`,
   partitionnement) sans **confirmation explicite** + vérification du **bon disque** (`lsblk`).
2. **Confirmer la machine cible** (hostname + IP) avant toute modification — ne rien supposer.
3. **Réseau = risque d'auto-déconnexion.** Avant de toucher `/etc/network/interfaces` :
   sauvegarder l'ancien fichier, vérifier que l'humain a un **accès console**, appliquer avec
   un **filet anti-lockout** (§6).
4. **La passphrase LUKS se saisit à chaque démarrage sur la console** : l'agent ne peut pas la
   taper à distance (sauf déverrouillage distant, §9). Prévenir l'humain.
5. **Ne pas exposer** l'interface web (8006) sur Internet sans pare-feu + 2FA + VPN.
6. **Afficher les commandes destructives/réseau avant exécution** si l'humain le demande.

## Variables à collecter (avant la Phase B) — NE PAS LES INVENTER

| Variable | Exemple | Description |
|---|---|---|
| `SSH_TARGET` | `root@192.168.1.50` | Accès SSH root (après Phase A) |
| `NIC` | `eno1` / `enp3s0` / `ens18` | Interface réseau physique (`ip -br link`) |
| `STATIC_IP` | `192.168.1.50/24` | IP statique de l'hôte (CIDR) |
| `GATEWAY` | `192.168.1.1` | Passerelle |
| `DNS` | `192.168.1.1` / `1.1.1.1` | Serveur DNS |
| `HOSTNAME` / `FQDN` | `pve` / `pve.maison.lan` | Nom court / complet |
| `CONSOLE_OK` | oui/non | Accès console/KVM/IPMI dispo ? (obligatoire) |

Si `CONSOLE_OK` = non → **s'arrêter** et expliquer qu'un accès console est requis.

---

## 3. Phase A — Debian 13 + LUKS (l'agent guide l'humain)

Démarrer la machine sur une clé **Debian 13 « Trixie » netinst** (vérifier le SHA256 sur
cdimage.debian.org). Consignes à donner :

1. **Graphical install** (valider vite — compte à rebours du menu).
2. Langue/pays/clavier. ⚠️ le clavier sert aussi à **taper la passphrase LUKS au boot** :
   choisir le clavier réellement utilisé sur la machine.
3. Réseau : DHCP pour l'install (IP statique figée en Phase B).
4. Hostname = `HOSTNAME` ; domaine = partie domaine de `FQDN` (ou vide).
5. Mot de passe **root** fort (obligatoire pour Proxmox) + créer un utilisateur.
6. **Partitionnement → « Assisté – disque entier + LVM chiffré »** (= LUKS), bon disque,
   schéma **« tout dans une seule partition »**.
7. Écrire les changements ; **passphrase de chiffrement forte** (demandée à chaque boot, à noter
   en lieu sûr). L'effacement aléatoire peut être sauté sur un disque neuf.
8. tasksel : **décocher tout bureau**, **cocher « serveur SSH »** + « utilitaires usuels ».
9. Installer GRUB sur le disque système.
10. Redémarrer, **saisir la passphrase LUKS**, vérifier que Debian démarre + SSH OK, puis fournir
    `SSH_TARGET` à l'agent (clé SSH recommandée).

> Quand l'humain confirme « Debian chiffré démarre, SSH OK » → Phase B.

## 4. Phase B — Pré-vérifications (par SSH)

```bash
. /etc/os-release; echo "OS=$PRETTY_NAME"   # Debian 13 (trixie) attendu
id                                           # root attendu
ip -br link ; ip -4 -br addr                 # confirmer $NIC + IP DHCP actuelle
ping -c1 deb.debian.org                       # Internet ?
lsblk                                         # disques (info)
```
OS ≠ Debian 13 ou pas root → **stop**.

## 5. Phase B — Conversion en Proxmox VE

```bash
# 5.1 Le hostname doit résoudre vers l'IP réelle (pas 127.0.1.1)
cp /etc/hosts /etc/hosts.bak
# Mettre :  <IP sans /CIDR>   <FQDN> <HOSTNAME>     (ex: 192.168.1.50 pve.maison.lan pve)
hostname --ip-address     # doit renvoyer l'IP réelle

# 5.2 Dépôt Proxmox VE (Trixie) + clé GPG, format deb822 officiel
cat > /etc/apt/sources.list.d/pve-install-repo.sources <<'EOF'
Types: deb
URIs: http://download.proxmox.com/debian/pve
Suites: trixie
Components: pve-no-subscription
Signed-By: /usr/share/keyrings/proxmox-archive-keyring.gpg
EOF
wget https://enterprise.proxmox.com/debian/proxmox-archive-keyring-trixie.gpg \
  -O /usr/share/keyrings/proxmox-archive-keyring.gpg
apt update                # dépôt 'pve' sans erreur GPG

# 5.3 Mise à niveau + noyau Proxmox
DEBIAN_FRONTEND=noninteractive apt -y -o Dpkg::Options::=--force-confold full-upgrade
apt -y install proxmox-default-kernel
```
**Reboot #1** (humain saisit la passphrase LUKS) → reconnexion SSH → `uname -r` contient `-pve`.

```bash
# 5.4 Paquets Proxmox (postfix préconfiguré)
echo "postfix postfix/main_mailer_type select Local only" | debconf-set-selections
echo "postfix postfix/mailname string $FQDN" | debconf-set-selections
DEBIAN_FRONTEND=noninteractive apt -y -o Dpkg::Options::=--force-confold \
  install proxmox-ve postfix open-iscsi chrony

# 5.5 Retirer le noyau Debian (ÉTAPE OFFICIELLE) + os-prober
# ⚠️ `linux-image-amd64` = méta seul. Retirer AUSSI l'image versionnée (6.12* sur Trixie),
# sinon le noyau Debian reste installé ET dans GRUB. But : ne booter QUE sur -pve.
dpkg -l 'linux-image-*' | grep '^ii'                 # repérer (garder l'image *-pve)
apt -y remove os-prober linux-image-amd64 'linux-image-6.12*'   # adapter 6.12* à ta série
update-grub ; pveversion        # GRUB = que -pve ; pveversion = pve-manager/9.x
```

## 6. Phase B — Réseau statique sur bridge `vmbr0` (ÉTAPE CRITIQUE)

> **Pourquoi :** Proxmox a besoin d'un **bridge `vmbr0`** (réseau des futures VM) ; et `ifupdown2`
> (installé par `proxmox-ve`) peut activer un **IPv6 auto** ⇒ `dhclient -6` **bloque le démarrage**
> (service réseau qui ne finit jamais). Une **IP statique sans DHCP** corrige les deux.

```bash
cp /etc/network/interfaces /etc/network/interfaces.bak
echo "nameserver <DNS>" > /etc/resolv.conf
```
Contenu de `/etc/network/interfaces` (adapter `NIC`, `STATIC_IP`, `GATEWAY`) :
```
auto lo
iface lo inet loopback

iface <NIC> inet manual

auto vmbr0
iface vmbr0 inet static
        address <STATIC_IP>
        gateway <GATEWAY>
        bridge-ports <NIC>
        bridge-stp off
        bridge-fd 0
```
**Filet anti-lockout** si application à chaud :
```bash
( sleep 180 && cp /etc/network/interfaces.bak /etc/network/interfaces && ifreload -a ) &
REVERT_PID=$!
ifreload -a              # applique (ifupdown2)
# si l'accès est confirmé OK :  kill $REVERT_PID
```
> En cas de doute : ne pas `ifreload` à chaud — appliquer la conf et laisser l'humain rebooter
> (il a la console). En cas de souci, il restaure `interfaces.bak` au clavier.

## 7. Phase B — Reboot final + vérification

**Reboot #2** (passphrase LUKS), puis :
```bash
systemctl is-active pveproxy pvedaemon pve-cluster sshd   # tous 'active'
ip -4 -br addr show vmbr0                                  # porte STATIC_IP
ss -tln | grep :8006                                       # web UI en écoute
```
URL pour l'humain : **`https://<IP de l'hôte>:8006`** — user `root`, Realm
**« Linux PAM standard authentication »**, mot de passe root de la Phase A.

## 8. Diagnostic (démarrage bloqué)

```bash
systemctl --failed
journalctl -b -p err --no-pager | tail -50
journalctl -b -u networking --no-pager     # cause fréquente
```
Hang typique après conversion : `networking.service` reste « activating » (IPv6/`dhclient -6`)
→ appliquer la config statique `vmbr0` du §6 (c'est LE correctif).

## 9. Optionnel

- **Déverrouillage LUKS distant** : `dropbear-initramfs` + clé SSH + IP dans l'initramfs →
  `ssh` au boot pour saisir la passphrase (évite la console à chaque reboot).
- **Sécurité** : pare-feu Proxmox, 2FA root, accès via VPN plutôt que d'exposer 8006.
- **Sauvegarde** : noter la passphrase LUKS hors-ligne — sans elle, données **irrécupérables**.

## Rollback
- Restaurer `/etc/hosts.bak`, `/etc/network/interfaces.bak` en cas d'erreur.
- La conversion Proxmox ne se « désinstalle » pas proprement : en cas d'échec global,
  réinstaller Debian+LUKS (Phase A) est plus sûr.

## Checklist finale
- [ ] Debian 13 + LUKS démarre et déverrouille
- [ ] `hostname --ip-address` = IP réelle
- [ ] Dépôt PVE ajouté, `apt update` OK
- [ ] Noyau `-pve` actif (reboot #1)
- [ ] `proxmox-ve` installé (`pveversion` 9.x), noyau Debian retiré
- [ ] **IP statique sur `vmbr0`** (pas de DHCP)
- [ ] Reboot #2 OK, services `active`, web UI `:8006` accessible
