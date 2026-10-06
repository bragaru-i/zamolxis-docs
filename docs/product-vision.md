# Zamolxis — Product Vision

Zamolxis is a **local-first control plane and orchestration system for autonomous coding agents**.

## The problem

Engineering work can span several AI coding runtimes, multiple workstations, repositories, and concurrent tasks. Each runtime has its own session model and interface. The user should not need to think in terms of terminals, PIDs, machines, or vendor-specific session lists.

Zamolxis provides one coherent model:

```text
Workspace → Product → Orchestrator Conversation → Work Session → Tasks → Agent Runs → Live Activity
```

The Orchestrator Conversation is the user's persistent entry point across work. A Work Session is
a durable unit of investigation or execution that the Orchestrator creates or reuses when needed.
Agents are temporary executors inside a Session. Ordinary questions and status checks do not
create Work Sessions, Tasks, Agent Runs, Verifiers or Repairs.

## Example

A team is building **Acme Platform**. In the main chat, a user asks:

> Investigate the existing data model, implement the API and UI in parallel, run tests, then review the result.

The Orchestrator creates a **Product Alpha — Reporting Dashboard** Work Session, then may create
research workers, implementation workers and a review worker. They may use different runtimes and
execute on different registered workstations, but the user sees one linked Work Session and one
coherent Orchestrator history.

Later the user asks:

> What is causing the chart grouping problem?

The Orchestrator finds the linked Session and its runs, summarizes their evidence and returns links
to the relevant Session and Agent Run. It creates nothing. If it suggests a repair, the proposal
remains inert until the user says, for example, “Open this work” or selects the equivalent action.
It then continues the existing Session or creates a new one when Product/repository isolation or
the goal requires it.

## Product principles

1. **Local execution.** Source code, credentials, shells and coding runtimes remain on registered workstations.
2. **Runtime independence.** No single coding agent is the center of the architecture.
3. **Sessions over processes.** The primary UI represents work, not terminals.
4. **Durable orchestration.** Workflow state survives individual agent processes and transient failures.
5. **Observable execution.** Users can inspect active workers, hierarchy, activity and outcomes in realtime.
6. **Least privilege.** Each workstation explicitly grants repositories and capabilities.
7. **Human control at risk boundaries.** Sensitive actions can require approval.
8. **Extensibility.** New agent runtimes can be added through adapters without redesigning the product.
