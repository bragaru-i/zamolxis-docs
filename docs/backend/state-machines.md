# Backend State Machines

State transitions should be enforced by backend functions rather than arbitrary document patches.

## Agent Run

```text
QUEUED
  |
  v
STARTING -----> FAILED
  |
  v
RUNNING <----> WAITING
  |   \
  |    -> NEEDS_APPROVAL
  |             |
  |             v
  |          RUNNING
  |
  +----> STOPPING ----> STOPPED
  |
  +----> COMPLETED
  |
  +----> FAILED
  |
  '---> LOST
```

LOST means cloud cannot prove the native process state after timeout/reconciliation. It must not be silently converted to FAILED if local work may still exist.

## Workspace

```text
REQUESTED
   |
PROVISIONING
   |
 READY
   |
 IN_USE
   |
 +-------> DIRTY
 |           |
 |           v
 |       INTEGRATING
 |           |
 |           v
 +------> COMPLETED
             |
       CLEANUP_PENDING
             |
           REMOVED
```

ERROR may be entered from provisioning/validation/integration failures.

## Task

```text
PLANNED
   |
 BLOCKED <---- dependencies incomplete
   |
 READY
   |
 RUNNING
   |
 +----> WAITING
 +----> COMPLETED
 +----> FAILED
 '---> CANCELLED
```

A Task is COMPLETED based on task outcome policy, not merely because one Agent Run exited successfully.

## Work Session

A session derives much of its status from tasks and approvals but stores an explicit durable status for UI and workflow semantics.

```text
PLANNING -> RUNNING -> WAITING/NEEDS_INPUT
                    -> COMPLETED
                    -> FAILED
                    -> CANCELLED
```

## Idempotency rules

- repeating `workspace.provision` with the same idempotency key returns the same Workspace
- repeating `runtime.start` must not create a second native process
- duplicate Run Events do not re-advance a workflow
- completion transitions are monotonic unless an explicit recovery operation is used
- cleanup is safe to retry
