# Flycut Prompts

Source prompts for Flycut's Prompt Library feature, synced automatically via GitHub Pages.

## What is this?

This repository stores prompt files as Markdown with YAML frontmatter. A GitHub Actions workflow builds all prompt files into a single `prompts.json` and deploys it to the `gh-pages` branch. Flycut syncs from that URL.

**Sync URL:**

```
https://raw.githubusercontent.com/generalarcade/flycut-prompts/gh-pages/prompts.json
```

### How Flycut syncs

When you trigger a sync, Flycut fetches `prompts.json` and applies it to your local prompt store:

1. **Upsert by `id`** — Each prompt's `id` is the stable key. Existing prompts are updated; new ones are inserted.
2. **Version-gated updates** — A prompt is only updated when `remote.version > local.version`. Bumping the `version` field in the source file is the only mechanism for pushing updates to users.
3. **User customization protection** — If you have edited a prompt locally (`isUserCustomized`), sync skips it entirely. Your edits are never overwritten.
4. **Revert flow** — "Revert to Original" in Flycut clears the customization flag; the next sync restores the upstream version.
5. **No deletion** — Sync only upserts. Removing a prompt from `prompts.json` does not delete it from users' local stores.

## Using prompts in Flycut

The sync URL above is pre-configured in Flycut. To sync manually:

1. Open Flycut **Settings > Prompts**
2. Paste the sync URL into the **JSON URL** field
3. Click **Sync Now**

## Adding a prompt

1. **Choose the right category directory:**
   - `prompts/coding/` — code tasks (review, refactor, debug, tests)
   - `prompts/writing/` — text tasks (summarize, rewrite, grammar, email)
   - `prompts/analysis/` — analysis tasks (data insights, comparisons, action items)
   - `prompts/creative/` — creative tasks (brainstorming, stories, names, outlines)

2. **Create the file** at `prompts/<category>/<kebab-case-id>.md`. The filename without `.md` becomes the prompt `id`.
   - Use lowercase `a-z`, digits `0-9`, and hyphens only
   - The `id` must be unique across all categories (not just within the directory)
   - Example: `prompts/writing/summarize-meeting.md` → id `summarize-meeting`

3. **Write the frontmatter block** at the top of the file. At minimum, `title` and `version: 1` are required:

   ```yaml
   ---
   title: "Your Prompt Title"
   version: 1
   description: "One-line description (not sent to Flycut)"
   ---
   ```

4. **Write the prompt body** below the closing `---`. Use `{{clipboard}}` to insert the user's clipboard text at paste time (see [Template variables](#template-variables)).

5. **Verify locally:**

   ```bash
   bash scripts/build.sh
   ```

   Expected output:
   ```
   Built /path/to/prompts.json: 24 prompt(s), catalog version 2
   Validation passed: prompts.json structure is valid
   ```

6. **Submit a pull request.** CI runs the build script automatically and fails the PR if there are any errors.

### Complete example

**File: `prompts/writing/summarize-text.md`**

```yaml
---
title: "Summarize Text"
version: 1
description: "Concise summary of text"
variables: ["clipboard"]
---

Summarize this text concisely. Capture:

1. **Main points** — the central ideas or arguments presented
2. **Key arguments** — the supporting reasoning or evidence
3. **Conclusions** — what the text ultimately asserts or recommends

Text to summarize:

{{clipboard}}

Keep the summary to 3-5 sentences unless the source is very long.
```

### Frontmatter field reference

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `title` | string | **yes** | Display title shown in Flycut's prompt list |
| `version` | integer | **yes** | Monotonically increasing. Bump to push updates. Start at 1. |
| `category` | string | no | Overrides directory-derived category. Rarely needed. |
| `description` | string | no | One-line description. Not sent to Flycut. |
| `tags` | string[] | no | For future use. Not sent to Flycut. |
| `variables` | string[] | no | Documents which `{{variables}}` the prompt uses. |

Fields sent to Flycut: `id` (from filename), `title`, `category`, `version`, `content` (from body).
Fields that are metadata only (not sent to Flycut): `description`, `tags`, `variables`.

## Updating a prompt

To update an existing prompt, edit the content in the file and bump the `version` field in the same commit.

> **Important:** Every content change to a prompt file **must** include a version bump (e.g., `version: 1` to `version: 2`). Without this, Flycut's sync system will see `remote.version == local.version` and skip the update silently. Users will not receive the change.

Example: updating `prompts/coding/fix-bug.md`

```yaml
---
title: "Fix Bug"
version: 2          # ← bumped from 1
description: "Find and fix bugs in code"
---
```

