# Phase 03: CI/CD and Deployment - Research

**Researched:** 2026-03-12
**Domain:** GitHub Actions workflow, gh-pages deployment, vanilla HTML/CSS/JS landing page
**Confidence:** HIGH

<user_constraints>
## User Constraints (from CONTEXT.md)

### Locked Decisions
- `_deploy/` staging directory pattern: CI copies prompts.json + index.html into `_deploy/`, then deploys that directory to gh-pages
- index.html is a **static file at repo root** that fetches prompts.json client-side via JavaScript — not regenerated on each build
- CI copies the static index.html + freshly-built prompts.json into the deploy staging area
- **Category filter tabs** with counts: All (23) | Coding (8) | Writing (6) | Analysis (5) | Creative (4) — click to filter the grid
- **Search/filter bar** — text input that filters prompts by title/description as you type
- **Prompt cards with copy button** — each prompt shown as card with title, category badge, description, and full content visible (not collapsed). Copy-to-clipboard button on each card
- **Dark/light mode toggle** — theme switcher that respects system preference
- **Full prompt content visible** on cards — no expand/collapse, all content shown for scanability
- **Nicely styled** — designed as a foundation to build upon later
- **Inspiration:** prompts.chat (card grid, copy buttons, modern aesthetic) and block.github.io/goose/prompt-library/ (category tabs, clean documentation style)
- Schema validation (prompts.json against schema/prompt.schema.json via jq) added **to build.sh itself** — works locally and in CI without duplication
- CI runs build.sh as a single step; build.sh handles both compilation and validation
- Validation failure causes build.sh to exit non-zero, failing the CI step
- index.html is a **static file checked into the repo root** — not generated during build
- It dynamically loads prompts.json via fetch() at runtime and renders the UI client-side
- Zero build-time dependencies for the page — pure HTML/CSS/JS

### Claude's Discretion
- Workflow trigger path filters (which file changes trigger the build)
- gh-pages branch management strategy (force-push vs incremental)
- Exact CSS styling approach and card layout details
- GitHub Actions runner version and action versions
- Error notification approach (default GitHub behavior)

### Deferred Ideas (OUT OF SCOPE)
- Search within prompt content (full-text search) — future enhancement to landing page
- Prompt content preview/expand interaction — future UX improvement
- Richer browsing capabilities (tags, filtering by multiple criteria) — future phase
- PR-time validation workflow (build without deploying) — v2 requirement (EVAL-01)
- CHANGELOG generation — v2 requirement (EVAL-02)
</user_constraints>

<phase_requirements>
## Phase Requirements

| ID | Description | Research Support |
|----|-------------|-----------------|
| CICD-01 | GitHub Actions workflow triggers on push to `main` when `prompts/**`, `catalog.yaml`, or `scripts/build.sh` change | on.push.paths filter syntax verified via GitHub Docs |
| CICD-02 | Workflow supports manual trigger via `workflow_dispatch` | on.workflow_dispatch syntax — path filters do NOT apply to manual triggers (always runs) |
| CICD-03 | Workflow runs `build.sh`, validates output JSON, and deploys to `gh-pages` branch | build.sh handles validation; peaceiris/actions-gh-pages@v4 deploys to gh-pages |
| CICD-04 | Deployment uses `_deploy/` staging directory containing only `prompts.json` and a simple `index.html` | publish_dir parameter of peaceiris/actions-gh-pages accepts arbitrary staging directory |
| CICD-05 | Workflow validates no duplicate IDs, all required fields present, and valid JSON structure | build.sh already validates duplicates and required fields; jq validation step in build.sh covers JSON structure |
</phase_requirements>

---

## Summary

This phase builds a GitHub Actions workflow that triggers on qualifying pushes to `main` (path-filtered to `prompts/**`, `catalog.yaml`, `scripts/build.sh`), runs the existing `build.sh` script (which handles compilation + validation in one step), stages the output into a `_deploy/` directory alongside a static `index.html`, and deploys that directory to the `gh-pages` branch using `peaceiris/actions-gh-pages@v4`. The workflow also supports manual dispatch via `workflow_dispatch`.

