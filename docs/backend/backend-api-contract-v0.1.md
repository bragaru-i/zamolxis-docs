# Backend API Contract v0.1

> Concrete cloud/API contract for the first Zamolxis implementation.

## Boundary model

```text
Web UI
  -> public Convex queries/mutations
       -> application use cases
            -> domain rules

Supervisor / Workflow
  -> internal functions
       -> application use cases

Zamolxis Node
  -> node-scoped functions
       -> command/event/reconciliation use cases
```

Convex functions are already a delivery boundary. Do not add artificial HTTP-style controllers around them.

## Public Orchestrator API

### orchestrator.listConversations — query

Returns the owner's durable top-level conversations. These are not Work Sessions.

### orchestrator.messages — query

```ts
{ conversationId: Id<"orchestratorConversations"> }
```

Returns authorized messages, persisted routing decisions and typed links to canonical work items.

### orchestrator.submit — mutation

```ts
{
  conversationId?: Id<"orchestratorConversations">;
  text: string;
  productId?: Id<"products">;
  repositoryId?: Id<"repositories">;
  idempotencyKey: string;
}
```

Persists the user message and queues one Orchestrator decision. It does not create a Work Session.
The accepted decision may answer, link existing work, store an inert proposal, continue an
authorized Session, create a Session for new durable work, or ask a question. Session creation and
reuse are separate deterministic backend transitions with Product/repository isolation and
idempotency.

## Public session API

### sessions.listMine — query

Input:

```ts
{
  status?: SessionStatus;
  paginationOpts: PaginationOptions;
}
```

Returns paginated `WorkSessionSummaryDto`.

### sessions.get — query

```ts
{ workSessionId: Id<"workSessions"> }
```

Returns session header/context. Tasks, runs and event history are separate reads.

### sessions.create — mutation

```ts
{
  title: string;
  goal: string;
  productId?: Id<"products">;
  repositoryIds?: Id<"repositories">[];
}
```

Creates the Work Session and repository relationships transactionally.

### sessions.cancel — mutation

Validates ownership and session state, requests cancellation of active work, and lets application logic enqueue required stop commands.

Normal completion is not a conversational guess or a manual close gesture. The workflow closes a
Session only when all Tasks are terminal, no Run, approval, verifier, repair or integration is
pending, and every required exact-SHA trust decision is settled. The Orchestrator reports and links
that state; it cannot override it.

## Task API

### tasks.listBySession — query

```ts
{ workSessionId: Id<"workSessions"> }
```

### tasks.create — mutation/internalMutation

```ts
{
  workSessionId: Id<"workSessions">;
  title: string;
  description: string;
  kind: string;
  priority: number;
  runtimePolicy: {
    mode: "auto" | "preferred" | "forced";
    runtime?: string;
  };
  dependencies?: Array<{
    taskId: Id<"tasks">;
    type: "completion" | "success";
  }>;
}
```

Validates dependency ownership and rejects cycles.

### tasks.cancel — mutation

Cancels eligible work and coordinates stop policy for active Runs.

## Workspace API

Workspace provisioning is normally internal. The UI observes it; workflows/application services create it.

### workspaces.listBySession — query

Used for session inspection/debug views.

### internal.workspaces.allocate

```ts
{
  workSessionId: Id<"workSessions">;
  taskId?: Id<"tasks">;
  repositoryId: Id<"repositories">;
  workstationPreference?: Id<"workstations">;
  kind: "worktree" | "integration";
  baseRef: string;
}
```

Behavior:

1. choose an eligible RepositoryLocation
2. create Workspace in REQUESTED
3. create durable `workspace.provision` command
4. return Workspace ID

Convex never executes Git.

### node.workspaces.markProvisioning

Node-authenticated transition after claiming provision command.

### node.workspaces.markReady

```ts
{
  workspaceId: Id<"workspaces">;
  commandId: Id<"commands">;
  localPath: string;
  baseSha: string;
  branchName?: string;
  currentHeadSha: string;
}
```

Validates command, Workstation and state ownership.

### node.workspaces.reportSnapshot

Updates safe observed Git state such as HEAD, dirty flag and changed-file count. It cannot arbitrarily overwrite leases or ownership.

### internal.workspaces.requestCleanup

Checks active Runs, integration dependencies, dirty state and retention policy before enqueuing cleanup.

## Run API

### runs.listBySession — query

Returns bounded/paginated Run summaries.

### runs.get — query

Returns current hot Run state.

### internal.runs.start

