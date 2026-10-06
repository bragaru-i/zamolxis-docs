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
|   Products / Sessions / Tasks / Runs / Events / Approvals |
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

The Supervisor is first a conversational project lead. It can answer questions, summarize
current state, explain failures, review evidence and propose a task breakdown without creating
Tasks or Agent Runs. Execution begins only after explicit delegation: direct execution language
from the user or the owner's action to open a displayed proposal.

The Supervisor interprets user intent. It decides:

- continue an existing Work Session or create a new one
- which tasks are needed
- whether work can run concurrently
- which runtime capability is appropriate
- whether an additional worker is useful
- when user approval is required

Its conversational outcomes are `answer`, `propose` and `ask`. `propose` is inert: the proposal
is stored with its repository context but no worker is dispatched. `delegate` crosses the durable
execution boundary and lets the backend validate and create Tasks. Ambiguous or legacy planning
output is handled conservatively as a proposal.

The Supervisor recommends **what could happen**. The owner explicitly delegates, and the backend
authorizes **what will happen**. The Supervisor is not the durable source of truth for **what is
happening**.

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
