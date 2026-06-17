# TRB — Troubleshooting M7: CLI Shortcuts

Registro degli incidenti affrontati durante lo sviluppo e il test di M7.

---

## TRB-M7-001 — `Repository not found` con token corretto nel remote URL

**Sintomo**: `NomadSyncPull.bat` fallisce con:
```
fatal: repository 'https://github.com/AleDeP10/nomad-test-vault/' not found
```
nonostante `git remote get-url origin` mostri l'URL con il token.

**Root cause**: il token GitHub era **scaduto**. GitHub risponde `Repository not found`
invece di `401 Unauthorized` quando il token esiste ma non ha accesso al repository
(token scaduto o scope insufficiente).

**Diagnosi**:
```powershell
# Verifica che il token sia valido
Invoke-WebRequest -Uri "https://api.github.com/user" `
    -Headers @{"Authorization" = "token ghp_..."} | Select-Object StatusCode
# 200 = token valido; 401 = token scaduto/revocato

# Verifica che il repository esista con quel nome
Invoke-WebRequest -Uri "https://api.github.com/repos/<owner>/<name>" `
    -Headers @{"Authorization" = "token ghp_..."} | Select-Object StatusCode
# 200 = esiste; 404 = non esiste o token senza scope repo
```

**Fix**: rigenerare il token su GitHub con scope `repo` (full control of private
repositories). Aggiornare con:
```bat
NomadSyncConfig.bat --git.token=ghp_nuovo_token
```

**Prevenzione**: usare token senza scadenza (`No expiration`) per automazioni,
o impostare reminder per il rinnovo.

---

## TRB-M7-002 — `buildAuthenticatedCommand` bypassa il token di `bootstrapVault`

**Sintomo**: `bootstrapVault` logga `remote URL configured` ma il pull usa ancora
l'URL senza token.

**Root cause**: integrazione di codice esterno (`buildAuthenticatedCommand`) che
costruisce un approccio alternativo alle credenziali via `credential.helper=` e
`credential.username=` inline nel comando. I due approcci sono in conflitto:
`bootstrapVault` scrive l'URL con token in `.git/config`, ma `buildAuthenticatedCommand`
svuota il credential helper (`credential.helper=`) e usa un meccanismo diverso che
non funziona su Windows senza Git Bash.

**Fix**: eliminare `buildAuthenticatedCommand`. `pull` e `push` usano
`git pull/push` semplice — il token è già nell'URL del remote configurato da
`bootstrapVault`. Nessuna credenziale nei comandi, nessuna dipendenza da shell.

**Lezione**: non mescolare due approcci di autenticazione Git nello stesso flusso.
Il token nell'URL del remote è cross-platform e non richiede configurazioni
aggiuntive nei comandi.

---

## TRB-M7-003 — Token loggato in chiaro da `NomadSyncConfig`

**Sintomo**: il log mostrava `config: set git.token=ghp_...` con il token in chiaro.

**Root cause**: `handleConfig` in `Main` loggava tutti i flag git senza distinzione.

**Fix**:
1. In `Main.handleConfig`: `git.token` loggato come `<hidden>` via confronto con
   `NomadProperties.Git.TOKEN`
2. In `CommandUtil`: nuovo overload `runCommand(directory, command, Set<String> sensitiveArgs, logService)` — gli argomenti nel set sono sostituiti da `<hidden>`
   nel log prima di avviare il processo
3. In `GitService.bootstrapVault`: la chiamata `remote set-url` passa
   `Set.of(remoteUrl)` per mascherare l'URL contenente il token


---

## TRB-M7-004 — `NomadSyncStatus` output su una sola riga

**Sintomo**: `NomadSyncStatus.bat` stampava tutto su una riga senza separatori:
```
On branch mainYour branch is up to date with 'origin/main'.nothing to commit...
```

**Root cause**: `CommandUtil.runCommandWithOutput` concatena le righe senza
separatori — progettato per output machine-readable come `--porcelain`.

**Fix**: nuovo metodo `CommandUtil.runCommandWithLines` che usa `System.lineSeparator()`
tra le righe. `GitService.status()` usa `runCommandWithLines` invece di
`runCommandWithOutput`.

---

## TRB-M7-005 — Processo appeso dopo `NomadSyncPull` da IntelliJ

**Sintomo**: `NomadSyncPull.bat` eseguito da IntelliJ non terminava —
il processo rimaneva in `blocking take()` sulla queue.

**Root cause**: il worker thread di `SyncOrchestrator` attende indefinitamente
nuovi eventi. In produzione termina via Ctrl+C (shutdown hook); da IntelliJ il
processo padre non invia il segnale automaticamente.

**Fix**: flag `--daemon`. Senza flag (default), `Main` chiama `awaitIdle()` dopo
aver pubblicato l'evento — poll ogni 250ms sulle code per-vault; a coda vuota +
500ms di settling, chiama `System.exit(0)`. Con `--daemon`: comportamento originale.

**Nota**: anche `pause` in `NomadSync.bat` causava l'attesa di un tasto — rimosso.

---

## TRB-M7-006 — Vault non trovato: cartella rinominata diversamente dal repository

**Sintomo**: `fatal: repository 'https://github.com/.../nomad-test-vault/' not found`
anche dopo la rigenerazione del token.

**Root cause**: il repository su GitHub si chiama `nomad-test-vault`, ma la cartella
locale si chiama `nomad-test`. Il campo `name` in `vaults.json` era impostato
correttamente (`nomad-test-vault`), ma la verifica via API GitHub restituiva 404.

**Diagnosi**: il token era scaduto indipendentemente dalla questione del nome.
Dopo la rigenerazione del token, l'API ha restituito 200 e il pull ha funzionato.

**Lezione**: `Repository not found` con token valido indica quasi sempre token
scaduto o scope insufficiente, non un problema di naming. Verificare sempre il
token per primo.
