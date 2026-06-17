# GRM — Milestone 8: Setup & Vault Management

**Branch principale**: `feature/m8-setup` **Sub-branch Sprint A**: `feature/vault-management`

---

## Contesto

M7 ha completato il CLI tecnico: tutte le operazioni di sincronizzazione sono accessibili da terminale con flag ben definiti. Rimane però una barriera insormontabile per l'utente non tecnico: installazione manuale, configurazione a mano di `config.properties` e `vaults.json`, registrazione del Task Scheduler a mano.

M8 abbatte questa barriera introducendo:

1. **CLI CRUD vault** — `NomadSync vault <subcommand>` per gestire i vault senza mai toccare `vaults.json`
2. **Wizard di setup console** — `NomadSync setup` guida l'utente dalla prima esecuzione alla prima sincronizzazione funzionante
3. **Registrazione automatica OS** — Task Scheduler (Windows) e launchd agent (macOS) configurati dal wizard
4. **Operazioni diagnostiche** — `log`, `version`, `doctor` per il supporto
5. **Installer via jpackage** — `.exe` su Windows, `.pkg` su macOS, JRE bundled

Al termine di M8, un utente non tecnico può installare NomadSync, aggiungere vault e avere la sincronizzazione automatica funzionante senza aprire un file di configurazione manualmente.

---

## macOS — strategia logoff (DTR-053)

**v1.0.0 — Opzione B**: NomadSync gira come user agent `launchd` in modalità `--daemon`. Il processo viene avviato da `launchd` al login con `pull --daemon`: esegue il pull al logon, poi resta vivo in ascolto. Al logout macOS invia `SIGTERM` → lo shutdown hook Java pubblica `PUSH_LOGOFF` → `awaitIdle` → exit.

```
Windows:  Task Scheduler logoff trigger → NomadSyncPush.bat → PUSH_LOGOFF event
macOS:    launchd SIGTERM → Java shutdown hook → publish PUSH_LOGOFF → awaitIdle → exit
```

Il codice Java è identico su entrambe le piattaforme. Il daemon usa `pull --daemon` (non `sync --daemon` — il push automatico in startup è inaccettabile) per garantire l'allineamento al login senza push non intenzionali.

L'esito del push viene scritto nel log — consultabile al prossimo avvio tramite `NomadSync doctor`.

**v2.0.0 — Opzione C**: `NSWorkspaceWillPowerOffNotification` (bridge Cocoa/Swift) — tempo pre-shutdown garantito. Distribuzione con certificato Apple Developer Belmani-Apex.

---

## Dicotomia `vault update` / `NomadSync config`

`vault update` gestisce i metadati strutturali del vault: `owner`, `name`, `path`, `gitBranch`, `gitRemote` **e le credenziali** (`gitToken`, `gitName`, `gitEmail`, `gitUsername`). Da utente è controintuitivo dover usare due comandi per aggiornare cose che stanno tutte in `vaults.json`.

`NomadSync config` resta il punto di accesso per gli aggiornamenti **globali** su `config.properties` (valori che si applicano a tutti i vault). La distinzione è: **per-vault** → `vault update`; **globale** → `config`. Da documentare chiaramente nel README pre-v1.0.0.

---

## Obiettivi

|#|Obiettivo|Criterio di accettazione|
|---|---|---|
|A|`NomadSync vault` — CRUD completo da CLI|add/update/remove/list/show funzionanti, vaults.json mai da toccare|
|B|`NomadSync setup` — wizard console|dalla prima esecuzione alla prima sync senza aprire file|
|C|Registrazione OS automatica|Task Scheduler (Windows) e launchd agent (macOS) dal wizard|
|D|`NomadSync log / version / doctor`|diagnostica self-service per l'utente finale|
|E|jpackage installer|`.exe` Windows + `.pkg` macOS, JRE bundled, Maven integrato|

---

## Sprint A — `NomadSync vault`: CRUD completo

**Branch**: `feature/vault-management`

### Sottocomandi

```
NomadSync vault add    --owner=<owner> --name=<name> --path=<path> [--git.*=...]
NomadSync vault update --vault=<name|owner/name> [--owner=<owner>] [--name=<name>]
                                                  [--path=<path>] [--git.branch=...]
                                                  [--git.remote=...] [--git.token=...]
                                                  [--git.name=...] [--git.email=...]
                                                  [--git.username=...]
NomadSync vault remove --vault=<name|owner/name>
NomadSync vault list
NomadSync vault show   --vault=<name|owner/name>
```

### Comportamento dettagliato

