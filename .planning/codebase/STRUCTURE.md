# Codebase Structure

**Analysis Date:** 2026-03-11

## Directory Layout

```
prompt-library/
├── dist/                                    # Compiled/distributed outputs
│   ├── goose-prompt-library/               # Web UI package for Goose
│   │   ├── index.tsx                       # HomePage component (search/browse)
│   │   ├── detail.tsx                      # Detail page component
│   │   ├── types/
│   │   │   └── index.tsx                   # TypeScript type definitions
│   │   ├── data/
│   │   │   └── prompts/                    # JSON prompt definitions (50+ files)
│   │   └── styles/
│   │       └── main.css                    # Component styling
│   │
│   └── prompt-library/                     # Main prompt library package
│       ├── README.md                       # Primary documentation
│       ├── cli/                            # CLI tool implementation
│       │   ├── bin/
│       │   │   └── prompt-library.js       # CLI entry point
│       │   ├── src/
│       │   │   ├── index.js                # CLI setup with Commander.js
│       │   │   ├── constants.js            # Config: GitHub paths, tool mappings
│       │   │   ├── commands/               # Command handlers
│       │   │   │   ├── init.js             # Interactive setup wizard
│       │   │   │   ├── list.js             # List available items
│       │   │   │   └── add.js              # Add individual item
│       │   │   ├── core/                   # Business logic
│       │   │   │   ├── registry.js         # Query agents/prompts/scenarios
│       │   │   │   ├── installer.js        # Install items to project
│       │   │   │   ├── fetcher.js          # Fetch from GitHub with retry
│       │   │   │   └── file-system.js      # Filesystem operations
│       │   │   └── utils/
│       │   │       └── logger.js           # Colored console output
│       │   ├── .prompt-library/
│       │   │   └── registry.json           # Master registry of all items
│       │   ├── jest.config.js              # Test configuration
│       │   └── tests/                      # Unit tests
│       │
│       ├── scenarios/                      # Domain-specific bundles
│       │   ├── dotnet-clean-architecture/
│       │   ├── python-data-science/
│       │   ├── typescript-frontend/
│       │   └── devops-infrastructure/
│       │
│       ├── shared/                         # Cross-scenario resources
│       │   ├── agents/                     # Reusable agent definitions
│       │   ├── prompts/                    # General-purpose prompts
│       │   ├── templates/                  # PR and doc templates
│       │   └── github-copilot/             # Shared Copilot instructions
│       │
│       └── examples/                       # Real-world usage examples
│
└── .planning/
    └── codebase/                           # Documentation artifacts
        ├── ARCHITECTURE.md
        └── STRUCTURE.md
```

## Directory Purposes

**dist/goose-prompt-library:**
- Purpose: Web-based prompt browser interface for Goose AI
- Contains: React/Docusaurus pages, JSON prompt data, styling
- Key files: `index.tsx` (homepage), `detail.tsx` (detail page), `data/prompts/` (50+ prompt definitions)
- Generated/Committed: Committed to repo as distributed bundle

**dist/prompt-library/cli:**
- Purpose: Command-line interface for installing prompts into projects
- Contains: Commander.js setup, interactive commands, registry querying, GitHub integration
- Key files: `src/index.js` (CLI entry), `src/commands/*` (init/list/add), `src/core/*` (business logic)
- Committed: Source code committed, registry.json as data source

**dist/prompt-library/scenarios:**
- Purpose: Domain-specific bundles grouping agents and prompts by use case
- Contains: Directories for .NET, Python, TypeScript, DevOps with scenario-specific instructions
- Organization: Each scenario has `claude-code/`, `github-copilot/` subdirectories for tool-specific content
- Committed: Instruction files and README for each scenario

**dist/prompt-library/shared:**
- Purpose: Reusable resources across all scenarios
- Contains: Agent definitions (Hemingway, Archy, Chester, etc.), general prompts, PR templates
- Organization: agents/, prompts/, templates/ subdirectories by resource type
- Committed: Shared markdown and template files

**dist/prompt-library/examples:**
- Purpose: Real-world usage examples showing how to use specific scenarios
- Contains: Example project structures demonstrating best practices
- Committed: Example code and configuration

## Key File Locations

