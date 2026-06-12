# Guide Agents - App-ShinedeBox-API

Lire d'abord:

1. `P:\AGENTS.md`
2. `P:\ECOSYSTEM.md`
3. `P:\DEV\GitHub\README.md`
4. `P:\DEV\GitHub\App-ShinedeBox-API\README.md`
5. `P:\DEV\GitHub\App-ShinedeBox\README.md`

## Role

Backend PHP de ShinedeBox, code projet stable `box`. Le runtime public est
`P:\PROD\API\box`, expose sous `https://api.shinederu.ch/box/`.

## Regles de travail

- Modifier la source dans `P:\DEV\GitHub\App-ShinedeBox-API`.
- Ne jamais committer `.env`, logs, `_ratelimit`, caches, vendor local ou fichiers
  uploades.
- Les secrets restent dans les `.env` runtime/projet ou `P:\DEV\Access`.
- Les fichiers utilisateurs restent hors webroot, dans
  `P:\PROD\ShinedeBoxStorage\files` en production.
- Verifier les fichiers PHP avec:

```powershell
Get-ChildItem P:\DEV\GitHub\App-ShinedeBox-API -Recurse -Filter *.php | % { php -l $_.FullName }
```

## Deploiement

Copier uniquement le runtime necessaire vers `P:\PROD\API\box`:

- `*.php`
- `services\`
- `sql\`
- `.env.example`

Preserver en production:

- `.env`
- `logs\`
- `_ratelimit\`
- tout stockage persistant hors repo

## Contrats ecosysteme

- Auth commune via `auth_sessions`, `users` et cookie `sid`.
- Un utilisateur banni (`users.is_banned`) est refuse et sa session Box est
  supprimee si la colonne existe.
- Permission stable: `box.files.manage` via
  `Module-ShinedeCore-PHP\services\ProjectAccessService.php`.
- Tables proprietaires: `box_files`, `box_shares`, `box_download_events`.
- Aucun Mercure actuellement; toute commande metier passe par API HTTP et doit
  rester rejouable par lecture HTTP.
