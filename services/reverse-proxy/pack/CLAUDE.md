# CLAUDE.md — Installer un reverse proxy HTTPS (Nginx Proxy Manager) sur un Docker existant

> Donne ce fichier à ton agent IA (Claude Code, Codex, etc.). Il décrit comment **installer Nginx
> Proxy Manager (NPM) avec Docker Compose** sur une machine qui a **déjà Docker + Docker Compose**,
> proxifier des services en **HTTPS**, et obtenir des certificats (Let's Encrypt **ou** CA interne).
>
> NPM = reverse proxy avec **interface web** : un point d'entrée unique (80/443) qui route par nom
> d'hôte vers plusieurs services internes, gère les certificats, la redirection HTTP→HTTPS et les
> en-têtes de sécurité. Stack = **1 seul conteneur** (SQLite intégrée depuis v2.10+).

---

## 0. Rôle de l'agent
Tu installes NPM **en autonomie** sur un hôte Docker existant. Tout se fait en CLI sauf la config des
hôtes/certificats qui passe par l'UI web (:81). Déroule les sections 3 → 8 seul, en t'arrêtant aux
points **[DEMANDER]**.

## 1. Règles de sécurité (NON négociables)
1. **Ne jamais exposer l'UI d'admin (port 81)** : binder sur `127.0.0.1` (défaut de ce pack) ou sur
   une IP LAN précise. **Jamais** `81:81` (= 0.0.0.0). Idéal : accès via VPN.
2. **Ne jamais exposer un backend en clair** : les services proxifiés restent en `127.0.0.1` ou sur un
   **réseau Docker interne** partagé avec NPM. Seuls 80/443 (NPM) sont publics.
3. **Activer Force SSL + HTTP/2 + HSTS** sur chaque hôte. HTTPS partout.
4. **Sauvegarder AVANT** toute mise à jour (cf. `UPDATE.md`).
5. **Secrets** : token DNS et clé de CA interne ne se committent jamais ; clé de CA stockée
   hors-ligne, chiffrée.

## 2. Variables / décisions à demander  **[DEMANDER]**
| Choix | Défaut conseillé | Description |
|---|---|---|
| `INSTALL_DIR` | `/srv/docker/npm` | Dossier projet (compose + data + letsencrypt) |
| `EXPOSURE` | LAN + VPN | L'admin :81 reste en loopback/LAN ; jamais Internet |
| **Stratégie TLS** | voir §6 | **A)** domaine réel + Let's Encrypt (DNS-01) — *prod* · **B)** CA interne `*.lan` — *local* |
| Hôtes voulus | `photos.<domaine>`, … | 1 nom d'hôte par service à publier |

Prérequis : `docker --version`, `docker compose version`, **ports 80/443/81 libres**
(`ss -tulpn | grep -E ':(80|81|443)\b'`). Si **DNS-01**, un **domaine** + un **token API DNS**.

### 2bis. Si Docker n'est PAS installé  **[DEMANDER confirmation]**
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"        # re-login requis ensuite
sudo systemctl enable --now docker
docker run --rm hello-world
```
> VM Proxmox neuve : créer la VM (image cloud Debian + cloud-init) puis ce bloc.

## 3. Déployer NPM
```bash
mkdir -p "$INSTALL_DIR" && cd "$INSTALL_DIR"
# Récupérer docker-compose.yml de ce pack (1 conteneur, admin bindée 127.0.0.1, image épinglée)
docker compose config        # valide la syntaxe
docker compose pull
docker compose up -d
docker compose ps            # npm_app "Up" : 0.0.0.0:80, 0.0.0.0:443, 127.0.0.1:81
docker compose logs -f app   # jusqu'à "Backend PID ... listening on port 3000"
```

## 4. Compte admin (1er accès UI)
`http://127.0.0.1:81` (ou via tunnel SSH `ssh -L 81:127.0.0.1:81 user@hote`).
- NPM v2.14 : écran **« Welcome! »** → **créer** le compte admin. **[DEMANDER]** email + mot de passe fort.
- (Vieilles versions : login `admin@example.com` / `changeme` à **changer immédiatement**.)

## 5. Backends à proxifier
Pour que NPM joigne un service par son **nom de conteneur**, mettre NPM **et** le backend sur un
**réseau Docker partagé** :
```bash
docker network create proxy 2>/dev/null || true
# attacher NPM et chaque backend à "proxy" (compose: networks: default: name: proxy)
```
Sinon, proxifier une **IP:port** LAN directement (ex Immich `192.168.1.180:2283`). Vérifier l'atteinte :
```bash
docker exec npm_app curl -sf http://<backend>:<port>/    # ou l'IP LAN
```

## 6. Certificats — choisir le scénario  **[DEMANDER]**

