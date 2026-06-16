# GRM — Milestone 8: Backend Refactoring + TrayManager

---

## Contesto

Il backend di ObsidianSync è completo e testato (61 test, 0 failure). Questa milestone introduce il refactoring necessario per supportare la nuova conflict strategy verificata sul campo, e implementa l'intera interfaccia tray che rende il tool utilizzabile da un utente non tecnico.

Componenti UI di M5: `TrayIcon`, `ContextMenu`, `VaultSwitcherPanel`, `ToastNotification`. `MainWindow` (JavaFX) è rimandato a M6.

---

## Obiettivi

|#|Obiettivo|Criterio di accettazione|
|---|---|---|
|1|`SYNCHRONIZE` sostituisce `PULL_MANUAL` e `PUSH_MANUAL`|`EventType` aggiornato, suite verde|
|2|`GitService.synchronize()` — conflict strategy completa|algoritmo FIFO backup + `-X ours` + remote-conflicts implementato e testato|
|3|`SyncOrchestrator` aggiornato|switch gestisce `SYNCHRONIZE`, casi obsoleti rimossi|
|4|`VaultService` — unicità vault.name|`VaultException` su duplicato, validazione all'avvio|
|5|`TrayIcon`|quattro stati visivi, click sx → `SYNCHRONIZE`, click dx → ContextMenu, tooltip|
|6|`ContextMenu`|struttura a tre sezioni, tutte le azioni funzionanti|
|7|`VaultSwitcherPanel`|sottomenu con lista vault, spunta corrente, checkbox Save on selection|
|8|`ToastNotification`|tre scenari: success, conflitto azionabile, fallimento rete|
|9|`Main` modalità tray|carica vaults.json, avvia SocketServer e TrayManager|
|10|Task Scheduler Windows|3 task configurati e testati manualmente|
|11|Test e2e manuale|ciclo completo logon/logoff su vault reale, conflitto intenzionale|
|12|Release 1.0.0|`git flow release finish 1.0.0`, tag pushato|

---

## Ordine implementativo

### Layer 1 — Refactoring backend

**Step 1 — `EventType`** Rimuovere `PULL_MANUAL` e `PUSH_MANUAL`. Aggiungere `SYNCHRONIZE` con priorità 2. Da eseguire per primo — impatta tutta la suite. Verde su `mvn test` prima di proseguire.

Scala di priorità aggiornata:

| Priorità | Evento        |
| -------- | ------------- |
| 1        | `PULL_LOGON`  |
| 2        | `SYNCHRONIZE` |
| 3        | `PUSH_LOGOFF` |
| 4        | `AUTOSAVE`    |

**Step 2 — `GitService.synchronize()`**

```
SYNCHRONIZE
│
├─ SE presenti modifiche locali (git status --porcelain != vuoto)
│   └─ git add -A
│   └─ git commit -m "sync: local changes before pull"
│
├─ git pull
│   ├─ SUCCESS (exit code 0) → git push → fine
│   └─ CONFLICT (exit code != 0)
│       ├─ git merge --abort   (ignorare exit code != 0)
│       ├─ FIFO backup → backups/<vault-name>_{timestamp}/  (max 3 snapshot)
│       ├─ git pull -X ours --no-edit
│       ├─ per ogni riga "Auto-merging" nello stdout
│       │   └─ git --no-pager show FETCH_HEAD:<filepath>
│       │       → remote-conflicts/<vault-name>_{timestamp}/<filename>
│       ├─ git push
│       └─ restituisce lista file conflittati al chiamante
```

Evidenze dal campo:

- `git merge --abort` restituisce errore se non c'è merge in corso — ignorare corretto
- `-X ours` richiede modifiche locali committate prima
- `MERGE_HEAD` non esiste dopo il merge — usare `FETCH_HEAD`
- `--no-pager` obbligatorio su `git show` per evitare apertura `less`
- `--no-edit` obbligatorio su `git pull -X ours` per evitare apertura editor

**Step 3 — `SyncOrchestrator`** Switch aggiornato: `SYNCHRONIZE` chiama `gitService.synchronize()`, che restituisce la lista file conflittati. Se non vuota, passata al `NotificationHook` per il toast azionabile. Rimozione case `PULL_MANUAL`/`PUSH_MANUAL`.

**Step 4 — `VaultService`** `create()` e `update()` lanciano `VaultException("duplicated vault name: " + name)` se il nome è già presente. Validazione all'avvio di tutti i nomi in `vaults.json`.

---

### Layer 2 — TrayManager

**Step 5 — `TrayIcon`**

Implementazione: `SystemTray.getSystemTray()` + `TrayIcon` AWT.

|Stato|Icona|Quando|
|---|---|---|
|Idle|verde statica|nessuna operazione in corso|
|Syncing|animata / rotante|pull o push in esecuzione|
|Error|rossa|ultimo sync fallito dopo 3 retry|
|Conflict|arancione|file in `remote-conflicts/` in attesa|

