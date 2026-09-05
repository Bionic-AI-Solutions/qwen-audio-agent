---
project_name: 'qwen-audio-agent'
user_name: 'Dude'
date: '2026-09-05'
sections_completed:
  [
    'technology_stack',
    'language_rules',
    'framework_rules',
    'api_compatibility',
    'persistence_rules',
    'security_rules',
    'testing_rules',
    'workflow_rules',
  ]
existing_patterns_found: 24
source: 'Derived from the SKAD document-project deep scan — see docs/skad/index.md'
---

# Project Context for AI Agents

Critical rules for implementing code in **qwen-audio-agent** v1.11.0. Focused on **unobvious**
details that cause real breakage. Background and full detail: [`docs/skad/`](../docs/skad/index.md).

---

## Technology Stack & Versions

| Concern | Choice | Constraint |
| --- | --- | --- |
| Language | **JavaScript ESM only** — no TypeScript | Sources are `.mjs`; React files are `.jsx`; `.cjs` only where a runtime demands it |
| Node | `^22.22.2 \|\| ^24.15.0 \|\| >=26.0.0` | `.nvmrc` pins 22.22.2; CI tests 22.22.2 **and** 24.15.0 — don't use APIs missing from 22 |
| npm | `>=10`, `packageManager npm@10.9.4` | Use `npm ci`, never `npm install`, in scripts and CI |
| Monorepo | npm workspaces — no Turbo/Nx/Lerna | 6 workspaces: server, web, tui, desktop, cli, mobile |
| Server | Express ^4.22.2, `ws` ^8.21.1 | Express **4**, not 5 |
| Agent protocol | `@agentclientprotocol/sdk` 1.3.0, `@a2a-js/sdk` | Pinned exactly |
| Tools | `@modelcontextprotocol/sdk` 1.30.0 | |
| Validation | `zod` ^4.4.3 | zod **4** API |
| Web | React ^19.1.0, Vite ^7.0.0 | |
| Desktop | Electron ^43.2.0, sherpa-onnx 1.13.4, electron-builder 26.15.7 | |
| Mobile | Capacitor 8.5.1 | |
| Lint | ESLint 10.8.0 flat config | `eslint.config.mjs` |
| Test | **`node --test` built-in** | No Jest, Vitest, or Mocha — never add one |

**No build step for server code.** `npm run build` builds only `web/`. Edit `.mjs`, restart, done.

**Do not add a dependency to `cli/`.** It is deliberately zero-dependency so the published
`qwenaudio` binary installs anywhere. Put shared logic in `shared/` instead.

---

## Language & Style Rules

| Rule | Detail |
| --- | --- |
| No semicolons | |
| Single quotes | |
| 2-space indent, trailing commas in multiline literals | |
| Unused bindings | Prefix with `_` — `argsIgnorePattern: '^_'`, `varsIgnorePattern: '^_'` |
| Empty `catch {}` is allowed and idiomatic | `caughtErrors: 'none'`. Diagnostics must never break startup — wrap logging/telemetry in try/catch |
| Frozen constants | `Object.freeze()` on exported constant tables and policies (~50 uses). Follow it |
| Private fields | `#name` class fields, e.g. `TaskRepository.#serializedRecords` |
| Comments explain **why**, not what | Mixed English and Chinese, sometimes in one file. Match the surrounding file |
| No `console.log` in `server/src/` | Use `logger` from `core/logger.mjs`; `runWithLogContext` for request scope |

### Errors carry a `code`

Two conventions coexist — match the surrounding module:

- `SCREAMING_SNAKE` for host/lifecycle failures: `QWAUDIO_GATEWAY_CONFIG_DIR_REQUIRED`,
  `QWAUDIO_GATEWAY_SETUP_REQUIRED`, `QWAUDIO_INPUT_OWNER_REQUIRED`.
