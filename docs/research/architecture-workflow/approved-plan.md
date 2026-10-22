# Plan: reuse-first architecture for `interface`, written into the repo

> **Status: paused at your request, before any writing.**
> - So far, I've created the local branch `docs/reuse-first-architecture` from `main`. It has no commits and nothing is pushed.
> - I've read the files to be revised (`add-mcp-hub` proposal, design, spec and tasks; README; research notes).
> - Nothing has been written yet. When you say "execute", I'll start at step 1 of "What gets written after approval" (`docs/architecture.md`).

## Context

You read the prior-art research (`docs/research/2026-10-prior-art.md`), which showed that most pieces of the idea already exist as open-source projects. You asked to **redesign the architecture around reusing other people's public code and ideas**, and to write "the whole approach to how this will be built" into the repo, where you can read it at your own pace. You also decided that **our own desktop app is the core of the project**. Earlier decisions still hold: free, MIT, public repo; friends contribute; GitHub Actions CI with a worktree -> PR -> squash-merge loop; spec-driven with OpenSpec.

How the design was made: a workflow of 10 agents did four things:
- It checked 29 facts. 15 turned out wrong or only partly true.
- It wrote 3 competing designs: thin fork, core-first and max assembly.
- 3 judges scored them, and the winner was merged with the best ideas from the other two.
- A critic attacked the result and raised 37 issues: 17 major, none blocking.

The plan below is that merged design with the critic's fixes applied.

Facts that changed the design:
- **Parallel Code doesn't pass MCP config to every agent.** It only does so for Claude (its own "canvas" server, on every fresh start), Codex and Copilot. OpenCode and custom agents get nothing. So we add our own step inside `spawnAgent()` in `electron/ipc/pty.ts`, the one path every agent start goes through.
- **All three CLIs can take an MCP server when they start:**
  - Claude: `--mcp-config`
  - OpenCode: `OPENCODE_CONFIG_CONTENT` together with `--standalone`
  - Muse: `-c`
  This hasn't been tested end to end yet, so stage S0 tests it first.
- **Your CLIs already have your keys in plaintext.** Your Claude Code, OpenCode and Muse configs already contain Firecrawl, Parallel, Context7 and Octen entries, with keys in plaintext headers. The critic read names only, not values. Interface would add duplicates unless it imports those keys and turns the old entries off.
- **The fork and packaging plan had several gaps:**
  - GitHub won't let a bot push upstream's workflow-file changes.
  - Upstream CI only runs on `main`.
  - The AppImage needs `libfuse.so.2`, which is missing on this machine (checked).
  - rpm installs never auto-update.
  - Parallel Code bundles a proprietary Anthropic SDK.

## Recommended architecture

### In plain words
- **Our app is "Interface", a rebranded fork of Parallel Code (MIT).**
  - Its terminals, worktrees, diff review and usage bar are reused as they are.
  - We change about 25 lines of its code. The budget is at most 12 files and 60 lines; everything new lives in our own files.
- **The hub runs inside the app.**
  - Keys live in the GNOME keyring.
  - Each agent the app starts gets the hub's tools, plus its own short-lived token (a one-time pass).
  - No CLI config file is edited. The only exceptions are things you opt into: importing your existing entries (S2), and global sync and vendor skills (S3).
- **Our code is plain TypeScript in this repo** (`packages/core`, `packages/hub`). Friends can work on it without building the desktop app. The fork pins it by commit.
- **First come 1-3 days of experiments (S0).** They prove each CLI accepts the hub at launch. Then the app ships in small stages.

```
+----------------- Interface app (a fork of Parallel Code) ------------------+
| reused as-is: tasks, git worktrees, agent terminals, diff review           |
| ours:         Muse preset, Hub & Keys window, one launch seam              |
+-----------+---------------------------------------+------------------------+
            | starts each agent with its own token  | runs in the background
            v                                       v
  claude | opencode | muse ---- token ------> hub on 127.0.0.1:7337
  (your own subscriptions)                     |            |
                                   secret-tool |            +--> Parallel
                                               v            +--> Firecrawl
                                         GNOME Keyring      +--> Octen
                                                            +--> Context7
```

### What we reuse, what we write