The landing page is a single static `index.html` checked into the repo root. It uses pure HTML/CSS/JS — zero dependencies — and fetches `prompts.json` at runtime via `fetch()`. It renders a card grid with category filter tabs, a search bar, dark/light mode toggle, and copy-to-clipboard buttons on each card.

Key constraints driving all decisions: the build script is already complete and handles validation; `jq` is pre-installed on `ubuntu-latest` (version 1.6); no Node.js or external tools are needed; the deployment produces only two files on `gh-pages` (`prompts.json` + `index.html`).

**Primary recommendation:** Use `peaceiris/actions-gh-pages@v4` with `force_orphan: true` and `github_token: ${{ secrets.GITHUB_TOKEN }}`. Use `on.push.paths` for automatic triggers and `on.workflow_dispatch` for manual dispatch. Write `index.html` as a single self-contained file with inline CSS and JS.

---

## Standard Stack

### Core

| Library/Tool | Version | Purpose | Why Standard |
|---|---|---|---|
| `peaceiris/actions-gh-pages` | v4.0.0 (released 2025-04-08) | Deploy directory to gh-pages branch | Most widely used gh-pages deploy action; supports GITHUB_TOKEN, force_orphan, publish_dir |
| `actions/checkout` | v4 (v4.3.1 stable) | Checkout repo in CI | Standard; v6 exists but v4 is stable and universally documented |
| `jq` | 1.6 (pre-installed on ubuntu-latest) | JSON validation + assembly | Already pre-installed, already used by build.sh |
| `ubuntu-latest` | Latest GitHub-hosted runner | CI execution environment | Standard; jq 1.6 pre-installed |

### Supporting

| Tool | Version | Purpose | When to Use |
|---|---|---|---|
| `GITHUB_TOKEN` | Automatic | Authenticate gh-pages push | No setup required; default approach |
| `navigator.clipboard.writeText()` | Web API | Copy prompt text to clipboard | Modern browsers, HTTPS only |
| CSS custom properties + `data-theme` | CSS3 | Dark/light mode theming | Clean toggle without JS frameworks |
| `prefers-color-scheme` media query | CSS3 | Detect system dark/light preference | Used on initial load |
| `localStorage` | Web API | Persist user theme preference | Survives page reload |

### Alternatives Considered

| Instead of | Could Use | Tradeoff |
|---|---|---|
| `peaceiris/actions-gh-pages@v4` | `actions/deploy-pages` (official GitHub) | Official action requires pages.write permission + upload-pages-artifact; more setup; peaceiris is simpler for force-push to branch |
| `force_orphan: true` | Incremental commits to gh-pages | force_orphan keeps gh-pages history clean (single commit); incremental grows history indefinitely with no benefit for this use case |
| `actions/checkout@v4` | `actions/checkout@v6` | v6 is latest (Jan 2025); v4 is still maintained; either works — v4 is safer given it's what all current examples use |

**Installation (no install step needed):** All tools are pre-installed on `ubuntu-latest`. The workflow uses GitHub Actions marketplace actions only.

---

## Architecture Patterns

### Recommended Project Structure

```
.github/
└── workflows/
    └── deploy.yml          # The single workflow file for this phase

index.html                  # Static landing page (checked into repo root)
prompts.json                # Build output (already in repo root)
scripts/
└── build.sh                # Existing; CI calls this directly
```

The `_deploy/` staging directory is created ephemerally during CI and is never committed to `main`.

### Pattern 1: Single Workflow File with Combined Trigger

**What:** One workflow file with both `on.push.paths` and `on.workflow_dispatch`. Path filters apply only to push events — manual dispatch always runs regardless of which files changed.

**When to use:** Always — this is the only pattern that satisfies CICD-01 and CICD-02 simultaneously.

**Example:**
```yaml
# Source: https://docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions
on:
  push:
    branches: [main]
    paths:
      - 'prompts/**'
      - 'catalog.yaml'
      - 'scripts/build.sh'
  workflow_dispatch:
```

**Important:** `workflow_dispatch` has no `paths` filter — it is not supported for manual triggers. When a user clicks "Run workflow" in the GitHub Actions UI, the workflow always runs. This is the correct behavior for CICD-02 ("manual trigger runs the same build and deploy steps").