- `snake_case` for protocol/domain failures: `gateway_invitation_expired`, `document_too_large`,
  `client_action_unsupported`, `invalid_edit`, `knowledge_owner_required`.

`QWAUDIO_*` names are **error codes, not environment variables** (except `QWAUDIO_CONFIG_DIR` and
`QWAUDIO_DATA_DIR`, which are real). Don't put a code in a config file.

---

## The Three API-Compatibility Traps

None of these fail a test. All three are real breakage.

### 1. `package.json#exports` is the public API — 42 entries

They reach into **every** workspace, not just `shared/`. `server/src/app/gateway-application.mjs`,
`server/src/session/session-journal.mjs`, and `desktop/src/settings-store.mjs` are all published.

> **Before renaming, moving, or changing the signature of any module, grep `package.json#exports`.**
> A path that looks internal may be public. Anything *not* in the map is genuinely internal.

### 2. `shared/` has no tests of its own

It is covered only transitively by consumers, and third-party consumers are not covered at all.
A `shared/` change is automatically a public API change.

### 3. Persisted-shape changes require a `fileVersion` bump

`JsonSnapshotStore` / `VersionedJsonStore` route a version mismatch to quarantine. **Without the
bump, an old file is parsed as the new format, every record fails validation, and the store silently
becomes empty — data loss with no error.**

---

## Architecture Invariants

### The frontend/agent boundary (the core rule)

> **Preserve the boundary between the Realtime frontend and the background Agent**
> (`docs/architecture/deep-dive.md`).

Voice (`voice/`, `frontend/`, `conversation/`) must **never block** on background work
(`agent/`, `backend/`, `process/`, `task/`). Progress is reported as events, never as a blocking
call. Code that makes the voice path `await` backend completion violates this, even when it looks
simpler.

### Capability-based versioning, never version checks

`GATEWAY_PROTOCOL_VERSION` 5.7.0 · GCP6 `6.0.0` · **29 frozen `GATEWAY_CAPABILITIES`**.

- Clients branch on a **capability string**, never a product version.
- Minor bump = additive capability. Major bump = breaking change to anything in the list.
- Every capability is **locked by a test** — adding one means adding its test.
- Anything unlisted is internal and may change freely.
- Never allowlist a `clientType`: it is descriptive metadata, behavior is negotiated by capability.

### `web/` is not a leaf

Desktop loads `web/dist` via `desktopOrbUrl()`; mobile does
`await import('../../web/src/App.jsx')` at runtime. **A change to `web/src/App.jsx` ships to three
clients.** `.github/workflows/mobile.yml` triggering on `web/**` is correct.

### Middleware order is a security boundary

In `server/src/app/gateway-application.mjs`:

```
GET  /connect             line 543   ← pre-auth (HTML pairing shell)
POST /api/access/pair     line 554   ← pre-auth (origin-allowlisted)
─────────────────────────────────
app.use(enforceSameOrigin) line 598  ← everything BELOW is authenticated
```

> **A route added above line 598 is public.** Add new routes after the auth middleware.

Note also: `GET /connect` is **not** the WebSocket. The socket is `WS /api/realtime?sessionId=<id>`;
`rejectUnsupportedRealtimeUpgrade()` destroys an upgrade on any other path.

---

## Framework Rules

### React (`web/src/**`, `mobile/src/**`)

- `react-hooks/rules-of-hooks` and **`react-hooks/exhaustive-deps` are `error`**, not warnings.
- **No state library.** State lives in three tiers, and picking the wrong one is the common mistake:

| Tier | Where | Use for |
| --- | --- | --- |
| Protocol state | `shared/gateway-client-state.mjs` — a **pure reducer** | Anything derived from Gateway server events |
| Session state | `useState` in `App.jsx` (26 hooks) | UI/session values |
| Imperative state | `useRef` in `useRealtimeVoice.js` (~20 refs) | Values that must not re-render: socket, AudioContext, callbacks, playback ledger |

