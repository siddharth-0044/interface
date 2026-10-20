# Is this idea new? What can we reuse?

*Research notes from 2026-10-20. Written to be read slowly, with coffee.*

---

## TL;DR

- **Most of the idea already exists, spread across 20+ open-source projects.** That's good news: we can borrow instead of building from scratch.
- **What seems genuinely new is our hub:** your API keys stored once in the Linux keyring, served as **one** protected MCP entry that's written automatically into Claude Code, OpenCode **and Muse Code**, with no Docker needed.
- **The other new thing is the bundle:** a free, MIT-licensed version of what JOY sells for $14.99/month.
- **One assumption was wrong:** JOY is **not macOS-only any more**. It now ships Linux builds, including a Fedora `.rpm`. It's still paid and closed source.
- **CLIs matter too:** some Parallel features (FindAll, Monitor) are only in its CLI, not its MCP. The hub should eventually feed keys to CLIs safely, not just to MCP.

> **Caveat:** the step where agents actively tried to prove "this already exists" never finished. "The hub looks new" means nobody *found* a match, not that someone tried hard to disprove it. That check can be re-run later.

---

## 1. Do we need the Firecrawl and Parallel CLIs?

Short answer: **partly, yes.**

| Provider | What MCP gives you | What only the CLI/API gives you |
|---|---|---|
| **Parallel** | Search MCP (`web_search`, `web_fetch`), plus a separate Task MCP (deep research, task groups) | **FindAll** (find lists of companies or people) and **Monitor** (watch the web for changes) |
| **Firecrawl** | Everything, **as long as the API key is sent**. Without a key it quietly drops to just 3 tools | Mostly the same features, plus "skills" files that teach agents how to use the CLI |

Parallel even *recommends* its CLI (`parallel-cli`) plus skills for coding agents, because a CLI call uses fewer tokens than loading MCP tool descriptions.

**What this means for us:** keep the MCP hub, and later add a safe way to hand keys to CLIs:

```
                 +----------- keyring (keys stored once) -----------+
                 |                                                  |
                 v                                                  v
   hub on 127.0.0.1 (MCP)                           hub run parallel-cli findall ...
   used by claude / opencode / muse                 (key goes to that ONE command only)
```

Why not just `export PARALLEL_API_KEY=...` in your shell? Because then every program you run can read it. The research also found that a stray `ANTHROPIC_API_KEY` in your environment silently switches Claude Code from your subscription to pay-per-use API billing. That's an expensive surprise.

Neither CLI is installed on this machine yet (`claude`, `opencode` and `muse` are).

