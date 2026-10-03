# Convex Schema v0.1 — Build Specification

> Status: proposed backend contract for the first implementation.
>
> This document is intentionally concrete. An implementation agent may translate it directly into `convex/schema.ts`, while preserving the invariants below.

## Design rules

1. IDs between Convex entities use `v.id("<table>")`, not arbitrary strings.
2. External/native IDs remain strings.
3. Hot current state is stored on the owning entity; history is append-only.
4. Large event/transcript streams are paginated and never `collect()`ed unbounded.
5. Every user-facing list has an index matching its access pattern.
6. Repository identity is global/logical; filesystem paths belong to Repository Locations and Workspaces.
7. Workspace is a first-class entity between Task and Agent Run.
8. The local Node is authoritative for filesystem/process truth; Convex is authoritative for desired/durable product state.
9. Denormalized summary fields are allowed on hot UI paths when maintained transactionally.
10. Raw secrets, repository contents and environment files are not stored in these tables.

## Proposed `convex/schema.ts`

```ts
import { defineSchema, defineTable } from "convex/server";
import { v } from "convex/values";

const workstationStatus = v.union(
  v.literal("online"),
  v.literal("offline"),
  v.literal("degraded"),
  v.literal("revoked"),
);

const sessionStatus = v.union(
  v.literal("planning"),
  v.literal("running"),
  v.literal("waiting"),
  v.literal("needs_input"),
  v.literal("completed"),
  v.literal("failed"),
  v.literal("cancelled"),
);

const taskStatus = v.union(
  v.literal("planned"),
  v.literal("blocked"),
  v.literal("ready"),
  v.literal("running"),
  v.literal("waiting"),
  v.literal("completed"),
  v.literal("failed"),
  v.literal("cancelled"),
);

const workspaceKind = v.union(
  v.literal("canonical"),
  v.literal("worktree"),
  v.literal("integration"),
);

const workspaceStatus = v.union(
  v.literal("requested"),
  v.literal("provisioning"),
  v.literal("ready"),
  v.literal("in_use"),
  v.literal("dirty"),
  v.literal("integrating"),
  v.literal("completed"),
  v.literal("cleanup_pending"),
  v.literal("removed"),
  v.literal("error"),
);

const runStatus = v.union(
  v.literal("queued"),
  v.literal("starting"),
  v.literal("running"),
  v.literal("waiting"),
  v.literal("needs_approval"),
  v.literal("completed"),
  v.literal("failed"),
  v.literal("stopping"),
  v.literal("stopped"),
  v.literal("lost"),
);

const commandStatus = v.union(
  v.literal("pending"),
  v.literal("claimed"),
  v.literal("acknowledged"),
  v.literal("completed"),
  v.literal("failed"),
  v.literal("expired"),
);

export default defineSchema({
  // Auth provider owns identity. This table stores Zamolxis profile data only.
  users: defineTable({
    authSubject: v.string(),
    displayName: v.optional(v.string()),
    createdAt: v.number(),
  })
    .index("by_auth_subject", ["authSubject"]),

  workstations: defineTable({
    ownerId: v.id("users"),
    name: v.string(),
    status: workstationStatus,

    nodeVersion: v.optional(v.string()),
    nodeInstanceId: v.optional(v.string()),
    platform: v.optional(v.string()),
    architecture: v.optional(v.string()),

    // Compact advertised capability summary. Detailed runtime records live below.
    capabilityRevision: v.optional(v.string()),

    lastHeartbeatAt: v.optional(v.number()),
    registeredAt: v.number(),
    revokedAt: v.optional(v.number()),
  })
    .index("by_owner", ["ownerId"])
    .index("by_owner_status", ["ownerId", "status"])
    .index("by_status_heartbeat", ["status", "lastHeartbeatAt"]),

  runtimeInstallations: defineTable({
    workstationId: v.id("workstations"),
    runtime: v.string(),              // "codex", "claude", "hermes", future values
    version: v.optional(v.string()),
    status: v.union(
      v.literal("available"),
      v.literal("unavailable"),
      v.literal("degraded"),
    ),
    capabilities: v.array(v.string()),
    detectedAt: v.number(),
    metadata: v.optional(v.any()),
  })
    .index("by_workstation", ["workstationId"])
    .index("by_workstation_runtime", ["workstationId", "runtime"]),

  products: defineTable({
    ownerId: v.id("users"),
    name: v.string(),
    slug: v.string(),
    description: v.optional(v.string()),
    archivedAt: v.optional(v.number()),
    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_owner", ["ownerId"])
    .index("by_owner_slug", ["ownerId", "slug"]),

  repositories: defineTable({
    ownerId: v.id("users"),
    productId: v.optional(v.id("products")),
    name: v.string(),

    remoteUrl: v.optional(v.string()),
    provider: v.optional(v.string()),
    externalRepositoryId: v.optional(v.string()),
    defaultBranch: v.optional(v.string()),

    createdAt: v.number(),
    updatedAt: v.number(),
  })
    .index("by_owner", ["ownerId"])
    .index("by_product", ["productId"])
    .index("by_owner_remote", ["ownerId", "remoteUrl"]),

  repositoryLocations: defineTable({
    repositoryId: v.id("repositories"),
    workstationId: v.id("workstations"),

    canonicalPath: v.string(),
    gitCommonDir: v.optional(v.string()),
    defaultBranch: v.optional(v.string()),
    lastKnownHead: v.optional(v.string()),

    status: v.union(
      v.literal("available"),
      v.literal("missing"),
      v.literal("invalid"),
      v.literal("busy"),
    ),

    verifiedAt: v.optional(v.number()),
    updatedAt: v.number(),
  })
    .index("by_repository", ["repositoryId"])
    .index("by_workstation", ["workstationId"])
    .index("by_repository_workstation", ["repositoryId", "workstationId"]),

  workSessions: defineTable({
    ownerId: v.id("users"),
    productId: v.optional(v.id("products")),

    title: v.string(),
    goal: v.string(),
    status: sessionStatus,

    // Hot UI / Supervisor context.
    contextSummary: v.optional(v.string()),
    currentPlanSummary: v.optional(v.string()),

    // Denormalized counters for My Work.
    activeRunCount: v.number(),
    completedTaskCount: v.number(),
    totalTaskCount: v.number(),
    needsInputCount: v.number(),

    lastActivityAt: v.number(),
    createdAt: v.number(),
    updatedAt: v.number(),
    completedAt: v.optional(v.number()),
  })
    .index("by_owner_activity", ["ownerId", "lastActivityAt"])
    .index("by_owner_status_activity", ["ownerId", "status", "lastActivityAt"])
    .index("by_product_activity", ["productId", "lastActivityAt"]),

  sessionRepositories: defineTable({
    workSessionId: v.id("workSessions"),
    repositoryId: v.id("repositories"),
    role: v.union(
      v.literal("primary"),
      v.literal("dependency"),
      v.literal("secondary"),
    ),
  })
    .index("by_session", ["workSessionId"])
    .index("by_repository", ["repositoryId"])
    .index("by_session_repository", ["workSessionId", "repositoryId"]),

  tasks: defineTable({
    workSessionId: v.id("workSessions"),

    title: v.string(),
    description: v.string(),
    kind: v.string(),
    status: taskStatus,

    runtimePolicyMode: v.union(
      v.literal("auto"),
      v.literal("preferred"),
      v.literal("forced"),
    ),
    runtimePolicyRuntime: v.optional(v.string()),

    priority: v.number(),

    createdAt: v.number(),
    updatedAt: v.number(),
    startedAt: v.optional(v.number()),
    completedAt: v.optional(v.number()),
  })
    .index("by_session", ["workSessionId"])
    .index("by_session_status", ["workSessionId", "status"])
    .index("by_session_priority", ["workSessionId", "priority"]),

  taskDependencies: defineTable({
    workSessionId: v.id("workSessions"),
    taskId: v.id("tasks"),
    dependsOnTaskId: v.id("tasks"),
    type: v.union(v.literal("completion"), v.literal("success")),
  })
    .index("by_task", ["taskId"])
    .index("by_dependency", ["dependsOnTaskId"])
    .index("by_session", ["workSessionId"]),

  workspaces: defineTable({
    workSessionId: v.id("workSessions"),
    taskId: v.optional(v.id("tasks")),
    repositoryId: v.id("repositories"),
    repositoryLocationId: v.id("repositoryLocations"),
    workstationId: v.id("workstations"),

    kind: workspaceKind,
    status: workspaceStatus,

    // Assigned by Node. Not chosen by Supervisor/runtime.
    localPath: v.optional(v.string()),

    baseRef: v.string(),
    baseSha: v.optional(v.string()),
    branchName: v.optional(v.string()),
    currentHeadSha: v.optional(v.string()),

    // Lease/ownership.
    ownerRunId: v.optional(v.id("agentRuns")),
    leaseNodeInstanceId: v.optional(v.string()),
    leaseAcquiredAt: v.optional(v.number()),
    leaseRenewedAt: v.optional(v.number()),

    dirty: v.boolean(),
    changedFileCount: v.number(),

    createdAt: v.number(),
    updatedAt: v.number(),
    removedAt: v.optional(v.number()),
    errorCode: v.optional(v.string()),
    errorMessage: v.optional(v.string()),
  })
    .index("by_session", ["workSessionId"])
    .index("by_task", ["taskId"])
    .index("by_workstation_status", ["workstationId", "status"])
    .index("by_repository_status", ["repositoryId", "status"])
    .index("by_owner_run", ["ownerRunId"]),

  agentRuns: defineTable({
    workSessionId: v.id("workSessions"),
    taskId: v.id("tasks"),
    workspaceId: v.id("workspaces"),
    workstationId: v.id("workstations"),

    runtime: v.string(),
    nativeSessionId: v.optional(v.string()),
    parentRunId: v.optional(v.id("agentRuns")),

    status: runStatus,
    attempt: v.number(),

    // Human-readable live state for UI; not full transcript.
    activityLabel: v.optional(v.string()),
    resultSummary: v.optional(v.string()),
    exitReason: v.optional(v.string()),

    startedAt: v.optional(v.number()),
    lastActivityAt: v.number(),
    heartbeatAt: v.optional(v.number()),
    completedAt: v.optional(v.number()),

    initialHeadSha: v.optional(v.string()),
    finalHeadSha: v.optional(v.string()),
  })
    .index("by_session_activity", ["workSessionId", "lastActivityAt"])
    .index("by_session_status", ["workSessionId", "status"])
    .index("by_task", ["taskId"])
    .index("by_workspace", ["workspaceId"])
    .index("by_workstation_status", ["workstationId", "status"])
    .index("by_parent", ["parentRunId"])
    .index("by_native_session", ["workstationId", "runtime", "nativeSessionId"]),

  runEvents: defineTable({
    runId: v.id("agentRuns"),
    workstationId: v.id("workstations"),

    eventId: v.string(),
    sequence: v.number(),
    type: v.string(),

    occurredAt: v.number(),
    payload: v.any(),
  })
    .index("by_run_sequence", ["runId", "sequence"])
    .index("by_run_event_id", ["runId", "eventId"])
    .index("by_workstation_time", ["workstationId", "occurredAt"]),

  commands: defineTable({
    workstationId: v.id("workstations"),

    type: v.string(),
    targetType: v.string(),
    targetId: v.string(),

    idempotencyKey: v.string(),
    status: commandStatus,

    payload: v.any(),
    result: v.optional(v.any()),
    error: v.optional(v.string()),

    createdAt: v.number(),
    claimedAt: v.optional(v.number()),
    acknowledgedAt: v.optional(v.number()),
    completedAt: v.optional(v.number()),
    expiresAt: v.optional(v.number()),
  })
    .index("by_workstation_status", ["workstationId", "status"])
    .index("by_workstation_created", ["workstationId", "createdAt"])
    .index("by_idempotency_key", ["idempotencyKey"]),

  approvals: defineTable({
    ownerId: v.id("users"),
    workSessionId: v.id("workSessions"),
    runId: v.optional(v.id("agentRuns")),
    workstationId: v.optional(v.id("workstations")),

    action: v.string(),
    risk: v.union(
      v.literal("low"),
      v.literal("medium"),
      v.literal("high"),
      v.literal("critical"),
    ),

    request: v.any(),
    status: v.union(
      v.literal("pending"),
      v.literal("approved"),
      v.literal("rejected"),
      v.literal("expired"),
    ),

    requestedAt: v.number(),
    resolvedAt: v.optional(v.number()),
    resolvedBy: v.optional(v.id("users")),
  })
    .index("by_owner_status", ["ownerId", "status"])
    .index("by_session", ["workSessionId"])
    .index("by_run", ["runId"]),

  artifacts: defineTable({
    workSessionId: v.id("workSessions"),
    taskId: v.optional(v.id("tasks")),
    runId: v.optional(v.id("agentRuns")),

    kind: v.string(),
    name: v.string(),

    storage: v.union(
      v.literal("local"),
      v.literal("convex_storage"),
      v.literal("git"),
    ),
    locator: v.string(),
    metadata: v.optional(v.any()),

    createdAt: v.number(),
  })
    .index("by_session", ["workSessionId"])
    .index("by_task", ["taskId"])
    .index("by_run", ["runId"]),

  sessionDecisions: defineTable({
    workSessionId: v.id("workSessions"),
    taskId: v.optional(v.id("tasks")),
    runId: v.optional(v.id("agentRuns")),

    category: v.string(),
    summary: v.string(),
    rationale: v.optional(v.string()),

    createdAt: v.number(),
  })
    .index("by_session", ["workSessionId"])
    .index("by_task", ["taskId"]),

  nodeEventCursors: defineTable({
    workstationId: v.id("workstations"),
    nodeInstanceId: v.string(),
    runId: v.id("agentRuns"),
    lastSequence: v.number(),
    updatedAt: v.number(),
  })
    .index("by_workstation", ["workstationId"])
    .index("by_run", ["runId"])
    .index("by_instance_run", ["nodeInstanceId", "runId"]),
});
```

