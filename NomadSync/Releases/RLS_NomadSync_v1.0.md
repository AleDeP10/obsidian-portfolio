# NomadSync — Release Checklist (v1.0.0)

> Da completare interamente prima di `git flow release start 1.0.0`.
> Ogni voce non spuntabile va tracciata come issue/DTR prima del merge su `main`.

---

## 1. Codice e test

- [ ] `mvn clean test` — tutti i test green
- [ ] `mvn test -DrunOrder=random` — eseguito almeno 3 volte consecutive, sempre green
      (esclude race condition latenti come quella di `VaultServiceTest`)
- [ ] `mvn clean package` — build pulita, JAR generato senza warning bloccanti
- [ ] Nessun `TODO`/`FIXME`/`[IN_REVIEW]` residuo nel codice di produzione
      (`grep -rn "TODO\|FIXME\|IN_REVIEW" src/main/java`)
- [ ] Coverage verificata per ogni classe modificata in questa milestone —
      nessuna riga di business logic non esercitata da almeno un test

---

## 2. Configurazione e sicurezza

- [ ] Nessun secret (token, password, API key) hardcoded o loggato in chiaro
      — verificare in particolare `-- listing properties --` o debug dump
      simili che stampano l'intero `Properties`
- [ ] `config.properties.template` e `vaults.json.template` aggiornati e
      coerenti con il codice corrente (nomi property, struttura JSON)
- [ ] `.gitignore` copre `config.properties`, `config.dev.properties`,
      `config.prod.properties`, `vaults.json` (solo i `.template` committati)
- [ ] Tutti i path di default (`log.path`, `path.backup`, `path.conflicts`)
      vengono creati automaticamente se assenti — nessuna directory deve
      essere creata manualmente al primo avvio

---

## 3. Esecuzione end-to-end

- [ ] **Livello 0** — `java -jar NomadSync.jar pull config.properties <vaultId>`
      su macchina pulita (cartella dedicata, non dentro il progetto IntelliJ)
- [ ] **Livello 1** — `pull` su repository aggiornato → `Already up to date`,
      nessun errore, shutdown pulito (Ctrl+C)
- [ ] **Livello 2** — `commit` con messaggio custom su vault con modifiche →
      commit locale presente in `git log`, nessun push
- [ ] **Livello 3** — `push` su repository remoto reale → commit visibile su
      GitHub
- [ ] **Livello 3** — `sync` senza conflitto → commit locale (se dirty) →
      pull → push, nessun backup/conflict creato
- [ ] **Livello 4** — `sync` con conflitto reale (due clone simulati) →
      backup FIFO creato, versione remota salvata in `remote_conflicts/`,
      versione locale preservata, push riuscito
- [ ] **Livello 4** — `autosave` con `autosave.interval.minutes` basso,
      almeno 2 cicli osservati, broadcast su tutti i vault registrati
      (verificato con ≥2 vault in `vaults.json`)
- [ ] **Livello 5** — shutdown pulito durante autosave attivo: ordine
      `scheduler.stop()` → `broadcaster.interrupt()` →
      `orchestrators.forEach(stop)` → `logService.close()` confermato in log

---

## 4. DTR-046 — unicità nome vault, a runtime

- [ ] `vaults.json` con due entry stesso `name`, owner diversi → avvio
      fallisce con `VaultException: duplicated vault name`
- [ ] `vaults.json` ripristinato → avvio normale

---

## 5. Cross-platform

- [ ] `git.executable=git` (default, risolto via PATH) verificato su almeno
      una macchina non-Windows
- [ ] Script `.sh` (`NomadSync.sh`, `NomadSyncPull.sh`, `NomadSyncPush.sh`,
      `NomadSyncSync.sh`, `NomadSyncCommit.sh`) eseguiti almeno una volta su
      macOS/Linux — permessi esecuzione (`chmod +x`) documentati
- [ ] `path.*` con separatori misti (`/` vs `\`) non causano errori — `Path.of()`
      normalizza correttamente su entrambe le piattaforme

---

## 6. Logging e osservabilità

- [ ] `log.writers=console,file` (senza `seq`) — nessun errore, nessun
      tentativo di connessione a Seq
- [ ] `log.writers=console,file,seq` con Seq non raggiungibile — nessun
      errore bloccante, warning su `stderr`, esecuzione normale
- [ ] `log.writers=console,file,seq` con Seq raggiungibile — eventi visibili
      in Seq UI, filtrabili per `repoSlug`, nessun `404`/errore HTTP

---

## 7. Documentazione

- [ ] `README.md` aggiornato — posizionamento generico (non solo Obsidian),
      multi-vault, multi-OS, script CLI documentati
- [ ] `docs/DTR.md` e `docs/DTR_it.md` — unificati, allineati, nessuna voce
      `Planned` che in realtà è già implementata (e viceversa)
- [ ] Tutorial e grooming della milestone corrente presenti in
      `Tutorials/`/`Groomings/`/`Milestones/`
- [ ] Retrospettiva della milestone (`MLS_NomadSync_M*.md`) completata,
      nessuna sezione aperta

---

## 8. Git e versioning

- [ ] `git status` pulito su `feature/*` corrente prima del merge
- [ ] Messaggio di commit descrittivo, referenzia i DTR rilevanti
- [ ] `git flow feature finish <nome>` eseguito
- [ ] `develop` aggiornato e verde
- [ ] `git flow release start 1.0.0` → eventuali bump di versione
      (`pom.xml`, changelog) → `git flow release finish 1.0.0`
- [ ] Tag `v1.0.0` pushato su `main` e `develop`

---

## 9. Post-release (entro 24h)

- [ ] Verifica che il tag sia visibile su GitHub con il changelog corretto
- [ ] Smoke test del JAR scaricato/buildato da `main` (non da `feature/*`)
      su una macchina diversa da quella di sviluppo
- [ ] Comunicazione a tutti i collaboratori (Gabriela) del nuovo stato e
      delle eventuali azioni richieste (es. `git pull` su `develop`,
      ri-build locale)