| Piece | Reused (license, how) | We write |
|---|---|---|
| Desktop app | Parallel Code (MIT, fork base) | Muse preset (idea from Orca, MIT), seams, build config, fork CI |
| Hub | MCP TypeScript SDK v2 (Apache-2.0, dependency); 1MCP tool naming (idea only) | Aggregator, naming rule, guards, health checks |
| Keys | `secret-tool` from Fedora's libsecret (a separate program) | `SecretStore` + an in-memory store for tests |
| Global sync (S3, opt-in) | jsonc-parser (MIT), `claude mcp`, rulesync's Muse field mapping (MIT, ported with its notice) | Backups, atomic writes, unsync |
| Provider CLIs (S3) | `parallel-cli`, `firecrawl-cli` (pinned versions) | `ifc run`, PATH shims |
| Handoff (S4) | Parallel Code's handoff composer; `muse export`, `opencode session export`; Waybill format (idea) | Transcript readers, brief packer |
| Limits (S5) | Parallel Code's usage readers + Claude's StopFailure hook; cezar `usage-limit.ts` (MIT, vendored with its notice) | Limit watcher, "waiting for usage" state |

Not used:
- add-mcp: non-atomic writes and no Muse support.
- 1MCP and mcp-proxy at runtime.
- CodexBar.
- cli-continues is optional only, because it has had no commits since May.

No GPL, AGPL, BUSL or Elastic-licensed code.

### Key design rules (critic's fixes included)

**Repos and fork**
- **Two repos:**
  - This one holds the plans and our packages.
  - `siddharth-0044/interface-desktop` is the GitHub fork. Its `main` branch mirrors upstream; the default branch `ifc` carries our files.
- **Fork CI.** On PRs to `ifc` it runs:
  - upstream's own npm checks;
  - `check-seams`, which enforces the seam budget and an allowlist of upstream workflows;
  - a release guard.
- **Upstream sync:**
  - One local script runs `gh repo sync` and opens a PR that's merged with a merge commit.
  - A weekly reminder issue says when to run it.
  - Branches only, no tags; our releases are tagged `desktop-v*`.
  - Fork scripts always pass `--repo siddharth-0044/interface-desktop --base ifc`, so PRs never go upstream by accident.
- **When to hard-fork (stop syncing):** upstream changes its license, stops for 90 days, or sync needs manual conflict fixes 3 weeks in a row.

**Launch and hub**
- **Starting an agent:**
  - The launch seam in `spawnAgent()` waits up to about 3 seconds for the hub to confirm the agent's token.
  - If the hub isn't ready, the agent starts without hub tools and you see a notice.
  - The app keeps the list of live tokens and re-registers them after a hub restart. A token is revoked when its agent exits, is replaced or fails to start.
- **Per CLI (S0 confirms these):**
  - Claude gets a 0600 file added to the same `--mcp-config` list as upstream's canvas config.
  - OpenCode gets `--standalone` plus `OPENCODE_CONFIG_CONTENT`, with a `{file:}` reference to the token.
  - Muse gets `-c`, matching the root key your settings file already uses. The token comes from `${IFC_HUB_TOKEN}`.
  - We inject only when the CLI's `--help` shows the needed flag (checked once per binary), never based on exact version numbers.
- **Skipped:** plain shells, Docker tasks (for now), unknown CLIs, and CLIs with global sync turned on.
- **The hub:**
  - Fixed port 7337, with no silent port hopping.
  - It connects to all providers at start and answers `tools/list` from a cached catalog.
  - It accepts POST `/mcp` only: GET returns 405 and there's no legacy `/sse`. It speaks both MCP protocol versions the CLIs use.
  - Tool names are `<provider>_<tool>`, unless the tool already starts with the provider's name. Names are kept short so `mcp__ifc__<name>` stays within Claude's 64-character limit.

**Keys and safety**
- **Where keys live:**
  - Only in the Secret Service, through `secret-tool`, with the secret passed on stdin. There's no fallback store.
  - The Keys window can set keys but never show them.
  - Each key goes only to its own provider's host: redirects are refused, and key values are scrubbed from errors and logs.
- **Honest threat model.** Any agent that can run shell commands can read your unlocked keyring.
  - The opt-in "lock keys after loading" keeps keys in their own collection, which the hub locks after loading them. I verified the commands exist: `secret-tool store --collection` and `secret-tool lock`.
  - Core dumps are turned off for the hub, so a crash can't write keys to disk.
