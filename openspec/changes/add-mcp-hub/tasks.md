# Tasks

## 1. Repository, CI and dev loop (first, so everything after goes through PRs)

- [x] 1.1 Run `git init` in the project root, add `.gitignore` (node_modules, dist, coverage), MIT `LICENSE` and a stub `README.md`. Verify with `git status`, which should list only the intended files.
- [ ] 1.2 Scaffold a TypeScript Node package (`package.json` with a `hub` bin, `tsconfig.json`, linter, Vitest). Verify that `npm run lint`, `npm run typecheck`, `npm test` and `npm run build` all pass on an empty `src/index.ts`.
- [ ] 1.3 Add `.github/workflows/ci.yml` (pull_request + push to main: `npm ci` → lint → typecheck → test → build, Node LTS, npm cache). Verify it with `actionlint`, or by its first green run in task 1.5.
- [ ] 1.4 Add `scripts/wt-new`, `scripts/ship` and `scripts/wt-done` (design D8) and document the loop in `CONTRIBUTING.md`. Verify that `shellcheck scripts/*` passes and that `scripts/wt-new demo` creates a worktree on a new branch from `origin/main` (create the repo in 1.5 first).
- [ ] 1.5 After confirming the repo name and visibility with the user, create the GitHub repo with `gh repo create`, push `main`, and turn on repo settings: squash-merge only, auto-merge allowed, delete head branches on merge. Verify by running `gh repo view --json` and checking the merge settings.
- [ ] 1.6 Protect `main` so the CI check is required, and test the full loop with a trivial PR (`wt-new` → change → `ship` → `wt-done`). Verify that the PR is squash-merged automatically after CI passes, the remote branch and local worktree are gone, and local `main` matches `origin/main`.

## 2. Secret store and hub config

- [ ] 2.1 Implement the `SecretStore` interface with an in-memory implementation and a Secret Service implementation (design D2). Verify with unit tests on the in-memory store, and a manual `hub` smoke test on Fedora that stores and reads back a key visible in Seahorse.
- [ ] 2.2 Implement the XDG config file for providers, port and enabled flags (design D3). Verify with unit tests covering file creation, round-trip and a corrupt file producing a clear error.
- [ ] 2.3 Add the built-in provider catalog (Parallel, Firecrawl, Octen, Context7) with endpoint and key-application mode (`header` | `url-template`), confirming each endpoint against its provider's current docs. Verify with a unit test asserting every catalog entry has a URL and a key mode.
- [ ] 2.4 Implement `hub provider add|remove|list`, including custom URL and command providers and a hidden key prompt. Verify with tests that cover these spec scenarios: add from catalog, add custom, remove provider (key deleted), status masks keys, and secret store unavailable.

## 3. Hub server

- [ ] 3.1 Implement per-provider MCP clients (HTTP and stdio) that attach credentials as the catalog defines. Verify with tests against a local mock MCP server that check the Authorization header or URL key arrives as expected.
- [ ] 3.2 Implement the aggregating Streamable HTTP server on 127.0.0.1 with always-prefixed tool names and call routing (design D4). Verify with tests for the spec scenarios "Aggregated tool list", "Tool call routing" and "Two providers expose web_search".
- [ ] 3.3 Add bearer-token auth with the token kept in the secret store, plus `hub token rotate` (design D5). Verify with tests for "Request without token" being rejected and a valid token being accepted.
- [ ] 3.4 Add provider health tracking with backoff, so a failing provider is left out of the tool list and reported (design D7). Verify with a test for the "Provider with an invalid key" scenario.
- [ ] 3.5 Implement `hub start` (foreground) and `hub service install|uninstall` (systemd user unit with `Restart=on-failure`). Verify that `systemctl --user status hub` shows it active after install, and add the service section to the README.
- [ ] 3.6 Add an integration job to CI that starts the hub with two mock providers and runs an MCP client against it. Verify that the job passes in a PR.

## 4. CLI adapters and sync

- [ ] 4.1 Define the adapter interface and the shared safe-edit helper (parse, change only `hub`, back up before the first write, write atomically, skip no-op writes). Verify with unit tests for "Existing entries preserved", "Idempotent sync" and "Unparseable config".
- [ ] 4.2 Implement the Claude Code adapter using `claude mcp add/remove --scope user`, with a JSON fallback. Verify with tests on a stubbed `claude` binary, and a manual run where `claude mcp list` shows `hub` connected.
- [ ] 4.3 Implement the OpenCode adapter with JSONC-preserving edits under `mcp.hub`. Verify with fixture tests (before → after, comments preserved), and manually that OpenCode lists the hub's tools.
- [ ] 4.4 Implement the Muse Code adapter for `~/.config/muse/settings.json` under `mcpServers.hub`. Verify with fixture tests, and manually that Muse lists the hub's tools.
- [ ] 4.5 Implement `hub sync` and `hub unsync` across detected CLIs, reporting each CLI that isn't installed as "not found". Verify with tests for "Sync with all three installed", "CLI not installed" and "Unsync".
- [ ] 4.6 Document adding a new adapter in `CONTRIBUTING.md`. Verify the doc's steps against a throwaway adapter in a test.

## 5. Diagnostics and docs

- [ ] 5.1 Implement `hub status` and `hub doctor` with per-provider and per-CLI state and suggested fixes. Verify with a test for "Hub not running", and check that no key value appears in output (snapshot test).
- [ ] 5.2 Write the README: install, quickstart (add keys → start → sync), a security note (loopback, token, keyring, Muse Code's data-sharing default) and uninstall (`hub unsync`, `hub service uninstall`). Verify by following the quickstart on a clean user account on Fedora.

## 6. End-to-end check

- [ ] 6.1 On Fedora with Claude Code, OpenCode and Muse Code installed and real Parallel and Context7 keys: add providers, start the hub, sync, and call one Parallel tool and one Context7 tool from each CLI. Verify that all six calls succeed and `hub doctor` reports everything healthy.
- [ ] 6.2 Run `openspec validate add-mcp-hub --strict`. Verify that it passes.

## Workflow follow-up

- Archive this change with `/opsx:archive add-mcp-hub` once it is merged and the end-to-end check passes.
- Start the next change (`fork-shell-integration`: Parallel Code fork with OpenCode/Muse presets, hub settings panel, manual "Continue in…" handoff) with `/opsx:explore`.
