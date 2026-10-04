# 00 — Orientation: StarNet fork (javontaymcelroy/starnet)

Audit date: 2026-10-04. Audited commit: `e0a36dcd` on `feat/harness-backend` (v0.13.1).
Scope rule for this and later phases: read-only against source; findings go to `audit/findings.md`.

## 1. What this actually is (premise check)

The audit brief calls this a "forked production web app". It is not a hosted web app. It is a
**local-first desktop AI-agent harness**: one Node process ("sidecar") that serves a vanilla-JS
single-page UI over loopback HTTP and runs the agent loop in-process, wrapped in a Tauri 2 desktop
shell for Windows/macOS. There is no multi-tenant server, no user accounts by default, no shared
database. "Production" here means the shipped installer plus the operator-run cloud pieces
(credits/billing, skill market, phone relay), which live mostly in *other* repos.

Consequences for the audit plan:

- "Authorization on every endpoint / IDOR" translates to: loopback API token model, Origin and Host
  pinning, the capability (tool-grant) gate, consent broker, and per-agent workspace containment.
- "Never touch production" translates to: never point a test sidecar at a real workspace directory,
  never run with real provider keys, keep `STARNET_REMOTE_RELAY=off`, never call
  `account.starnetos.com` / `starnetos.com` market endpoints.
- "Mobile / tablet widths" is partly moot for a desktop-only shell, but still relevant to the
  browser-served mode (`npm start` at :8787) and the phone remote app in `relay/app/`.

## 2. Stack

| Layer | Technology | Where |
| --- | --- | --- |
| Backend ("sidecar") | Node.js >= 18 (22 in CI), CommonJS, mostly core modules. HTTP + SSE + NDJSON streaming on `127.0.0.1:8787`. | `sidecar/` (~72k lines, 230+ files). `sidecar/index.js` alone is ~25k lines and is the composition root plus all routes. |
| Agent loop | `runAgentLoop()` messages-array loop, tool-call accumulation and repair | `sidecar/loop.js` (~2.4k lines) |
| Frontend | Vanilla JS, no bundler, no framework. `index.html` loads ~254 `<script>` tags in dependency order; canvas-rendered pixel-art station. | `frontend/` (`frontend/app/` ~146k lines, ~250 modules) |
| Shared contract | Zero-dep event schema validator and ~60 frozen event types | `shared/` (~1.1k lines) |
| Desktop shell | Tauri 2 (Rust), embedded Node binary, NSIS/DMG, signed updater, `keyring` crate for OS credential store | `src-tauri/` (~7.9k lines Rust) |
| CLI | `starnet -p "..."` headless runner that finds or boots a sidecar | `bin/starnet.js` |
| Phone remote relay | Dependency-free WebSocket relay + PWA phone client, deployed to Fly.io (`remote.starnetos.com`) | `relay/` |
| Marketing/market site | Static site (`starnetos.com`), includes a generated demo copy of the app under `website/app` | `website/` |
| Tests | ~1,250 plain Node scripts using a tiny in-repo assert helper (`test/_assert.js`), run sequentially one-process-per-file by `scripts/run-test-list.mjs` from manifests. CDP-driven e2e tests need a Chrome binary. | `test/` (~188k lines), `test/fast.list` (1,054 lines), `test/http.list` (186), `test/customer-journeys.list` (49) |
| QA/release tooling | Large bespoke QA "station": ledgers, golden screenshots, soak, release tiers t0–t5 | `scripts/`, `qa/`, `loops/`, `dev/` |

No TypeScript, no ESLint/Prettier/Biome config, no `tsconfig`/`jsconfig`. The only static
checks are bespoke scripts (e.g. `lint-determinism`, `lint:evidence-secrets`, Gitleaks for history).

### Runtime npm dependencies (package.json)

| Package | Used by | Notes |
| --- | --- | --- |
| `ajv` 8.20.0 | `sidecar/result-contract.js` | schema validation of tool results |
| `undici` ^6 | `sidecar/index.js` (connector agent), `tools/builtin/web.js` | pinned dispatcher / IPv4 fallback for web tools |
| `node-pty` ^1.1 | `sidecar/index.js` terminal sessions | native module; loaded in a try/catch |
| `@huggingface/transformers`, `kokoro-js`, `ogg-opus-decoder` | `sidecar/local-voice.js`, `media-service.js` | local STT/TTS; staged into the desktop bundle by `scripts/stage-voice-deps.mjs` |
| `@napi-rs/canvas` (dev) | screenshot/golden tooling | native |
| `@tauri-apps/cli` (dev) | desktop build | |
| overrides: `adm-zip` 0.6.1, `sharp` 0.35.4 | transitive pins | |