### A) Domaine réel → Let's Encrypt, challenge DNS-01 (recommandé en prod)
Certificats **valides** sans ouvrir de port. Il faut un **token DNS** chez **le fournisseur qui héberge
réellement la zone**.
0. **[CRITIQUE] Vérifier les NS réellement délégués** (≠ l'interface du registrar) :
   ```bash
   dig NS <domaine> +short          # le plugin DNS-01 doit correspondre à CE fournisseur
   ```
   Piège vécu : un domaine `.ovh` peut être délégué à **Cloudflare** → un token OVH donnera `NXDOMAIN`.
1. Créer un token **scopé à la zone** chez le bon fournisseur :
   - **Cloudflare** : dash.cloudflare.com/profile/api-tokens → template « Edit zone DNS » → zone =
     `<domaine>` → **sans restriction d'IP** (sinon la VM/conteneur est bloqué, erreur `9109`).
   - **OVH** : api.ovh.com/createToken → droits `GET/POST/PUT/DELETE` sur `/domain/zone/*`,
     **Validity = Unlimited**, endpoint `ovh-eu`.
2. **Tester le token depuis la machine cible** (pas seulement `verify` — il est exempté du filtrage IP) :
   faire une **vraie écriture TXT** via l'API (cf. TROUBLESHOOTING.md).
3. UI NPM → **Certificates → Add Certificate → Let's Encrypt via DNS** → choisir le provider →
   credentials (Cloudflare : `dns_cloudflare_api_token=<token>` · OVH : `dns_ovh_endpoint=ovh-eu` + keys) →
   domaines `*.<domaine>` + `<domaine>` (wildcard OK) → Propagation 30–120 s.
4. NPM crée un TXT `_acme-challenge` temporaire, LE valide, le supprime. Cert émis (**aucun port ouvert**).

### B) Local → CA interne `*.maison.lan` (auto-suffisant, sans domaine)
```bash
mkdir -p "$INSTALL_DIR/ca" && cd "$INSTALL_DIR/ca"
# CA racine (10 ans)
openssl genrsa -out maison-ca.key 4096
openssl req -x509 -new -nodes -key maison-ca.key -sha256 -days 3650 \
  -subj "/C=FR/O=Labo Maison/CN=Maison LAN Root CA" -out maison-ca.crt
# Cert wildcard *.maison.lan AVEC SAN (obligatoire)
openssl genrsa -out wildcard.key 2048
cat > san.cnf <<'EOF'
[req]
distinguished_name=dn
req_extensions=v3_req
prompt=no
[dn]
C=FR
O=Labo Maison
CN=*.maison.lan
[v3_req]
basicConstraints=CA:FALSE
keyUsage=digitalSignature,keyEncipherment
extendedKeyUsage=serverAuth
subjectAltName=@alt
[alt]
DNS.1=*.maison.lan
DNS.2=maison.lan
EOF
openssl req -new -key wildcard.key -out wildcard.csr -config san.cnf
openssl x509 -req -in wildcard.csr -CA maison-ca.crt -CAkey maison-ca.key -CAcreateserial \
  -out wildcard.crt -days 825 -sha256 -extfile san.cnf -extensions v3_req
openssl verify -CAfile maison-ca.crt wildcard.crt    # -> OK
```
- UI NPM → **Certificates → Add → Custom Certificate** : Key=`wildcard.key`, Certificate=`wildcard.crt`,
  Intermediate=`maison-ca.crt`.
- Résolution des noms : DNS local (Pi-hole/AdGuard/routeur) **ou** `/etc/hosts` :
  `echo '127.0.0.1 photos.maison.lan hello.maison.lan' | sudo tee -a /etc/hosts`
- **Importer la CA** (`maison-ca.crt`) sur chaque appareil client (sinon avertissement, **normal**) :
  voir `TROUBLESHOOTING.md` (Chrome NSS, Firefox, Windows, macOS, Android).

## 7. Créer un hôte proxy (par service)
UI NPM → **Hosts → Proxy Hosts → Add Proxy Host** :
- **Details** : Domain `photos.<domaine>`, Scheme `http`, Forward `<backend>:<port>`,
  cocher **Block Common Exploits** (+ **Websockets Support** si l'app en a besoin, ex Immich).
- **SSL** : choisir le certificat, cocher **Force SSL**, **HTTP/2**, **HSTS Enabled** → Save.
- L'hôte doit passer **Online**.

## 8. Vérification TLS (à faire, ne pas supposer)
```bash
# Redirection HTTP -> HTTPS
curl -sI http://photos.<domaine> | grep -i location          # 301 -> https://...
# Chaîne valide (CA interne : ajouter --cacert maison-ca.crt ; LE : rien)
curl -s [--cacert ca/maison-ca.crt] -o /dev/null \
  -w 'code=%{http_code} tls=%{ssl_verify_result} proto=%{http_version}\n' https://photos.<domaine>/
# tls=0 = chaîne OK. proto=2 = HTTP/2.
openssl s_client -connect <hote>:443 -servername photos.<domaine> [-CAfile ca/maison-ca.crt] </dev/null 2>/dev/null \
  | grep -E 'subject=|issuer=|Verify return'
# En-tête HSTS présent :
curl -sI [--cacert ...] https://photos.<domaine> | grep -i strict-transport
```

## 9. Sauvegarde / restauration / mise à jour
- **Backup** : `data/` + `letsencrypt/` (+ compose). Voir `BACKUP.md`. (CA interne : sauver `ca/` à part.)
- **Restauration** : `tar xzf` → `up -d` → hôtes Online. Voir `RESTORE.md` (testée).
- **Mise à jour** : bumper le tag, `pull` + `up -d`, migrations auto. Voir `UPDATE.md`. **Sauver d'abord.**

## Rollback / désinstallation
```bash
docker compose down                 # arrête, conserve data/ + letsencrypt/
# Suppression COMPLÈTE (DESTRUCTIF — demander confirmation) :
# docker compose down && sudo rm -rf "$INSTALL_DIR"   # + retirer la ligne /etc/hosts + la CA des clients
```

## Checklist finale
- [ ] Docker + Compose présents ; ports 80/443/81 libres
- [ ] Image épinglée (`:2.14.0`), admin bindée `127.0.0.1`
- [ ] Compte admin créé (mot de passe fort) ; UI :81 non exposée
- [ ] Certificat OK (LE DNS-01 *valide*, ou CA interne + CA importée sur les clients)
- [ ] Hôte(s) proxy **Online** ; Force SSL + HTTP/2 + HSTS ; Block Common Exploits
- [ ] Vérif TLS réelle : 301 HTTP→HTTPS, `tls=0`, HSTS présent, backend répond à travers le proxy
- [ ] **Backup fait + restauration testée**
