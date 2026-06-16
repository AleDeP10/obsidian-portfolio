# MLS — Milestone 5: Multi-Vault Architecture (Foundations)

---

## Obiettivi raggiunti

Il refactoring multi-vault delle fondamenta è completo e coperto da 106 test green.

**Identità del vault.** `Vault` ha guadagnato un campo `owner` e un metodo derivato `getRepoSlug()` (`<owner>/<name>`), usato come identificatore universale nel logging e nel routing degli eventi. Le annotazioni Jackson sono state rimosse dalla classe domain e isolate in `VaultDto`, `VaultRootDto` e `SocketMessageDto`, ciascuno con metodi simmetrici `toDomain()` / `fromDomain()`. `JsonMapper` instrada ora tutta la persistenza dei vault attraverso questo layer DTO; gli overload ridondanti (`toJson(Vault)`, `toJson(SocketMessage)`, `toVault()`) sono stati rimossi in favore di un `toJson(Object)` generico.

**VaultContext come record.** Su suggerimento dell'IDE, `VaultContext` è diventato un record — composition over inheritance, zero boilerplate.

**LogService multi-writer e vault-scoped.** `LogService` costruisce dinamicamente la lista di `LogWriter` a partire dalla property `log.writers` (console, file, seq), con un costruttore privato unico e `List.copyOf` come punto centrale di immutabilità. Il metodo `withVault(repoSlug)` deriva istanze scoped a un vault specifico condividendo i writer fisici sottostanti — nessun file riaperto, nessun thread duplicato, solo il `repoSlug` scritto su ogni riga cambia.

**GitignoreService stateless.** `load`, `save` e `forSnapshot` operano interamente su parametri e valori di ritorno, senza stato mutabile tra chiamate. Il cast da `List<GitignorePattern>` a `List<SystemPattern>` è stato risolto in modo type-safe via Stream (`filter(isInstance)` + `map(cast)`), eliminando un cast grezzo con `@SuppressWarnings`.

**VaultService — CRUD, snapshot, conflitti.** CRUD completo con persistenza immediata su `vaults.json`. `makeVaultSnapshot` implementa il FIFO a 3 snapshot con `Files.walkFileTree` e `SimpleFileVisitor`, escludendo file e directory secondo i pattern attivi di `.gitignore`. `backupsRoot` e `conflictsRoot` sono configurabili via `backup.path`/`conflicts.path` con fallback su sottodirectory di `user.dir`. `saveConflict` sposta atomicamente (`Files.move`) il file di conflitto dalla posizione temporanea alla destinazione finale.

**GitService.synchronize — gestione conflitti.** Il loop sui file conflittati usa `CommandUtil.runCommandToFile` per scrivere `git show FETCH_HEAD:<file>` direttamente su un file temporaneo (binary-safe, nessun passaggio per la JVM heap), poi delega lo spostamento a `VaultService.saveConflict`. `GitignoreException` durante lo snapshot viene wrappata come `GitException` per uniformare la strategia di retry dell'orchestrator.

---

## Problematiche affrontate

**Covarianza dei generici.** `List<SystemPattern>` non è assegnabile a `List<GitignorePattern>` nonostante `SystemPattern extends GitignorePattern` — i generici Java sono invarianti per default. Risolto con un cast type-safe via Stream invece di un cast grezzo sull'intera lista.

**Gestione di contenuti binari da processo esterno.** `git show FETCH_HEAD:<file>` può restituire contenuto binario (immagini, PDF). Leggere riga per riga via `BufferedReader` lo corromperebbe. Risolto con `ProcessBuilder.redirectOutput(File)`, che fa scrivere l'output direttamente dal sistema operativo su file, bypassando completamente la JVM.

**Resource leak su `Files.list()`.** Lo `Stream<Path>` restituito da `Files.list()` mantiene aperto un file descriptor del sistema operativo finché non viene chiuso esplicitamente. Risolto sistematicamente con `try-with-resources` ovunque `Files.list()` venga usato.

**Breaking change su LogService.** Il refactoring ha cambiato la semantica del costruttore a due argomenti — da `(Properties, path)` implicito a `(Properties, repoSlug)` esplicito. La migrazione ha richiesto l'aggiornamento di tutti i test che costruivano `LogService` con quella firma, in particolare `LogServiceTest`, dove sono stati aggiunti test espliciti per `withVault` e per i casi limite di `buildWriters` (token sconosciuti, property mancanti).

---

## Cheat-sheet acquisiti

- **TTR_NomadSync_M5_Collections** — covarianza, cast type-safe via Stream, `Collectors.groupingBy`/`toMap` con merge function, copie difensive
- **TTR_NomadSync_M5_FileSystem** — `Path`/`Files`, `FileVisitor`/`SimpleFileVisitor`, `PathMatcher` e pattern glob, cancellazione ricorsiva via `Files.walk` + `sorted(reverseOrder)`
- **TTR_NomadSync_M5_Maven** — esecuzione selettiva dei test (`-Dtest`), report `surefire`, fat JAR con `maven-assembly-plugin`, risorse copiate in `target/` con `maven-resources-plugin`