# Spec Delta

## Purpose

Lets a user configure their paid MCP tool providers once and use them from every supported coding CLI through a single local endpoint, with credentials kept out of plaintext config files.

## ADDED Requirements

### Requirement: Provider credentials are stored in the desktop keyring
The system SHALL store each provider's API key in the user's desktop secret store and SHALL NOT write the key to any configuration file, log, or command output.

#### Scenario: Adding a provider key
- **WHEN** the user adds a provider (for example Parallel) and enters its API key
- **THEN** the key is saved in the desktop secret store under that provider's name
- **AND** no file written by the system contains the key

#### Scenario: Status output masks keys
- **WHEN** the user runs the status command
- **THEN** each provider is listed as having a key or not, and no key value is printed

#### Scenario: Secret store unavailable
- **WHEN** no desktop secret store is reachable
- **THEN** adding a provider fails with an error that names the missing secret service
- **AND** the key is not saved anywhere else

### Requirement: Providers can be added and removed
The system SHALL let the user add a provider from a built-in catalog (at least Firecrawl, Parallel, Octen and Context7) or as a custom remote URL or local command, and SHALL let the user remove a provider together with its stored key.

#### Scenario: Add from catalog
- **WHEN** the user adds the catalog provider "context7" with a key
- **THEN** the provider appears in the provider list with the catalog's endpoint

#### Scenario: Add custom provider
- **WHEN** the user adds a custom provider with a name and an HTTP MCP URL
- **THEN** the provider is listed and its tools are served once the hub is running

#### Scenario: Remove provider
- **WHEN** the user removes a provider
- **THEN** it is no longer listed, its key is deleted from the secret store, and its tools disappear from the hub

### Requirement: Hub serves one aggregated MCP endpoint on loopback
The system SHALL run a single MCP server over Streamable HTTP, bound only to 127.0.0.1, that exposes the combined tools of every enabled provider and forwards each tool call to the provider that owns it.

#### Scenario: Aggregated tool list
- **WHEN** an MCP client connects to the hub while Parallel and Context7 are enabled
- **THEN** the tool list contains the tools of both providers

#### Scenario: Tool call routing
- **WHEN** a client calls a tool that belongs to Parallel
- **THEN** the hub forwards the call to Parallel with Parallel's credentials and returns its result unchanged

#### Scenario: Not reachable from the network
- **WHEN** a request reaches the hub's port from an address other than loopback
- **THEN** the connection is refused

### Requirement: Tool names do not collide across providers
The system SHALL expose every tool under a name that is unique across all enabled providers, using a stable provider prefix whenever two providers offer the same tool name.

#### Scenario: Two providers expose web_search
- **WHEN** two enabled providers both offer a tool named `web_search`
- **THEN** the hub lists two distinct tools whose names identify each provider
- **AND** calling either one reaches the correct provider

### Requirement: One failing provider does not take down the hub
The system SHALL keep serving the tools of healthy providers when one provider is unreachable or rejects its credentials, and SHALL report the failing provider in status output.

#### Scenario: Provider with an invalid key
- **WHEN** one provider rejects its API key at hub start
- **THEN** the hub still serves the other providers' tools
- **AND** the status command shows that provider as failing, with the reason

### Requirement: Hub endpoint requires a local access token
The system SHALL require every client request to the hub to carry a locally generated bearer token, and SHALL reject requests without it.

#### Scenario: Request without token
- **WHEN** a local process connects to the hub without the token
- **THEN** the request is rejected as unauthorized

#### Scenario: Synced CLIs carry the token
- **WHEN** a CLI's config has been synced by the system
- **THEN** that CLI's hub entry includes the token, and its requests are accepted

### Requirement: Supported CLIs are pointed at the hub
The system SHALL detect which supported CLIs are installed (Claude Code, OpenCode, Muse Code) and, on sync, SHALL write exactly one MCP entry named `hub` into each one's user-level config that points at the hub endpoint.

#### Scenario: Sync with all three installed
- **WHEN** Claude Code, OpenCode and Muse Code are installed and the user runs sync
- **THEN** each CLI's MCP config contains one `hub` entry pointing at the hub's URL

#### Scenario: CLI not installed
- **WHEN** a supported CLI is not installed
- **THEN** sync skips it and reports it as not found, without creating its config

### Requirement: CLI config edits are safe and reversible
The system SHALL change only its own `hub` entry when editing a CLI config, SHALL keep a backup of the file before the first modification, and SHALL produce the same file when sync runs twice.

#### Scenario: Existing entries preserved
- **WHEN** a CLI config already defines other MCP servers and the user runs sync
- **THEN** those entries are unchanged and only the `hub` entry is added or updated

#### Scenario: Idempotent sync
- **WHEN** the user runs sync twice in a row
- **THEN** the second run makes no changes to any file

#### Scenario: Unsync
- **WHEN** the user runs unsync
- **THEN** the `hub` entry is removed from every CLI config and all other content is left as it was

#### Scenario: Unparseable config
- **WHEN** a CLI's config file cannot be parsed
- **THEN** sync leaves that file untouched and reports an error for that CLI only

### Requirement: Diagnostics explain the current state
The system SHALL provide a doctor command that reports, for each provider and each supported CLI, whether it is configured, reachable and synced, together with a suggested fix for each problem it finds.

#### Scenario: Hub not running
- **WHEN** the user runs doctor while the hub is stopped
- **THEN** the output says the hub is not running and shows the command to start it
