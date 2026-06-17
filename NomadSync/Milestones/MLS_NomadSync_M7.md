# MLS — Milestone 7: CLI Shortcuts + Vault Identity Refactoring

**Branch**: `feature/cli-shortcuts`
**Tests**: 147 green (↑ da 125 a chiusura M6)
**Commit**: 2 (`fix(vault-identity)` + `feat(cli-shortcuts)`)
**Data**: Giugno 2026

---

## Obiettivi raggiunti

### Sprint A — VaultService: identità e vincoli

Corretta DTR-046: il vincolo di unicità era su `Vault.name` — sbagliato.
GitHub garantisce unicità su `owner/name` (repoSlug). Due vault con lo stesso
`name` ma `owner` diversi sono repository GitHub distinti e legittimi.
Aggiunto vincolo di unicità sul `path` locale: due orchestratori sullo stesso
repository Git locale produrrebbero comportamenti indefiniti.

- `findByName` → **eliminato** (ogni uso era un bug latente)
- `findByRepoSlug(String)` → nuovo, match esatto su `owner/name`
- `findAllByName(String)` → nuovo, restituisce `List<Vault>` per rilevare ambiguità CLI
- `validateUniqueRepoSlugs` → sostituisce `validateUniqueNames`
- `validateUniquePaths` → nuovo
- `create`/`update` → check su repoSlug **e** path
- 32 test green su `VaultServiceTest`

### Sprint B — GitService: Vault-aware e credenziali per-vault (DTR-031 chiuso)

Tutti i metodi pubblici di `GitService` passano da `String vaultPath` a `Vault vault`.
`bootstrapVault(Vault)` scrive `user.name`, `user.email` e remote URL autenticato
(`https://<token>@github.com/<owner>/<name>`) nel `.git/config` locale del vault.
Token mai loggato — `CommandUtil.runCommand` con `Set<String> sensitiveArgs` mascheра
l'URL nel log come `<hidden>`. `synchronize` usa `repoSlug.replace("/", "_")` come
prefisso directory snapshot/conflict, eliminando collisioni tra vault con stesso
`name` e `owner` diversi.

### Sprint C — SyncOrchestrator: riceve Vault

`SyncOrchestrator` riceve `Vault` invece di `Properties`. `Main` non crea più
Properties derivate per-vault.

### Sprint D — CLI a flag, COMMIT_MANUAL, NomadSync config, OsUtil

- CLI riorganizzata: `operation` posizionale, resto `--key=value` ordine libero
- `--vault=<name>`: unambiguo se esattamente un match; errore con lista repoSlug
  se zero o >1. `--vault=<owner>/<name>`: match esatto
- `EventType.mandatoryVault`: assente + mandatory → errore; assente + !mandatory → broadcast
- `COMMIT_MANUAL` (priorità 4, `mandatoryVault=true`): editor interattivo via
  `ProcessBuilder.inheritIO().waitFor()`. Editor: `--editor` → `commit.editor` →
  `EDITOR` env → OS default
- `AUTOSAVE` → priorità 5
- `NomadSync config`: aggiorna `config.properties` (globale) o `vaults.json`
  (per-vault). `--git.token` loggato come `<hidden>`
- `NomadSync status`: `git status` human-readable su vault singolo o broadcast.
  Early-exit, no orchestratori
- `--daemon`: senza flag → early-exit automatico a code svuotate (`awaitIdle`).
  Con flag → processo long-running (modalità Tray futura)
- `OsUtil` + `Os { WINDOWS, UNIX }`

### Sprint E — NomadProperties / NomadPropertiesLoader

- `NomadProperties`: registro centralizzato di tutte le chiavi di `config.properties`
  (pattern identico a ForgeUI `ForgeProperties`)
- `NomadPropertiesLoader`: loader da classpath con fallback graceful, `getBoolean`,
  `getEnum` (pattern identico a ForgeUI `ForgePropertiesLoader`)
- Migrazione completa: zero magic strings in `Main`, `LogService`, `GitService`,
  `VaultService`, `SocketClient`, `SocketServer`, `AutosaveScheduler`, `SeqHttpLogWriter`

---

## Artefatti prodotti

### Nuovi file

