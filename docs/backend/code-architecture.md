# Code Architecture & Monorepo Structure

> Goal: Zamolxis code must be understandable by a human engineer without first understanding Convex internals, a particular AI runtime, or agent-generated conventions.

## Architectural style

Use a pragmatic **ports-and-adapters / clean architecture** approach.

Do not build ceremonial enterprise layers where every function has a Controller, Service, ServiceImpl and Repository wrapper. Add a boundary only when it has a real responsibility.

The dependency direction is:

```text
Delivery / Framework
        |
        v
   Application
        |
        v
      Domain
        ^
        |
 Infrastructure implements Domain/Application ports
```

The Domain must not import Convex, Next.js, Git CLI, Codex, Claude, Hermes or operating-system process APIs.

## Monorepo

Proposed structure:

```text
zamolxis/
├── apps/
│   ├── web/                         # Next.js product UI
│   └── node/                        # local Zamolxis Node executable
│
├── convex/                          # thin Convex delivery/persistence layer
│   ├── schema.ts
│   ├── sessions.ts
│   ├── tasks.ts
│   ├── workspaces.ts
│   ├── runs.ts
│   ├── commands.ts
│   ├── approvals.ts
│   ├── events.ts
│   ├── workstations.ts
│   └── internal/
│
├── packages/
│   ├── domain/                      # pure business model
│   ├── application/                 # use cases / orchestration
│   ├── contracts/                   # DTOs and wire protocol
│   ├── git/                         # Git/worktree adapter
│   ├── runtime-core/                # runtime port + normalized model
│   ├── runtime-codex/
│   ├── runtime-claude/
│   ├── runtime-hermes/
│   ├── node-core/                   # local Node application logic
│   ├── supervisor/                  # Supervisor policies/tools
│   ├── skills/                      # optional skill definitions/registry
│   └── test-kit/                    # fakes/builders/protocol fixtures
│
├── docs/
└── package.json
```

## 1. Domain

`packages/domain` contains concepts that would still exist if Convex and all current agent runtimes were replaced tomorrow.

```text
packages/domain/src/
├── session/
│   ├── work-session.ts
│   ├── session-status.ts
│   └── session-policy.ts
├── task/
│   ├── task.ts
│   ├── task-status.ts
│   └── dependency-graph.ts
├── workspace/
│   ├── workspace.ts
│   ├── workspace-status.ts
│   ├── workspace-policy.ts
│   └── workspace-lease.ts
├── run/
│   ├── agent-run.ts
│   ├── run-status.ts
│   └── run-policy.ts
├── repository/
│   ├── repository.ts
│   └── repository-location.ts
└── shared/
    ├── errors.ts
    └── result.ts
```

Prefer meaningful functions:

```ts
canStartRun(workspace, run)
canAcquireWorkspaceLease(workspace, runId)
canTransitionRun(from, to)
getReadyTasks(tasks, dependencies)
```

instead of scattering state-transition conditionals across Convex mutations.

Domain code is pure TypeScript and should be fast to unit test.

## 2. Contracts / DTOs

`packages/contracts` is the stable boundary between processes.

```text
packages/contracts/src/
├── node/
│   ├── handshake.dto.ts
│   ├── heartbeat.dto.ts
│   └── reconciliation.dto.ts
├── commands/
│   ├── command.dto.ts
│   ├── provision-workspace.dto.ts
│   ├── start-runtime.dto.ts
│   └── stop-runtime.dto.ts
├── events/
│   ├── event-envelope.dto.ts
│   ├── run-events.dto.ts
│   └── workspace-events.dto.ts
└── runtime/
    ├── capabilities.dto.ts
    └── session.dto.ts
```

A DTO describes data crossing a boundary. It should not contain business logic.

Avoid one giant `types.ts`.

## 3. Application layer

Application code expresses use cases.

