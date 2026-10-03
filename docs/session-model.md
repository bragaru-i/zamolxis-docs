# Work Session Model

## Session != Agent

A **Work Session** is the durable context of a piece of work.

An **Agent Run** is a temporary executor operating inside that context.

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
       +-- related to active session? -- yes --> Continue Session
       |
       '-- no -------------------------------> Create Session
```

Signals can include product/repository identity, issue references, goal similarity, worktree/branch relationship, recent activity and explicit user wording.

The routing decision is stored so the system can explain and correct it later.

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
