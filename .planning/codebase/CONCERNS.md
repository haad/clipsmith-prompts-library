# Codebase Concerns

**Analysis Date:** 2026-03-11

## Tech Debt

**Incomplete Command Implementation:**
- Issue: Future commands (`check` and `update`) are commented out and never implemented
- Files: `dist/prompt-library/cli/src/index.js` (lines 74-104)
- Impact: Users cannot check for updates to installed items or update installed agents/prompts, reducing utility of the tool
- Fix approach: Complete implementation of `check` and `update` commands following the pattern established by `init`, `list`, and `add` commands

**Hardcoded GitHub Repository:**
- Issue: GitHub repository reference is hardcoded in constants, making it inflexible
- Files: `dist/prompt-library/cli/src/constants.js` (lines 5-7)
- Impact: Cannot support alternative registry sources or private repositories; requires code change to switch registries
- Fix approach: Accept registry URL as configuration option, allow environment variable override, or support configuration file

**Missing Configuration File Support:**
- Issue: No mechanism for project-level configuration files (e.g., `.prompt-library.config.json` or `prompt-library.json`)
- Files: Entire CLI lacks config file parsing
- Impact: Tool is less flexible for team environments; path mappings and registry sources cannot be customized per project
- Fix approach: Add support for `.prompt-library.config.json` in project root with schema for tool-specific paths and custom registry URLs

**Registry Data Bundled at Build Time:**
- Issue: Registry is embedded in distribution as `registry.json` at module build time
- Files: `dist/prompt-library/cli/src/core/registry.js` (line 5) - requires static `../../.prompt-library/registry.json`
- Impact: Registry updates require rebuilding and releasing new CLI version; cannot hotfix registry issues
- Fix approach: Fetch registry from GitHub at runtime with caching, fallback to bundled registry if network unavailable

## Known Bugs

**Incomplete Installation Status Handling:**
- Symptoms: CLI shows items as "installed" based on tracking file, but doesn't verify actual file existence or integrity
- Files: `dist/prompt-library/cli/src/core/installer.js` (lines 184-196), `dist/prompt-library/cli/src/commands/list.js` (lines 16-23)
- Trigger: User manually deletes an installed file; `prompt-library list` still shows it as installed
- Workaround: Manually delete `.prompt-library.json` tracking file to reset state

**Fuzzy Matching Collision Risk:**
- Symptoms: `findByName()` may match wrong item if search term appears in multiple items
- Files: `dist/prompt-library/cli/src/core/registry.js` (lines 84-101)
- Trigger: User runs `prompt-library add test` when multiple items contain "test" in name/ID
- Workaround: Use full ID or more specific name prefix; exact match is prioritized but may still cause confusion

**Tool Compatibility Not Enforced During `init`:**
- Symptoms: User selects agents/prompts that aren't compatible with their chosen tool
- Files: `dist/prompt-library/cli/src/commands/init.js` (lines 82-115) - no validation that selected items support chosen tool
- Trigger: Select GitHub Copilot, then select an agent that only supports Claude Code
- Workaround: Installation will fail with error; user must reinitialize and reselect

## Security Considerations

**No Path Traversal Validation:**
- Risk: If registry contains malicious `sourcePath` values (e.g., `../../.ssh/id_rsa`), files could be written outside intended directories
- Files: `dist/prompt-library/cli/src/core/installer.js` (lines 17-32, especially line 21-23 where `getTargetPath()` joins paths)
- Current mitigation: GitHub as trusted source; registry changes are visible in git history
- Recommendations:
  - Validate that resolved target paths always fall within tool-specific directories (`.claude/`, `.github/`)
  - Use path normalization and startswith checks: `path.resolve(targetPath).startsWith(path.resolve(allowedDir))`
  - Add validation in `getTargetPath()` to reject paths containing `..` or absolute paths

**Unvalidated File Content from Network:**
- Risk: Downloaded files from GitHub raw content are written directly without validation
- Files: `dist/prompt-library/cli/src/core/installer.js` (line 65 - `writeFile` called with fetched content), `dist/prompt-library/cli/src/core/fetcher.js` (line 38)
- Current mitigation: GitHub HTTPS enforces in-transit security; content SHA is computed but not verified against known hash
- Recommendations:
  - Store expected SHA-256 hashes in registry for each item
  - Verify downloaded file hash matches registry hash before writing
  - Reject files that don't match expected hash
  - Log all write operations with file paths and hashes for audit trail

