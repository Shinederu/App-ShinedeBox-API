# ShinedeBox API

## Role

Backend PHP de ShinedeBox. Il gere les metadonnees SQL, le stockage physique
hors webroot, les telechargements controles et les liens publics par token.

Le code projet stable est `box`.

## Repo et deploiement

- Repo source: `P:\DEV\GitHub\App-ShinedeBox-API`
- Repo GitHub: `https://github.com/Shinederu/App-ShinedeBox-API.git`
- Runtime API: `P:\PROD\API\box`
- Endpoint public: `https://api.shinederu.ch/box/`
- Frontend proprietaire: `P:\DEV\GitHub\App-ShinedeBox`
- Runtime frontend: `P:\PROD\ShinedeBox`
- Stockage persistant: `P:\PROD\ShinedeBoxStorage\files`

`P:\PROD\API\box` doit contenir uniquement le runtime necessaire a l'API. Les
fichiers `.env`, `logs/` et `_ratelimit/` sont des elements runtime preserves,
pas des sources a copier depuis DEV.

## Endpoints

- `GET /box/auth.php?action=status`
- `GET /box/auth.php?action=logout`
- `GET /box/list.php`
- `POST /box/upload.php`
- `POST /box/rename.php`
- `POST /box/delete.php`
- `GET /box/share.php?id=<file_id>`: liste des liens d'un fichier, permission
  requise.
- `POST /box/share.php`: creation ou revocation d'un lien public, permission
  requise.
- `GET /box/share.php?token=<token>`: lecture publique des infos d'un partage.
- `GET /box/download.php?id=<file_id>`: telechargement avec permission.
- `GET /box/download.php?token=<token>`: telechargement public via partage.

Les reponses JSON suivent le contrat transversal:

```json
{
  "success": true,
  "data": {}
}
```

Pour compatibilite avec le frontend statique actuel, les champs utiles restent
aussi exposes au premier niveau (`files`, `stats`, `share`, etc.).

Les erreurs suivent:

```json
{
  "success": false,
  "error": "Message lisible"
}
```

## Authentification et permissions

- Session commune via cookie `sid` et tables `auth_sessions` / `users`.
- Un utilisateur banni (`users.is_banned`) est refuse; si la colonne existe, sa
  session Box est supprimee lors du controle.
- Acces metier via `Module-ShinedeCore-PHP/services/ProjectAccessService.php`.
- Permission stable requise: `box.files.manage`.
- `core.super_admin` donne le bypass global via le service partage.

Modele d'acces actuel:

- ShinedeBox fonctionne comme une bibliotheque commune pour les utilisateurs
  autorises.
- Tout utilisateur avec `box.files.manage` peut lister, uploader, telecharger,
  renommer, supprimer et partager les fichiers actifs.
- `box_files.owner_user_id` conserve l'utilisateur deposant pour audit; il ne
  filtre pas encore les droits.
- Les liens publics par token donnent acces uniquement au fichier cible, sans
  session ni acces a la bibliotheque.

## Base de donnees

Instance: MySQL `8.0`, schema partage `ShinedeCore`.

Migration principale:

- `sql/001_box_files.sql`

Tables proprietaires:

- `box_files`: metadonnees fichier, proprietaire d'audit, nom public, nom
  stocke, taille, MIME, checksum, soft delete.
- `box_shares`: tokens publics, createur d'audit, expiration optionnelle, limite
  optionnelle de telechargements.
- `box_download_events`: journal minimal des telechargements avec hashes
  IP/user-agent.

Les suppressions UI sont des soft deletes (`deleted_at`) et ne suppriment pas
immediatement le fichier physique.

## Dossiers runtime et fichiers partages

- API runtime: `P:\PROD\API\box`
- Stockage fichiers: `P:\PROD\ShinedeBoxStorage\files`
- Logs API: `P:\PROD\API\box\logs\box.log`
- Rate limit local: `P:\PROD\API\box\_ratelimit`

Le dossier de stockage appartient a ShinedeBox. Les autres projets doivent y
acceder par API HTTP, pas par ecriture directe.

## Temps reel et evenements

