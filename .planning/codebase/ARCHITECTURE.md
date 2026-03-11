# Architecture

**Analysis Date:** 2026-03-11

## Pattern Overview

**Overall:** Multi-tier, data-driven prompt distribution system with three main distribution channels (CLI, Website UI, programmatic registry)

**Key Characteristics:**
- Registry-based discovery pattern for agents, prompts, and scenarios
- Decoupled distribution layers (CLI, web, embedded)
- GitHub-sourced content with local installation targets
- Tool-agnostic with specific path mappings per AI assistant
- Scenario bundling for related content collections

## Layers

**Presentation Layer:**
- Purpose: User interfaces for browsing and selecting prompts
- Location: `dist/goose-prompt-library/` (React/Docusaurus pages), `dist/prompt-library/cli/bin/` (CLI entry point)
- Contains: React components, CLI command handlers, UI utilities
- Depends on: Registry service, data fetchers, file system operations
- Used by: End users (web browser, command line)

**Command Layer:**
- Purpose: Parse user intent and orchestrate operations
- Location: `dist/prompt-library/cli/src/commands/`
- Contains: init.js, list.js, add.js - interactive wizards for setup and selection
- Depends on: Registry, installer, logger utilities
- Used by: CLI entry point

**Core Logic Layer:**
- Purpose: Business logic for item discovery, validation, and installation
- Location: `dist/prompt-library/cli/src/core/`
- Contains: registry.js (query items), installer.js (manage installation), fetcher.js (fetch from GitHub), file-system.js (local I/O)
- Depends on: GitHub repository data, local filesystem, axios
- Used by: Commands and installer

**Data Layer:**
- Purpose: Centralized inventory of all distributable items
- Location: `dist/prompt-library/cli/.prompt-library/registry.json`, `dist/goose-prompt-library/data/prompts/`
- Contains: Structured metadata for agents, prompts, scenarios, templates with tool compatibility
- Depends on: Nothing (source of truth)
- Used by: Registry service for queries

**Infrastructure Layer:**
- Purpose: Low-level operations for network and filesystem
- Location: `dist/prompt-library/cli/src/core/file-system.js`, `dist/prompt-library/cli/src/core/fetcher.js`
- Contains: File I/O (read, write, hash), HTTP requests with retry logic
- Depends on: axios, Node.js fs module
- Used by: Installer and commands

## Data Flow

**User Selection Flow:**

1. User runs `prompt-library init` (CLI entry point at `bin/prompt-library.js`)
2. Init command (`commands/init.js`) loads registry via `registry.getScenarios()` and `registry.getAgents()`
3. User selects tool (Claude Code, GitHub Copilot, both) and chooses scenario or individual items
4. Selected items passed to `installItems()` in `installer.js`
5. For each item, `fetchFile()` retrieves content from GitHub via raw content URL
6. `writeFile()` writes to tool-specific path (mapped via `PATH_MAPPINGS`)
7. `updateTrackingFile()` updates `.prompt-library.json` with installation record
8. Results displayed with success/failure summary

**Web Browsing Flow:**

1. User navigates to prompt library website (Docusaurus site in `dist/goose-prompt-library/`)
2. HomePage component (`index.tsx`) renders search/filter interface
3. `searchPrompts()` utility queries loaded prompts data
4. Results filtered by category (business/technical/productivity) and job role
5. PromptCard component displays each result with metadata
6. User can click for detail view (`detail.tsx`) showing extensions and installation instructions

**State Management:**
- Registry state: Loaded from static JSON file, immutable, queried via registry.js functions
- Installation state: Tracked in `.prompt-library.json` with file hashes for change detection
- UI state: React useState hooks manage search query, selected filters, pagination, mobile menu

## Key Abstractions

**Registry:**
- Purpose: Query interface for agents, prompts, scenarios, and templates
- Examples: `dist/prompt-library/cli/src/core/registry.js`
- Pattern: Functions for getAgents(), getPrompts(), getScenarios(), findById(), findByTag(), findByTool()
- Acts as: Single source of truth for what's available

**Installer:**
- Purpose: Orchestrates fetching and writing installation packages
- Examples: `dist/prompt-library/cli/src/core/installer.js`
- Pattern: installItem()/installItems() with success/error tracking, supports dry-run mode
- Acts as: Atomic operation wrapper for installation with rollback capability

**Fetcher:**
- Purpose: Reliable GitHub content retrieval with retry/backoff
- Examples: `dist/prompt-library/cli/src/core/fetcher.js`
- Pattern: fetchFile() with exponential backoff (1s, 2s, 4s), fileExists() for validation
- Acts as: Resilient HTTP layer abstracting GitHub API details

**File System Wrapper:**
- Purpose: Localized filesystem operations with validation
- Examples: `dist/prompt-library/cli/src/core/file-system.js`
- Pattern: writeFile(), readJson(), computeHash() for change tracking
- Acts as: Cross-platform filesystem abstraction

## Entry Points

**CLI Entry Point:**
- Location: `dist/prompt-library/cli/bin/prompt-library.js`
- Triggers: NPM install as global or `npx` invocation
- Responsibilities: Loads CLI module and calls `cli.run()`

**CLI Main:**
- Location: `dist/prompt-library/cli/src/index.js`
- Triggers: Executed by bin wrapper
- Responsibilities: Defines Commander.js program with init/list/add commands

**Web Entry Point:**
- Location: `dist/goose-prompt-library/index.tsx`
- Triggers: Browser navigation to root path
- Responsibilities: HomePage component with search/filter UI

**Detail Page:**
- Location: `dist/goose-prompt-library/detail.tsx`
- Triggers: Click on prompt card or direct URL navigation
- Responsibilities: Display full prompt details with extensions and metadata

## Error Handling

**Strategy:** Graceful degradation with user-facing error messages

**Patterns:**
- Fetcher: Try up to 3 times with exponential backoff, fail immediately on 404 (file missing)
- Installer: Collect successful and failed items separately, show summary with per-item errors
- Commands: Wrap in try-catch, pass errors to logger for formatting and exit with code 1
- Web: Display Admonition error component when data fetch fails, show empty state for no results

## Cross-Cutting Concerns

**Logging:**
- Framework: Custom logger utility at `dist/prompt-library/cli/src/utils/logger.js`
- Output: Structured console logs with colors (chalk), spinners for progress
- Usage: Commands and core modules log operations, progress, errors via logger methods

**Validation:**
- Registry: findById(), findByName() with case-insensitive lookup and fuzzy matching
- Installation: Path construction validates tool type and item type before writing
- Web: Type system via TypeScript interfaces for Prompt, Extension types

**Tool Compatibility:**
- Path Mappings: `dist/prompt-library/cli/src/constants.js` defines where each tool stores items
- Installation Flow: Tool selection in init command determines target paths for all items
- Metadata: Each item has `tools` array indicating Claude Code, GitHub Copilot, or both compatibility

---

*Architecture analysis: 2026-03-11*
