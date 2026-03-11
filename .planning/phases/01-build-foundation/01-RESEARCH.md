# Phase 1: Build Foundation - Research

**Researched:** 2026-03-11
**Domain:** POSIX bash scripting, YAML frontmatter parsing, jq JSON construction, JSON Schema draft 2020-12
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions

**Repository Structure**
- `prompts/` directory with four subdirectories: `coding/`, `writing/`, `analysis/`, `creative/`
- `catalog.yaml` at repo root with `version: 1` and categories list
- `scripts/build.sh` at `scripts/build.sh` (executable)
- `schema/prompt.schema.json` at `schema/prompt.schema.json`
- `.gitignore` with standard exclusions (.DS_Store, *.swp, *.swo, *~, node_modules/)
- `prompts.json` committed to main (not gitignored) — useful for PR review, overwritten by CI

**Build Script Behavior**
- Zero dependencies: bash + sed/awk/grep + jq (jq required for JSON assembly, not optional)
- Uses `jq -Rs '.'` for safe multiline content escaping (handles quotes, backslashes, newlines)
- No `declare -A` or bash 4+ features — must work on macOS bash 3.2
- Duplicate ID detection via `sort | uniq -d` (POSIX-compatible)
- Prompts sorted alphabetically by `id` in output JSON for deterministic builds
- Exits non-zero with descriptive error on: duplicate IDs, missing title/version, invalid category
- Category resolution: frontmatter `category:` field overrides directory-derived category
- Body extraction: everything after second `---`, with leading blank lines stripped
- Frontmatter parsing: sed to extract between first and second `---`, grep/awk for key-value pairs
- Title parsing must handle quoted values and values containing colons

**Output Format (Exact)**
- Top-level: `{"version": <catalog_version>, "prompts": [...]}`
- Each prompt: `{"id": "<filename>", "title": "<title>", "category": "<category>", "version": <int>, "content": "<body>"}`
- No additional fields beyond these 5 per prompt — Flycut's Decodable types reject extras
- `additionalProperties: false` in JSON Schema enforces this

**Validation Schema**
- JSON Schema draft 2020-12
- id pattern: `^[a-z0-9-]+$`
- category enum: `["coding", "writing", "analysis", "creative"]`
- version minimum: 1
- Non-empty strings for title and content

