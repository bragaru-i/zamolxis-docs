# Zamolxis Architecture

## Overview

```text
                         Zamolxis UI
                      Web / Mobile PWA
                              |
                              v
+------------------------------------------------------------+
|                    CLOUD CONTROL PLANE                     |
|                                                            |
|   Global Supervisor       Durable Workflow Engine          |
|          |                         |                       |
|          +----------+--------------+                       |
|                     v                                      |
|   Conversations / Products / Sessions / Tasks / Runs      |
|   Events / Approvals / Evidence / Links                   |
+---------------------+--------------------------------------+
                      |
               secure outbound links
             /                       \
            v                         v
+------------------------+   +------------------------+
| Workstation A          |   | Workstation B          |
| Zamolxis Node          |   | Zamolxis Node          |
|                        |   |                        |
| Runtime Router         |   | Runtime Router         |
|  |- Runtime Adapter A  |   |  |- Runtime Adapter A  |
|  |- Runtime Adapter B  |   |  |- Runtime Adapter B  |
|  '- Runtime Adapter C  |   |  '- Runtime Adapter C  |
|                        |   |                        |
| local repos/tools/git  |   | local repos/tools/git  |
+------------------------+   +------------------------+
```

## Control plane

The control plane owns durable coordination, not source code execution.

It stores:

- registered workstations and capabilities
- durable Orchestrator conversations and routing decisions
- products and repositories
- work sessions
- tasks and dependency graphs
- agent runs and parent/child relationships
- normalized activity events
- commands and acknowledgements
- approvals
- heartbeats
- summaries, decisions and artifacts metadata

Raw terminal output should be retained locally or selectively streamed. The cloud layer receives normalized events and only the transcript data required by the product experience.

## Global Supervisor

> **Breaking architecture boundary:** the app's primary conversation is an Orchestrator
> Conversation. It is not a Work Session, and submitting a message must not create a Work Session
> before routing decides one is required.

The Global Supervisor is the Orchestrator behind the app's primary chat. Its conversation exists
outside Work Sessions. It can answer questions, summarize current state, explain failures, review
evidence and link existing tickets, Sessions, Tasks, Agent Runs and pull requests without creating
a Work Session, Task or Agent Run.

The Supervisor interprets user intent. It decides:

- answer directly from durable control-plane state
- link relevant tickets, Sessions, Tasks, Agent Runs, evidence and pull requests
- continue an existing Work Session or create a new one when work is required
- which tasks are needed
- whether work can run concurrently
- which runtime capability is appropriate
- whether an additional worker is useful
- when user approval is required

Its routing outcomes are `answer`, `link`, `propose`, `continue`, `create` and `ask`. `answer`,
`link` and `propose` are inert: no Work Session or worker is created. `continue` and `create`
cross the durable work boundary only after explicit execution intent or a justified need for
durable investigation. The backend validates Product isolation, route targets and authorization
before changing workflow state.

The Orchestrator may summarize structured state itself or select an existing Agent Run when its
evidence is relevant. Starting a new agent is a durable routing decision, never a side effect of a
status question. Role profiles choose runtime/model/effort for future Supervisor, Builder,
Verifier and Repair runs; each Run snapshots and exposes the actual model used.

The Supervisor recommends **what could happen**. The owner explicitly delegates, and the backend
authorizes **what will happen**. The Supervisor is not the durable source of truth for **what is
happening**. Session closure is deterministic: all Tasks are terminal, no Run, approval,
verification, repair or integration remains pending, and every required exact-SHA trust decision
is settled.

## Workflow engine

A deterministic durable workflow layer tracks dependencies, retries, waiting states and completion.

```text
                 Research
                    |
          +---------+---------+
          |                   |
          v                   v
    API Implementation   UI Implementation
          |                   |
          +---------+---------+
                    |
                    v
                  Tests
                    |
                    v
                  Review
```

Long-running work happens on a workstation. The control plane does not keep a cloud function alive for the lifetime of a coding agent. It records the run, waits for events, and advances the graph when completion arrives.

## Zamolxis Node

Each registered workstation runs a local Zamolxis Node.

Responsibilities:

- authenticate with the control plane
- maintain an outbound connection
- advertise installed runtime capabilities
- enforce local repository/capability permissions
- start, resume, steer and stop supported runtimes
- discover supported native sessions
- normalize native activity into Zamolxis events
- report heartbeat and health
- optionally detect manually started native sessions

No inbound public port is required.

## Runtime Router

The Runtime Router resolves an abstract execution request to a local adapter.

```text
Run Request
    |
    v
Runtime Policy
    |
    +-- auto
    +-- preferred runtime
    '-- forced runtime
    |
    v
Runtime Adapter
    |
    v
Native Agent Runtime
```

The control plane and UI must not depend on vendor-specific process semantics.
