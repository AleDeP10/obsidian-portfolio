# TTR — Seq: logging strutturato per NomadSync

---

## Cos'è Seq

Seq è un server di logging strutturato: riceve eventi in formato CLEF (Compact
Log Event Format, JSON-based) via HTTP POST, li indicizza, e offre una UI web
per cercarli, filtrarli e correlarli. A differenza di un file di log piatto,
ogni campo di un evento (livello, `repoSlug`, messaggio, timestamp, eccezioni)
è interrogabile separatamente.

In NomadSync, `SeqHttpLogWriter` (DTR-035) è uno dei writer che `LogService`
può attivare via `log.writers=console,file,seq`. Ogni vault scrive sullo stesso
stream Seq, taggato con il proprio `repoSlug` — questo rende Seq il punto
naturale per osservare il comportamento multi-vault in tempo reale.

---

## Setup Docker

Seq gira come container. Comando di avvio base:

```powershell
docker run -d --name seq `
  -e ACCEPT_EULA=Y `
  -p 5341:5341 `
  -p 80:80 `
  -v C:\seq-data:/data `
  datalust/seq:latest
```

**Porte**:
- `5341` — Ingestion API (`/api/events/raw`) e API REST, usata anche dalla UI
  per le chiamate dati
- `80` — UI web (servita su HTTP, **non HTTPS** — vedi nota sotto)

**Volume**: `C:\seq-data:/data` persiste lo stream degli eventi e il metastore
(utenti, dashboard, query salvate) tra restart del container.

**Verifica avvio** — `docker logs seq` deve mostrare:

```
[INF] Seq 2025.2.16202 running on OS Ubuntu 22.04.5 LTS
[INF] Seq listening on ["http://localhost/", "https://localhost/", "http://localhost:5341/", "https://localhost:45341/"]
[INF] Ingestion enabled
```

---

## Accesso alla UI — la trappola HTTPS

**Problema riscontrato**: Seq risponde sia su HTTP che HTTPS sulla stessa
porta logica, ma il certificato HTTPS è auto-generato e non valido per
`localhost` nel contesto Docker — il browser lo rifiuta con
`ERR_SSL_PROTOCOL_ERROR`.

**Soluzione**: usa sempre `http://`, mai `https://`, sulla porta `80`:

```
http://localhost/#/login
```

Non `https://localhost:80` — quello fallisce. La UI è una SPA, le route vivono
sotto `#/` (hash routing): `#/login`, `#/events`, `#/settings`, ecc.

---

## Login e primo accesso

Se il metastore (`C:\seq-data\Documents`) contiene già dati da un'installazione
precedente, Seq presenta direttamente la schermata di login con un account
admin preesistente — non una schermata di "crea il primo admin".

Se non ricordi le credenziali e il metastore è vuoto/nuovo, Seq mostra invece
"Create the initial administrator account" al primo accesso.

**Reset completo** (solo se i log storici non servono): stoppa il container,
rinomina `Documents/` e `Stream/` dentro `C:\seq-data` per forzare un metastore
vergine, riavvia — riparte da zero con la schermata di creazione admin.

---

## Configurazione lato NomadSync

In `config.properties`:

```properties
log.writers=console,file,seq
seq.url=http://localhost:5341/api/events/raw
```

`SeqHttpLogWriter` (DTR-035) formatta ogni evento come CLEF e lo invia via
POST a questo endpoint, con consegna asincrona tramite coda interna — non
blocca mai il thread chiamante. Se Seq non è raggiungibile, gli eventi vengono
scartati con warning su `stderr`, il servizio continua normalmente.

**Test di raggiungibilità manuale** (PowerShell):

```powershell
Invoke-WebRequest -Uri "http://localhost:5341/api/events/raw" `
  -Method POST `
  -ContentType "application/vnd.serilog.clef" `
  -Body '{"@t":"2026-06-14T20:50:00Z","@mt":"test event","@l":"Information"}' `
  -UseBasicParsing
```

Risposta attesa: `201 Created`, body `{"MinimumLevelAccepted":null}`.

> Nota: `Invoke-WebRequest` senza `-UseBasicParsing` chiede conferma per motivi
> di sicurezza prima di processare la risposta — sempre aggiungere il flag in
> script/automazioni.

---

## Usare la UI per il debug multi-vault

**Schermata Events** (default dopo login): lista cronologica di tutti gli
eventi, più recenti in alto. La query box in alto accetta sintassi simile a SQL
su proprietà strutturate.

**Filtrare per vault** — ogni evento NomadSync porta `repoSlug`:

```
repoSlug = 'AleDeP10/nomad-test-vault'
```

**Filtrare per livello** — usa i Signal predefiniti nella colonna destra
(`Errors`, `Warnings`) o scrivi:

```
@Level = 'Error'
```

**Combinare filtri**:

```
repoSlug = 'AleDeP10/nomad-test-vault' and @Level = 'Warning'
```

**Range temporale** — il selettore "Last 1d" in alto a destra della lista
eventi permette di restringere/allargare la finestra. Utile durante un test
e2e per isolare solo gli eventi dell'ultima esecuzione.

**Live tail** — l'icona di sincronizzazione/refresh accanto al pulsante play
attiva l'aggiornamento automatico: utile per osservare gli eventi
`AutosaveScheduler: publishing` → `Broadcaster` → `Performing AUTOSAVE` su
tutti i vault in tempo reale durante il test e2e.

---

## Pattern di verifica per il test e2e

Durante FASE 1-4 della roadmap e2e, Seq permette di confermare visivamente:

- **FASE 1**: un evento `Performing PULL_LOGON` seguito da `PULL_LOGON
  completed` per il `repoSlug` del vault testato
- **FASE 3**: la sequenza completa `Performing SYNCHRONIZE` → eventuali log di
  conflitto (`Auto-merging`, backup creato) → `SYNCHRONIZE completed`
- **FASE 4**: con autosave a 1 minuto, una riga `AutosaveScheduler: publishing`
  ogni 60s, seguita da N righe `Performing AUTOSAVE` (una per vault registrato)
  — il numero di righe per ciclo corrisponde esattamente al numero di vault in
  `vaults.json`
- **FASE 5**: allo shutdown, sequenza `AutosaveScheduler: stopped cleanly` →
  `Broadcaster: stopped` → uno shutdown log per orchestrator

Se un evento attivo manca da Seq ma è presente nel file di log locale,
verificare prima la raggiungibilità di `seq.url` — `SeqHttpLogWriter` non
blocca, quindi un Seq irraggiungibile non causa errori visibili ma solo
assenza di eventi.