The README's claim that the sidecar "uses Node core modules only" is no longer strictly true
(`ajv`, `undici`, `node-pty` are required at runtime paths), though the server boots without them
for the basic UI path.

## 3. Architecture

```
Browser or Tauri webview  ──HTTP/SSE/NDJSON──►  sidecar/index.js  (127.0.0.1:8787)
   frontend/app/*.js (U.bus)                      ├─ ~100 /api/* route families
   localStorage save v6 + sidecar mirror           ├─ loop.js  → providers/*  → model APIs (your key)
                                                   ├─ capability/ (station layout → tool allowlist)
                                                   ├─ tools/builtin/* (fs, shell, web, browser, computer, …)
                                                   ├─ channels/ (telegram, discord, slack, matrix, signal)
                                                   ├─ mcp/ (MCP connectors, OAuth 2.1, Google/Gmail)
                                                   ├─ cron*.js, nightshift, loopjob (unattended work)
                                                   ├─ remote/ (phone relay client; optional LAN listener 0.0.0.0:8797)
                                                   └─ *-store.js  → JSON/NDJSON files under <workspaces>/
```

### Authentication and session model

- **No user accounts.** Single OS user, single station.
- **Loopback API token**: a per-launch random 32-byte hex token (`SKYNET_API_TOKEN` or generated)
  must be sent as `X-StarNet-Token` on every `/api/*` route except a documented exempt set
  (`/api/key`, `/api/channels/token`, `/api/health`, two OAuth callbacks). Header-less surfaces
  (file open, EventSource, unload beacon) use short-lived scoped tickets (`sidecar/apitickets.js`).
  Logic lives in `sidecar/apiauth.js` with a written threat model: defends against malicious
  websites (Host pin vs DNS rebinding, Origin allow-list, custom header); explicitly does *not*
  defend against same-OS-user malware.
- **Desktop**: Rust shell generates the token, passes it via env to the sidecar, and injects it
  into the webview via an init script (`src-tauri/src/main.rs`).
- **Provider credentials**: OS keychain via Tauri (`credentials.rs`); in browser mode the sidecar
  holds keys server-side. OAuth flows exist for ChatGPT/Codex, Google, Spotify, MCP connectors,
  GitHub device flow.
- **Authorization inside the app**: capability gate (`sidecar/capability/`), consent broker
  (`permissions.js`, `permgrants.js`), per-agent workspace jail (`workspace-*.js`, `pathtrust.js`),
  "Full Access" bypass scopes, spend caps (`budgetcaps.js`, $25/day soft default).
- **Inbound network surfaces beyond loopback**: channel webhook ingress
  (`POST /api/channels/webhook/<channel>`, HMAC + nonce replay guard in `channels/webhook-auth.js`),
  optional LAN remote listener on `0.0.0.0` when `STARNET_REMOTE_LAN=1`, and the outbound relay
  client to `remote.starnetos.com` (default on unless `STARNET_REMOTE_RELAY=off`).

### Data layer

- Everything is files. Per-subsystem stores write JSON / NDJSON / JSONL with an atomic
  fsync-then-rename primitive (`durable-write.js`, `savestore.js`). Dozens of file kinds
  (`jobs.json`, `ledger.jsonl`, `connectors.json`, `permissions`, transcripts, checkpoints, …).
- Default roots: Windows `%APPDATA%\ai.skynet.harness\workspaces` (desktop) or
  `%LOCALAPPDATA%\StarNet\workspaces` (bare sidecar); macOS
  `~/Library/Application Support/ai.skynet.harness/workspaces`; Linux `~/.local/share/...`.
  Override with `SKYNET_WORKSPACES` / `STARNET_WORKSPACES`.
- Frontend keeps a versioned `localStorage` save (schema `starnet.save`, v6) mirrored to the
  sidecar via `/api/save`.