```ts
{
  taskId: Id<"tasks">;
  workspaceId: Id<"workspaces">;
  runtimePolicyOverride?: RuntimePolicyDto;
  parentRunId?: Id<"agentRuns">;
}
```

Behavior:

1. load Task and Workspace
2. validate domain state
3. resolve runtime/workstation capability
4. create QUEUED AgentRun
5. acquire/prepare Workspace ownership
6. enqueue `runtime.start`
7. return Run ID

### runs.sendMessage — mutation

Creates durable `runtime.send` command. Browser never talks directly to native runtime.

### runs.stop — mutation

Creates stop intent/command and transitions through STOPPING.

### node.runs.markStarted

Records nativeSessionId, initial HEAD and observed runtime identity.

### node.runs.complete

```ts
{
  runId: Id<"agentRuns">;
  finalHeadSha?: string;
  resultSummary?: string;
  workspaceSnapshot: WorkspaceSnapshotDto;
}
```

Transactionally:

- transition Run
- update Workspace snapshot
- release lease according to policy
- update Task/workflow eligibility
- update Work Session counters/activity

## Event API

### events.listByRun — query

```ts
{
  runId: Id<"agentRuns">;
  paginationOpts: PaginationOptions;
}
```

Uses the run/sequence index.

### node.events.ingestBatch — mutation

```ts
{
  runId: Id<"agentRuns">;
  events: NormalizedRunEventDto[];
}
```

Rules:

- bounded batch
- authenticate Node/Workstation
- deduplicate by event ID
- validate Run ownership
- preserve sequence semantics
- update hot Run activity only for meaningful events
- return acknowledgement cursor

Do not ingest terminal output token-by-token.

## Command API

Commands are typed product intents, not an arbitrary remote-shell endpoint.

### node.commands.listPending — query

Node-authenticated and scoped to its Workstation.

### node.commands.claim — mutation

Atomic PENDING -> CLAIMED.

### node.commands.acknowledge — mutation

CLAIMED -> ACKNOWLEDGED.

### node.commands.complete — mutation

Records command execution result. Entity-specific completion still goes through its domain transition.

### node.commands.fail — mutation

Records stable error code plus diagnostic message.

## Workstation API

### workstations.listMine — query
### node.register — bootstrap mutation
### node.heartbeat — mutation
### node.reconcile — mutation

Reconciliation receives bounded actual local state: active Runs, native session IDs, Workspace leases and Git snapshots.

Never destroy ambiguous local work to force cloud desired state.

## Repository API

### repositories.listByProduct — query
### repositories.create — mutation
### node.repositories.registerLocation — mutation
### node.repositories.verifyLocation — mutation

Only the owning Node reports local filesystem paths.

## Approval API

### approvals.listPendingMine — query
### approvals.resolve — mutation
### internal.approvals.request — internalMutation

Approval resolution resumes orchestration; it does not directly execute a dangerous operation from the browser.

## Supervisor tool surface

Expose narrow product semantics:

```text
findRelevantSessions
findRelevantTickets
getLinkedWorkState
getSessionContext
createWorkSession
continueWorkSession
createTaskPlan
allocateWorkspace
startAgentRun
sendToAgentRun
stopAgentRun
requestApproval
summarizeSession
linkWorkItem
```

Never expose Supervisor tools such as:

```text
executeShell
writeAnyFile
runGit
patchAnyDatabaseDocument
```

The Supervisor expresses intent; deterministic services enforce execution.

## Stable error contract

```ts
type ZamolxisErrorCode =
  | "NOT_FOUND"
  | "FORBIDDEN"
  | "INVALID_STATE"
  | "WORKSPACE_BUSY"
  | "WORKSPACE_MISMATCH"
  | "WORKSTATION_OFFLINE"
  | "RUNTIME_UNAVAILABLE"
  | "COMMAND_CONFLICT"
  | "DEPENDENCY_CYCLE"
  | "APPROVAL_REQUIRED"
  | "RECONCILIATION_REQUIRED";
```

UI behavior keys off code, not message text.

## Idempotency

Mandatory for:

- Workspace provisioning
- runtime start
- runtime stop
- command completion
- event ingestion
- reconciliation
- cleanup

Network retry must never create a second worktree or second native agent process.

## Authorization

Every public function resolves the authenticated User server-side.

Every Node function authenticates device/workstation identity and verifies referenced resources belong to that Workstation where required.

Every internal function is restricted to trusted server workflows/actions.

Client-side visibility is never authorization.
