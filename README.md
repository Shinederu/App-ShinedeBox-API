# ShinedeBox API

## Role

Backend PHP de ShinedeBox. Il est proprietaire des metadonnees fichiers, du
stockage physique hors webroot, des liens publics, des compteurs de
telechargement et du controle d'acces metier.

Le code projet stable est `box`. Il est utilise dans les permissions, les logs,
les topics recommandes et la documentation.

## Repo et deploiement

- Repo source: `P:\DEV\GitHub\App-ShinedeBox-API`
- Repo GitHub: `https://github.com/Shinederu/App-ShinedeBox-API.git`
- Branche normale: `main`
- Runtime API: `P:\PROD\API\box`
- Endpoint public: `https://api.shinederu.ch/box/`
- Frontend proprietaire: `P:\DEV\GitHub\App-ShinedeBox`
- Runtime frontend: `P:\PROD\ShinedeBox`
- Stockage persistant: `P:\PROD\ShinedeBoxStorage\files`
- Schema DB: `ShinedeCore`
- Prefixe tables: `box_*`

`P:\PROD\API\box` doit contenir uniquement le runtime necessaire a l'API. Les
fichiers `.env`, `logs/` et `_ratelimit/` sont des elements runtime preserves.
Ils ne doivent pas etre remplaces par une copie naive depuis DEV.

## Structure

- `auth.php`: statut de session Box et logout local des cookies.
- `list.php`: liste des fichiers actifs et statistiques.
- `upload.php`: upload multi-fichiers autorise.
- `rename.php`: renommage d'un fichier actif.
- `delete.php`: soft delete d'un fichier actif.
- `share.php`: lecture publique des partages, liste admin, creation et
  revocation.
- `download.php`: streaming telechargement autorise ou public via token.
- `config.php`: chargement env, DB, auth, permissions, reponses JSON,
  validation fichiers et rate limit local.
- `services/BoxFileService.php`: logique metier fichiers, partages et
  telechargements.
- `sql/001_box_files.sql`: migration idempotente initiale.
- `.env.example`: modele de configuration sans secret.
- `AGENTS.md`: consignes de reprise locales.

## Endpoints

Base publique:

```text
https://api.shinederu.ch/box/
```

Toutes les reponses JSON de succes exposent l'enveloppe commune:

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

### Auth

`GET /box/auth.php?action=status`

- Public.
- Lit le cookie `sid` si present.
- Retourne l'etat authentifie, l'utilisateur et le droit effectif Box.
- Refuse silencieusement une session absente, expiree ou bannie.

`GET /box/auth.php?action=logout`

- Public.
- Expire les cookies `sid` et `session_id` sur `.shinederu.ch`.
- Renvoie l'URL logout Auth centrale.

### Fichiers

`GET /box/list.php`

- Permission: `box.files.manage`.
- Retourne `files` et `stats`.

`POST /box/upload.php`

- Permission: `box.files.manage`.
- Body: `multipart/form-data`, champ `files[]`.
- Retourne `results`, un resultat par fichier.
- Un upload partiellement echoue peut renvoyer `success: true` avec des entrees
  `success: false` dans `results`.

`POST /box/rename.php`

- Permission: `box.files.manage`.
- Body JSON ou form:

```json
{
  "id": 123,
  "name": "nouveau-nom.ext"
}
```

- Alias historique accepte: `new_name`.
- Retourne `file`.

`POST /box/delete.php`

- Permission: `box.files.manage`.
- Body JSON, form ou query:

```json
{
  "id": 123
}
```

- Effectue un soft delete (`deleted_at`), sans supprimer immediatement le
  fichier physique.

### Partages

`GET /box/share.php?id=<file_id>`

- Permission: `box.files.manage`.
- Retourne les liens connus du fichier, actifs et inactifs.

`GET /box/share.php?token=<token>`

- Public.
- Retourne les infos du partage si le token est actif, non expire et sous la
  limite de telechargements.

`POST /box/share.php`

- Permission: `box.files.manage`.
- Creation par defaut:

```json
{
  "file_id": 123,
  "expires_days": 30,
  "max_downloads": 10
}
```