**axios validateStatus Configuration:**
- Risk: `validateStatus: (status) => status === 200` rejects all non-200 responses, but error handling could mask server-side security issues (403 Forbidden, 401 Unauthorized)
- Files: `dist/prompt-library/cli/src/core/fetcher.js` (line 35)
- Current mitigation: Proper error messages shown to user
- Recommendations: Consider treating 403/401 as distinct errors; may indicate registry access control or authentication issues

**No Rate Limiting on Parallel Fetches:**
- Risk: `fetchMultipleFiles()` spawns unlimited concurrent axios requests; could be used in DoS attack pattern
- Files: `dist/prompt-library/cli/src/core/fetcher.js` (lines 71-82)
- Current mitigation: Typical user scenarios (init with ~10 items) are harmless
- Recommendations:
  - Implement concurrent request limit (e.g., max 5 concurrent)
  - Add global timeout protection
  - Consider rate limiting for file exists checks

## Performance Bottlenecks

**Sequential File Installation:**
- Problem: `installItems()` installs items one-by-one; with 100+ items could take significant time
- Files: `dist/prompt-library/cli/src/core/installer.js` (lines 96-113)
- Cause: Simple for-loop with await; no parallelization
- Improvement path: Use `Promise.all()` to fetch files in parallel while maintaining sequential writes to avoid race conditions on tracking file

**Registry Loaded Entirely Into Memory:**
- Problem: Full registry data loaded for every CLI invocation
- Files: `dist/prompt-library/cli/src/core/registry.js` (line 5 - requires full registry.json)
- Cause: Hard-coded require() call loads entire module into memory
- Improvement path:
  - For large registries (1000+ items), consider lazy-loading or index-based lookup
  - Implement registry caching with expiration
  - Profile actual memory usage first (likely not a real issue for current ~100 items)

**Redundant File Existence Checks:**
- Problem: `isInstalled()` reads entire tracking file for each check
- Files: `dist/prompt-library/cli/src/core/installer.js` (lines 184-196), called in `add.js` line 42
- Cause: Separate file read per query
- Improvement path: Cache tracking file in memory, invalidate on write; or pass tracking data as parameter through function calls

## Fragile Areas

**Tracking File Format Compatibility:**
- Files: `dist/prompt-library/cli/src/core/installer.js` (lines 121-163 - updateTrackingFile)
- Why fragile: Schema could break if registry item structure changes; backwards compatibility code exists (lines 140-143) but forward compatibility not handled
- Safe modification:
  - Always add a `version` field to tracking file format
  - Check version on read, migrate old formats, provide clear error if future version encountered
  - Test with manually crafted tracking files of different versions
- Test coverage: Gaps - no tests for tracking file migration or corruption handling

**Scenario Bundle Installation:**
- Files: `dist/prompt-library/cli/src/commands/add.js` (lines 94-183), `dist/prompt-library/cli/src/commands/init.js` (lines 64-79)
- Why fragile: References items by ID without validating they exist; `filter(Boolean)` silently drops broken references
- Safe modification:
  - Check that all referenced item IDs exist before starting installation
  - Fail explicitly if scenario references missing items
  - Provide clear error message listing unavailable items
- Test coverage: Gaps - no test for scenario with missing item references

**Interactive Prompts Without Timeout:**
- Files: `dist/prompt-library/cli/src/commands/init.js`, `dist/prompt-library/cli/src/commands/add.js`
- Why fragile: `inquirer.prompt()` can hang indefinitely if stdin is unavailable or closed
- Safe modification:
  - Add timeout to inquirer prompts (e.g., 5 minute timeout)
  - Handle `isTtyError` properly (partially done on line 223-224 of init.js but not in add.js)
  - Test in non-interactive CI environment
- Test coverage: Gaps - no tests for non-interactive environment

**Hard Mocked process.cwd() in Tests:**
- Files: `dist/prompt-library/cli/tests/core/installer.test.js` (lines 6-8)
- Why fragile: Tests pass with CWD '/test/project' but actual behavior depends on real `process.cwd()` which may have permission issues or special characters
- Safe modification:
  - Test with actual temp directories created during test
  - Test with various path patterns (spaces, special chars, long paths)
  - Test write permission checking
