# Backend Domain & Convex Schema

This is the target logical schema. Exact Convex syntax may change during implementation, but entity boundaries and invariants should remain.

## Entity graph

```text
User
 └── Workstation
      └── RepositoryLocation

Product
 └── Repository
      └── WorkSession
           └── Task
                └── Workspace
                     └── AgentRun
                          └── RunEvent
```

A Repository is logical/global. A RepositoryLocation says where that repository exists on a particular Workstation.

## workstations

Fields:

```ts
{
  ownerId,
  name,
  status: "online" | "offline" | "degraded",
  nodeVersion,
  platform,
  architecture,
  capabilities,
  lastHeartbeatAt,
  registeredAt,
  revokedAt?
}
```

Indexes:

- by owner
- by status
- by lastHeartbeatAt

## products

```ts
{
  ownerId,
  name,
  slug,
  description?,
  createdAt,
  archivedAt?
}
```

## repositories

Logical repository identity.

```ts
{
  ownerId,
  productId?,
  name,
  remoteUrl?,
  defaultBranch?,
  provider?,
  externalRepositoryId?,
  createdAt
}
```

Do not store a workstation filesystem path here.

## repositoryLocations

Maps a logical Repository to a local clone.

```ts
{
  repositoryId,
  workstationId,
  canonicalPath,
  gitCommonDir?,
  defaultBranch?,
  lastKnownHead?,
  status: "available" | "missing" | "invalid" | "busy",
  verifiedAt
}
```

Unique logical constraint: one canonical location per repository/workstation unless multiple clones are explicitly supported later.

## workSessions

```ts
{
  ownerId,
  productId?,
  title,
  goal,
  status:
    | "planning"
    | "running"
    | "waiting"
    | "needs_input"
    | "completed"
    | "failed"
    | "cancelled",
  contextSummary?,
  createdAt,
  updatedAt,
  completedAt?
}
```

## sessionRepositories

A Work Session may touch more than one repository.

```ts
{
  workSessionId,
  repositoryId,
  role: "primary" | "dependency" | "secondary"
}
```

## tasks

```ts
{
  workSessionId,
  title,
  description,
  kind,
  status:
    | "planned"
    | "blocked"
    | "ready"
    | "running"
    | "waiting"
    | "completed"
    | "failed"
    | "cancelled",
  runtimePolicy: {
    mode: "auto" | "preferred" | "forced",
    runtime?: string
  },
  createdAt,
  startedAt?,
  completedAt?
}
```

## taskDependencies

```ts
{
  taskId,
  dependsOnTaskId,
  type: "completion" | "success"
}
```

No cyclic dependency graph is allowed.

## workspaces

A Workspace is a Zamolxis-owned execution environment.

```ts
{
  workSessionId,
  taskId?,
  repositoryId,
  workstationId,

  kind: "canonical" | "worktree" | "integration",

  state:
    | "requested"
    | "provisioning"
    | "ready"
    | "in_use"
    | "dirty"
    | "integrating"
    | "completed"
    | "cleanup_pending"
    | "removed"
    | "error",

  localPath,
  baseRef,
  baseSha,
  branchName?,
  currentHeadSha?,

  ownerRunId?,
  createdAt,
  updatedAt,
  removedAt?
}
```

Important: cloud stores the Workspace identity and metadata; the Node is authoritative for actual local filesystem state.

## agentRuns

```ts
{
  workSessionId,
  taskId,
  workspaceId,
  workstationId,

  runtime,
  nativeSessionId?,
  parentRunId?,

  status:
    | "queued"
    | "starting"
    | "running"
    | "waiting"
    | "needs_approval"
    | "completed"
    | "failed"
    | "stopping"
    | "stopped"
    | "lost",

  attempt,
  startedAt?,
  heartbeatAt?,
  completedAt?,
  exitReason?,
  resultSummary?,
  finalHeadSha?
}
```

A Run must not start until its Workspace is READY or explicitly reusable IN_USE by that Run.

## runEvents

```ts
{
  runId,
  workstationId,
  sequence,
  eventId,
  type,
  occurredAt,
  receivedAt,
  payload
}
```

Required uniqueness: `(runId, eventId)`.

Prefer per-run monotonic `sequence` so reconnect replay can be ordered.

## commands

```ts
{
  workstationId,
  type,
  targetType,
  targetId,
  idempotencyKey,
  payload,
  status: "pending" | "claimed" | "acknowledged" | "completed" | "failed" | "expired",
  createdAt,
  claimedAt?,
  completedAt?,
  error?
}
```

Required uniqueness: `idempotencyKey`.

## approvals

```ts
{
  workSessionId,
  runId?,
  workstationId?,
  action,
  risk,
  request,
  status: "pending" | "approved" | "rejected" | "expired",
  requestedAt,
  resolvedAt?,
  resolvedBy?
}
```

## artifacts

Artifacts reference useful results without requiring all bytes to live in Convex.

```ts
{
  workSessionId,
  taskId?,
  runId?,
  kind,
  name,
  storage: "local" | "cloud" | "git",
  locator,
  metadata?
}
```

## Invariants

1. An AgentRun references exactly one Workspace.
2. A worktree Workspace belongs to exactly one Repository and Workstation.
3. A Workspace path is assigned by the Node, not an LLM.
4. Two mutating AgentRuns must not concurrently own the same Workspace unless an explicit shared-workspace mode is introduced later.
5. A worktree records its base SHA at provisioning time.
6. Runtime replacement/retry may reuse a Workspace without changing repository identity.
7. Commands and events are idempotent.
8. Cloud status is reconciled against Node truth after reconnect.
