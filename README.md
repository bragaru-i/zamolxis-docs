# Zamolxis

**Local-first control plane and orchestration for autonomous coding agents.**

Zamolxis provides one workspace for supervising work performed by multiple AI coding runtimes across registered local workstations.

Instead of exposing terminals and processes as the primary abstraction, Zamolxis models:

```text
Product → Repository → Work Session → Task → Workspace → Agent Run → Runtime

Owner → Orchestrator Conversation → answer / linked Work Session
```

Executable behavior lives in `bragaru-i/zamolxis`. This repository contains
architecture/reference material; the capability lists below describe the design.
See [Alpha implementation boundary](docs/alpha-implementation.md) for tested
behavior and current limitations.

## What Zamolxis does

- keeps coding execution local
- presents persistent Work Sessions above individual agent processes
- starts, resumes, steers and observes supported native agent runtimes
- coordinates parallel and sequential agent work
- routes natural-language instructions through a Supervisor
- maintains durable workflow state independently of any one LLM
- provides realtime web/mobile visibility
- enforces permissions and approval boundaries on each workstation
- remains independent from any single runtime vendor

## Architecture at a glance

```text
                       Zamolxis UI
                    Web / Mobile PWA
                           |
                           v
                +---------------------+
                |    Control Plane    |
                |---------------------|
                | Orchestrator        |
                | Session Supervisor  |
                | Work Sessions       |
                | Durable Workflows   |
                | Events / Approvals  |
                +----------+----------+
                           |
                 secure outbound links
                    /             \
                   v               v
          +---------------+ +---------------+
          | Workstation A | | Workstation B |
          | Zamolxis Node | | Zamolxis Node |
          +-------+-------+ +-------+-------+
                  |                 |
             Runtime Router    Runtime Router
              /    |    \      /    |    \
             v     v     v    v     v     v
           Codex Claude Hermes ...native runtimes...
```

## Documentation

| Document | Purpose |
| --- | --- |
| [Product Vision](docs/product-vision.md) | Problem, product definition and principles |
| [Architecture](docs/architecture.md) | System boundaries and end-to-end architecture |
| [Work Session Model](docs/session-model.md) | Durable work context, tasks and Agent Runs |
| [Runtime Adapters](docs/runtime-adapters.md) | Runtime-independent execution contract |
| [Zamolxis Node](docs/local-node.md) | Local execution bridge and workstation responsibilities |
| [Control Plane](docs/control-plane.md) | Convex-backed state, events, commands and Supervisor |
| [Backend Build Spec](docs/backend/README.md) | Implementation order and backend component map |
| [Backend Domain Schema](docs/backend/domain-schema.md) | Concrete entities, fields, relationships and invariants |
| [Git Workspaces](docs/backend/git-workspaces.md) | Required Git worktree lifecycle, locking and integration |
| [Node Protocol](docs/backend/node-protocol.md) | Commands, events, offline buffering and reconciliation |
| [Backend State Machines](docs/backend/state-machines.md) | Valid Run, Workspace, Task and Session transitions |
| [Backend API Contract v0.1](docs/backend/backend-api-contract-v0.1.md) | Concrete queries, mutations, Node operations and Supervisor tools |
| [Frontend Architecture v0.1](docs/frontend-architecture-v0.1.md) | Next.js/shadcn stack, package boundaries, realtime UI and graph architecture |
| [Realtime UI](docs/realtime-ui.md) | Session-centric operator experience |
| [Security](docs/security.md) | Permissions, trust boundaries and approvals |
| [Roadmap](docs/roadmap.md) | Milestones from architecture PoC to Alpha |

## Status

**Pre-Alpha / Architecture phase**

The immediate objective is a vertical slice:

```text
Web UI
  → Control Plane
  → Zamolxis Node
  → native local coding runtime
  → normalized live events
  → Web UI
```

Once this loop works reliably for one runtime and one workstation, Zamolxis expands to session intelligence, Supervisor routing, multi-agent workflows, additional runtime adapters and multiple workstations.

## Name

**Zamolxis** takes its name from the figure associated with the ancient Getae and Dacians. The name is used for the product identity; internal architecture deliberately keeps clear engineering terminology such as Supervisor, Work Session, Agent Run, Runtime Adapter, Node and Workflow.
