# Runtime Adapter Architecture

## Objective

Zamolxis is independent from any single coding-agent vendor.

Every supported runtime is integrated through an adapter that translates native runtime behavior into Zamolxis concepts.

## Conceptual interface

```ts
interface AgentRuntimeAdapter {
  detect(): Promise<RuntimeDetection>;
  capabilities(): Promise<RuntimeCapabilities>;

  listSessions(): Promise<NativeSession[]>;
  inspectSession(nativeSessionId: string): Promise<NativeSessionSnapshot>;

  start(input: StartRunInput): Promise<StartedRun>;
  resume(input: ResumeRunInput): Promise<StartedRun>;
  send(input: SendInput): Promise<void>;
  stop(input: StopRunInput): Promise<void>;

  subscribe(input: SubscribeInput): AsyncIterable<RuntimeEvent>;
}
```

The concrete TypeScript API may evolve; the architectural boundary should not.

## Capability negotiation

Different runtimes expose different levels of observability and control. An adapter advertises capabilities rather than Zamolxis assuming feature parity.

Potential capabilities include:

- native session discovery
- session resume
- structured messages
- tool-call events
- command execution events
- changed-file events
- usage metrics
- child-agent hierarchy
- live transcript
- steering
- pause/resume
- stop
- native approval requests

The UI progressively enhances based on these capabilities.

## Runtime selection

Execution policy has three modes.

### Auto

Zamolxis chooses a runtime based on task requirements, availability, policy and capabilities.

### Preferred

A runtime is preferred but Zamolxis may fall back when unavailable or unsuitable.

### Forced

The selected runtime is mandatory.

## Runtime vs model

A runtime is an execution environment. A model is an inference choice. They are separate dimensions.

A runtime may support multiple models, while another runtime may manage model selection internally.

## Initial integrations

The first practical adapters can target Codex, Claude Code and Hermes. They are integrations, not architectural dependencies.

Future adapters can be introduced without changing the Work Session model, Supervisor or primary UI.