## Why Workspace has both taskId and ownerRunId

They describe different ownership layers.

- `taskId`: why the Workspace exists.
- `ownerRunId`: which Run currently holds the mutation lease.

A runtime can fail and a replacement Run can continue the same Workspace without changing the Task.

## Why RepositoryLocation exists

Do not model this:

```text
Repository.localPath
```

because the same logical repository can exist at different paths on different workstations.

Correct model:

```text
Repository
   |
   +-- RepositoryLocation / Workstation A / path A
   '-- RepositoryLocation / Workstation B / path B
```

Workspaces are provisioned from one concrete RepositoryLocation.

## Hot state vs history

### Hot documents

Used by realtime UI:

- workSessions
- tasks
- workspaces
- agentRuns
- workstations
- approvals

These contain current status and small summaries.

### Append-only/history

- runEvents
- sessionDecisions
- artifacts

The UI subscribes only to the small visible range it needs.

## Query shapes that must exist

### My Work

Input: authenticated owner, optional status.

Uses:

```text
workSessions.by_owner_activity
workSessions.by_owner_status_activity
```

Paginated newest activity first.

### Work Session screen

Fetch separately:

1. Work Session header.
2. Tasks by `tasks.by_session`.
3. Active/recent Runs by `agentRuns.by_session_activity`.
4. Workspaces by `workspaces.by_session`.
5. Pending approvals for the session.