- `expires_days` est optionnel et borne entre 1 et 3650.
- `max_downloads` est optionnel et borne entre 1 et 1000000.
- Revocation:

```json
{
  "action": "revoke",
  "token": "012345..."
}
```

### Telechargements

`GET /box/download.php?id=<file_id>`

- Permission: `box.files.manage`.
- Stream le fichier physique.
- Incremente `box_files.download_count`.
- Ajoute une entree `box_download_events`.

`GET /box/download.php?token=<token>`

- Public si le token est utilisable.
- Stream le fichier physique.
- Incremente le compteur fichier et le compteur partage.
- Ajoute une entree `box_download_events`.

## Authentification et permissions

Source auth:

- API: `https://api.shinederu.ch/auth/`
- Cookie: `sid`
- Tables: `auth_sessions`, `users`

Controle metier:

- Service: `Module-ShinedeCore-PHP/services/ProjectAccessService.php`
- Appel attendu: `hasPermission($userId, 'box', 'files.manage')`
- Permission stable: `box.files.manage`
- Bypass global: `core.super_admin`

Regles:

- Un utilisateur non authentifie recoit `401`.
- Un utilisateur authentifie sans permission recoit `403`.
- Si `users.is_banned` existe et vaut vrai, la session Box est supprimee et
  l'utilisateur est traite comme non authentifie.
- Les anciens roles (`users.role`) ne servent pas de contrat metier.

Modele d'acces actuel:

- Bibliotheque commune pour tous les utilisateurs autorises.
- Tout utilisateur avec `box.files.manage` peut lister, uploader, telecharger,
  renommer, soft-delete et partager tous les fichiers actifs.
- `owner_user_id` est une donnee d'audit, pas un filtre d'acces.
- Les liens publics par token n'ouvrent que le fichier cible.

## Base de donnees

Instance: MySQL `8.0`, schema partage `ShinedeCore`.

Migration source:

- `sql/001_box_files.sql`

Tables proprietaires:

- `box_files`
  - metadonnees fichier;
  - `public_id` unique;
  - proprietaire d'audit `owner_user_id`;
  - noms original, affichage et stockage;
  - extension, MIME, taille, checksum;
  - compteur de telechargements;
  - soft delete via `deleted_at`.
- `box_shares`
  - token public unique;
  - activation/revocation;
  - expiration optionnelle;
  - limite optionnelle de telechargements;
  - createur d'audit.
- `box_download_events`
  - journal minimal des telechargements;
  - relation fichier, partage et utilisateur optionnel;
  - hash IP et hash user-agent.

Contraintes:

- `box_files.owner_user_id` reference `users.id` avec `ON DELETE SET NULL`.
- `box_shares.file_id` reference `box_files.id` avec `ON DELETE CASCADE`.
- `box_download_events.file_id` reference `box_files.id` avec `ON DELETE CASCADE`.
- Les suppressions UI ne suppriment pas immediatement le fichier physique.

Les changements DB doivent rester non destructifs sauf demande explicite. Toute
evolution significative doit etre ajoutee dans `sql/` sous forme idempotente et
documentee ici.

## Dossiers runtime et fichiers partages

- API runtime: `P:\PROD\API\box`
- Stockage fichiers: `P:\PROD\ShinedeBoxStorage\files`
- Logs API: `P:\PROD\API\box\logs\box.log`
- Rate limit local: `P:\PROD\API\box\_ratelimit`
- Frontend public: `P:\PROD\ShinedeBox`

Le stockage appartient a ShinedeBox. Aucun autre projet ne doit ecrire
directement dans `P:\PROD\ShinedeBoxStorage`. Toute integration future doit
passer par API HTTP documentee.

Les fichiers runtime persistants ne doivent pas etre remplaces pendant un
deploiement:

- `.env`
- `logs/`
- `_ratelimit/`
- stockage `P:\PROD\ShinedeBoxStorage`

## Temps reel et evenements

Aucun flux Mercure n'est publie ou consomme actuellement.

Sources de resynchronisation HTTP:

- `GET https://api.shinederu.ch/box/list.php`
- `GET https://api.shinederu.ch/box/share.php?id=<file_id>`
- `GET https://api.shinederu.ch/box/share.php?token=<token>`

Si un temps reel est ajoute plus tard, Mercure doit porter des evenements ou
snapshots, jamais des commandes critiques. Toute commande reste HTTP.

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

Payload inter-projets recommande si necessaire:

```json
{
  "schema_version": 1,
  "type": "box.file.created",
  "source": "box",
  "target": "broadcast",
  "occurred_at": "2026-06-26T12:00:00Z",
  "actor": {
    "user_id": 1
  },
  "resource": {
    "type": "file",
    "id": 123
  },
  "data": {},
  "trace_id": "optional-correlation-id"
}
```

## Dependances inter-projets

- `Module-Auth-API`: sessions et utilisateurs.
- `Module-ShinedeCore-PHP`: permissions centralisees.
- `App-ShinedeBox`: frontend statique consommateur.

Regle de perimetre: une tache API ShinedeBox ne modifie pas ces projets voisins.
Si un probleme semble venir d'Auth, Core ou d'un autre projet, documenter le
constat et attendre une demande explicite.

## Configuration

Ordre de chargement reel:

1. `.env` local a l'API (`P:\PROD\API\box\.env` en production).
2. `.env` de l'API Auth comme fallback pour les credentials DB partages:
   `P:\PROD\API\auth\.env`.
3. Fallback equivalent selon la structure montee par Nginx/PHP-FPM, utile quand
   le chemin runtime est vu depuis `/var/www`.

`.env.example` est seulement un modele sans secret. Il n'est pas charge par le
runtime.

Variables principales:

- `BASE_URL`: base publique frontend, par defaut `https://box.shinederu.ch`.
- `BOX_API_BASE`: base publique API, par defaut `https://api.shinederu.ch/box`.
- `AUTH_PORTAL_URL`: portail auth, par defaut `https://auth.shinederu.ch`.
- `AUTH_API_BASE`: API Auth, par defaut `https://api.shinederu.ch/auth/`.
- `DB_TYPE`, `DB_HOST`, `DB_PORT`, `DB_NAME`, `DB_USER`, `DB_PASS`.
- `BOX_DB_TYPE`, `BOX_DB_HOST`, `BOX_DB_PORT`, `BOX_DB_NAME`, `BOX_DB_USER`,
  `BOX_DB_PASS` pour surcharger la DB de cette API.
- `UPLOAD_DIR`: chemin stockage fichiers hors webroot.
- `MAX_FILE_MB`: taille maximale par fichier, en MB.
- `ALLOWED_EXT`: extensions autorisees, `*` pour tout autoriser hors blocage.
- `BLOCKED_EXT`: extensions interdites, y compris en double extension.
- `ALLOWED_MIME`: MIME autorises, `*` pour tout MIME detecte non vide.

Compatibilite historique:

- `MQ_DB_*` peut encore etre lu en dernier fallback.
- Ne pas utiliser `MQ_DB_*` pour une nouvelle configuration ShinedeBox.

Valeurs attendues en production Docker:

```text
DB_HOST=MySQL
UPLOAD_DIR=/var/www/ShinedeBoxStorage/files
```

Depuis le poste Codex Windows, les tests CLI peuvent utiliser l'hote MySQL local
documente par le workspace si une connexion DB est necessaire. Ne jamais ecrire
les valeurs secretes dans le repo ou dans une reponse utilisateur.

## Securite et validation

- Les endpoints admin exigent `box.files.manage`.
- Les endpoints publics par token ne donnent acces qu'au partage cible.
- Les noms de fichiers sont normalises et refuses s'ils sont vides, trop longs
  ou contiennent des separateurs/controles.
- Les extensions bloquees sont refusees meme en double extension.
- Le MIME est detecte avec `finfo`.
- Les fichiers sont stockes sous un nom genere, pas sous le nom utilisateur.
- Les fichiers stockes sont chmod `0640` quand possible.
- Les reponses JSON envoient `X-Content-Type-Options: nosniff`.
- Les telechargements nettoient le nom de fichier dans `Content-Disposition`.
- Les IP et user-agents des telechargements sont hashes avant journalisation DB.
- Ne jamais logger de secret, mot de passe, cookie `sid`, token prive, JWT ou
  valeur `.env`.