- Boot-time "station recovery" scans real app-data roots for a lost station. The test runner
  sandboxes `HOME`/`APPDATA`/`XDG_DATA_HOME` per step specifically so test sidecars never ingest a
  developer's real station. Any manual run in this audit must do the same.

### Third-party services (all outbound, all optional except the update check)

| Service | Purpose | Trigger |
| --- | --- | --- |
| OpenRouter, OpenAI, Anthropic, Google Gemini, ChatGPT OAuth, xAI, Groq, Together, Ollama (local), Claude CLI | Model providers | user-configured |
| Telegram, Discord, Slack, Matrix, Signal (signal-cli REST) | Chat channels | user-connected |
| Google (Gmail IMAP/SMTP, Drive, Docs, Sheets), GitHub, generic MCP servers (HTTP/stdio) | Connectors | user-connected |
| Spotify | Playback integration | user-enabled |
| Mojeek, DuckDuckGo, Jina Reader | Keyless web search/read | agent tool use |
| ElevenLabs, Edge TTS | Voice | user-configured |
| `github.com/androoAGI/starnet-releases` `latest.json` | Update check | on by default |
| `starnetos.com/.well-known/starnet-skills*.json`, `/skills/` | Skill Market | when opened / while installed |
| `account.starnetos.com` | StarNet Credits (managed billing) | only if the user buys credits |
| `remote.starnetos.com` (Fly.io) | Phone remote relay | default on (`STARNET_REMOTE_RELAY=off` disables) |
| Docker (`node:20-bookworm`) | Optional sandboxed execution backend | `STARNET_EXEC_BACKEND` |

PRIVACY.md documents most of this and matches what grep found; the relay default-on and
the Skill Market revocation poll are the items most worth re-verifying in Phase 4.

## 4. Fork vs upstream (important)

- Upstream: `androoAGI/starnet`. Fork: `javontaymcelroy/starnet`.
- The fork has **one branch** (`feat/harness-backend`), **zero PRs, zero issues**, and its HEAD
  `e0a36dcd` is **identical to upstream's `feat/harness-backend` HEAD** as of today. There are
  currently no fork-introduced commits to diff.
- Upstream has 138 refs (many `agent/*` lanes). All 126 commits visible in this shallow clone are
  authored by a single identity ("andro"). Commit style suggests heavy use of autonomous coding
  agents merging many lanes per day; docs warn that plan/audit docs go stale within hours.
- The clone is shallow (126 commits), which blocks full-history secret scanning and the
  "claims-authority" test that CI runs with `fetch-depth: 0`.

## 5. Install, run, test locally

Toolchain present in this container: Node 22.22, npm 10.9, cargo/rustc 1.97, python3/make/g++,
Playwright Chromium at `/opt/pw-browsers/chromium`. No system Chrome. `node_modules` not installed.

```bash
npm ci                                   # native deps: node-pty, @napi-rs/canvas, sharp → needs build tools (present)

# bare sidecar, cold first-run flow (browser at http://127.0.0.1:8787)
SKYNET_WORKSPACES=/tmp/starnet-ws node sidecar/index.js

# pre-onboarded dev seed (skips onboarding); needs dev/.env.dev (copy from dev/.env.dev.example)
npm run dev:seed            # fresh scratch workspace each run, SKYNET_DEV=1, SKYNET_FULL_ACCESS=1
npm run dev:seed:keep

# tests
npm run test:fast           # THE merge gate: 1,054-line manifest, sequential, ~4 min in CI
npm run test:http           # live sidecar HTTP/e2e suite (186 steps)
npm run qa:customer-journeys
npm test                    # fast + http
npm run eval:gate           # credential-free eval slice CI runs on every PR
npm run security:secrets    # needs gitleaks in PATH and full history

# desktop (not attempted here; needs Tauri prerequisites + webkit on Linux)
npm run desktop:dev / desktop:build
```

CI (`.github/workflows/fast-gate.yml`) wraps Chrome in a `--no-sandbox` shim and exports it as
`SKYNET_CHROME`; the same will be needed here for the ~27 CDP e2e tests. There is no `lint` or
`typecheck` script to run; Phase 1 should record that explicitly rather than invent one.

Phase 1 will also need `dev/.env.dev` with a key, or a mock provider, to exercise any agent run.
Many e2e tests already use loopback mock providers, so a real key is not required for the gates.

## 6. Environment variables (observed, not exhaustive)