Aucun flux Mercure n'est publie ou consomme actuellement. Les clients peuvent
toujours reconstruire l'etat via:

- `GET https://api.shinederu.ch/box/list.php`
- `GET https://api.shinederu.ch/box/share.php?id=<file_id>`

Si ShinedeBox ajoute du temps reel plus tard, utiliser Mercure pour les
evenements et garder l'API HTTP comme source de resynchronisation.

Topics recommandes:

```text
https://api.shinederu.ch/box/topics/files
https://api.shinederu.ch/box/topics/files/{public_id}
```

Types recommandes:

```text
box.file.created
box.file.renamed
box.file.deleted
box.share.created
box.share.revoked
box.file.downloaded
```

## Dependances inter-projets

- `Module-Auth-API`: sessions et utilisateurs.
- `Module-ShinedeCore-PHP`: permissions centralisees.
- `App-ShinedeBox`: frontend statique consommateur.

ShinedeBox ne doit pas modifier directement les tables proprietaires d'un autre
projet. Toute integration future passe par API HTTP documentee.

## Configuration

Ordre de chargement reel:

1. `.env` local au runtime API (`P:\PROD\API\box\.env` en production).
2. `.env` de l'API Auth comme fallback pour les credentials DB partages.

`.env.example` est seulement un modele sans secret; il n'est pas charge par le
runtime.

Variables principales:

- `BASE_URL`
- `BOX_API_BASE`
- `AUTH_PORTAL_URL`
- `AUTH_API_BASE`
- `DB_TYPE`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS`
- `BOX_DB_TYPE`, `BOX_DB_HOST`, `BOX_DB_PORT`, `BOX_DB_NAME`, `BOX_DB_USER`,
  `BOX_DB_PASS` pour surcharger la DB de cette API.
- `UPLOAD_DIR`
- `MAX_FILE_MB`
- `ALLOWED_EXT`
- `BLOCKED_EXT`
- `ALLOWED_MIME`

Compatibilite historique: les variables `MQ_DB_*` peuvent encore etre lues en
dernier fallback, mais ne doivent pas etre utilisees pour une nouvelle config
ShinedeBox.

En production Docker:

```text
DB_HOST=MySQL
UPLOAD_DIR=/var/www/ShinedeBoxStorage/files
```

Depuis le poste Codex Windows, les tests CLI peuvent utiliser `192.168.10.10`.

## Logs et observabilite

Les erreurs applicatives sont ecrites via `error_log` et dans:

```text
P:\PROD\API\box\logs\box.log
```

Ne jamais logger de secret, mot de passe, token de session ou JWT complet.

## Verifications

```powershell
cd P:\DEV\GitHub\App-ShinedeBox-API
Get-ChildItem . -Recurse -Filter *.php | % { php -l $_.FullName }
```

## Deploiement

Copier uniquement le runtime utile vers `P:\PROD\API\box`:

```powershell
Copy-Item -LiteralPath *.php -Destination P:\PROD\API\box -Force
Copy-Item -LiteralPath services -Destination P:\PROD\API\box -Recurse -Force
```

Preserver en production:

- `.env`
- `logs/`
- `_ratelimit/`
- stockage `P:\PROD\ShinedeBoxStorage`

Ne pas deployer `.git`, `README.md`, `AGENTS.md`, caches, tests, brouillons,
secrets, logs locaux, fichiers uploades, `.env.example` ou `sql/` en routine.
Le dossier `sql/` reste source; le copier uniquement sur demande explicite de
migration ou de reprise.

## Notes de reprise

- L'API est volontairement simple, sans Composer obligatoire.
- `ProjectAccessService.php` est resolu depuis `P:\PROD\API\core` en production
  et depuis `P:\DEV\GitHub\Module-ShinedeCore-PHP` en developpement local.
- Les liens publics utilisent le format lisible
  `https://box.shinederu.ch/s/<token>/<nom-du-fichier>`.
- L'ancien format `https://box.shinederu.ch/?share=<token>` reste gere cote
  frontend.
- Aucun changement destructif de donnees ne doit etre fait sans demande
  explicite.
