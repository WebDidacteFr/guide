# CLAUDE.md — Installer Immich (galerie photo auto-hébergée) sur un Docker existant

> Donne ce fichier à ton agent IA (Claude Code, Codex, etc.). Il décrit comment **installer Immich
> avec Docker Compose** sur une machine qui a **déjà Docker + Docker Compose**, le tester pour de
> vrai (upload photo/vidéo), et mettre en place sauvegarde/restauration.
>
> Immich = remplaçant auto-hébergé de Google Photos (timeline, albums, appli mobile, recherche par
> IA, reconnaissance faciale). Stack = 4 conteneurs : **server + machine-learning + PostgreSQL(vecteurs) + Valkey(Redis)**.

---

## 0. Rôle de l'agent
Tu installes Immich **en autonomie** sur un hôte Docker existant (pas d'install ISO, pas de console).
Tout se fait en ligne de commande + une création de compte admin via l'UI web. Tu peux dérouler les
sections 3 → 8 seul, en t'arrêtant aux points marqués **[DEMANDER]**.

## 1. Règles de sécurité (NON négociables)
1. **Ne jamais exposer** le port web (2283) en clair sur Internet : photos perso. HTTPS via reverse
   proxy + 2FA, ou accès VPN uniquement. (cf. `SECURITY.md`)
2. **Ne jamais publier** PostgreSQL ni Valkey/Redis sur l'hôte (pas de section `ports:` pour eux).
   Le compose officiel est déjà correct — ne pas le « casser ».
3. **Ne jamais** faire `docker compose down -v` ni `rm -rf` sur `UPLOAD_LOCATION`/`DB_DATA_LOCATION`
   sans **confirmation explicite** de l'humain : ce sont les photos + la base (irremplaçables).
4. **Sauvegarder AVANT** toute mise à jour (cf. `UPDATE.md`).
5. `DB_PASSWORD` = secret aléatoire fort (`A-Za-z0-9`). Le `.env` ne se commite jamais.

## 2. Variables à demander/confirmer  **[DEMANDER]**
| Variable | Défaut raisonnable | Description |
|---|---|---|
| `INSTALL_DIR` | `/srv/docker/immich` | Dossier du projet (compose + .env) |
| `UPLOAD_LOCATION` | `./library` | Où sont stockées les photos/vidéos. **Gros volume → disque/montage dédié** |
| `DB_DATA_LOCATION` | `./postgres` | Données PostgreSQL. **Jamais sur un partage réseau** |
| `IMMICH_VERSION` | dernière `vX.Y.Z` | Épingler une version précise (pas `release`/`v2` flottant) |
| `TZ` | `Europe/Paris` | Fuseau horaire |
| `EXPOSURE` | LAN / reverse proxy / VPN | Conditionne le binding du port (cf. §6) |

Prérequis à vérifier : `docker --version`, `docker compose version`, **port 2283 libre**
(`ss -tulpn | grep 2283`), espace disque suffisant (images ≈ 6 Go + tes photos), RAM ≥ ~4 Go conseillée
(la machine-learning est gourmande).

## 3. Récupérer les fichiers officiels (versions épinglées)
```bash
mkdir -p "$INSTALL_DIR" && cd "$INSTALL_DIR"
curl -fsSL -o docker-compose.yml https://github.com/immich-app/immich/releases/latest/download/docker-compose.yml
curl -fsSL -o example.env       https://github.com/immich-app/immich/releases/latest/download/example.env
```
> Toujours partir du compose **de la release** (celui de `main` peut être incompatible).

## 4. Configurer `.env`
```bash
cp example.env .env
# Éditer .env :
#   UPLOAD_LOCATION=...      DB_DATA_LOCATION=...      TZ=Europe/Paris
#   IMMICH_VERSION=vX.Y.Z    (épingler la version exacte)
#   DB_PASSWORD=$(openssl rand -hex 16)   (A-Za-z0-9 only)
chmod 600 .env
```

## 5. Démarrer et vérifier
```bash
docker compose config        # valide la syntaxe + confirme qu'AUCUN ports: n'expose db/redis
docker compose pull          # ~6 Go (server 3 Go, ML 2 Go, postgres 1 Go, valkey)
docker compose up -d
docker compose ps            # attendre les 4 conteneurs "healthy" (~30 s à plusieurs min au 1er boot)
docker compose logs -f immich-server   # jusqu'à "Immich Microservices is running [vX.Y.Z]"
curl http://localhost:2283/api/server/ping     # -> {"res":"pong"}
```
Au 1er démarrage, le serveur importe une base geodata (~224 k entrées) → patience.

## 6. Exposition / réseau  **[DEMANDER]**
- **Accès LAN/local simple** : laisser `2283:2283` (défaut). OK pour tester.
- **Derrière un reverse proxy** : binder en local pour que SEUL le proxy accède à Immich —
  dans `docker-compose.yml`, service `immich-server`, remplacer `'2283:2283'` par `'127.0.0.1:2283:2283'`,
  puis `docker compose up -d`. Mettre HTTPS + 2FA devant. **Jamais** 2283 nu sur Internet.

## 7. Mise en service (UI) — création de l'admin
1. Ouvrir `http://<hôte>:2283` → « Bienvenue sur Immich » → **Commencer**.
2. Le **1er utilisateur créé = administrateur**. Renseigner email + mot de passe fort + nom. **[DEMANDER]** les identifiants.
3. Se connecter, dérouler l'onboarding (thème, langue, vie privée, modèle de stockage, sauvegardes).
4. **Activer la 2FA** sur le compte admin (Paramètres) si l'instance est exposée.

## 8. Test fonctionnel réel (à faire, ne pas supposer)
- Uploader **une photo ET une vidéo** depuis l'UI (bouton *Envoyer*). Vérifier qu'elles apparaissent
  dans la timeline et s'ouvrent. Côté disque : `docker exec immich_server ls -laR /data/upload`.

## 8bis. Appli mobile (LE cas d'usage : remplacer Google Photos) — testé sur Android réel
C'est ce pour quoi la plupart des gens installent Immich : **sauvegarde auto des photos du téléphone**.
1. Installer **Immich** (Play Store / App Store / APK des releases GitHub — bien prendre l'**ABI** du tel :
   `arm64-v8a` pour la majorité, `armeabi-v7a` pour un tel 32-bit, ou l'APK universel `app-release.apk`).
2. Ouvrir l'appli → saisir l'**URL du serveur** (`http://<hôte>:2283` en LAN, **`https://…` via reverse proxy**
   pour un accès hors domicile) → se connecter (mêmes identifiants que le web).
3. Autoriser l'accès aux photos. Dans **Sauvegarde** : choisir **quels albums** synchroniser (ne pas tout
   prendre par réflexe), puis activer l'interrupteur **Sauvegarde**. Vérifier le compteur **Sauvegardé = Total**.
4. Vérif serveur : l'asset arrive avec un `deviceId` = identifiant du téléphone (≠ `WEB`) → c'est bien la
   sauvegarde native. La photo apparaît aussi dans l'UI web.
> ⚠️ **Hors WiFi maison (4G/5G), la sauvegarde ne marche QUE si le serveur est joignable de l'extérieur** :
> reverse proxy HTTPS + 2FA, ou VPN. **Jamais** le port 2283 nu sur Internet (cf. `SECURITY.md`).

## 9. Sauvegarde / restauration / mise à jour
- **Sauvegarde** (à planifier dès maintenant) : dump PostgreSQL + dossiers `upload/library/profile`.
  Voir `BACKUP.md`. Stratégie 3-2-1 ; les originaux sont irremplaçables.
- **Restauration** : voir `RESTORE.md`. ⚠️ **Piège connu** : si le backup n'inclut pas
  `thumbs/`+`encoded-video/`+`backups/`, recréer ces dossiers avec un `touch .immich` sinon le
  serveur **crash-loop** (`Failed to read .immich`).
- **Mise à jour** : voir `UPDATE.md` (lire les release notes, **sauvegarder d'abord**, bumper
  `IMMICH_VERSION`, `pull` + `up -d` ; migrations DB automatiques).

## Rollback / désinstallation
```bash
docker compose down                 # arrête sans rien supprimer (données conservées)
# Suppression COMPLÈTE (DESTRUCTIF — demander confirmation) :
# docker compose down -v && sudo rm -rf "$UPLOAD_LOCATION" "$DB_DATA_LOCATION"
```

## Checklist finale
- [ ] Docker + Compose présents, port 2283 libre
- [ ] Compose + .env de la **release** épinglée (`IMMICH_VERSION=vX.Y.Z`), `DB_PASSWORD` aléatoire
- [ ] `db`/`redis` **non publiés** sur l'hôte (vérifié via `docker compose ps`)
- [ ] 4 conteneurs `healthy`, `ping` → pong
- [ ] Compte admin créé, **2FA** si exposé, reverse proxy HTTPS si non-LAN
- [ ] Upload photo+vidéo OK (timeline) ; **sauvegarde testée** (dump + originaux) et **restauration vérifiée**