### Pattern 2: Staging Directory Deploy

**What:** CI creates a `_deploy/` directory, copies only the files that should be published (prompts.json + index.html), then passes that directory to the deploy action. The gh-pages branch contains exactly those two files.

**When to use:** Always — this is the locked decision for CICD-04.

**Example:**
```yaml
# Source: peaceiris/actions-gh-pages documentation
- name: Prepare deploy staging
  run: |
    mkdir -p _deploy
    cp prompts.json _deploy/prompts.json
    cp index.html _deploy/index.html

- name: Deploy to gh-pages
  uses: peaceiris/actions-gh-pages@v4
  with:
    github_token: ${{ secrets.GITHUB_TOKEN }}
    publish_dir: ./_deploy
    force_orphan: true
```

### Pattern 3: Concurrency Guard for Deployments

**What:** Use `concurrency` to queue deployments and prevent simultaneous pushes from racing. For deployments, `cancel-in-progress: false` ensures in-flight deploys complete rather than being interrupted mid-push.

**When to use:** Recommended for any workflow that deploys — prevents partial gh-pages states.

**Example:**
```yaml
# Source: https://docs.github.com/actions/writing-workflows/choosing-what-your-workflow-does/control-the-concurrency-of-workflows-and-jobs
concurrency:
  group: deploy-gh-pages
  cancel-in-progress: false
```

### Pattern 4: Minimal Permissions Scope

**What:** Explicitly declare only the permissions the workflow needs. For deploying to gh-pages via GITHUB_TOKEN, the workflow only needs `contents: write`. All other permissions default to none.

**When to use:** Always — security best practice, and required for GITHUB_TOKEN to push to gh-pages.

**Example:**
```yaml
# Source: https://docs.github.com/en/actions/security-guides/automatic-token-authentication
permissions:
  contents: write
```

### Pattern 5: index.html Architecture — Fetch-at-Runtime

**What:** The static `index.html` file makes a `fetch()` call to `prompts.json` at page load time (relative URL works since both files are served from the same gh-pages root). No build step generates HTML — the page reads whatever `prompts.json` is currently deployed.

**When to use:** Always — this is the locked decision. Zero build-time dependencies.

**Example (fetch pattern):**
```javascript
// Source: MDN Web Docs (web.dev/patterns/clipboard/copy-text)
async function loadPrompts() {
  const response = await fetch('./prompts.json');
  const catalog = await response.json();
  renderPrompts(catalog.prompts);
}
```

### Pattern 6: Dark/Light Mode with CSS Custom Properties

**What:** Define a palette of CSS custom properties on `:root` for light mode and override them under `[data-theme="dark"]`. Toggle by setting `document.documentElement.setAttribute('data-theme', 'dark')`. Persist to `localStorage`. On initial load, check localStorage first, then `window.matchMedia('(prefers-color-scheme: dark)')`.

**When to use:** Single-file HTML with no framework — this is the simplest correct approach.

**Example:**
```css
/* Source: CSS-in-Real-Life pattern */
:root {
  --bg: #ffffff;
  --text: #1a1a1a;
  --card-bg: #f5f5f5;
  --border: #e0e0e0;
  --accent: #0066cc;
}
[data-theme="dark"] {
  --bg: #1a1a1a;
  --text: #f0f0f0;
  --card-bg: #2a2a2a;
  --border: #444444;
  --accent: #4da6ff;
}
```

```javascript
// On load — prevent flash of wrong theme
const saved = localStorage.getItem('theme');
const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
document.documentElement.setAttribute('data-theme',
  saved || (prefersDark ? 'dark' : 'light'));

// On toggle button click
function toggleTheme() {
  const current = document.documentElement.getAttribute('data-theme');
  const next = current === 'dark' ? 'light' : 'dark';
  document.documentElement.setAttribute('data-theme', next);
  localStorage.setItem('theme', next);
}
```

### Pattern 7: Copy-to-Clipboard with Feedback

**What:** Use `navigator.clipboard.writeText()` (async, returns Promise). Show brief visual feedback ("Copied!") for ~2 seconds, then revert button label.

