# Realtime Product UI

## Home: My Work

The primary screen is session-centric.

```text
MY WORK

● Product Alpha — Reporting Dashboard
  3 agents active · 2 completed
  Last activity 12 sec ago

! Acme Platform — Authentication Upgrade
  Waiting for you
  1 agent active

✓ Product Beta — Import Pipeline
  Completed 38 min ago
```

Workstation identity is secondary metadata, not primary navigation.

## Work Session view

```text
Product Alpha — Reporting Dashboard                 RUNNING

Goal
Build reporting dashboard

WORKFLOW

✓ Architecture Research
  4m

        +------------------+
        |                  |
        v                  v
● API Implementation   ● UI Implementation
  18m                    11m

        +------------------+
                 |
                 v
             ○ Tests
               waiting
                 |
                 v
             ○ Review
               waiting
```

## Agent Run view

Selecting a run reveals the richest supported native activity.

```text
API IMPLEMENTATION                                  LIVE

Working for 18m 32s

> Inspecting reporting aggregation

  Read schema
  Read reporting service

> Updating aggregation...

  Modified:
  reporting-service
  reporting-schema

> Running tests

  41 passed
  2 failed

> Investigating failures...                         ●

[ Message ] [ Steer ] [ Stop ]
```

The exact detail depends on runtime adapter capabilities.

## Needs You

Approval and blocking states should be first-class.

```text
NEEDS YOU

Product Alpha
Agent wants to run production deployment.

[Approve] [Reject] [Open Session]
```

The user should be able to supervise meaningful work from a phone without watching terminals.
