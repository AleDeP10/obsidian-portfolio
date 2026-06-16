# GRM — Milestone 7: CLI Shortcuts + Vault Identity Refactoring

---

## Contesto

Durante la progettazione di `NomadSyncCommit` — il primo comando CLI single-vault-mandatory, interattivo, non broadcastabile — sono emerse debolezze strutturali da correggere prima dell'e2e e della v1.0.0:

1. **DTR-046 vincola il campo sbagliato.** Il vincolo di unicità è su `Vault.name`, ma l'identità canonica è `repoSlug` (`owner/name`). Due vault con stesso `name` e `owner` diversi sono repository GitHub distinti e legittimi.
    
2. **Unicità del path non verificata.** Due vault che puntano allo stesso path fisico produrrebbero comportamenti indefiniti.
    
3. **`GitService` riceve `String vaultPath`, non `Vault`.** Non ha accesso a `repoSlug` per il naming snapshot/conflict, né alle credenziali per-vault. DTR-031 open item non chiuso.
    
4. **Credenziali Git per-vault non implementate.** Gli e2e multi-owner fallirebbero silenziosamente con le credenziali globali dell'utente.
    
5. **CLI usa UUID come identificatore vault.** Inutilizzabile sia per developer che per utente finale.
    
6. **Argomenti CLI posizionali.** Fragili, non estensibili, ambigui con più parametri opzionali.
    

Questa milestone risolve tutto e aggiunge: script CLI per-evento, `COMMIT_MANUAL` con editor interattivo, sintassi a flag, operazione `config` per la gestione di proprietà globali e per-vault, e una layer di configurazione tipizzata (`NomadProperties`/`NomadPropertiesLoader`) analoga a ForgeUI.

---

## Gerarchia delle configurazioni

```
config.properties             → valori di default per tutti i vault
                                (git.executable, git.remote, git.branch,
                                 git.name, git.email, git.username, git.token)

vaults.json (per-vault)       → sovrascrittura opzionale per singolo vault
                                (gitName, gitEmail, gitUsername, gitToken,
                                 gitBranch, gitRemote)
                                Utile per: vault di altri owner, repo con
                                branch "master" invece di "main", remote
                                non-standard, token dedicati.

Risoluzione: vaults.json > config.properties > ~/.gitconfig
```

`vaults.json` non committato — solo `vaults.json.template`. Rimozione dal tracking con `git rm --cached vaults.json`.

---

## Operazione `config` — panoramica

`NomadSync config` è un'operazione di gestione della configurazione, non di sincronizzazione. Non pubblica eventi, non avvia orchestratori. Agisce a due livelli:

**Livello globale** (senza `--vault`): modifica `config.properties`. Le proprietà aggiornate si applicano a tutti i vault che non le sovrascrivono in `vaults.json`. Utile per impostare il nome Git globale, l'email, o il token di default.

**Livello vault** (con `--vault=<repoSlug|name>`): modifica il vault corrispondente in `vaults.json`. Le proprietà specificate sovrascrivono i valori globali solo per quel vault. Utile per impostare un token dedicato, cambiare il branch di default (`git.branch=master`), o usare un remote non-standard (`git.remote=upstream`).

In entrambi i casi, le modifiche vengono applicate immediatamente: `bootstrapVault` viene rieseguito per il vault interessato, così le nuove credenziali entrano in vigore senza riavviare il processo.

```bash
# Aggiorna git.name globale (config.properties)
NomadSync.bat config --git.name="Alessandro De Prato"

# Aggiorna token per un vault specifico (vaults.json)
NomadSync.bat config --vault=AleDeP10/public-vault --git.token=ghp_...

# Cambia branch di default per un vault legacy con "master"
NomadSync.bat config --vault=AleDeP10/legacy-vault --git.branch=master
```

---

## Obiettivi

|#|Obiettivo|Criterio di accettazione|
|---|---|---|
|A|`VaultService` — unicità su `repoSlug` e `path`|`create/update/load` validano repoSlug e path; stesso-name/owner-diverso accettato|
|B|`GitService` — Vault-aware, credenziali per-vault|tutte le firme ricevono `Vault`; `bootstrapVault` scrive `.git/config` locale|
|C|`SyncOrchestrator` — riceve `Vault` invece di `Properties`|costruttore `(Vault, GitService, LogService, SyncEventQueue, NotificationHook)`|
|D|CLI a flag + `COMMIT_MANUAL` + `NomadSync config`|`--vault`, `--config`; editor interattivo; aggiornamento proprietà globali e per-vault|
|E|`NomadProperties` / `NomadPropertiesLoader`|layer tipizzato analogo a ForgeUI, elimina magic string da `Main`|
|F|`OsUtil` — rilevamento OS|enum `Os { WINDOWS, UNIX }`; usato per fallback editor e futuri branch OS-specific|