Do not return the entire event history in the session query.

### Agent Run live view

Header comes from `agentRuns`.

Activity feed:

```text
runEvents.by_run_sequence
```

Paginated. A client may subscribe to a bounded latest window while older events are loaded on demand.

### Node command poll/subscription

```text
commands.by_workstation_status(workstationId, "pending")
```

The Node claims commands through a mutation that atomically verifies current status.

### Recovery

Relevant indexes:

```text
agentRuns.by_workstation_status
workspaces.by_workstation_status
repositoryLocations.by_workstation
commands.by_workstation_status
```

## Mutation boundaries

Do not expose generic "patch document" mutations.

Prefer domain mutations such as:

```text
sessions.create
sessions.setStatus

tasks.create
tasks.addDependency
tasks.markReady
tasks.complete

workspaces.request
workspaces.markProvisioning
workspaces.markReady
workspaces.acquireLease
workspaces.releaseLease
workspaces.markDirty
workspaces.requestCleanup

runs.queue
runs.markStarting
runs.markRunning
runs.recordActivity
runs.complete
runs.fail
runs.markLost

commands.enqueue
commands.claim
commands.acknowledge
commands.complete
commands.fail

events.ingestBatch

approvals.request
approvals.resolve
```

Each mutation validates allowed state transitions.