Interazioni:

- **Click sinistro** → `SYNCHRONIZE` sul vault corrente via `SocketServer.publish()`
- **Click destro** → apre `ContextMenu`
- **Tooltip** → "Last sync: X minutes ago"

**Step 6 — `ContextMenu`**

`PopupMenu` AWT. Principio guida: zero decisioni cognitive.

```
┌─────────────────────────────┐
│  ● Personal             ▶   │  → VaultSwitcherPanel sottomenu
├─────────────────────────────┤
│  Sync current vault         │  → SYNCHRONIZE su vault corrente
│  Sync all vaults            │  → SYNCHRONIZE broadcast
│  Pull current vault         │  → PULL_LOGON su vault corrente
├─────────────────────────────┤
│  Last sync: 3 min ago       │  (label non cliccabile)
│  View log                   │  → tab Log in MainWindow (M6)
├─────────────────────────────┤
│  Open Dashboard             │  → MainWindow (M6, placeholder)
│  Open vault folder          │  → Desktop.getDesktop().open(vault.path)
├─────────────────────────────┤
│  Exit                       │
└─────────────────────────────┘
```

**Step 7 — `VaultSwitcherPanel`**

`Menu` AWT annidato come sottomenu della prima voce del ContextMenu. È il componente con più responsabilità nel menu rapido: aggiorna `current-vault.json`, aggiorna il tooltip della `TrayIcon`, e opzionalmente pubblica `SYNCHRONIZE`.

```
┌─────────────────┐
│ ✓ Personal      │  ← vault corrente con spunta
│   Work          │
│   Research      │
│ ─────────────── │
│ ☐ Save on sel.  │  ← sempre visibile, fuori dallo scroll
└─────────────────┘
```

Save on selection: se attivo, dopo aggiornamento `current-vault.json` pubblica automaticamente `SYNCHRONIZE` sul vault appena selezionato.

**Step 8 — `ToastNotification`**

**Success** — toast AWT nativo, auto-dismiss:

```java
trayIcon.displayMessage("ObsidianSync", "✓ Personal sincronizzato", MessageType.INFO)
```

**Conflitto risolto** — `JDialog` persistente:

```
⚠ Sync completato con conflitti

File aggiornati dal remoto:
  • Gabriela_experiment.md
  • note-lavoro.md

La tua versione locale è stata mantenuta.

[ Apri versioni remote ]

Hai perso delle modifiche? → Apri backup locale
```

- "Apri versioni remote" → `Desktop.getDesktop().open(remote-conflicts/<vault>_{timestamp}/)`
- "Apri backup locale" → `Desktop.getDesktop().open(backups/<vault>_{timestamp}/)`
- Apre lo snapshot più recente, non la root

**Fallimento rete** — `JDialog` persistente, solo eventi priorità 1:

```
✗ Pull non riuscito

Nessuna connessione disponibile dopo 3 tentativi.
Il vault potrebbe non essere aggiornato.

[ OK ]
```

---

### Layer 3 — Integrazione e release

**Step 9 — `Main` modalità tray**

```
Main.main()
  → carica config.properties
  → carica vaults.json via VaultService
  → costruisce SocketServer
  → registra tutti i vault
  → costruisce TrayManager(socketServer, vaultService)
  → trayManager.start()
  → socketServer.start()
  → blocca su Object.wait()
```

**Step 10 — Task Scheduler Windows**

- `Pull@Logon` → `java -jar ObsidianSync.jar pull`
- `Push@Logoff` → `java -jar ObsidianSync.jar push`
- `Autosave@15min` → `java -jar ObsidianSync.jar autosave`

**Step 11 — Test e2e manuale** Ciclo completo: modifica su PC A → logoff → logon su PC B → modifiche presenti. Verifica conflitto intenzionale e toast azionabile con lista file.

**Step 12 — Release 1.0.0**

```
git flow release start 1.0.0
git flow release finish 1.0.0
git push origin main develop --tags
```

---

## Rischi

| Rischio                                                  | Probabilità | Impatto | Mitigazione                                          |
| -------------------------------------------------------- | ----------- | ------- | ---------------------------------------------------- |
| Messaggi stderr Git localizzati — whitelist non funziona | Media       | Alto    | testare sulla macchina reale prima del release       |
| Trigger logoff scatta prima del push completato          | Media       | Alto    | timeout task 2 min; verificare cronologia esecuzioni |
| `TrayIcon` AWT non supportata su JDK headless            | Bassa       | Alto    | verificare con `SystemTray.isSupported()` all'avvio  |
| Percorso JAR con spazi rompe l'invocazione .bat          | Media       | Medio   | racchiudere tutti i path tra virgolette              |
| Token GitHub scaduto rompe push al logoff                | Bassa       | Alto    | token fine-grained senza scadenza                    |