**`vault add`**:

```
// 1. Valida --owner, --name, --path (tutti obbligatori)
// 2. Verifica path esiste su filesystem
// 3. Verifica path contiene .git/ (deve essere un repo Git inizializzato)
// 4. Chiama vaultService.load() per verificare unicità repoSlug e path
// 5. Chiama vaultService.create(owner, name, path)
// 6. Applica eventuali --git.* flags al vault appena creato
// 7. Chiama gitService.bootstrapVault(vault)
// 8. Stampa: "Vault AleDeP10/public-vault added successfully."
```

**`vault update`**:

```
// 1. Risolve --vault (stessa logica M7)
// 2. Crea copia del vault con i campi aggiornati dai flags presenti
//    (solo i flags esplicitamente passati vengono modificati)
// 3. Se owner o name cambia → repoSlug cambia → bootstrapVault obbligatorio
// 4. Applica --git.* flags (credenziali) se presenti
// 5. Chiama vaultService.update(vault)
// 6. SE repoSlug o credenziali cambiano → chiama gitService.bootstrapVault(vault)
// 7. Stampa: "Vault AleDeP10/public-vault updated."
//
// NOTA: nessun flag = no-op con avviso "Nothing to update."
```

**`vault remove`**:

```
// 1. Risolve --vault
// 2. Stampa: "Remove AleDeP10/public-vault from vaults.json? [y/N]: "
//    (default N — operazione distruttiva)
// 3. Se confermato: vaultService.delete(vault.getId())
// 4. NON cancella la cartella locale
// 5. Stampa: "Vault AleDeP10/public-vault removed."
```

**`vault list`**:

```
// Output tabellare su System.out:
// VAULT                       PATH                          BRANCH  REMOTE  TOKEN
// AleDeP10/public-vault       C:\Users\aless\vaults\public  main    origin  ✓
// AleDeP10/private-vault      C:\Users\aless\vaults\private main    origin  ✗
```

**`vault show`**:

```
// Output dettagliato per un vault specifico:
// Vault:    AleDeP10/public-vault
// Path:     C:\Users\aless\vaults\public
// Branch:   main
// Remote:   origin
// Token:    ***
// Name:     Alessandro De Prato
// Email:    ale@example.com
//
// Status (git status --short, max 3 lines):
//  M notes/todo.md
//  M projects/nomadsync.md
//  ? new-file.md
//  ... (8 more changes)
```

### Dispatch in `Main`

```
case "vault":
  // args[1] = sottocomando
  switch (args[1]) {
      case "add"    -> handleVaultAdd(flags, vaultService, gitService, logService)
      case "update" -> handleVaultUpdate(flags, vaultService, gitService, logService)
      case "remove" -> handleVaultRemove(flags, vaultService, logService)
      case "list"   -> handleVaultList(vaultService)
      case "show"   -> handleVaultShow(flags, vaultService, gitService, logService)
      default       -> errore "Unknown vault subcommand: " + args[1]
  }
  // early-exit, no orchestratori
```

### Script

```
NomadSyncVault.bat         → NomadSync vault %*         (entry point unificato)
NomadSyncVaultAdd.bat      → NomadSync vault add %*
NomadSyncVaultUpdate.bat   → NomadSync vault update %*
NomadSyncVaultRemove.bat   → NomadSync vault remove %*
NomadSyncVaultList.bat     → NomadSync vault list %*
NomadSyncVaultShow.bat     → NomadSync vault show %*
```

Analoghi `.sh` per ciascuno.

**[COMMIT A]** `feat(vault-management): NomadSync vault add/update/remove/list/show`

---

## Sprint B — `NomadSync setup`: wizard console

### Flusso completo

```
=== NomadSync Setup ===

Git identity
  Name:            _
  Email:           _
  GitHub username: _
  GitHub token:    (input mascherato, non appare sullo schermo)

First vault
  Local path:                _
  GitHub repository (owner/name): _

[Windows]  Register Task Scheduler tasks (logon pull + logoff push)? [Y/n]: _
[macOS]    Register launchd agent (pull at login, push at logout)? [Y/n]: _

Setup complete.
Run NomadSyncPull to sync your first vault now.
```

**Regole**:

- Solo il sistema operativo corrente viene mostrato nella sezione "Register" — rilevato via `OsUtil.detect()`
- Token: `System.console().readPassword()` — mai echoed; fallback a `Scanner(System.in)` se `console == null` (es. IDE)
- Nessun default pre-compilato con dati personali
- Path vault: validato in tempo reale (esiste? contiene `.git/`?)
- `owner/name`: validato come formato (deve contenere `/`)
- Campo vuoto → skip (setup parziale completabile con `vault update` / `config` in seguito)

