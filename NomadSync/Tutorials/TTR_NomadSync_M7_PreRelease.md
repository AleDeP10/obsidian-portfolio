# TTR — Pre-release Hardening: best practice

---

## Cos'è un hardening sprint

Un **hardening sprint** (o stabilization phase) è il periodo dedicato
esclusivamente a portare un sistema feature-complete a uno stato
release-ready, senza introdurre nuove feature — solo fix emersi dall'uso
reale, verifica sistematica, e chiusura dei gap tra "il codice compila e i
test passano" e "il sistema funziona in condizioni reali, end-to-end, su
ambienti diversi da quello di sviluppo".

È distinto dal normale ciclo di sviluppo perché il criterio di accettazione
cambia: non "la feature fa quello che doveva fare in isolamento" (coperto
dai test unitari), ma "il sistema nel suo complesso si comporta correttamente
quando tutte le parti interagiscono, su una macchina che non è quella su cui
è stato scritto il codice".

---

## Perché i test unitari non bastano

112 test green garantiscono che ogni componente, isolato e con dipendenze
mockate, si comporta secondo contratto. Non garantiscono che:

- la directory di log esista al primo avvio su una macchina pulita
  (`FileLogWriter` — bug trovato in questa sessione)
- un URL di configurazione concatenato runtime produca l'endpoint corretto
  (`SeqHttpLogWriter` — bug trovato in questa sessione)
- l'ordine di esecuzione dei test non nasconda una race condition nel codice
  di produzione (`VaultServiceTest` — bug trovato in questa sessione)
- le credenziali di sistema (Git, token) funzionino per *tutti* i repository
  configurati, non solo quello con cui si è sviluppato

Questi sono tutti bug che `mvn test` non può rivelare per costruzione — richiedono
l'esecuzione reale del JAR, su filesystem reale, contro servizi reali.

---

## Metodologia: verifica incrementale per livelli

L'approccio efficace, applicato in questa sessione, è stratificato:

**Livello 0 — il processo si avvia** — `java -jar NomadSync.jar <op> config.properties`
non lancia eccezioni al bootstrap. Se fallisce qui, il problema è in
configurazione o dipendenze, non in logica di business.

**Livello 1 — un'operazione singola, su un solo vault, senza side-effect
di rete** — `pull` su un repository già aggiornato (`Already up to date`).
Verifica: parsing argomenti, caricamento config, wiring orchestratore, logging.

**Livello 2 — un'operazione con side-effect locale** — `commit`/`autosave`
su un vault con modifiche reali. Verifica: `GitService` interagisce
correttamente col filesystem e con Git locale.

**Livello 3 — un'operazione con side-effect di rete** — `push`/`sync` su un
repository remoto reale. Verifica: credenziali, connettività, gestione errori
di rete (retry/backoff).

**Livello 4 — scenario multi-componente** — `sync` con conflitto reale,
`autosave` broadcast su più vault. Verifica: interazione tra componenti,
ordine degli eventi, side-effect cumulativi.

**Livello 5 — lifecycle completo** — avvio, esecuzione prolungata (più
cicli di autosave), shutdown pulito (Ctrl+C). Verifica: nessuna risorsa
orfana, nessun thread bloccato, log coerente dall'avvio alla chiusura.

Ogni livello che fallisce blocca l'avanzamento a quello successivo — ma il
fix di un livello non richiede di ripetere da zero i livelli precedenti se
il fix è isolato (es. il fix di `FileLogWriter` a Livello 0 non ha richiesto
di rifare il troubleshooting di `VaultServiceTest` a Livello -1/test).

---

## Il valore degli strumenti di osservabilità durante l'hardening

L'introduzione di Seq durante l'hardening non è stata solo "bello da avere" —
ha permesso di:

- vedere il flusso temporale degli eventi (`AutosaveScheduler: publishing` →
  `Queue: published` → `Performing AUTOSAVE` → `completed`) senza dover
  correlare manualmente timestamp in un file di log piatto
- filtrare per `repoSlug` quando si introdurranno più vault nel test e2e,
  isolando il comportamento di un singolo vault dal rumore degli altri
- osservare in tempo reale (live tail) il comportamento di un processo
  long-running, senza dover fare `tail -f` e interpretare a occhio

Per sistemi multi-componente con esecuzione asincrona (code, thread,
scheduler), l'osservabilità non è un lusso: è quello che rende
diagnosticabile un comportamento emergente che nessun singolo test unitario
potrebbe riprodurre.

---

## Pattern di troubleshooting emersi

**"Nothing to compile - all classes are up to date"** — se Maven riporta
questo dopo aver "applicato un fix", il file sul disco del progetto non è
stato effettivamente sostituito. Verifica sempre con un `grep` di una stringa
distintiva del fix prima di interpretare il risultato del test.

**Test che falliscono solo con `-DrunOrder=random`** — sintomo di stato
condiviso non isolato correttamente, spesso un oggetto mutabile restituito da
un metodo di factory/create e poi mutato direttamente dal test, quando invece
andrebbe copiato. L'ordine `random` di Surefire è il modo più rapido per
scovare questa classe di bug — eseguirlo regolarmente durante l'hardening,
non solo come fix puntuale.

**Messaggi di errore generici da librerie HTTP/IO** (`HTTP 404`, `Unable to
write log file`) — quasi sempre il problema è a un livello di astrazione più
basso di quanto il messaggio suggerisca: un path non normalizzato, una
directory mancante, un URL concatenato due volte. Riprodurre la chiamata
manualmente (es. `Invoke-WebRequest` per testare un endpoint HTTP) isola
se il problema è nel servizio esterno o nel codice che lo chiama.

---

## Checklist mentale prima di considerare un fix "fatto"

1. Il fix risolve la causa, non il sintomo? (es. normalizzare l'URL, non
   solo cambiare il valore in config per evitare la concatenazione)
2. Il fix è coperto da un test, o almeno non rompe quelli esistenti?
3. Il fix è stato verificato con `mvn clean` (non solo `mvn test`) per
   escludere artefatti di build stantii?
4. Il comportamento corretto è stato osservato end-to-end (log/Seq), non solo
   dedotto dall'assenza di errori?
