# Zamolxis Roadmap

The roadmap is milestone-based rather than date-driven. Each milestone must produce a demonstrable end-to-end capability before the next abstraction is added.

## M0 — Architecture & contracts

**Goal:** freeze the minimum domain model and protocol boundaries.

Deliverables:

- Product / Work Session / Task / Agent Run model
- Node ↔ Control Plane command/event protocol
- Runtime Adapter interface
- capability negotiation model
- local permission model
- first Convex schema

Exit criterion: a mocked Node can appear online and drive a mocked Agent Run through queued → running → completed.

## M1 — Repository & Workspace foundation

**Goal:** make Git repository identity and isolated execution deterministic before native agents are allowed to mutate code.

Deliverables:

- Repository Registry
- per-workstation Repository Locations
- first-class Workspace entity
- Git Worktree Manager
- deterministic branch/worktree naming
- Workspace ownership leases
- pre-run repository/worktree validation
- dirty-state capture
- restart/reconnect reconciliation
- Integration Workspace primitive

Exit criterion: two parallel mocked tasks receive separate worktrees from the same repository, can modify the same source path independently, survive a Node restart, and can be combined through an Integration Workspace without touching the canonical working directory.

## M2 — First local runtime vertical slice

**Goal:** prove real remote control of one native local coding session.

Deliverables:

- Zamolxis Node prototype
- workstation registration
- first runtime adapter
- start/resume/stop
- normalized activity events
- basic Work Session UI

Exit criterion:

```text
Phone/Web
  -> create Work Session
  -> start native local agent
  -> see live state
  -> agent completes
  -> session records result
```

## M3 — Realtime operator UI

**Goal:** make Zamolxis useful without opening the workstation terminal.

Deliverables:

- My Work dashboard
- session workflow graph
- Agent Run live view
- message/steer/stop controls
- waiting/error states
- workstation health
- session history

Exit criterion: a user can understand what a running session is doing from the web/mobile UI.

## M4 — Session intelligence

**Goal:** make work persistent above individual agent processes.

Deliverables:

- canonical session context
- decisions/findings/progress summaries
- artifact references
- native-session attachment
- manual local session discovery where supported

Exit criterion: stop one worker, start another, and continue the same Work Session without losing the product-level context.

## M5 — Supervisor

**Goal:** accept natural-language instructions instead of requiring manual runtime operations.

Deliverables:

- global Supervisor
- existing-session vs new-session routing
- task planning
- runtime policy: auto/preferred/forced
- Supervisor tool API
- routing audit trail

Exit criterion: a user can issue a natural-language task and Zamolxis routes it to the correct Work Session and runtime.

## M6 — Multi-agent workflows

**Goal:** coordinate parallel and sequential work durably.

Deliverables:

- dependency graph
- parallel fan-out
- wait/resume
- retry/failure policies
- parent/child Agent Runs
- dynamic worker spawning

Exit criterion: research → parallel implementation → tests → review can complete without a continuously running orchestration process.

## M7 — Multiple runtime adapters

**Goal:** prove runtime independence.

Deliverables:

- Codex integration
- Claude Code integration
- Hermes integration
- capability-aware UI
- cross-runtime workflow

Exit criterion: one Work Session can contain runs from multiple native runtimes without special-casing the session model.

## M8 — Multi-workstation execution

**Goal:** treat registered computers as interchangeable execution nodes.

Deliverables:

- multiple online Nodes
- capability/availability routing
- per-workstation workspace mappings
- reconnect/recovery
- offline handling

Exit criterion: one Work Session can coordinate work on Workstation A and Workstation B.

## M9 — Approvals & hardening

**Goal:** make autonomous operation safe enough for daily use.

Deliverables:

- approval inbox
- local command policies
- destructive-action protection
- audit log
- device revocation
- reconnect/idempotency hardening
- secrets review

Exit criterion: unattended local agents can run within explicit boundaries while risky operations reliably stop for approval.

## M10 — Alpha

**Goal:** daily-driver product.

Alpha should support:

- web/mobile supervision
- persistent Work Sessions
- live Agent Run visibility
- runtime abstraction
- multiple registered workstations
- natural-language Supervisor
- multi-agent workflow graphs
- approvals
- historical inspection

Alpha is successful when the user can leave the workstations running, operate ongoing coding work primarily from Zamolxis, and only open a native agent UI when deep manual intervention is required.