- **Billing guard:**
  - Every Claude launch (Docker tasks too) drops `ANTHROPIC_API_KEY`, `ANTHROPIC_AUTH_TOKEN` and `CLAUDE_CODE_USE_*`, unless you opt in per agent.
  - Diagnostics check Claude's settings for `env` and `apiKeyHelper`, and warn about provider keys exported in your shell.
- **Existing entries:**
  - On first run, the app offers "import this key" and "turn off the direct entry", one CLI at a time, with a backup first.
  - "Remove all Interface data" arrives in S2, and `ifc uninstall` in S3.

**Packaging (S1)**
- **Our own identity:** name, version, appId, window class and data folder. That way it runs next to an installed Parallel Code without sharing data.
- **AppImage:** uses the newer runtime that doesn't need libfuse2, and updates itself.
- **rpm:** depends on Electron's defaults plus libsecret and nodejs. It's upgraded with `dnf` until a COPR repository exists.
- **Electron fuses are on.**
- **Anthropic's Agent SDK is left out,** so upstream's chat view is off and shows "not available in this build".
- **Docs and notices:** README and PRIVACY are replaced in the fork. THIRD_PARTY_NOTICES goes in this repo, and a NOTICE file in the fork.

### Roadmap

| Stage | You can use | Done when | OpenSpec change(s) | Time* |
|---|---|---|---|---|
| **S0** spikes | Nothing yet: proof for the design | Each CLI loads a mock hub at launch with our token; results written into `add-mcp-hub/design.md` | `add-mcp-hub` group 0 | 1-3 days |
| **S1** Desktop 0.0.1 | Interface on Fedora (rpm + AppImage): Claude Code, OpenCode, Muse in parallel worktrees | Installs on stock Fedora 44; runs next to Parallel Code with its own data; Muse preset works | `setup-dev-loop`, `fork-desktop-shell` | 1-2 weeks |
| **S2** Desktop 0.1.0, first usable | Paste keys once; every agent you start gets Parallel, Firecrawl, Octen, Context7 | Key-only calls (Firecrawl Bearer, Octen `x-api-key`) work in all 3 CLIs; no stored key value found in configs, app data, runtime files or agent environments | `add-mcp-hub`, `desktop-hub-integration` | 3-4 weeks |
| **S3** 0.2 | `ifc launch <cli>` in any terminal, optional systemd hub, opt-in global sync; `parallel-cli` and `firecrawl` with per-process keys | - | `terminal-and-global-sync`, `provider-cli-keys` | 2-3 weeks |
| **S4** 0.3 | "Continue in..." another CLI, same worktree | - | `cross-cli-handoff` | ~2 weeks |
| **S5** 0.4 | "Waiting for usage", auto-switch, resume at reset | - | `usage-limits-autoswitch` | ~3 weeks |
| Later | Codex, Gemini, Grok | - | `add-more-agent-clis` | - |

*These are guesses for one beginner building with AI agents. S1 and the hub engine (`add-mcp-hub`) can be built at the same time by different people.

## Decisions I made for you (say so at approval if you want any changed)

1. **Thin fork,** not an app from scratch or a 1MCP + mcp-proxy assembly. The maintainability and safety judges both ranked it first.
2. **Our own hub on the MCP SDK,** not 1MCP. This settles the research notes' open question.
3. **Two repos, with the fork pinning our code by commit.** It's the simplest cross-repo loop. Release bundles come only once someone needs them.
4. **Upstream sync uses merge commits, through a local script.** GitHub blocks bots from doing this, and merge commits never break friends' open PRs.
5. **`secret-tool`,** no native keyring module.
6. **The hub lives inside the app.**
   - A systemd service is optional, in S3.
   - Muse never gets global sync; `ifc launch muse` covers it.
7. **Machine-facing name `ifc`.**
   - It's used for the CLI, the MCP key (tools show up as `mcp__ifc__parallel_web_search`), `IFC_*` environment variables, the keyring, folders and the fork branch.
   - "Interface" stays the display name.
8. **Billing guard on by default.**
   - The Agent SDK and chat view are left out.
   - Muse's quota comes only from Muse's own messages (no CodexBar).

## Yours, later (nothing blocks on these)
- **Product name,** before the first public release (S1).
- **Turning on "lock keys after loading":** safer, but you'd get a keyring password prompt each time the app starts.
- **Whether Muse agents auto-trust new worktrees** (`--trust-workspace`), in S1.

