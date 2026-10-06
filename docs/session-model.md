# Work Session Model

## Session != Agent

A **Work Session** is the durable context of a piece of work.

An **Agent Run** is a temporary executor operating inside that context.

An **Orchestrator Conversation** is the durable human-facing chat above Work Sessions. It may link
and summarize many Sessions without belonging to any of them.

```text
Workspace
  |
  '-- Acme Platform
       |
       '-- Product Alpha — Reporting Dashboard
            |
            +-- Architecture Research       DONE
            +-- API Implementation          RUNNING
            +-- UI Implementation           RUNNING
            '-- Review                      WAITING
```

Workers may execute in parallel, sequentially, or be created dynamically when another worker becomes blocked.

## Canonical session context

Each Work Session maintains:

- goal
- product/repository references
- branch/worktree references where applicable
- constraints
- decisions
- current plan
- progress
- important findings
- unresolved questions
- artifacts
- active and historical runs

This context is compact and durable. It is not simply an ever-growing raw transcript.

## Routing a new user message

```text
User Instruction
       |
       v
Global Supervisor
       |
       +-- question/status? -----------------> Answer + link existing state
       |
       +-- proposal only? -------------------> Store inert proposal
       |
       +-- execution related to a session? --> Continue Session
       |
       +-- new durable work? -----------------> Create Session
       |
       '-- ambiguous ------------------------> Ask in Orchestrator chat
```

Signals can include Product/repository identity, issue or ticket references, linked pull requests,
goal similarity, worktree/branch relationship, recent activity, terminal workflow state and
explicit user wording. A question never becomes execution merely because the Orchestrator finds a
possible improvement.

The routing decision is stored so the system can explain and correct it later.

Creating or continuing a Session binds Product, repositories and the effective role profiles.
Closing it is a backend trust decision: every Task is terminal, no Run, approval, verifier, repair
or integration is pending, and all required exact-SHA evidence is settled.

## Run graph

```text
Product Alpha
     |
     +-- Data Research
     |
     +-- API Worker
     |      |
     |      '-- Schema Investigation
     |
     +-- UI Worker
     |
     '-- Review Worker
```

This lineage remains inspectable after completion.