**Security constraint:** Clipboard API only works on HTTPS pages. GitHub Pages is always HTTPS — no issue here.

**Example:**
```javascript
// Source: web.dev/patterns/clipboard/copy-text
async function copyPrompt(text, button) {
  try {
    await navigator.clipboard.writeText(text);
    const original = button.textContent;
    button.textContent = 'Copied!';
    setTimeout(() => { button.textContent = original; }, 2000);
  } catch (err) {
    // Fallback: document.execCommand('copy') for older browsers
    console.error('Copy failed:', err);
  }
}
```

### Anti-Patterns to Avoid

- **Generating index.html in CI:** The decision is locked — index.html is a static file. Never add a build step that regenerates it.
- **Deploying the full repo to gh-pages:** Only `_deploy/` contents should land on gh-pages. Deploying the whole repo leaks source files and build scripts.
- **Using `cancel-in-progress: true` for deployments:** This can interrupt a deployment mid-push, leaving gh-pages in a broken state. Use `false`.
- **Hardcoding the raw.githubusercontent.com URL in index.html:** The fetch should use a relative URL (`./prompts.json`) so it works on any GitHub Pages domain, not just a specific repo path.
- **Using `jq` for JSON Schema validation in CI:** `jq` does not implement JSON Schema. Validation is handled by `build.sh` itself (structural checks, field validation, duplicate detection). Do not add a separate jq-schema-validate step — the build.sh exit code is sufficient.

---

## Don't Hand-Roll

| Problem | Don't Build | Use Instead | Why |
|---|---|---|---|
| Deploy to gh-pages branch | Git push scripting, custom deploy step | `peaceiris/actions-gh-pages@v4` | Handles branch creation, force-push, commit authoring, token auth, force_orphan — many edge cases |
| JSON schema validation in CI | Custom jq script for schema checking | build.sh itself (already validates) | jq 1.6 has no JSON Schema support; build.sh structural validation is sufficient for this phase |
| Dark mode persistence | Custom cookie-based storage | `localStorage` + CSS custom properties | `localStorage` is synchronous, same-origin, no server needed |
| Copy to clipboard | `document.execCommand('copy')` | `navigator.clipboard.writeText()` | execCommand is deprecated; Clipboard API is the modern standard |

**Key insight:** The most complex CI concern (deploying to gh-pages) is fully solved by `peaceiris/actions-gh-pages@v4`. The most complex validation concern is already solved by `build.sh`. This phase is primarily configuration + a single HTML file.

---

## Common Pitfalls

### Pitfall 1: GitHub Pages Not Enabled Before First Deploy

**What goes wrong:** The workflow deploys to gh-pages successfully (branch is created and updated), but the GitHub Pages site is never live. Users see a 404 at the expected URL.

**Why it happens:** GitHub Pages must be manually enabled in repository Settings → Pages → Source → "Deploy from a branch" → select `gh-pages` / `/(root)`. This is a one-time manual step that cannot be automated via Actions.

**How to avoid:** Document this as a required post-deploy step in the task plan. The STATE.md already flags this: "[Phase 3]: GitHub Pages must be manually enabled in repository Settings after first gh-pages push — document as post-deploy step."

**Warning signs:** Workflow completes successfully but the site URL returns 404.

### Pitfall 2: GITHUB_TOKEN Permissions Insufficient for gh-pages Push

**What goes wrong:** The `peaceiris/actions-gh-pages@v4` action fails with a permission error when trying to push to the `gh-pages` branch.

**Why it happens:** Repository or organization settings may restrict default GITHUB_TOKEN permissions to read-only. The `contents: write` permission must be explicitly declared in the workflow.

**How to avoid:** Always include `permissions: contents: write` at the workflow level.

**Warning signs:** Action fails with "Error: Action failed with "not found"" or "Error: remote: Permission to ... denied to github-actions[bot]".

### Pitfall 3: workflow_dispatch Appears Missing From Actions UI

**What goes wrong:** The "Run workflow" button is absent from the GitHub Actions UI even though `workflow_dispatch` is in the YAML.

**Why it happens:** `workflow_dispatch` only appears on the default branch (main). If the workflow file is only on a feature branch, the button won't appear.

