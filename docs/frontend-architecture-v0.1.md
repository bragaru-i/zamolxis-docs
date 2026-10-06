# Frontend Architecture v0.1

## Stack

```text
Next.js App Router
TypeScript
Tailwind CSS
shadcn/ui
Convex React
React Flow (@xyflow/react)
React Flow UI
TanStack Virtual
Lucide
```

shadcn/ui is the primary design/component layer.

Specialized libraries are introduced only for interaction classes that a standard component kit should not implement itself.

## Package boundary

Reusable frontend libraries do not live inside `apps/web/lib`.

```text
packages/
├── ui/                         # Zamolxis UI system and reusable product components
└── lib/                        # shared technical wrappers/integrations
    ├── convex/
    ├── graph/
    ├── virtualization/
    ├── validation/
    └── formatting/
```

`apps/web` is a composition/application layer.

## Why this stack

### shadcn/ui

Use for almost all standard product UI:

- Sidebar
- Button / Button Group
- Card
- Badge
- Tabs
- Dialog / Alert Dialog
- Drawer / Sheet
- Dropdown / Context Menu
- Command
- Select / Combobox
- Tooltip / Hover Card
- Table
- Scroll Area
- Resizable
- Collapsible / Accordion
- Skeleton / Spinner
- Progress
- Input / Textarea / Field
- Breadcrumb
- Pagination

Do not introduce a second general-purpose design system such as MUI, Ant Design, Chakra or Mantine.

### React Flow + React Flow UI

Use for:

- Task dependency graph
- Agent hierarchy
- parallel execution
- integration branches
- workflow progress

React Flow UI keeps graph nodes visually compatible with shadcn.

The v0.1 graph is primarily an observer/navigation surface, not a generic visual workflow editor.

### TanStack Virtual

Use only for genuinely large lists:

- live Run event history
- long audit history
- large historical feeds

Do not virtualize small lists.

## No terminal dependency in v0.1

Primary Run UI renders normalized semantic activity:

```text
Read repository schema
Modified 3 files
Ran tests
41 passed / 2 failed
Investigating failures
```

If a true interactive PTY is later required, add xterm.js behind a dedicated feature/package. Do not make it foundational.

## Monorepo frontend structure

```text
apps/web/
├── app/
│   ├── (app)/
│   │   ├── layout.tsx
│   │   ├── page.tsx
│   │   ├── sessions/[sessionId]/page.tsx
│   │   ├── runs/[runId]/page.tsx
│   │   ├── approvals/page.tsx
│   │   └── settings/
│   │       ├── workstations/page.tsx
│   │       ├── repositories/page.tsx
│   │       └── runtimes/page.tsx
│   └── layout.tsx
│
├── features/
│   ├── sessions/
│   ├── tasks/
│   ├── runs/
│   ├── workflow-graph/
│   ├── approvals/
│   ├── workstations/
│   └── command-palette/
│
└── styles/

packages/ui/
├── src/
│   ├── primitives/             # shadcn-owned/generated primitives
│   ├── layout/
│   ├── status/
│   ├── sessions/
│   ├── runs/
│   ├── workflow/
│   └── activity/
└── index.ts

packages/lib/
├── convex/
├── graph/
├── virtualization/
├── validation/
└── formatting/
```

Pages compose features. Product behavior lives in features. Reusable visual components live in `packages/ui`. Technical library wrappers live in `packages/lib`.

## Dependency direction

```text
apps/web
   |
   +--> features
   |      |
   |      +--> packages/ui
   |      +--> packages/lib
   |
   '--> packages/ui

packages/ui
   +--> shadcn primitives
   +--> packages/lib/graph where required

packages/lib
   +--> third-party technical libraries
```

Avoid direct `@xyflow/react` imports throughout the application. Keep graph-specific adaptation behind the graph/UI packages.

## Product UI components

Build semantic reusable components on top of shadcn:

```text
StatusBadge
RuntimeBadge
WorkstationBadge
SessionCard
TaskCard
RunCard
RunStatusIndicator
ElapsedTime
NeedsAttentionBanner
ApprovalCard
ActivityEvent
ActivityFeed
WorkflowGraph
TaskNode
RunNode
IntegrationNode
WorkspaceStatus
```

Do not wrap every shadcn primitive just to rename it.

## Application shell

Desktop:

