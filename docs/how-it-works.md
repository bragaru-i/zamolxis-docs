# How Zamolxis works today

State as of **2026-10-07** (private Alpha). The executable repository
[`bragaru-i/zamolxis`](https://github.com/bragaru-i/zamolxis) is the source of truth; its
`docs/alpha-status.md` tracks every change and known gap in detail. This page explains the
product as it actually behaves now, and the ideas it is built on. Older documents in this
repository are design notes; where they differ from this page, this page wins (see
[Where the design notes differ](#where-the-design-notes-differ)).

## In one paragraph

You talk to Zamolxis from your phone or laptop. Your own computers do the coding: AI coding
agents (Codex or Claude Code) work in isolated copies of your repository, a second agent checks
the result independently, the computer runs the project's own checks, and only work that passes
is offered to you as a pull request. Anything risky (the internet, files outside the project,
controlling the computer) waits for your approval. You can see at any moment which agent is
working, on which model, what it is doing, what it costs in tokens, and why something failed.

## The parts

```text
 Phone / laptop browser                      Your computers (Mac, Linux)
 ┌────────────────────────┐                  ┌──────────────────────────────┐
 │ Web app (PWA)          │                  │ Zamolxis Node (a service)    │
 │ zamolxis.bragaru.cc    │                  │  - git worktrees per task    │
 └──────────┬─────────────┘                  │  - runs Codex / Claude Code  │
            │                                │  - runs the project's checks │
            v                                │  - pushes trusted branches   │
 ┌────────────────────────┐   outbound only  └──────────────┬───────────────┘
 │ Control plane (Convex) │ <───────────────────────────────┘
 │ state, commands,       │   (the computer connects out; nothing connects in)
 │ authorization, events  │
 └────────────────────────┘
```

- **Web app**: Next.js, deployed on Vercel, installable on the iPhone home screen. It talks only
  to the control plane, never directly to your computers.
- **Control plane**: Convex. It stores every Product, Session, Task, Run, approval and piece of
  evidence, queues commands for computers, and decides what is allowed. Sign-in is Google, and
  an admin must approve each account.
- **Node**: a background service on each computer (launchd on macOS, user systemd on Linux). It
  is paired once with a QR code, has its own revocable identity (separate from yours), and keeps
  running when the terminal is closed.
- **Runtimes**: the coding agents themselves. **Codex** (through `codex app-server`) and **Claude
  Code** (through the `claude` CLI) are implemented. Hermes is only a reserved name. A runtime is
  not a model: each role picks a runtime *and* a model.

## From a message to a pull request

```text
Home chat ──"Open Work Session"──> Supervisor ──plan──> Builders (≤3, parallel)
                                                          │ candidate commit
                                                          v
                         Repair (≤2) <──fail── Verifier + project checks ──pass──> Trust
                                                                                    │
                                     Pull request <──"Open pull request"── Integration branch
```

1. **Home chat (the Orchestrator).** Ask anything. It answers questions and status requests from
   the control plane's real state, with links to Sessions, approvals, pull requests and running
   agents. When you ask for work it writes an inert *proposal*; nothing starts until you press
   **Open Work Session and start planning** and choose the product, repository and (when several
   have it) the computer.
2. **Session Supervisor.** A read-only agent looks at the repository at an exact commit (the
   plan is bound to that commit) and either answers, asks a question, proposes tasks or, when you
   explicitly asked for execution, delegates them. Each task names the exact files to change and
   read, so builders do not waste tokens searching. The backend validates every plan (structure,
   no dependency cycles) before anything runs.
3. **Builders.** Up to three tasks run in parallel, each in its own git worktree on the chosen
   computer. Before a builder starts, the Node prepares the worktree outside the sandbox: it
   installs the project's dependencies from the lockfile and provides the exact pnpm version the
   project pins. The builder's commit is a **candidate**, never trusted by itself.
4. **Verifier and checks.** An independent Verifier runs in its own worktree at the exact
   candidate commit. It sees the acceptance criteria and public failure evidence, never the
   builder's private reasoning. The Node also runs the repository's own check scripts. Every
   result is recorded against that exact commit.
5. **Trust and repair.** Trust is decided by rules, not by an AI. If the evidence fails, a
   Repair agent gets the failure and tries again (at most two repairs), and the new candidate is
   verified again. History is kept.
6. **Integration and publishing.** A trusted commit goes to a local integration branch. When you
   press **Open pull request**, the Node pushes that exact commit to a new `zamolxis/...` branch
   (never forced, never to main) with that repository's own GitHub token and opens the PR. You
   merge it. This full path was proven in production with PR #123.

## The ideas it is built on

- **Local-first.** Code, credentials and execution stay on your computers. The cloud holds
  state and decisions, not your repositories.
- **AI proposes, the backend authorizes.** Agents suggest plans and changes; the control plane
  enforces ownership, capacity, state transitions and what may run.
- **A candidate is not trust.** Nothing an agent says about its own work counts. Trust comes from
  an independent check at the exact commit, by deterministic rules.
- **Isolation.** Every task works in its own worktree. Your main checkout is never an agent
  workspace and is never changed by tests.
- **You decide anything risky.** Agents work inside a sandbox; leaving it needs your approval.
- **Roles, runtimes and models are configurable.** Settings → Agents sets the runtime, model,
  reasoning effort, instructions and approval policy for each role (Orchestrator,
  Supervisor, Builder, Verifier, Repair; Integration has no agent in Alpha). Each run keeps a snapshot of the settings it started with.
- **Spend tokens carefully.** Every model call resends the whole conversation, so tasks name
  exact files, agents never read whole handbooks, and large requests are split into small
  independent tasks. Token use is visible everywhere.
- **Tell the truth, in plain words.** Statuses mean what they say; a failure says who failed,
  on which model, when and why; untested things are called untested.

## Safety: sandbox and approvals

| | Codex agents | Claude Code agents |
|---|---|---|
| Can write | its worktree and its own private temporary folder | its worktree and temporary folders |
| Internet | off; asks you | off; asks you |
| Ordinary commands (scripts, tests, builds) | run without asking | run without asking, including commands with variables or loops |
| Leaving the sandbox | asks you | impossible (a local server or browser fails instead of asking) |

- **Approvals** appear as toasts on every screen, including the open Session and Run detail, with
  Approve, Approve for run (for low and medium risk commands Codex can remember), Reject and Open
  session. Critical requests need a second tap. A "Needs approval ›" chip on a run brings its
  request back to the top. Unanswered requests are rejected after 30 minutes.
- **Approval policy**: a Builder or Repair profile can let the backend approve low (or low and
  medium) risk commands automatically. High and critical requests always wait for you.
- The Supervisor and Verifier are read-only; their requests to leave the sandbox are always
  refused.
- Agents never receive GitHub tokens. Activity summaries are redacted for secrets before they
  leave the computer. Screenshots agents save as proof are uploaded as taken.

## Seeing what happens

- **Web app**: *Working now* on Home lists every running agent with its runtime, model, current
  activity, time and tokens. A Session shows its tasks on a Plan → Build → Check → Fix → Ready map.
  Run detail shows the activity timeline, files changed, the execution trace and token usage
  (fresh, cached and output). Settings → Usage shows tokens by role and model. Failures show
  "Builder · Codex · model · failed at 14:03" and the provider's reason.
- **Terminal**: `pnpm zamolxis watch` on a computer shows that Node's live activity, tokens per
  run and today's total, warns when a run passes 500k tokens, and names who failed and why.
  Ctrl+C closes the window; the Node keeps running.

## Running the computers

In the computer's `zamolxis` checkout:

| Command | What it does |
|---|---|
| `pnpm zamolxis setup` | First-time setup and repair: checks prerequisites, pairs with a QR code, chooses repositories, installs the service |
| `pnpm zamolxis update` | Pulls the latest code, installs dependencies, waits until the website runs that version, restarts the service and confirms it is back online |
| `pnpm zamolxis watch` | Live activity and token totals |
| `pnpm zamolxis github-token <owner/repo>` | Adds the token a repository publishes pull requests with |
| `pnpm zamolxis doctor` | Checks prerequisites |

Every change merged to `main` that passes CI deploys the backend and the web app automatically.
The Nodes are updated separately with `pnpm zamolxis update`; restarting the service is enough,
never the computer.

## Status and known limits (2026-10-07)

- **Private Alpha is complete**: the release gate (a new production pull request opened through
  Zamolxis) passed with PR #123.
- **Postponed checks** (#130): a real iPhone (camera, home screen, offline), a second Google
  account and a second computer.
- **Not yet measured**: how many approval prompts and tokens a real owner task needs after the
  sandbox fixes of 2026-10-07.
- **Known limits**: Codex agents still ask before starting a local server or browser, and Claude
  agents cannot do that at all; no provider reports cost, so cost shows "Subscription"; the
  Supervisor's tokens appear when it answers, not while it thinks; links to external tickets
  (GitHub, Linear) need a connector.
- **After Alpha** (GitHub issues labelled `post-alpha`): Hermes and more runtimes (#7), integration
  and parallel coding across tasks (#8), security hardening (#9), deeper verification and
  execution-trace work (#26, #28–#32), and a workflow graph view (#23).

## Where the design notes differ

The documents dated 2026-10-03 describe the intended design. Where they disagree with reality:

- **Frontend**: the app uses its own component package (`packages/ui`, plain CSS tokens), not
  shadcn/ui or Tailwind; there is no React Flow graph yet.
- **Supervisor and Orchestrator** are real model calls on your computer (read-only), not
  deterministic boundaries; the backend still validates and authorizes everything they propose.
- **Claude Code** is implemented next to Codex; Hermes is not.
- **Approvals** are bridged from the agents to the phone and work today.
- **Publishing** a pull request is implemented, as an explicit owner action.
- **Chats**: Home has many chats (listed, renamed, deleted), not one conversation per owner.
