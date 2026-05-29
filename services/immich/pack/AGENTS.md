# AGENTS.md — Installer Immich (galerie photo auto-hébergée) sur un Docker existant

> Fichier d'instructions pour un agent IA de codage (Codex, Claude Code, Cursor, Aider, etc.).
> Objectif : **installer Immich via Docker Compose** sur une machine disposant **déjà de Docker +
> Docker Compose**, le tester réellement (upload photo/vidéo), puis mettre en place
> sauvegarde/restauration.
>
> Immich remplace Google Photos en auto-hébergé (timeline, albums, appli mobile, recherche IA,
> reconnaissance faciale). Stack de **4 conteneurs** : `server` + `machine-learning` +
> `PostgreSQL` (avec extension vecteurs) + `Valkey` (Redis).

---

## Périmètre de l'agent
Installation **entièrement automatisable** sur un hôte Docker existant (ni ISO, ni console). Toutes
les commandes passent en CLI ; seule la création du compte admin se fait dans l'UI web. Déroule les
sections 3 → 8 de façon autonome, en t'arrêtant aux points **[DEMANDER]**.

## 1. Contraintes de sécurité (impératives)
1. Le port web (2283) ne doit **jamais** être exposé en clair sur Internet (ce sont des photos
   personnelles) : HTTPS via reverse proxy + 2FA, ou accès par VPN. Voir `SECURITY.md`.
2. PostgreSQL et Valkey/Redis ne doivent **jamais** être publiés sur l'hôte (aucune section `ports:`
   pour ces services). Le compose officiel respecte déjà cette règle : ne pas la défaire.
3. Aucune opération destructive (`docker compose down -v`, `rm -rf` sur `UPLOAD_LOCATION` /
   `DB_DATA_LOCATION`) sans **accord explicite** de l'humain : ce sont les photos + la base.
4. Toujours **sauvegarder avant** une montée de version (voir `UPDATE.md`).
5. `DB_PASSWORD` = secret aléatoire fort (`A-Za-z0-9`) ; ne jamais committer le `.env`.

## 2. Paramètres à confirmer  **[DEMANDER]**
| Variable | Défaut conseillé | Rôle |
|---|---|---|
| `INSTALL_DIR` | `/srv/docker/immich` | Dossier projet (compose + .env) |
| `UPLOAD_LOCATION` | `./library` | Stockage photos/vidéos ; **gros volume → disque dédié** |
| `DB_DATA_LOCATION` | `./postgres` | Données PostgreSQL ; **jamais sur partage réseau** |
| `IMMICH_VERSION` | dernière `vX.Y.Z` | Épingler (éviter `release`/`v2` flottant) |
| `TZ` | `Europe/Paris` | Fuseau horaire |
| `EXPOSURE` | LAN / reverse proxy / VPN | Détermine le binding du port (§6) |

