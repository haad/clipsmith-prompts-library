# External Integrations

**Analysis Date:** 2026-03-11

## APIs & External Services

**GitHub Repository API:**
- Service: GitHub raw content API
- What it's used for: Fetching agent and prompt files from the prompt-library repository
  - SDK/Client: axios (HTTP client)
  - Base URL: `https://raw.githubusercontent.com/shawnewallace/prompt-library/main/`
  - Auth: Not required (public repository)
  - Implementation: `src/core/fetcher.js` - `fetchFile()`, `fetchMultipleFiles()`, `fileExists()`

**File Fetching Features:**
- Automatic retry logic with exponential backoff (1s, 2s, 4s)
- Maximum 3 retries by default (configurable)
- 30-second timeout per request
- HTTP status validation
- 404 detection to avoid unnecessary retries

## Data Storage

**Local File Storage:**
- Type: Local filesystem only
- Installation tracking: `.prompt-library.json` (project root)
- Installed items: Stored in user-specified directories:
  - Claude Code: `.claude/agents/` and `.claude/commands/`
  - GitHub Copilot: `.github/agents/`, `.github/prompts/`, `.github/PULL_REQUEST_TEMPLATE/`
- Implementation: `src/core/file-system.js` - `writeFile()`, `readJson()`, `writeJson()`

**Backup Strategy:**
- File backup capability: `.prompt-library/backups/` directory
- Backup naming: `{filename}.{timestamp}.bak`
- Implementation: `backupFile()` in file-system module

**Tracking Format:**
- Location: `.prompt-library.json`
- Contents:
  - Installation timestamp
  - Tool type (claude-code or github-copilot)
  - Installed items with SHA-256 hashes
  - Source paths and target paths

**File Caching:**
- Registry data cached in: `.prompt-library/registry.json`
- Local JSON format (no external caching service)

## Authentication & Identity

**Auth Provider:** None

**Access Control:**
- All authentication is implicit (GitHub public repository access)
- CLI runs with user's local file system permissions
- Installation targets user's project directories
- No user accounts or authentication required

## Monitoring & Observability

**Error Tracking:**
- Local console output via chalk-colored messages
- Custom logger: `src/utils/logger.js`
- Error types:
  - Network failures with HTTP status codes
  - File system permission errors
  - File not found (404) errors
  - Installation failures with detailed messages

**Logs:**
- Console-based only (no persistent logging)
- Color-coded output (blue=info, green=success, red=error, yellow=warning)
- Spinner-based progress indication via ora

## CI/CD & Deployment

**Hosting:**
- npm Registry - Package published as `@shawnwallace/prompt-library`
- GitHub - Source repository at `shawnewallace/prompt-library`

**Deployment:**
- CLI installation via npm: `npm install -g @shawnwallace/prompt-library`
- Bin entry point: `bin/prompt-library.js`
- No CI/CD pipeline configuration in codebase

**Version Management:**
- Current version: 1.1.1
- Semantic versioning: `major.minor.patch`
- Published to npm registry

## Environment Configuration

**Required env vars:**
- None explicitly required
- System PATH must include npm/node binary locations

**Secrets location:**
- Not applicable - no API keys or secrets needed
- GitHub access is unauthenticated (public repository)

**File Permissions:**
- Requires write access to installation directories
- Validation: `hasWritePermission()` in file-system module
- Test via temporary `.write-test-*` files

## Registry System

**Registry Source:**
- Location: `.prompt-library/registry.json`
- Contains metadata for agents, prompts, scenarios, and templates
- Fields per item:
  - `id` - Unique identifier
  - `name` - Display name
  - `description` - Item description
  - `sourcePath` - GitHub repository path
  - `type` - Item type (agent, prompt, scenario, template)
  - `tools` - Compatible tools (claude-code, github-copilot)
  - `tags` - Search/filtering tags

**Registry Operations:**
- Query functions: `getAgents()`, `getPrompts()`, `getScenarios()`, `getTemplates()`
- Search functions: `findById()`, `findByName()`, `findByTag()`, `findByTool()`
- Metadata: `getMetadata()` returns version and lastUpdated timestamp

## Webhooks & Callbacks

**Incoming:**
- Not applicable - CLI tool, no server

**Outgoing:**
- Not applicable - No webhook support

## Content Types Supported

**Supported File Types:**
- Markdown (`.md`) - Agent and prompt definitions
- JSON (`.json`) - Registry and configuration

**Directory Mappings:**
- Claude Code: Agents in `.claude/agents/`, Prompts in `.claude/commands/`
- GitHub Copilot: Agents in `.github/agents/`, Prompts in `.github/prompts/`, Templates in `.github/PULL_REQUEST_TEMPLATE/`

## Network Requirements

**Connectivity:**
- Requires network access to: `raw.githubusercontent.com`
- Protocol: HTTPS
- Timeout: 30 seconds (configurable)

**Fallback/Offline:**
- No offline mode
- No local caching beyond registry
- Installation fails gracefully with error messages if network unavailable

---

*Integration audit: 2026-03-11*
