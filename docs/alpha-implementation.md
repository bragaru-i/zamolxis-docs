# Alpha implementation boundary

The executable `bragaru-i/zamolxis` repository is the behavioral source of truth.
This note describes the Alpha integration implementation, not a deployed service.
The older specifications in this repository remain design/reference material.

The hierarchy is Product -> Repository -> Work Session -> Task -> Workspace ->
Agent Run -> Runtime. Human OIDC authentication and Node enrollment are separate.
One configured HTTPS application origin binds OAuth redirects, QR and bootstrap;
the phone talks to Convex and the Mac connects outbound.

The Alpha implementation adds:

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
without operator configuration. Live deployment validation was blocked by automatic
approval review of `convex dev --once`; no production settings were changed.