- Callbacks are mirrored into refs (`eventRef.current = onEvent`) rather than added to dependency
  arrays. Follow that instead of suppressing `exhaustive-deps`.
- **Do not touch `playbackRef`** from a component or convert it to `useState`. It tracks per-response
  audio scheduling with `Set`/`Map` members that keep playback start/end idempotent under
  out-of-order events.

> **New logic goes into a pure module in `web/src/` with a matching `web/test/*.test.mjs` — not into
> a new branch in `App.jsx` or another `useEffect`.** This 1:1 convention is why 19 suites cover the
> web client with no DOM harness.

### Mobile import firewall (lint-enforced)

`mobile/src/**` may import `web/` and `shared/`, but **never `server/` or `desktop/`**
(`no-restricted-imports`). Mobile stays on the public client contract.

### Electron (`desktop/`)

- Every `BrowserWindow`: `contextIsolation: true`, `nodeIntegration: false`, `sandbox: true`.
- **`backgroundThrottling: false` on orb windows is load-bearing**, not a perf tweak. The orb is
  usually unfocused; throttled timers delay Web Audio playback receipts, and the Gateway treats a
  missing receipt as failed playback and retries.
- `preload.cjs` is the **entire** main↔renderer contract (33 members). New subscribers must:
  (1) return a no-op disposer for a non-function callback, (2) wrap the listener so the renderer
  never sees the Electron `event`, (3) return a `removeListener` disposer. Sanitize payloads in the
  preload.
- `createSettingsStore` **requires** an explicit `configDir` — it throws otherwise, by design.

### Server (`server/`)

- Domains are wired in exactly one place: `app/gateway-application.mjs`. `bootstrap.mjs` is 9 lines.
- **Dependency injection by default parameter**:
  `createGatewayApplication({ config = defaultConfig, logger = defaultLogger, agent = defaultAgent })`.
  Production gets the singleton, tests pass a substitute. Don't import a singleton inside a domain.
- **`BackendPort` adapters implement all 10 methods.** Optional behavior is advertised by
  `describe()` and rejected explicitly — callers never infer support from a missing method.
  `assertBackendPort()` throws `BackendPortContractError`. Run `backend-adapter-conformance.mjs`.
- A new ACP agent should need **configuration, not code** — `generic-acp` + `ACP_COMMAND`,
  `ACP_ARGS`, `ACP_WORKSPACE`, `ACP_LABEL`.
- New event projections are **added as a projector** (`transport/*-projector.mjs`); never change the
  canonical event.

---

## Persistence Rules

**There is no database.** No ORM, no migrations, no SQL. Durable state is versioned JSON snapshots.

Use `JsonSnapshotStore` or `VersionedJsonStore` (`server/src/core/`) — never raw `fs`. They encode:

| Policy | Why |
| --- | --- |
| Atomic write: `<path>.<pid>.tmp` → rename | A crash can never leave a half-file |
| Mode `0600` (dirs `0700`) | Several stores hold credentials |
| Corrupt/version-mismatched file → `<path>.corrupt-<ts>` + warn + continue empty | Never fail startup on bad data |
| Degrade, never crash — disable persistence rather than lose the original | |
| `ENOENT` is a **first run**, not an error | |
| `filePath = null` ⇒ memory-only | A supported mode, not a misconfiguration |

Every store caps its own growth (`MAX_OWNERS`, `MAX_EDIT_ITEMS`, `MAX_TASK_NUMBER`, …) — there is no
external compaction. Add a cap when you add a store.

---

## Security Rules

