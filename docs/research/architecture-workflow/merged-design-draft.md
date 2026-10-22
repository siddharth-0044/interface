# Interface architecture

*Plan as of 2026-10-20; nothing is built yet. It merges three competing proposals: the "thin fork" won, with ideas grafted from the other two, all checked against verified facts. Jargon is explained in section 12.*

## 1. Summary

Interface is a free, MIT-licensed Fedora desktop app that runs Claude Code, OpenCode and Muse Code side by side, each on your own subscription. The app is a GitHub fork of [Parallel Code](https://github.com/johannesjo/parallel-code) (MIT, Electron + SolidJS) with about 25 changed upstream lines. Our code lives in this repo as plain TypeScript packages and ships into the app as one checksummed bundle. The headline feature is one local MCP hub. You store your Parallel, Firecrawl, Octen and Context7 keys once in the GNOME keyring. The hub serves their tools on 127.0.0.1, and every agent the app starts connects with its own short-lived token. No CLI config file is edited. Later stages add provider CLIs, "Continue in..." handoff and usage-limit switching.

## 2. The big picture

```
+---------------- Interface Desktop (fork of Parallel Code) ----------------+
| Upstream, unchanged:                Ours (new files + ~25 seam lines):    |
| tasks, git worktrees, diff review,  Settings > MCP Hub, Hub & Keys window,|
| agent terminals, usage bar,         launch seam, core bundle (this repo)  |
| second-opinion composer                                                   |
+-----------+---------------------------------+-----------------------------+
            | starts agents in terminals      | starts as utilityProcess
            v                                 v
 claude | opencode | muse --- token T --> MCP hub on 127.0.0.1 --> Parallel Search + Task
 (your subscriptions)      (per launch)        |                --> Firecrawl
                                              | secret-tool    --> Octen
                                              v                --> Context7
                                GNOME Keyring (Secret Service)
```

## 3. Components

| Component | What it does | What we reuse (project, license, how) | What we write ourselves |
|---|---|---|---|
| Desktop shell | Worktrees, agent terminals, diff review, usage bar, handoff composer | [Parallel Code](https://github.com/johannesjo/parallel-code), MIT, fork base | Muse preset, seam lines, generated build config, workflows |
| Desktop glue (`packages/desktop`) | Runs the hub; Hub & Keys window, launch hook, tokens | Parallel Code's 0600-file and delete-on-exit patterns, MIT, copied | `start(host)` contract, Keys window |
| Key vault (`core/vault`) | Keeps keys in the Secret Service | `secret-tool` from [libsecret](https://gitlab.gnome.org/GNOME/libsecret), LGPL-2.1, separate program; [@napi-rs/keyring](https://github.com/Brooooooklyn/keyring-node), MIT, fallback only | `SecretStore`, in-memory store for CI |
| Catalog (`core/catalog`) | URL and key style per provider | Provider docs | Parallel Search + Task (one key; Search works keyless), Firecrawl (Bearer), Octen (`x-api-key`), Context7 (optional Bearer) |
| Hub (`packages/hub`) | One guarded MCP endpoint that forwards calls with the right key | [MCP TypeScript SDK v2](https://github.com/modelcontextprotocol/typescript-sdk), Apache-2.0, npm; [1MCP](https://github.com/1mcp-app/agent) tool naming, idea only | Aggregator, `<provider>_<tool>` names, health checks, token set |
| Launch planner (`core/launch`) | Extra args, env and a 0600 file per agent launch | Parallel Code `agent-args.ts`, MIT, idea only | Claude, OpenCode and Muse builders with per-version fixtures; guards |
| `ifc` CLI (`packages/cli`) | `doctor` (S2); `launch`, `run`, `sync`, `service` (S3) | Node's built-in `parseArgs` | Commands |
| Global sync (S3, opt-in) | One `interface` entry in user configs | [jsonc-parser](https://github.com/microsoft/node-jsonc-parser), MIT, npm; `claude mcp`, subprocess; [rulesync](https://github.com/dyoshikawa/rulesync) Muse field mapping, MIT, ported | Backups, atomic writes, unsync |
| Provider CLI keys (S3) | Key for `parallel-cli` or `firecrawl` in one process only | Vendor CLIs, subprocess, pinned | `ifc run`, PATH shims |
| Handoff (S4) | Continue a task in another CLI | Parallel Code handoff composer, MIT, fork base; `muse export`, `opencode session export`; [Waybill](https://github.com/wardmos/waybill), Apache-2.0, format idea; [continues](https://github.com/yigitkonur/cli-continues), MIT, optional | Transcript readers, brief packer |
| Limits (S5) | Detect limits, wait, switch, resume | Parallel Code usage readers and StopFailure hook, MIT, fork base; [cezar](https://github.com/open-mercato/cezar) `usage-limit.ts`, MIT, vendored | State machine, Waiting-for-usage UI |

Not used, on purpose: [add-mcp](https://github.com/neon-solutions/add-mcp) (non-atomic writes, no Muse), 1MCP and [mcp-proxy](https://github.com/punkpeye/mcp-proxy) at runtime (mcp-proxy's defaults leave an unauthenticated path), [CodexBar](https://github.com/steipete/CodexBar) (undocumented Muse key endpoint). No code from GPL, AGPL, BUSL or Elastic-licensed projects.

## 4. Repositories and layout

```
siddharth-0044/interface        this repo, MIT: plans + our code
|-- openspec/                   every change, fork patches included
|-- packages/
|   |-- core/      vault, catalog, launch planner, later handoff + limits
|   |-- hub/       MCP server + clients (only user of MCP SDK v2)
|   |-- desktop/   start(host): hub supervisor, Hub & Keys window
|   +-- cli/       ifc
|-- scripts/       wt-new, ship, wt-done
+-- .github/workflows/  ci.yml, release-core.yml
          |
          | release asset interface-core-<v>.tgz (sha256-pinned)
          v
siddharth-0044/interface-desktop   GitHub fork of johannesjo/parallel-code
|-- main         mirror of upstream (bot only; base for upstream PRs)
+-- interface    default branch = main + our files and seams
    |-- interface/            PATCHES.md, core.lock.json, scripts
    |-- electron/interface/   host.ts, launch-seam.ts, core.d.ts
    |-- src/interface/        HubSettings.tsx
    |-- .github/workflows/    interface-ci, interface-release,
    |                         upstream-sync, core-bump
    +-- seams: main.ts +2, pty.ts +2, agents.ts +10, skip-permissions.ts +1,
        SettingsDialog.tsx +4, channel-manifest.json +1, preload.cjs +1,
        store/core.ts +1 (only if decision 5 is yes)
```

- **How they connect.** A `core-v*` tag here publishes `interface-core-<v>.tgz` and its sha256. A scheduled job in the fork opens a PR that bumps `interface/core.lock.json`. The build runs `fetch-core.mjs`, which checks the hash and unpacks into `dist-electron/interface-core/`. Upstream already git-ignores and packs that folder, so the fork never edits `package.json` or the lockfile. `fetch-core --local ../interface` uses your local build. Without the bundle, the app runs as plain Parallel Code.
- **Build.** Our script generates the electron-builder config: appId, product name, update feed (our fork), AppImage + rpm (requiring `libsecret` and `nodejs`), Electron fuses.
- **Upstream sync.** A weekly bot mirrors upstream into `main` and opens a PR into `interface`. It is merged with a merge commit, never squashed or force-pushed. `check-seams` fails CI when files outside our folders and `PATCHES.md` differ from upstream. Upstream's `release.yml` and `android.yml` are switched off with `gh workflow disable`. CI on bot PRs starts after a maintainer clicks approve.
- **Contributing.** Every change, fork patches included, starts as an OpenSpec change here. Most code goes into `packages/` (plain TypeScript, tests run anywhere). Building the fork on Fedora needs `sudo dnf install gcc-c++ make rpm-build`, because node-pty has no Linux prebuild. Both repos use one loop: `scripts/wt-new`, change, `scripts/ship` (PR, CI, squash auto-merge), `scripts/wt-done`.
- **CI here:** lint, typecheck and tests. Dependency rules: core imports neither Electron nor the MCP SDK, and only the hub and `ifc run` read keys. Hub tests run against mock Bearer and `x-api-key` providers, including rejected requests. Also: launch fixtures, a spawn-env test, `openspec validate --strict`, a license allowlist and gitleaks.
- **CI in the fork:** upstream's own `ci.yml`, check-seams, a contract typecheck against the pinned bundle, a build, and checks on appId, update owner and fuses. The license allowlist has entries for upstream's MPL-2.0, OFL-1.1 and proprietary Agent SDK packages. Gitleaks runs too. A `v*` tag publishes AppImage + rpm.

## 5. What happens when you start an agent

1. At app start, `main.ts` calls our `host.ts`. It loads the core bundle and hands it a few upstream functions (`onPtyEvent`, a launch-hook setter).
2. The core starts the hub as a utilityProcess. The hub reads `~/.config/interface/hub.json` (no secrets), gets keys with `secret-tool lookup`, binds 127.0.0.1:7337 (or a free port) and connects to providers lazily.
3. You start a task. For Claude, upstream adds its own canvas MCP config. Then `spawnAgent()` in `electron/ipc/pty.ts` builds the environment and calls `applyAgentHookLaunch`.
4. Our seam runs right after, synchronously. It skips shells, Docker tasks, unknown CLIs and globally synced CLIs. It removes `INTERFACE_*` variables coming from env files, applies the billing guard, mints token T and posts it to the hub. Then it adds, per CLI:
   - Claude: a 0600 file in `$XDG_RUNTIME_DIR/interface/launch/`, added to the existing `--mcp-config` list.
   - OpenCode: `--standalone` plus `OPENCODE_CONFIG_CONTENT` with `mcp.servers.interface` (`type: remote`, `oauth: false`).
   - Muse: `-c mcpServers.interface={...,"required":false}`, with header `${INTERFACE_HUB_AUTH}` set in that process's env.
5. The CLI connects with `Bearer T`. The hub checks Host, Origin and T, then lists tools like `parallel_web_search`.
6. Calls are forwarded with the key from the keyring, and results come back unchanged.
7. When the terminal exits, T is revoked and the file is deleted.

```
you     app main + seam          hub (utilityProcess)    agent CLI          provider
 | start  |                         |                       |                  |
 |------->| mint token T            |                       |                  |
 |        |-- register T ---------->|                       |                  |
 |        |-- spawn CLI, T in file or env ----------------->|                  |
 |        |                         |<-- POST /mcp + T -----|                  |
 |        |                         | check Host, Origin, T |                  |
 |        |                         |-- tool list --------->|                  |
 |        |                         |<-- call tool ---------|                  |
 |        |                         |-- call + key from keyring -------------->|
 |        |                         |<----------------------------- result ----|
 |        |                         |-- result ------------>|                  |
 |        |<-- terminal exits ------------------------------|                  |
 |        |-- revoke T, delete file >|                      |                  |
```

The seam doesn't reach upstream's Claude chat view or Docker tasks.

## 6. Keys and security rules

- Keys live only in GNOME Keyring, through `secret-tool`, with the secret passed on stdin. There is no fallback store.
- Only the hub (and from S3, one `ifc run` child) holds keys. They never go into CLI configs, argv, logs, the renderer, upstream state or agent environments. A CI test checks every preset's spawn env.
- Windows can set, delete and check keys, never read them.
- Each key goes only to its own provider's host.
- With a locked keyring, keyless providers (Parallel Search, Context7) keep working and keyed ones show as failing. Keyless Firecrawl is flagged, because it silently drops to three tools.
- The hub listens on 127.0.0.1 only, checks Host, rejects any request with an Origin header, sends no CORS headers and has no SSE endpoint. On every route it checks the token (constant-time) before reading the body.
- Tokens are per launch, kept in memory and revoked on exit. Claude gets a 0600 file; OpenCode and Muse get the token in that one process's env. Only opt-in global sync uses a long-lived token, and configs reference it instead of containing it.
- The seam deletes `INTERFACE_*` variables from env files before setting its own.
- Billing guard: Claude launches drop `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` and `CLAUDE_CODE_USE_*` by default, with a notice and a per-agent opt-in. A stray `ANTHROPIC_API_KEY` silently switches Claude Code to pay-per-use billing.
- Vendor installers run pinned, with `--skip-auth`, because Firecrawl's browser login saves the key in plaintext.
- Shims run absolute binary paths recorded at setup.
- Handoff briefs are untrusted. They are quoted as data and shown to you before sending, except in opt-in auto-switch.
- Honest limit: the keyring has no per-app access control, and on this machine any process running as you can read other processes' memory (ptrace_scope is 0). The keyring prevents plaintext at rest and accidental leaks. It does not stop a malicious agent running as you.

## 7. Roadmap

**S1, Desktop 0.0.1: the fork runs as our app (1-2 weeks)**
- You can use: the rpm or AppImage on Fedora, running Claude Code, OpenCode and Muse (new preset) in parallel worktrees with diff review. Updates come from our releases.
- Includes: `setup-dev-loop` and `fork-desktop-shell`, a Muse-preset issue upstream, and helping rpm PR [#315](https://github.com/johannesjo/parallel-code/pull/315) land.

**S2, Desktop 0.1.0: the first usable release (3-4 weeks)**
- You can use: Settings > MCP Hub > Hub & Keys. Paste keys once, and every Claude Code, OpenCode and Muse agent you start gets the Parallel, Firecrawl, Octen and Context7 tools, with a health dot per provider.
- Done when, on Fedora 44: you install the rpm, add Parallel and Context7 keys and start one agent per CLI. `parallel_web_search` and one Context7 tool succeed in each (6 calls). grep finds no key and no `interface` entry in `~/.claude.json`, the OpenCode config or `~/.config/muse/settings.json`.
- Includes: a 2-3 day spike gate first (section 11). If a CLI fails its spike, it gets the global-sync adapter in this stage instead. Then `add-mcp-hub` (revised) and `desktop-hub-integration`, plus an upstream issue asking for a launch hook in `spawnAgent`.

**S3, 0.2: terminal and provider CLIs (2-3 weeks)**
- You can use: `ifc launch claude|opencode|muse` in any terminal, an optional systemd hub and opt-in global sync. `parallel-cli` (FindAll, Monitor) and `firecrawl` work inside agents with per-process keys, and buttons install vendor skills.
- Includes: `terminal-and-global-sync`, `provider-cli-keys`.

**S4, 0.3: Continue in... (about 2 weeks)**
- You can use: one action writes a handoff brief and continues the task in another CLI in the same worktree.
- Includes: `cross-cli-handoff`.

**S5, 0.4: limits and auto-switch (about 3 weeks)**
- You can use: "Waiting for usage" with the reset time, a suggested switch, opt-in auto-switch, and resume at reset.
- Includes: `usage-limits-autoswitch`; later `add-more-agent-clis`.

## 8. OpenSpec changes

| # | Change | Action | Scope |
|---|---|---|---|
| 1 | `setup-dev-loop` | new | Package scaffold, CI, worktree scripts, repo settings; takes `add-mcp-hub` tasks 1.1-1.6 |
| 2 | `fork-desktop-shell` | new | Fork, branches, Muse preset, build config, sync, check-seams, releases |
| 3 | `add-mcp-hub` | revise | Vault, catalog, hub, launch planner, spikes |
| 4 | `desktop-hub-integration` | new | Host contract, seams, hub process, Hub & Keys window, tokens, core releases, S2 acceptance |
| 5 | `terminal-and-global-sync` | new | `ifc` in the rpm, `ifc launch`, systemd unit, opt-in global sync |
| 6 | `provider-cli-keys` | new | `ifc run`, shims, skill installers |
| 7 | `cross-cli-handoff` | new | Transcript readers, briefs, Continue in... |
| 8 | `usage-limits-autoswitch` | new | Limit watcher, waiting state, auto-switch, resume |
| 9 | `add-more-agent-clis` | new | Codex, Gemini, Grok |
| - | `fork-shell-integration` | drop | Only a planned name in `add-mcp-hub/tasks.md`; split into 2, 4 and 7 |

**What happens to `add-mcp-hub`** (revise it with `/opsx:update`): it stops being "terminal-first, no GUI" and becomes the app's engine. Task groups 1 (repo, CI) and 4 (global adapters) move out. D2 becomes `secret-tool` behind `SecretStore`. D5 becomes per-launch tokens with Host and Origin rules. D6 becomes per-launch injection, and in D7 the hub runs inside the app. The catalog gains Octen's header, Parallel's two endpoints, Firecrawl's keyless warning and Context7's optional key. The entry is renamed `hub` -> `interface`. Spike tasks come first.

## 9. Decisions we still need from you

1. **Product name.** It goes into the appId, rpm, keyring entries and update feed. *Recommend:* decide before the S1 release; renaming later breaks updates.
2. **Upstream sync:** a weekly merge-commit PR, or a rebased patch stack with force-pushes? *Recommend:* merge-commit PR, so friends' open PRs never lose their base.
3. **Keyring backend:** `secret-tool`, or the native `@napi-rs/keyring`? *Recommend:* `secret-tool` (part of Fedora's libsecret, no native build). This revises design D2.
4. **Hub lifetime:** owned by the app now with an optional systemd service in S3, or systemd from the start? *Recommend:* app-owned now.
5. **Upstream's Claude chat view:** off by default, or on? Turning it off means flipping upstream's existing "Chat is turned off for Claude Code" setting, a one-line default change. *Recommend:* off. The safety review flagged Anthropic's policy on Agent-SDK products, and the hub doesn't reach that view.
6. **Billing guard:** strip Anthropic API variables from Claude launches by default, or only warn? *Recommend:* strip, with a per-agent opt-in.
7. **Muse quota (S5):** limit messages only, or opt-in CodexBar? *Recommend:* limit messages only.

## 10. Risks of depending on other projects

| Dependency | Risk | Mitigation |
|---|---|---|
| Parallel Code | One main maintainer, about 6 commits a day (`pty.ts` 28 and `register.ts` 68 in 90 days) | Logic in new files, 1-4 line seams, check-seams, weekly sync PR whose typecheck catches seam breaks, upstream hook issue. Fallback: wrapper-script custom agents |
| Agent CLIs | Flags and config shapes drift (Muse `-c`, OpenCode v1/v2, Claude `--mcp-config`) | Spikes. Fixtures pinned to Claude 2.1.304, OpenCode 2.0.32, Muse 1.4.11. `doctor` flags unknown versions. Skip injection rather than break a launch; global-sync fallback |
| MCP SDK v2 | New major line | Exact pin, hub package only, Apache license text kept |
| libsecret | Missing or locked keyring | The rpm requires libsecret; clear errors; `@napi-rs/keyring` pinned to `secret-service` as fallback |
| Vendor CLIs, continues | Plaintext logins. continues has had no commits since 2026-05-07 and parses OpenCode's old database | Pins, `--skip-auth`, `doctor` scans credential files, official exports first |
| Inherited packages | Proprietary Agent SDK, MPL-2.0, OFL-1.1 | Explicit allowlist entries; chat view off |
| Build tools | node-pty compiles on Linux; rpm is built on Ubuntu runners | Prerequisites check; CI installs `rpm` (as PR #315 does) |

## 11. Not verified yet

Spike gate, at the start of S2:
- Muse accepts `-c mcpServers.interface=<JSON>` for a new server name, with a `${VAR}` header (test with `--provider echo`).
- OpenCode reads a per-launch `OPENCODE_CONFIG_CONTENT` only with `--standalone`, those sessions show in the normal list, and prefixed tools work under Code Mode.
- Claude accepts our config next to upstream's in one `--mcp-config` list, without prompts.
- The CLIs send no Origin header.
- A utilityProcess runs the hub from the packaged app, and the fuses break nothing. Upstream's own MCP server uses system `node`, which is a good sign.

Later:
- A locked keyring shows an unlock prompt when `secret-tool` runs from the app or a systemd service.
- electron-builder builds our rpm on Ubuntu, AppImage updates work from our fork, and the app keeps its own data folder next to an installed Parallel Code.
- Agent sandboxes let shims reach D-Bus (Muse's sandbox is on unless `--yolo`). Also unverified: Parallel's skill installer, and whether skills call bare `parallel-cli` and `firecrawl`.
- `muse export` and `opencode session export` are enough for briefs; StopFailure text identifies Claude limits; what OpenCode and Muse print at a limit.
- Claude user-scope entries accept `headersHelper`, and tokens reach the hub before agents connect.
- Docker tasks on Linux might reach host loopback (an upstream comment says so for `--network host`). This is untested, so Docker tasks are skipped.

**Corrected after checking (refuted claims):**
- Parallel Code does not pass `--mcp-config` to OpenCode or custom agents, and `buildMcpLaunchArgs` isn't called from `pty.ts`. So the thin fork's "fix" (P3) and its upstream bug report (U1) are dropped, and the upstream hook request (U3) now targets `spawnAgent`.
- Canvas MCP is not limited to orchestration: every fresh Claude spawn gets it, so our config joins that same `--mcp-config` list.
- Upstream builds AppImage and deb, not rpm. We add rpm ourselves (PR #315 is still open).
- OpenCode issue #5299 is fixed, so `{env:}` and `{file:}` substitution can be used.

## 12. Glossary

- **MCP:** the standard way agents find and call tools. A *server* offers tools; the agent is the *client*.
- **Hub:** our MCP server, bundling many providers behind one address.
- **Provider:** a tool service you pay for (Parallel, Firecrawl, Octen, Context7).
- **CLI:** a command-line program; here, the coding agents.
- **Keyring / Secret Service:** the desktop password store (GNOME Keyring); `secret-tool` talks to it.
- **Loopback (127.0.0.1):** an address only your machine can reach. A *port* (7337) is a numbered door on it.
- **Bearer token:** a secret string in the `Authorization` header that proves access.
- **Host / Origin headers, CORS, DNS rebinding:** request details and browser rules. Checking them stops web pages from using the hub.
- **argv / environment:** a program's arguments and named variables; other local programs can often read both.
- **0600:** a file only you can read and write.
- **Fork / upstream:** our copy of a repo / the original.
- **Seam:** a tiny edit in an upstream file that calls into our own files.
- **Rebase / merge commit / squash-merge:** ways to combine branches. Rebase rewrites history, a merge commit keeps it, and squash turns a PR into one commit.
- **Worktree:** an extra folder of the same repo, on its own branch.
- **PR / CI:** a proposed change / the automatic checks (GitHub Actions) it must pass.
- **OpenSpec change:** a folder with the proposal, design, specs and tasks for one feature.
- **Electron:** desktop apps built with web tech. The *main process* runs Node, the *renderer* draws the UI, *preload* and *IPC* connect them, and a *utilityProcess* is an extra background process.
- **PTY:** the pseudo-terminal each CLI runs in.
- **N-API:** a stable interface for compiled Node modules, so they needn't be rebuilt for each Node version.
- **AppImage / rpm / electron-builder:** a portable Linux app / Fedora's package format / the tool that builds both.
- **Electron fuses:** build-time switches that turn off risky Electron features.
- **sha256:** a file fingerprint; a mismatch means the file changed.
- **systemd user service:** a background program started at login.
- **Handoff / rate limit:** passing a task to another CLI / a subscription's usage cap.

