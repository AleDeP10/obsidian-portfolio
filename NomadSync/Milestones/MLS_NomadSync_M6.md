# MLS — Milestone 6: Multi-Vault Wiring (Main, Broadcast, Release)

---

## Obiettivi raggiunti

Il wiring multi-vault è completo e la suite è verde — 112 test, esecuzione
stabile sia in ordine `filesystem` che `random`.

**Main — bootstrap e caricamento multi-vault.** `Main` valida gli argomenti CLI
(`operation`, `configPath`, `vaultId` opzionale), istanzia le dipendenze
condivise (`LogService`, `GitignoreService`, `VaultService`, `GitService`,
`NotificationHook`), e carica `vaults.json` tramite `vaultService.load()`,
gestendo sia `IOException` che la nuova `VaultException` (DTR-046).

**Wiring per-vault.** Per ogni `Vault` registrato, `Main` deriva una copia di
`Properties` con `vault.path` impostato, una `LogService` vault-scoped via
`withVault(repoSlug)`, una `SyncEventQueue` dedicata, e un `SyncOrchestrator`
con queste dipendenze derivate — esattamente come descritto in DTR-040.

**Broadcast queue e dispatcher.** Introdotta una `SyncEventQueue` di broadcast
separata dalle code per-vault, consumata da un thread dedicato
(`nomadsync-broadcaster`). Il dispatcher instrada per `vaultId`: `null` →
fan-out su tutte le code; valore specifico → routing sulla coda corrispondente,
con warning e scarto se il vault non esiste. `AutosaveScheduler` è tornato alla
sua firma originale a singola coda, ora puntata sulla broadcast queue —
zero modifiche alla sua logica interna (DTR-039).

**CLI multi-operazione.** `pull`/`push` risolvono il vault target con fallback
al primo vault registrato se `vaultId` è assente o non trovato. `sync` senza
`vaultId` pubblica un `SYNCHRONIZE` broadcast; con `vaultId` instrada
puntualmente. `autosave` non pubblica nulla — delegato interamente allo
scheduler.

**Shutdown ordinato.** Lo shutdown hook estende DTR-021:
`scheduler.stop()` → `broadcaster.interrupt()` →
`orchestrators.forEach(stop)` → `logService.close()`. Ogni
`SyncOrchestrator.start()` (bloccante) gira sul proprio thread; il thread main
fa il join su tutti.

**DTR-046 — unicità nome vault, implementata.** `VaultService.create()` e
`update()` lanciano `VaultException("duplicated vault name: ...")` su
collisione; `load()` valida tutti i nomi in `vaults.json` all'avvio. Il fix è
stato innescato dall'aggiunta di `belmani-apex` (owner `Belmani`, diverso da
`AleDeP10`) a `vaults.json` — uno scenario multi-owner reale che ha reso visibile
il gap.

**Allineamento proprietà.** Verificato e confermato che `path.backup`/
`path.conflicts`/`path.vaults` sono coerenti nella sezione `# Paths` di
`config.properties.template` — un draft intermedio di `VaultService` aveva
invertito il prefisso (`backup.path`), corretto prima del merge (DTR-041).

---

## Problematiche affrontate

**Race condition nel test di rinomina duplicata.** Il test
`update_renameToAnotherVaultsName_throwsVaultExceptionAndDoesNotPersist`
falliva in modo non deterministico solo con `-DrunOrder=random`, mai con
`filesystem`. La causa: il test mutava `vaultA.setName(...)` direttamente
sull'istanza viva restituita da `create()` — lo stesso oggetto già presente
nella map interna di `VaultService`. Questo creava temporaneamente due entry
con lo stesso `name` prima ancora di chiamare `update()`. `findByName()`, basato
su `findFirst()` su `HashMap.values()` (ordine di iterazione non garantito),
a volte trovava `vaultB` (comportamento corretto, test passa) e a volte trovava
`vaultA` stesso — già rinominato, quindi `existing.getId().equals(vault.getId())`
risultava `true`, la guardia di unicità non scattava, nessuna eccezione,
test fallito. Fix: il test ora costruisce una **copia** (`new Vault(...)`) per
la mutazione, non toccando mai l'istanza condivisa in memoria — pattern da
applicare ad ogni test futuro che modifica un oggetto restituito da `create()`.

**File non sincronizzato tra sessioni.** Più iterazioni di troubleshooting sono
state vanificate da `mvn clean test` che riportava "Nothing to compile - all
classes are up to date" — il file `VaultService.java` aggiornato non era stato
copiato nel progetto reale, solo il test lo era. Lezione: dopo ogni
modifica prodotta in sessione, verificare con un `grep` di una stringa
distintiva (es. `"duplicated vault name"`) che il file sul disco del progetto
corrisponda effettivamente alla versione attesa, prima di interpretare un
risultato di test come comportamento del nuovo codice.

**Seq — porta e protocollo.** L'istanza Seq via Docker mappa `5341:5341`
(ingestion API) e `80:80` (UI), ma il certificato HTTPS auto-generato non è
valido per `localhost` — la UI risponde solo su `http://`, non `https://`,
sulla porta 80. La root `/api`-style risponde `{"Error":"Not found."}` su
richieste senza header `Accept` corretti; la UI vive sotto hash-routing
(`/#/login`, `/#/events`). Risolto identificando l'URL corretto
`http://localhost/#/login`.

---

## Cheat-sheet acquisiti

- **TTR_NomadSync_M6_Threads** *(rename da TTR_NomadSync_M5_Threads — il
  contenuto su thread dedicati, `Thread` vs `ExecutorService`, e join multipli
  è maturato durante il wiring multi-vault di M6, non durante M5)*
- **TTR_NomadSync_M7_Seq** — setup Docker, trappola HTTPS/porta, login,
  query CLEF, pattern di verifica per test e2e multi-vault

---

## Stato in chiusura

112 test green, ordine `filesystem` e `random` entrambi stabili. `mvn clean
package` produce il JAR aggiornato. Pronto per il test end-to-end su
`nomad-test-vault` (FASE 0-1 della roadmap e2e in corso).
