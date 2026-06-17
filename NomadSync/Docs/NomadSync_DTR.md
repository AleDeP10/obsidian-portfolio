# NomadSync — Decision Track Record (Italiano)

> Registro delle decisioni tecniche e architetturali significative prese durante
> il progetto, con motivazione e alternative valutate. Aggiornato all'evolversi
> del progetto.

---

## DTR-001 — Sincronizzazione via Git invece di cloud storage

**Milestone**: M1 | **Stato**: Accettata

**Contesto**: le cartelle devono restare sincronizzate su più macchine. I servizi
di cloud sync sono incompatibili con applicazioni che tengono file aperti.

**Decisione**: Git + GitHub.

**Motivazione**: le operazioni pull/push sono atomiche e sincrone; il diff nativo
consente autosave differenziale; nessun processo esterno tocca i file mentre le
applicazioni sono aperte.

**Alternative scartate**: OneDrive (conflitti per scritture concorrenti); Syncthing
(P2P, richiede almeno un device sempre acceso); Obsidian Sync (a pagamento ~4$/mese).

---

## DTR-002 — Strategia `theirs` per la risoluzione dei conflitti in pull

**Milestone**: M1 | **Stato**: Superata da DTR-013 (M5)

**Decisione**: `git pull -X theirs` — in caso di conflitto vince sempre la versione remota.

**Rischi accettati**: modifiche locali non committate prima del push potrebbero
essere perse. Mitigato da autosave periodico e `git stash` prima del pull.

---

## DTR-003 — Sequenza logon: stash → pull → stash pop

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `git stash` prima del pull e `git stash pop` dopo.

**Motivazione**: preserva le modifiche locali senza richiedere un commit esplicito.
**Alternativa scartata**: `git reset --hard` — distrugge le modifiche locali senza
possibilità di recupero.

---

## DTR-004 — Aggancio logon/logoff tramite Task Scheduler

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: Task Scheduler (`taskschd.msc`). Disponibile su tutte le edizioni
di Windows inclusa Home; interfaccia ispezionabile; cronologia esecuzioni; export
XML per replicabilità.

**Alternativa scartata**: Group Policy Editor — non disponibile su Windows Home.

---

## DTR-005 — Autosave differenziale tramite `git diff --quiet`

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `git diff --quiet` come guard — exit code 0 = nessuna modifica,
exit code 1 = modifiche presenti. Previene commit vuoti.

---

## DTR-006 — Fat JAR tramite `maven-assembly-plugin`

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `maven-assembly-plugin` con descriptor `jar-with-dependencies`.
Singolo artefatto autonomo; deploy copiando la cartella `target/`.

---

## DTR-007 — Risorse copiate in `target/` tramite `maven-resources-plugin`

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `maven-resources-plugin` copia da `src/main/resources/` a `target/`
durante la fase `package`. Deploy autocontenuto in `target/`.

---

## DTR-008 — Due file di configurazione separati per ambiente

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `config.dev.properties`/`config.prod.properties` esclusi da
`.gitignore`; `config.properties.template` committato come riferimento.
`vaults.json` anch'esso escluso — solo `vaults.json.template` committato.

---

## DTR-009 — Logging thread-safe tramite `synchronized`

**Milestone**: M1 | **Stato**: Superata da DTR-034 (M5)

**Decisione**: metodo `LogService.log()` dichiarato `synchronized`. Semplice e
corretto per al massimo due thread.

---

## DTR-010 — `SyncOrchestrator` come layer intermedio

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `SyncOrchestrator` tra `Main` e `GitService`. La logica di
coordinamento (stash prima del pull, gestione exit code) non appartiene né all'uno
né all'altro.

---

## DTR-011 — Architettura event-driven per SyncOrchestrator

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: coda a priorità. I chiamanti pubblicano eventi; l'orchestratore
consuma serialmente. Git è seriale — operazioni concorrenti sullo stesso repository
causano conflitti.

**Alternativa scartata**: chiamate dirette — la gestione della concorrenza si
distribuisce nel codice.

---

## DTR-012 — Scala di priorità degli eventi (originale)

**Milestone**: M2 | **Stato**: Superata da DTR-013 (M5)

Scala originale: PULL_LOGON(1), PUSH_MANUAL(2), PUSH_LOGOFF(3), AUTOSAVE(4).

---

## DTR-013 — SYNCHRONIZE sostituisce PULL_MANUAL e PUSH_MANUAL

**Milestone**: M5 | **Stato**: Accettata — sostituisce DTR-012

**Decisione**: evento unico `SYNCHRONIZE`. Scala aggiornata in M7:

| Priorità | Evento |
|---|---|
| 1 | `PULL_LOGON` |
| 2 | `SYNCHRONIZE` |
| 3 | `PUSH_LOGOFF` |
| 4 | `COMMIT_MANUAL` |
| 5 | `AUTOSAVE` |

---

## DTR-014 — Strategia di deduplicazione: latest wins

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: un evento in coda viene sostituito dal più recente dello stesso tipo.
Un evento in esecuzione non viene interrotto.

---

## DTR-015 — Retry con exponential backoff

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: massimo 3 tentativi. Delay progressivo: 30s → 60s → 120s. Dopo il
terzo fallimento l'evento viene scartato e viene invocato `NotificationHook.onFailure`.

---

## DTR-016 — Hook di notifica come dependency inversion

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: interfaccia `NotificationHook`. Implementazione di default scrive
su log. La Tray si aggancia implementando la stessa interfaccia senza modificare
l'orchestratore.

---

## DTR-017 — Separazione commit locale / push remoto

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: `AUTOSAVE` → solo commit locale. `PUSH_LOGOFF` / `SYNCHRONIZE` →
commit locale + push remoto. Il retry si applica solo alle operazioni remote.

---

## DTR-018 — Guard `hasUncommittedChanges()` prima di stash/stashPop

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: `git status --porcelain` come guard. `git diff --quiet` non
sufficiente — non rileva le modifiche staged.

---

## DTR-019 — `notify()` rinominato in `onFailure()` in NotificationHook

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: `onFailure(SyncEvent, String message)`. Evita collisione con
`Object.notify()`; `message` comunica la causa del fallimento alla Tray.