**Entry Points:**
- `dist/prompt-library/cli/bin/prompt-library.js` - CLI executable entry point (shebang: #!/usr/bin/env node)
- `dist/goose-prompt-library/index.tsx` - Web homepage React component

**Configuration:**
- `dist/prompt-library/cli/src/constants.js` - GitHub repo URL, branch, tool path mappings, tracking file name
- `dist/prompt-library/cli/.prompt-library/registry.json` - Master registry with all agents, prompts, scenarios, templates

**Core Logic:**
- `dist/prompt-library/cli/src/core/registry.js` - Query interface for registry items (getAgents, getPrompts, findById, findByTag, findByTool)
- `dist/prompt-library/cli/src/core/installer.js` - Installation orchestration (installItem, installItems, updateTrackingFile)
- `dist/prompt-library/cli/src/core/fetcher.js` - GitHub content retrieval with exponential backoff retry
- `dist/prompt-library/cli/src/core/file-system.js` - Filesystem operations (writeFile, readJson, writeJson, computeHash)

**Commands:**
- `dist/prompt-library/cli/src/commands/init.js` - Interactive wizard for initial setup (tool selection, scenario/items selection)
- `dist/prompt-library/cli/src/commands/list.js` - Display available agents, prompts, scenarios with filtering
- `dist/prompt-library/cli/src/commands/add.js` - Add single agent or prompt by name/ID

**Testing:**
- `dist/prompt-library/cli/tests/` - Unit tests for core modules
- `dist/prompt-library/cli/jest.config.js` - Jest test configuration

**Web UI Components:**
- `dist/goose-prompt-library/index.tsx` - HomePage: search, category filter, sidebar job role filter, pagination
- `dist/goose-prompt-library/detail.tsx` - Detail view: prompt metadata, extensions list, installation instructions
- `dist/goose-prompt-library/types/index.tsx` - Type definitions for Prompt, Extension, Category types

**Data:**
- `dist/goose-prompt-library/data/prompts/*.json` - Individual prompt definitions (50+ JSON files with title, description, example_prompt, extensions, category, job)

## Naming Conventions

**Files:**
- CLI commands: lowercase action name (init.js, list.js, add.js)
- Core modules: lowercase domain name (registry.js, installer.js, fetcher.js, file-system.js)
- Utils: function purpose (logger.js)
- Prompts: kebab-case from ID (code-documentation-migrator.json, pr-impact-analyzer.json)
- React pages: lowercase component function (index.tsx, detail.tsx)
- Test files: module name + .test.js suffix (registry.test.js)

**Directories:**
- Feature modules: lowercase plural (commands/, scenarios/, shared/)
- Internal organization: lowercase descriptive (core/, utils/, bin/)
- Scenarios: kebab-case domain (dotnet-clean-architecture/, python-data-science/, typescript-frontend/)
- Tool-specific: tool name (claude-code/, github-copilot/)

**Functions:**
- Async operations: camelCase verb (fetchFile, installItems, writeFile)
- Getters: get* prefix (getAgents, getPrompts, getTargetPath)
- Finders: find* prefix (findById, findByName, findByTag)
- Predicates: is/has prefix (fileExists, isExpanded)
- Utilities: camelCase noun (computeHash, formatCommand)

**Constants:**
- UPPERCASE_SNAKE_CASE: GITHUB_REPO, GITHUB_BRANCH, TOOL_TYPES, PATH_MAPPINGS, TRACKING_FILE

## Where to Add New Code

**New Prompt:**
- Add JSON definition to `dist/goose-prompt-library/data/prompts/{id}.json` following schema from existing prompts (id, title, description, example_prompt, category, job, extensions)
- Register in registry.json under prompts array
- Source markdown file in GitHub repo for installer to fetch

**New Command:**
- Create `dist/prompt-library/cli/src/commands/{action}.js` exporting async function
- Import and add to program in `src/index.js` using program.command()
- Follow inquirer.prompt pattern used in init.js for user interaction
- Use logger utility for output formatting

**New Scenario Bundle:**
- Create directory `dist/prompt-library/scenarios/{scenario-name}/`
- Add subdirectories: `claude-code/commands/`, `github-copilot/instructions/`
- Create markdown instruction files for each tool
- Add README.md describing scenario purpose and use cases
- Register in registry.json under scenarios array with includes.agents and includes.prompts references

**New Scenario-Specific Prompt:**
- Create markdown file in `dist/prompt-library/scenarios/{scenario}/prompts/{name}.prompt.md`
- Register in scenario's prompts folder with metadata file
- Link from scenario README

**Utilities:**
- Shared helpers: `dist/prompt-library/cli/src/utils/{module}.js`
- Export functions from module, import in commands/core as needed
- Follow existing logger.js pattern with exported functions

**Web UI Enhancement:**
- Components: Add to `dist/goose-prompt-library/` alongside index.tsx and detail.tsx
- Types: Extend `dist/goose-prompt-library/types/index.tsx` with new TypeScript interfaces
- Styles: Update `dist/goose-prompt-library/styles/main.css` or add scoped styles to components
- Data: New prompts automatically picked up from `data/prompts/` directory

## Special Directories

**dist/prompt-library/cli/.prompt-library:**
- Purpose: Static data directory bundled with CLI package
- Generated: No (committed source)
- Committed: Yes
- Contains: registry.json master inventory

**dist/.git:**
- Purpose: Submodule git directory for prompt-library repo
- Generated: Yes (git clone)
- Committed: No

**dist/*/node_modules:**
- Purpose: Dependencies for CLI and web packages
- Generated: Yes (npm install)
- Committed: No (.gitignore)

**dist/prompt-library/.claude:**
- Purpose: Claude Code specific configuration
- Generated: No (user-created via CLI)
- Committed: No (.gitignore)

---

*Structure analysis: 2026-03-11*