**Test Prompt (Pipeline Verification)**
- Include one test prompt (e.g., `prompts/coding/code-review-swift.md`) to validate the full pipeline
- This prompt will remain in the final catalog (it's one of the 23 seed prompts)
- Must include `{{clipboard}}` placeholder to verify template variable passthrough

### Claude's Discretion
- Exact sed/awk patterns for frontmatter extraction (as long as they handle edge cases)
- Build script summary output format
- Whether to add a `--help` flag to build.sh
- Internal variable naming in build script

### Deferred Ideas (OUT OF SCOPE)
None — discussion stayed within phase scope
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| REPO-01 | Repository has `prompts/` directory with `coding/`, `writing/`, `analysis/`, `creative/` subdirectories | Standard `mkdir -p` — trivial |
| REPO-02 | `catalog.yaml` defines catalog version and valid categories list | YAML structure specified verbatim in design doc |
| REPO-03 | `.gitignore` excludes .DS_Store, swap files, and node_modules | Exact contents in design doc |
| PRMT-01 | Each prompt is a Markdown file with YAML frontmatter containing required `title` and `version` fields | Frontmatter parsing patterns documented below |
| PRMT-02 | Optional frontmatter fields supported: `category`, `description`, `tags`, `variables` | Script ignores unknown fields naturally — no extra work |
| PRMT-03 | Prompt `id` derived from filename (without `.md`), lowercase kebab-case, unique across all categories | `basename "$file" .md` + duplicate detection via `sort \| uniq -d` |
| PRMT-04 | Category resolved from parent directory name, overridable via frontmatter `category` field | `basename "$(dirname "$file")"` + grep frontmatter for override |
| PRMT-05 | Prompt content supports `{{clipboard}}` and custom `{{variable}}` template placeholders | Verbatim passthrough — jq -Rs preserves literal `{{...}}` strings |
| BILD-01 | `scripts/build.sh` compiles all `.md` files under `prompts/` into `prompts.json` at repo root | Core script implementation |
| BILD-02 | Build script is zero-dependency (bash + sed/awk/grep + jq for JSON assembly) | jq is required, not optional; all others are POSIX standard |
| BILD-03 | Build script reads catalog version from `catalog.yaml` | `grep '^version:' catalog.yaml \| awk '{print $2}'` |
| BILD-04 | Build script derives `id` from filename and `category` from directory (with frontmatter override) | Established patterns below |
| BILD-05 | Build script fails with error on duplicate IDs across categories | Collect IDs into newline-delimited string, pipe through `sort \| uniq -d` |
| BILD-06 | Build script fails with error on missing required frontmatter fields (title, version) | Validate after parsing; name the file in the error message |
| BILD-07 | Build script fails with error on invalid category (not in catalog.yaml list) | `echo "$VALID_CATEGORIES" \| grep -qx "$category"` |
| BILD-08 | Build script outputs prompts sorted alphabetically by `id` for deterministic builds | `find ... \| sort` on input files ensures output order |
| BILD-09 | Build script properly JSON-escapes multiline Markdown content (newlines, quotes, backslashes) | `jq -Rs '.'` — the only safe approach |
| BILD-10 | Build script works on macOS bash 3.2 (no `declare -A` or bash 4+ features) | POSIX string accumulation pattern documented below |
| BILD-11 | Build script strips leading blank lines from extracted prompt body content | `awk` post-body-extraction with leading-blank-line stripping |
| JSON-01 | `prompts.json` matches Flycut's `PromptCatalog` structure: `{version: Int, prompts: [PromptDTO]}` | jq assembly with `--argjson v` and `-s` documented below |
| JSON-02 | Each prompt object has exactly 5 fields: `id`, `title`, `category`, `version`, `content` (no extras) | `jq -n` with explicit field selection |
| JSON-03 | `prompts.json` validates against `schema/prompt.schema.json` | jq-based schema validation command documented below |
| SCHM-01 | `schema/prompt.schema.json` validates the complete prompts.json structure with JSON Schema draft 2020-12 | Complete schema specified verbatim in design doc |
| SCHM-02 | Schema enforces id pattern (`^[a-z0-9-]+$`), valid category enum, minimum version 1, non-empty strings | Schema JSON documented in Code Examples below |
</phase_requirements>

## Summary

Phase 1 is a bash scripting and configuration problem, not an architecture problem. The design document (`dist/PROMPTS-REPO-DESIGN.md`) provides a complete specification including pseudocode — the implementation work is taking that pseudocode and making it production-safe. Three areas require careful implementation beyond what the pseudocode shows: (1) the pseudocode uses `declare -A` which breaks macOS bash 3.2, requiring a POSIX-compatible duplicate ID detection replacement; (2) frontmatter title parsing must handle quoted strings and colons without truncating after the first colon; (3) body extraction must strip leading blank lines that `awk` captures after the closing `---`.

The entire phase is self-contained and produces verifiable output. Success is unambiguous: `bash scripts/build.sh` on stock macOS produces a `prompts.json` that passes `jq` validation against `schema/prompt.schema.json`. No CI, no external services, no network — just local file operations.

**Primary recommendation:** Follow the design document pseudocode structure exactly, but replace `declare -A SEEN_IDS` with the POSIX string-accumulation pattern, replace title parsing with a sed pattern that preserves colons, and add a leading-blank-line strip after body extraction. Use `jq --arg` for all string injection into JSON — never concatenate strings to build JSON.

## Standard Stack

### Core

| Tool | Version | Purpose | Why Standard |
|------|---------|---------|--------------|
| bash | 3.2+ (macOS), 5.x (Ubuntu CI) | Build script runtime | Universally available; zero-install on macOS and Ubuntu |
| jq | 1.6+ | JSON construction and escaping | Only safe way to handle multiline content with quotes/backslashes/newlines |
| sed | POSIX (BSD and GNU compatible) | Frontmatter block extraction | Available on macOS and Linux without install |
| awk | POSIX (BSD and GNU compatible) | Body extraction after second `---` | Available on macOS and Linux without install; handles multi-pass logic cleanly |
| grep | POSIX | Key-value extraction from frontmatter, category validation | Available everywhere; `-qx` for exact-line matching |

### Supporting

| Tool | Version | Purpose | When to Use |
|------|---------|---------|-------------|
| find | POSIX | Locate all `.md` files under `prompts/` | `find "$PROMPTS_DIR" -name '*.md' -type f \| sort` |
| sort / uniq | POSIX | Duplicate ID detection | Collect IDs, `sort \| uniq -d` returns only duplicates |
| basename / dirname | POSIX | Extract filename (id) and directory (category) from file path | `basename "$file" .md` and `basename "$(dirname "$file")"` |
| printf | POSIX (preferred over echo) | Output accumulated JSON fragments | `printf '%s\n'` is portable; `echo` has escaping surprises |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|------------|-----------|----------|
| jq for JSON construction | Python json module, Node.js JSON.stringify | Both violate zero-dependency constraint |
| sed/awk for frontmatter | yq (YAML processor) | External dependency; not pre-installed on macOS |
| `sort \| uniq -d` for duplicates | `declare -A` associative array | `declare -A` requires bash 4+; breaks macOS bash 3.2 |
| `find ... \| sort` for file ordering | `ls -1` | `ls` output is locale-dependent; `find \| sort` is stable |

**Installation:** No installation required. `jq` is the only non-system tool; it is pre-installed on `ubuntu-latest` GitHub Actions runners. On macOS, contributors may need `brew install jq` — document this prerequisite in README.

## Architecture Patterns

### Recommended Project Structure

```
flycut-prompts/
├── catalog.yaml                   # Catalog version + valid categories allowlist
├── prompts.json                   # AUTO-GENERATED — committed for PR review visibility
├── prompts/
│   ├── coding/                    # 8 prompts (Phase 2 populates fully; 1 test prompt in Phase 1)
│   ├── writing/                   # 6 prompts (Phase 2)
│   ├── analysis/                  # 5 prompts (Phase 2)
│   └── creative/                  # 4 prompts (Phase 2)
├── scripts/
│   └── build.sh                   # Executable build script
├── schema/
│   └── prompt.schema.json         # JSON Schema draft 2020-12
└── .gitignore
```

### Pattern 1: POSIX-Compatible Duplicate ID Detection (replaces `declare -A`)

**What:** Accumulate IDs into a newline-delimited string as files are processed; after all files, pipe through `sort | uniq -d` to find any duplicates. If `uniq -d` produces output, fail.

**When to use:** Any time bash 3.2 compatibility is required and you need a set membership check.

**Example:**
```bash
# Source: design doc adaptation + macOS bash 3.2 constraint from CONTEXT.md

SEEN_IDS=""   # newline-delimited accumulator

for file in $(find "$PROMPTS_DIR" -name '*.md' -type f | sort); do
    id=$(basename "$file" .md)
    SEEN_IDS="${SEEN_IDS}${id}
"
done

# Detect duplicates after collection
DUPS=$(printf '%s' "$SEEN_IDS" | sort | uniq -d)
if [ -n "$DUPS" ]; then
    echo "ERROR: Duplicate prompt IDs detected: $DUPS" >&2
    exit 1
fi
```

**Note:** This approach detects duplicates after processing all files. If you want to fail immediately on the second occurrence (naming the conflicting file pair), you need a different strategy: process files twice — first pass collects IDs, detects duplicates, second pass builds JSON. The two-pass approach is cleaner for error messages.

### Pattern 2: Frontmatter Extraction with sed

**What:** Extract the YAML block between first and second `---` using `sed -n`, then parse individual keys with grep and a colon-safe sed pattern.

**When to use:** Simple flat YAML frontmatter (no nested keys, no arrays in values that matter to the build script).

**Example:**
```bash
# Source: dist/PROMPTS-REPO-DESIGN.md pseudocode + colon-safe adaptation

# Extract frontmatter block (content between first and second ---)
frontmatter=$(sed -n '1{/^---$/!q};/^---$/{n;:loop;/^---$/q;p;n;b loop}' "$file")

# Safe alternative using line numbers approach:
frontmatter=$(awk '/^---$/{count++; if(count==1){next}; if(count==2){exit}} count==1{print}' "$file")

# Parse title — strip leading "title:" prefix, handle quoted and unquoted values,
# preserve colons within the value
title=$(printf '%s\n' "$frontmatter" | grep '^title:' | sed 's/^title:[[:space:]]*//' | sed 's/^"\(.*\)"$/\1/')

# Parse version — simple integer extraction
version=$(printf '%s\n' "$frontmatter" | grep '^version:' | awk '{print $2}')

# Parse optional category override
fm_category=$(printf '%s\n' "$frontmatter" | grep '^category:' | sed 's/^category:[[:space:]]*//' | sed 's/^"\(.*\)"$/\1/')
```

**Critical:** The title sed pattern `sed 's/^title: *//' | sed 's/^"\(.*\)"$/\1/'` is a two-step approach:
1. Strip `title:` prefix and any leading spaces
2. If remaining value is wrapped in double quotes, strip them

This preserves colons that appear inside the title value. Never use `cut -d: -f2` — it discards everything after the first colon.

### Pattern 3: Body Extraction with Leading Blank Line Stripping

**What:** Use `awk` to capture all lines after the second `---`, then strip any leading blank lines from the result.

**When to use:** Always — every prompt file has a blank line between the closing `---` and the prompt body.

**Example:**
```bash
# Source: dist/PROMPTS-REPO-DESIGN.md pseudocode + BILD-11 requirement

# Extract body (everything after second ---)
body=$(awk 'BEGIN{n=0} /^---$/{n++; if(n==2){found=1; next}} found{print}' "$file")

# Strip leading blank lines (BILD-11)
# Use sed to delete leading empty lines
body=$(printf '%s\n' "$body" | sed '/./,$!d')
```

The `sed '/./,$!d'` pattern: delete all lines from the start up to (but not including) the first line containing any character. This is a POSIX-portable way to strip leading blank lines.

### Pattern 4: Safe JSON Construction with jq

**What:** Use `jq --arg` for string values and `--argjson` for integers; never concatenate strings to build JSON.

**When to use:** Always. No exceptions. Hand-concatenated JSON breaks on backslashes, quotes, and newlines.

**Example:**
```bash
# Source: dist/PROMPTS-REPO-DESIGN.md pseudocode

# Escape multiline body as JSON string
content_json=$(printf '%s\n' "$body" | jq -Rs '.')

# Build single prompt JSON object
prompt_json=$(jq -n \
    --arg id       "$id" \
    --arg title    "$title" \
    --arg category "$category" \
    --argjson version  "$version" \
    --argjson content  "$content_json" \
    '{id: $id, title: $title, category: $category, version: $version, content: $content}')

# Accumulate into array (bash array — safe because we're not using declare -A)
PROMPT_JSONS+=("$prompt_json")
```

**Note:** `declare -a` (indexed arrays) works on bash 3.2. Only `declare -A` (associative arrays) requires bash 4+. Indexed arrays are safe to use.

### Pattern 5: Final JSON Assembly

**What:** Pipe accumulated JSON objects through `jq -s` to build the final output array wrapped in the catalog structure.

**Example:**
```bash
# Source: dist/PROMPTS-REPO-DESIGN.md

printf '%s\n' "${PROMPT_JSONS[@]}" | \
    jq -s --argjson v "$CATALOG_VERSION" '{version: $v, prompts: .}' > "$OUTPUT"
```

### Pattern 6: jq-Based Schema Validation

**What:** Use `jq` to validate `prompts.json` against the schema at build time. Since `jq` does not natively support JSON Schema, validation is implemented as a series of `jq -e` assertions.

**Example:**
```bash
# Structural validation (no external schema validator needed — jq -e assertions)
jq -e '.version and (.version | type == "number")' "$OUTPUT" > /dev/null
jq -e '.prompts | type == "array" and length > 0' "$OUTPUT" > /dev/null
jq -e '.prompts | all(.id and .title and .category and .version and .content)' "$OUTPUT" > /dev/null
jq -e '.prompts | all(.id | test("^[a-z0-9-]+$"))' "$OUTPUT" > /dev/null
jq -e '.prompts | all(.category | IN("coding", "writing", "analysis", "creative"))' "$OUTPUT" > /dev/null
```

**Note:** True JSON Schema draft 2020-12 validation against the schema file requires `ajv-cli` (Node.js) or `check-jsonschema` (Python). The build script uses jq assertions for structural validation. The schema file itself (`schema/prompt.schema.json`) is the canonical contract for CI and documentation — it documents what is valid even if the bash build script validates via assertions rather than native schema validation.

### Anti-Patterns to Avoid

- **`declare -A` for associative arrays:** Breaks macOS bash 3.2. Use string accumulation + `sort | uniq -d`.
- **`cut -d: -f2` for title parsing:** Drops everything after the first colon. Use sed prefix-strip instead.
- **String concatenation to build JSON:** `json="{\"id\":\"$id\"}"` — breaks on any special character. Always use `jq --arg`.
- **`echo` with backslash content:** `echo` behavior is shell-dependent. Use `printf '%s\n'` for portable output.
- **`grep '^title:' | cut -d: -f2`:** Loses "Swift Code Review: Advanced" → "Swift Code Review". Use sed prefix-strip.
- **Not stripping leading blank lines from body:** Prompt content starts with `\n`, which Flycut pastes verbatim, producing a leading blank line in the UI.
- **`/bin/sh` shebang:** Use `#!/bin/bash` explicitly — `/bin/sh` on macOS is `dash`, not bash.

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---------|-------------|-------------|-----|
| JSON string escaping | String concatenation with `\"` replacement | `jq -Rs '.'` | Backslashes, embedded quotes, control characters, Unicode all handled correctly |
| Multiline string → JSON | Manual `\n` substitution with sed | `jq -Rs '.'` | `\r\n` vs `\n`, tab characters, null bytes — all edge cases |
| JSON object construction | `echo "{\"id\":\"$id\"}"` | `jq -n --arg id "$id" ...` | Injection-safe; works with any string content |
| YAML parsing | Custom parser with IFS and read | sed/awk for flat key-value frontmatter | Real YAML has quoting, multiline, anchors; sed sufficient for known-flat format |
| ID uniqueness tracking | Shell functions with grep on accumulator | `sort \| uniq -d` | POSIX standard; no off-by-one in accumulator management |

**Key insight:** `jq` is the difference between a build script that works on toy examples and one that handles real-world content. Markdown prompts will contain code examples (backslashes, quotes), URLs (special characters), and multiline formatting. All of these break hand-rolled JSON construction.

## Common Pitfalls

### Pitfall 1: macOS bash 3.2 and `declare -A`

**What goes wrong:** Build script works in CI (Ubuntu, bash 5.x) but fails immediately on macOS with `declare: -A: invalid option`.

**Why it happens:** Apple ships bash 3.2 at `/bin/bash` due to GPLv3 licensing freeze. `declare -A` (associative arrays) was added in bash 4.0.

**How to avoid:** Use indexed arrays (`declare -a`, available in bash 3.2) and string accumulation patterns. For duplicate detection, collect all IDs into a string, then use `sort | uniq -d`.

**Warning signs:** Any `declare -A` in the script. Test locally on macOS with `bash --version` confirming version 3.2.

### Pitfall 2: Title Truncated at First Colon

**What goes wrong:** A title like `"Swift Code Review: Advanced"` produces `"Swift Code Review"` in the output JSON.

**Why it happens:** `grep '^title:' | cut -d: -f2` splits on all colons, not just the delimiter after the key name.

**How to avoid:** Use sed to strip only the `title:` prefix, then separately strip surrounding quotes if present:
```bash
title=$(... | grep '^title:' | sed 's/^title:[[:space:]]*//' | sed 's/^"\(.*\)"$/\1/')
```

**Warning signs:** Include a test fixture with a colon in the title (e.g., `title: "Code Review: Swift"`) and verify the full value appears in `prompts.json`.

### Pitfall 3: Leading Blank Line in Prompt Body

**What goes wrong:** All prompts in `prompts.json` have `"content": "\nReview this..."` — content starts with a newline.

**Why it happens:** Authors put a blank line after the closing `---` for readability. The `awk` body extractor captures this blank line as the first line of content.

**How to avoid:** After extracting the body, strip leading blank lines with:
```bash
body=$(printf '%s\n' "$body" | sed '/./,$!d')
```

**Warning signs:** `jq '.prompts[] | select(.content | startswith("\n"))' prompts.json` returns any results.

### Pitfall 4: `set -e` Exits on grep No-Match

**What goes wrong:** With `set -euo pipefail`, `grep '^category:' "$frontmatter"` exits the entire script with code 1 when the optional `category` field is absent (grep exit code 1 = no match).

**Why it happens:** `set -e` treats any non-zero exit code as a fatal error, including grep's "no match found."

**How to avoid:** Use `|| true` or `|| echo ""` for optional-field grep commands:
```bash
fm_category=$(printf '%s\n' "$frontmatter" | grep '^category:' | sed '...' || true)
```

**Warning signs:** Script exits silently when processing a prompt without a `category:` frontmatter field.

### Pitfall 5: `printf '%s\n'` vs `echo` for Body Content

**What goes wrong:** `echo "$body"` on some shells interprets escape sequences (`\n` in body content becomes a newline within the echo output, not a literal backslash-n). On others it does not.

**Why it happens:** `echo` behavior with backslashes is shell-dependent (POSIX leaves it implementation-defined).

**How to avoid:** Always use `printf '%s\n' "$body"` for body content output. This is portable and does not interpret backslash sequences.

**Warning signs:** Code samples in prompts (which contain `\n` in string literals) appearing with unexpected newlines in the JSON output.

### Pitfall 6: `jq -Rs '.'` Captures Trailing Newline

**What goes wrong:** `echo "$body" | jq -Rs '.'` captures the trailing newline added by `echo`, so the JSON string ends with `\n`.

**Why it happens:** `jq -Rs '.'` reads all of stdin as a raw string, including the trailing newline that `echo` appends.

**How to avoid:** Use `printf '%s' "$body" | jq -Rs '.'` (no trailing newline in printf without `\n` format). Or accept the trailing `\n` as harmless for prompt display (Flycut trims whitespace at paste time — verify this is the case).

**Warning signs:** All content fields ending with `\n` in `prompts.json` when inspected with `jq '.prompts[0].content'`.

## Code Examples

Verified patterns from the design document and POSIX documentation:

### catalog.yaml Structure

```yaml
# catalog.yaml
# Source: dist/PROMPTS-REPO-DESIGN.md

version: 1

categories:
  - coding
  - writing
  - analysis
  - creative
```

### Reading catalog.yaml in bash

```bash
# Source: dist/PROMPTS-REPO-DESIGN.md pseudocode

CATALOG_VERSION=$(grep '^version:' "$CATALOG" | awk '{print $2}')

# Read categories as newline-delimited string
VALID_CATEGORIES=$(grep '^ *- ' "$CATALOG" | sed 's/^ *- //')
```

### Category Validation

```bash
# Source: dist/PROMPTS-REPO-DESIGN.md pseudocode

if ! printf '%s\n' "$VALID_CATEGORIES" | grep -qx "$category"; then
    echo "ERROR: Invalid category '$category' in $file" >&2
    exit 1
fi
```

### Complete Prompt File (Test Prompt)

```markdown
---
title: "Swift Code Review"
version: 1
description: "Review Swift code for correctness, safety, and idiomatic style"
variables: ["clipboard"]
---

Review this Swift code for correctness, safety, and style. Focus on:

1. **Correctness** — logic errors, off-by-one, nil handling, race conditions
2. **Safety** — force unwraps, unowned references, unchecked casts
3. **Style** — naming conventions, Swift idioms, unnecessary complexity
4. **Performance** — obvious N+1 patterns, unnecessary allocations

Code to review:

{{clipboard}}

Provide specific line-by-line feedback with suggested fixes.
```

### JSON Schema (Complete)

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "type": "object",
  "required": ["version", "prompts"],
  "additionalProperties": false,
  "properties": {
    "version": {
      "type": "integer",
      "minimum": 1,
      "description": "Catalog-level version number"
    },
    "prompts": {
      "type": "array",
      "items": {
        "type": "object",
        "required": ["id", "title", "category", "version", "content"],
        "additionalProperties": false,
        "properties": {
          "id": {
            "type": "string",
            "pattern": "^[a-z0-9-]+$",
            "description": "Stable slug derived from filename"
          },
          "title": {
            "type": "string",
            "minLength": 1,
            "description": "Display title"
          },
          "category": {
            "type": "string",
            "enum": ["coding", "writing", "analysis", "creative"],
            "description": "Prompt category"
          },
          "version": {
            "type": "integer",
            "minimum": 1,
            "description": "Per-prompt version for sync"
          },
          "content": {
            "type": "string",
            "minLength": 1,
            "description": "Prompt content with optional {{variable}} placeholders"
          }
        }
      }
    }
  }
}
```

### Build Script Skeleton (Production-Safe)

```bash
#!/bin/bash
# Source: dist/PROMPTS-REPO-DESIGN.md pseudocode, adapted for macOS bash 3.2