---

## DTR-020 — Thread semplice invece di ExecutorService per il worker loop

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: `Thread` semplice istanziato internamente. `ExecutorService` non
aggiunge valore per un singolo worker con loop infinito. Shutdown via `interrupt()`
+ `join()`.

---

## DTR-021 — Shutdown hook registrato in Main

**Milestone**: M2 | **Stato**: Accettata — estesa in M6 (DTR-040), M7 (DTR-050)

**Decisione**: shutdown hook in `Main`. Ordine: `scheduler.stop()` →
`broadcaster.interrupt()` → `orchestrators.forEach(stop)` → `logService.close()`.

---

## DTR-022 — Git Flow come strategia di branching

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: Git Flow AVH Edition. `main` (solo release), `develop`
(integrazione continua), `feature/*` (un branch per obiettivo di grooming),
`release/*`, `hotfix/*`.

---

## DTR-023 — `CommandUtil` come utility condivisa per l'esecuzione di processi

**Milestone**: M3 | **Stato**: Accettata — estesa in M5 (DTR-037), M7 (DTR-049)

**Decisione**: classe statica `CommandUtil` nel package `util`. Parametro
`LogService` opzionale; overload senza per i test helper.

---

## DTR-024 — Costruttori package-private di test su `SyncEvent` e `AutosaveScheduler`

**Milestone**: M3 | **Stato**: Accettata

**Decisione**: costruttore package-private su `SyncEvent` con timestamp
controllato. Costruttore package-private su `AutosaveScheduler` con `TimeUnit`
per intervalli sub-minuto nei test.

---

## DTR-025 — `SyncEventQueue` usata come istanza reale in `SyncOrchestratorTest`

**Milestone**: M3 | **Stato**: Accettata

**Decisione**: istanza reale di `SyncEventQueue`; mock solo `GitService` e
`NotificationHook`. Logica pura senza side effect — candidata ideale all'uso
reale nei test.

---

## DTR-025b — Downgrade JDK da 25 a 21

**Milestone**: M3 | **Stato**: Accettata

**Decisione**: Oracle OpenJDK 21 LTS. Mockito 5.12.0 / ByteBuddy non supportava
i class file Java 25.

---

## DTR-025c — Convenzioni di test: gestione file assente e risoluzione timer

**Milestone**: M3 | **Stato**: Accettata

**Decisione**: `readLogFile` restituisce stringa vuota se il file è assente.
Minimo `Thread.sleep(50)` nei test dello scheduler — risoluzione timer Windows ~15ms.

---

## DTR-026 — Strategia conflitti SYNCHRONIZE: `-X ours` con backup FIFO

**Milestone**: M5 | **Stato**: Accettata

**Decisione**: `-X ours`. Sequenza: commit locale → pull → in caso di conflitto:
merge --abort → backup FIFO → pull -X ours --no-edit → estrazione versioni remote
via `git show FETCH_HEAD:<file>` → push.

Vincoli verificati sul campo: `--no-pager` obbligatorio su `git show`;
`--no-edit` obbligatorio su `pull -X ours`; usare `FETCH_HEAD` non `MERGE_HEAD`.

---

## DTR-027 — Backup FIFO: massimo 3 snapshot per vault

**Milestone**: M5 | **Stato**: Accettata

**Decisione**: max 3 per vault. FIFO — il più vecchio viene eliminato al
raggiungimento del limite. `backups/<owner>_<name>_<timestamp>/`. Esclude i
pattern di `.gitignore`.

---

## DTR-028 — TrayIcon: quattro stati visivi

**Milestone**: M5 | **Stato**: Pianificata — backend pronto, UI rimandata a M8+

Idle (verde), Syncing (animato), Error (rosso), Conflict (arancione).

---

## DTR-029 — ContextMenu: zero decisioni cognitive

**Milestone**: M5 | **Stato**: Pianificata — UI rimandata a M8+

`PopupMenu` AWT con azioni sync/pull/log/cartella. Le etichette descrivono
i risultati, non le operazioni Git.

---

## DTR-030 — VaultSwitcherPanel e notifiche

**Milestone**: M5 | **Stato**: Pianificata — UI rimandata a M8+

`CheckboxMenuItem` per vault. `Dialog<R>`/`Alert` JavaFX per notifiche
persistenti su conflitti/errori di rete (non Swing `JDialog`).

---

## DTR-031 — Identità del vault: campo `owner` e `repoSlug` derivato

**Milestone**: M5/M6 | **Stato**: Accettata — credenziali per-vault implementate M7

**Decisione**: `Vault.owner` + `getRepoSlug()` derivato come `<owner>/<name>`.
Non persistito separatamente. Usato come `universalId` nel log, prefisso directory
snapshot/conflict, chiave di risoluzione CLI.

**Credenziali per-vault** (chiuso M7): `gitName`, `gitEmail`, `gitUsername`,
`gitToken`, `gitBranch`, `gitRemote` — opzionali, risolti via
`StringUtil.coalesce(valoreVault, valoreGlobale)`. Applicati al bootstrap via
`GitService.bootstrapVault(Vault)`.

---

## DTR-032 — Separazione domain/DTO: no annotazioni Jackson nelle classi domain

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: annotazioni Jackson solo nel package `dto`. Convention uniforme
`toDomain()`/`fromDomain()`. `JsonMapper` unica classe che interagisce con
entrambi i layer.

---

## DTR-033 — `VaultContext` come record

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: record Java con quattro componenti (`Vault`, `SyncEventQueue`,
`SyncOrchestrator`, `ScheduledFuture`). Impostati una sola volta, mai sostituiti.

---

## DTR-034 — `LogService`: fan-out multi-writer e scoping per vault

**Milestone**: M5/M6 | **Stato**: Accettata — sostituisce DTR-009

**Decisione**: `List<LogWriter>` dalla property `log.writers`. `withVault(repoSlug)`
restituisce una nuova istanza condividendo gli stessi writer. Thread-safety per
writer: `FileLogWriter` usa `synchronized`; `SeqHttpLogWriter` usa `BlockingQueue`.
`InMemoryLogWriter` non configurabile — istanziato direttamente dal codice runtime.

