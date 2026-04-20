
# Git Flow

## Concetto base

Git Flow è una convenzione sui branch che separa nettamente il lavoro in corso dal codice stabile. I branch hanno ruoli fissi:

|Branch|Ruolo|
|---|---|
|`main`|Solo codice rilasciato e stabile. Taggato ad ogni release.|
|`develop`|Integrazione continua dello sprint. Base per le feature.|
|`feature/*`|Una feature alla volta. Nasce da `develop`, torna su `develop`.|
|`release/*`|Preparazione al rilascio. Bugfix finali, bump versione.|
|`hotfix/*`|Fix urgenti su `main`. Nasce da `main`, torna su `main` e `develop`.|

Per il workflow corrente le branch rilevanti sono `main`, `develop` e `feature/*`.

---

## Installazione

Git Flow AVH Edition è **incluso automaticamente da Git for Windows** — nessuna installazione separata necessaria.

```bash
# verifica disponibilità
git flow version
# output atteso: 1.12.3 (AVH Edition)
```

---

## Inizializzazione su repo esistente

`git flow init` si esegue su una repo esistente senza distruggere niente. Legge i branch già presenti e li mappa sui ruoli di Git Flow.

```bash
git flow init
# accetta tutti i default premendo Invio
```

Branch `main` e `develop` già presenti vengono riconosciuti direttamente. Il remote `origin` e la history rimangono intatti.

---

## Flusso feature — utilizzo quotidiano

```bash
# inizio lavoro su una nuova feature
git flow feature start nome-feature

# ... lavori, committi normalmente su feature/nome-feature ...

# feature completata: merge automatico su develop, cancella il branch locale
git flow feature finish nome-feature

# pubblica develop aggiornato
git push origin develop
```

---

## Flusso release — a fine milestone

```bash
# apre una release branch da develop
git flow release start 1.0.0

# aggiornamenti finali: versione nel pom.xml, README, changelog
# poi:
git flow release finish 1.0.0
# merge automatico su main E su develop
# crea il tag v1.0.0 automaticamente

git push origin main develop --tags
```

---

## Flusso hotfix — bug urgente su main

```bash
git flow hotfix start nome-fix

# fix del bug, commit

git flow hotfix finish nome-fix
# merge automatico su main E su develop
# crea tag automaticamente

git push origin main develop --tags
```

---

## Mapping sul workflow Agile

| Evento Scrum        | Git Flow                                           |
| ------------------- | -------------------------------------------------- |
| Inizio sprint       | `git flow feature start <nome>` per ogni obiettivo |
| Commit di lavoro    | commit normali sul branch feature                  |
| Fine feature        | `git flow feature finish <nome>`                   |
| Fine milestone      | `git flow release start/finish <versione>`         |
| Bug urgente su main | `git flow hotfix start/finish <nome>`              |

---

## Naming delle feature

Kebab-case, un branch per obiettivo del grooming:

```
feature/sync-event
feature/sync-event-queue
feature/notification-hook
feature/sync-orchestrator
feature/git-service-extended
```

Il diff risultante sarà pulito e leggibile anche per i recruiter.

---

## Cheat-sheet comandi

```bash
git flow init                              # inizializza su repo esistente
git flow feature start <nome>             # nuovo branch feature da develop
git flow feature finish <nome>            # merge su develop, cancella branch
git flow release start <versione>         # branch di preparazione release
git flow release finish <versione>        # merge su main e develop, tag
git flow hotfix start <nome>              # fix urgente da main
git flow hotfix finish <nome>             # merge su main e develop, tag
git push origin main develop --tags       # push completo post-release/hotfix
```


---
---

# `synchronized` — ripasso completo

### Il problema senza sincronizzazione

```
Thread A (autosave):         Thread B (logoff):
legge coda — AUTOSAVE presente
                             legge coda — AUTOSAVE presente
rimuove AUTOSAVE
                             rimuove AUTOSAVE  ← elemento già rimosso, no-op
inserisce nuovo AUTOSAVE
                             inserisce nuovo AUTOSAVE  ← duplicato!
```

Risultato: due AUTOSAVE in coda. Il lock garantisce che la sequenza sia **atomica e indivisibile**.

---

### Tre forme di `synchronized`

**1. Metodo d'istanza** — lock su `this`

java

```java
public synchronized void publish(SyncEvent event) {
    // un solo thread alla volta esegue questo metodo
}
```

**2. Blocco esplicito su `this`**

java

```java
public void publish(SyncEvent event) {
    synchronized (this) {
        // solo questa sezione è protetta
        // codice fuori dal blocco è liberamente concorrente
    }
}
```

**3. Blocco su oggetto dedicato** — più flessibile

java

```java
private final Object lock = new Object();

public void publish(SyncEvent event) {
    synchronized (lock) {
        // puoi avere lock diversi per sezioni diverse
    }
}
```