set -euo pipefail

SCRIPT_DIR="$(cd "$(dirname "$0")" && pwd)"
REPO_ROOT="$(dirname "$SCRIPT_DIR")"
PROMPTS_DIR="$REPO_ROOT/prompts"
OUTPUT="$REPO_ROOT/prompts.json"
CATALOG="$REPO_ROOT/catalog.yaml"

# Read catalog metadata
CATALOG_VERSION=$(grep '^version:' "$CATALOG" | awk '{print $2}')
VALID_CATEGORIES=$(grep '^ *- ' "$CATALOG" | sed 's/^ *- //')

# Two-pass approach:
# Pass 1: Collect all IDs, detect duplicates upfront (POSIX, no declare -A)
ALL_IDS=""
for file in $(find "$PROMPTS_DIR" -name '*.md' -type f | sort); do
    id=$(basename "$file" .md)
    ALL_IDS="${ALL_IDS}${id}
"
done

DUPS=$(printf '%s' "$ALL_IDS" | sort | uniq -d)
if [ -n "$DUPS" ]; then
    echo "ERROR: Duplicate prompt IDs detected: $(printf '%s' "$DUPS" | tr '\n' ' ')" >&2
    exit 1
fi

# Pass 2: Build JSON for each file
declare -a PROMPT_JSONS=()   # declare -a (indexed) is safe on bash 3.2