A PR that changes prompt content but does not bump the `version` field in that same file will cause users to miss the update.

## Template variables

Prompt content can contain `{{variable}}` placeholders. Flycut substitutes these at paste time.

- **`{{clipboard}}`** — Built-in. Always available. Contains the clipboard text at the moment the user pastes.
- **`{{anything_else}}`** — Looked up from user-defined key-value pairs in Flycut **Settings > Prompts > Template Variables**.
- **Unknown variables are left as-is** — If `{{name}}` has no matching key in Settings, the literal text `{{name}}` appears in the pasted output.

Example prompt using `{{clipboard}}`:

```
Review this code and identify any bugs:

{{clipboard}}

For each bug found, explain what it does wrong and suggest a fix.
```

Example prompt using a custom variable:

```
Write a professional reply to this email:

{{clipboard}}

Tone: {{tone}}
```

For the `{{tone}}` placeholder to be substituted, the user must add a `tone` key in Flycut Settings > Prompts > Template Variables.

## Versioning rules

1. **Adding a new prompt:** Set `version: 1`.
2. **Updating prompt content:** Bump the prompt's `version` (e.g., `1` to `2`).
3. **Fixing a typo before anyone synced:** You can keep `version: 1` if you are certain no one has synced yet. Otherwise, bump.
4. **Never decrease a version number.** Flycut only updates when `remote.version > local.version`. A lower or equal version is silently skipped.
5. **Catalog version (`catalog.yaml`):** Bumped by maintainers on merge. Contributors do not need to bump the catalog version in their PRs — this avoids merge conflicts and is handled as part of the merge process.

## Building locally

Prerequisites: `jq` must be installed.

```bash
brew install jq   # first time only
```

Build command:

```bash
bash scripts/build.sh
```

Expected output:

```
Built /path/to/prompts.json: 23 prompt(s), catalog version 2
Validation passed: prompts.json structure is valid
```

The build script finds all `.md` files under `prompts/`, parses their frontmatter, extracts the body, and writes a validated `prompts.json` to the repo root. It exits non-zero on duplicate IDs, missing required fields, or structural validation failure.

## Maintainer notes

**One-time GitHub Pages setup:** After the first successful CI run (which pushes to the `gh-pages` branch), enable GitHub Pages in the repository:

Settings > Pages > Source: **Deploy from branch** > Branch: `gh-pages` > Folder: `/ (root)` > Save

Without this step, the sync URL returns a 404 even though the `gh-pages` branch exists and `prompts.json` is present.

---

## Prompt catalog

All 65 prompts, grouped by category and sorted alphabetically by prompt ID.

### Coding (21 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `add-error-handling` | Add Error Handling | Add proper error handling to code |
| `code-review-swift` | Swift Code Review | Review Swift code for correctness, safety, and idiomatic style |
| `convert-to-async` | Convert to Async/Await | Modernize callback/completion handler code |
| `create-adr` | Create Architectural Decision Record | Create a structured ADR document for architecture decisions |
| `create-react-component` | Create React Component | Scaffold a new React component with TypeScript types, tests, and accessibility |
| `csharp-coding-style` | C# Coding Style | C# naming conventions, formatting, and modern language features guide |
| `domain-driven-design` | Domain-Driven Design | DDD patterns including aggregates, entities, value objects, and domain events |
| `dotnet-clean-architecture` | Clean Architecture (.NET) | Four-layer clean architecture guide for .NET projects |
| `dotnet-unit-integration-tests` | Unit and Integration Tests (.NET) | Testing guide with xUnit, FakeItEasy, and Testcontainers for .NET |
| `explain-code` | Explain Code | Step-by-step explanation of how code works |
| `fix-bug` | Fix Bug | Find and fix bugs in code |
| `object-calisthenics` | Object Calisthenics | Nine rules for writing cleaner, more maintainable object-oriented code |
| `optimize-performance` | Optimize Performance | Identify and fix performance issues |
| `python-coding-style` | Python Coding Style | PEP 8 conventions, type hints, docstrings, and error handling for Python |
| `python-testing` | Python Testing Guide | pytest patterns, fixtures, parametrize, and coverage for Python projects |
| `react-patterns` | React Patterns | React component patterns, hooks, state management, and accessibility guide |
| `react-testing` | React Testing Guide | Jest and React Testing Library patterns for testing React components |
| `refactor-code` | Refactor Code | Improve code structure without changing behavior |
| `review-react-component` | Review React Component | Review a React component against TypeScript, accessibility, and testing standards |
| `typescript-style` | TypeScript Style Guide | TypeScript strict mode conventions, naming, imports, and null handling |
| `write-tests` | Write Tests | Generate comprehensive unit tests |