| Rule | Enforced in |
| --- | --- |
| Local access is zero-config; **remote requires a credential** | |
| **Secrets never travel inside GCP messages** | Subprotocol or `Authorization` header only |
| Credential + non-`wss:` socket → **throw** | `web/src/gateway-transport.js` |
| A Gateway URL must be a bare **origin** (no path/query/fragment/userinfo) | `cleanGatewayUrl`, `parseMobileGatewayProfile` |
| Mobile: `https:` only — checked at pairing **and** at profile load | `mobile-profile.js` |
| Electron: `https:` only, or `http:` on loopback | `security.mjs#validateAppUrl` |
| Pairing invitation stays in the **URL fragment** | Never in a request or access log |
| Device tokens stored **hashed** | `GatewayDeviceRegistry` |
| `shell.openExternal` gated by `isSafeExternalUrl` | |

**Never commit** `.env`, API keys, user profiles, task state, logs, or backend working directories.

---

## Testing Rules

- **`node --test` only.** Tests in `test/` beside the code, named `*.test.mjs`.
- Web convention is **1:1** — `web/src/foo.js` ⇄ `web/test/foo.test.mjs`. No DOM harness; logic is
  extracted into pure modules precisely so it is testable without a browser.
- **A fix must include a test that reproduces the bug**, not just one showing the fix works.
- Each of the 29 capabilities is locked by a test.
- New backend adapters run the conformance suite.
- Pure-function seams to extend rather than bypass: `desktop-surface-layout.mjs` (geometry without
  Electron), `gateway-client-state.mjs` (protocol state without React).

---

## i18n Rules

- **`t()` takes the Simplified Chinese string as the key**: `t('当前浏览器不支持实时语音播放')`.
  Author new user-facing strings in Chinese. This extends to asset filenames
  (`desktop/src/assets/icons/删除.svg`).
- **Every `.md` under `docs/` needs a `.zh.md` companion**, or `scripts/sync-docs-site.mjs` *throws*
  and `npm run docs:build` fails. It runs as a `pre` hook on all three `docs:*` scripts.
  Exclusions live in `EXCLUDED_DIRS` / `EXCLUDED_FILES` (currently `promo`, `roadmap`, `skad`,
  `node_modules`).

---

## Configuration Rules

- **New configuration must have a safe default**, and must update `.env.example` **and**
  `docs/configuration/` in the same change.
- Three distinct, non-interchangeable prefixes: `DASHSCOPE_*`, `QWEN_AUDIO_*` (feature flags),
  `QWEN_AUDIO_AGENT_*` (Gateway runtime, ~64 vars).
- Resolution order: stored `config.env` first, then live `process.env` for unset slots.
- Model changes are **not** hot-reloaded — the CLI emits a "restart the Gateway" follow-up.

---

## Workflow Rules

### Before any PR

```bash
npm run lint && npm test && npm run build && npm run release:check
```

### Commits

Conventional Commits with a scope and trailing PR number:
`feat(memory): support VoiceMem native audio input (#332)`

### PR description

State the **problem**, the **fix**, the **verification commands**, and **compatibility impact**.
Changes touching **networking, permissions, persistence, process management, or release** must also
list security implications and the rollback path.

### Other

- Don't reorder or rewrite large unrelated blocks — keep diffs reviewable.
- User-visible behavior changes update `CHANGELOG.md`.
- Releases: use `npm run version:*` (syncs 7 manifests), never hand-edit versions. Release work goes
  on a `release/x.y.z` branch. Publishing is changelog-driven — nothing ships unless the version
  changes.

---

## Quick Orientation

| Question | File |
| --- | --- |
| What routes exist? | `server/src/app/gateway-application.mjs` (28 registrations) |
| What is public API? | `package.json#exports` (42 entries) |
| What is the protocol? | `docs/gateway-protocol.md`, `docs/contract.md` (**normative**) |
| Protocol constants | `server/src/core/gateway-protocol.mjs` |
| Client protocol state | `shared/gateway-client-state.mjs` |
| Backend contract | `server/src/backend/backend-port.mjs` |
| Desktop IPC contract | `desktop/src/preload.cjs` |
| Web network chokepoint | `web/src/gateway-transport.js` |
| Full scan documentation | `docs/skad/index.md` |
