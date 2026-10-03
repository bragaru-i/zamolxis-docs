# Zamolxis — Product Vision

Zamolxis is a **local-first control plane and orchestration system for autonomous coding agents**.

## The problem

Engineering work can span several AI coding runtimes, multiple workstations, repositories, and concurrent tasks. Each runtime has its own session model and interface. The user should not need to think in terms of terminals, PIDs, machines, or vendor-specific session lists.

Zamolxis provides one coherent model:

```text
Workspace → Product → Work Session → Tasks → Agent Runs → Live Activity
```

A Work Session represents a durable unit of intent. Agents are temporary executors inside it.

## Example

A team is building **Acme Platform**. A user opens a session called **Product Alpha — Reporting Dashboard** and asks:

> Investigate the existing data model, implement the API and UI in parallel, run tests, then review the result.

Zamolxis may create research workers, implementation workers and a review worker. They may use different runtimes and execute on different registered workstations, but the user sees one Work Session and one coherent history.

Later the user says:

> Check why the chart grouping is incorrect.

The Supervisor decides whether this belongs to the existing Product Alpha session or should become new work.

## Product principles

1. **Local execution.** Source code, credentials, shells and coding runtimes remain on registered workstations.
2. **Runtime independence.** No single coding agent is the center of the architecture.
3. **Sessions over processes.** The primary UI represents work, not terminals.
4. **Durable orchestration.** Workflow state survives individual agent processes and transient failures.
5. **Observable execution.** Users can inspect active workers, hierarchy, activity and outcomes in realtime.
6. **Least privilege.** Each workstation explicitly grants repositories and capabilities.
7. **Human control at risk boundaries.** Sensitive actions can require approval.
8. **Extensibility.** New agent runtimes can be added through adapters without redesigning the product.