### Writing (8 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `create-readme` | Create README | Create a comprehensive, well-structured README.md file for a project |
| `expand-bullet-points` | Expand Bullet Points | Turn bullet points into full paragraphs |
| `fix-grammar` | Fix Grammar | Grammar and spelling correction |
| `review-writing-clarity` | Review Writing Clarity | Review and improve writing clarity using Orwell's six rules |
| `rewrite-formal` | Rewrite Formally | Professional tone rewrite |
| `simplify-language` | Simplify Language | Make text clearer and simpler |
| `summarize-text` | Summarize Text | Concise summary of text |
| `write-email-reply` | Write Email Reply | Draft a professional email reply |

### Analysis (9 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `analyze-data` | Analyze Data | Extract key insights from data |
| `analyze-dataset` | Analyze Dataset | Profile a dataset and produce a structured summary with insights |
| `compare-options` | Compare Options | Pros/cons comparison |
| `create-eda-notebook` | Create EDA Notebook | Scaffold an exploratory data analysis Jupyter notebook for a dataset |
| `create-summary-table` | Create Summary Table | Structure text into a comparison table |
| `extract-action-items` | Extract Action Items | Pull action items from meeting notes or text |
| `extract-meeting-details` | Extract Meeting Details | Extract and analyze information from client discovery kickoff meeting notes |
| `identify-risks` | Identify Risks | Risk analysis of a plan or proposal |
| `python-data-analysis-guide` | Python Data Analysis Guide | Best practices for pandas, numpy, and visualization in data analysis |

### Creative (4 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `brainstorm` | Brainstorm Ideas | Creative brainstorming on a topic |
| `create-outline` | Create Outline | Structured outline from a topic |
| `generate-names` | Generate Names | Name ideas for projects, products, or features |
| `write-story` | Write Story | Short story from a prompt |

### DevOps (5 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `ci-cd-pipeline-guide` | CI/CD Pipeline Guide | Pipeline structure, secrets management, caching, and deployment gates |
| `infrastructure-security` | Infrastructure Security | Security best practices for secrets, IAM, networking, containers, and scanning |
| `review-infrastructure-code` | Review Infrastructure Code | Review IaC against security, Terraform, and CI/CD standards |
| `scaffold-ci-cd-pipeline` | Scaffold CI/CD Pipeline | Generate a complete CI/CD pipeline for any platform and stack |
| `terraform-guide` | Terraform Guide | Terraform module structure, remote state, variables, and style conventions |

### Product (7 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `decision-document` | Write Decision Document | Structure a product decision with options, trade-offs, and recommendation |
| `feature-prioritization` | Prioritize Features | Score and rank feature requests using structured frameworks |
| `prd-generation` | Generate PRD | Create a product requirements document from a feature description |
| `roadmap-prioritization` | Prioritize Roadmap | Evaluate and prioritize features using impact vs effort analysis |
| `stakeholder-update` | Draft Stakeholder Update | Write a clear product update for stakeholders |
| `swot-analysis` | SWOT Analysis | Generate a SWOT analysis for a product or initiative |
| `user-persona` | Create User Persona | Generate detailed user personas from customer data or descriptions |

### Research (6 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `competitor-analysis` | Competitor Analysis | Analyze competitor strengths, weaknesses, positioning, and strategy |
| `market-sizing` | Market Sizing | Estimate TAM, SAM, and SOM for a product or market opportunity |
| `market-trends` | Market Trend Analysis | Identify and evaluate market trends, industry shifts, and emerging opportunities |
| `product-market-fit` | Assess Product-Market Fit | Evaluate product-market fit from user data and market signals |
| `sentiment-analysis` | Consumer Sentiment Analysis | Track shifts in consumer sentiment and emerging preferences |
| `user-feedback-synthesis` | Synthesize User Feedback | Aggregate and prioritize insights from user feedback, reviews, or support tickets |

### Development (5 prompts)

| Prompt ID | Title | Description |
|-----------|-------|-------------|
| `api-design-review` | API Design Review | Review and improve REST or GraphQL API design |
| `architecture-decision-record` | Architecture Decision Record | Document an architectural decision with context, options, and rationale |
| `incident-postmortem` | Incident Postmortem | Write a blameless incident postmortem with root cause analysis |
| `sprint-planning` | Sprint Planning | Break down epics or features into sprint-ready user stories with estimates |
| `technical-debt-assessment` | Assess Technical Debt | Identify, categorize, and prioritize technical debt |