### Implementazione

`NomadSync setup` è early-exit. Chiama in sequenza:

1. `handleConfig` per i `git.*` globali
2. `handleVaultAdd` per il primo vault
3. `handleOsRegistration` per Task Scheduler / launchd

`Main` rileva il primo avvio verificando l'assenza di `config.properties` e lancia `setup` automaticamente:

```java
// All'inizio di main(), prima del parsing flags:
if (!new File(configPath).exists() && !"setup".equals(operation)) {
    logService.info("First run detected — launching setup wizard.");
    handleSetup(...);
    System.exit(0);
}
```

**[COMMIT B]** `feat(setup): NomadSync setup wizard console`

---

## Sprint C — Registrazione OS automatica

### Windows — Task Scheduler

```java
// Due task registrati via schtasks.exe
// Richiedono privilegi elevati — l'installer gira già come admin

// Pull at logon
new ProcessBuilder("schtasks", "/create",
    "/tn", "NomadSync\\Pull at logon",
    "/tr", "\"" + installDir + "\\NomadSyncPull.bat\"",
    "/sc", "ONLOGON",
    "/ru", System.getProperty("user.name"),
    "/f")

// Push at logoff
new ProcessBuilder("schtasks", "/create",
    "/tn", "NomadSync\\Push at logoff",
    "/tr", "\"" + installDir + "\\NomadSyncPush.bat\"",
    "/sc", "ONLOGONEXIT",
    "/ru", System.getProperty("user.name"),
    "/f")
```

Verifica: `schtasks /query /tn "NomadSync\Pull at logon"` → exit code 0 = registrato. Rimozione (uninstaller): `schtasks /delete /tn "NomadSync\..." /f`

### macOS — launchd user agent

Il wizard scrive `~/Library/LaunchAgents/io.aledep10.nomadsync.plist`:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN"
    "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key>
    <string>io.aledep10.nomadsync</string>
    <key>ProgramArguments</key>
    <array>
        <!-- pull --daemon: esegue pull al login, resta vivo per il push al logout -->
        <!-- NON sync --daemon: push automatico in startup non accettabile -->
        <string>/Applications/NomadSync/NomadSync.sh</string>
        <string>pull</string>
        <string>--daemon</string>
    </array>
    <key>RunAtLoad</key>
    <true/>
    <key>KeepAlive</key>
    <false/>
    <key>StandardOutPath</key>
    <string>/tmp/nomadsync.log</string>
    <key>StandardErrorPath</key>
    <string>/tmp/nomadsync-error.log</string>
</dict>
</plist>
```

Attivazione: `launchctl load ~/Library/LaunchAgents/io.aledep10.nomadsync.plist` Verifica: `launchctl list | grep nomadsync` Rimozione: `launchctl unload ...` + cancellazione plist

**[COMMIT C]** `feat(os-registration): Task Scheduler + launchd automation`

---

## Sprint D — Diagnostica: `log`, `version`, `doctor`

### `NomadSync version`

```
NomadSync 1.0.0
Java 21.0.x (Oracle OpenJDK)
OS: Windows 11 / macOS 14.x
```

Versione letta da `MANIFEST.MF` — iniettata da Maven durante il build via `maven-jar-plugin` con `<addDefaultImplementationEntries>true</addDefaultImplementationEntries>`.

### `NomadSync log`

```
NomadSync log [--lines=50] [--vault=<name|owner/name>] [--level=INFO]
```

Legge `log.path` da `config.properties`, stampa le ultime N righe filtrate per vault e/o livello su `System.out`. Early-exit.

### `NomadSync doctor`

```
NomadSync Doctor
────────────────────────────────────────────────────────
✓ config.properties        ./config.properties
✓ vaults.json              2 vault registrati
✓ git executable           git 2.44.0
✓ AleDeP10/public-vault
    path                   C:\Users\aless\vaults\public (exists)
    remote                 reachable
    token                  configured
✗ AleDeP10/private-vault
    path                   C:\Users\aless\vaults\private (NOT FOUND)