```text
packages/application/src/
├── sessions/
│   ├── create-session.use-case.ts
│   └── route-instruction.use-case.ts
├── tasks/
│   ├── create-task.use-case.ts
│   └── advance-task-graph.use-case.ts
├── workspaces/
│   ├── allocate-workspace.use-case.ts
│   ├── acquire-workspace.use-case.ts
│   ├── integrate-workspaces.use-case.ts
│   └── cleanup-workspace.use-case.ts
├── runs/
│   ├── start-run.use-case.ts
│   ├── stop-run.use-case.ts
│   └── complete-run.use-case.ts
└── ports/
    ├── session.repository.ts
    ├── task.repository.ts
    ├── workspace.repository.ts
    ├── run.repository.ts
    ├── command-bus.port.ts
    └── clock.port.ts
```

A use case should read approximately like the product operation:

```ts
export async function startRun(input: StartRunInput, deps: StartRunDeps) {
  const task = await deps.tasks.get(input.taskId);
  const workspace = await deps.workspaces.get(input.workspaceId);

  assertRunCanStart(task, workspace);

  const run = createQueuedRun(...);

  await deps.runs.save(run);
  await deps.commands.enqueue(createStartRuntimeCommand(run, workspace));

  return run;
}
```

Framework mechanics do not belong here.

## 4. Convex layer

Convex files are **delivery + persistence adapters**, not the place for all business logic.

Example:

```text
convex/runs.ts
    |
    | auth + input validation + transaction adapter
    v
application/start-run.use-case.ts
    |
    v
domain policies
```

Conceptually:

```ts
export const start = mutation({
  args: startRunArgs,
  handler: async (ctx, args) => {
    const user = await requireUser(ctx);

    return startRun(args, {
      runs: convexRunRepository(ctx, user),
      tasks: convexTaskRepository(ctx, user),
      workspaces: convexWorkspaceRepository(ctx, user),
      commands: convexCommandBus(ctx, user),
      clock: systemClock,
    });
  },
});
```

Do not create HTTP-style controllers inside Convex just to imitate NestJS. Convex query/mutation/action functions already form the delivery boundary.

## 5. Local Node architecture

```text
apps/node
   |
   v
packages/node-core
   |
   +--> Workspace Manager ----> packages/git
   |
   +--> Runtime Manager ------> runtime adapters
   |
   +--> Policy Engine
   |
   +--> Local Outbox
   |
   '--> Reconciliation
```

Suggested structure:

```text
packages/node-core/src/
├── connection/
├── commands/
│   ├── command-dispatcher.ts
│   └── handlers/
│       ├── provision-workspace.handler.ts
│       ├── start-runtime.handler.ts
│       └── stop-runtime.handler.ts
├── workspace/
│   ├── workspace-manager.ts
│   └── workspace-lock.ts
├── runtime/
│   └── runtime-manager.ts
├── policy/
├── outbox/
└── reconciliation/
```

Here a **command handler** is useful because it is a real delivery boundary from cloud command to local use case.

## 6. Git package

Git is infrastructure.

```text
packages/git/src/
├── git-client.ts
├── git-command-runner.ts
├── repository-inspector.ts
├── worktree-manager.ts
├── workspace-validator.ts
└── types.ts
```

Expose semantic operations:

```ts
interface GitWorkspacePort {
  inspectRepository(path: string): Promise<RepositorySnapshot>;

  createWorktree(input: CreateWorktreeInput): Promise<WorktreeSnapshot>;
  inspectWorktree(path: string): Promise<WorktreeSnapshot>;
  removeWorktree(input: RemoveWorktreeInput): Promise<void>;

  getStatus(path: string): Promise<GitStatus>;
}
```

Application code should not contain strings such as `git worktree add`.

Only the Git adapter translates semantic operations into CLI commands.

## 7. Runtime packages

```text
runtime-core
   |
   +-- runtime-codex
   +-- runtime-claude
   '-- runtime-hermes
```

`runtime-core` owns:

- `AgentRuntime` interface
- capabilities
- normalized runtime events
- start/resume/stop inputs
- native session abstraction

Each integration owns only vendor-specific behavior.

No Codex/Hermes/Claude conditionals should appear in Session, Workspace or Task services.

Bad:

```ts
if (runtime === "codex") { ... }
else if (runtime === "claude") { ... }
```

Good:

```ts
const adapter = runtimeRegistry.get(runtime);
await adapter.start(input);
```

## 8. Skills

Skills are **not core domain entities in v0.1**.

A skill is a reusable capability/instruction exposed to a Supervisor or compatible runtime.

