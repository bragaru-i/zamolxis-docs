# Backend Build Specification

This section is implementation-oriented. It is intended to be usable by an engineering agent building Zamolxis, not only as product documentation.

## Backend responsibilities

The backend is split across two trust domains:

1. **Cloud Control Plane** — durable coordination, realtime product state, Supervisor and workflow orchestration.
2. **Zamolxis Node** — trusted local execution, Git/worktree ownership, runtime lifecycle and local policy enforcement.

The critical rule is:

> The LLM/runtime never chooses an arbitrary filesystem directory. Zamolxis resolves and validates a Workspace first, then starts the runtime with that Workspace as its working directory.

## Backend components

```text
Cloud
├── Domain API
├── Realtime State
├── Session Service
├── Task Graph Service
├── Run Service
├── Command Bus
├── Event Ingestion
├── Approval Service
├── Supervisor Tools
└── Workflow Engine

Local Node
├── Node Connection
├── Repository Registry
├── Workspace Manager
│   └── Git Worktree Manager
├── Runtime Registry
├── Runtime Adapters
├── Event Normalizer
├── Command Executor
├── Policy Engine
└── Recovery/Reconciliation
```

## Required build order

1. Domain IDs, enums and state machines.
2. Convex schema and indexes.
3. Node registration/heartbeat.
4. Command/event protocol.
5. Repository Registry.
6. Workspace + Git Worktree Manager.
7. Mock Runtime Adapter.
8. First native Runtime Adapter.
9. Run lifecycle.
10. Task dependency engine.
11. Integration Workspace workflow.
12. Reconnect/reconciliation.
13. Supervisor tools.
14. Approvals and hardening.

See:

- [Convex Schema v0.1](convex-schema-v0.1.md) — concrete build-ready table/index proposal
- [Domain Model](domain-schema.md) — conceptual entity model and invariants
- [Node Protocol](node-protocol.md)
- [Git Workspaces](git-workspaces.md)
- [State Machines](state-machines.md)
