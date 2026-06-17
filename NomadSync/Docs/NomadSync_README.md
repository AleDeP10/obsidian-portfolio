# NomadSync

Uno strumento Java leggero e cross-platform che mantiene una o più cartelle
sincronizzate su più macchine usando Git e GitHub — senza abbonamenti.

Qualsiasi cartella con un repository Git può essere gestita: appunti,
file di configurazione, progetti, dotfile, o vault di strumenti come Obsidian,
Logseq e simili. NomadSync supporta più cartelle indipendenti ("vault"), ciascuna
mappata su un proprio repository GitHub e, opzionalmente, con credenziali dedicate.

---

## Requisiti

- Java 21 o superiore
- Git installato e disponibile nel PATH (o configurato in `config.properties`)
- Un repository GitHub privato per ogni vault
- Windows, macOS/Linux

---

## Installazione

1. Copia la cartella `target/` nella posizione desiderata (es. `C:\tools\nomadsync\`)
2. Copia `config.properties.template` in `config.properties` e compilalo
3. Copia `vaults.json.template` in `vaults.json` e registra i tuoi vault
4. Aggiungi la cartella di installazione al PATH di sistema per usare gli script
   da qualsiasi terminale

---

## Configurazione

### `config.properties` — valori globali

```properties
# Git
git.executable=git
git.remote=origin
git.branch=main
git.name=Il Tuo Nome
git.email=tuaemail@example.com
git.username=tuo-username-github
git.token=ghp_...

# Percorsi
path.vaults=./vaults.json
path.backup=./backups
path.conflicts=./remote-conflicts

# Log
log.writers=console,file
log.path=logs/nomadsync.log
log.level=INFO

# Autosave
autosave.interval.minutes=15

# Editor per i commit manuali (opzionale)
commit.editor=notepad++
```

### `vaults.json` — vault registrati

```json
[
  {
    "id": "uuid-generato-automaticamente",
    "owner": "TuoUsername",
    "name": "nome-repository",
    "path": "C:\\Users\\tu\\vault",
    "gitToken": "ghp_token_specifico_per_questo_vault"
  }
]
```

Tutti i campi `git*` in `vaults.json` sono **opzionali**: se assenti, vengono
usati i valori globali di `config.properties`. Utile per vault di altri utenti
o repository legacy con branch `master` invece di `main`.

> **Sicurezza**: `vaults.json` non è committato — può contenere credenziali.
> Solo `vaults.json.template` viene committato. Il token è memorizzato nel
> `.git/config` locale del vault (mai committato) e non compare mai nei log.

---

## Uso

### Operazioni principali

```bat
REM Sincronizzazione completa (pull + risoluzione conflitti + push)
NomadSyncSync.bat --vault=nome-vault

REM Pull all'avvio sessione (broadcast su tutti i vault se --vault assente)
NomadSyncPull.bat

REM Push a fine sessione
NomadSyncPush.bat

REM Commit locale con messaggio personalizzato (apre l'editor)
NomadSyncCommit.bat --vault=nome-vault

REM Mostra git status
NomadSyncStatus.bat
NomadSyncStatus.bat --vault=nome-vault
```

### Gestione configurazione

```bat
REM Aggiorna token globale (config.properties)
NomadSyncConfig.bat --git.token=ghp_nuovo_token

REM Aggiorna token per un vault specifico (vaults.json)
NomadSyncConfig.bat --vault=nome-vault --git.token=ghp_token_vault

REM Cambia branch per un repository legacy
NomadSyncConfig.bat --vault=vault-legacy --git.branch=master
```

### Risoluzione vault

`--vault` accetta il nome del vault o il repoSlug completo:

```bat
NomadSyncSync.bat --vault=public-vault
NomadSyncSync.bat --vault=TuoUsername/public-vault
```

Se più vault condividono lo stesso nome (proprietari diversi), NomadSync richiede
il repoSlug completo:

```
vault name 'public-vault' is ambiguous.
Matches: AleDeP10/public-vault, Belmani/public-vault.
Use --vault=<owner>/<name>
```

---

## Commit interattivo

`NomadSyncCommit` apre l'editor di testo configurato, attende la chiusura e usa
il testo salvato come messaggio di commit.

- **Salva e chiudi** → commit eseguito
- **Chiudi senza salvare** → operazione annullata, nessun commit

Editor usato (in ordine di priorità):
1. Flag `--editor` sulla riga di comando
2. `commit.editor` in `config.properties`
3. Variabile d'ambiente `EDITOR`
4. `notepad` su Windows, `nano` su Unix

---

## Risoluzione conflitti

In caso di conflitto durante `sync`:

1. NomadSync crea uno snapshot FIFO del vault in `backups/`
   (max 3 snapshot per vault)
2. Applica `git pull -X ours` — la versione locale prevale
3. Salva la versione remota di ogni file in conflitto in `remote-conflicts/`
   per revisione manuale

Le directory di backup e conflitti usano il formato `<owner>_<name>_<timestamp>`,
garantendo separazione tra vault con lo stesso nome ma proprietari diversi.

---

## Modalità daemon

Per default NomadSync termina automaticamente al completamento delle operazioni
(modalità one-shot). Passa `--daemon` per tenere il processo vivo indefinitamente
(per l'uso con la Tray, in arrivo in una release futura):

```bat
NomadSync.bat pull --daemon
```

---

## Task Scheduler (Windows)

Per automatizzare pull al logon e push al logoff:

```
Trigger: Al logon utente  → Azione: NomadSyncPull.bat
Trigger: Alla disconnessione → Azione: NomadSyncPush.bat
```

Senza `--vault`, entrambi operano su tutti i vault registrati.

---

## Troubleshooting

### `Repository not found` su pull/push

1. Verifica che il repository esista su GitHub con il nome esatto indicato in `vaults.json`
2. Controlla che il token non sia scaduto:
   GitHub → Settings → Developer settings → Personal access tokens
3. Verifica che il token abbia lo scope `repo` (full control of private repositories)
4. Riesegui il bootstrap: `NomadSyncConfig.bat --vault=<nome> --git.token=ghp_nuovo`
5. Controlla l'URL remoto: `git -C <percorso-vault> remote get-url origin`
   — deve iniziare con `https://ghp_...@github.com/`

### Il processo non termina dopo pull/push

Senza `--daemon`, il processo termina automaticamente quando tutte le code sono
svuotate. Se si blocca, probabilmente sta aspettando un'operazione di rete (retry
con backoff). Controlla i log per voci `Network error`. Il processo terminerà
dopo al massimo 3 retry (~3.5 minuti).

### Token visibile nei log

NomadSync non logga mai il token. Se compare in un file di log, è stato passato
come argomento da uno script esterno — rivedi i tuoi wrapper e usa
`NomadSyncConfig.bat --git.token=...` invece.

### Output di `git status` su una sola riga

Assicurati di usare `NomadSyncStatus.bat` (che chiama `Main` con `status`) e
non di chiamare `git status` direttamente da uno script che rimuove i newline.
