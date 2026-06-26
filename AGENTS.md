# Guide Agents - App-ShinedeBox-API

## Lecture de demarrage

Lire dans cet ordre:

1. `P:\AGENTS.md`
2. `P:\ECOSYSTEM.md`
3. `P:\DEV\GitHub\README.md`
4. `P:\DEV\GitHub\AGENTS.md`
5. `P:\DEV\GitHub\App-ShinedeBox-API\README.md`

Lire aussi `P:\DEV\GitHub\App-ShinedeBox\README.md` si la tache inclut le
frontend ou si une analyse de dependance est necessaire sans modification.

## Role

Backend PHP de ShinedeBox, code projet stable `box`. Le runtime public est
`P:\PROD\API\box`, expose sous `https://api.shinederu.ch/box/`.

## Perimetre strict

Modifier uniquement:

- `P:\DEV\GitHub\App-ShinedeBox-API`
- `P:\PROD\API\box` lors d'un deploiement API demande

Ne pas modifier au passage:

- `App-ShinedeBox`
- `Module-Auth-API`
- `Module-ShinedeCore-PHP`
- tout autre repo applicatif
- `P:\PROD\ShinedeBox`
- `P:\PROD\ShinedeBoxStorage`, sauf operation explicitement demandee sur le
  stockage

Si un probleme semble venir d'un autre projet, le documenter dans le compte-rendu
ou dans `P:\DEV\AI-Exchange\Reports\Codex`, sans secret, puis attendre une
demande explicite.

## Regles de travail

- Travailler sur `main`.
- Faire `git pull --rebase` avant modification.
- Ne jamais committer `.env`, logs, `_ratelimit`, caches, vendor local ou
  fichiers uploades.
- Les secrets restent dans les `.env` runtime/projet ou dans `P:\DEV\Access`.
- Les fichiers utilisateurs restent hors webroot, dans
  `P:\PROD\ShinedeBoxStorage\files` en production.
- Toute evolution DB significative doit etre idempotente et placee dans `sql/`.
- Ne pas supprimer de donnees sans demande explicite.

## Contrats ecosysteme

- API publique: `https://api.shinederu.ch/box/`.
- Auth commune via `auth_sessions`, `users` et cookie `sid`.
- Un utilisateur banni (`users.is_banned`) est refuse et sa session Box est
  supprimee si la colonne existe.
- Permission stable: `box.files.manage`.
- Service permissions:
  `Module-ShinedeCore-PHP\services\ProjectAccessService.php`.
- Tables proprietaires: `box_files`, `box_shares`, `box_download_events`.
- Stockage proprietaire: `P:\PROD\ShinedeBoxStorage\files`.
- Aucun Mercure actuellement; toute commande metier passe par API HTTP et doit
  rester rejouable par lecture HTTP.

## Verification

```powershell
Get-ChildItem P:\DEV\GitHub\App-ShinedeBox-API -Recurse -Filter *.php | % { php -l $_.FullName }
git -c safe.directory=* -C P:\DEV\GitHub\App-ShinedeBox-API status --short --branch
```

Smoke public:

```powershell
Invoke-WebRequest -Uri 'https://api.shinederu.ch/box/auth.php?action=status' -UseBasicParsing
```

## Deploiement

Copier uniquement le runtime necessaire vers `P:\PROD\API\box`:

- `*.php`
- `services\`

Preserver en production:

- `.env`
- `logs\`
- `_ratelimit\`
- `P:\PROD\ShinedeBoxStorage`

Ne pas deployer README, AGENTS, `.env.example`, `sql\`, `.git`, `.github`,
caches, tests, brouillons, logs locaux, fichiers uploades ou secrets.

Le dossier `sql\` reste dans le repo source; ne le deployer ou appliquer que sur
demande explicite de migration ou de reprise.

## Verification de fin apres deploiement

```powershell
Get-ChildItem -LiteralPath P:\PROD\API\box -Force
Get-ChildItem -LiteralPath P:\PROD\API\box -Recurse -Filter *.php | % { php -l $_.FullName }
```
