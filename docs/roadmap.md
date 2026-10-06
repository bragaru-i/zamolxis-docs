# Zamolxis Roadmap

The roadmap is milestone-based rather than date-driven. Each milestone describes a
product target, not a claim that every listed item is shipped. The executable
`bragaru-i/zamolxis` repository is the behavioral source of truth.

## Current position — private Alpha candidate

As of 2026-10-06, executable main through PR #104 spans the core outcomes of these
milestones: the deployed control plane can plan from natural language, run parallel
Builders in isolated worktrees, verify exact candidate SHAs independently, apply a
deterministic trust/repair policy, prepare local integration, and expose the workflow,
approvals and history through the web/mobile UI. Codex and Claude are operational
runtime adapters.

The engineering Alpha is therefore built, but the Alpha exit criterion is not yet
fully proven. Final acceptance passed on exact executable main SHA `85c1f32`: the full
check and all five authenticated Codex groups (intent loop, Supervisor, Orchestrator,
approval reject/approve and restart/resume) passed with the canonical checkout
unchanged. The sole non-deferred gate is one production PR opened end to end through
Zamolxis. The owner has deferred real iPhone/PWA/offline checks, deployed
second-Google-account isolation and validation with a second workstation. Production
currently demonstrates one Mac. Items below such as Hermes, external ticket connectors
and other explicitly unimplemented details remain roadmap scope; presence in a
milestone does not mean they shipped.

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

Implementation order:

```text
App Shell
  -> My Work
  -> Work Session
  -> Agent Run
  -> normalized realtime activity
  -> message/stop controls
  -> mobile states
  -> workflow graph
```

Frontend foundation:

- Next.js App Router + TypeScript
- shadcn/ui as primary component layer
- reusable UI in `packages/ui`
- third-party wrappers/integrations in `packages/lib`
- Convex reactive queries for server state
- React Flow/React Flow UI for workflow visualization
- TanStack Virtual for large event feeds

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

The Alpha target is:

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

Current assessment: **private Alpha candidate**. The engineering workflow and final
current-main acceptance are complete; the remaining production-publishing proof and
owner-deferred device, account-isolation and multi-workstation checks are recorded
above. Until those gates are complete, this roadmap does not label the product
Alpha-complete.