## What gets written after approval

Source material for writing (not committed):
- Merged design draft: `/tmp/claude-1000/-home-fedora-Documents-interface/a4378b88-2344-4adc-9e87-6955ac3594f7/scratchpad/wf/synth-doc.md`
- Per-change edit lists and the list of unverified items: `/tmp/claude-1000/-home-fedora-Documents-interface/a4378b88-2344-4adc-9e87-6955ac3594f7/scratchpad/wf/synth-rest.json`
- The critic's 37 issues: `/home/fedora/.claude/projects/-home-fedora-Documents-interface/a4378b88-2344-4adc-9e87-6955ac3594f7/tool-results/buisfk0zk.txt`
- The 29 fact checks: `/tmp/claude-1000/-home-fedora-Documents-interface/a4378b88-2344-4adc-9e87-6955ac3594f7/scratchpad/wf/verification.json`
- Full run journal: `/home/fedora/.claude/projects/-home-fedora-Documents-interface/a4378b88-2344-4adc-9e87-6955ac3594f7/subagents/workflows/wf_54d888c1-87a/journal.jsonl`

1. **`docs/architecture.md`** (new, ASCII only). It's the merged design with every critic fix applied. Sections:
   - TL;DR and the big picture.
   - The reuse table.
   - The repos and how they connect: commit pin, `fetch-core --local`, the sync script, the seam budget, CI in both repos.
   - Starting an agent: the steps, a sequence diagram, a per-CLI table, skip rules and the token handshake.
   - How the hub behaves.
   - Keys and security: threat model, lock option, billing guard, existing entries, uninstall.
   - Packaging.
   - The roadmap, with "done when" for each stage.
   - A map of the OpenSpec changes.
   - Decisions made and decisions still open.
   - Risks of depending on other projects.
   - What isn't verified yet (S0, S1 exit, S2 gate, later), plus the claims that checking proved wrong.
   - Contributing walkthroughs: a first change here, a first cross-repo change, the flow for outside contributors, and the fork's commit-message format.
   - A glossary.
2. **`docs/research/2026-10-prior-art.md`:**
   - Append "## 6. So what will we build?" (about 10 lines plus a link to architecture.md).
   - Tick the items in section 5 that are now decided.
   - Add short corrections: add-mcp is Apache-2.0 and not used; `secret-tool` replaces `@napi-rs/keyring`; 1MCP isn't used at runtime; cli-continues is optional.
3. **`README.md`:**
   - Lead with the desktop app ("based on Parallel Code").
   - Status: "planning; next step S0".
   - "Start here" links to the architecture and the research.
   - A roadmap S0-S5 that links to the changes.
   - Contributing points to S0 and `setup-dev-loop`.
   - ASCII only, so the emoji and ellipses go.