`ENV(suffix)` in `sidecar/index.js` reads `STARNET_<suffix>` first, then `SKYNET_<suffix>`.

| Variable | Role |
| --- | --- |
| `PORT` / `SKYNET_PORT` | sidecar port (default 8787) |
| `SKYNET_API_TOKEN` | per-launch API token (desktop sets; else random) |
| `SKYNET_IPC_TOKEN` | guards `/api/key` and `/api/channels/token` desktop pushes |
| `SKYNET_WORKSPACES` / `STARNET_WORKSPACES` | data root |
| `SKYNET_DEV`, `SKYNET_FULL_ACCESS` | dev seed fast-path, consent bypass (dev only) |
| `SKYNET_OPENROUTER_KEY`, `OPENROUTER_API_KEY`, `SKYNET_OPENROUTER_BASE`, `SKYNET_DEFAULT_MODEL` | BYOK dev provider |
| `SKYNET_CHROME` / `STARNET_CHROME`, `SKYNET_CDP_PORT` | browser tool + e2e tests |
| `STARNET_REMOTE_RELAY` (`off` or URL), `STARNET_REMOTE_LAN`, `STARNET_REMOTE_PORT` | phone remote |
| `STARNET_CREDITS_URL` | managed credits backend (unset = feature inert) |
| `STARNET_CONNECTOR_ENCRYPTION_KEY` | connector vault at-rest key |
| `STARNET_CHANNEL_WEBHOOK_SECRET` | inbound webhook HMAC |
| `STARNET_EXEC_BACKEND`, `STARNET_DOCKER_IDLE_MINUTES` | execution backend |
| `STARNET_SKILL_MARKET_URL`, `STARNET_SKILL_MARKET_KEY(S)` | skill market override / signing |
| `SKYNET_FALLBACK_MODELS`, `SKYNET_ANTHROPIC_CACHE_TTL`, `SKYNET_BUDGET_MANAGED_PER_RUN` | model routing / spend |
| `TAURI_SIGNING_PRIVATE_KEY*`, `APPLE_*`, `AZURE_*`, `RELEASES_TOKEN` | release signing (CI only) |
| ~60 `STARNET_T0..T5_*`, `STARNET_PHASE*_*` | release-tier proof scripts, not runtime |

## 7. Prior security work already in the repo

- `docs/SECURITY_HARDENING.md`: local API auth rework (token on every route, Origin/Host pinning).
- `dev/SECURITY-AUDIT-0904.md`: five fixed P1/P2 issues (connector credential inheritance across
  endpoints, MCP header leak on cross-origin redirect, plaintext secrets in "secrets excluded"
  backups, backup round-trip re-enabling disabled connectors, permission reset reporting success
  after failed write). These are good regression targets and good hints at the classes of bugs
  this codebase produces.
- `.gitleaksignore` lists reviewed false positives; `security:secrets` is a CI workflow.
- `qa/KNOWN_ISSUES.md`, `qa/BUGS.md`, `qa/STATUS.md`: live defect ledgers (very long).

## 8. What I could not determine

1. **Whether this fork is meant to diverge.** With zero fork-only commits, Phase 1's "diff against
   upstream" is empty today. Either the divergence is planned, or lives on a branch not pushed,
   or the audit is really of upstream v0.13.1.
2. **Which "production" the user means**: the shipped desktop installer, the browser-served
   sidecar, or the operator cloud (credits, market, relay) whose code is in other repos
   (`starnet-cloud`, `starnet-releases`) not in scope here.
3. **Full git history**: shallow clone; cannot run history secret scans or the claims-authority
   test without unshallowing (upstream is public, so this is fixable).
4. **Desktop (Tauri) behaviour**: cannot build or run the Windows/macOS shell in this Linux
   container. Keychain, updater, WebView2 recovery, signing are static-review only.
5. **Native dependency install**: `npm ci` not yet attempted; `node-pty` and `sharp` may need
   network access to prebuilt binaries or a compile.
6. **Live provider behaviour**: no keys; all agent-run testing will use the in-repo mock providers.
7. **Whether the CODE_MAP/BRAIN line counts are trusted anywhere**: `sidecar/index.js` is ~25k
   lines vs the 6.5k–14.8k the docs claim. Treat docs as hypotheses, as the repo itself advises.
8. **Which areas matter most to the user** (see questions).
