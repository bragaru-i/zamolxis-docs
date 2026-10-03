# Control Plane

## Role

The cloud control plane coordinates local work and powers the realtime user experience.

The initial implementation is designed around Convex for reactive state, server functions and durable orchestration components.

## Core entities

```text
users
workstations
products
repositories
workSessions
tasks
agentRuns
runEvents
commands
approvals
artifacts
sessionSummaries
```

## Event model

Prefer semantic events over raw terminal bytes.

Examples:

```text
node.connected
node.disconnected

session.created
session.updated

run.queued
run.started
run.waiting
run.completed
run.failed
run.stopped

run.child_created
run.message
run.tool_started
run.tool_completed
run.files_changed

approval.requested
approval.resolved
```

Adapters may additionally provide richer runtime-specific payloads, but core product behavior should depend on normalized events.

## Commands

Commands are durable intents sent from the control plane to a Node.

Examples:

```text
runtime.start
runtime.resume
runtime.send
runtime.stop
session.inspect
node.refresh_capabilities
```

Commands have IDs, acknowledgement state and idempotency semantics.

## Supervisor

The Supervisor can use an agent framework for natural-language reasoning and tool calls, while deterministic application state remains in normal database tables.

Supervisor tools should operate at product-level semantics:

```text
findRelevantSessions()
createWorkSession()
planTasks()
startAgentRun()
sendToAgentRun()
requestApproval()
summarizeSession()
```

The Supervisor should not directly execute arbitrary local shell commands.

## Durable workflows

Workflow state tracks dependency graphs independently of the LLM.

A local run can execute for an extended period. Completion is reported asynchronously by the Node, after which the workflow schedules newly eligible work.