Rate limit local par IP:

- `auth`: 60 requetes / 60 s
- `list`: 120 requetes / 60 s
- `upload`: 30 requetes / 60 s
- `rename`: 30 requetes / 60 s
- `delete`: 60 requetes / 60 s
- `share`: 120 requetes / 60 s
- `download`: 240 requetes / 60 s

Le rate limit ecrit dans `_ratelimit/`, dossier runtime preserve en production.

## Logs et observabilite

Erreurs applicatives:

```text
P:\PROD\API\box\logs\box.log
```

Chaque entree ajoute la date ISO, le type d'exception, le message et la position
source. Les evenements de telechargement sont traces en DB dans
`box_download_events`.

Si des integrations inter-projets sont ajoutees plus tard, ajouter un
`trace_id` ou `request_id` relie aux logs API et evenements Mercure.

## Verifications

Lint PHP source:

```powershell
cd P:\DEV\GitHub\App-ShinedeBox-API
Get-ChildItem . -Recurse -Filter *.php | % { php -l $_.FullName }
```

Verification SQL source:

```powershell
Get-Content .\sql\001_box_files.sql
```

Smoke HTTP public sans cookie:

```powershell
Invoke-WebRequest -Uri 'https://api.shinederu.ch/box/auth.php?action=status' -UseBasicParsing
```

Smoke manuel recommande avec compte autorise:

- `GET list.php`;
- upload d'un petit fichier;
- renommage;
- creation d'un lien public;
- ouverture du lien public sans session;
- telechargement public;
- revocation du lien;
- suppression logique du fichier.

## Deploiement

Copier uniquement le runtime utile vers `P:\PROD\API\box`:

```powershell
cd P:\DEV\GitHub\App-ShinedeBox-API
Copy-Item -LiteralPath *.php -Destination P:\PROD\API\box -Force
Copy-Item -LiteralPath services -Destination P:\PROD\API\box -Recurse -Force
```

Preserver en production:

- `.env`
- `logs/`
- `_ratelimit/`
- `P:\PROD\ShinedeBoxStorage`

Ne pas deployer en routine:

- `.git`
- `.github`
- `README.md`
- `AGENTS.md`
- `.env.example`
- `sql/`
- caches
- tests
- brouillons
- logs locaux
- fichiers uploades
- secrets

Le dossier `sql/` reste source. Le copier ou appliquer uniquement sur demande
explicite de migration/reprise.

Apres copie:

```powershell
Get-ChildItem -LiteralPath P:\PROD\API\box -Force
Get-ChildItem -LiteralPath P:\PROD\API\box -Recurse -Filter *.php | % { php -l $_.FullName }
```

## Notes de reprise

- L'API est volontairement sans Composer obligatoire.
- `ProjectAccessService.php` est resolu depuis `P:\PROD\API\core` en production
  et depuis `P:\DEV\GitHub\Module-ShinedeCore-PHP` en developpement local.
- Les liens publics utilisent le format lisible:

```text
https://box.shinederu.ch/s/<token>/<nom-du-fichier>
```

- L'ancien format `https://box.shinederu.ch/?share=<token>` reste gere cote
  frontend.
- Le frontend actuel lit les champs top-level; garder cette compatibilite tant
  que `script.js` n'a pas migre vers `data`.
- Aucun changement destructif de donnees ne doit etre fait sans demande
  explicite.
- Aucune integration Corelink, Wake, Arcadia ou UniFi n'existe actuellement.

## Limites connues

- Bibliotheque commune seulement, pas d'espace prive par utilisateur.
- Pas de dossiers, tags ou descriptions editables via API publique.
- Pas d'endpoint de restauration des soft deletes.
- Pas de suppression physique automatisee documentee.
- Pas de quotas par utilisateur.
- Pas de scan antivirus integre.
- Pas de temps reel Mercure.
- Pas de tests automatises.