---

## DTR-035 — `SeqHttpLogWriter`: consegna asincrona via coda interna e daemon thread

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: `BlockingQueue` (capacità 1000) + daemon thread `seq-log-writer`.
Coda piena → evento scartato silenziosamente su `stderr`. `close()` svuota la
coda (timeout 5s) poi interrompe il worker.

---

## DTR-036 — `GitignoreService`: stateless, casting type-safe

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: nessuno stato mutabile di istanza — thread-safe su vault concorrenti.
Cast type-safe via `filter(isInstance).map(cast)`.

---

## DTR-037 — `CommandUtil.runCommandToFile`: redirezione OS binary-safe

**Milestone**: M5/M6 | **Stato**: Accettata — estende DTR-023

**Decisione**: `ProcessBuilder.redirectOutput(File)` — l'OS scrive direttamente
sul file, bypassando l'heap JVM. Obbligatorio per contenuto binario da `git show`.

---

## DTR-038 — `VaultService.saveConflict`: spostamento atomico da file temporaneo

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: `Files.move` con `REPLACE_EXISTING` — atomico su spostamenti
nello stesso filesystem. Il chiamante usa `try/finally` con `deleteIfExists`
come rete di sicurezza.

---

## DTR-039 — Coda broadcast e dispatcher per eventi cross-vault

**Milestone**: M6 | **Stato**: Accettata

**Decisione**: singola `SyncEventQueue` broadcast. Thread broadcaster smista:
`vaultId == null` → fan-out su tutti; `vaultId != null` → solo la coda corrispondente.

---

## DTR-040 — Wiring per-vault di `SyncOrchestrator`, startup con thread

**Milestone**: M6 | **Stato**: Accettata — estende DTR-021, aggiornata M7

**Decisione**: un `SyncOrchestrator` per vault. Ogni `start()` gira su un thread
dedicato; il thread principale fa join su tutti. M7: `SyncOrchestrator` riceve
`Vault` direttamente invece di `Properties`.

---

## DTR-041 — Naming delle property di configurazione

**Milestone**: M6 | **Stato**: Accettata

**Decisione**: `path.backup` → `backup.path`, `path.conflicts` → `conflicts.path`.
Il codice è la source of truth.

---

## DTR-042 — JavaFX invece di Swing per MainWindow

**Milestone**: M6 | **Stato**: Pianificata — UI rimandata a M8+

**Decisione**: JavaFX per tutti i componenti `MainWindow`. AWT mantenuto solo per
`SystemTray`/`TrayIcon` (nessun equivalente JavaFX). Coesistenza via
`Platform.setImplicitExit(false)` e `Platform.runLater()`.

---

## DTR-043 — MainWindow: sei tab, apertura contestuale

**Milestone**: M6 | **Stato**: Pianificata — UI rimandata a M8+

`TabPane`: Home, Properties, Log, Conflicts, Backup, Settings. Vault switcher
nella toolbar. Apertura contestuale dal thread AWT via `Platform.runLater()`.

---

## DTR-044 — ForgeUI: design system JavaFX condiviso

**Milestone**: M6 | **Stato**: In corso (progetto separato)

**Decisione**: progetto Maven separato `forgeui`. Tre temi via CSS swap.
Candidato a Maven Central.

---

## DTR-045 — i18n: 10 lingue, ResourceBundle

**Milestone**: M6 | **Stato**: Pianificata — UI rimandata a M8+

10 locale che coprono ~75% degli utenti internet globali. RTL via
`NodeOrientation.RIGHT_TO_LEFT`.

---

## DTR-046 — VaultService: vincoli di unicità

**Milestone**: M5/M7 | **Stato**: Accettata — implementata M7

**Contesto**: i nomi delle directory di backup/conflict derivavano originariamente
dal solo `vault.name`, permettendo collisioni tra vault con lo stesso nome ma
owner diversi.

**Decisione**: unicità imposta su `repoSlug` (`owner/name`), non su `name`.
Aggiunto vincolo di unicità separato per il `path`. Entrambi verificati in
`create()`, `update()` e `load()`. Prefisso directory snapshot/conflict cambiato
in `<owner>_<name>_<timestamp>`.

---

## DTR-047 — CLI: argomenti a flag invece di posizionali

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: gli argomenti posizionali sono ambigui quando esistono più parametri
opzionali (`properties_file`, `vaultId`, `messagePath`).

**Decisione**: `operation` posizionale (args[0]); tutto il resto come flag
`--key=value` in ordine libero. `--config` default `./config.properties`.
Parsing manuale senza dipendenze esterne.

---

## DTR-048 — `EventType.mandatoryVault`: broadcast vs. errore nel dispatch

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: `COMMIT_MANUAL` porta un messaggio dell'utente significativo solo
per un singolo repository — il broadcast non ha senso.

**Decisione**: campo `boolean mandatoryVault` su `EventType`. `--vault` assente +
`mandatoryVault=true` → `System.exit(1)` con errore. `--vault` assente +
`mandatoryVault=false` → broadcast. Solo `COMMIT_MANUAL` è mandatory.

---

## DTR-049 — Sicurezza token: mascheramento degli argomenti sensibili in CommandUtil

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: il token GitHub embedded nell'URL remoto non deve apparire nei log.

**Decisione**: overload `CommandUtil.runCommand` con `Set<String> sensitiveArgs`.
Gli argomenti nel set sono sostituiti da `<hidden>` nel log; il processo riceve i
valori reali. `bootstrapVault` passa `Set.of(remoteUrl)`. `handleConfig` logga
`git.token=<hidden>`.

**Proprietà di sicurezza**: il token appare solo in `.git/config` (locale, mai
committato) — mai come argomento di comando, mai nei file di log.

---

## DTR-050 — Flag daemon: processo one-shot vs. long-running

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: le operazioni CLI devono terminare automaticamente dopo il
completamento. La Tray richiede un processo long-running.

