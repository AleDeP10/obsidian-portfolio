# ObsidianSync — Report Milestone 1

## Obiettivi Raggiunti

- Definita l'architettura di sincronizzazione: Git-based, innescata dal logon/logoff Windows tramite Task Scheduler
- Definita la struttura del progetto: Maven, fat JAR, risorse copiate a fianco dell'artefatto
- Implementato `Main.java`: parsing argomenti CLI, caricamento properties, dispatch delle operazioni
- Implementato `GitService.java`: `push`, `pull`, `autosave` tramite `ProcessBuilder`
- Implementato `LogService.java`: logging a livelli, append-only, thread-safe tramite `synchronized`
- Configurato `config.dev.properties` e `.gitignore` con esclusione delle credenziali
- Creati `ObsidianSync.bat` (generico) e `ObsidianSyncPush.bat` (shortcut specializzato)
- Risolti autonomamente multipli conflitti Git e problemi di configurazione del repository
- Scaffolding di `AutosaveScheduler` e `SyncOrchestrator` — implementazione rimandata alla Milestone 2

---

## Problematiche Affrontate

| Problema                                         | Risoluzione                                                                                                     |
| ------------------------------------------------ | --------------------------------------------------------------------------------------------------------------- |
| GitHub 2FA bloccata sul vecchio telefono         | Identificata la causa radice (segreto TOTP legato al vecchio device); risoluzione demandata all'accesso da casa |
| `git branch` non mostrava nulla dopo `git init`  | Compreso che i branch senza commit non vengono listati; proceduto con il primo commit                           |
| Errori ripetuti `unrelated histories`            | Applicato il flag `--allow-unrelated-histories` in modo consistente                                             |
| `.idea/workspace.xml` bloccava il merge          | Usato `git rm --cached -r .idea/` per togliere dal tracking i file IDE; aggiornato `.gitignore`                 |
| `remote rejected` sulla cancellazione del branch | Identificato il lock del branch di default su GitHub; risolto da interfaccia Settings                           |
| `merge --abort` falliva per file non staged      | Applicato `git reset --hard HEAD` per pulire l'indice                                                           |
| Finestra `.bat` che si chiudeva immediatamente   | Aggiunto `pause` dopo l'invocazione del JAR                                                                     |
| `FileWriter` sovrascriveva il file di log        | Aggiunto il flag `append=true` al costruttore di `FileWriter`                                                   |
| Catch `FileNotFoundException` irraggiungibile    | Compresa la gerarchia delle eccezioni; consolidato in un unico catch `IOException`                              |
| Try/catch duplicato nei tre case dello switch    | Refactor: try/catch unificato all'esterno dello switch                                                          |

---

## Cheat-Sheet Acquisiti

### Git
- `git rm --cached -r <path>` — rimuove dal tracking senza cancellare i file locali
- `git pull --allow-unrelated-histories` — unisce branch senza antenato comune
- `git pull -X theirs` — risolve i conflitti preferendo il remoto
- `git stash / stash pop` — mette da parte e ripristina le modifiche non committate
- `git push origin --delete <branch>` — cancella un branch remoto
- `.gitignore` agisce solo sui file non ancora tracciati — usare `git rm --cached` per i già tracciati

### Windows / Batch
- `%1`, `%2` — argomenti posizionali nel `.bat`
- `exit /b` vs `exit` — esce dallo script vs chiude la finestra del terminale
- `call` — invoca un altro `.bat` e restituisce il controllo al chiamante
- `pause` — tiene aperta la finestra in attesa di un tasto
- `AppData` è nascosta per default — abilitare da Visualizza in Explorer
- Task Scheduler: trigger su logon/logoff/schedule; azione punta a `java -jar`

### Java
- `ProcessBuilder` — lancia processi esterni; `.directory()` imposta la working directory
- `redirectErrorStream(true)` — unifica stdout e stderr in un unico stream
- `process.waitFor()` — va chiamato **dopo** la lettura dello stream per evitare deadlock
- `synchronized` su metodo d'istanza — acquisisce il lock su `this`, serializza le chiamate concorrenti
- `new FileWriter(file, true)` — apre in modalità append
- `LocalDateTime.now().format(DateTimeFormatter.ofPattern(...))` — generazione timestamp
- `"pattern".formatted(args)` — interpolazione moderna (Java 15+)
- `Thread.currentThread().interrupt()` — ripristina il flag di interruzione dopo aver catturato `InterruptedException`
- `Runtime.getRuntime().addShutdownHook(new Thread(...))` — esegue codice alla chiusura della JVM

---

## Valutazione Personal Trainer

### Punteggio Complessivo: 8.4 / 10

| Dimensione | Punteggio | Note |
|---|---|---|
| Velocità di acquisizione know-how | 8.5/10 | Concetti spiegati una volta sola applicati subito e correttamente. Git internals, process management Java, thread safety — assorbiti on the fly senza dover tornare indietro. |
| Velocità di delivery | 7.5/10 | Buon ritmo generale. Attrito significativo da problemi ambientali (2FA GitHub, conflitti Git, interferenze IDE) non imputabili a lacune tecniche. |
| Impegno | 9.0/10 | Ha affrontato ogni ostacolo senza perdere il filo. Ha posto le domande giuste al momento giusto. Non ha mai saltato i passaggi né chiesto la risposta prima di tentare. |
| Consapevolezza della qualità del codice | 8.5/10 | Ha identificato e corretto proattivamente inconsistenze (exit /b, ordine dei catch, try/catch duplicato). Ha lasciato TODO significativi invece di ignorare i punti aperti. Il refactor finale del Main dimostra maturità progettuale. |

### Note del Trainer

Il candidato dimostra un istinto solido per la **delivery incrementale e verificabile** — costruisce lo scheletro prima della logica, valida ogni layer prima di procedere. L'abitudine di chiedersi *perché* una soluzione funziona, invece di accettarla passivamente, è un moltiplicatore di crescita che si vedrà sempre di più nelle milestone successive.

**Aree su cui lavorare nella Milestone 2:**
- Tenere `Main` snello — `SyncOrchestrator` deve assorbire la complessità crescente
- Validazione delle properties prima dell'istanziazione dei servizi: piccolo sforzo, alto valore difensivo
- Introdurre i primi unit test su `LogService` e `GitService` — la copertura parte adesso o non parte mai

Il ritmo è quello giusto. Le fondamenta sono pulite.
