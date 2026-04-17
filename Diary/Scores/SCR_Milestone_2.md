# Pagelle ottenute durante Milestone 2

---

## EventType / SyncEvent

**Voto complessivo: 8 / 10**

| Dimensione          | Voto | Note                                                                                                                                                                   |
| ------------------- | ---- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| Rapidità delivery   | 9/10 | Enum con campo e costruttore al primo tentativo, senza imbeccate. Struttura corretta e autonoma.                                                                       |
| Manutenibilità      | 7/10 | `SyncEvent` con tutti setter esposti è mutabile — rischio di modifiche accidentali dopo la creazione. Un oggetto che rappresenta un evento dovrebbe essere immutabile. |
| Autonomia operativa | 8/10 | Package separation, separazione `EventType`/`SyncEvent` in file distinti — scelte proattive e corrette.                                                                |