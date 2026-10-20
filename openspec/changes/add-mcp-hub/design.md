# Design

## Context

See proposal.md (Why) for motivation and specs/mcp-hub/spec.md for the behavior contract.

Constraints that shape the approach:
- Target is Fedora on x86_64 first. The main user works terminal-first, so v1 has no GUI.
- Node is installed on the dev machine and Go is not. The planned shell (a later change) is a fork of Parallel Code, which is Electron + SolidJS + TypeScript.
- Most catalog providers are already **remote** MCP servers over HTTP (for example Parallel at `https://search.parallel.ai/mcp` with a Bearer key). So the hub is mostly a credential-attaching router, not a process supervisor.
- MCP tools connected through a claude.ai account (connectors) are tied to Claude and can't be shared. The hub only handles providers the user has their own key for.
- The CLIs store MCP config in different files and formats: `~/.claude.json` (managed by `claude mcp`), OpenCode's `opencode.json` (JSON with comments allowed), and Muse Code's `~/.config/muse/settings.json` (`mcpServers`).

## Goals / Non-Goals

**Goals:**
- A single `hub` CLI with subcommands: `provider add|remove|list`, `start`, `sync`, `unsync`, `status`, `doctor`.
- An adapter boundary small enough that adding a new CLI (Codex, Gemini, Grok…) is one file plus tests.
- Every change reaches `main` through a PR that GitHub Actions has checked.

**Non-Goals:**
- No GUI, tray icon or desktop notifications.
- No OAuth flows for providers. API keys only in v1.
- No Windows or macOS support work. Nothing should block it, but nothing is tested there.
- No auto-update or distro packaging. Install is `npm i -g` from the repo or registry.

## Decisions

### D1. TypeScript on Node, not a Go `mcp-proxy` sidecar
Build the hub with the official MCP TypeScript SDK, using its server side (Streamable HTTP) and its client side (one client per provider).
- *Why:* there's one toolchain, and it's already installed. It's the same language as the Parallel Code fork, so the shell can later import the hub as a library instead of spawning a foreign binary. CI also needs only one language setup.
- *Alternative:* bundle `tbxark/mcp-proxy` (a Go single binary) and generate its `config.json`. It's less code, but it adds a second toolchain and a binary we'd need to ship for each architecture. Keyring integration and tool name prefixing would then live outside our code. Keep it as a fallback if the SDK's proxying turns out to be weak.

