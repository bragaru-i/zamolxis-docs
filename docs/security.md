# Security and Permissions

## Trust model

Zamolxis separates cloud coordination from local authority.

The control plane may request an operation. The local Node decides whether that operation is allowed.

## Principles

- outbound-only workstation connection
- per-device authentication
- explicit workspace allowlists
- minimum required filesystem access
- no cloud storage of local credentials
- no automatic upload of complete repositories
- command policy enforcement on the Node
- revocable workstation registration
- auditable approval decisions

## Permission levels

A possible initial policy:

```text
SAFE
read repository
edit repository
run tests
run linters
create worktree
local commits

REQUIRES POLICY / OPTIONAL APPROVAL
install dependencies
network tools
access external directories

REQUIRES HUMAN APPROVAL
merge protected branches
production deploy
database migration
destructive commands
credential/security changes
```

Policies are configurable per workspace.

## Cloud data minimization

Store normalized operational metadata and the transcript/context required for the UI.

Avoid sending secrets, environment files, full repository snapshots or unbounded raw terminal streams to the control plane.