for file in $(find "$PROMPTS_DIR" -name '*.md' -type f | sort); do
    id=$(basename "$file" .md)
    dir_category=$(basename "$(dirname "$file")")

    # Extract frontmatter block (lines between first and second ---)
    frontmatter=$(awk '/^---$/{count++; if(count==1){next}; if(count==2){exit}} count==1{print}' "$file")

    # Parse required fields
    title=$(printf '%s\n' "$frontmatter" | grep '^title:' | sed 's/^title:[[:space:]]*//' | sed 's/^"\(.*\)"$/\1/')
    version=$(printf '%s\n' "$frontmatter" | grep '^version:' | awk '{print $2}')

    # Parse optional category override (|| true prevents set -e exit on no-match)
    fm_category=$(printf '%s\n' "$frontmatter" | grep '^category:' | sed 's/^category:[[:space:]]*//' | sed 's/^"\(.*\)"$/\1/' || true)

    # Resolve category
    category="${fm_category:-$dir_category}"

    # Validate required fields
    if [ -z "$title" ]; then
        echo "ERROR: Missing title in $file" >&2; exit 1
    fi
    if [ -z "$version" ]; then
        echo "ERROR: Missing version in $file" >&2; exit 1
    fi
    if ! printf '%s\n' "$VALID_CATEGORIES" | grep -qx "$category"; then
        echo "ERROR: Invalid category '$category' in $file" >&2; exit 1
    fi

    # Extract body (everything after second ---) and strip leading blank lines
    body=$(awk 'BEGIN{n=0} /^---$/{n++; if(n==2){found=1; next}} found{print}' "$file" \
          | sed '/./,$!d')

    # JSON-encode body safely (handles newlines, quotes, backslashes)
    content_json=$(printf '%s' "$body" | jq -Rs '.')

    # Build prompt JSON object (all fields explicit — no extras)
    prompt_json=$(jq -n \
        --arg     id       "$id" \
        --arg     title    "$title" \
        --arg     category "$category" \
        --argjson version  "$version" \
        --argjson content  "$content_json" \
        '{id: $id, title: $title, category: $category, version: $version, content: $content}')

    PROMPT_JSONS+=("$prompt_json")