Pré-vérifs : `docker --version`, `docker compose version`, port **2283 libre**
(`ss -tulpn | grep 2283`), disque (≈ 6 Go d'images + les photos), RAM ≥ ~4 Go (la ML est gourmande).

### 2bis. Docker absent (machine vierge)  **[DEMANDER confirmation]**
Ce guide part d'un hôte Docker existant. Si `docker --version` échoue, installer Docker d'abord
(testé Debian/Ubuntu, script officiel) — opération système, **accord humain requis** :
```bash
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker "$USER"        # re-login requis ensuite
sudo systemctl enable --now docker
docker run --rm hello-world
```
> VM Proxmox neuve : créer la VM (image cloud Debian + cloud-init) puis ce bloc. Hors Debian/Ubuntu :
> https://docs.docker.com/engine/install/ (éviter le `docker.io` trop ancien des distros).

## 3. Fichiers officiels (épinglés)
```bash
mkdir -p "$INSTALL_DIR" && cd "$INSTALL_DIR"
curl -fsSL -o docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
curl -fsSL -o example.env       https://github.com/immich-app/immich/releases/latest/download/example.env
```
Toujours utiliser le compose **de la release** (celui de `main` peut diverger).

## 4. Configurer `.env`
```bash
cp example.env .env
#   UPLOAD_LOCATION / DB_DATA_LOCATION / TZ=Europe/Paris
#   IMMICH_VERSION=vX.Y.Z (version exacte)
#   DB_PASSWORD=$(openssl rand -hex 16)
chmod 600 .env
```

## 5. Démarrage + vérification
```bash
docker compose config       # syntaxe OK + confirme qu'aucun ports: n'expose db/redis
docker compose pull         # ~6 Go
docker compose up -d
docker compose ps           # 4 conteneurs "healthy"
docker compose logs -f immich-server   # jusqu'à "Immich Microservices is running [vX.Y.Z]"
curl http://localhost:2283/api/server/ping     # -> {"res":"pong"}
```
Premier démarrage : import d'une base geodata (~224 k entrées) → délai normal.

## 6. Réseau / exposition  **[DEMANDER]**
- Accès LAN/local : conserver `2283:2283`.
- Derrière reverse proxy : remplacer `'2283:2283'` par `'127.0.0.1:2283:2283'` (service `immich-server`),
  puis `docker compose up -d` ; HTTPS + 2FA devant. Jamais 2283 nu sur Internet.

## 7. Création du compte admin (1er utilisateur = administrateur)
**[DEMANDER]** email + mot de passe fort + nom.

**Méthode A — API (recommandée pour un agent sans navigateur)** — vérifiée sur v2.7.x :
```bash
# Crée l'admin (uniquement tant qu'aucun admin n'existe ; sinon 400 "already has an admin")
curl -fsS -X POST http://localhost:2283/api/auth/admin-sign-up \
  -H 'Content-Type: application/json' \
  -d '{"email":"<ADMIN_EMAIL>","password":"<ADMIN_PWD>","name":"<NOM>"}'
# (Optionnel) token pour piloter le reste par API :
curl -fsS -X POST http://localhost:2283/api/auth/login \
  -H 'Content-Type: application/json' \
  -d '{"email":"<ADMIN_EMAIL>","password":"<ADMIN_PWD>"}'   # -> {"accessToken":"...","isAdmin":true,...}
```
> Si `400 {"message":"The server already has an admin"}` : l'admin existe déjà → passer à la connexion.

**Méthode B — UI web (navigateur dispo / humain)** : `http://<hôte>:2283` → *Commencer* → formulaire admin
→ connexion → onboarding (thème, langue, vie privée, modèle de stockage, sauvegardes).

Dans les deux cas : activer la **2FA** admin si l'instance est exposée.

## 8. Test fonctionnel réel
Uploader **une photo ET une vidéo** via l'UI, vérifier leur présence dans la timeline et leur
ouverture. Contrôle disque : `docker exec immich_server ls -laR /data/upload`.

## 8bis. Appli mobile (LE cas d'usage : remplacer Google Photos) — testé sur Android réel
Sauvegarde automatique des photos du téléphone (la vraie raison d'installer Immich).
1. Installer l'appli **Immich** (Play Store / App Store / APK GitHub selon l'**ABI** du tel : `arm64-v8a`,
   `armeabi-v7a` pour un 32-bit, ou l'APK universel `app-release.apk`).
2. Saisir l'**URL serveur** (`http://<hôte>:2283` en LAN, **`https://…` via reverse proxy** hors domicile) → connexion.
3. Autoriser l'accès photos → écran **Sauvegarde** : sélectionner **les albums voulus** (pas tout par défaut)
   → activer l'interrupteur. Vérifier **Sauvegardé = Total**.
4. Côté serveur, l'asset a un `deviceId` propre au téléphone (≠ `WEB`) et apparaît dans l'UI web.
> ⚠️ Hors WiFi maison (4G/5G), la sauvegarde exige un serveur joignable de l'extérieur : reverse proxy
> HTTPS + 2FA, ou VPN. Jamais le 2283 nu sur Internet (cf. `SECURITY.md`).

## 9. Sauvegarde / restauration / mise à jour
- **Sauvegarde** : dump PostgreSQL + dossiers `upload/library/profile` (voir `BACKUP.md`). 3-2-1.
- **Restauration** : voir `RESTORE.md`. ⚠️ Piège : sans `thumbs/`+`encoded-video/`+`backups/` dans le
  backup, recréer ces dossiers + `touch .immich`, sinon le serveur boucle (`Failed to read .immich`).
- **Mise à jour** : `UPDATE.md` (release notes → sauvegarde → bump `IMMICH_VERSION` → `pull` + `up -d`).

## Rollback / désinstallation
```bash
docker compose down                 # stop, données conservées
# DESTRUCTIF (confirmation requise) :
# docker compose down -v && sudo rm -rf "$UPLOAD_LOCATION" "$DB_DATA_LOCATION"
```

## Checklist finale
- [ ] Docker + Compose présents, port 2283 libre
- [ ] Compose + .env de la **release** épinglée, `DB_PASSWORD` aléatoire
- [ ] `db`/`redis` non publiés (vérifié `docker compose ps`)
- [ ] 4 conteneurs `healthy`, `ping` → pong
- [ ] Admin créé, 2FA si exposé, reverse proxy HTTPS si non-LAN
- [ ] Upload photo+vidéo OK ; **sauvegarde testée** et **restauration vérifiée**