```text
Supervisor
    |
    v
Skill Registry
    |
    +-- code-review
    +-- investigate-tests
    +-- prepare-release
    '-- future skill
```

Core backend only needs an optional reference:

```ts
{
  requestedSkills?: string[]
}
```

Do not put skill-specific columns on Tasks, Workspaces or Runs.

A future `packages/skills` can define:

```ts
interface SkillDefinition {
  id: string;
  version: string;
  description: string;
  requiredCapabilities: string[];
  instructions: string;
}
```

The runtime/supervisor resolves the skill. Workspace and Git behavior remains unchanged.

This allows skills to be added without schema migrations.

## 9. Service naming

Use `Service` only for a cohesive domain/application capability with multiple operations.

Good:

- `WorkspaceManager`
- `RuntimeRegistry`
- `RepositoryInspector`
- `SessionRouter`

Prefer a named use case when there is one operation:

- `startRun()`
- `allocateWorkspace()`
- `completeTask()`

Avoid:

- `RunServiceImpl`
- `CommonService`
- `UtilsService`
- `HelperService`

## 10. Repository interfaces

Repository here means persistence interface, not Git repository.

Use narrow interfaces owned by the application layer:

```ts
interface WorkspaceRepository {
  get(id: WorkspaceId): Promise<Workspace | null>;
  save(workspace: Workspace): Promise<void>;
  findActiveByTask(taskId: TaskId): Promise<Workspace[]>;
}
```

Convex implements these ports.

Do not leak `QueryCtx`, `MutationCtx`, Convex document types or `Id<...>` throughout pure domain code.

## 11. Dependency rules

Allowed:

```text
domain          -> nothing infrastructure-specific
contracts       -> shared validation/types only
application     -> domain + contracts
git             -> domain/contracts where needed
runtime-core    -> domain/contracts where needed
runtime-*       -> runtime-core
node-core       -> application + contracts + git + runtime-core
convex          -> application + domain + contracts
web             -> public API/contracts
```

Forbidden:

```text
domain      -> convex
domain      -> runtime-codex
application -> next
application -> child_process
convex      -> runtime-codex native process APIs
web         -> Git CLI
```

Enforce these boundaries with ESLint import rules or package-level dependency constraints.

## 12. Human readability rules

### One concept per file

Prefer:

```text
allocate-workspace.use-case.ts
workspace.repository.ts
workspace.ts
```

over:

```text
workspace-stuff.ts
helpers.ts
types.ts
```

### Explicit names

Prefer `repositoryLocationId` over `locationId`.

Prefer `workSessionId` over `session`.

### Small public surfaces

Each package exposes a deliberate `index.ts`; internal helpers remain internal.

### Comments explain why

Do not comment obvious syntax.

Good:

```ts
// Preserve dirty workspaces after reconnect: local edits may not exist
// anywhere else and automatic cleanup would cause data loss.
```

### No agent-oriented naming

Do not add abstractions only because an AI generator prefers them. A human engineer should be able to trace:

```text
Convex mutation
 -> use case
 -> domain rule
 -> repository/command port
 -> adapter
```

without jumping through unnecessary wrappers.

## 13. Testing structure

```text
domain
  -> pure unit tests

application
  -> use-case tests with fake repositories

git
  -> integration tests against temporary Git repositories

runtime adapters
  -> contract tests + runtime-specific integration tests

node-core
  -> command/recovery tests with fake runtime + temporary Git

convex
  -> authorization, indexes and transaction integration tests

end-to-end
  -> Control Plane -> Node -> mock/native runtime -> events
```

Every runtime adapter must pass a shared Runtime Adapter contract suite.

Every Workspace implementation must pass a shared Workspace lifecycle suite.

## 14. Architectural test

A new engineer should be able to answer these questions by directory names alone:

- Where is the rule that decides whether a Run may start? -> `domain/run`
- Where is the use case that starts it? -> `application/runs`
- Where does Convex expose it? -> `convex/runs.ts`
- Where is the command handled locally? -> `node-core/commands/handlers`
- Where is Git worktree created? -> `packages/git/worktree-manager.ts`
- Where does a particular runtime start? -> its `runtime-*` adapter

If those answers become ambiguous, the architecture has started to decay.
