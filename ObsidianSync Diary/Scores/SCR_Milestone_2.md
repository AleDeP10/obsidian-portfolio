# Pagelle ottenute durante Milestone 2

---

## EventType / SyncEvent

**Voto complessivo: 8 / 10**

| Dimensione          | Voto | Note                                                                                                                                                                   |
| ------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rapidità delivery   | 9/10 | Enum con campo e costruttore al primo tentativo, senza imbeccate. Struttura corretta e autonoma.                                                                       |
| Manutenibilità      | 7/10 | `SyncEvent` con tutti setter esposti è mutabile — rischio di modifiche accidentali dopo la creazione. Un oggetto che rappresenta un evento dovrebbe essere immutabile. |
| Autonomia operativa | 8/10 | Package separation, separazione `EventType`/`SyncEvent` in file distinti — scelte proattive e corrette.                                                                |

---

## Pagella SyncEventQueue

**Voto: 7.5/10**

| Dimensione         | Voto | Note                                                                                            |
| ------------------ | ---- | ----------------------------------------------------------------------------------------------- |
| Rapidità delivery  | 8/10 | Struttura centrata al primo tentativo, synchronized applicato correttamente                     |
| Correttezza logica | 6/10 | Bug nel publish: inserisce per ogni elemento non-matching invece che una volta sola             |
| Autonomia          | 8/10 | `consume()` implementato senza imbeccate, gestione `InterruptedException` corretta              |
| Manutenibilità     | 7/10 | `System.exit(1)` in consume è aggressivo — un metodo di coda non dovrebbe terminare il processo |

---

## Pagella SyncOrchestrator

**Voto: 8/10**

| Dimensione         | Voto | Note                                                                                        |
| ------------------ | ---- | ------------------------------------------------------------------------------------------- |
| Rapidità delivery  | 8/10 | Guard `hasUncommittedChanges` introdotto autonomamente e correttamente                      |
| Correttezza logica | 8/10 | Flusso stash/pull/pop corretto, AUTOSAVE separato da PUSH correttamente                     |
| Autonomia          | 9/10 | `retry` con signature estesa a `Exception` — scelta proattiva e migliorativa                |
| Manutenibilità     | 7/10 | `pool` passato come `null` nel Main, `ExecutorService` nel costruttore senza utilizzo reale |