────────────────────────────────────────────────────────
Windows Task Scheduler
✓ NomadSync\Pull at logon  registered
✓ NomadSync\Push at logoff registered
────────────────────────────────────────────────────────
[macOS only] Last shutdown push: completed / INCOMPLETE (see log)
────────────────────────────────────────────────────────
1 issue found. Run NomadSync doctor --fix to attempt auto-repair.
```

`--fix`: ri-registra task OS mancanti, ri-esegue `bootstrapVault` per vault con token presente, suggerisce `vault update --path=...` per path non trovati.

**[COMMIT D]** `feat(diagnostics): NomadSync log/version/doctor`

---

## Sprint E — jpackage installer

### Configurazione Maven

```xml
<plugin>
    <groupId>org.panteleyev</groupId>
    <artifactId>jpackage-maven-plugin</artifactId>
    <configuration>
        <name>NomadSync</name>
        <appVersion>1.0.0</appVersion>
        <vendor>Belmani-Apex</vendor>
        <input>target/</input>
        <mainJar>NomadSync-1.0.0-jar-with-dependencies.jar</mainJar>
        <mainClass>io.aledep10.nomadsync.Main</mainClass>
        <!-- Windows: niente directory chooser — path standard Program Files -->
        <winShortcut>true</winShortcut>
        <winMenu>true</winMenu>
        <!-- macOS -->
        <macPackageIdentifier>io.aledep10.nomadsync</macPackageIdentifier>
    </configuration>
</plugin>
```

### Comportamento

L'utente scarica da Softonic, esegue l'installer (doppio click), NomadSync viene installato in `Program Files\NomadSync` / `/Applications/NomadSync`. Al primo avvio si apre un terminale con il wizard di setup. Al termine, NomadSync è operativo.

Nessun directory chooser — destinazione fissa standard per entrambi i sistemi. L'utente avanzato può sovrascrivere da riga di comando se necessario.

### Output

- `NomadSync-1.0.0.exe` — installer Windows, UAC, menu Start shortcut
- `NomadSync-1.0.0.pkg` — installer macOS; v1.0.0 non firmato (documentato nel README); v2.0.0 firmato con certificato Apple Developer Belmani-Apex (nessun bypass richiesto agli utenti finali)

**[COMMIT E]** `build(jpackage): installer Windows + macOS via jpackage`

---

## Script prodotti in M8

Tutti in cartella unica — prefisso come unico criterio organizzativo.

|Script|Operazione|
|---|---|
|`NomadSyncVault.bat/.sh`|entry point unificato vault|
|`NomadSyncVaultAdd.bat/.sh`|vault add|
|`NomadSyncVaultUpdate.bat/.sh`|vault update|
|`NomadSyncVaultRemove.bat/.sh`|vault remove|
|`NomadSyncVaultList.bat/.sh`|vault list|
|`NomadSyncVaultShow.bat/.sh`|vault show|
|`NomadSyncSetup.bat/.sh`|setup wizard|
|`NomadSyncDoctor.bat/.sh`|doctor|
|`NomadSyncLog.bat/.sh`|log tail|
|`NomadSyncVersion.bat/.sh`|version|

---

## Rischi

|Rischio|Probabilità|Impatto|Mitigazione|
|---|---|---|---|
|macOS SIGTERM timeout < tempo push su rete lenta|Bassa|Alto|Warning nel log; `doctor` riporta esito al prossimo avvio|
|`schtasks` trigger `ONLOGONEXIT` non disponibile su Windows Home|Media|Alto|Verificare durante e2e; fallback a `ONLOGON` con task di push manuale|
|`System.console()` null in ambienti senza terminale|Media|Basso|Fallback a `Scanner(System.in)`|
|jpackage bundled JRE aumenta dimensione installer (~50MB)|Certa|Basso|Accettabile; documentare|
|macOS Gatekeeper blocca `.pkg` non firmato|Alta|Medio|Documentare `xattr -cr` nel README; v2.0.0 con certificato|
|`vault update` con cambio owner/name richiede remote URL aggiornato|Certa|Basso|`bootstrapVault` ri-eseguito automaticamente se repoSlug cambia|

---

## Note implementative

**Ordine obbligatorio**: A → B → C → D → E.

**`NomadSync setup`** non ha uno script dedicato — è il comportamento di default al primo avvio. Richiamabile esplicitamente con `NomadSync.bat setup` per ri-configurare, e con `NomadSyncSetup.bat` come shortcut.

**README pre-v1.0.0**: documentare chiaramente la dicotomia `vault update` (per-vault) vs `NomadSync config` (globale).

**To-do e2e M8**:

- [ ] Verificare `ONLOGONEXIT` disponibile su Windows Home
- [ ] Testare launchd plist su macOS (Gabriela)
- [ ] Verificare `xattr -cr` workaround Gatekeeper
- [ ] Testare `System.console()` fallback da launcher jpackage