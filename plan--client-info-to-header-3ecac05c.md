---
type: plan
slug: client-info-to-header-3ecac05c
---

# Plan: Move `clientInfo` from JSON body to HTTP header

## Context

Commit `e6acca1` (foster.han, 2026-04-28) introduced a `clientInfo: { kind, version }` field in the push payload as part of the JOLLI-1335 push contract, alongside a deprecated `pluginVersion` body field kept "for one release as a forward-compat bridge."

Decision: `clientInfo` is transport metadata about *who* is calling, not part of the business payload. Move it to an HTTP header so:

- It applies uniformly to all Jolli API calls (push, bindings, LLM proxy) without each handler re-parsing the body
- Server middleware can log/route/version-gate without touching the body
- Future endpoints inherit it automatically
- `JolliPushPayload` and other DTOs stay focused on domain data

## Decisions

| # | Decision |
|---|----------|
| 1 | Header name: **`x-jolli-client`** (lowercase kebab-case, matching existing `x-tenant-slug` / `x-org-slug`) |
| 2 | Format: `<kind>/<version>` — `vscode-plugin/1.2.3`, `intellij-plugin/0.9.0`, `cli/1.0.0` |
| 3 | `pluginVersion` body field is **deleted entirely**, no compat bridge (`e6acca1` shipped same day, no external consumers) |
| 4 | CLI also sends `x-jolli-client` (currently sends nothing — fills the gap) |
| 5 | Server-side coordination handled separately by backend (out of this repo) |

## Changes

### vscode

**Design note — no parameterization**: `vscode/src/services/*.ts` are vscode-specific files (cli has its own `LlmClient.ts`, intellij has its own `JolliApiClient.kt`). Threading a `clientInfo` argument through every public API has no consumer to justify it, so each surface's service module reads its own identity directly from the local `package.json` and stamps the header internally.

**`vscode/src/services/ClientInfo.ts`** (new)
- Owns the `ClientInfo` interface (kind + version)
- Exports `VSCODE_CLIENT_INFO` — the singleton built from this plugin's `package.json` version

**`vscode/src/services/JolliPushService.ts`**
- Imports `VSCODE_CLIENT_INFO` and uses it inside `buildJolliApiHeaders` to emit `x-jolli-client: vscode-plugin/<version>`
- `buildJolliApiHeaders` signature unchanged from the original (no new params)
- `pushToJolli` / `deleteFromJolli` signatures unchanged from the original
- `JolliPushPayload`: remove `clientInfo` and `pluginVersion` fields (these are header-only now)
- Top-of-file doc-comment updated to reflect header location
- `ClientInfo` type re-exported for back-compat (consumers can import from either file)

**`vscode/src/services/JolliMemoryApiService.ts`**
- No signature changes — `buildJolliApiHeaders` injects the header internally

**`vscode/src/views/SummaryWebviewPanel.ts`**
- Three push call sites (lines 847, 1094, 1165) stop putting `clientInfo` / `pluginVersion` in the payload — payload only carries domain data
- No new args passed to `pushToJolli` / `deleteFromJolli`
- Top-of-file `pluginVersion` import (used only for the deleted body field) is removed

**`vscode/src/views/BindingChooserWebviewPanel.ts`**
- No changes — it just calls `JolliMemoryApiService.*` which now stamps the header automatically

### cli

**`cli/src/core/LlmClient.ts`**
- In `callProxy` (lines 210–217), add `"x-jolli-client": \`cli/${version}\`` to the fetch headers
- Read version from `cli/package.json` directly inside the module — no parameterization through `callLlm`/`callProxy` signatures

**`cli/src/core/LlmClient.test.ts`**
- Add assertion that the `x-jolli-client` header is sent with proxy-mode requests

### intellij

**`intellij/src/main/kotlin/ai/jolli/jollimemory/services/JolliApiClient.kt`**
- Remove `val pluginVersion: String? = null` from request DTO (line 39)
- Read plugin version once inside the API client (e.g. via PluginManagerCore in init) and stamp `x-jolli-client: intellij-plugin/${pluginVersion}` on every request

**`intellij/src/main/kotlin/ai/jolli/jollimemory/toolwindow/SummaryPanel.kt`**
- Stop reading `pluginVersion` and passing it through the call chain — the API client owns it now (lines 270/287/307)

**`intellij/src/test/kotlin/ai/jolli/jollimemory/services/JolliApiClientTest.kt`**
- Replace `payload.pluginVersion shouldBe null` (line 169) with header assertion

## Verification

- `npm run typecheck` (cli + vscode)
- `npm run lint` (biome `--error-on-warnings`)
- `npm run test` (vitest, cli enforces 97% coverage)
- `cd intellij && ./gradlew test`

## Out of scope (for separate backend PR)

- `PushRouter.ts` zod schema: drop `pluginVersion` field, read header instead
- `MIN_PLUGIN_VERSION` gate moves from body parse to header parse
- `PushRouter.test.ts` updates accordingly