---

## Sprint A — VaultService: identità e vincoli

### A1 — Metodi query

**`findByRepoSlug(String repoSlug) → Optional<Vault>`** — nuovo

```java
// return vaults.values().stream()
//         .filter(v -> v.getRepoSlug().equals(repoSlug))
//         .findFirst();
```

**`findAllByName(String name) → List<Vault>`** — sostituisce `findByName`

```java
// return vaults.values().stream()
//         .filter(v -> v.getName().equals(name))
//         .collect(Collectors.toList());
// NOTA: restituisce lista — il chiamante decide in base a .size():
//   0 → not found, 1 → risolto univocamente, >1 → ambiguo
```

**`findByName` → ELIMINATO** (non deprecato).

### A2 — Validazione al load

**`validateUniqueRepoSlugs(List<Vault>)`** — sostituisce `validateUniqueNames`

```java
// Set<String> seen = new HashSet<>();
// for (Vault vault : loaded) {
//     if (!seen.add(vault.getRepoSlug()))
//         throw new VaultException(
//             "duplicated repoSlug in vaults.json: " + vault.getRepoSlug());
// }
```

**`validateUniquePaths(List<Vault>)`** — nuovo

```java
// Set<String> seen = new HashSet<>();
// for (Vault vault : loaded) {
//     if (!seen.add(vault.getPath()))
//         throw new VaultException(
//             "duplicated path in vaults.json: " + vault.getPath());
// }
```

`load()` chiama entrambe prima di popolare la map.

### A3 — create/update: check su repoSlug

**`create()`** — da `findByName(name).isPresent()` a `findByRepoSlug(owner+"/"+name).isPresent()`

**`update()`** — da `findByName(vault.getName())` a `findByRepoSlug(vault.getRepoSlug())`

Messaggio eccezione: `"duplicated repoSlug: " + repoSlug`.

### A4 — Javadoc VaultService

Aggiornare `<h2>Vault name uniqueness</h2>` → vincolo su repoSlug + path.

### A5 — VaultServiceTest (linee guida)

Nuovi/modificati:

- `load_duplicateRepoSlugsInFile_throwsVaultException`
- `load_duplicatePathsInFile_throwsVaultException` ← NUOVO
- `create_sameNameDifferentOwner_doesNotThrow` ← invertito da `throwsVaultException`
- `create_sameRepoSlug_throwsVaultException` ← NUOVO (stesso owner+name)
- `update_renameToAnotherVaultsRepoSlug_throwsVaultException`
- `update_changeOwner_toExistingRepoSlug_throwsVaultException` ← NUOVO

**[COMMIT A]** `fix(vault-identity): enforce repoSlug+path uniqueness in VaultService`

---

## Sprint B — GitService: Vault-aware e credenziali per-vault

### B0 — `StringUtil.coalesce` (prerequisito)

Aggiungere a `StringUtil` prima di qualsiasi altra modifica in questo sprint — `bootstrapVault` e la risoluzione credenziali dipendono da esso.

````java
// /**
//  * Returns the first non-null value in the given sequence,
//  * or {@code null} if all values are null.
//  *
//  * <p>Intended for credential resolution chains where vault-level
//  * values take precedence over global config, which in turn takes
//  * precedence over the system Git configuration:
//  * <pre>
//  *   StringUtil.coalesce(vault.getGitToken(),
//  *                        properties.getProperty("git.token"))
//  * </pre>
//  *
//  * @param values the values to evaluate in order
//  * @return the first non-null value, or {@code null}
//  */
// public static String coalesce(String... values) {
//     for (String v : values) {
//         if (v != null) return v;
//     }
//     return null;
// }

### B1 — Domain model: nuovi campi su `Vault`

Aggiungere `gitBranch` e `gitRemote` opzionali a `Vault`:

