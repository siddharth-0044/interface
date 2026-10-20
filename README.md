# interface

A free, open-source workspace for AI coding CLIs on Linux (Fedora first).

Use Claude Code, OpenCode and Muse Code side by side, each on your own subscription. They all share **one local MCP hub**, so the tools you pay for (Firecrawl, Parallel, Octen, Context7…) are set up once and work in every CLI. When one CLI runs out of usage, you carry on in another with the same tools.

> **Status: planning.** There's no code yet. The first piece is fully specified and ready to build.
>
> 📖 **Is this idea new, and what can we reuse?** See the [prior-art research notes](docs/research/2026-10-prior-art.md).

## Roadmap

| Stage | What | State |
|---|---|---|
| 1 | **MCP hub**: a terminal tool that keeps your API keys in the desktop keyring, serves one aggregated MCP endpoint on `127.0.0.1`, and points each CLI's config at it | Planned. See [`add-mcp-hub`](openspec/changes/add-mcp-hub/) |
| 2 | **Desktop shell**: a fork of [Parallel Code](https://github.com/johannesjo/parallel-code) with OpenCode/Muse presets, a hub settings panel, and a manual "Continue in…" handoff | Idea |
| 3 | **Auto-switch**: detect when a CLI runs out of usage, move the task to another, and resume when the limit resets | Idea |

## How this repo works

This project uses [OpenSpec](https://github.com/Fission-AI/OpenSpec), which means every feature is planned before it's built:

```
openspec/changes/<change>/
  proposal.md   why and what
  design.md     how, and the decisions behind it
  specs/        the behavior contract (requirements + scenarios)
  tasks.md      the implementation checklist
```

If you use Claude Code, the `/opsx:*` commands in `.claude/` drive this workflow:

- `/opsx:explore`: think through an idea
- `/opsx:propose`: turn an idea into a change
- `/opsx:apply add-mcp-hub`: implement the tasks of a change
- `/opsx:archive`: finish a change and merge its specs into `openspec/specs/`

## Contributing

1. Read [`proposal.md`](openspec/changes/add-mcp-hub/proposal.md) and [`design.md`](openspec/changes/add-mcp-hub/design.md) for the hub.
2. Pick an unchecked task in [`tasks.md`](openspec/changes/add-mcp-hub/tasks.md). Open an issue saying you're taking it.
3. Work on a branch and open a pull request. Once CI exists (task group 1), PRs are squash-merged after checks pass.

Ideas and questions are welcome as issues.

## License

[MIT](LICENSE)
