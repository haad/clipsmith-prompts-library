# Pitfalls Research

**Domain:** Markdown-to-JSON build pipeline with bash YAML frontmatter parsing and GitHub Pages deployment
**Researched:** 2026-03-11
**Confidence:** HIGH — pitfalls drawn from direct analysis of the design document spec combined with verified community patterns

---

## Critical Pitfalls

### Pitfall 1: macOS bash 3.2 Fails on `declare -A` Associative Arrays

**What goes wrong:**
The design document pseudocode uses `declare -A SEEN_IDS=()` for duplicate ID detection. On macOS, the system bash is version 3.2 (due to Apple's GPLv2 licensing freeze). Associative arrays (`declare -A`) were introduced in bash 4.0. Running the build script locally on macOS produces `declare: -A: invalid option` and the script exits immediately — before processing any prompts.

**Why it happens:**
macOS ships bash 3.2 at `/bin/bash` as a deliberate licensing choice. Developers writing bash 4+ features (associative arrays, `mapfile`, etc.) on Linux CI assume the same capability exists on macOS without testing. The shebang `#!/bin/bash` on macOS resolves to 3.2, not a Homebrew-installed 4.x or 5.x.

**How to avoid:**
Replace the associative array duplicate-ID check with a POSIX-compatible alternative. Sort the ID list and use `uniq -d` to detect duplicates, or build a plain string of seen IDs and use `grep -w` for membership tests. This approach works on bash 3.2, bash 5.x, zsh, and CI Ubuntu runners identically.

```bash
# Instead of declare -A SEEN_IDS:
SEEN_IDS=""

# After extracting id:
if echo "$SEEN_IDS" | grep -qw "$filename"; then
    echo "ERROR: Duplicate id '$filename'" >&2
    exit 1
fi
SEEN_IDS="$SEEN_IDS $filename"
```

Alternatively, after collecting all IDs, detect duplicates at the end with: `echo "$ALL_IDS" | sort | uniq -d`.

**Warning signs:**
- Script works in CI (Ubuntu) but fails when run locally on macOS with `declare: -A: invalid option`
- Contributors report the build script "doesn't work" on their machines
- `bash --version` shows version 3.x on the contributor's machine

**Phase to address:**
Build script implementation phase (when `scripts/build.sh` is written). Test the script on macOS before considering it done.

---

### Pitfall 2: Quoted YAML Title Values with Colons Silently Lose Content

**What goes wrong:**
A `sed`/`grep` frontmatter parser extracts `title:` values using a pattern like:
```bash
grep '^title:' | sed 's/^title: *"\{0,1\}\(.*\)"\{0,1\}$/\1/'
```
When a title contains a colon — e.g., `title: "Swift Code Review: Best Practices"` — the regex may match only the portion before the colon (treating `:` as a YAML key-value separator), or strip the surrounding quotes and leave the colon in the output, breaking downstream JSON generation. In the worst case, the title silently becomes `"Swift Code Review"` with no error.

**Why it happens:**
YAML uses `:` followed by a space as the key-value separator. When parsing line-by-line with `sed`, a colon in the value is indistinguishable from a nested key unless the parser understands YAML quoting rules. Simple `awk '{print $2}'` or `cut -d: -f2` approaches drop everything after the first colon.

**How to avoid:**
Use `jq` to produce the final JSON — pass field values as `--arg` parameters. This sidesteps title escaping entirely in the output. For the extraction step, use a sed pattern that captures everything after `title: ` (optionally stripping one leading/trailing double-quote), and rely on `jq --arg title "$title"` for safe JSON encoding. Test with a title that includes a colon before declaring the parser complete.

```bash
title=$(echo "$frontmatter" | grep '^title:' | sed 's/^title:[[:space:]]*//' | sed 's/^"\(.*\)"$/\1/')
# Then pass to jq with --arg (jq handles all escaping):
jq -n --arg title "$title" '{"title": $title}'
```

**Warning signs:**
- Built `prompts.json` contains truncated titles (e.g., `"title": "Swift Code Review"` instead of `"title": "Swift Code Review: Best Practices"`)
- Titles with apostrophes, slashes, or special characters appear garbled in the JSON output
- The build exits successfully but `jq '.prompts[].title'` shows mangled values

**Phase to address:**
Build script implementation phase. Include at least one test prompt with a colon in its title as a verification fixture.

---

### Pitfall 3: Leading Newline in Prompt Content from Body Extraction

**What goes wrong:**
The `awk` body extraction pattern (everything after the second `---` delimiter) captures a leading blank line from the Markdown file structure. The typical file layout is:
```
---
title: "..."
---

Prompt content starts here.
```
The blank line after the closing `---` becomes the first character of `body`. After `jq -Rs '.'` encodes it as JSON, the `content` field starts with `\n\n` or `\n` rather than the prompt text directly. Flycut pastes this leading whitespace into the user's document.

**Why it happens:**
The awk range `awk 'BEGIN{n=0} /^---$/{n++; if(n==2){found=1; next}} found{print}'` outputs every line after the second `---` delimiter, including blank lines that authors naturally add for visual separation in the source file. This is standard Markdown convention — the blank line after frontmatter is universal and expected.

**How to avoid:**
Strip leading and trailing whitespace from the extracted body before JSON-encoding. A portable approach:

```bash
# Strip leading/trailing blank lines:
body=$(awk '...' "$file" | sed '/./,$!d' | sed -e :a -e '/^\n*$/{$d;N;ba}')
```

Or more simply, trim the leading newline explicitly:
```bash
body=$(awk '...' "$file" | sed '1{/^$/d}')
```

Decide and document the canonical form: content should start at the first non-blank line of the Markdown body.

**Warning signs:**
- `jq '.prompts[0].content'` output begins with `"\n\n"` or `"\n"`
- Flycut pastes content with an unexpected blank line before the prompt text
- Prompt content looks correct in the `.md` file but has leading whitespace in the JSON

**Phase to address:**
Build script implementation phase. Verify content field values using `jq '.prompts[] | {id, content_start: .content[0:20]}'` after building.

---

### Pitfall 4: Version Not Bumped on Content Edit — Silent Sync Failure

**What goes wrong:**
A contributor edits prompt content and opens a PR. The build succeeds. The PR is merged. CI deploys an updated `prompts.json` to gh-pages. But the edited prompt's `version` field was not changed (it remains at, say, `1`). Flycut's sync client fetches the new JSON, compares `remote.version (1) > local.version (1)`, evaluates to `false`, and skips the update. Every user who already synced once continues seeing the old content indefinitely — with no error, no warning, and no mechanism to discover this happened.

**Why it happens:**
Version bumping is a manual, social convention enforced only by README documentation. There is no automated check that detects "content changed but version did not." It is invisible in PR reviews unless a reviewer specifically looks at the version field alongside the content diff.

**How to avoid:**
Add a build-time check that is git-aware: if a `.md` file has changed content relative to its previous committed state but the `version` field has not increased, the build should warn (not block, since the script must work without git context in some environments). More practically, add a CI step that runs `git diff HEAD~1 -- prompts/**` and validates that any file with a content diff also has an incremented version. Document the version requirement prominently in the PR template and README with a checklist item.

```yaml
# In GitHub Actions, after build:
- name: Check version bumps
  run: |
    changed=$(git diff HEAD~1 --name-only -- 'prompts/**/*.md')
    for f in $changed; do
      old_ver=$(git show HEAD~1:"$f" 2>/dev/null | grep '^version:' | awk '{print $2}' || echo 0)
      new_ver=$(grep '^version:' "$f" | awk '{print $2}')
      if [ "$new_ver" -le "$old_ver" ]; then
        echo "ERROR: $f content changed but version not bumped ($old_ver -> $new_ver)" >&2
        exit 1
      fi
    done
```

**Warning signs:**
- Flycut users report that a "fixed" prompt still shows the old broken content after syncing
- PR review shows modified file content but the `version:` field diff is absent
- `git log --oneline prompts/coding/fix-bug.md` shows multiple changes but `grep version prompts/coding/fix-bug.md` always returns `version: 1`

**Phase to address:**
CI/GitHub Actions workflow phase. The check must be automated — pure README documentation is insufficient for a contributor-facing repo.

---

### Pitfall 5: gh-pages Branch Never Configured in Repository Settings

**What goes wrong:**
The GitHub Actions workflow runs `peaceiris/actions-gh-pages@v4` successfully (it creates and pushes the `gh-pages` branch), but `prompts.json` is not accessible at the intended URL. Specifically, `https://raw.githubusercontent.com/generalarcade/flycut-prompts/gh-pages/prompts.json` is accessible immediately when the branch exists, but `https://generalarcade.github.io/flycut-prompts/prompts.json` returns a 404 until GitHub Pages is manually enabled in repository Settings → Pages → Source: Deploy from branch → `gh-pages`.

**Why it happens:**
GitHub Actions can create and push to the `gh-pages` branch without GitHub Pages being enabled on the repo. The raw.githubusercontent URL works from the branch content alone, but the `github.io` domain requires the Pages feature to be explicitly activated in repository settings. This is a one-time manual step that CI cannot perform via GITHUB_TOKEN.

**How to avoid:**
Document the one-time manual setup in the README. After the first CI run pushes to gh-pages, navigate to Settings → Pages and select `gh-pages` / `root` as the publishing source. The Flycut default URL uses `raw.githubusercontent.com` (which works without Pages enabled), so this pitfall only affects the alternative `github.io` domain. Decide which URL is canonical before shipping.

**Warning signs:**
- CI workflow shows green, gh-pages branch exists and contains `prompts.json`, but the `github.io` URL returns 404
- Repository Settings → Pages shows "GitHub Pages is currently disabled"
- The `raw.githubusercontent.com` URL works but the `github.io` URL does not

**Phase to address:**
CI/deployment phase, specifically the first-deployment verification checklist. This is a post-deployment manual step, not a code issue.

---

### Pitfall 6: Prompt Body Contains Backslash or `{{` Sequences That Corrupt JSON

**What goes wrong:**
Prompt content that contains backslashes (e.g., regex examples: `\n`, `\d+`) or template variable syntax like `{{clipboard}}` is processed by `jq -Rs '.'`, which correctly JSON-encodes backslashes as `\\`. However, if intermediate bash string manipulation (variable interpolation, `echo`, or `printf`) is used before passing to `jq`, backslash sequences can be consumed by the shell. The result is that `\n` in the source prompt becomes a literal newline in the JSON (invalid), or `\\` in the source becomes `\` (dropped).

**Why it happens:**
Bash interprets `\n`, `\\`, and other escape sequences inside `$()` command substitution and `echo` depending on locale settings, shell flags, and whether `echo -e` is active. The problem is environment-dependent: the same script may produce correct output in one shell and broken output in another.

**How to avoid:**
Always route content through `jq` for encoding, never build JSON strings by shell concatenation. Use `printf '%s'` instead of `echo` for content that may contain backslashes. Use `echo "$body" | jq -Rs '.'` rather than building the JSON string manually. Verify with a test fixture that includes `\n`, `\\`, and `{{clipboard}}` in the body.

```bash
# Safe: route through jq
content_json=$(printf '%s' "$body" | jq -Rs '.')

# Dangerous: shell expansion may eat backslashes
content_json="\"$(echo $body)\""
```

The `{{variable}}` syntax in prompt content is safe — curly braces have no special meaning in `jq` string values — but must not be confused with `jq` filter syntax.

**Warning signs:**
- Prompt content with regex examples appears garbled in `prompts.json`
- `{{clipboard}}` in the JSON output is correct but `\n` (literal backslash-n) in source appears as a real newline character in the JSON
- `jq empty prompts.json` exits with "Invalid string: control characters from U+0000 through U+001F must be escaped" — indicates a raw newline inside a JSON string

**Phase to address:**
Build script implementation phase. Add a test fixture `prompts/coding/test-escaping.md` with a body containing backslashes and `{{clipboard}}`, run the build, and verify with `jq . prompts.json` before removing the test file.

---

### Pitfall 7: Renaming a Prompt File Changes Its `id` and Orphans All Users

**What goes wrong:**
A contributor renames `fix-bug.md` to `debug-code.md` to better reflect the prompt's purpose. The build succeeds. The new `prompts.json` contains `"id": "debug-code"` but no longer contains `"id": "fix-bug"`. Every Flycut user who previously synced `fix-bug` retains it locally forever (the sync protocol has no deletion). They now have both the orphaned `fix-bug` (old content) and the new `debug-code` (same content). The old `fix-bug` id is dead — it can never be updated again because it is absent from the JSON.

**Why it happens:**
The filename-as-ID pattern creates a hard coupling between a filesystem path and a stable distributed identifier. Renaming is a breaking change that the build pipeline cannot detect because it has no knowledge of historical IDs. There is no tombstone or deprecation mechanism in the current sync protocol.

**How to avoid:**
Treat filenames as immutable once a prompt has been deployed to gh-pages (i.e., once any real user could have synced it). Document this as a hard rule in CONTRIBUTING.md: "Never rename a prompt file. If you want to reorganize, create a new prompt file with a new name and update the content of the old one." If a rename is truly necessary, keep the old file with updated content pointing to the new prompt, or accept the orphan.

**Warning signs:**
- A PR diff shows a file deletion and a file addition with similar content (classic rename indicator)
- The old ID disappears from `prompts.json` while a new, similar ID appears
- Contributor rationale says "just renaming for clarity"

**Phase to address:**
Contributor documentation phase (README/CONTRIBUTING.md). Also add a CI check that alerts (not blocks) when an ID disappears from the current `prompts.json` relative to the previous build.

---

## Technical Debt Patterns

| Shortcut | Immediate Benefit | Long-term Cost | When Acceptable |
|----------|-------------------|----------------|-----------------|
| No version-bump CI enforcement | Simpler build script, no git dependency in CI | Contributors forget; silent sync failures for end users | Never — automate this check from the start |
| `publish_dir: .` (whole repo to gh-pages) | One-line deploy config | Source files, scripts, and `.github/` exposed at the public URL; Flycut works but the page is a mess | Never — use `_deploy/` staging directory |
| Skipping JSON schema validation in CI | Faster CI setup | Schema drift goes undetected; Flycut decoder may fail silently on a missing field | Never — `jq` is already installed in CI for the build |
| Using bash associative arrays for duplicate detection | Cleaner-looking code | Breaks on macOS default bash 3.2; contributors can't run the build locally | Never — use POSIX-compatible alternative |
| Committing test fixture prompts with weak content | Fast initial build verification | Test prompts ship to all Flycut users if not removed before first gh-pages deployment | Acceptable in development only; remove before first real deployment |
| Using `raw.githubusercontent.com/main/` as Flycut URL | Simpler — no Pages setup required | `main` copy may lag CI or be manually edited; not a guaranteed clean build artifact | Acceptable for initial development testing only; switch to gh-pages URL before publishing |

---

## Integration Gotchas

| Integration | Common Mistake | Correct Approach |
|-------------|----------------|------------------|
| `peaceiris/actions-gh-pages@v4` | Using `include_files:` parameter which does not exist in this action | Use a `_deploy/` staging directory and set `publish_dir: ./_deploy` |
| `peaceiris/actions-gh-pages@v4` | Missing `contents: write` in workflow `permissions:` | Explicitly declare `permissions: contents: write` at the job or workflow level |
| GitHub Pages | Expecting the `github.io` URL to work immediately after first CI run | Manually enable Pages in repository Settings → Pages after the `gh-pages` branch is created |
| `jq -Rs '.'` content encoding | Assuming `echo "$body"` is safe before piping to `jq` | Use `printf '%s' "$body"` to avoid shell escape interpretation |
| `raw.githubusercontent.com` CORS preflight | Fetching the URL from browser JavaScript with `mode: 'cors'` and credentials | Flycut uses URLSession (iOS/macOS native), not browser fetch — this is only a pitfall if someone adds a web-based viewer |
| `catalog.yaml` version parsing | `grep '^version:'` matching version fields inside the `categories:` block if structure changes | Anchor the grep to the start of the file: `head -5 catalog.yaml | grep '^version:'` |

---

## Performance Traps

| Trap | Symptoms | Prevention | When It Breaks |
|------|----------|------------|----------------|
| Spawning a new `jq` process per prompt in a loop | Build time grows linearly; 100 prompts = 100 jq invocations | Acceptable at current scale (23 prompts = trivial). Optimize only if build time exceeds 5 seconds | Not a concern until 500+ prompts |
| `find` without `-maxdepth` finds nested prompts in unexpected directories | Non-prompt `.md` files (e.g., READMEs inside category dirs) processed as prompts | Use `find "$PROMPTS_DIR" -maxdepth 2 -name '*.md'` to constrain search depth | Any time a nested subdirectory or README.md is added to `prompts/` |
| Repeatedly parsing `catalog.yaml` per file | Negligible for current scale | Parse once at script start, cache in variables | Not a concern at any realistic scale |

---

## Security Mistakes

| Mistake | Risk | Prevention |
|---------|------|------------|
| Storing secrets or API keys in prompt content | Key exposed via public `prompts.json` on gh-pages | Prompts are public content by design — never put sensitive values in them. This is inherent to the model. |
| Using `eval` to process YAML values in bash | Arbitrary code execution if a prompt file is maliciously crafted | Never use `eval` for frontmatter parsing; use `grep`/`sed`/`awk` with fixed patterns only |
| Accepting PRs from forks without review | Fork CI has `pull_request` trigger, potentially exposing GITHUB_TOKEN | Default GitHub PR workflow for public repos runs with read-only token on fork PRs — this is safe by default; do not change it |

---

## "Looks Done But Isn't" Checklist

- [ ] **Build script runs on macOS:** Test with `/bin/bash scripts/build.sh` (bash 3.2) before declaring it complete — CI Ubuntu succeeds but macOS may fail on `declare -A`
- [ ] **Content field has no leading newlines:** Run `jq '.prompts[] | select(.content | startswith("\n"))' prompts.json` — should return nothing
- [ ] **Version bump check is automated:** README documentation alone is not sufficient — CI must enforce this
- [ ] **gh-pages deploys only `prompts.json`:** Verify the gh-pages branch contains only `prompts.json` (and optionally `index.html`), not the full repo tree
- [ ] **GitHub Pages enabled in settings:** Navigate to repo Settings → Pages and confirm the `gh-pages` branch is the publishing source
- [ ] **All 23 prompts present in output:** `jq '.prompts | length' prompts.json` must equal 23 — a silent awk/sed parse failure can cause a prompt to be silently skipped
- [ ] **No duplicate IDs in output JSON:** `jq '[.prompts[].id] | group_by(.) | map(select(length > 1))' prompts.json` must return `[]`
- [ ] **Prompt with colon in title parses correctly:** Verify at least one prompt title contains a colon and appears correctly in `prompts.json`
- [ ] **Catalog version in `catalog.yaml` matches `prompts.json`:** `jq '.version' prompts.json` must equal the value in `catalog.yaml`
- [ ] **`build.sh` is executable:** `ls -la scripts/build.sh` should show `-rwxr-xr-x`; otherwise CI fails with "Permission denied"

---

## Recovery Strategies

| Pitfall | Recovery Cost | Recovery Steps |
|---------|---------------|----------------|
| Version not bumped, users have stale prompts | MEDIUM | Bump the version to (current + 1), rebuild, deploy — affected users get the update on next sync |
| Prompt file renamed (ID changed) | HIGH | No rollback possible for orphaned IDs in user local stores. Create new file with new ID, keep old file with a "deprecated" note in title, or accept both coexist forever |
| Broken JSON deployed to gh-pages | LOW | Fix the source, push to main, CI redeploys — gh-pages reverts to a valid build. Users are unaffected until they retry sync. |
| macOS `declare -A` failure | LOW | Rewrite the duplicate-check to use POSIX string operations — one-time fix, no user impact |
| Leading newlines in content shipped to users | MEDIUM | Fix the body extraction, bump affected prompt versions, rebuild and deploy — users receive corrected content on next sync (version gate triggers the update) |
| GitHub Pages not enabled, `github.io` URL 404 | LOW | Manually enable in Settings → Pages; no code change needed |

---

## Pitfall-to-Phase Mapping

| Pitfall | Prevention Phase | Verification |
|---------|------------------|--------------|
| macOS bash 3.2 / `declare -A` failure | Build script implementation | Run `bash --version` to confirm 3.x on macOS, then run `bash scripts/build.sh` — must succeed |
| Quoted title with colon silently truncated | Build script implementation | Build with a test fixture containing `title: "Test: With Colon"` and verify JSON output |
| Leading newline in `content` field | Build script implementation | `jq '.prompts[] | select(.content | startswith("\n"))' prompts.json` returns empty |
| Version not bumped on content edit | CI workflow phase | CI version-bump check step runs and fails on a test PR with modified content but unchanged version |
| gh-pages branch not enabled in Settings | Deployment / first-deploy phase | Navigate to Settings → Pages, confirm `gh-pages` branch selected |
| Backslash / `{{` sequences corrupting JSON | Build script implementation | Test fixture with `\n` and `{{clipboard}}` in body; `jq empty prompts.json` exits 0 |
| Prompt file rename orphans users | Contributor documentation | CONTRIBUTING.md explicitly forbids renaming; CI alert (not block) when an existing ID disappears from output |
| All-repo publish to gh-pages | CI workflow phase | Verify gh-pages branch tree contains only `prompts.json` and optionally `index.html` |
| `build.sh` not executable | Repository setup phase | `git ls-files --stage scripts/build.sh` shows mode `100755` not `100644` |
| Catalog version not bumped on release | Release / contributor workflow | CI or PR template checklist item: "catalog.yaml version bumped?" |

---

## Sources

- `/dist/PROMPTS-REPO-DESIGN.md` — Primary specification; source of build pseudocode analyzed for pitfalls (HIGH confidence)
- `.planning/PROJECT.md` — Constraints and design decisions (HIGH confidence)
- `.planning/research/ARCHITECTURE.md` — Anti-patterns and integration points (HIGH confidence)
- [Associative array error on macOS for bash: `declare -A`](https://dipeshmajumdar.medium.com/associative-array-error-on-macos-for-bash-declare-a-invalid-option-16466534e145) — macOS bash 3.2 limitation (HIGH confidence, confirmed by multiple sources)
- [Bash: Fixing `declare: -A: invalid option`](https://www.ianoutterside.com/bash-invalid-option/) — POSIX-compatible workarounds (MEDIUM confidence)
- [Escaping Characters in YAML Front Matter](https://inspirnathan.com/posts/134-escape-characters-in-yaml-frontmatter/) — Colon-in-title parsing issues (MEDIUM confidence)
- [YAML title not properly parsed — Obsidian Forum](https://forum.obsidian.md/t/yaml-title-not-properly-parsed-in-preview-frontmatter/9165) — Real-world colon parsing failures (MEDIUM confidence)
- [jq newline output issue #787](https://github.com/jqlang/jq/issues/787) — jq always appends newline; handling content correctly (MEDIUM confidence)
- [Preserve special characters in jq output #1881](https://github.com/jqlang/jq/issues/1881) — Newline handling in jq string encoding (MEDIUM confidence)
- [Build a JSON String With Bash Variables — Baeldung](https://www.baeldung.com/linux/bash-variables-create-json-string) — jq `--arg` / `--argjson` as safe escaping pattern (HIGH confidence)
- [GitHub Pages — Configuring a publishing source](https://docs.github.com/en/pages/getting-started-with-github-pages/configuring-a-publishing-source-for-your-github-pages-site) — Manual first-time Pages setup requirement (HIGH confidence)
- [CORS and raw.githubusercontent.com — GitHub Community](https://github.com/orgs/community/discussions/69281) — CORS preflight limitation on raw.githubusercontent.com (HIGH confidence, GitHub infrastructure)
- [peaceiris/actions-gh-pages](https://github.com/peaceiris/actions-gh-pages) — `publish_dir` and permissions documentation (HIGH confidence)
- [GitHub Actions — path filters don't apply to workflow_dispatch](https://docs.github.com/actions/using-workflows/events-that-trigger-workflows) — Documented limitation (HIGH confidence)
- [BSD/macOS sed vs GNU sed](https://riptutorial.com/sed/topic/9436/bsd-macos-sed-vs--gnu-sed-vs--the-posix-sed-specification) — sed portability differences between macOS and Ubuntu CI (HIGH confidence)
- [How to Escape Characters in Bash for JSON — tutorialpedia.org](https://www.tutorialpedia.org/blog/escaping-characters-in-bash-for-json/) — Never concatenate shell variables into JSON strings (MEDIUM confidence)

---
*Pitfalls research for: Flycut Prompts Repository (Markdown-to-JSON build pipeline with bash and GitHub Pages)*
*Researched: 2026-03-11*