```java
// private String gitBranch;   // es. "master" per repo legacy
// private String gitRemote;   // es. "upstream" per remote non-standard
// getter/setter + aggiornamento VaultDto per serializzazione
````

### B2 — Firme pubbliche: `String vaultPath` → `Vault vault`

Tutti i metodi pubblici di `GitService`: `push`, `pull`, `stash`, `stashPop`, `commitLocal`, `hasChanges`, `hasUncommittedChanges`, `synchronize`.

Internamente: `vault.getPath()` dove prima c'era `vaultPath`; `vault.getGitRemote()` / `vault.getGitBranch()` con fallback a `properties.getProperty("git.remote")` / `properties.getProperty("git.branch")`.

### B3 — `bootstrapVault(Vault vault)`

Nuovo metodo pubblico. Chiamato da `Main` all'avvio per ogni vault e ri-chiamato ogni volta che le credenziali cambiano (operazione `config`).

```java
// String path = vault.getPath();
//
// // user.name / user.email
// String name  = StringUtil.coalesce(vault.getGitName(),
//                                    properties.getProperty("git.name"));
// String email = StringUtil.coalesce(vault.getGitEmail(),
//                                    properties.getProperty("git.email"));
// SE name  != null → git -C path config user.name  "<name>"
// SE email != null → git -C path config user.email "<email>"
//
// // remote URL con token embedded (cross-platform, nessuna dipendenza da shell)
// String token    = StringUtil.coalesce(vault.getGitToken(),
//                                       properties.getProperty("git.token"));
// String username = StringUtil.coalesce(vault.getGitUsername(),
//                                       properties.getProperty("git.username"));
// String remote   = StringUtil.coalesce(vault.getGitRemote(),
//                                       properties.getProperty("git.remote", "origin"));
// SE token != null:
//   git -C path remote set-url <remote>
//       https://<username>@github.com/<vault.owner>/<vault.name>
//   [NOTA] token nell'URL è cross-platform; git non chiede credenziali aggiuntive
//   [NOTA] git remote -v mostrerà l'URL con token — accettabile per tool desktop
//          personale; documentare nel README
```

`StringUtil.coalesce` (già in `StringUtil`):

```java
// public static String coalesce(String... values) {
//     for (String v : values) if (v != null) return v;
//     return null;
// }
```

### B4 — `synchronize(Vault vault)`: naming con repoSlug

```java
// String snapshotPrefix = vault.getRepoSlug().replace("/", "_");
// // "AleDeP10/public-vault" → "AleDeP10_public-vault"
// // usato per makeVaultSnapshot e conflictDirName
```

### B5 — GitServiceTest

Aggiornare firme. Nuovi test:

- `bootstrapVault_setsLocalGitConfigNameAndEmail`
- `bootstrapVault_setsRemoteUrlWithToken`
- `bootstrapVault_vaultValuesOverrideGlobal`
- `bootstrapVault_noCredentials_doesNothing`
- `synchronize_snapshotPrefixUsesRepoSlug`

**[COMMIT B]** `feat(git-vault-aware): GitService receives Vault, bootstrapVault, per-vault credentials`

---

## Sprint C — SyncOrchestrator: riceve Vault

### C1 — Costruttore

```java
// PRIMA: SyncOrchestrator(Properties, GitService, LogService,
//                          SyncEventQueue, NotificationHook)
// DOPO:  SyncOrchestrator(Vault, GitService, LogService,
//                          SyncEventQueue, NotificationHook)
// this.vault = vault;  — vault.getPath() sostituisce vaultPath
// Properties rimosso — non serviva per altro
```

### C2 — execute(): passa Vault a GitService

`gitService.xxx(vaultPath)` → `gitService.xxx(vault)` ovunque.

### C3 — Main: semplificazione wiring

```java
// PRIMA:
// Properties vaultProperties = new Properties(properties);
// vaultProperties.setProperty("vault.path", vault.getPath());
// new SyncOrchestrator(vaultProperties, ...);

// DOPO:
// gitService.bootstrapVault(vault);
// new SyncOrchestrator(vault, ...);
```

Properties derivate per-vault → eliminate.

**[COMMIT C]** `refactor(orchestrator): SyncOrchestrator receives Vault directly`

---

## Sprint D — CLI: flag, COMMIT_MANUAL, operazione config

### D1 — `OsUtil`

```java
// package io.aledep10.nomadsync.util;
// public enum Os { WINDOWS, UNIX }
//
// public final class OsUtil {
//     private OsUtil() {}
//     public static Os detect() {
//         return System.getProperty("os.name", "")
//                      .toLowerCase().contains("win") ? Os.WINDOWS : Os.UNIX;
//     }
//     public static boolean isWindows() { return detect() == Os.WINDOWS; }
// }
```

### D2 — `EventType`: `mandatoryVault` + riordino priorità

```java
PULL_LOGON(1,    false),
SYNCHRONIZE(2,   false),
PUSH_LOGOFF(3,   false),
COMMIT_MANUAL(4, true),    // mandatoryVault=true: --vault obbligatorio
AUTOSAVE(5,      false);

