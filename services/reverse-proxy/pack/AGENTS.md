# AGENTS.md — Installer un reverse proxy HTTPS (Nginx Proxy Manager) sur un Docker existant

> Fichier d'instructions pour un agent IA de codage (Codex, Claude Code, Cursor, Aider, etc.).
> Objectif : **installer Nginx Proxy Manager (NPM) via Docker Compose** sur une machine disposant
> **déjà de Docker + Docker Compose**, proxifier des services en **HTTPS**, et obtenir des certificats
> (Let's Encrypt DNS-01 **ou** CA interne).
>
> NPM = reverse proxy à **interface web** : point d'entrée unique 80/443, routage par nom d'hôte,
> certificats, redirection HTTP→HTTPS, en-têtes de sécurité. **1 conteneur** (SQLite intégrée v2.10+).

---

## Périmètre de l'agent
Installation automatisable sur hôte Docker existant. CLI pour le déploiement ; UI web (:81) pour les
hôtes/certificats. Dérouler 3 → 8 en autonomie, s'arrêter aux points **[DEMANDER]**.

## 1. Contraintes de sécurité (impératives)
1. UI d'admin (81) **jamais exposée** : `127.0.0.1:81:81` (défaut) ou IP LAN précise ; jamais `0.0.0.0`.
2. Backends **jamais en clair** : `127.0.0.1` ou réseau Docker interne partagé avec NPM. Public = 80/443 seulement.
3. **Force SSL + HTTP/2 + HSTS** sur chaque hôte.
4. **Sauvegarder avant** toute montée de version (`UPDATE.md`).
5. Secrets (token DNS, clé de CA) : jamais commités ; clé de CA hors-ligne et chiffrée.

## 2. Paramètres / décisions à confirmer  **[DEMANDER]**
| Choix | Défaut | Rôle |
|---|---|---|
| `INSTALL_DIR` | `/srv/docker/npm` | Dossier projet |
| `EXPOSURE` | LAN + VPN | Admin :81 jamais sur Internet |
| Stratégie TLS | §6 | A) domaine + Let's Encrypt DNS-01 (prod) · B) CA interne `*.lan` (local) |
| Hôtes | `photos.<domaine>`, … | 1 par service publié |

Pré-vérifs : `docker --version`, `docker compose version`, ports **80/443/81 libres**. DNS-01 → domaine + token DNS.

### 2bis. Docker absent  **[DEMANDER confirmation]**
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"        # re-login
sudo systemctl enable --now docker
docker run --rm hello-world
```

## 3. Déployer NPM
```bash
mkdir -p "$INSTALL_DIR" && cd "$INSTALL_DIR"   # placer le docker-compose.yml du pack
docker compose config
docker compose pull
docker compose up -d
docker compose ps            # npm_app Up : 80, 443 publics ; 81 sur 127.0.0.1
docker compose logs -f app   # "listening on port 3000"
```

## 4. Compte admin
`http://127.0.0.1:81` (ou tunnel SSH `-L 81:127.0.0.1:81`). NPM v2.14 : écran « Welcome! » → créer
l'admin **[DEMANDER]** email + mot de passe fort. (Anciennes versions : `admin@example.com`/`changeme` → changer.)

## 5. Backends
Réseau Docker partagé pour joindre par nom de conteneur :
```bash
docker network create proxy 2>/dev/null || true
```
Sinon proxifier `IP:port` LAN. Tester : `docker exec npm_app curl -sf http://<backend>:<port>/`.

## 6. Certificats  **[DEMANDER]**

### A) Let's Encrypt DNS-01 (domaine réel, prod)
0. **[CRITIQUE] Vérifier les NS réellement délégués** : `dig NS <domaine> +short`. Le plugin DNS-01 doit
   correspondre au fournisseur qui **héberge la zone** (piège vécu : `.ovh` délégué à Cloudflare → un
   token OVH donne `NXDOMAIN`).