**How to avoid:** The workflow file must be committed to the `main` branch. Ensure `.github/workflows/deploy.yml` is merged to main before expecting the button to appear.

**Warning signs:** Button absent from Actions → workflow tab. Also appears if branch name in `branches: [main]` doesn't match the actual default branch name.

### Pitfall 4: Path Filter With Both push and workflow_dispatch

**What goes wrong:** Developer expects `paths` filter to also limit when `workflow_dispatch` fires, and is confused when manual runs always execute even without relevant file changes.

**Why it happens:** `paths` and `paths-ignore` filters apply only to `push` and `pull_request` events. `workflow_dispatch` always runs when manually triggered — path filters are silently ignored for it.

**How to avoid:** This is the correct/expected behavior. CICD-02 explicitly requires "manual trigger runs the same build and deploy steps" — so always-runs for `workflow_dispatch` is correct, not a bug.

**Warning signs:** Developer adds `paths:` under `workflow_dispatch:` — it will be silently ignored by GitHub.

### Pitfall 5: fetch() CORS Issue in index.html During Local Development

**What goes wrong:** Opening `index.html` directly from the filesystem (`file://`) causes the `fetch('./prompts.json')` call to fail with a CORS error.

**Why it happens:** Browsers block `fetch()` for `file://` URLs.

**How to avoid:** When testing locally, serve with a local HTTP server (e.g., `python3 -m http.server 8080`). On GitHub Pages (HTTPS), fetch works correctly. Add a note in the dev workflow documentation.

**Warning signs:** Console shows "Cross-Origin Request Blocked" when opening `file:///.../index.html`.

### Pitfall 6: Clipboard API Requires HTTPS

**What goes wrong:** `navigator.clipboard.writeText()` throws a security error when tested on `http://` or `file://`.

**Why it happens:** The Clipboard API is restricted to secure contexts (HTTPS or localhost).

**How to avoid:** GitHub Pages always serves over HTTPS — no issue in production. For local testing, use `localhost` (e.g., via `python3 -m http.server`), not the `file://` protocol.

---

## Code Examples

Verified patterns from official sources:

### Complete Workflow YAML Structure

```yaml
# Source: GitHub Docs (workflow-syntax-for-github-actions) + peaceiris/actions-gh-pages README
name: Build and Deploy

on:
  push:
    branches: [main]
    paths:
      - 'prompts/**'
      - 'catalog.yaml'
      - 'scripts/build.sh'
  workflow_dispatch:

concurrency:
  group: deploy-gh-pages
  cancel-in-progress: false

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout
        uses: actions/checkout@v4

      - name: Build
        run: bash scripts/build.sh

      - name: Prepare staging
        run: |
          mkdir -p _deploy
          cp prompts.json _deploy/prompts.json
          cp index.html _deploy/index.html

      - name: Deploy to gh-pages
        uses: peaceiris/actions-gh-pages@v4
        with:
          github_token: ${{ secrets.GITHUB_TOKEN }}
          publish_dir: ./_deploy
          force_orphan: true
```

### Schema Validation Step in build.sh (Addition)

The CONTEXT.md locks that build.sh itself handles schema validation. The following addition to `scripts/build.sh` (after the prompts.json is generated) uses `jq` for structural validation — confirming the output is valid JSON and matches expected structure:

```bash
# Source: jq documentation (jq -e exits non-zero if result is false/null)
# Validate output is valid JSON with required top-level fields
if ! jq -e '.version and (.prompts | type == "array") and (.prompts | length > 0)' "$OUTPUT" > /dev/null; then
    echo "ERROR: prompts.json failed structural validation" >&2
    exit 1
fi
echo "Validation passed: prompts.json structure is valid"
```

Note: jq 1.6 (pre-installed on ubuntu-latest) does not support JSON Schema draft 2020-12. Full schema validation would require `ajv` or similar — but the locked decision is that build.sh structural checks are sufficient for this phase.

### index.html Skeleton Pattern

