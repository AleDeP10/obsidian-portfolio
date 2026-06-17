# NomadSync — README Developer

## Panoramica dell'architettura

```
Main
 ├── VaultService          carica vaults.json, CRUD, snapshot, conflitti
 ├── GitService            git CLI via ProcessBuilder, bootstrapVault
 ├── LogService            fan-out multi-writer, scoping per vault
 │    ├── ConsoleLogWriter
 │    ├── FileLogWriter
 │    └── SeqHttpLogWriter (asincrono, BlockingQueue + daemon thread)
 ├── broadcast SyncEventQueue
 ├── broadcaster Thread    smista vaultId=null a tutte le code
 ├── AutosaveScheduler     AUTOSAVE periodico sulla coda broadcast
 └── per vault:
      ├── SyncEventQueue   coda a priorità, deduplicazione latest-wins
      └── SyncOrchestrator worker thread, retry con exponential backoff
```

---

## Decisioni chiave (sintesi — vedi DTR per il dettaglio completo)

| DTR | Decisione |
|-----|-----------|
| DTR-011 | Event-driven, coda a priorità. Git è seriale per repository. |
| DTR-014 | Deduplicazione latest-wins sullo stesso tipo di evento. |
| DTR-015 | Exponential backoff: 30s→60s→120s, max 3 tentativi. |
| DTR-026 | SYNCHRONIZE: `-X ours`, backup FIFO, remote-conflicts. |
| DTR-031 | `repoSlug = owner/name` come identificatore universale del vault. |
| DTR-039 | Coda broadcast: `vaultId=null` fa fan-out su tutte le code per-vault. |
| DTR-046 | Unicità su `repoSlug` E `path`, non solo su `name`. |
| DTR-048 | Flag `mandatoryVault` su `EventType`: broadcast vs. errore. |
| DTR-049 | Token mai loggato: mascheramento via `CommandUtil.sensitiveArgs`. |
| DTR-050 | Flag `--daemon`: one-shot (default) vs. long-running (Tray). |

---

## Struttura dei package

```
io.aledep10.nomadsync
 ├── Main.java
 ├── config/
 │    ├── NomadProperties.java         registro delle chiavi
 │    └── NomadPropertiesLoader.java   loader da classpath
 ├── dto/
 ├── exception/
 ├── gitignore/
 ├── hook/
 ├── logging/
 │    ├── LogLevel.java, LogWriter.java, LogFormatter.java
 │    ├── LineFormatter.java, ClefFormatter.java
 │    ├── ConsoleLogWriter.java, FileLogWriter.java
 │    ├── InMemoryLogWriter.java, SeqHttpLogWriter.java
 ├── orchestrator/
 │    ├── Vault.java, VaultContext.java (record)
 │    ├── EventType.java  (mandatoryVault, priority)
 │    ├── SyncEvent.java  (message, forVault())
 │    ├── SyncEventQueue.java  (priority, dedup, isEmpty())
 │    └── SyncOrchestrator.java  (worker thread, retry)
 ├── scheduler/
 │    └── AutosaveScheduler.java
 ├── service/
 │    ├── GitService.java  (bootstrapVault, status())
 │    ├── GitignoreService.java
 │    ├── LogService.java
 │    └── VaultService.java
 ├── tray/
 └── util/
      ├── CommandUtil.java  (sensitiveArgs, runCommandWithLines)
      ├── Os.java, OsUtil.java
      ├── StringUtil.java  (coalesce())
      └── ValidationUtil.java
```

---

## Build

```bash
mvn clean package          # compile + test + fat JAR
mvn test                   # solo test
mvn test -DrunOrder=random # ordine casuale (rilevazione regressioni)
```

Output: `target/NomadSync-1.0-SNAPSHOT-jar-with-dependencies.jar`

---

## Convenzioni di test

- **Unit test**: mock `GitService`, `NotificationHook`. `SyncEventQueue` reale.
- **Integration test** (`GitServiceTest`): repository Git reale in directory
  temporanea. `@BeforeEach` esegue `git init` + `git config user.*`;
  `@AfterEach` cancella.
- **Costruzione Vault nei test**: usa `new Vault(uuid, owner, name, path)` come
  copia — non mutare mai l'istanza live restituita da `create()`, è la stessa
  referenza memorizzata nella `HashMap` interna.
- **Ordine casuale**: `-DrunOrder=random` via Maven Surefire. Ogni test deve
  essere indipendente — nessuno stato mutabile condiviso tra test.
- **Costruttori package-private**: `SyncEvent(type, vaultId, message, timestamp,
  retryDelay)` e `AutosaveScheduler(queue, log, interval, TimeUnit)` per
  scenari di test deterministici.

---

## Riferimento CLI

```
java -jar NomadSync.jar <operation> [flags...]

Operazioni:
  pull     Pull dal remoto (broadcast se --vault assente)
  push     Push al remoto  (broadcast se --vault assente)
  sync     Sincronizzazione bidirezionale completa (broadcast se --vault assente)
  status   Output di git status su stdout (broadcast se --vault assente)
  commit   Commit locale con messaggio da editor (--vault obbligatorio)
  autosave Periodico — gestito da AutosaveScheduler, non per uso manuale
  config   Aggiorna config.properties o vaults.json (early-exit)

Flag:
  --config=<path>           percorso config.properties (default: ./config.properties)
  --vault=<name|owner/name> vault target (assente = broadcast per op. non-mandatory)
  --daemon                  mantieni processo vivo (modalità Tray)
  --editor=<path>           editor per il messaggio di commit
  --git.<key>=<value>       flag credenziali/configurazione per l'operazione 'config'
```

---

## Aggiungere un nuovo EventType

1. Aggiungi costante a `EventType` con priorità e valore `mandatoryVault`
2. Aggiungi `case` in `SyncOrchestrator.execute()`
3. Aggiungi `case` nel dispatch switch di `Main.main()`
4. Aggiungi `case` in `Main.operationToEventType()`
5. Crea script `NomadSync<Operazione>.bat/.sh`
6. Aggiungi test in `SyncOrchestratorTest`

---

## Sicurezza del token

Il token GitHub:
- È memorizzato nel `.git/config` locale del vault via `git remote set-url`
- Non viene mai passato come argomento di comando a nessun processo Git
- Non viene mai scritto in nessun log (mascherato come `<hidden>` via
  `CommandUtil.sensitiveArgs`)
- Non viene mai committato (`.git/` è escluso dal tracking per design Git)
- `config.properties` è in `.gitignore` — mai committato

---

## Strategia di branching

Git Flow AVH Edition (incluso in Git for Windows):

```
main      → solo release taggate
develop   → integrazione continua degli sprint
feature/* → un branch per obiettivo di grooming
release/* → bump versione + fix finali
hotfix/*  → fix urgenti su main, merged su develop
```