1. Token **scopé à la zone** chez le bon fournisseur, **sans restriction d'IP** (Cloudflare : erreur `9109`
   sinon) : Cloudflare = template « Edit zone DNS » ; OVH = `/domain/zone/*` + Validity Unlimited + `ovh-eu`.
2. **Tester depuis la machine cible** une vraie écriture TXT (le `verify` est exempté du filtrage IP → trompeur).
3. UI → Certificates → **Let's Encrypt via DNS** → provider → credentials
   (Cloudflare : `dns_cloudflare_api_token=<token>` · OVH : `dns_ovh_endpoint=ovh-eu` + keys) →
   `*.<domaine>` + `<domaine>`, propagation 30–120 s. **Aucun port à ouvrir.**

### B) CA interne `*.maison.lan` (local)
```bash
mkdir -p "$INSTALL_DIR/ca" && cd "$INSTALL_DIR/ca"
openssl genrsa -out maison-ca.key 4096
openssl req -x509 -new -nodes -key maison-ca.key -sha256 -days 3650 \
  -subj "/C=FR/O=Labo Maison/CN=Maison LAN Root CA" -out maison-ca.crt
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
openssl verify -CAfile maison-ca.crt wildcard.crt
```
UI → Certificates → **Custom Certificate** (Key/Certificate/Intermediate=CA). Résolution : DNS local ou
`/etc/hosts`. **Importer `maison-ca.crt`** sur chaque client (sinon avertissement = normal, cf. TROUBLESHOOTING).
⚠️ Le **SAN** est obligatoire (CN seul refusé par les navigateurs modernes).

## 7. Hôte proxy (par service)
UI → Hosts → Proxy Hosts → Add :
- Details : Domain `photos.<domaine>`, Scheme `http`, Forward `<backend>:<port>`, Block Common Exploits
  (+ Websockets si besoin).
- SSL : certificat + **Force SSL** + **HTTP/2** + **HSTS** → Save → **Online**.

## 8. Vérification TLS (obligatoire)
```bash
curl -sI http://photos.<domaine> | grep -i location                 # 301 -> https
curl -s [--cacert ca/maison-ca.crt] -o /dev/null \
  -w 'code=%{http_code} tls=%{ssl_verify_result} proto=%{http_version}\n' https://photos.<domaine>/
openssl s_client -connect <hote>:443 -servername photos.<domaine> [-CAfile ca/maison-ca.crt] </dev/null 2>/dev/null \
  | grep -E 'subject=|issuer=|Verify return'
curl -sI [--cacert ...] https://photos.<domaine> | grep -i strict-transport
```
Attendu : 301, `tls=0`, `proto=2`, HSTS présent, backend répond à travers le proxy.

## 9. Backup / restore / update
- Backup : `data/` + `letsencrypt/` (+ compose), CA à part. `BACKUP.md`.
- Restore : `tar xzf` → `up -d` → Online. `RESTORE.md` (testée).
- Update : bump tag → `pull` + `up -d` (migrations auto). `UPDATE.md`. Sauver d'abord.

## Rollback / désinstallation
```bash
docker compose down                 # conserve data/
# DESTRUCTIF (confirmation requise) :
# docker compose down && sudo rm -rf "$INSTALL_DIR"   # + /etc/hosts + CA des clients
```

## Checklist finale
- [ ] Docker/Compose ; ports 80/443/81 libres
- [ ] Image épinglée `:2.14.0` ; admin `127.0.0.1`
- [ ] Admin créé (mot de passe fort) ; :81 non exposée
- [ ] Cert OK (LE DNS-01 valide, ou CA interne + CA importée clients)
- [ ] Hôte(s) Online ; Force SSL + HTTP/2 + HSTS ; Block Common Exploits
- [ ] Vérif TLS réelle (301, tls=0, HSTS, backend OK)
- [ ] Backup + restauration testée
