# Node ↔ Control Plane Protocol

## Goals

The protocol must tolerate:

- workstation sleep
- network interruption
- Node restart
- duplicate delivery
- delayed events
- a runtime continuing locally while cloud connectivity is unavailable

## Connection

The Node initiates an authenticated outbound connection.

On connect it sends a handshake containing:

```ts
{
  nodeId,
  workstationId,
  nodeVersion,
  instanceId,
  platform,
  runtimeCapabilities,
  lastAcknowledgedCommand?,
  eventCursors
}
```

## Heartbeat

Heartbeat reports:

- Node instance ID
- uptime
- active Runs
- active Workspace leases
- runtime health
- free/pressure indicators where useful
- last event sequence per active Run

Heartbeat is operational metadata, not the sole source of completion truth.

## Commands

Core command types:

```text
repository.verify

workspace.provision
workspace.inspect
workspace.cleanup

runtime.start
runtime.resume
runtime.send
runtime.stop
runtime.inspect

node.refresh_capabilities
node.reconcile
```

Every command has an idempotency key.

Node execution pattern:

```text
PENDING
  -> CLAIMED
  -> ACKNOWLEDGED
  -> COMPLETED | FAILED
```

ACKNOWLEDGED means the Node accepted responsibility. It does not mean the native operation completed.

## Events

Core normalized events:

```text
node.connected
node.reconciled

workspace.provisioning
workspace.ready
workspace.dirty
workspace.error
workspace.removed

run.starting
run.started
run.waiting
run.needs_approval
run.completed
run.failed
run.stopped
run.lost

run.message
run.tool_started
run.tool_completed
run.command_started
run.command_completed
run.files_changed
run.child_created
run.child_updated
```

## Event envelope

```ts
{
  eventId,
  workstationId,
  nodeInstanceId,
  runId?,
  workspaceId?,
  sequence?,
  type,
  occurredAt,
  payload
}
```

Events are at-least-once deliverable. Cloud ingestion must deduplicate by eventId.

## Offline buffering

The Node maintains a small durable local outbox.

If cloud connectivity disappears:

1. native runtimes may continue according to policy
2. normalized events are appended locally
3. command intake stops
4. on reconnect, Node replays unacknowledged events
5. cloud deduplicates and reconciles state

## Reconciliation

Reconnect is not just "mark online".

The Node sends actual local state:

```text
active runtime processes
native session IDs
workspace registrations
workspace leases
HEAD/branch state
pending local events
```

The control plane compares desired state to actual state.

Examples:

- cloud says RUNNING, process gone -> Run LOST/FAILED after policy
- cloud says STARTING, process exists -> adopt and mark RUNNING
- cloud says workspace IN_USE, lease absent -> reconcile lease
- local dirty orphan workspace -> preserve and surface it

Never delete local work as a reconciliation shortcut.
