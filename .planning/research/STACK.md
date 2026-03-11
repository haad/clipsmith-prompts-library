# Stack Research

**Domain:** Markdown-to-JSON static build pipeline with GitHub Pages deployment
**Researched:** 2026-03-11
**Confidence:** HIGH

## Recommended Stack

### Core Technologies

| Technology | Version | Purpose | Why Recommended |
|------------|---------|---------|-----------------|
| bash | 5.x (system) | Build script runtime | Preinstalled on every macOS/Ubuntu CI environment; `set -euo pipefail` makes scripts fail-fast and safe; no dependency installation needed — exactly what the zero-dependency constraint requires |
| jq | 1.6 (Ubuntu apt) | JSON construction and validation | The only safe way to build JSON from shell variables; `jq -Rs '.'` correctly escapes newlines, tabs, quotes, and Unicode in prompt content; `--arg`/`--argjson` flags prevent injection; `jq -n` builds objects cleanly — do NOT hand-roll JSON string concatenation |
| sed + awk + grep | POSIX (system) | YAML frontmatter parsing | Standard tools available on macOS and Ubuntu without any installation; sufficient for flat key-value YAML frontmatter (which is all this project uses); complex YAML structures would require a real parser, but `title:`, `version:`, `category:` are trivially parseable |
| actions/checkout | v4 | Git checkout in CI | Current stable version; v3 reached EOL; uses Node 20 runtime (v3 used Node 16 which is EOL in GitHub's infrastructure) |
| peaceiris/actions-gh-pages | v4 | Deploy to gh-pages branch | Push-to-branch deployment model matches this project's needs exactly: write `prompts.json` on main, deploy single file to `gh-pages` branch; simpler than the official `actions/deploy-pages` which requires an artifact upload step |

### Supporting Libraries

| Library | Version | Purpose | When to Use |
|---------|---------|---------|-------------|
| jq (validation mode) | 1.6 | Post-build JSON validation | Use `jq empty` to verify parse-valid JSON; use `jq -e` assertions to verify schema conformance without a schema validator binary; sufficient for all required checks |
| yamllint | preinstalled on ubuntu-latest | Optional: lint `catalog.yaml` | Only needed if catalog.yaml grows in complexity; for the current simple structure, the build script's grep-based parsing serves as implicit validation |

### Development Tools

| Tool | Purpose | Notes |
|------|---------|-------|
| bash scripts/build.sh | Local build | Developers run this to regenerate `prompts.json` before committing; requires `jq` locally (installable via `brew install jq` on macOS, `apt install jq` on Ubuntu) |
| jq (local) | Manual inspection | `jq '.prompts | length' prompts.json`, `jq '.prompts[] | select(.id == "fix-bug")' prompts.json` — standard debugging workflow |
| GitHub Actions | CI/CD | Triggered on push to main affecting `prompts/`, `catalog.yaml`, or `scripts/build.sh`; runs build + validate + deploy in a single job |

## Installation

```bash
# No npm/pip/gem — this project has zero package dependencies.

# macOS development setup (jq is the only non-system tool needed)
brew install jq

# Ubuntu/Debian (CI already has this via apt)
sudo apt-get install -y jq

# Make build script executable after creating it
chmod +x scripts/build.sh
```

## Alternatives Considered

| Recommended | Alternative | When to Use Alternative |
|-------------|-------------|-------------------------|
| peaceiris/actions-gh-pages@v4 | actions/deploy-pages (official GitHub action) | Use `actions/deploy-pages` when you need OIDC-based authentication for compliance/audit requirements, or when deploying a full site with GitHub Pages environment integration. For this project's single-file use case, `peaceiris/actions-gh-pages` is less boilerplate (no separate upload-artifact step required). |
| sed/awk/grep frontmatter parsing | yq (mikefarah/yq) | Use `yq` if frontmatter fields become nested, multi-value arrays, or contain special YAML syntax (multiline scalars, anchors). For flat `key: value` frontmatter, yq adds unnecessary dependency complexity on CI. |
| sed/awk/grep frontmatter parsing | Python one-liner with PyYAML | Use Python if the CI environment guarantees Python (it does, ubuntu-latest has Python 3), but introduces risk of YAML edge cases being "handled" silently instead of failing loudly. |
| jq for JSON construction | printf/echo string concatenation | Never use string concatenation for JSON with untrusted/multiline content. A prompt body containing `"`, `\`, or newlines will produce malformed JSON that silently corrupts the sync. jq guarantees correct escaping. |
| jq for JSON construction | python -c "import json" | Valid fallback if jq is unavailable, but ubuntu-latest has jq 1.6 preinstalled, making Python unnecessary. |
| jq -e assertions for validation | ajv-cli JSON Schema validator | Use `ajv-cli` if the schema evolves to use complex validation rules ($ref, oneOf, patterns, etc.). For the current schema with basic type/required checks, `jq -e` assertions are sufficient and require no Node.js installation. |

## What NOT to Use

| Avoid | Why | Use Instead |
|-------|-----|-------------|
| Node.js / npm build tools (Vite, webpack, rollup) | This project produces a static JSON file from Markdown — it has no JavaScript bundle, no assets, no transpilation. Adding Node.js means requiring `npm install` on every CI run and every contributor machine. The design spec explicitly prohibits this. | bash + jq |
| Python scripts (gray-matter, frontmatter packages) | Python is available on CI but adds a language boundary. The frontmatter format is intentionally simple — four flat keys. Using Python for this creates a maintenance surface that outweighs any benefit. | sed/awk/grep |
| Jekyll (GitHub Pages default) | GitHub Pages can run Jekyll automatically on the `gh-pages` branch, but Jekyll would try to process `prompts.json` as a site, adding unnecessary complexity and requiring `.nojekyll` to disable it anyway. The `peaceiris/actions-gh-pages` action creates `.nojekyll` by default. | peaceiris/actions-gh-pages with `.nojekyll` |
| yq (mikefarah/yq or kislyuk/yq) | Two incompatible tools share the name `yq` — the Go version (mikefarah) and the Python wrapper around jq (kislyuk). This naming collision is a real CI footgun. Since `jq` is already required, using it for everything avoids the confusion. | jq |
| Hand-rolled JSON string escaping | `content="$(cat file)"` then `"content": "$content"` in a JSON string will break on any prompt containing double quotes, backslashes, or newlines. This is guaranteed to happen with real prompt content. Produces silent data corruption. | `jq -Rs '.'` piped through `jq --argjson` |
| actions/checkout@v3 | Uses Node 16 which is EOL in GitHub Actions infrastructure. GitHub has issued warnings and will eventually drop support. | actions/checkout@v4 |
| Committing to gh-pages manually | Manual branch manipulation is error-prone and doesn't produce a clean deployment history. | peaceiris/actions-gh-pages@v4 via CI |
| `include_files` option on peaceiris/actions-gh-pages | This option is not supported in any released version of the action. The design doc notes this as a "Note" — the correct approach is to copy only the desired files to a `_deploy/` staging directory and set `publish_dir: ./_deploy`. | Staging directory pattern (see below) |

## Stack Patterns by Variant

**For deploying only `prompts.json` to gh-pages (not the entire repo):**
- Use a staging directory: `mkdir -p _deploy && cp prompts.json _deploy/`
- Add a minimal `index.html` to `_deploy/` for human browsing
- Set `publish_dir: ./_deploy` in the deploy step
- Because the `include_files` input on `peaceiris/actions-gh-pages` is not implemented — attempting to use it will deploy everything

**For jq not available on a runner:**
- Add `run: sudo apt-get install -y jq` as an explicit step before the build
- Ubuntu 22.04 and 24.04 have jq 1.6 via apt; this step is idempotent if jq is already present
- macOS runners have jq 1.7+ via Homebrew preinstall

**For frontmatter values containing colons (e.g., `title: "Foo: Bar"`):**
- The `grep '^title:'` + `sed` approach correctly handles quoted values when the sed pattern strips surrounding quotes
- Use the pattern: `sed 's/^title: *"\?\(.*\)"\?$/\1/'` — strips optional surrounding double-quotes
- Test with: `echo 'title: "Swift Code Review"'` piped through the sed expression before shipping

**For validating prompts without jq installed locally:**
- Run `python3 -c "import json; json.load(open('prompts.json'))"` as a fallback parse check
- This validates JSON syntax but not the schema; install jq for full validation

## Version Compatibility

| Component | Compatible With | Notes |
|-----------|-----------------|-------|
| jq 1.6 (ubuntu apt) | bash 5.x, POSIX sed/awk | jq 1.6 supports all required features: `-Rs`, `-n`, `--arg`, `--argjson`, `-e`, `empty`. No jq 1.7+ features needed. |
| peaceiris/actions-gh-pages@v4 | actions/checkout@v4, ubuntu-latest | v4 requires Node 20 (provided by the runner). Do not pin to v3 (Node 16 EOL). |
| actions/checkout@v4 | ubuntu-latest, ubuntu-22.04, ubuntu-24.04 | Compatible with all current GitHub-hosted Ubuntu runners. |
| bash `declare -A` (associative arrays) | bash 4.0+ | Used for duplicate ID detection (`declare -A SEEN_IDS`). Bash 4+ is standard on Ubuntu. On macOS, the system bash is 3.2 (no associative arrays) — contributors must use `brew install bash` or run via `bash scripts/build.sh` explicitly if needed for local dev. This is a real pitfall: see PITFALLS.md. |

## CI Workflow Skeleton (Annotated)

The following pattern is the validated approach for this project:

```yaml
name: Build and Deploy Prompts

on:
  push:
    branches: [main]
    paths:
      - 'prompts/**'       # Only trigger when prompt files change
      - 'catalog.yaml'     # Or catalog metadata changes
      - 'scripts/build.sh' # Or the build script itself changes
  workflow_dispatch:       # Allow manual re-run from GitHub UI

permissions:
  contents: write          # Required by peaceiris/actions-gh-pages to push to gh-pages branch

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      # jq 1.6 is preinstalled on ubuntu-latest via apt.
      # This step is kept explicit to document the dependency and
      # make the workflow self-describing. It's idempotent.
      - name: Install jq
        run: sudo apt-get install -y jq

      - name: Build prompts.json
        run: bash scripts/build.sh

      # jq -e exits with code 1 if expression is false/null.
      # This is the validation layer — complements the build script's
      # input validation with output structure verification.
      - name: Validate output
        run: |
          jq empty prompts.json
          jq -e '.version | type == "number"' prompts.json > /dev/null
          jq -e '.prompts | length > 0' prompts.json > /dev/null
          jq -e '.prompts | all(.id and .title and .category and .version and .content)' prompts.json > /dev/null
          TOTAL=$(jq '.prompts | length' prompts.json)
          UNIQUE=$(jq '[.prompts[].id] | unique | length' prompts.json)
          [ "$TOTAL" = "$UNIQUE" ] || { echo "ERROR: Duplicate IDs in output"; exit 1; }
          echo "Validation passed: $TOTAL prompts"

      # Stage only prompts.json (and a minimal index.html) for deployment.
      # Do NOT use include_files — it is not implemented in any released version.
      - name: Stage deployment files
        run: |
          mkdir -p _deploy
          cp prompts.json _deploy/
          echo '<html><body><p>Flycut Prompts Library. <a href="prompts.json">prompts.json</a></p></body></html>' > _deploy/index.html

      - name: Deploy to gh-pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./_deploy
          publish_branch: gh-pages
          force_orphan: true   # Keep gh-pages history clean — only latest commit needed
```

**Why `force_orphan: true`:** The gh-pages branch doesn't need history. Each deployment is a complete replacement of the published content. Orphan commits keep the branch size minimal and avoid git history bloat from binary JSON diffs.

**Why `paths:` filter:** Without it, every push to main (including README edits, `.gitignore` changes) triggers a full redeploy. The paths filter ensures deploys only happen when the build output could actually change.

## Key Constraint: macOS bash 3.2 vs bash 4+

The build script uses `declare -A` for associative arrays (duplicate ID detection). macOS ships bash 3.2 (GPLv2, not updated since 2007). Bash 4+ is required for `declare -A`.

**Impact:** `bash scripts/build.sh` on macOS stock bash will fail with a syntax error.

**Solutions:**
1. Use `#!/usr/bin/env bash` as the shebang and document that contributors need `brew install bash`
2. Or rewrite duplicate detection using `sort | uniq -d` approach (POSIX-compatible, no associative arrays needed)
3. **Recommended:** Use the `sort | uniq` approach — it works on macOS 3.2, Ubuntu, and CI with zero setup

```bash
# POSIX-compatible duplicate ID detection (no declare -A needed)
find "$PROMPTS_DIR" -name '*.md' -type f | \
  xargs -I{} basename {} .md | sort | \
  uniq -d | \
  while read -r dup; do
    echo "ERROR: Duplicate id '$dup'" >&2; exit 1
  done
```

## Sources

- [peaceiris/actions-gh-pages GitHub repo](https://github.com/peaceiris/actions-gh-pages) — current version, inputs, known limitations (HIGH confidence)
- [actions/runner-images issue #9550](https://github.com/actions/runner-images/issues/9550) — confirmed jq 1.6 on ubuntu-latest from apt; update to 1.7.1 closed as not planned (HIGH confidence)
- [jq 1.8 manual](https://jqlang.org/manual/) — `-Rs`, `--arg`, `--argjson`, `-e`, `empty` usage (HIGH confidence)
- [GitHub Actions workflow syntax docs](https://docs.github.com/en/actions/reference/workflow-syntax-for-github-pages) — `paths:` filter, permissions, `workflow_dispatch` (HIGH confidence)
- [MegaLinter discussion on YAML frontmatter linting](https://github.com/oxsecurity/megalinter/discussions/4066) — confirmed no native yamllint support for extracting markdown frontmatter (MEDIUM confidence)
- [jq -Rs slurp issue #2415](https://github.com/jqlang/jq/issues/2415) — multiline raw-input behavior confirmed; no bugs affecting this use case (MEDIUM confidence)

---
*Stack research for: Markdown-to-JSON build pipeline, GitHub Pages deployment*
*Researched: 2026-03-11*