```html
<!DOCTYPE html>
<html lang="en" data-theme="light">
<head>
  <meta charset="UTF-8">
  <meta name="viewport" content="width=device-width, initial-scale=1.0">
  <title>Prompt Library</title>
  <style>
    /* CSS custom properties for theming */
    :root {
      --bg: #f8f9fa; --surface: #ffffff; --text: #212529;
      --text-muted: #6c757d; --border: #dee2e6; --accent: #0d6efd;
      --badge-bg: #e9ecef; --card-shadow: 0 1px 3px rgba(0,0,0,0.1);
    }
    [data-theme="dark"] {
      --bg: #121212; --surface: #1e1e1e; --text: #e0e0e0;
      --text-muted: #9e9e9e; --border: #333333; --accent: #4da6ff;
      --badge-bg: #2a2a2a; --card-shadow: 0 1px 3px rgba(0,0,0,0.4);
    }
  </style>
</head>
<body>
  <!-- Filter tabs, search bar, card grid -->
  <script>
    // Apply saved theme before DOM renders (prevents flash)
    const saved = localStorage.getItem('theme');
    const prefersDark = window.matchMedia('(prefers-color-scheme: dark)').matches;
    document.documentElement.setAttribute('data-theme',
      saved || (prefersDark ? 'dark' : 'light'));

    async function init() {
      const res = await fetch('./prompts.json');
      const { prompts } = await res.json();
      // render cards...
    }

    document.addEventListener('DOMContentLoaded', init);
  </script>
</body>
</html>
```

---

## State of the Art

| Old Approach | Current Approach | When Changed | Impact |
|---|---|---|---|
| `actions/checkout@v2` | `actions/checkout@v4` (v6 is latest) | 2022-2025 | v4 uses Node 20; v6 uses Node 24. Use v4 for stability. |
| `peaceiris/actions-gh-pages@v3` | `peaceiris/actions-gh-pages@v4` | April 2025 | v4 is the latest; v3 is still maintained. Use v4. |
| `document.execCommand('copy')` | `navigator.clipboard.writeText()` | ~2020 | execCommand deprecated; Clipboard API is standard |
| CSS class toggling for dark mode | CSS custom properties + `data-theme` attribute | ~2021 | Custom properties allow zero-JS theme changes; cleaner |
| Deploying full repo to gh-pages | Deploy staging directory only | Ongoing best practice | Prevents leaking source files; clean published output |

**Deprecated/outdated:**
- `document.execCommand('copy')`: Deprecated in all major browsers; replace with `navigator.clipboard.writeText()`
- `peaceiris/actions-gh-pages@v3`: Still works but v4 is latest (April 2025)

---

## Open Questions

1. **jq schema validation depth**
   - What we know: jq 1.6 is pre-installed on ubuntu-latest; jq cannot validate against JSON Schema; build.sh structural checks cover duplicate IDs, required fields, valid categories
   - What's unclear: Whether the planner wants to add a separate `jq -e` structural check step in build.sh or rely entirely on the existing field-level checks
   - Recommendation: Add a minimal `jq -e` check at the end of build.sh to verify the output JSON parses cleanly and has the expected top-level shape. This satisfies CICD-05 without requiring external tools.

2. **GitHub Pages manual enable timing**
   - What we know: gh-pages branch is created by the first successful workflow run; GitHub Pages must then be enabled manually
   - What's unclear: The repo may already have GitHub Pages enabled (we cannot verify from local context)
   - Recommendation: The plan should include a verification task that checks the Pages URL is live after the first deployment. Document the manual step clearly.

3. **index.html relative URL vs absolute URL for prompts.json**
   - What we know: GitHub Pages serves from `https://<org>.github.io/<repo>/` by default; `fetch('./prompts.json')` is relative to the page URL
   - What's unclear: If GitHub Pages uses a custom domain or non-root path, the relative URL may need adjustment
   - Recommendation: Use `./prompts.json` (relative). This works correctly for both the default `github.io/<repo>/` path and any custom domain configuration.

---

## Validation Architecture

> nyquist_validation is enabled in .planning/config.json

### Test Framework

| Property | Value |
|---|---|
| Framework | bash + manual curl/verification (no unit test framework — zero-dependency constraint) |
| Config file | none — tests are workflow verification steps |
| Quick run command | `bash scripts/build.sh && echo "Build OK"` |
| Full suite command | `bash scripts/build.sh && jq -e '.version and (.prompts | length > 0)' prompts.json` |