**Decisione**: flag `--daemon`. Senza flag: `awaitIdle()` fa polling ogni 250ms su
tutte le code per-vault; a coda vuota + 500ms di settling, chiama `System.exit(0)`.
Con `--daemon`: blocca su `worker.join()` come prima. Le operazioni early-exit
(`status`, `config`) terminano sempre subito.

**Motivazione**: binario unico, due modalità. Task Scheduler usa one-shot (senza
flag); la Tray usa `--daemon`.

---

## DTR-051 — `NomadSync status`: early-exit, output su System.out

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: l'output di `git status` è una risposta interattiva all'utente,
non un evento di sistema.

**Decisione**: `status` è un'operazione early-exit — nessun orchestratore, nessuno
scheduler. `GitService.status(Vault)` usa `runCommandWithLines` (preserva i
newline). Output su `System.out`; errori su `logService.error`. Senza `--vault`:
broadcast su tutti i vault con intestazioni `=== repoSlug ===`.

---

## DTR-052 — `NomadProperties`/`NomadPropertiesLoader`: pattern ForgeUI applicato

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: magic string per le chiavi di property sparse in `Main`, `LogService`,
`GitService`, `VaultService`, `SocketClient`, `SocketServer`, `AutosaveScheduler`,
`SeqHttpLogWriter`.

**Decisione**: `NomadProperties` (registro chiavi, nested class per dominio: `Git`,
`Path`, `Log`, `Autosave`, `Commit`, `Socket`) e `NomadPropertiesLoader` (loader
da classpath, `getBoolean`, `getEnum`) — pattern identico a ForgeUI
`ForgeProperties`/`ForgePropertiesLoader`. Tutte le magic string eliminate.

---

# NomadSync — Decision Track Record (Italiano)

> Registro delle decisioni tecniche e architetturali significative prese durante
> il progetto, con motivazione e alternative valutate. Aggiornato all'evolversi
> del progetto.

---

## DTR-001 — Sincronizzazione via Git invece di cloud storage

**Milestone**: M1 | **Stato**: Accettata

**Contesto**: le cartelle devono restare sincronizzate su più macchine. I servizi
di cloud sync sono incompatibili con applicazioni che tengono file aperti.

**Decisione**: Git + GitHub.

**Motivazione**: le operazioni pull/push sono atomiche e sincrone; il diff nativo
consente autosave differenziale; nessun processo esterno tocca i file mentre le
applicazioni sono aperte.

**Alternative scartate**: OneDrive (conflitti per scritture concorrenti); Syncthing
(P2P, richiede almeno un device sempre acceso); Obsidian Sync (a pagamento ~4$/mese).

---

## DTR-002 — Strategia `theirs` per la risoluzione dei conflitti in pull

**Milestone**: M1 | **Stato**: Superata da DTR-013 (M5)

**Decisione**: `git pull -X theirs` — in caso di conflitto vince sempre la versione remota.

**Rischi accettati**: modifiche locali non committate prima del push potrebbero
essere perse. Mitigato da autosave periodico e `git stash` prima del pull.

---

## DTR-003 — Sequenza logon: stash → pull → stash pop

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `git stash` prima del pull e `git stash pop` dopo.

**Motivazione**: preserva le modifiche locali senza richiedere un commit esplicito.
**Alternativa scartata**: `git reset --hard` — distrugge le modifiche locali senza
possibilità di recupero.

---

## DTR-004 — Aggancio logon/logoff tramite Task Scheduler

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: Task Scheduler (`taskschd.msc`). Disponibile su tutte le edizioni
di Windows inclusa Home; interfaccia ispezionabile; cronologia esecuzioni; export
XML per replicabilità.

**Alternativa scartata**: Group Policy Editor — non disponibile su Windows Home.

---

## DTR-005 — Autosave differenziale tramite `git diff --quiet`

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `git diff --quiet` come guard — exit code 0 = nessuna modifica,
exit code 1 = modifiche presenti. Previene commit vuoti.

---

## DTR-006 — Fat JAR tramite `maven-assembly-plugin`

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `maven-assembly-plugin` con descriptor `jar-with-dependencies`.
Singolo artefatto autonomo; deploy copiando la cartella `target/`.

---

## DTR-007 — Risorse copiate in `target/` tramite `maven-resources-plugin`

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `maven-resources-plugin` copia da `src/main/resources/` a `target/`
durante la fase `package`. Deploy autocontenuto in `target/`.

---

## DTR-008 — Due file di configurazione separati per ambiente

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `config.dev.properties`/`config.prod.properties` esclusi da
`.gitignore`; `config.properties.template` committato come riferimento.
`vaults.json` anch'esso escluso — solo `vaults.json.template` committato.

---

## DTR-009 — Logging thread-safe tramite `synchronized`

**Milestone**: M1 | **Stato**: Superata da DTR-034 (M5)

**Decisione**: metodo `LogService.log()` dichiarato `synchronized`. Semplice e
corretto per al massimo due thread.

---

## DTR-010 — `SyncOrchestrator` come layer intermedio

**Milestone**: M1 | **Stato**: Accettata

**Decisione**: `SyncOrchestrator` tra `Main` e `GitService`. La logica di
coordinamento (stash prima del pull, gestione exit code) non appartiene né all'uno
né all'altro.

---

## DTR-011 — Architettura event-driven per SyncOrchestrator

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: coda a priorità. I chiamanti pubblicano eventi; l'orchestratore
consuma serialmente. Git è seriale — operazioni concorrenti sullo stesso repository
causano conflitti.

**Alternativa scartata**: chiamate dirette — la gestione della concorrenza si
distribuisce nel codice.

---

## DTR-012 — Scala di priorità degli eventi (originale)

**Milestone**: M2 | **Stato**: Superata da DTR-013 (M5)

Scala originale: PULL_LOGON(1), PUSH_MANUAL(2), PUSH_LOGOFF(3), AUTOSAVE(4).

---

## DTR-013 — SYNCHRONIZE sostituisce PULL_MANUAL e PUSH_MANUAL

**Milestone**: M5 | **Stato**: Accettata — sostituisce DTR-012

**Decisione**: evento unico `SYNCHRONIZE`. Scala aggiornata in M7:

| Priorità | Evento |
|---|---|
| 1 | `PULL_LOGON` |
| 2 | `SYNCHRONIZE` |
| 3 | `PUSH_LOGOFF` |
| 4 | `COMMIT_MANUAL` |
| 5 | `AUTOSAVE` |

---

## DTR-014 — Strategia di deduplicazione: latest wins

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: un evento in coda viene sostituito dal più recente dello stesso tipo.
Un evento in esecuzione non viene interrotto.

---

## DTR-015 — Retry con exponential backoff

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: massimo 3 tentativi. Delay progressivo: 30s → 60s → 120s. Dopo il
terzo fallimento l'evento viene scartato e viene invocato `NotificationHook.onFailure`.

---

## DTR-016 — Hook di notifica come dependency inversion

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: interfaccia `NotificationHook`. Implementazione di default scrive
su log. La Tray si aggancia implementando la stessa interfaccia senza modificare
l'orchestratore.

---

## DTR-017 — Separazione commit locale / push remoto

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: `AUTOSAVE` → solo commit locale. `PUSH_LOGOFF` / `SYNCHRONIZE` →
commit locale + push remoto. Il retry si applica solo alle operazioni remote.

---

## DTR-018 — Guard `hasUncommittedChanges()` prima di stash/stashPop

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: `git status --porcelain` come guard. `git diff --quiet` non
sufficiente — non rileva le modifiche staged.

---

## DTR-019 — `notify()` rinominato in `onFailure()` in NotificationHook

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: `onFailure(SyncEvent, String message)`. Evita collisione con
`Object.notify()`; `message` comunica la causa del fallimento alla Tray.

---

## DTR-020 — Thread semplice invece di ExecutorService per il worker loop

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: `Thread` semplice istanziato internamente. `ExecutorService` non
aggiunge valore per un singolo worker con loop infinito. Shutdown via `interrupt()`
+ `join()`.

---

## DTR-021 — Shutdown hook registrato in Main

**Milestone**: M2 | **Stato**: Accettata — estesa in M6 (DTR-040), M7 (DTR-050)

**Decisione**: shutdown hook in `Main`. Ordine: `scheduler.stop()` →
`broadcaster.interrupt()` → `orchestrators.forEach(stop)` → `logService.close()`.

---

## DTR-022 — Git Flow come strategia di branching

**Milestone**: M2 | **Stato**: Accettata

**Decisione**: Git Flow AVH Edition. `main` (solo release), `develop`
(integrazione continua), `feature/*` (un branch per obiettivo di grooming),
`release/*`, `hotfix/*`.

---

## DTR-023 — `CommandUtil` come utility condivisa per l'esecuzione di processi

**Milestone**: M3 | **Stato**: Accettata — estesa in M5 (DTR-037), M7 (DTR-049)

**Decisione**: classe statica `CommandUtil` nel package `util`. Parametro
`LogService` opzionale; overload senza per i test helper.

---

## DTR-024 — Costruttori package-private di test su `SyncEvent` e `AutosaveScheduler`

**Milestone**: M3 | **Stato**: Accettata

**Decisione**: costruttore package-private su `SyncEvent` con timestamp
controllato. Costruttore package-private su `AutosaveScheduler` con `TimeUnit`
per intervalli sub-minuto nei test.

---

## DTR-025 — `SyncEventQueue` usata come istanza reale in `SyncOrchestratorTest`

**Milestone**: M3 | **Stato**: Accettata

**Decisione**: istanza reale di `SyncEventQueue`; mock solo `GitService` e
`NotificationHook`. Logica pura senza side effect — candidata ideale all'uso
reale nei test.

---

## DTR-025b — Downgrade JDK da 25 a 21

**Milestone**: M3 | **Stato**: Accettata

**Decisione**: Oracle OpenJDK 21 LTS. Mockito 5.12.0 / ByteBuddy non supportava
i class file Java 25.

---

## DTR-025c — Convenzioni di test: gestione file assente e risoluzione timer

**Milestone**: M3 | **Stato**: Accettata

**Decisione**: `readLogFile` restituisce stringa vuota se il file è assente.
Minimo `Thread.sleep(50)` nei test dello scheduler — risoluzione timer Windows ~15ms.

---

## DTR-026 — Strategia conflitti SYNCHRONIZE: `-X ours` con backup FIFO

**Milestone**: M5 | **Stato**: Accettata

**Decisione**: `-X ours`. Sequenza: commit locale → pull → in caso di conflitto:
merge --abort → backup FIFO → pull -X ours --no-edit → estrazione versioni remote
via `git show FETCH_HEAD:<file>` → push.

Vincoli verificati sul campo: `--no-pager` obbligatorio su `git show`;
`--no-edit` obbligatorio su `pull -X ours`; usare `FETCH_HEAD` non `MERGE_HEAD`.

---

## DTR-027 — Backup FIFO: massimo 3 snapshot per vault

**Milestone**: M5 | **Stato**: Accettata

**Decisione**: max 3 per vault. FIFO — il più vecchio viene eliminato al
raggiungimento del limite. `backups/<owner>_<name>_<timestamp>/`. Esclude i
pattern di `.gitignore`.

---

## DTR-028 — TrayIcon: quattro stati visivi

**Milestone**: M5 | **Stato**: Pianificata — backend pronto, UI rimandata a M8+

Idle (verde), Syncing (animato), Error (rosso), Conflict (arancione).

---

## DTR-029 — ContextMenu: zero decisioni cognitive

**Milestone**: M5 | **Stato**: Pianificata — UI rimandata a M8+

`PopupMenu` AWT con azioni sync/pull/log/cartella. Le etichette descrivono
i risultati, non le operazioni Git.

---

## DTR-030 — VaultSwitcherPanel e notifiche

**Milestone**: M5 | **Stato**: Pianificata — UI rimandata a M8+

`CheckboxMenuItem` per vault. `Dialog<R>`/`Alert` JavaFX per notifiche
persistenti su conflitti/errori di rete (non Swing `JDialog`).

---

## DTR-031 — Identità del vault: campo `owner` e `repoSlug` derivato

**Milestone**: M5/M6 | **Stato**: Accettata — credenziali per-vault implementate M7