```text
+--------------------------------------------------------------+
| Sidebar | Main content                           | Inspector |
|         |                                        | optional  |
| Chats   | Orchestrator conversation              | Linked    |
| My Work | Session / Run detail                   | work      |
| Needs   |                                        |           |
| You     |                                        |           |
| Nodes   |                                        |           |
+--------------------------------------------------------------+
```

Use shadcn Sidebar and Resizable.

Mobile:

```text
+------------------------------+
| Header                 Menu  |
+------------------------------+
|                              |
| Primary content              |
|                              |
+------------------------------+
| Contextual actions           |
+------------------------------+
```

Sidebar becomes Sheet/Drawer. Inspector becomes Drawer.

Mobile is first-class because remote supervision from a phone is a core Zamolxis use case.

## My Work

Primary route: `/`.

```text
MyWorkPage
├── OrchestratorHeader
├── OrchestratorConversation
│   ├── MessageBlock
│   ├── RoutingDecision
│   └── LinkedWorkCard
├── NeedsYouSection
├── OrchestratorComposer
└── WorkDrawer
    ├── ActiveSessionList
    └── RecentSessions
```

The home composer says **Send**, not **Start**, and never creates a Work Session merely to store a
question. Product/repository selectors are optional context for routing. The Orchestrator can
answer from control-plane state, return typed links, continue an existing Session, create a
Session for new execution, or ask a clarifying question. The chosen route is persisted and shown
when useful.

LinkedWorkCard can target an external ticket, Work Session, Task, Agent Run, approval, trust
decision, artifact or pull request. It shows enough state to understand the answer and opens the
canonical detail view. It never copies a second mutable version of that state into the transcript.

Settings exposes **Orchestration** by role: Supervisor/Orchestrator, Builder, Verifier, Repair and
Integration. Opening a role shows its runtime, model, reasoning effort and instructions. New Runs
snapshot the effective Product/global profile and their detail shows the actual model; changing a
profile never rewrites running or historical work.

SessionCard shows:

- title
- Product
- status
- active Run count
- task progress
- needs-input state
- last activity
- secondary runtime/workstation hints

Do not make Workstation the primary information hierarchy.

## Work Session

```text
SessionPage
├── SessionHeader
│   ├── status
│   ├── goal
│   └── actions
├── SessionTabs
│   ├── Overview
│   ├── Workflow
│   ├── Activity
│   ├── Decisions
│   └── Artifacts
└── SessionInspector
```

Workflow uses the graph package.

On mobile, provide a vertical task list/timeline as the default and allow opening the graph when useful.

The Work Session page is entered from linked work or My Work. It is not the application's default
chat. Its composer continues or steers this specific work; returning to the home conversation does
not end the Session.

## Agent Run

```text
RunPage
├── RunHeader
│   ├── runtime
│   ├── status
│   ├── elapsed time
│   └── workspace
├── CurrentActivity
├── ActivityFeed
├── ChangedFilesSummary
└── RunControls
    ├── Message
    ├── Steer
    └── Stop
```

ActivityFeed consumes normalized events. It does not parse raw terminal text in React components.

## Realtime data

Use Convex reactive queries directly for hot product state.

Do not add Redux/Zustand for server state in v0.1.

Local UI state remains local:

- selected graph node
- open Drawer
- temporary filter
- unsent input
- panel size

Server state remains in Convex.

## Loading/error states

Every major feature explicitly implements:

```text
loading
empty
offline
stale/reconnecting
permission denied
not found
failed
normal
```

Do not rely on generic spinners for workstation disconnect or Run LOST states.

## UI library rule

Before adding a dependency:

1. Can shadcn/Radix solve it cleanly?
2. Is this a specialized interaction class?
3. Can the library be isolated behind `packages/ui` or `packages/lib`?
4. Is it actively maintained and compatible with React/Next.js?
5. Does it force a competing design system?

If #5 is yes, reject it unless there is a compelling isolated use case.

## Frontend milestone acceptance

The first meaningful frontend vertical slice must allow a user to:

```text
open My Work
 -> ask the Orchestrator for status without creating a Session
 -> open a linked Work Session or Agent Run
 -> see current task/run state reactively
 -> open Agent Run
 -> watch normalized live activity
 -> send a message
 -> stop the Run
 -> see disconnected/reconnecting state
```

The graph is added after this linear vertical slice works; visualization must not block core control functionality.