### Phase Requirements → Test Map

| Req ID | Behavior | Test Type | Automated Command | File Exists? |
|---|---|---|---|---|
| CICD-01 | Push to main with prompts/ change triggers workflow | integration | Manual: push a change, verify Actions run | N/A — GitHub Actions trigger |
| CICD-02 | workflow_dispatch triggers workflow | integration | Manual: GitHub UI "Run workflow" button | N/A — GitHub Actions trigger |
| CICD-03 | Workflow runs build.sh and deploys to gh-pages | integration | `bash scripts/build.sh` (local smoke test) | ✅ scripts/build.sh exists |
| CICD-04 | _deploy/ contains only prompts.json + index.html | smoke | `ls _deploy/` after build step | ❌ Wave 0 — workflow doesn't exist yet |
| CICD-05 | Invalid prompt causes workflow to fail | unit | `bash scripts/build.sh` with a corrupted test prompt | ✅ build.sh validation already tested in Phase 1 |

### Sampling Rate

- **Per task commit:** `bash scripts/build.sh && echo "Build OK"`
- **Per wave merge:** `bash scripts/build.sh && jq -e '.version and (.prompts | length > 0)' prompts.json`
- **Phase gate:** Full suite green + workflow runs successfully in GitHub Actions UI before `/gsd:verify-work`

### Wave 0 Gaps

- [ ] `.github/workflows/deploy.yml` — the workflow file (primary deliverable of this phase)
- [ ] `index.html` — static landing page at repo root (primary deliverable of this phase)
- [ ] Manual verification: GitHub Pages enabled in repository Settings after first push

*(build.sh and schema/prompt.schema.json already exist — no framework install needed)*

---

## Sources

### Primary (HIGH confidence)
- GitHub Docs (docs.github.com/en/actions/using-workflows/workflow-syntax-for-github-actions) — on.push.paths, workflow_dispatch, permissions, concurrency syntax
- GitHub Docs (docs.github.com/en/actions/security-guides/automatic-token-authentication) — GITHUB_TOKEN permissions, contents: write
- peaceiris/actions-gh-pages GitHub README — publish_dir, force_orphan, github_token parameters; v4.0.0 released 2025-04-08
- actions/checkout GitHub Releases — v4.3.1 stable; v6.0.2 latest (Jan 2025)
- GitHub Actions runner-images issue #9550 — confirmed jq 1.6 is pre-installed on ubuntu-latest; no upgrade planned
- MDN Web Docs (Clipboard/writeText) — navigator.clipboard.writeText() API, HTTPS requirement, Promise interface
- web.dev/patterns/clipboard/copy-text — copy-to-clipboard with visual feedback pattern

### Secondary (MEDIUM confidence)
- GitHub Docs concurrency control (control-the-concurrency-of-workflows-and-jobs) — cancel-in-progress: false for deployments
- CSS-in-Real-Life (css-irl.info) — CSS custom properties + data-theme dark mode pattern
- GitHub community discussions — confirmed workflow_dispatch ignores path filters (by design)

### Tertiary (LOW confidence)
- Search results re: prompts.chat and Goose prompt library visual inspiration — not verified via direct site fetch; referenced for design direction only

---

## Metadata

**Confidence breakdown:**
- Standard stack: HIGH — actions/checkout and peaceiris/actions-gh-pages versions verified via GitHub releases; jq pre-install confirmed via runner-images issue
- Architecture: HIGH — workflow YAML patterns from official GitHub Docs; path filter + workflow_dispatch interaction confirmed via community discussions
- Landing page (index.html): HIGH — vanilla HTML/CSS/JS patterns are stable; Clipboard API and CSS custom properties are well-documented standards
- Pitfalls: HIGH — gh-pages manual enable and workflow_dispatch path-filter behavior confirmed via official documentation and community discussions

**Research date:** 2026-03-12
**Valid until:** 2026-04-12 (stable domain — GitHub Actions syntax changes infrequently; peaceiris action is at v4 with no breaking changes expected short-term)
