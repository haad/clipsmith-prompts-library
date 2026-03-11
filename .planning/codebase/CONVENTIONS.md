# Coding Conventions

**Analysis Date:** 2026-03-11

## Naming Patterns

**Files:**
- Modules: `kebab-case.js` (e.g., `file-system.js`, `registry.js`)
- Command files: verb-based kebab-case (e.g., `init.js`, `add.js`)
- Test files: same as source with `.test.js` suffix (e.g., `registry.test.js`)

**Functions:**
- camelCase for function names (e.g., `getAgents()`, `findById()`, `installItem()`)
- Verb-first pattern for action functions: `get*`, `fetch*`, `find*`, `install*`, `write*`, `read*`
- Utility functions follow domain logic: `computeHash()`, `ensureDirectory()`

**Variables:**
- camelCase for constants that are not universally static (e.g., `selectedItems`, `dryRun`)
- UPPER_SNAKE_CASE for exported constants: `GITHUB_REPO`, `TOOL_TYPES`, `PATH_MAPPINGS`
- Prefix boolean flags with is/has/should: `dryRun`, `dryRun`, `fileExists`

**Types:**
- Objects as plain objects with JSDoc typing for clarity
- Type hints provided in JSDoc comments
- Single responsibility per object structure (agents, prompts, templates, scenarios)

## Code Style

**Formatting:**
- Prettier with config in `.prettierrc`
- `semi: true` - Semicolons required
- `singleQuote: true` - Single quotes for strings
- `tabWidth: 2` - 2 spaces per indent
- `trailingComma: 'es5'` - Trailing commas where valid in ES5
- `printWidth: 100` - Maximum 100 character line length

**Linting:**
- ESLint with `@eslint/js` recommended config
- `no-console: off` - Console logging allowed
- `no-unused-vars: error` - Unused variables must use `_` prefix to ignore (e.g., `_unused`)
- Source type: `commonjs`
- ES2021 + Node.js globals enabled

## Import Organization

**Order:**
1. Node.js built-in modules (e.g., `path`, `crypto`, `fs`)
2. Third-party dependencies (e.g., `chalk`, `axios`, `inquirer`)
3. Local modules using relative paths (e.g., `'./registry.js'`, `'../constants.js'`)

**Path Aliases:**
- No path aliases configured
- Relative paths used throughout: `'../../src/core/registry.js'`
- Consistent path depth structure in imports

**Module Exports:**
- CommonJS pattern: `module.exports = { function1, function2 }`
- Barrel files not used; explicit exports per function
- Each module exports all public functions at end of file

## Error Handling

**Patterns:**
- Try-catch blocks around async operations (see `fetchFile()` in `fetcher.js`)
- Error objects preserve message and context: `error: error.message`
- Installation results return success/failure object: `{ success: true/false, error: errorMessage }`
- Errors thrown with descriptive messages: `throw new Error('Unknown item type: ${itemType}')`
- Graceful degradation: silent fallbacks for non-critical errors (e.g., missing templates key)

**User-Facing Errors:**
- CLI errors logged via `logger.error()` with red color output
- Process exit on fatal errors: `process.exit(1)`
- User-friendly error messages separate from technical details

## Logging

**Framework:** Console via `logger.js` utility module

**Patterns:**
- Centralized logger at `src/utils/logger.js` with semantic functions
- `info()` - Blue info messages with ℹ icon
- `success()` - Green success messages with ✓ icon
- `error()` - Red error messages with ✗ icon
- `warning()` - Yellow warning messages with ⚠ icon
- `log()` - Plain text, no color
- `dim()` - Dimmed secondary text
- `header()` - Bold blue section headers
- `divider()` - Repeating dash separator line
- `spinner()` - Animated loading indicator via `ora` package
- `listItem()` - Indented bullet point
- `formatPath()`, `formatCommand()`, `formatName()` - Semantic formatting for display

**When to Log:**
- User prompts and responses
- Installation progress and results
- Warnings before destructive operations
- Next steps guidance after operations

## Comments

**When to Comment:**
- JSDoc blocks for all exported functions
- Section separators for test suites: `// ─── functionName ────────`
- Inline comments for non-obvious logic (e.g., exponential backoff calculations)
- Avoid redundant comments; let code be self-documenting

**JSDoc/TSDoc:**
- Comprehensive JSDoc for all public functions
- Document parameters: `@param {type} name - Description`
- Document return values: `@returns {type} Description`
- Document throws: `@throws {Error} When condition occurs`
- Example patterns in `logger.js`, `registry.js`, `installer.js`

```javascript
/**
 * Fetch a file from GitHub raw content
 * @param {string} path - Path to file in repository
 * @param {object} options - Fetch options
 * @param {number} options.maxRetries - Maximum number of retries (default: 3)
 * @returns {Promise<string>} File content as string
 * @throws {Error} If fetch fails after all retries
 */
async function fetchFile(path, options = {}) {
  // Implementation
}
```

## Function Design

**Size:** Functions generally 10-50 lines; utility functions shorter, command handlers longer

**Parameters:**
- Options objects for extensibility: `function(requiredParam, options = {})`
- Destructure options: `const { dryRun = false } = options`
- Document optional parameters with defaults in JSDoc

**Return Values:**
- Consistent return types per function
- Results objects for operations with multiple outcomes: `{ success, item, targetPath, sha }`
- Arrays for collections: `getAgents()` returns `Array<object>`
- Null for not found: `findById('unknown')` returns `null`
- Promises for async operations

## Module Design

**Exports:**
- All public functions exported at module end
- No default exports; named exports only
- Module focused on single responsibility (e.g., `registry.js` handles queries, `installer.js` handles installation)

**Module Organization:**
- Utilities in `src/utils/` (e.g., `logger.js`)
- Core logic in `src/core/` (e.g., `registry.js`, `installer.js`, `fetcher.js`, `file-system.js`)
- Commands in `src/commands/` (e.g., `init.js`, `list.js`, `add.js`)
- Constants in `src/constants.js`
- Main entry in `src/index.js`

**Cohesion:**
- Each module handles one concern
- `registry.js` - Query and search operations
- `installer.js` - Installation and tracking
- `fetcher.js` - GitHub API interactions
- `file-system.js` - File I/O operations
- `logger.js` - User-facing output
- Commands import and orchestrate core modules

---

*Convention analysis: 2026-03-11*
