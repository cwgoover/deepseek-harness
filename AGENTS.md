# Repository Guidelines

DeepSeek Harness is an all-plugin [Cordis](docs/cordis-primer.md) agent harness (TypeScript, ESM, pnpm workspaces). Nothing runs "in the loop": every capability is a plugin mounted into a scoped context. Read [docs/architecture.md](docs/architecture.md) before changing `packages/`.

## Project Overview

A modular agent runtime. Profiles (`headless`, `web`, `sdk`, `sdk-minimal`, `acp`) load ordered Cordis bundle patches to compose a service tree, then drive an agent loop: model requests, tool execution, and a durable append-only session log. Both a TypeScript SDK and Python SDK project the same loop.

## Architecture & Data Flow

- **Cordis substrate.** A plugin is a function or `Service` subclass claiming a stable context key (`ctx.sessions`, `ctx.agents`, `ctx.llm`, `ctx.tools`). `static inject` expresses readiness/dependency ordering — no manual boot sequencing.
- **Registration is an effect.** Listeners, adapters, tools, prompt sections, and projections register via `ctx.effect()` / `ctx.on()`; they unwind on unload. A registry's `register()` returns the disposer.
- **Capability seam** = Service Definition + Provider + Consumer. Example: `LlmRuntime` (`ctx.llm`) owns the definition/registry; `LlmAdapter` is the provider seam; agent-loop and UI consume `ctx.llm` without importing provider implementations. Never one role only ([glossary](docs/glossary.md#capability-seam)).
- **Turn flow** (`packages/core/agent-loop/src/agent.ts`, `ReactLoopAgent`):
  1. Message enters inbox (`send`/`followup`/`steer`/`inject`); wake starts the driver. Cancellation = `AbortController` + typed `AgentCancelCause`.
  2. `turn()` appends `turn/start`, assembles `ctx.systemPrompt`, runs the `agent/pre-step` waterfall (a listener may rewrite/reject input).
  3. Per step: append `step/start`, `prepareRequest()` runs the `agent/request` waterfall, `ctx.llm.prepareCall()` **resolves the exact adapter/model route before** committing model-visible messages.
  4. Loop appends `system/message`, `user/message`, request header/context events, deep-freezes the request derived from session state, streams via the bound prepared call.
  5. Live `agent/assistant-stream` frames (start/chunk/end) are process-local; the settled result commits atomically as durable `assistant/message` (or `assistant/attempt` on failure/retry). Tool-call blocks go through `tools/pre-execute` → `tools/execute` → `tools/post-execute`, appending `tool/result` and queuing the next step.
  6. `step/end`; `agent/turn-stopping` (serial); `turn/end` carries merge-extensible `TurnEndReason` (`completed`/`aborted`/`blocked`/`error`/`max-tokens`/`interrupted`).
- **Session is the source of truth.** `SessionEventMap` (`packages/core/session/src/types.ts`) is the durable contract; model history is reconstructed from the append-only log via `deriveMessages()`. `SessionStore.append()` validates data/sequence/JSON-safety, freezes payloads, publishes `session/event`. Persistence is a *separate* plugin subscribing to those events — the core `session` package has no persistence. **Model-visible ⟺ logged**: any model-visible input requires a session event.

## Key Directories

Packages live at `packages/<group>/<pkg>/`, each `@deepseek-ai/dsh-<name>`.

- `packages/core/` — `agent` (Agent contract, `AgentRegistry`=`ctx.agents`), `agent-loop` (`ctx.agentLoop`, `ReactLoopAgent`), `session` (append-only log), `tools` (`ctx.tools`, guarded pipeline), `system-prompt` (`PromptAssembly`).
- `packages/llm/` — provider-neutral messages + `LlmRuntime`/`LlmAdapter`; `registerAdapter()` validates routes atomically, returns disposer with `.replace()`.
- `packages/session/` — JSONL persistence, telemetry, query/export, projections — all folding durable events.
- `packages/subagent/`, `packages/preset/` — subagent seam + per-session composition roots (`agent-preset/selected` stored in session metadata).
- `packages/bundle/` — patchable Cordis row sets: `base` (shared infra), `headless`, `web-app`, `sdk-app`, ACP layers.
- `packages/boot/app-boot/` — profile resolution + Loader glue (shared by CLI and packaged runtimes).
- `apps/cli/` — the `dsh` launcher (`@deepseek-ai/dsh`). `vendor/` — pinned Cordis source. `python/` — Python SDK/runtime. `native/system/` — node addon. `scripts/` — gates/generators. `docs/`, `website/`, `.agents/`, `benchmarks/`, `snapshots/`.

Group map: [packages/README.md](packages/README.md).

## Development Commands

```sh
pnpm install                                # Node ^22.19 || >=24, pnpm@11.7.0
pnpm run build                              # tsx scripts/build.ts → native-system, lib (tsc→tsdown), web
pnpm run typecheck                          # builds host contracts, then tsc -b tsconfig.client.json
pnpm run lint                               # oxlint (build:lib:host first)
pnpm run duplication                        # jscpd across packages/ scripts/
pnpm run hygiene                            # publint + workspace/package/dependency + NodeNext checks
pnpm run doc-sync                           # documentation gates (scripts/run-gates.ts)
pnpm run check:all                          # full local gate graph
pnpm dsh --profile headless "task"          # run one task from source (needs DEEPSEEK_API_KEY)
pnpm dsh web                                # web profile; add --patch <overlay> for source patches
pnpm run demo:ptc -- "task"                 # headless PTC run
```

Build pipeline: `tsc -b` emits declarations + JS under `lib/types`; `tsdown` consumes that JS and writes ESM runtime bundles under `lib` (host and client faces built separately, `--env.DSH_BUILD_FACE`).

## Code Conventions & Common Patterns

- **ESM everywhere** (`"type": "module"`). Cross-package imports use package names; local relative imports keep `.ts`; built ESM uses `.js`. Source-launch runs through tsx's ESM hook (`node --import tsx/esm`) — reached modules must stay ESM (no CJS-only exports).
- **Typed events via declaration merging.** Augment `@deepseek-ai/cordis` `Context`/`Events`; merge-extensible maps (`SessionEventMap`, `TurnEndReasonMap`, `ContentBlockMap`, `FinishReasonMap`). Event JSDoc needs `@mode` + `@param`. Only structural format changes bump `SESSION_FORMAT_VERSION`.
- **Waterfall listeners MUST call `next()`** to delegate; returning short-circuits ([semantics](docs/cordis-primer.md#cordis-waterfall-semantics)).
- **Switch on discriminant tags.** Closed unions end in `assertNever`; merge-extensible unions fall through a documented `default`.
- **Resolve then run.** Defaulting is an explicit `resolve(request): Spec` step in the owning implementation, never a hidden `?? default` inside `run()` (see `LlmRuntime.prepareCall()` / the `dsh-shell` request/spec split).
- **Branded ids.** Opaque cross-boundary ids use `Branded<B>` from `dsh-brand` (`SessionId`, `SessionSeq`, `SessionLogOffset`), never bare `string`.
- **Trust TypeScript at typed same-process boundaries.** Validate only at parser/config, model/tool JSON, durable/file, worker, process, and wire boundaries.
- **Error handling.** Empty `catch` must name the error and keep its `try` to one statement. Agent-loop preserves `LlmError.failure`; unknown failures normalize to `{ message: errorChain(error), code: 'UNKNOWN' }`; adapters never place credentials in diagnostics. Misconfiguration fails loud at load (or earliest resolvable point); never silently skip a missing referent.
- **No hardcoded tunables in plugins**: deployment-varying choices are validated `Config` fields settable from `cordis.yml`. Protocol/security constants stay fixed.
- **Extension, not loop edits.** New behavior = new plugin/tool/provider. Add a provider → register on the service (`ctx.llm`, `ctx.fs`, `ctx.sandbox`); model capability → scoped tool; durable state → merge into `SessionEventMap`; scope to one agent → register through `agent.ctx`. Changing `agent-loop` requires updating docs/architecture.md.
- **Client UI copy is locale-owned** (typed dictionaries + `t`); `verify-client-ui-i18n` rejects hardcoded copy.
- **Comments/docs state contracts, not reasoning transcripts.** Concise, concrete; use exact terms over "shape"/"boundary". Files end with exactly one trailing newline (`git diff --cached --check` gates it). Markers: `FIXME`/`TODO`/`XXX` by urgency.
- **Agent Notes** (`.agents/notes/`) only for durable decision rationale; archived notes are frozen (never edit). PR labels: one `kind/*` (`feature`/`bug-fix`/`doc`/`testing`/`cleanup`/`dependency`), all material `area/*`, native Issue Type.

## Important Files

- `apps/cli/src/bin.ts` — `dsh` dispatch (`profile`/`plugin`/`dump-config`).
- `apps/cli/src/profile-boot.ts` — stacks bundle → profile → home → `--patch` layers, mounts Loader, gates readiness, bounded shutdown.
- `packages/boot/app-boot/src/index.ts` — shared profile/Loader boot; loads `.env` via `process.loadEnvFile`.
- `packages/bundle/base/cordis.patch.yml` — shared model/session/agent/tool/persistence rows; other bundles patch by row `id`.
- `packages/core/agent-loop/src/agent.ts` — `ReactLoopAgent` driver. `packages/core/session/src/types.ts` — `SessionEventMap`.
- Config: `package.json`, `pnpm-workspace.yaml`, `.npmrc`, `tsconfig.json` (solution) + `tsconfig.{host,client}.json` + `tsconfig.base.json`, `tsdown.config.ts`, `.oxlintrc.json`, `.jscpd.json`, `scripts/run-gates.ts`.

`cordis.yml` entries have `id`/`name`/`config`/optional `disabled`; later layers replace a row's whole config by id (last write wins). `!!js` (never `!js`) is allowed under plugin `config` and entry `disabled`, e.g. `!!js Number(process.env.PORT ?? 3081)`.

## Runtime/Tooling Preferences

- **Node** `^22.19.0 || >=24.0.0` (Node 23 is out of range); **pnpm@11.7.0** (`packageManager` pin). Workspaces: `vendor/*`, `packages/*/*`, `native/system`, `apps/*`, `website`, `python/sdk-runtime`.
- TypeScript solution config is program-less (`files: []`); host/client faces have separate reference graphs so host and browser Cordis contexts never merge. Base: `target es2024`, `module esnext`, `moduleResolution bundler`, `strict`, `allowImportingTsExtensions`. **Source plane vs artifact plane**: gates/tests resolve workspace imports through tsconfig `paths` → `src`; gates consuming built `lib/` declare that dependency.
- `.env` optional; credential order: process env → managed credentials → cwd `.env` → `$DSH_HOME/.env`. Never commit credentials. Only `dsh` profiles launch supported Node apps (no package-bin/demo/SDK argv escapes).

## Testing & QA

- **Vitest** is the primary runner. Unit specs live beside owners as `packages/*/*/tests/**/*.spec.ts` (also `apps/*/tests`, `scripts/**/*.spec.ts`, `website/tests/**/*.spec.ts`). Live tests are `*.e2e.ts`; recorded-session drivers `*.snapshot.ts`; owner-local process expectations `*.expected.e2e.ts`. Python SDK uses **pytest** (`python/sdk/tests/test_*.py`). Forked workers — tests own ports (`listen(0)`), temp dirs, subprocesses, and teardown.

```sh
pnpm run test           # build:native-system && vitest run (unit)
pnpm run test:coverage  # CI coverage gate: per-file 100% on packages/*/*/src
pnpm run test:e2e       # real-API; self-skips without DEEPSEEK_API_KEY
pnpm run test:expected  # owner-local assembled CLI/process expectations
pnpm run test:snapshot  # keyless recorded-session replay through shipped profiles
pnpm run test:snapshot:record   # re-record (needs key)
pnpm run test:docs      # quick doc-quick gate aggregate (no build)
# filter a scenario:
pnpm run test:snapshot -- snapshots/sdk/sdk.snapshot.ts -t text-turn
```

- **Coverage gate is `test:coverage`, not `test`** — per-file 100% on `packages/*/*/src`; treat an uncovered line as a dead-code candidate before adding a test. Line coverage is necessary, not sufficient: prove shipped behavior.
- **Snapshot required for every non-trivial model-, protocol-, or human-visible change** in the same PR; fixtures replay identically on macOS/Linux — fix fixtures/metadata, not normalizers. Top-level `snapshots/` is session-driven (`session/`, `sdk/`, `acp/`, `web/`); keep other expected output owner-local. Workspace side effects need a `workspace.expected/` oracle.
- **Both SDKs project the loop.** Agent-loop / session-lifecycle / `SessionEventMap` changes update TypeScript **and** Python SDK expected outputs in the same PR (`scripts/snapshots/python-sdk-single-exe/`); `pnpm run test` covers neither.
- **Tests describe behavior, not implementation.** Prefer real registry/pipeline/persistence/Loader composition; mock only LLM/network/clock/nondeterministic boundaries. Change obsolete behavior with its tests and explain why. See [docs/testing.md](docs/testing.md).