## Event ingestion

Nodes should batch small groups of events instead of issuing one mutation for every terminal token.

`events.ingestBatch` should:

1. authenticate the Workstation/Node.
2. validate Run/Workstation ownership.
3. deduplicate each event using `by_run_event_id`.
4. insert new events.
5. update `agentRuns.lastActivityAt` and `activityLabel` only when appropriate.
6. update a cursor/acknowledgement.
7. avoid storing token-by-token stdout as semantic events.

High-volume raw logs can remain local or use a separate bounded/log-storage strategy.

## Counters on Work Session

`activeRunCount`, `completedTaskCount`, `totalTaskCount`, and `needsInputCount` exist for the My Work screen.

They are denormalized and must be updated transactionally by domain mutations. They are not independent sources of truth.

A periodic repair/reconciliation function may recompute them for integrity.

## Uniqueness

Convex indexes are used to look up candidate unique records, but uniqueness must be enforced inside the write mutation.

Required logical uniqueness:

- users.authSubject
- products: (ownerId, slug)
- repositoryLocations: (repositoryId, workstationId)
- sessionRepositories: (workSessionId, repositoryId)
- taskDependencies: (taskId, dependsOnTaskId)
- runtimeInstallations: (workstationId, runtime)
- commands.idempotencyKey
- runEvents: (runId, eventId)
- nodeEventCursors: (nodeInstanceId, runId)

## Deliberately not in v0.1

Avoid premature schema complexity for:

- organizations/teams/RBAC
- billing
- marketplace skills
- cloud execution nodes
- arbitrary workflow DSL
- vector embeddings for every event
- full Git object modeling
- PR/review provider mirroring
- long-term raw terminal log storage

These can be added without breaking the core Session → Task → Workspace → Run model.