private final int     priority;
private final boolean mandatoryVault;  // rename da "mandatory"
```

### D3 — `SyncEvent`: campo message

```java
// private final String message;  // opzionale, null per tutti tranne COMMIT_MANUAL
// costruttore esistente invariato (message = null)
// nuovo overload: SyncEvent(EventType, String vaultId, String message)
// getMessage() → String
```

### D4 — Main: parsing a flag

`operation` posizionale (`args[0]`). Resto: `--key=value`, ordine libero.

```java
// Map<String, String> flags = new LinkedHashMap<>();
// Arrays.stream(args).skip(1)
//         .filter(a -> a.startsWith("--"))
//         .forEach(arg -> {
//             String[] parts = arg.substring(2).split("=", 2);
//             flags.put(parts[0], parts.length > 1 ? parts[1] : "");
//         });
// String configPath = flags.getOrDefault("config", "./config.properties");
// String vaultFlag  = flags.get("vault");  // null se assente
```

### D5 — Risoluzione `--vault`

```
--vault=<name>  (no slash)
  candidati = vaultService.findAllByName(name)
  size == 0 → errore: "vault '<name>' not found.
               Registered: AleDeP10/public-vault, Belmani/belmani-apex, ..."
  size == 1 → quel vault (indipendentemente dall'owner)
  size >  1 → errore: "vault name '<name>' is ambiguous.
               Matches: AleDeP10/public-vault, Belmani/public-vault.
               Use --vault=<owner>/<name>"

--vault=<owner>/<name>  (con slash)
  findByRepoSlug(vaultFlag) → Optional
  empty → errore: "vault '<owner>/<name>' not found.
                   Registered: <lista repoSlug>"
```

Tutti i messaggi d'errore mostrano **repoSlug completi**.

Dispatch:

```
SE vaultFlag presente → risolvi vault → routeToVault (puntuale)
ALTRIMENTI:
  SE eventType.mandatoryVault → errore "<op> richiede --vault", exit 1
  ALTRIMENTI                  → broadcast (vaultId=null)
```

`resolveTargetId` con fallback-to-first → **ELIMINATO**.

### D6 — COMMIT_MANUAL: editor interattivo

```java
// case "commit":
//   1. vault risolto (D5) — mandatoryVault, errore se assente
//   2. tempFile = Files.createTempFile("nomadsync-commit-", ".txt")
//   3. editorFlag = flags.get("editor")
//   4. editor = StringUtil.coalesce(
//                   editorFlag,
//                   properties.getProperty(NomadProperties.Commit.EDITOR),
//                   System.getenv("EDITOR"),
//                   OsUtil.isWindows() ? "notepad" : "nano")
//   5. new ProcessBuilder(editor, tempFile.toString())
//              .inheritIO().start().waitFor()
//   6. message = Files.readString(tempFile).strip()
//   7. Files.deleteIfExists(tempFile)
//   8. SE message.isEmpty() → log "Empty commit message — aborting", exit 0
//   9. publish COMMIT_MANUAL con message, routing puntuale
```

Documentare nel README: "chiudi l'editor senza salvare per annullare".

### D7 — `NomadSync config`: gestione proprietà

```java
// case "config":
//   credentialFlags = flags.entrySet().stream()
//       .filter(e -> e.getKey().startsWith("git."))
//       .collect(...)
//
//   SE vaultFlag presente:
//     vault = risolvi(vaultFlag)
//     per ogni flag git.*:
//       git.name     → vault.setGitName(value)
//       git.email    → vault.setGitEmail(value)
//       git.username → vault.setGitUsername(value)
//       git.token    → vault.setGitToken(value)
//       git.branch   → vault.setGitBranch(value)
//       git.remote   → vault.setGitRemote(value)
//     vaultService.update(vault)
//     gitService.bootstrapVault(vault)   ← ri-applica immediatamente
//     log "vault <repoSlug> updated"
//
//   SE vaultFlag assente:
//     leggi config.properties esistente
//     per ogni flag git.*: aggiorna chiave corrispondente
//     scrivi config.properties (Properties.store())
//     [NOTA] i commenti del file vengono persi — accettabile per v1.0.0;
//            documentare nel README
//     log "global config updated"
```

Script: `NomadSyncConfig.bat` / `.sh` — wrapper triviale (`call NomadSync.bat config %*`).

**[COMMIT D]** `feat(cli-shortcuts): flag CLI, COMMIT_MANUAL, NomadSync config, OsUtil`

---

## Sprint E — NomadProperties / NomadPropertiesLoader

Pattern identico a `ForgePropertiesLoader`/`ForgeProperties` (allegato al grooming come riferimento).

### E1 — `NomadPropertiesLoader`

```java
// package io.aledep10.nomadsync.config;
// Stesso pattern di ForgePropertiesLoader:
//   load() → Properties (mai null, mai throw)
//   get(key) → String|null
//   get(key, defaultValue) → String
//   getBoolean(key, defaultValue) → boolean
//   getEnum(key, enumType, defaultValue) → E
```

### E2 — `NomadProperties`

```java
// package io.aledep10.nomadsync.config;
// Costanti per tutte le chiavi di config.properties:
//
// public static final class Git {
//     public static final String EXECUTABLE = "git.executable";
//     public static final String REMOTE     = "git.remote";
//     public static final String BRANCH     = "git.branch";
//     public static final String NAME       = "git.name";
//     public static final String EMAIL      = "git.email";
//     public static final String USERNAME   = "git.username";
//     public static final String TOKEN      = "git.token";
// }
//
// public static final class Path {
//     public static final String VAULTS    = "path.vaults";
//     public static final String BACKUP    = "path.backup";
//     public static final String CONFLICTS = "path.conflicts";
// }
//
// public static final class Log {
//     public static final String WRITERS   = "log.writers";
//     public static final String PATH      = "log.path";
//     public static final String LEVEL     = "log.level";
//     public static final String SEQ_URL   = "log.seq.url";
// }
//
// public static final class Autosave {
//     public static final String INTERVAL_MINUTES = "autosave.interval.minutes";
// }
//
// public static final class Commit {
//     public static final String EDITOR = "commit.editor";
// }
//
// public static final class Socket {
//     public static final String HOST         = "socket.host";
//     public static final String PORT         = "socket.port";
//     public static final String RETRY_DELAY  = "socket.retryDelay";
// }
```

### E3 — Migrazione magic string

`Main`, `LogService`, `VaultService`, `GitService`, `AutosaveScheduler`, `SeqHttpLogWriter` — sostituire tutti i `properties.getProperty("key.string")` con `NomadProperties.<Class>.KEY`.

**[COMMIT E]** `refactor(config): NomadProperties+NomadPropertiesLoader, eliminate magic strings`

---

## To-do da verificare ad inizio e2e

- [ ] Testare `commit.editor` con `notepad`, `notepad++`, `nano` su rispettive piattaforme — verificare che il processo padrebloccherà correttamente fino alla chiusura dell'editor

---

## Rischi

|Rischio|Probabilità|Impatto|Mitigazione|
|---|---|---|---|
|Token embedded nel remote URL visibile in `git remote -v`|Alta|Medio|Accettabile per tool desktop personale; documentare nel README|
|`notepad` su Windows potrebbe non bloccare il padre se già aperto|Bassa|Medio|Testare durante e2e; item nel to-do|
|`git rm --cached vaults.json` richiede coordinamento con tutti i collaboratori per il re-clone o `git pull`|Bassa|Basso|Comunicare a Gabriela prima del merge|
|`Properties.store()` perde i commenti da `config.properties`|Alta|Basso|Documentare nel README; accettabile per v1.0.0|
|`findAllByName` e `findByRepoSlug` coesistono — rischio uso sbagliato|Media|Medio|Javadoc esplicito: `findAllByName` solo per ambiguità CLI|

---

## Note implementative

**Ordine obbligatorio**: A → B → C → D → E. Ogni sprint = un commit con suite verde.

**`vaults.json` dal tracking**:

```bash
git rm --cached vaults.json
echo "vaults.json" >> .gitignore
git add .gitignore
git commit -m "chore: remove vaults.json from tracking"
```

Comunicare a Gabriela: fare `git pull` e ricreare `vaults.json` da template.