| File | Note |
|------|------|
| `NomadProperties.java` | Registro chiavi config.properties |
| `NomadPropertiesLoader.java` | Loader classpath, pattern ForgeUI |
| `NomadPropertiesLoaderTest.java` | Test getBoolean, getEnum |
| `Os.java` | Enum `WINDOWS \| UNIX` |
| `OsUtil.java` | `detect()`, `isWindows()` |
| `NomadSync.bat/.sh` | Entry point con `--daemon` documentato |
| `NomadSyncPull.bat/.sh` | Wrapper pull |
| `NomadSyncPush.bat/.sh` | Wrapper push |
| `NomadSyncSync.bat/.sh` | Wrapper sync |
| `NomadSyncStatus.bat/.sh` | Wrapper status |
| `NomadSyncCommit.bat/.sh` | Wrapper commit con editor interattivo |
| `NomadSyncConfig.bat/.sh` | Wrapper config globale/per-vault |

### File modificati

| File | Modifiche principali |
|------|---------------------|
| `VaultService.java` | repoSlug+path uniqueness, findByRepoSlug, findAllByName |
| `VaultServiceTest.java` | 32 test, edge case DTR-046 corretta |
| `GitService.java` | Vault-aware, bootstrapVault, status(), NomadProperties |
| `GitServiceTest.java` | Vault parameter, 4 test bootstrapVault |
| `SyncOrchestrator.java` | Riceve Vault, COMMIT_MANUAL, Javadoc |
| `SyncOrchestratorTest.java` | 3 test COMMIT_MANUAL, 1 VaultException swallowed |
| `SyncEvent.java` | Campo message, costruttori aggiornati |
| `SyncEventTest.java` | forVault, message field, tutti 5 livelli priorità |
| `SyncEventQueue.java` | isEmpty() per awaitIdle |
| `EventType.java` | mandatoryVault, COMMIT_MANUAL=4, AUTOSAVE=5 |
| `CommandUtil.java` | sensitiveArgs masking, runCommandWithLines |
| `Main.java` | Riscritta: flag parser, vault resolution, editor, config, status, daemon |
| `LogService.java` | NomadProperties.Log.* |
| `AutosaveScheduler.java` | NomadProperties.Autosave.INTERVAL_MINUTES |
| `SeqHttpLogWriter.java` | normaliseUrl(), NomadProperties, fix BodyPublishers |
| `SocketClient.java` | NomadProperties.Socket.* |
| `SocketServer.java` | NomadProperties.Socket.PORT |
| `Vault.java` | gitBranch, gitRemote, Javadoc completo |
| `VaultDto.java` | gitBranch, gitRemote, getter stile uniforme |
| `ClefFormatter.java` | Javadoc, @Override |
| `ConsoleLogWriter.java` | Javadoc, costanti uppercase |
| `FileLogWriter.java` | Auto-crea parent dir, Path invece di File |
| `InMemoryLogWriter.java` | Javadoc |
| `LineFormatter.java` | Stack trace con `\tat` prefix, Javadoc |

---

## DTR aggiornate

| ID | Modifica |
|----|---------|
| DTR-031 | Chiuso: credenziali per-vault implementate in `GitService.bootstrapVault` |
| DTR-046 | Corretto: unicità su `repoSlug` (non `name`) + unicità su `path`. Snapshot/conflict prefix usa `owner_name` |

---

## Nuove DTR (M7)

| ID | Decisione |
|----|-----------|
| DTR-047 | CLI a flag `--key=value` invece di argomenti posizionali |
| DTR-048 | `EventType.mandatoryVault`: dispatch broadcast vs. errore |
| DTR-049 | Token Git mai loggato: `CommandUtil.sensitiveArgs` + `<hidden>` |
| DTR-050 | `--daemon` flag: one-shot vs. long-running. `awaitIdle` per terminazione automatica |
| DTR-051 | `NomadSync status`: early-exit, output su `System.out` non `LogService` |
| DTR-052 | `NomadProperties`/`NomadPropertiesLoader`: pattern ForgeUI applicato a NomadSync |

---

## Note tecniche

**`vaults.json` rimosso dal tracking** prima di M7 — `git rm --cached` eseguito.

**Editor interattivo**: "salva e chiudi per committare, chiudi senza salvare per
annullare" — pattern identico a `git commit` senza `-m`. Documentato nei script.

**`SocketServer.publish(String, SyncEvent)`**: predisposto per Tray (M8),
non ancora chiamato. Annotato nel Javadoc.

**Token security**: il token GitHub è scritto nel `.git/config` locale del vault
via `git remote set-url` — mai come argomento di comando, mai nei log.

---

## Pending (pre-v1.0.0)

- [ ] Test manuale editor: `notepad`, `notepad++`, `nano` durante e2e
- [ ] End-to-end FASE 2–6 su `nomad-test-vault`
- [ ] Coordinamento con Gabriela per allineamento branch dopo rimozione `vaults.json`
