# CONTEXT: where we left off

*Written 2026-10-22 so a brand-new session can pick up the work. Read this first, then `docs/research/architecture-workflow/approved-plan.md`.*

## 1. The idea
A free, open-source (MIT) desktop workspace for AI coding CLIs on Linux (Fedora x86_64 first). It is like the paid, closed JOY app (enjoy.dev), which now ships Linux builds too, including a Fedora rpm.

- It runs Claude Code, OpenCode and Muse Code side by side. Each uses the user's own subscription. Codex, Gemini and Grok come later.
- The twist is **one local MCP hub**. The user's paid provider keys (Firecrawl, Parallel, Octen, Context7) are stored once in the desktop keyring (Secret Service). Every CLI the app starts gets those tools.
- Later: "Continue in..." another CLI when one runs out of usage, then limit detection, auto-switch and resume.

The owner is terminal-friendly but likes GUI session/worktree apps. They are not a senior developer and build with AI coding agents. Explain things plainly, keep answers short, and write long material into repo docs instead of chat.

## 2. Decisions the owner made
- Free and open source, bring-your-own-keys. Public repo: https://github.com/siddharth-0044/interface (MIT).
- Friends will contribute. GitHub Actions CI, plus a scripted worktree -> PR -> squash-merge -> cleanup flow (never typed by hand).
- **Our own desktop app is the core of the project.**
- They asked for a reuse-first architecture (reuse other people's public code and ideas), written into the repo to read at their own pace.
- They approved the plan in `docs/research/architecture-workflow/approved-plan.md`, then said "pause until I say execute" and "wait". **The docs from that plan are NOT written yet.**

## 3. What is in the repo now
- `README.md`, `LICENSE`, `.gitignore`.
- `docs/research/2026-10-prior-art.md`: the "is this new, what can we reuse" notes. It still says some things the new plan replaces (e.g. `@napi-rs/keyring`, "build on 1MCP?").
- `openspec/changes/add-mcp-hub/`: the OLD hub plan (terminal-first, writes global CLI configs). It needs revising; see section 6.
- `docs/research/architecture-workflow/`: raw outputs of the architecture workflow (10 agents), copied here so they are not lost:
  - `approved-plan.md`: the approved plan, including everything to write next.
  - `merged-design-draft.md`: the winning design as a full document. It needs the critic's fixes applied.
  - `critic-issues.txt`: the critic's 37 issues (17 major, none blocking), each with a fix.
  - `fact-checks.json`: 29 fact checks (15 refuted or partly true).
  - `judges.json`, `synthesis-extras.json`: judge scores, per-change edit lists, decisions and unverified items.
- Git: `main` has 49e1c08 "Plan the MCP hub and set up the repo" and 13ea0db "Add prior-art research notes". This file and the folder above were added after those.

## 4. The architecture in one screen
- **Interface** is a thin, rebranded fork of Parallel Code (https://github.com/johannesjo/parallel-code, MIT, Electron + SolidJS + TypeScript). The fork is `siddharth-0044/interface-desktop` and does not exist yet. About 25 upstream lines change, under a budget of at most 12 files and 60 lines.
- Our own code is plain TypeScript in this repo (`packages/core`, `packages/hub`). The fork pins it by commit.
- The hub runs inside the app and listens on 127.0.0.1:7337. It uses the MCP TypeScript SDK v2, and its tool names look like `<provider>_<tool>`.
- Keys live only in the Secret Service via `secret-tool` (key on stdin, no fallback store).
- Each agent the app starts gets its own short-lived token. No CLI config file is edited, except for opt-in extras.
- Per CLI, injection is meant to work like this (**not yet tested; that is stage S0**):
  - Claude Code: `--mcp-config <0600 file>`, added to the list Parallel Code already uses.
  - OpenCode: `--standalone` plus the `OPENCODE_CONFIG_CONTENT` env var (v2 shape `mcp.servers.<name>`).
  - Muse: `-c mcpServers.ifc=<JSON>`, with the token as a whole-value `${IFC_HUB_TOKEN}`.
- The machine-facing name is `ifc`: CLI, MCP key, `IFC_*` env vars, keyring attribute, folders and the fork's default branch. "Interface" is only the display name.
- Stages: S0 spikes (1-3 days), S1 fork runs as our app, S2 hub in the app (first usable release), S3 terminal use and provider CLIs, S4 "Continue in...", S5 limits and auto-switch.

## 5. Facts that matter (verified in this session unless noted)
- Parallel Code passes `--mcp-config` only to Claude (its own canvas server on every fresh start), Codex and Copilot. OpenCode and custom agents get nothing. Our hook goes in `spawnAgent()` in `electron/ipc/pty.ts`, right after `applyAgentHookLaunch`.
- Parallel Code has no Muse preset. It has OpenCode. Its AppImage/deb build exists; the rpm PR is #315 and is still open.
- Upstream CI only runs on `main`. GitHub blocks bots from pushing workflow-file changes, so upstream sync is a local script (`gh repo sync` plus a merge-commit PR).
- The AppImage needs `libfuse.so.2`; this machine has only fuse3. Use the static AppImage runtime.
- rpm installs never auto-update in upstream's updater.
- Parallel Code bundles the proprietary `@anthropic-ai/claude-agent-sdk`. The plan excludes it and turns the chat view off.
- The owner's Claude Code, OpenCode and Muse configs already have Firecrawl, Parallel, Context7 and Octen entries with plaintext Authorization headers. The app must offer to import the keys and turn those entries off, or tools appear twice. (Only names and field shapes were read, never values.)
- A stray `ANTHROPIC_API_KEY` in the environment switches Claude Code to pay-per-use billing. The billing guard strips it from Claude launches.
- Provider facts:
  - Parallel Search MCP: `https://search.parallel.ai/mcp` (works without a key).
  - Parallel Task MCP: `https://task-mcp.parallel.ai/mcp`.
  - Parallel FindAll and Monitor exist only in `parallel-cli` or the SDK.
  - Firecrawl: `https://mcp.firecrawl.dev/v2/mcp` with a Bearer key. Without a key it drops to 3 tools.
  - Octen: `https://mcp.octen.ai/mcp`, header `x-api-key`.
  - Context7: `https://mcp.context7.com/mcp`, optional Bearer.
- Claude Code limits the full tool name `mcp__<server>__<tool>` to 64 characters, so hub-side names must stay short.
- `secret-tool` has `store --collection`, `lock` and `search --unlock` (checked locally). Any agent that can run shell commands can read an unlocked keyring; this is stated honestly in the plan.
- Installed here: `claude` 2.1.304, `opencode` 2.0.32, `muse` 1.4.11. Not installed: codex, firecrawl, parallel-cli, Go. Node and `gh` (logged in as siddharth-0044) are available.

## 6. What to do next (in order)
1. Tell the session "execute" only if the owner says so. Then do the plan's "What gets written after approval":
   1. Write `docs/architecture.md` from `merged-design-draft.md`, applying every issue in `critic-issues.txt` (ASCII only).
   2. Append "## 6. So what will we build?" to the prior-art notes and add the corrections listed in the plan.
   3. Rewrite `README.md` (desktop app first, roadmap S0-S5, ASCII only).
   4. Revise the `add-mcp-hub` change (use the `openspec-update-change` skill). Create the new `setup-dev-loop` and `fork-desktop-shell` changes. Run `openspec validate --strict`.
   5. Run the verification list in the plan, including a fresh reviewer agent for contradictions.
   6. Branch, commit, PR and squash-merge.
2. Then run **S0**: a ~30-line mock MCP server that logs headers, and a test of each CLI against it (details in the plan and in `merged-design-draft.md` section 11).
3. Pending repo setup: squash-only merges, auto-merge, delete branches on merge, and branch protection (old tasks 1.5 and 1.6). Friends' GitHub usernames were never given.

## 7. Open choices (nothing blocks)
Product name before the first public release; "lock keys after loading" (a keyring prompt on every app start); whether Muse agents auto-trust new worktrees (`--trust-workspace`).

## 8. Working agreements
- Plain, short language. Long answers go into repo docs.
- Never put keys in files, argv, logs or a global environment. No code from GPL, AGPL, BUSL, Elastic or proprietary projects (JOY, Conductor, Superset, Claude Squad, McpMux and similar).
- Confirm before outward-facing actions. Commits end with a Co-Authored-By line; PR bodies end with the Claude Code line.
- Diagrams are ASCII only.

## 9. Caveats
- The "try to prove it already exists" check on the hub idea never finished, so "looks new" only means nobody found a match.
- Effort estimates are guesses.
- Several per-CLI mechanisms are unverified until S0 (the list is in `merged-design-draft.md` section 11).