**Decisione**: `Vault.owner` + `getRepoSlug()` derivato come `<owner>/<name>`.
Non persistito separatamente. Usato come `universalId` nel log, prefisso directory
snapshot/conflict, chiave di risoluzione CLI.

**Credenziali per-vault** (chiuso M7): `gitName`, `gitEmail`, `gitUsername`,
`gitToken`, `gitBranch`, `gitRemote` — opzionali, risolti via
`StringUtil.coalesce(valoreVault, valoreGlobale)`. Applicati al bootstrap via
`GitService.bootstrapVault(Vault)`.

---

## DTR-032 — Separazione domain/DTO: no annotazioni Jackson nelle classi domain

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: annotazioni Jackson solo nel package `dto`. Convention uniforme
`toDomain()`/`fromDomain()`. `JsonMapper` unica classe che interagisce con
entrambi i layer.

---

## DTR-033 — `VaultContext` come record

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: record Java con quattro componenti (`Vault`, `SyncEventQueue`,
`SyncOrchestrator`, `ScheduledFuture`). Impostati una sola volta, mai sostituiti.

---

## DTR-034 — `LogService`: fan-out multi-writer e scoping per vault

**Milestone**: M5/M6 | **Stato**: Accettata — sostituisce DTR-009

**Decisione**: `List<LogWriter>` dalla property `log.writers`. `withVault(repoSlug)`
restituisce una nuova istanza condividendo gli stessi writer. Thread-safety per
writer: `FileLogWriter` usa `synchronized`; `SeqHttpLogWriter` usa `BlockingQueue`.
`InMemoryLogWriter` non configurabile — istanziato direttamente dal codice runtime.

---

## DTR-035 — `SeqHttpLogWriter`: consegna asincrona via coda interna e daemon thread

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: `BlockingQueue` (capacità 1000) + daemon thread `seq-log-writer`.
Coda piena → evento scartato silenziosamente su `stderr`. `close()` svuota la
coda (timeout 5s) poi interrompe il worker.

---

## DTR-036 — `GitignoreService`: stateless, casting type-safe

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: nessuno stato mutabile di istanza — thread-safe su vault concorrenti.
Cast type-safe via `filter(isInstance).map(cast)`.

---

## DTR-037 — `CommandUtil.runCommandToFile`: redirezione OS binary-safe

**Milestone**: M5/M6 | **Stato**: Accettata — estende DTR-023

**Decisione**: `ProcessBuilder.redirectOutput(File)` — l'OS scrive direttamente
sul file, bypassando l'heap JVM. Obbligatorio per contenuto binario da `git show`.

---

## DTR-038 — `VaultService.saveConflict`: spostamento atomico da file temporaneo

**Milestone**: M5/M6 | **Stato**: Accettata

**Decisione**: `Files.move` con `REPLACE_EXISTING` — atomico su spostamenti
nello stesso filesystem. Il chiamante usa `try/finally` con `deleteIfExists`
come rete di sicurezza.

---

## DTR-039 — Coda broadcast e dispatcher per eventi cross-vault

**Milestone**: M6 | **Stato**: Accettata

**Decisione**: singola `SyncEventQueue` broadcast. Thread broadcaster smista:
`vaultId == null` → fan-out su tutti; `vaultId != null` → solo la coda corrispondente.

---

## DTR-040 — Wiring per-vault di `SyncOrchestrator`, startup con thread

**Milestone**: M6 | **Stato**: Accettata — estende DTR-021, aggiornata M7

**Decisione**: un `SyncOrchestrator` per vault. Ogni `start()` gira su un thread
dedicato; il thread principale fa join su tutti. M7: `SyncOrchestrator` riceve
`Vault` direttamente invece di `Properties`.

---

## DTR-041 — Naming delle property di configurazione

**Milestone**: M6 | **Stato**: Accettata

**Decisione**: `path.backup` → `backup.path`, `path.conflicts` → `conflicts.path`.
Il codice è la source of truth.

---

## DTR-042 — JavaFX invece di Swing per MainWindow

**Milestone**: M6 | **Stato**: Pianificata — UI rimandata a M8+

**Decisione**: JavaFX per tutti i componenti `MainWindow`. AWT mantenuto solo per
`SystemTray`/`TrayIcon` (nessun equivalente JavaFX). Coesistenza via
`Platform.setImplicitExit(false)` e `Platform.runLater()`.

---

## DTR-043 — MainWindow: sei tab, apertura contestuale

**Milestone**: M6 | **Stato**: Pianificata — UI rimandata a M8+

`TabPane`: Home, Properties, Log, Conflicts, Backup, Settings. Vault switcher
nella toolbar. Apertura contestuale dal thread AWT via `Platform.runLater()`.

---

## DTR-044 — ForgeUI: design system JavaFX condiviso

**Milestone**: M6 | **Stato**: In corso (progetto separato)

**Decisione**: progetto Maven separato `forgeui`. Tre temi via CSS swap.
Candidato a Maven Central.

---

## DTR-045 — i18n: 10 lingue, ResourceBundle

**Milestone**: M6 | **Stato**: Pianificata — UI rimandata a M8+

10 locale che coprono ~75% degli utenti internet globali. RTL via
`NodeOrientation.RIGHT_TO_LEFT`.

---

## DTR-046 — VaultService: vincoli di unicità

**Milestone**: M5/M7 | **Stato**: Accettata — implementata M7

**Contesto**: i nomi delle directory di backup/conflict derivavano originariamente
dal solo `vault.name`, permettendo collisioni tra vault con lo stesso nome ma
owner diversi.

**Decisione**: unicità imposta su `repoSlug` (`owner/name`), non su `name`.
Aggiunto vincolo di unicità separato per il `path`. Entrambi verificati in
`create()`, `update()` e `load()`. Prefisso directory snapshot/conflict cambiato
in `<owner>_<name>_<timestamp>`.

---

## DTR-047 — CLI: argomenti a flag invece di posizionali

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: gli argomenti posizionali sono ambigui quando esistono più parametri
opzionali (`properties_file`, `vaultId`, `messagePath`).

**Decisione**: `operation` posizionale (args[0]); tutto il resto come flag
`--key=value` in ordine libero. `--config` default `./config.properties`.
Parsing manuale senza dipendenze esterne.

