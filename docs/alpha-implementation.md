# Alpha implementation boundary

The executable `bragaru-i/zamolxis` repository is the behavioral source of truth.
This note describes the Alpha integration implementation, not a deployed service.
The older specifications in this repository remain design/reference material.

The execution hierarchy is Product -> Repository -> Work Session -> Task -> Workspace ->
Agent Run -> Runtime. An owner-level Orchestrator Conversation sits above Work Sessions and is
the primary human entry point. Human authentication now uses Convex Auth with Google;
Node enrollment remains separate. This supersedes the external OIDC frontend flow.
Convex owns human accounts/sessions and the operator grants or revokes product
access via `users.accessStatus` (`allowed`, `pending`, `blocked`). Missing status
denies access; Google sign-in does not self-authorize. All product APIs enforce
the database gate and retain ownership isolation. Blocking an owner also denies
Node cloud operations and credential refresh, without killing local processes.
See the executable repository's `docs/google-auth-access.md` for current setup.
Google OAuth still requires a Google Cloud client; no Auth0 account is required.
One configured HTTPS application origin binds OAuth redirects, QR and bootstrap;
the phone talks to Convex and the Mac connects outbound.

The Alpha implementation adds:

- a durable owner-level Orchestrator conversation whose status/architecture answers do not create
  Work Sessions, Tasks or Runs;
- persisted `answer`, `create` and `continue` routing decisions plus typed links to Sessions, pending
  approvals, pull requests, Tasks needing the owner, trust decisions and active Runs; explicit
  execution creates work, while "continue"/"do it" can reuse a recently linked Session in the
  selected repository;
- SHA/digest-bound discovery in a planning worktree before persisted Tasks;
- deterministic text planning (one Task) or a validated structured Task DAG;
- server reservation of three Builder/Repair slots and one Verifier slot;
- Product/global AgentProfile resolution, disabled fallback, immutable Run
  configuration, runtime/model/effort propagation and reported usage ingestion;
- candidate commits from isolated Builder worktrees, without self-certification;
- separate read-only native Verifier Runs/worktrees at exact candidate SHAs;
- executable repository checks, persisted evidence and deterministic trust policy;
- at most two repairs, new candidates, preserved history and re-verification;
- a dedicated local integration branch at a trusted exact SHA, Git artifact and
  trust-aware Task/Session completion;
- mobile task phases, repair/failure reasons and a standalone app manifest.

The home UI renders the Orchestrator transcript above the Sessions list and uses a **Send** composer.
Questions remain in this transcript. Explicit work is handed to the existing Session Supervisor,
which runs with the configured Supervisor runtime/model and can answer, ask, propose or delegate.
Settings configures runtime, model, reasoning effort and instructions for Supervisor, Builder,
Verifier, Repair and Integration roles.

Current Orchestrator limits are explicit: status answers link Sessions, pending approvals, pull
requests, attention Tasks (`needs_input`, `trust_failed`, `ready_for_integration`, `failed`), their
trust decisions and active Runs for the five most recent Sessions in scope. Link status is a
snapshot from answer time; the linked view is canonical. External-ticket links need a connector and
are not implemented, and there are no dedicated canonical cards beyond link buttons. The top-level
answer/router is deterministic and is not a separately configured model-backed role. There is one
active conversation per owner in Alpha; conversation listing/archiving is still target design.

Downstream Tasks inherit trusted prerequisite commits; multiple prerequisites
are merged inside their new worktree. Conflicts preserve local state and require
input. Independent output branches are not automatically merged into one release;
use a final dependent Task to create and verify the combined result.

Session completion means all required Tasks have prepared trusted local
integration branches and there are no active Runs. It does not mean protected
main is merged or candidate PRs are published. Those are human actions in Alpha.
Supervisor and Integration are deterministic boundaries; their profile roles are
reserved for future runtime execution. Claude/Hermes native adapters, semantic
LLM planning, native process resumption, approval bridging and automatic publishing
are not implemented. Alpha executes static/test/behavioral evidence; other
modalities exist in the schema/manual API. Repository check scripts run with local
Node authority and require trusted repository code.

## Evidence

Executable repository checks: `pnpm check` (lint, boundaries, workspace/Convex
TypeScript, unit/integration tests and production build).

Disposable Git acceptance covers failed trust, Repair, re-verification and local
integration. Another test proves two Builders execute concurrently and a dependent
Task receives both trusted commits. Native acceptance on 2026-10-05 used Codex
0.160.0 with an auth-only temporary profile, actual Builder edits, a candidate
commit, a separate native Verifier, executed checks, trust and local integration.
A second authenticated run deliberately failed the first candidate, then executed
native Repair, a new SHA, a second independent native Verifier and successful
trust/integration. Failed trust history remained intact. Canonical HEAD/status
stayed unchanged. Temporary credentials were removed.

Reproduce the native path in the executable repository:

```sh
ZAMOLXIS_CODEX_ACCEPTANCE=1 pnpm exec vitest run tests/control-plane-loop.test.ts -t 'runs text intent'
ZAMOLXIS_CODEX_ACCEPTANCE=1 ZAMOLXIS_CODEX_REPAIR_ACCEPTANCE=1 pnpm exec vitest run tests/control-plane-loop.test.ts -t 'runs text intent'
```

The test uses actual Convex function implementations in `convex-test` and fixture
identities. Public phone/OIDC/pairing/deployed-device/launchd E2E is not established
without operator configuration. Google browser OAuth is not yet validated. Live deployment validation was blocked by automatic
approval review of `convex dev --once`; no production settings were changed.
