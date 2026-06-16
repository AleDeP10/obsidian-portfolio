# COMMIT_MANUAL — pseudo-implementazione

## Contesto

`NomadSyncCommit` permette al developer di creare un commit locale con un
messaggio significativo, scritto a mano tramite editor di testo — distinto da
`AUTOSAVE` (messaggio automatico con timestamp) e da `SYNCHRONIZE` (commit +
pull + push). Nessun push: è un checkpoint locale puro.

---

## 1. EventType — nuova entry e priorità

`COMMIT_MANUAL` è un'operazione esplicita dell'utente, come `SYNCHRONIZE`, ma
locale-only — non compete per la rete. Priorità proposta: tra `SYNCHRONIZE` (2)
e `PUSH_LOGOFF` (3), oppure stesso livello di `AUTOSAVE` essendo entrambi
commit-only. Suggerimento: stessa priorità di `AUTOSAVE` (4), perché
condividono la natura "non bloccante, nessuna operazione di rete".

```java
/** Triggered explicitly by the user via NomadSyncCommit — local commit only,
 *  with a user-provided message. Never pushes. */
COMMIT_MANUAL(4),

/** Triggered periodically by the scheduler. Tolerant and deferrable. */
AUTOSAVE(4);
```

[NOTA] due eventi con stessa priorità: la PriorityQueue li ordina FIFO a parità
di priorità (comportamento di `PriorityBlockingQueue` con elementi equal-priority
— dipende dal `Comparator`, verificare se serve tie-break esplicito su timestamp,
già presente in `SyncEvent` per latest-wins).

---

## 2. SyncEvent — nuovo campo `message`

`COMMIT_MANUAL` necessita di portare il messaggio di commit nell'evento.
`AUTOSAVE`/`PULL_LOGON`/ecc. non lo usano — campo opzionale, `null` per gli
altri tipi.

```java
public class SyncEvent {
    private final EventType type;
    private final String vaultId;
    private final String message;  // ← nuovo, null per eventi che non lo usano
    // ... retryCount, retryDelay, timestamp esistenti

    // costruttore esistente invariato (message = null)
    public SyncEvent(EventType type, String vaultId) {
        this(type, vaultId, null);
    }

    // nuovo costruttore
    public SyncEvent(EventType type, String vaultId, String message) {
        // ...
        this.message = message;
    }

    public String getMessage() { return message; }
}
```

---

## 3. GitService — nuovo metodo

`commitLocal(vaultPath, message)` esiste già (usato da `PUSH_LOGOFF`/`AUTOSAVE`
con messaggi generati). Nessuna nuova API necessaria — `COMMIT_MANUAL` lo
chiama con il messaggio dell'utente invece di uno generato.

---

## 4. SyncOrchestrator — nuovo case

```java
case COMMIT_MANUAL -> {
    if (gitService.hasUncommittedChanges(vaultPath)) {
        String message = event.getMessage();
        if (message == null || message.isBlank()) {
            logService.warn("COMMIT_MANUAL: empty message, using fallback");
            message = "manual commit " + LocalDateTime.now();
        }
        gitService.commitLocal(vaultPath, message);
    } else {
        logService.info("No changes detected, skipping manual commit.");
    }
}
```

[NOTA] simmetrico ad AUTOSAVE — stessa guard `hasUncommittedChanges`, nessun push.

---

## 5. Main — nuovo case CLI + lettura messaggio da file

Il messaggio arriva via file temporaneo scritto dall'editor (vedi script),
passato come argomento posizionale dopo `vaultId`.

```
java -jar NomadSync.jar commit config.properties <vaultId> <messageFilePath>
```

```java
case "commit" -> {
    if (args.length < 4) {
        logService.error("commit requires a message file path as 4th argument");
        System.exit(1);
    }
    String id = resolveTargetId(vaults, targetVaultId, logService);
    String messagePath = args[3];
    String message;
    try {
        message = Files.readString(Path.of(messagePath)).strip();
    } catch (IOException e) {
        logService.error("Unable to read commit message file: " + messagePath, e);
        System.exit(1);
        return;
    }
    if (message.isEmpty()) {
        logService.warn("Empty commit message — aborting, no commit created.");
        System.exit(0);
    }
    routeToVault(queues, vaults,
            new SyncEvent(EventType.COMMIT_MANUAL, id, message), logService);
}
```

[ATTENZIONE] `commit` richiede `vaultId` esplicito (non opzionale come
pull/push) — un commit locale senza sapere su quale vault operare non ha
senso di default-to-first silenzioso. Se `targetVaultId` è `null`,
`resolveTargetId` fa comunque fallback al primo vault con un warning —
valutare se per `commit` questo comportamento va invece bloccato con errore
esplicito, dato il rischio di committare sul vault sbagliato.

---

## 6. Script NomadSyncCommit — editor + invocazione

Lo script:
1. crea un file temporaneo vuoto
2. apre l'editor di default (`%EDITOR%`/`$EDITOR`, fallback `notepad`/`nano`)
3. attende la chiusura
4. se il file non è vuoto, chiama `NomadSync.bat commit config.properties <vaultId> <tempfile>`
5. cancella il file temporaneo

Vedi implementazione negli script `.bat`/`.sh` allegati.
