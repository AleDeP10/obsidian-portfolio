# Multi-threading in Java e Thread Daemon

## 1. Cos'è un thread

Un thread è un filo di esecuzione indipendente all'interno dello stesso processo. Più thread condividono la stessa memoria heap — possono leggere e scrivere le stesse variabili, il che li rende potenti e pericolosi allo stesso tempo.

```
Processo JVM
├── Thread main          ← il tuo main()
├── Thread GC            ← garbage collector (daemon)
├── Thread finalizer     ← daemon
└── Thread seq-log-writer ← il nostro (daemon)
```

---

## 2. Thread normale vs Thread daemon

### Thread normale (non-daemon)

- La JVM **non si chiude** finché almeno un thread normale è vivo.
- Esempio: il thread `main`, i thread di un server HTTP, i thread dell'orchestratore.

### Thread daemon

- La JVM **si chiude senza aspettarlo** quando tutti i thread normali sono terminati.
- Viene **terminato forzatamente** quando la JVM esce — senza cleanup garantito.
- Esempi classici: Garbage Collector, finalizer, log asincroni, heartbeat.

```java
Thread t = new Thread(this::workerLoop, "seq-log-writer");
t.setDaemon(true);  // deve essere chiamato PRIMA di start()
t.start();
```

---

## 3. Perché seq-log-writer è un daemon

### Scenario senza daemon (thread normale):

```
main() termina
orchestrator.stop() chiamato
LogService.close() chiamato
worker.join() → aspetta che la coda si svuoti ✓
worker termina
JVM si chiude ✓
```

Fin qui funziona. Ma cosa succede se `close()` NON viene chiamato? Crash, kill -9, eccezione non gestita nel shutdown hook:

```
main() crasha
shutdown hook non gira
seq-log-writer è ancora vivo con eventi in coda
JVM NON si chiude → processo zombie per sempre ✗
```

### Scenario con daemon:

```
main() crasha
JVM nota: tutti i thread normali sono morti
seq-log-writer è daemon → viene terminato forzatamente
JVM si chiude ✓
```

**Il daemon è un'assicurazione contro lo zombie.** Nel caso normale, `close()` fa drain ordinato e lo termina pulitamente. Nel caso patologico, la JVM lo uccide comunque.

---

## 4. Il pattern Poison Pill per lo shutdown ordinato

`null` non funziona con `LinkedBlockingQueue` (lancia NPE). La soluzione è una stringa sentinel:

```java
private static final String POISON_PILL = "__SHUTDOWN__";

// producer (close()):
queue.clear();              // svuota eventi non ancora inviati
queue.offer(POISON_PILL);   // segnala shutdown
worker.join(5000);          // aspetta max 5 secondi
if (worker.isAlive()) worker.interrupt(); // forza se necessario

// consumer (workerLoop()):
while (true) {
    String event = queue.take(); // bloccante
    if (POISON_PILL.equals(event)) break; // shutdown ordinato
    // ... POST a Seq
}
```

**Perché `queue.clear()` prima del poison pill?** Se la coda ha 1000 eventi e Seq non è raggiungibile, il drain richiederebbe minuti. `clear()` + poison pill dà priorità allo shutdown veloce rispetto alla consegna degli eventi. Scelta di design: preferisci velocità di shutdown o garanzia di consegna? Per un log locale la velocità vince.

---

## 5. Thread safety — i problemi classici

### Race condition

Due thread leggono e scrivono la stessa variabile senza coordinazione:

```java
// SBAGLIATO — i++ non è atomico (leggi, incrementa, scrivi = 3 operazioni)
int counter = 0;
thread1: counter++;  // legge 0, scrive 1
thread2: counter++;  // legge 0 (non ancora aggiornato!), scrive 1
// risultato: 1 invece di 2
```

### synchronized

Garantisce che un solo thread alla volta esegua il blocco:

```java
private synchronized void log(...) { ... }
// oppure
synchronized (this) { ... }
```

### BlockingQueue — thread safety by design

`LinkedBlockingQueue` è thread-safe internamente — nessun `synchronized` necessario per `offer()` e `take()`. È progettata esattamente per il pattern producer-consumer che usiamo in `SeqHttpLogWriter`.

---

## 6. Il nostro caso — SeqHttpLogWriter

```
Thread LogService (main o orchestrator)
    │
    │ write() → queue.offer(clef)   ← non-blocking, velocissimo
    │
    ▼
LinkedBlockingQueue<String>  ← thread-safe, max 1000 eventi
    │
    │ queue.take()                  ← bloccante, aspetta se vuota
    ▼
Thread seq-log-writer (daemon)
    │
    │ POST HTTP a Seq
    ▼
Seq server
```

**Perché non fare il POST direttamente in `write()`?** `write()` è chiamato da `log()` che è `synchronized` — blocca tutti gli altri thread che vogliono loggare. Un POST HTTP può durare 200ms. Terrebbero il lock per 200ms → tutti i thread dell'applicazione in attesa → deadlock percettivo.

Con la coda: `write()` fa `offer()` in <1ms, rilascia il lock immediatamente. Il POST avviene in background senza bloccare nessuno.

---

## 7. Riepilogo pattern usati in NomadSync

| Componente              | Pattern                                 | Motivazione                                       |
| ----------------------- | --------------------------------------- | ------------------------------------------------- |
| `LogService.log()`      | `synchronized`                          | Due thread max (main + autosave), overhead minimo |
| `FileLogWriter.write()` | `synchronized`                          | Accesso esclusivo al file                         |
| `SeqHttpLogWriter`      | `BlockingQueue` + daemon thread         | POST HTTP non-blocking                            |
| `SyncOrchestrator`      | `PriorityBlockingQueue` + worker thread | Serializzazione operazioni Git per vault          |
| `SocketServer`          | Due thread (receiver + router)          | I/O asincrono su socket                           |