### Overall architecture
```mermaid
graph TD
    Main["**Main**\nargs · properties · wiring"]:::purple

    Main -->|"publish(event)"| Queue
    Main -->|"start()"| Orchestrator
    Main -->|"start()"| Scheduler

    Queue["**SyncEventQueue**\npriority · latest-wins"]:::teal
    Queue -->|"consume()"| Orchestrator

    Orchestrator["**SyncOrchestrator**\nworker loop · retry"]:::purple
    Orchestrator -->|"execute(event)"| GitService
    Orchestrator -->|"onFailure()"| Hook
    Orchestrator -->|"log()"| LogService

    Scheduler["**AutosaveScheduler**\nScheduledExecutorService"]:::teal
    Scheduler -->|"publish(AUTOSAVE)"| Queue

    GitService["**GitService**\nstash · pull · commit · push · hasChanges"]:::amber
    GitService -->|"log()"| LogService

    LogService["**LogService**\nsynchronized · append-only"]:::gray

    Hook["**NotificationHook**\ninterface — LogNotificationHook"]:::coral

    SyncEvent["**SyncEvent / EventType**\npriority · timestamp · retry"]:::gray
    SyncEvent --> Queue

    subgraph Priority
        P1["1 — PULL_LOGON · commit + push"]:::purple
        P2["2 — PUSH_MANUAL · commit + push"]:::teal
        P3["3 — PUSH_LOGOFF · commit + push"]:::amber
        P4["4 — AUTOSAVE · local commit only"]:::gray
    end

    classDef purple fill:#EEEDFE,stroke:#534AB7,color:#3C3489
    classDef teal   fill:#E1F5EE,stroke:#0F6E56,color:#085041
    classDef amber  fill:#FAEEDA,stroke:#854F0B,color:#633806
    classDef coral  fill:#FAECE7,stroke:#993C1D,color:#712B13
    classDef gray   fill:#F1EFE8,stroke:#5F5E5A,color:#444441
```

This diagram illustrates the component architecture and interaction flow of ObsidianSync.

**Main** acts as the application entry point and wiring layer — it bootstraps all dependencies,
translates CLI arguments into typed events, and delegates execution to the orchestrator.

**SyncEventQueue** is a thread-safe priority queue implementing latest-wins deduplication.
Publishers drop events in; the orchestrator pulls them out in priority order.
Lower priority number means higher urgency: a pending pull always precedes an autosave.

**SyncOrchestrator** owns the worker loop — a dedicated thread that consumes events serially,
preventing Git concurrency issues by design. On failure, it applies exponential backoff retry
(up to 3 attempts: 30s → 60s → 120s) before delegating to the NotificationHook.

**AutosaveScheduler** runs on a ScheduledExecutorService and periodically publishes AUTOSAVE
events to the queue. It never calls GitService directly — it is a publisher, not an executor.

**GitService** wraps Git CLI operations via ProcessBuilder. Each method maps to a single
Git command. Sequencing and error handling are the orchestrator's responsibility, not GitService's.

**LogService** provides levelled, append-only, thread-safe logging shared across all components.

**NotificationHook** is an interface following the Dependency Inversion Principle — the
orchestrator depends on the abstraction, not the implementation. The default implementation
(LogNotificationHook) writes failures to the log. A future tray icon implementation will
replace it without touching the orchestrator.
