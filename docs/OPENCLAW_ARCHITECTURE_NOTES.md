# OpenClaw Architecture Notes

This note maps the concrete runtime path for OpenClaw agent execution, with file-level pointers for the core call chain.

## 1. Startup entry points

### Process launcher

- `openclaw` CLI resolves to `openclaw.mjs` via `package.json` bin mapping.
- `openclaw.mjs` performs runtime checks and then forwards into TS entry.

### TypeScript main entry

- `src/index.ts` is the executable TS entrypoint.
- `runLegacyCliEntry()` lazy-imports `./cli/run-main.js` and invokes `runCli(argv)`.
- Main-mode installs global uncaught/unhandled handlers, then starts CLI routing.

Primary refs:

- `package.json`
- `src/index.ts`
- `src/cli/run-main.ts`

## 2. Config loading path

The config system is centered in `src/config/io.ts`, surfaced through `src/config/config.ts`.

### Call chain

- Consumers call `loadConfig()` / `getRuntimeConfig()` from config facade exports.
- IO layer resolves config path from env/state dir.
- Optional dotenv hydration is applied for process env usage.
- JSON5 is parsed.
- Include files are resolved.
- Env var substitution and plugin-aware validation are applied.
- Runtime snapshot/cache is produced for consumers.

Primary refs:

- `src/config/config.ts`
- `src/config/io.ts`

## 3. Agent Runtime location

The core runtime is the embedded PI runner stack.

### Key modules

- Public runner surface: `src/agents/pi-embedded.ts`
- Core orchestration loop: `src/agents/pi-embedded-runner/run.ts`
- Backend adapter: `src/agents/pi-embedded-runner/run/backend.ts`
- Harness execution handoff: `src/agents/harness/selection.ts`

### Runtime role split

- `run.ts` orchestrates attempts, retries, failover, tool integration, session/runtime state.
- `run/backend.ts` is intentionally thin and delegates to harness execution.
- Harness layer isolates provider/model-specific execution behavior.

## 4. Skill loading mechanism

Skill loading for agent runs is organized under `src/agents/skills.ts` and `src/agents/skills/workspace.ts`.

### Call chain

- Agent command resolves runtime config and execution context.
- Skill snapshot functions are called:
  - `buildWorkspaceSkillSnapshot`
  - `loadWorkspaceSkillEntries`
  - `resolveSkillsPromptForRun`
- Result feeds prompt/context assembly for the current run.

Primary refs:

- `src/agents/skills.ts`
- `src/agents/skills/workspace.ts`
- `src/agents/agent-command.ts`

## 5. Tool registration and execution

OpenClaw tool wiring is centralized in `src/agents/pi-tools.ts`.

### Registration/build path

- `createOpenClawCodingTools()` composes the runnable tool set.
- Composition includes:
  - base coding tools,
  - shell tools (`exec`, `process`) via lazy loaders,
  - channel tools,
  - OpenClaw-owned tools,
  - plugin-provided tools.
- Tool policy, provider/model policy, and wrappers are applied before exposure.

### Execution contract

- Runtime executes tools via `AnyAgentTool.execute(toolCallId, args, signal, onUpdate)`.
- Lazy tools (`exec`, `process`) forward execution to deferred-loaded implementations.

Primary refs:

- `src/agents/pi-tools.ts`

## 6. Plugin mechanism location

Plugin load + runtime integration is in `src/plugins/loader.ts` plus runtime context builders in `src/plugins/runtime`.

### Call chain (high-level)

- Build/resolve plugin runtime load context (`resolvePluginRuntimeLoadContext`).
- Discover plugin candidates (`discoverOpenClawPlugins`).
- Build plugin API surface (`buildPluginApi`).
- Load modules and populate plugin registry/runtime state.

Primary refs:

- `src/plugins/loader.ts`
- `src/plugins/runtime/load-context.ts`
- `src/plugins/types.ts`