### D2. Keys live in the Secret Service (GNOME Keyring / KWallet) through a native binding
Use a maintained Node Secret Service binding (candidate: `@napi-rs/keyring`), with service name `hub` and account set to the provider id.
- *Why:* it satisfies the "never plaintext" requirement without shelling out, and it works the same on GNOME and KDE.
- *Alternative:* shell out to `secret-tool` (libsecret-tools). It has zero native dependencies, but users may not have it installed and quoting is brittle. If a desktop has no Secret Service, fail clearly (that's what the spec requires). No silent fallback to an encrypted file.
- Code talks to an internal `SecretStore` interface. Tests and CI use an in-memory implementation, because GitHub's Ubuntu runners have no keyring daemon.

### D3. Non-secret state in XDG config
Provider list, port and enabled flags go in `~/.config/hub/config.json`. The local access token (see D5) and provider keys go in the keyring. Backups of CLI configs go in `~/.local/state/hub/backups/<cli>/<timestamp>`.

### D4. Always-prefixed tool names
Expose every tool as `<provider>_<tool>` (for example `parallel_web_search`), whether or not it collides with another provider's tool.
- *Why:* names stay stable when a provider is added later. With collision-only prefixing, a tool would get renamed under the agent's feet the day a second provider adds a tool with the same name.
- *Trade-off:* names get longer. Fine for agents.

### D5. Loopback bind plus a bearer token
Bind to `127.0.0.1` on a configurable port (default `7337`). Generate a random token on first run, keep it in the keyring, and write it into each synced CLI entry as an `Authorization: Bearer` header.
- *Why:* loopback stops other machines, and the token stops any random local process or browser page (DNS rebinding) from using the user's paid credits.
- *Trade-off:* the token has to sit in CLI config files. It only grants access to a loopback port, and `hub token rotate` plus a re-sync replaces it.

### D6. Adapters
Each adapter implements `detect()`, `read()`, `upsertHubEntry(url, token)`, `removeHubEntry()`.
- **Claude Code:** prefer the CLI itself (`claude mcp add --scope user --transport http hub <url> --header ...` and `claude mcp remove hub --scope user`), so `~/.claude.json` is never hand-edited while Claude may be writing it. Fall back to a JSON edit only if the command isn't available.
- **OpenCode:** edit its user config through a JSONC-aware editor that keeps comments and formatting, under `mcp.hub` with `type: "remote"`.
- **Muse Code:** edit `~/.config/muse/settings.json` under `mcpServers.hub`.
- Every file-editing adapter: parse, change only `hub`, back up before the first write, write to a temp file and rename it into place (an atomic write), and compare against the current content to skip no-op writes (that's how idempotence is met).
- Each adapter's exact entry shape is pinned in fixture tests (`fixtures/<cli>/before.json` → `after.json`).

### D7. Running the hub
`hub start` runs in the foreground, which is terminal-native and works fine in a tmux pane. `hub service install` writes a systemd `--user` unit so the hub starts at login. Providers are connected lazily and retried with backoff. A failing provider is marked unhealthy and left out of the tool list (that's how the "one failing provider" requirement is met).

### D8. Repository, CI and the worktree → PR loop
- **Hosting:** a public GitHub repository under the user's account, MIT licensed.
- **CI** (`.github/workflows/ci.yml`): runs on `pull_request` and on `push` to `main`, on `ubuntu-latest` with Node LTS and an npm cache. The steps are `npm ci` → lint → typecheck → unit tests → build. A separate integration job starts the hub against a mock MCP server and checks the aggregated tool list and call routing.
- **Branch protection on `main`:** the CI check is required, squash-merge only, auto-merge allowed, and head branches are deleted automatically after merge.
- **Dev loop scripts** (so nothing gets typed by hand):
  - `scripts/wt-new <name>`: `git worktree add ../<repo>-wt/<name> -b <name> origin/main`
  - `scripts/ship`: push, then `gh pr create --fill`, then `gh pr merge --squash --auto --delete-branch`
  - `scripts/wt-done <name>`: wait for the merge, `git worktree remove`, delete the local branch, `git -C <main checkout> pull --ff-only`
- The same flow is written down in `CONTRIBUTING.md` so Claude Code, OpenCode or Muse sessions can run it too.
- *Alternative considered:* a merge queue or release-please. Too much ceremony for a one-person repo right now.

## Risks / Trade-offs

- [The CLIs change their config formats] → Pin the expected shapes in fixture tests. Make `doctor` report "unknown format" instead of guessing. Each adapter is isolated, so one break doesn't affect the others.
- [Editing a config while the CLI is running] → Use the CLI's own command where one exists (Claude Code). Otherwise do atomic rename writes and tell the user to restart that CLI.
- [Muse Code reads Claude/Codex files and sends some of them to Meta by default (reported)] → Document this in the README. The hub token is local-only, but warn anyway.
- [A native keyring module fails to build on some machines] → Ship prebuilt binaries through the chosen package. If install keeps failing, switch to the `secret-tool` route (D2 alternative).
- [Some providers' remote MCP needs the key in the URL path instead of a header] → The catalog entry declares how the key is applied (`header` | `url-template`), and only the hub ever builds that URL.
- [A single point of failure: the hub down means no tools in any CLI] → `doctor` plus the systemd unit with `Restart=on-failure`. CLIs keep working without the tools.

## Migration Plan

New project, so there's nothing to migrate. Rollback for users: `hub unsync` removes every `hub` entry, and the backups in `~/.local/state/hub/backups` can restore the originals.

## Open Questions

- The exact remote endpoints and how the key is applied for Firecrawl, Octen and Context7. This is catalog data that gets verified during implementation and doesn't change the specs or tasks.
- The final project or package name (`hub` is a working name).
- Exact entry shapes for OpenCode `remote` and Muse `http` servers, to confirm against current docs when writing the fixtures.
