# Agent workflow

The documentation agent is implemented in `src/agent/`. It takes a command (`chat`, `init`, or `update`), gathers repository context, builds prompts, runs a DeepAgents session, and records successful update metadata — but only if the documentation content actually changed.

## Main flow

`src/agent/index.ts` follows this sequence for non-chat runs:

1. Load `~/.openwiki/.env` into `process.env`.
2. For `update` runs without a user message, check whether the repository has changed since the last successful update (see [Update no-op skip](#update-no-op-skip) below). If nothing changed, skip the agent run entirely and return early.
3. Resolve the provider via `resolveConfiguredProvider()` and ensure the provider's API key exists.
4. Resolve the model ID from CLI input, `OPENWIKI_MODEL_ID`, or the provider's default model.
5. Create a run context from Git state and prior update metadata.
6. Snapshot the current `openwiki/` content hash (before the run).
7. Build the system prompt and user prompt.
8. Create the provider-specific model client (`ChatAnthropic`, `ChatOpenRouter`, or `ChatOpenAI`).
9. Create a DeepAgents `LocalShellBackend` rooted at the repository with a SQLite checkpointer.
10. Stream messages and tool events back to the CLI.
11. For `init` and `update`, compare the post-run content snapshot to the pre-run snapshot. Write `openwiki/.last-update.json` **only if the content changed**.

Chat runs skip metadata writes entirely. Update runs that are skipped via the no-op check also skip metadata writes (and the agent run itself).

## Update no-op skip

For `update` runs where no user message is provided (e.g. `openwiki --update` in CI), the agent checks `getUpdateNoopStatus()` in `src/agent/utils.ts` before resolving a provider or building prompts. The check skips the entire agent run when:

- `.last-update.json` exists with a `gitHead`,
- the current HEAD matches that `gitHead` and the working tree is clean (ignoring changes to `openwiki/.last-update.json` itself), and
- if the HEAD has advanced, the only changed paths are inside `openwiki/`.

When the check passes, the agent emits a "no repository changes detected" message and returns `{ skipped: true }` without invoking the model. This avoids wasted API calls in scheduled CI workflows.

The check is bypassed when a user message is provided (`shouldCheckUpdateNoop` returns `false`), so `openwiki --update "refresh the API docs"` always runs the agent.

## Provider-specific model creation

`createModel()` in `src/agent/index.ts` branches by provider:

- **anthropic**: `new ChatAnthropic(modelId, { apiKey, anthropicApiUrl? })` — uses `@langchain/anthropic` directly. When `ANTHROPIC_BASE_URL` is set, the resolved alternative base URL is passed as `anthropicApiUrl` so requests can be routed to a self-hosted or proxied Anthropic-compatible endpoint instead of the default API.
- **openrouter**: `new ChatOpenRouter({ apiKey, baseURL, model, siteName: "OpenWiki" })` — uses the selected OpenRouter model directly.
- **openai**: `new ChatOpenAI({ apiKey, model, useResponsesApi: true })` — uses OpenAI's Responses API for official OpenAI calls.
- **baseten / fireworks / openai-compatible**: `new ChatOpenAI({ apiKey, configuration: { baseURL? }, model })` — OpenAI-compatible clients using the provider's base URL when configured. The `openai-compatible` provider has no default endpoint; its base URL is user-supplied via `OPENAI_COMPATIBLE_BASE_URL` and required (`requiresBaseUrl: true`), which lets OpenWiki target any OpenAI-compatible gateway (for example a LiteLLM gateway fronting upstream providers).

Base URLs are resolved through `resolveProviderBaseUrl()` in `src/constants.ts`, which prefers a provider's alternative base URL environment variable (`baseUrlEnvKey`) over the built-in default before falling back to the SDK's own default endpoint. Providers marked `requiresBaseUrl` are validated at startup by `ensureProviderBaseUrl()`.

## Prompting strategy

`src/agent/prompt.ts` encodes the product rules directly into the system prompt. The agent is instructed to:

- inspect the current codebase and write documentation under `openwiki/`,
- use filesystem discovery tools and git history rather than inventing facts,
- keep the initial wiki focused and navigable,
- avoid thin/slim pages — merge stubs into broader pages rather than creating many small directories,
- document the repository for both humans and future agents,
- respect the repository root as the only project in scope,
- avoid reading secrets or `.env` files,
- use git history for init and update runs,
- respect the temporary plan file and update metadata requirements,
- ensure top-level `/AGENTS.md` and/or `/CLAUDE.md` reference the OpenWiki quickstart (inserting or refreshing a standardized section).

The user prompt changes with the command:

- `init` includes the current Git summary and asks for fresh documentation.
- `update` includes last update metadata and a Git change summary.
- `chat` just forwards the user message.

## Git evidence and update metadata

`src/agent/utils.ts` is responsible for the repository evidence that the prompt sees:

- current working tree status,
- current HEAD,
- a change window since the last successful update when `.last-update.json` includes a `gitHead` or `updatedAt`,
- the most recent 20 commits with changed files for init runs (or updates without prior metadata),
- a diff summary against HEAD.

On successful init/update runs where content changed, the agent writes JSON metadata with:

- `updatedAt`
- `command`
- `gitHead`
- `model`

That metadata is later used to scope update runs.

### Content snapshot

`createOpenWikiContentSnapshot()` computes a SHA-256 hash of the entire `openwiki/` directory tree (excluding `.last-update.json`). The agent runtime takes a snapshot before and after the run. If they match — meaning the model made no documentation changes — the metadata file is not updated. This prevents scheduled update loops from churning the metadata when the wiki is already current.

## Model errors

The agent runtime uses only the selected provider and model for a run. If that
request fails, OpenWiki surfaces the provider error and stops instead of
retrying with another model.

## Why this matters

The agent is not just a generic chat wrapper. It is intentionally constrained so it can:

- write repository-local docs without wandering outside the repo,
- preserve continuity across runs via checkpointing and metadata,
- keep updates grounded in Git evidence,
- avoid metadata churn via the content-snapshot check,
- support both interactive and scheduled maintenance use cases.

## Things to watch when changing agent behavior

- Keep the prompt in sync with the actual filesystem tools and path conventions used by the CLI.
- Be careful with `.last-update.json` semantics, because update runs use it to decide what changed since the previous successful run.
- The content-snapshot check means a no-op update will not update metadata. If you change the snapshot logic, ensure `.last-update.json` is still excluded.
- The update no-op skip (see above) is a separate mechanism from the content-snapshot check. The no-op check runs *before* the agent and avoids the API call entirely; the content snapshot runs *after* the agent and only controls metadata writes.
- Credential loading happens before model resolution; changes there affect both onboarding and agent startup.
- When adding a provider, add a branch in `createModel()` and ensure the API key env key is checked in `ensureProviderKey()`.
- The DeepAgents backend is configured with `virtualMode: true`, which is important for documentation-only behavior.

## Source map

- `src/agent/index.ts`
- `src/agent/prompt.ts`
- `src/agent/utils.ts`
- `src/agent/types.ts`
- `src/constants.ts`
- `src/env.ts`
- Git evidence: commits `ceded10`, `f89b05d`, `dfa73cc`, `a82759f`, `0fa1430`