---

## DTR-048 — `EventType.mandatoryVault`: broadcast vs. errore nel dispatch

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: `COMMIT_MANUAL` porta un messaggio dell'utente significativo solo
per un singolo repository — il broadcast non ha senso.

**Decisione**: campo `boolean mandatoryVault` su `EventType`. `--vault` assente +
`mandatoryVault=true` → `System.exit(1)` con errore. `--vault` assente +
`mandatoryVault=false` → broadcast. Solo `COMMIT_MANUAL` è mandatory.

---

## DTR-049 — Sicurezza token: mascheramento degli argomenti sensibili in CommandUtil

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: il token GitHub embedded nell'URL remoto non deve apparire nei log.

**Decisione**: overload `CommandUtil.runCommand` con `Set<String> sensitiveArgs`.
Gli argomenti nel set sono sostituiti da `<hidden>` nel log; il processo riceve i
valori reali. `bootstrapVault` passa `Set.of(remoteUrl)`. `handleConfig` logga
`git.token=<hidden>`.

**Proprietà di sicurezza**: il token appare solo in `.git/config` (locale, mai
committato) — mai come argomento di comando, mai nei file di log.

---

## DTR-050 — Flag daemon: processo one-shot vs. long-running

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: le operazioni CLI devono terminare automaticamente dopo il
completamento. La Tray richiede un processo long-running.

**Decisione**: flag `--daemon`. Senza flag: `awaitIdle()` fa polling ogni 250ms su
tutte le code per-vault; a coda vuota + 500ms di settling, chiama `System.exit(0)`.
Con `--daemon`: blocca su `worker.join()` come prima. Le operazioni early-exit
(`status`, `config`) terminano sempre subito.

**Motivazione**: binario unico, due modalità. Task Scheduler usa one-shot (senza
flag); la Tray usa `--daemon`.

---

## DTR-051 — `NomadSync status`: early-exit, output su System.out

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: l'output di `git status` è una risposta interattiva all'utente,
non un evento di sistema.

**Decisione**: `status` è un'operazione early-exit — nessun orchestratore, nessuno
scheduler. `GitService.status(Vault)` usa `runCommandWithLines` (preserva i
newline). Output su `System.out`; errori su `logService.error`. Senza `--vault`:
broadcast su tutti i vault con intestazioni `=== repoSlug ===`.

---

## DTR-052 — `NomadProperties`/`NomadPropertiesLoader`: pattern ForgeUI applicato

**Milestone**: M7 | **Stato**: Accettata

**Contesto**: magic string per le chiavi di property sparse in `Main`, `LogService`,
`GitService`, `VaultService`, `SocketClient`, `SocketServer`, `AutosaveScheduler`,
`SeqHttpLogWriter`.

**Decisione**: `NomadProperties` (registro chiavi, nested class per dominio: `Git`,
`Path`, `Log`, `Autosave`, `Commit`, `Socket`) e `NomadPropertiesLoader` (loader
da classpath, `getBoolean`, `getEnum`) — pattern identico a ForgeUI
`ForgeProperties`/`ForgePropertiesLoader`. Tutte le magic string eliminate.

---
 
## DTR-052 — `NomadProperties`/`NomadPropertiesLoader`: pattern ForgeUI applicato
 
**Milestone**: M7 | **Stato**: Accettata
 
**Contesto**: magic string per le chiavi di property sparse in `Main`, `LogService`,
`GitService`, `VaultService`, `SocketClient`, `SocketServer`, `AutosaveScheduler`,
`SeqHttpLogWriter`.
 
**Decisione**: `NomadProperties` (registro chiavi, nested class per dominio: `Git`,
`Path`, `Log`, `Autosave`, `Commit`, `Socket`) e `NomadPropertiesLoader` (loader
da classpath, `getBoolean`, `getEnum`) — pattern identico a ForgeUI
`ForgeProperties`/`ForgePropertiesLoader`. Tutte le magic string eliminate.
 
---
 
## DTR-053 — Strategia logoff macOS: launchd daemon + shutdown hook SIGTERM
 
**Milestone**: M8 | **Stato**: Accettata per v1.0.0 — Opzione C rimandata a v2.0.0
 
**Contesto**: macOS non ha un trigger di logoff affidabile e non deprecato
equivalente al Task Scheduler di Windows. Opzioni valutate:
 
- `LogoutHook` via `loginwindow` defaults — deprecato su Ventura+, richiede sudo
- `launchd WatchPaths` su path di sistema — fragile, behavior interno non documentato
- `NSWorkspaceWillPowerOffNotification` (bridge Cocoa/Swift) — API nativa corretta,
  richiede un binario nativo separato; rimandato a v2.0.0
**Decisione v1.0.0 — Opzione B**: NomadSync gira come user agent `launchd`
in modalità `--daemon`. Il pull avviene all'avvio dell'agent (`RunAtLoad = true`
= logon). Il push avviene tramite lo shutdown hook Java quando macOS invia
`SIGTERM` al logout/shutdown — macOS garantisce circa 20 secondi prima di
forzare la chiusura, sufficienti per un push in condizioni normali.
 
```
Windows:  Task Scheduler logoff trigger → NomadSyncPush.bat → evento PUSH_LOGOFF
macOS:    launchd SIGTERM → shutdown hook Java → pubblica PUSH_LOGOFF → awaitIdle → exit
```
 
Il codice Java è identico su entrambe le piattaforme. Solo il meccanismo di
trigger esterno differisce.
 
**Motivazione**: zero codice nativo, zero dipendenze Cocoa, riutilizza il flag
`--daemon` e l'infrastruttura dello shutdown hook di M7 (DTR-050). Il timeout di
20 secondi di macOS è sufficiente per il push in condizioni di rete normali.
 
**Rischio accettato**: su reti molto lente il push potrebbe non completarsi prima
che macOS forzi la terminazione. Un warning viene scritto nel file di log —
consultabile al prossimo avvio tramite `NomadSync doctor`, che riporta se l'ultimo
push è andato a buon fine. Non è un alert interattivo durante lo shutdown.
 
