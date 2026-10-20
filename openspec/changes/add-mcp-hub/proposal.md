# Proposal

## Why

Every coding CLI (Claude Code, OpenCode, Muse Code) keeps its own MCP configuration. So tools a user already pays for (Firecrawl, Parallel, Octen, Context7) have to be set up again in each CLI, with API keys copied into several plaintext config files. Sometimes they never get set up at all. When one CLI runs out of usage and the user moves to another, their tools don't come with them. This is the first piece of a free, open-source Linux workspace for coding CLIs (in the spirit of the macOS-only JOY app). It can be built and shipped on its own, before any GUI exists.

## What Changes

- New terminal-first tool (working name `hub`) that stores provider API keys **once**, in the desktop keyring (GNOME Keyring via libsecret), never in plaintext config files.
- A local MCP hub server bound to `127.0.0.1` that aggregates the user's configured providers (remote HTTP MCP servers such as Parallel, Firecrawl and Context7, plus stdio servers) behind **one** Streamable HTTP endpoint. It attaches each provider's credentials when forwarding.
- Per-CLI **adapters** that add, update or remove a single `hub` entry in each installed CLI's MCP config (Claude Code, OpenCode, Muse Code). Edits are idempotent and backed up, and they leave the user's other entries untouched.
- Commands to add or remove a provider, start or stop the hub, sync CLIs, and show status (`hub doctor`).
- Repository and delivery setup: the project lives on GitHub, with GitHub Actions CI (lint, typecheck, tests, build) on every pull request. The development flow is **worktree → PR → CI green → squash-merge → delete branch and worktree → fast-forward local main**, scripted so nobody types it by hand.

**Out of scope** (later changes): the desktop shell (Parallel Code fork), the "Continue in…" handoff, automatic usage-limit detection and CLI switching, resume-at-reset, packaging (AppImage/RPM/Flatpak).

## Capabilities

### New Capabilities
- `mcp-hub`: storing provider credentials in the keyring, serving one aggregated local MCP endpoint, and keeping each supported CLI's MCP config pointed at that endpoint.

### Modified Capabilities
None. This is the project's first capability.

## Impact

- New TypeScript/Node codebase at the repository root (the repo currently holds only `openspec/`).
- Dependencies: the official MCP TypeScript SDK; a libsecret binding or the `secret-tool` CLI for keyring access.
- Writes to user files outside the repo, but only the `hub` entry: `~/.claude.json` (via `claude mcp` commands where possible), OpenCode's config, and `~/.config/muse/settings.json`.
- New GitHub repository with `.github/workflows/` CI and branch protection on `main` (CI must pass, squash-merge only).
- Opens a local TCP port on loopback only.