Links: [Parallel CLI](https://docs.parallel.ai/integrations/cli), [Parallel Task MCP](https://docs.parallel.ai/integrations/mcp/task-mcp), [Firecrawl CLI](https://docs.firecrawl.dev/sdks/cli)

---

## 2. Piece by piece: new or not?

| Piece of our idea | New? | Best thing to reuse |
|---|---|---|
| Desktop app that runs many coding CLIs | **No**, crowded (8+ open-source apps run on Linux) | [Parallel Code](https://github.com/johannesjo/parallel-code) (MIT) is still the best base. It **already has OpenCode**, so only Muse needs adding |
| "Continue this task in another CLI" | **No** | [cli-continues](https://github.com/yigitkonur/cli-continues) (MIT, TypeScript) reads the chat history of 16 CLIs (not Muse) |
| Spotting "out of usage", switching, resuming | **No** | [cezar](https://github.com/open-mercato/cezar) (MIT) already detects limits and auto-resumes |
| Combining many MCP servers behind one address | **No** | [1MCP](https://github.com/1mcp-app/agent) (TypeScript, Apache-2.0) does about 70% of our hub, but keeps keys in plain text |
| Writing an entry into each CLI's config | **Mostly no** | [add-mcp](https://github.com/neon-solutions/add-mcp) covers Claude Code + OpenCode. [rulesync](https://github.com/dyoshikawa/rulesync) has the only TypeScript Muse writer found |
| **Keyring + one protected hub entry auto-written into Claude, OpenCode and Muse, no Docker** | **Yes, looks new** | Nobody does all of this together. Closest: ToolHive and Docker MCP Gateway (both need containers) |

---

## 3. The shopping list: what to borrow

### For the hub (stage 1, what we're building first)
- **[MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)**: the official building blocks. Note that it does **not** include a "combine many servers" feature; we write that part.
- **[1MCP](https://github.com/1mcp-app/agent)** (Apache-2.0): tool-naming and collision code. We can copy it if we keep their license notice.
- **[MetaMCP](https://github.com/metatool-ai/metamcp)** (MIT): a working "fan out to many servers" proxy on the same SDK.
- **[punkpeye/mcp-proxy](https://github.com/punkpeye/mcp-proxy)** (MIT): serving old and new MCP clients from one address.
- **[@napi-rs/keyring](https://github.com/Brooooooklyn/keyring-node)** (MIT): talks to GNOME Keyring. Actively maintained.
- **[ToolHive](https://github.com/stacklok/toolhive)** (Go, so ideas only): the safest way to edit config files (lock the file, keep comments, write atomically).
- **[ai-memory](https://github.com/akitaonrails/ai-memory)**: the closest overall design to ours (a local server with a token, wired into 20+ CLIs, backups and clean uninstall).

### For the desktop app (stage 2)
- **[Parallel Code](https://github.com/johannesjo/parallel-code)** (MIT): the fork base. Agent presets live in one list (`electron/ipc/agents.ts`).
- **[Orca](https://github.com/stablyai/orca)** (MIT): already has a Muse preset to copy.
- **[emdash](https://github.com/generalaction/emdash)** (Apache-2.0): good per-CLI config code.

### For handoff and limits (stage 3)
- **[cli-continues](https://github.com/yigitkonur/cli-continues)** (MIT, TS): chat-history readers for 16 CLIs.
- **[session-migrate](https://github.com/xhluca/session-migrate)**: the only place that documents **Muse's** history format.
- **[Waybill](https://github.com/wardmos/waybill)** (Apache-2.0): a ready-made format for handoff notes, so we don't invent one.
- **[cezar](https://github.com/open-mercato/cezar)** (MIT): usage-limit detection and auto-resume logic.
- **Claude Code's own signals**: the [statusline](https://code.claude.com/docs/en/statusline) reports `rate_limits`, and a `StopFailure` hook fires when a limit is hit. That's better than scraping terminal text.
- **[CodexBar](https://github.com/steipete/CodexBar)**: reads usage quotas for many CLIs, including Muse.

### Don't copy code from these (license problems for an MIT project)
- **GPL/AGPL:** Claude Squad, McpMux, Magg, Agent Factory
- **"Source available", not open source:** Superset (Elastic), nodeterm and Golutra (BUSL)
- **Closed:** JOY, Conductor, JetBrains Air

Ideas are fine; code isn't.

---

## 4. Small things to fix in our design

These came out of the research. None are applied yet.

1. **Keyring safety.** Tell the keyring library explicitly to use the desktop keyring. Otherwise, if the keyring is locked or missing, it silently saves keys somewhere that's **wiped on reboot**.
2. **Octen uses a different header.** It expects `x-api-key`, not `Authorization: Bearer`, so the hub must allow any header name.
3. **Parallel has two addresses** (Search and Task) that share one key. The Search one even works without a key.
4. **Firecrawl without a key** quietly drops to 3 tools. The hub should warn instead of letting that happen silently.
5. **Muse needs `mode: optional`** on our entry. Otherwise Muse refuses to start whenever the hub is down.
6. **Web-page attack protection.** The hub should check the `Origin` header, so a malicious website can't reach it through your browser. Docker's gateway does this.
7. **Duplicate tools.** If someone already added Context7 directly, our adapter should notice and offer to remove the duplicate.
8. **Update the proposal**: it still says JOY is "macOS-only".

---

## 5. What's next (when you feel like it)

- [ ] Re-run the "try to prove it already exists" check (3 agents; the finished research is cached).
- [ ] Apply the design fixes in section 4 to `openspec/changes/add-mcp-hub/`.
- [ ] Add a requirement and task for `hub run <cli>` (handing keys safely to provider CLIs).
- [ ] Decide: build our own hub, or build on 1MCP and contribute keyring support upstream.
- [ ] Then start building with `/opsx:apply add-mcp-hub`.