**v2.0.0 — Opzione C**: `NSWorkspaceWillPowerOffNotification` tramite un helper
Swift — tempo pre-shutdown garantito, elimina il rischio di timeout. Distribuzione
con certificato Apple Developer Belmani-Apex (nessun bypass Gatekeeper richiesto
agli utenti finali).
 
---
 
## DTR-054 — Sottocomando `NomadSync vault`: CRUD completo da CLI
 
**Milestone**: M8 | **Stato**: Accettata
 
**Contesto**: `vaults.json` non deve mai essere modificato manualmente dall'utente
finale. M7 ha introdotto `NomadSync config` per gli aggiornamenti delle credenziali,
ma non esisteva una superficie CLI per registrare, modificare, rimuovere o
ispezionare i vault.
 
**Decisione**: `NomadSync vault <sottocomando>` con cinque sottocomandi:
`add`, `update`, `remove`, `list`, `show`. Dispatch in `Main`
(`operation == "vault"` → routing su `args[1]`). Tutte le operazioni sono early-exit.
 
`vault add` valida l'esistenza del path, la presenza di `.git/` e l'unicità del
repoSlug prima di chiamare `VaultService.create` + `GitService.bootstrapVault`.
`vault update` modifica i campi non-credenziali di un vault esistente (path, branch,
remote, name, owner) — le credenziali sono gestite da `NomadSync config`.
`vault remove` richiede conferma interattiva — non cancella mai la cartella locale.
`vault list` mostra repoSlug, path, branch, remote e presenza del token (mai il valore).
`vault show` include le prime 3 righe di `git status --short` con `...` se
esistono ulteriori modifiche.
 
**Script**: un wrapper per sottocomando (`NomadSyncVaultAdd.bat/.sh` ecc.) per
la scopribilità BUBEZ, più `NomadSyncVault.bat/.sh` come entry point unificato
per i developer. Entrambi coesistono.
 
---
 
## DTR-055 — `NomadSync setup`: wizard console per l'onboarding al primo avvio
 
**Milestone**: M8 | **Stato**: Accettata
 
**Contesto**: il processo di installazione manuale in quattro passi è inaccessibile
agli utenti non tecnici.
 
**Decisione**: `NomadSync setup` è un wizard console che raccoglie identità Git,
token e primo vault in una sequenza guidata, poi chiama in ordine `handleConfig`,
`VaultService.create`, `GitService.bootstrapVault` e `handleOsRegistration`.
L'input del token usa `System.console().readPassword()` — non appare mai sullo
schermo. Nessun default pre-compilato con dati personali nei prompt.
 
`Main` rileva il primo avvio verificando l'assenza di `config.properties` e lancia
automaticamente `setup`. jpackage produce un installer nativo (`.exe` su Windows,
`.pkg` su macOS) con JRE bundled — l'utente scarica da Softonic, esegue l'installer
(doppio click), NomadSync viene installato in `Program Files` / `/Applications`,
e al primo avvio parte automaticamente il wizard di configurazione.
 
**Motivazione**: unico punto di ingresso per gli utenti non tecnici; internamente
riutilizza tutta l'infrastruttura CLI esistente di M7 — nessuna nuova logica di
persistenza.
 
---
 
## DTR-056 — Registrazione OS automatica dal wizard di setup
 
**Milestone**: M8 | **Stato**: Accettata
 
**Contesto**: la registrazione di Task Scheduler (Windows) e launchd (macOS)
richiede comandi di sistema che gli utenti non tecnici non possono eseguire
manualmente.
 
**Decisione**:
- **Windows**: `schtasks.exe /create` per i task `Pull at logon` e `Push at logoff`,
  eseguiti via `ProcessBuilder`. Richiede privilegi elevati — l'installer jpackage
  gira già come admin.
- **macOS**: scrive `~/Library/LaunchAgents/io.aledep10.nomadsync.plist` e attiva
  via `launchctl load`. Non richiede elevazione per gli user agent.
Su entrambe le piattaforme: `setup` chiede conferma prima di registrare. Il
disinstallatore rimuove i task via `schtasks /delete` (Windows) e `launchctl
unload` + cancellazione del plist (macOS).
 
---
 
## DTR-057 — `NomadSync doctor`: health check con auto-riparazione opzionale
 
**Milestone**: M8 | **Stato**: Accettata
 
**Contesto**: il troubleshooting richiede l'apertura manuale dei log e l'esecuzione
di comandi Git. Su macOS, l'esito del push allo shutdown non è visibile
interattivamente — l'utente ha bisogno di un modo semplice per verificarlo al
riavvio.
 
**Decisione**: `NomadSync doctor` verifica `config.properties`, `vaults.json`,
eseguibile git, path dei vault, raggiungibilità del remote, presenza del token,
registrazione dei task OS ed esito dell'ultimo push allo shutdown (dal log).
Stampa un report strutturato su `System.out` con ✓/✗ per voce. `--fix` tenta
la riparazione automatica dei problemi rilevabili. Operazione early-exit.
 
---
 
## DTR-058 — Installer jpackage: nativo, JRE bundled, setup automatico al primo avvio
 
**Milestone**: M8 | **Stato**: Accettata
 
**Contesto**: richiedere Java 21 pre-installato è una barriera per gli utenti
non tecnici. Uno zip da scompattare manualmente non è accettabile per una v1.0.0.
 
**Decisione**: `jpackage` (integrato nel JDK 21) produce installer nativi `.exe`
(Windows) e `.pkg` (macOS) con JRE bundled. L'utente scarica da Softonic, esegue
l'installer (doppio click), NomadSync viene installato in `Program Files` /
`/Applications`. Al primo avvio, `Main` rileva l'assenza di `config.properties`
e invoca automaticamente `setup`.
 
**macOS Gatekeeper**: v1.0.0 — `.pkg` non firmato richiede `xattr -cr NomadSync.pkg`
prima dell'installazione; documentato nel README, accettabile per distribuzione
a developer e early adopter. v2.0.0: certificato Apple Developer Belmani-Apex —
distribuzione Gatekeeper-compliant, nessun bypass richiesto agli utenti finali.