4. **OpenSpec.** I'll get each artifact's format from `openspec instructions <artifact> --change <name> --json`.
   - **Revise `add-mcp-hub`,** following the `openspec-update-change` skill:
     - *proposal:*
       - The hub becomes the app's engine and reaches agents per launch.
       - Fix the JOY claim: JOY ships Linux too.
       - Repo and CI setup moves to `setup-dev-loop`; the global adapters move to the S3 change.
     - *design:*
       - D1: SDK v2 split packages, and both protocol versions.
       - D2: a `secret-tool` SecretStore with separate reader and writer halves (`application=ifc`), and no fallback.
       - D3: `~/.config/ifc` and `$XDG_RUNTIME_DIR/ifc`.
       - D4: the naming rule.
       - D5: per-launch tokens, Host and Origin rules, and a fixed port.
       - D6: the LaunchPlanner for each CLI, with capability checks, skip rules, billing and spoof guards, and a fallback for each CLI if its spike fails.
       - D7: the hub is hosted by whatever starts it; it connects eagerly and caches its catalog.
       - D8: provider-client safety: no redirects, keys bound to their host, scrubbing.
       - Then risks and open questions.
     - *spec:* requirements are rewritten in place. The change is still all ADDED, because nothing is archived yet.
       - Drop "CLIs are pointed at the hub" and "config edits are safe"; both move to S3.
       - The token requirement becomes per-launch, with scenarios for Host, Origin, OPTIONS, `/sse`, and "revoked after the agent exits".
       - Naming gets a length scenario.
       - Add five requirements:
         - "agents get the hub per launch";
         - "keys reach only their provider's host", with a redirect scenario;
         - "keyless providers work with a locked keyring";
         - "Claude launches drop Anthropic API variables";
         - "duplicate direct entries are reported".
       - Diagnostics report, per CLI: version, capability check, and whether injection is on.
     - *tasks:*
       - A new group 0 for the spikes; each writes its result into design.md.
       - The keyring store, plus a CI job against a real `gnome-keyring-daemon` under `dbus-run-session`.
       - The catalog.
       - The hub, with security tests.
       - The LaunchPlanner, with fixtures and a spawn-environment test.
       - Diagnostics, then validate.
       - Old groups 1 and 4 are removed, and so is the `fork-shell-integration` line.
   - **New `setup-dev-loop`** (capability `dev-workflow`). It takes over old tasks 1.2-1.6:
     - npm workspaces (`packages/core`, `packages/hub`), TypeScript, lint, Vitest and dependency-cruiser rules.
     - CI: lint, typecheck, test, build, `openspec validate --strict`, gitleaks and a license allowlist.
     - `scripts/wt-new`, `scripts/ship` and `scripts/wt-done`.
     - CONTRIBUTING.md and THIRD_PARTY_NOTICES.md.
     - Repo settings (squash only, auto-merge, delete branches) and protection for `main`.
     - A test of the whole loop with a trivial PR.
   - **New `fork-desktop-shell`** (capability `desktop-shell`):
     - The fork and its branch model, fork settings, and `ifc/PATCHES.md`.
     - The Muse preset.
     - A generated build config: identity, AppImage runtime, rpm dependencies, fuses, SDK excluded, chat view off.
     - The protocol-scheme decision.
     - Fork CI, check-seams and the release guard.
     - The sync script and its reminder, and fork-aware scripts.
     - Replaced README and PRIVACY files, and a NOTICE file.
     - A check for developer prerequisites.
     - S1 acceptance tests:
       - installs on stock Fedora 44;
       - runs next to Parallel Code;
       - an rpm upgrade from one version to the next keeps your data;
       - AppImage updates work.
     - An upstream issue for the Muse preset, and help with rpm PR #315.
   - **The later changes are described in architecture.md only:** `desktop-hub-integration`, `terminal-and-global-sync`, `provider-cli-keys`, `cross-cli-handoff`, `usage-limits-autoswitch` and `add-more-agent-clis`. Each is created with `/opsx:propose` when its stage starts, after S0, so a failed spike doesn't mean rewriting them.
5. **Git:**
   - Create the branch `docs/reuse-first-architecture` from main.
   - Commit; the message ends with the Co-Authored-By line.
   - Push, then open a PR with `gh pr create`; the body ends with the Claude Code line.
   - Merge with `gh pr merge --squash --delete-branch`. No CI exists yet, so there's nothing to wait for.
   - Back on `main`, run `git pull --ff-only`.

## Verification
1. `openspec validate --strict` passes for `add-mcp-hub`, `setup-dev-loop` and `fork-desktop-shell`.
2. `grep -nP '[^\x00-\x7F]'` prints nothing for `README.md`, `docs/architecture.md` and the three change folders.
3. Every relative link in README.md, architecture.md and the research notes resolves (a grep + `test -e` loop).
4. In each diagram, every box line has the same width (a small awk check). I'll also read the arrows through once.
5. A fresh reviewer agent reads the docs and the three changes. It lists contradictions in names, stages, decisions, per-CLI mechanisms and the "no config edited" exceptions. It also flags any claim stated as fact that the fact checks marked refuted or unverified. I fix what it finds and re-check until it's clean.
6. Each of the critic's 17 major issues maps to a section or task. I tick them off against the critique file while writing.
7. After the merge:
   - `gh pr view --json state` says MERGED.
   - `git log -1 origin/main` shows the squash commit.
   - `git status` is clean.
   - The remote branch is gone.

## Next, after this plan (separate step)
Run S0 with you:
- Build a mock MCP server of about 30 lines that logs the headers it receives.
- Start each CLI against it. Mostly no model calls are needed, because the mock logs the connection, and Muse can use `--provider echo`.
- At the end, make one tiny real tool call per CLI.
