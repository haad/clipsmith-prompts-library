# Technology Stack

**Analysis Date:** 2026-03-11

## Languages

**Primary:**
- JavaScript (Node.js) - CLI application and runtime execution
- TypeScript - React components for the Docusaurus site (`.tsx` files)
- Markdown - Prompt and agent content definitions

**Secondary:**
- JSON - Configuration and data files (registry, tracking)
- YAML - Configuration (`.yamllint.yaml`)
- CSS - Component styling

## Runtime

**Environment:**
- Node.js >= 18.0.0 (specified in `package.json` engines)

**Package Manager:**
- npm (uses `package-lock.json` v3)
- Lockfile: Present

## Frameworks

**Core:**
- Commander.js 14.0.2 - CLI argument parsing and command structure
- Docusaurus (implied by imports) - Static site generation for the Goose prompt library

**UI/Frontend:**
- React - Component framework for Docusaurus site
- Framer Motion - Animation library for prompt cards

**Testing:**
- Jest 30.2.0 - Test runner and assertion library

**Build/Dev:**
- ESLint 9.39.2 - Code linting with flat config support
- Prettier 3.0.0 - Code formatting
- @eslint/js 9.39.2 - ESLint recommended rules

## Key Dependencies

**Critical:**
- axios 1.6.0+ - HTTP client for fetching files from GitHub
- chalk 4.1.2 - Terminal color output and styling
- commander 14.0.2 - CLI framework for building command-line tools
- fs-extra 11.3.3 - Enhanced file system utilities with async support
- inquirer 9.3.8 - Interactive command-line prompts and dialogs
- ora 5.4.1 - Elegant terminal spinners for progress indication
- semver 7.5.0 - Semantic versioning utilities for version management
- yaml 2.3.0 - YAML parsing and stringification

**UI/Styling:**
- lucide-react - Icon library for React components
- motion/framer-motion - Animation and transition library

**Development:**
- globals 17.0.0 - Global variables for Node.js and browser environments
- @eslint/js - ESLint base configuration

## Configuration

**Environment:**
- No environment variables explicitly required in source code
- Configuration driven by interactive prompts (inquirer)
- GitHub repository location hardcoded in constants: `shawnewallace/prompt-library` on `main` branch

**Build:**
- ESLint config: `eslint.config.js` (flat config format)
- Jest config: `jest.config.js`
- No build compilation step for CLI (pure Node.js/CommonJS)

**Project Configuration:**
- `.yamllint.yaml` - YAML linting configuration
- `.claude/settings.json` and `.claude/settings.local.json` - Claude IDE settings

## Platform Requirements

**Development:**
- Node.js >= 18.0.0
- npm or compatible package manager
- Standard POSIX-compliant file system
- Network access to GitHub (for fetching files via raw.githubusercontent.com)

**Production:**
- Node.js >= 18.0.0 runtime
- File system write permissions for installation directories (`.claude/`, `.github/`)
- Network connectivity to GitHub to fetch agents and prompts
- Unix-like or Windows file system

**Note on Dependencies:**
- Package override in `package.json`: `onetime` pinned to 5.1.2 (likely for inquirer compatibility)

---

*Stack analysis: 2026-03-11*