## 7. Model invocation encapsulation

Model calls are abstracted through harness execution.

### Call chain

- Runner calls `runEmbeddedAttemptWithBackend(params)`.
- Backend delegates to `runAgentHarnessAttempt(params)`.
- Harness resolves and performs provider/model-specific request execution.

Primary refs:

- `src/agents/pi-embedded-runner/run/backend.ts`
- `src/agents/harness/selection.ts`

## 8. End-to-end user message call chain

A representative CLI message path:

1. `openclaw.mjs`
2. `src/index.ts::runLegacyCliEntry()`
3. `src/cli/run-main.ts::runCli(...)`
4. `src/agents/agent-command.ts` (session/agent/model/skill prep)
5. `src/agents/command/attempt-execution.ts::runAgentAttempt`
6. `src/agents/pi-embedded-runner/run.ts::runEmbeddedPiAgent`
7. `src/agents/pi-embedded-runner/run/backend.ts::runEmbeddedAttemptWithBackend`
8. `src/agents/harness/selection.ts::runAgentHarnessAttempt`
9. assistant/tool outputs return to attempt-execution and transcript persistence/output pipeline

## 9. Tool-call return flow: parse, validate, execute, return

When the model emits tool calls, OpenClaw applies normalization and repair before dispatch.

### 9.1 Parse + normalize

- Tool call block normalization and tool name canonicalization happen in:
  - `src/agents/pi-embedded-runner/run/attempt.tool-call-normalization.ts`
- Handles block-type compatibility (`toolCall`/`toolUse`/`functionCall`) and allowlist-sensitive name normalization.

### 9.2 Argument repair/validation pre-dispatch

- Streaming argument repair is handled in:
  - `src/agents/pi-embedded-runner/run/attempt.tool-call-argument-repair.ts`
- Logic repairs malformed partial JSON argument streams under bounded safety rules.

### 9.3 Execute

- Normalized tool calls are dispatched against the tool set built from `createOpenClawCodingTools()`.

### 9.4 Return to model loop

- Tool results are fed back into the session/conversation state.
- Runner continues the attempt loop with updated transcript/context for next model step.

## 10. Product-general modules vs minimum agent skeleton

### Product-general surfaces (optional for a minimal agent)

- Multi-channel integration and channel plugins (`src/channels/**`, `extensions/**`)
- Gateway server/control planes (`src/gateway/**`)
- Broad plugin ecosystem surfaces (catalogs/setup/runtime extras)
- Rich docs/ops/testing/QA infrastructure

### Minimum agent skeleton required

1. Config read/runtime snapshot:
   - `src/config/config.ts`, `src/config/io.ts`
2. Agent command orchestration:
   - `src/agents/agent-command.ts`
   - `src/agents/command/attempt-execution.ts`
3. Runtime loop:
   - `src/agents/pi-embedded-runner/run.ts`
4. Model backend abstraction:
   - `src/agents/pi-embedded-runner/run/backend.ts`
   - `src/agents/harness/selection.ts`
5. Tool substrate:
   - `src/agents/pi-tools.ts`
   - tool-call normalization/repair helpers

## Minimal viable skeleton dependency graph

```text
User input
  -> CLI ingress (openclaw.mjs -> src/index.ts -> src/cli/run-main.ts)
    -> Agent command orchestrator (src/agents/agent-command.ts)
      -> Attempt execution (src/agents/command/attempt-execution.ts)
        -> Embedded runtime loop (src/agents/pi-embedded-runner/run.ts)
          -> Backend adapter (run/backend.ts)
            -> Harness model execution (src/agents/harness/selection.ts)
          -> Toolset build/dispatch (src/agents/pi-tools.ts)
          -> Toolcall normalize/repair
             (attempt.tool-call-normalization.ts,
              attempt.tool-call-argument-repair.ts)
        <- assistant output/tool results
      -> transcript persist + reply output
```