- Test coverage: Gaps - no integration tests with real file system

## Scaling Limits

**Registry Size Impact:**
- Current capacity: ~100 items (7 agents + 4 prompts + 3 templates + 4 scenarios)
- Limit: At ~1000+ items, memory usage and listing performance may degrade
- Scaling path:
  - Implement pagination in `list` command
  - Add search/filter before displaying
  - Consider registry partitioning by category

**Tracking File Growth:**
- Current capacity: With ~7000 items installed, tracking file remains <1KB per item
- Limit: With millions of installs tracked (edge case), file could grow large
- Scaling path: Archive old tracking records, implement database-backed tracking, or implement per-item tracking files

## Dependencies at Risk

**axios ^1.6.0:**
- Risk: Major version 1.x security updates; no version pinning
- Impact: Updates to axios could introduce breaking changes or incompatibilities
- Migration plan: Pin to specific minor version (e.g., `^1.6.8`), periodically audit and test updates

**inquirer ^9.3.8:**
- Risk: Known issues with TTY handling in non-interactive environments; major version updates change API
- Impact: CLI hangs or crashes in CI/CD pipelines without TTY
- Migration plan: Consider alternatives like `prompts` package (simpler), add explicit TTY detection, test CI/CD environments

**fs-extra ^11.3.3:**
- Risk: Older version; could be updated to latest major version
- Impact: Limited - fs-extra is stable; update would likely be safe
- Migration plan: Update to latest with thorough testing of file operations

**jest ^30.2.0:**
- Risk: Version number appears incorrect (jest is at v29.x as of 2026); verify actual version
- Impact: Unknown - may indicate typo in package.json or very outdated tooling
- Migration plan: Audit package.json versions against npm registry; update jest to current stable

## Missing Critical Features

**Update Command:**
- Problem: No way to update individual installed items to new versions
- Blocks: Users cannot keep agents/prompts current with library updates
- Impact: Installed items become stale; manual deletion and reinstall required

**Verification Command:**
- Problem: No way to verify integrity of installed items
- Blocks: Users cannot detect if files were manually edited or corrupted
- Impact: Debugging issues with broken agents becomes difficult

**Uninstall Command:**
- Problem: No way to cleanly remove installed items
- Blocks: Users must manually delete files and edit tracking file
- Impact: Cluttered project directory with unused items; manual file management error-prone

**Export/Backup Command:**
- Problem: No way to export currently installed items list for sharing or backup
- Blocks: Teams cannot easily reproduce same setup across machines
- Impact: Each developer must manually install same items

## Test Coverage Gaps

**Commands Not Tested:**
- What's not tested: `commands/init.js`, `commands/add.js`, `commands/list.js` - no test files exist
- Files: No test files for CLI commands; only core modules (`registry.test.js`, `installer.test.js`) have tests
- Risk: Command logic including user interactions and error handling untested; regressions could break CLI for users
- Priority: **High** - commands are public API; should have comprehensive tests

**File System Operations Not Tested:**
- What's not tested: `core/file-system.js` - no tests for actual file I/O operations
- Files: No test file; functions tested indirectly through installer tests only
- Risk: File permission issues, path edge cases (long paths, special chars), permission errors not caught
- Priority: **High** - file operations can silently fail or corrupt project state

**Network Operations Not Tested:**
- What's not tested: `core/fetcher.js` - no tests for retry logic, timeouts, error handling
- Files: No test file; only integration tests would catch network issues
- Risk: Network issues (timeout, partial reads, intermittent failures) may not be properly handled
- Priority: **Medium** - critical for production reliability but could be tested with network mocking

**Logger Not Tested:**
- What's not tested: `utils/logger.js` - no tests for output formatting
- Files: No test file; relies on manual verification
- Risk: Output formatting regressions not caught; UX could break silently
- Priority: **Low** - cosmetic but affects user experience

**Edge Cases in Registry Lookup:**
- What's not tested:
  - Duplicate item names/IDs handling
  - Case sensitivity edge cases
  - Special characters in item names
  - Empty registry scenarios
- Files: `registry.test.js` covers basic happy paths but not edge cases
- Risk: Unexpected behavior when registry has malformed data
- Priority: **Medium** - improves robustness

---

*Concerns audit: 2026-03-11*
