# Troubleshooting Log

Documento ad aggiornamento dinamico. Traccia i problemi incontrati durante lo sviluppo,
le cause identificate e le soluzioni adottate. Utile come riferimento rapido e come
cheat-sheet per situazioni già risolte.

---

## [TRB-001] `fatal: refusing to merge unrelated histories`

**Contesto**: si presenta ogni volta che due repository Git non condividono alcun commit
antenato comune e si tenta un `git pull` o `git merge` tra di essi.

**Casistica riscontrata**:
- Repo locale creata con `git init` + repo remota inizializzata da GitHub con README/LICENSE
- Repo locale cancellata e reinizializzata con `git init` dopo un primo push
- Merge tra branch che non hanno mai condiviso un antenato comune

**Causa radice**: Git costruisce un DAG (Directed Acyclic Graph) di commit. Se due alberi
non hanno nodi in comune, Git rifiuta il merge per sicurezza — non sa se stai unendo
cose correlate o stai commettendo un errore.

**Soluzione**:
```bash
git pull origin main --allow-unrelated-histories
# risolvere eventuali conflitti, poi:
git add .
git commit -m "chore: merge remote initial files"
git push
```

**Prevenzione**: quando si crea una repo remota su GitHub, evitare di inizializzarla
con README/LICENSE se si ha già una repo locale. In alternativa, fare il primo push
prima di qualsiasi altra operazione sul remoto.

---

## [TRB-002] `remote rejected — refusing to delete the current branch`

**Contesto**: tentativo di cancellare un branch remoto che GitHub ha impostato come default.

**Causa radice**: GitHub non permette la cancellazione del branch di default per sicurezza.

**Soluzione**:
1. Su GitHub: `Settings → Branches → Default branch` → cambia il default su `main`
2. Da terminale:
```bash
git push origin --delete master
```

---

## [TRB-003] `merge --abort` fallisce — `Entry not uptodate. Cannot merge`

**Contesto**: tentativo di abortire un merge in corso con file modificati non staged.

**Causa radice**: IntelliJ tiene `.idea/workspace.xml` costantemente aperto e modificato.
Git trova il file in uno stato intermedio e si rifiuta di operare sull'indice.

**Soluzione**:
```bash
git reset HEAD
git merge --abort
# se ancora bloccato:
git reset --hard HEAD   # ⚠️ scarta le modifiche non committate
```

**Prevenzione**: aggiungere al `.gitignore` i file di stato dell'IDE prima di qualsiasi
operazione Git complessa. Per IntelliJ: almeno `.idea/workspace.xml` e `.idea/modules.xml`.

---

## [TRB-004] File `.idea/` e `.obsidian/` tracciati per errore

**Contesto**: file di configurazione IDE/tool committati involontariamente perché
aggiunti prima di configurare il `.gitignore`.

**Causa radice**: `.gitignore` previene il tracking futuro, non annulla quello passato.
I file già tracciati continuano ad essere seguiti da Git ignorando il `.gitignore`.

**Soluzione**:
```bash
git rm --cached -r .idea/
git rm --cached .obsidian/workspace.json
git add .gitignore
git commit -m "chore: untrack IDE and tool config files"
```

**File consigliati da escludere**:
- `.idea/workspace.xml` — stato UI IntelliJ, locale per definizione
- `.obsidian/workspace.json` — stato UI Obsidian (pannelli, cursore)
- `.obsidian/` interamente se non si vuole sincronizzare la configurazione del vault

---

## [TRB-005] Finestra `.bat` che si chiude immediatamente

**Contesto**: doppio click su `.bat` dall'Explorer — la finestra si apre e si chiude
prima che si possa leggere l'output.

**Causa radice**: Windows apre una finestra `cmd`, esegue il bat, e quando il bat
termina chiude la finestra — indipendentemente da cosa fa il processo figlio (il JAR).

**Soluzione**: aggiungere `pause` nel bat dopo ogni invocazione del JAR.
```bat
java -jar ObsidianSync.jar %1 %2
pause
```

**Nota produzione**: in contesti non interattivi (Task Scheduler) rimuovere o condizionare
il `pause` — non ci sarà nessun utente a premere un tasto.

---

## [TRB-006] `git branch` non mostra nulla dopo `git init`

**Contesto**: dopo `git init` e `git branch -M main`, il comando `git branch` non
restituisce output.

**Causa radice**: un branch esiste concretamente solo dopo il primo commit. Prima del
primo commit il branch è dichiarato ma non ha ancora un puntatore a nessun oggetto
nel DAG — Git non lo lista.

**Soluzione**: eseguire il primo commit prima di aspettarsi output da `git branch`.
```bash
git add .
git commit -m "chore: initial commit"
git branch   # ora mostra 'main'
```

---
## [TRB-007] `Process` non è `AutoCloseable` — incompatibilità Java 21/25 vs 26

**Contesto**: utilizzo di `Process` in un try-with-resources in `GitService.runCommand()`.
IntelliJ segnala: `Required type: AutoCloseable — Provided: Process`.

**Causa radice**: `Process` implementa `AutoCloseable` solo a partire da Java 26.
Su Java 21 e 25 non è utilizzabile nel try-with-resources.

**Soluzione**: rimuovere il try-with-resources esterno su `Process`, mantenendo solo
quello sul `BufferedReader`. `Process` viene rilasciato correttamente da `waitFor()`.

```java
Process process = pb.start();
try (BufferedReader reader = new BufferedReader(
        new InputStreamReader(process.getInputStream()))) {
    String line;
    while ((line = reader.readLine()) != null) {
        logService.info("[git] " + line);
    }
}
return process.waitFor();
```

**Allineamento versione**: target del progetto impostato a Java 21 LTS nel `pom.xml`.
JDK superiori installati localmente sono compatibili — Maven compila al livello
dichiarato tramite `maven.compiler.source` e `maven.compiler.target`.

**Criterio di scelta**: Java 21 è la LTS più diffusa in produzione (supporto fino al 2031)
e quella più probabilmente disponibile nell'ambiente del valutatore.