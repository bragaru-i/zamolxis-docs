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

The Supervisor interprets user intent. It decides:

- continue an existing Work Session or create a new one
- which tasks are needed
- whether work can run concurrently
- which runtime capability is appropriate
- whether an additional worker is useful
- when user approval is required

The Supervisor decides **what should happen**. It is not the durable source of truth for **what is happening**.

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
