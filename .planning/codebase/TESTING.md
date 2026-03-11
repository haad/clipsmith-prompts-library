# Testing Patterns

**Analysis Date:** 2026-03-11

## Test Framework

**Runner:**
- Jest 30.2.0
- Config: `jest.config.js`
- Environment: Node.js (`testEnvironment: 'node'`)

**Assertion Library:**
- Jest built-in expect API

**Run Commands:**
```bash
npm test              # Run all tests
npm test -- --watch  # Watch mode
npm test -- --coverage  # Generate coverage report
```

## Test File Organization

**Location:**
- Co-located in `tests/` directory parallel to source structure
- Test files mirror source organization: `tests/core/registry.test.js` for `src/core/registry.js`

**Naming:**
- Pattern: `{module-name}.test.js`
- Examples: `registry.test.js`, `installer.test.js`

**Structure:**
```
tests/
├── core/
│   ├── registry.test.js
│   └── installer.test.js
└── [additional test files as needed]
```

## Test Structure

**Suite Organization:**
```javascript
describe('registry', () => {
  describe('getAgents()', () => {
    it('returns an array', () => {
      // Test body
    });

    it('returns all 7 registered agents', () => {
      // Test body
    });
  });

  describe('getPrompts()', () => {
    // Tests for getPrompts
  });
});
```

**Patterns:**
- Top-level `describe` block per module
- Nested `describe` blocks per function/feature
- Section separators for readability: `// ─── getAgents ─────`
- One assertion per test when possible
- Descriptive test names: `it('returns null for unknown IDs', () => {...})`

**Setup/Teardown:**
```javascript
// Per-test setup
beforeEach(() => {
  jest.spyOn(process, 'cwd').mockReturnValue('/test/project');
});

// Per-test cleanup
afterEach(() => {
  jest.restoreAllMocks();
});
```

## Mocking

**Framework:** Jest built-in mocking via `jest.spyOn()` and `jest.mock()`

**Patterns:**
```javascript
// Mock process.cwd() for path resolution tests
jest.spyOn(process, 'cwd').mockReturnValue(CWD);

// Mock return values for property testing
jest.spyOn(obj, 'method').mockReturnValue(expectedValue);

// Restore all mocks after test
jest.restoreAllMocks();
```

**What to Mock:**
- Environment state: `process.cwd()`, `process.env`
- File system operations (test behavior, not actual I/O)
- External API calls in integration scenarios

**What NOT to Mock:**
- Pure data query functions (e.g., `registry.getAgents()` - test actual behavior)
- Core business logic (test real transformations)
- Test data from actual registry JSON

## Fixtures and Factories

**Test Data:**
- Loaded from actual `registry.json` file
- No mock data factories; tests use real registry data
- Benefits: tests verify actual data structure contracts

**Location:**
- Registry data at `src/core/registry.js` loads from `.prompt-library/registry.json`
- Tests reference actual items by ID (e.g., `'hemingway'`, `'adr'`)

## Coverage

**Requirements:** None explicitly enforced, but `jest.config.js` configured to collect coverage

**View Coverage:**
```bash
npm test -- --coverage
```

**Coverage Configuration:**
```javascript
collectCoverageFrom: [
  'src/**/*.js',
  '!src/**/*.test.js'
],
coverageDirectory: 'coverage'
```

## Test Types

**Unit Tests:**
- Scope: Individual functions in isolation (e.g., `getAgents()`, `findById()`)
- Approach: Test input→output behavior with various cases
- Example: `registry.test.js` tests all registry query functions
- Pattern: One function behavior per test file

**Integration Tests:**
- Not currently implemented as separate suite
- Could test: Command orchestration, multi-module interactions
- Future area: test `init.js` command with mocked installer

**E2E Tests:**
- Not implemented
- Would require: Full CLI invocation testing with real/mock file system

## Common Patterns

**Testing Array/Collection Results:**
```javascript
it('returns all 7 registered agents', () => {
  expect(registry.getAgents()).toHaveLength(7);
});

it('includes the ellie agent', () => {
  const ids = registry.getAgents().map((a) => a.id);
  expect(ids).toContain('ellie');
});
```

**Testing Object Properties:**
```javascript
it('every agent has required fields', () => {
  registry.getAgents().forEach((agent) => {
    expect(agent).toHaveProperty('id');
    expect(agent).toHaveProperty('name');
    expect(agent).toHaveProperty('description');
  });
});
```

**Testing Search/Query Functions:**
```javascript
it('finds an agent by ID', () => {
  const item = registry.findById('hemingway');
  expect(item).not.toBeNull();
  expect(item.type).toBe('agent');
  expect(item.id).toBe('hemingway');
});

it('is case-insensitive', () => {
  expect(registry.findById('HEMINGWAY')).not.toBeNull();
  expect(registry.findById('Hemingway')).not.toBeNull();
});

it('returns null for unknown IDs', () => {
  expect(registry.findById('does-not-exist')).toBeNull();
});
```

**Testing Error Conditions:**
```javascript
it('throws when attempting to install a template for claude-code', () => {
  expect(() => getTargetPath('claude-code', 'template', 'pr-template-feature.md'))
    .toThrow('Templates are not supported for this tool');
});

it('throws for an unknown item type', () => {
  expect(() => getTargetPath('github-copilot', 'unknown', 'foo.md'))
    .toThrow('Unknown item type: unknown');
});
```

**Testing Fuzzy Matching:**
```javascript
it('finds an item by exact name', () => {
  const item = registry.findByName('Hemingway');
  expect(item).not.toBeNull();
  expect(item.id).toBe('hemingway');
});

it('finds an item by partial name (fuzzy)', () => {
  const item = registry.findByName('Clarity');
  expect(item).not.toBeNull();
});

it('finds an item by partial ID (fuzzy)', () => {
  const item = registry.findByName('pr-template');
  expect(item).not.toBeNull();
  expect(item.type).toBe('template');
});
```

**Testing Filtering:**
```javascript
it('returns items matching a tag', () => {
  const items = registry.findByTag('architecture');
  expect(items.length).toBeGreaterThan(0);
  items.forEach((item) => {
    expect(item.tags.some((t) => t.includes('architecture'))).toBe(true);
  });
});

it('is case-insensitive', () => {
  const lower = registry.findByTag('architecture');
  const upper = registry.findByTag('ARCHITECTURE');
  expect(lower.length).toBe(upper.length);
});
```

## Test Characteristics

**Current Coverage Areas:**
- `registry.js` (10 test suites, ~25 assertions):
  - Data retrieval (getAgents, getPrompts, getTemplates, getScenarios)
  - Search/query functions (findById, findByName, findByTag, findByTool)
  - Metadata retrieval
  - Type assignment in combined results
  - Case-insensitivity
  - Fuzzy matching behavior

- `installer.js` (2 test suites, ~13 assertions):
  - Target path resolution for different tools
  - Target filename generation
  - Error conditions (unsupported templates, unknown types)
  - Tool-specific path mappings

**Gaps:**
- No async operation testing (install, fetch, write)
- No error handling for network failures
- No file system integration tests
- No command-level tests (init, list, add)

---

*Testing analysis: 2026-03-11*