done

# Assemble final JSON
printf '%s\n' "${PROMPT_JSONS[@]}" \
    | jq -s --argjson v "$CATALOG_VERSION" '{version: $v, prompts: .}' > "$OUTPUT"

COUNT=$(jq '.prompts | length' "$OUTPUT")
echo "Built $OUTPUT: $COUNT prompts, catalog version $CATALOG_VERSION"
```

### Verify Output After Build

```bash
# Verify JSON is parseable
jq empty prompts.json

# Verify structure
jq -e '.version' prompts.json > /dev/null
jq -e '.prompts | length > 0' prompts.json > /dev/null

# Verify no leading blank lines in content
jq '.prompts[] | select(.content | startswith("\n")) | .id' prompts.json
# Should produce no output

# Verify {{clipboard}} passthrough in test prompt
jq '.prompts[] | select(.content | contains("{{clipboard}}")) | .id' prompts.json
# Should show "code-review-swift"

# Verify id pattern
jq -e '.prompts | all(.id | test("^[a-z0-9-]+$"))' prompts.json > /dev/null
```

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|--------------|------------------|--------------|--------|
| `declare -A` for duplicate tracking | `sort \| uniq -d` POSIX pattern | bash 3.2 on macOS (frozen since 2007) | macOS users get working local builds |
| `echo "$var"` for content output | `printf '%s\n' "$var"` | POSIX standard (always preferred) | Backslash handling is portable |
| String concatenation for JSON | `jq -n --arg ...` | jq 1.5+ (2015) | Correct escaping of all content types |
| `cut -d: -f2` for YAML value extraction | sed prefix-strip pattern | Known gotcha since YAML with colons became common | Titles with colons preserved correctly |

**Deprecated/outdated:**
- `declare -A` in bash build scripts: Cannot be used — macOS bash 3.2 incompatibility
- Hand-rolled JSON escaping: Replaced by `jq` in all cases where correctness matters

## Open Questions

1. **Trailing newline behavior in `jq -Rs '.'`**
   - What we know: `printf '%s' "$body"` suppresses the trailing newline that echo adds; `jq -Rs '.'` captures exactly what it receives
   - What's unclear: Whether Flycut trims trailing whitespace from `content` at paste time — if yes, trailing `\n` is harmless; if no, it adds a blank line after pasted content
   - Recommendation: Use `printf '%s' "$body"` (no trailing newline) to be safe; this is the correct approach regardless of Flycut behavior

2. **`catalog.yaml` version starting value**
   - What we know: CONTEXT.md specifies `version: 1` for the initial catalog
   - What's unclear: Whether catalog version should be bumped when the Phase 1 test prompt is added, or only when all 23 seed prompts are added in Phase 2
   - Recommendation: Start at `version: 1` in Phase 1; Phase 2 plan should specify when to bump

## Validation Architecture

### Test Framework

| Property | Value |
|----------|-------|
| Framework | bash + jq assertions (no test framework required) |
| Config file | none — tests are inline build script verification commands |
| Quick run command | `bash scripts/build.sh && jq -e '.prompts \| length > 0' prompts.json` |
| Full suite command | `bash scripts/build.sh && bash scripts/build.sh --validate` (or equivalent validation block) |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|--------|----------|-----------|-------------------|-------------|
| BILD-01 | build.sh produces prompts.json | smoke | `bash scripts/build.sh && test -f prompts.json` | ❌ Wave 0 |
| BILD-05 | Duplicate ID causes non-zero exit | unit | Create duplicate test fixture; `bash scripts/build.sh; test $? -ne 0` | ❌ Wave 0 |
| BILD-06 | Missing title causes non-zero exit | unit | Create missing-title fixture; `bash scripts/build.sh; test $? -ne 0` | ❌ Wave 0 |
| BILD-07 | Invalid category causes non-zero exit | unit | Create invalid-category fixture; `bash scripts/build.sh; test $? -ne 0` | ❌ Wave 0 |
| BILD-09 | Multiline content is JSON-escaped | unit | `jq '.prompts[0].content \| contains("\\n")' prompts.json` | ❌ Wave 0 |
| BILD-10 | Works on macOS bash 3.2 | manual | `bash --version` confirm 3.2; `bash scripts/build.sh` runs without error | manual |
| BILD-11 | Leading blank lines stripped | unit | `jq -e '.prompts[] \| .content \| startswith("\\n") \| not' prompts.json` | ❌ Wave 0 |
| PRMT-05 | `{{clipboard}}` passes through verbatim | unit | `jq '.prompts[] \| select(.content \| contains("{{clipboard}}")) \| .id' prompts.json` | ❌ Wave 0 |
| SCHM-01 | prompts.json validates against schema | integration | `jq -e '.' prompts.json && jq -e '.prompts \| all(.id \| test("^[a-z0-9-]+$"))' prompts.json` | ❌ Wave 0 |
| JSON-02 | Each prompt has exactly 5 fields | unit | `jq -e '.prompts[] \| keys \| length == 5' prompts.json` | ❌ Wave 0 |

### Sampling Rate

- **Per task commit:** `bash scripts/build.sh && jq -e '.prompts | length > 0' prompts.json`
- **Per wave merge:** Full validation block (all jq assertions above)
- **Phase gate:** All jq assertions green + manual macOS bash 3.2 verification before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `scripts/build.sh` — the build script itself (core Phase 1 deliverable)
- [ ] `catalog.yaml` — required before build.sh can run
- [ ] `prompts/coding/code-review-swift.md` — test prompt for pipeline verification
- [ ] `schema/prompt.schema.json` — schema file (SCHM-01, SCHM-02)
- [ ] Error fixture prompts for negative-case testing (duplicate ID, missing title, invalid category)

*(All gaps are Phase 1 deliverables — this is a greenfield phase)*

## Sources

### Primary (HIGH confidence)

- `dist/PROMPTS-REPO-DESIGN.md` — Complete system specification; pseudocode, exact JSON structures, schema definition; read in full during this research session
- `.planning/phases/01-build-foundation/01-CONTEXT.md` — Locked implementation decisions; macOS bash 3.2 constraint, jq -Rs approach, POSIX duplicate detection, body extraction requirements
- `.planning/REQUIREMENTS.md` — Authoritative requirement IDs and descriptions for all REPO, PRMT, BILD, JSON, SCHM requirements
- `.planning/research/SUMMARY.md` — Prior project research with pitfall analysis; macOS bash 3.2, colon-in-title, leading blank line pitfalls all verified

### Secondary (MEDIUM confidence)

- POSIX specification for `sed`, `awk`, `grep`, `sort`, `uniq` — behavior of `/./,$!d` pattern for leading blank line strip is POSIX-defined
- bash 3.2 release notes — `declare -a` (indexed) available; `declare -A` (associative) added in bash 4.0

### Tertiary (LOW confidence)

- Community reports of macOS bash 3.2 `declare -A` failures — confirmed by multiple issue threads cited in SUMMARY.md

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — all tools are POSIX standard or widely pre-installed; design doc specifies exact tool choices
- Architecture: HIGH — design doc provides complete pseudocode; patterns are well-understood bash scripting
- Pitfalls: HIGH — pitfalls are derived from direct analysis of the design doc pseudocode against known bash/macOS constraints; not speculative

**Research date:** 2026-03-11
**Valid until:** 2026-06-11 (stable domain — bash portability patterns don't change; jq 1.6 API stable)
