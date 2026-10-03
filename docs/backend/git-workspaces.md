# Git Workspace & Worktree Manager

Git worktree support is a **required backend capability**, not an optional runtime feature.

## Why

Parallel coding agents must not edit the same working directory.

Zamolxis therefore owns the mapping:

```text
Task -> Workspace -> Git Worktree -> cwd -> Agent Run
```

The coding runtime receives a prepared cwd. It does not discover or invent the repository path.

## Directory strategy

A Node may use a managed root such as:

```text
~/.zamolxis/
  workspaces/
    <repository-id>/
      <workspace-id>/
```

The exact root is configurable.

The original clone remains the canonical RepositoryLocation. Managed worktrees live separately.

## Provision worktree

Input:

```ts
{
  repositoryLocationId,
  workspaceId,
  baseRef,
  branchName
}
```

Conceptual procedure:

1. Resolve canonical repository path from Repository Registry.
2. Verify path exists.
3. Verify it belongs to the expected Git repository.
4. Refresh `git worktree list --porcelain`.
5. Resolve and record base SHA.
6. Ensure branch/worktree names are not already owned.
7. Create branch/worktree using Git.
8. Validate the resulting worktree.
9. Persist local workspace record.
10. Emit `workspace.ready`.

The Node should use Git commands directly, not ask a coding agent to create the worktree.

## Naming

Use stable machine-readable identifiers rather than user titles.

Example:

```text
branch:
zam/<session-short-id>/<task-short-id>

path:
~/.zamolxis/workspaces/<repo-id>/<workspace-id>
```

Human-readable task titles remain metadata.

## Pre-run validation

Before every runtime start/resume:

```text
expected repository?
expected worktree path?
expected branch?
expected git common dir?
workspace not owned by another mutating run?
path still registered by git worktree list?
```

At minimum verify with Git-equivalent checks for:

- repository top-level
- current HEAD
- branch/detached state
- worktree registration
- common Git directory

If validation fails, do not start the runtime. Mark Workspace ERROR and reconcile.

## Ownership and locking

A mutating Workspace has one active owner Run.

```text
Workspace READY
    |
 acquire lease
    v
Workspace IN_USE -- ownerRunId
    |
 release lease
    v
Workspace DIRTY / COMPLETED
```

Use both cloud intent and local Node locking. Local locking protects against duplicate commands/reconnect races.

A lease contains:

- workspaceId
- runId
- acquiredAt
- lastRenewedAt
- node instance ID

## Runtime replacement

Workspace ownership is above the runtime.

```text
Workspace W
   |
 Run 1 / Runtime A
   |
 failure
   v
 Run 2 / Runtime B
```

Run 2 may continue in W after Run 1 is stopped and its lease is released. Uncommitted files remain available.

## Dirty state

A successful agent run does not imply a clean Git tree.

On run completion, Node captures:

- HEAD SHA
- branch
- porcelain status summary
- changed paths
- commits created since base
- untracked-file summary

Workspace becomes:

- COMPLETED if policy requirements are met
- DIRTY if useful changes exist but integration/commit policy is incomplete
- ERROR if repository identity is invalid

## Integration Workspace

Parallel tasks must not merge into the canonical working directory directly.

Create a dedicated integration Workspace:

```text
             Base SHA
                |
       +--------+--------+
       |                 |
       v                 v
 Workspace A         Workspace B
 commits A           commits B
       |                 |
       +--------+--------+
                |
                v
       Integration Workspace
                |
          merge/cherry-pick
                |
         +------+------+
         |             |
       clean        conflicts
         |             |
       tests     Integration Task
         |             |
         +------+------+
                |
              result
```

Integration policy may use merge, rebase or cherry-pick depending on project configuration. Do not hard-code one strategy into the domain model.

## Cleanup

Never remove a worktree simply because an Agent Run ended.

Cleanup eligibility requires:

- no active Run owns it
- no pending integration needs it
- required artifacts/commits are captured
- dirty state is handled
- retention policy permits deletion

Cleanup is a separate idempotent command.

## Recovery

On Node startup/reconnect:

1. enumerate managed local workspace records
2. enumerate `git worktree list --porcelain`
3. compare with cloud Workspace records
4. identify orphaned local worktrees
5. identify cloud workspaces missing locally
6. identify stale locks
7. emit reconciliation results
8. never auto-delete ambiguous dirty work

## Acceptance tests

Required tests include:

- two tasks create two different worktrees from the same repository
- both can modify the same source file without filesystem collision
- wrong cwd prevents runtime launch
- duplicate provision command is idempotent
- Node restart preserves workspace identity
- stale lease can be safely reconciled
- runtime A can fail and runtime B can resume same Workspace
- dirty worktree is not automatically deleted
- integration Workspace combines two completed task branches
- conflict creates an explicit integration task/state
