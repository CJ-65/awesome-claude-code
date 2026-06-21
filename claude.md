# CLAUDE.md — Awesome Claude Code

A curated collection of resources for Claude Code (skills, hooks, slash-commands, CLAUDE.md files, tooling, workflows). This document is the AI-assistant reference for the codebase.

---

## Repository at a Glance

| Item | Value |
|---|---|
| Language | Python 3.11+ |
| Package | `awesome-claude-code` v2.0.1 |
| Build / task runner | `Makefile` (always use `make`, not raw `python`) |
| Linter / formatter | `ruff` |
| Type checker | `mypy` |
| Tests | `pytest` |
| Source of truth | `THE_RESOURCES_TABLE.csv` |
| Config | `acc-config.yaml` |
| Python env (local) | `venv/bin/python3` (CI uses system `python3`) |

---

## Directory Structure

```
awesome-claude-code/
├── THE_RESOURCES_TABLE.csv       # Master data — all resources live here
├── acc-config.yaml               # README style & selector config
├── Makefile                      # All dev tasks; primary entry point
├── pyproject.toml                # Package metadata, ruff/mypy/pytest config
├── .pre-commit-config.yaml       # Pre-commit hooks (ruff + tests + README check)
│
├── scripts/                      # All Python automation
│   ├── readme/                   # README generation subsystem
│   │   ├── generate_readme.py    # Main generator entry point
│   │   ├── generators/           # One class per README style
│   │   │   ├── base.py           # ReadmeGenerator ABC
│   │   │   ├── awesome.py        # AwesomeReadmeGenerator
│   │   │   ├── minimal.py        # MinimalReadmeGenerator (Classic)
│   │   │   ├── visual.py         # VisualReadmeGenerator (Extra)
│   │   │   └── flat.py           # ParameterizedFlatListGenerator (44 views)
│   │   ├── helpers/              # Config, paths, assets, utilities
│   │   ├── markup/               # Markdown/HTML rendering per style
│   │   └── svg_templates/        # SVG badge/header/divider renderers
│   ├── resources/                # CSV resource management
│   │   ├── resource_utils.py     # Shared CSV helpers
│   │   ├── sort_resources.py     # Sort CSV by category/name
│   │   ├── create_resource_pr.py # Bot PR creation
│   │   ├── download_resources.py # Download referenced repos
│   │   └── parse_issue_form.py   # Parse GitHub issue form submissions
│   ├── validation/
│   │   └── validate_links.py     # URL accessibility checker
│   ├── categories/
│   │   └── add_category.py       # Interactive category-addition tool
│   ├── ids/
│   │   └── resource_id.py        # ID generation: `{prefix}-{sha256[:8]}`
│   ├── badges/                   # Badge notification system
│   ├── ticker/                   # Repo-ticker SVG generation
│   ├── maintenance/              # Health checks, GitHub release data
│   ├── testing/                  # Integration test helpers
│   └── utils/                    # git_utils, github_utils, repo_root
│
├── templates/
│   ├── categories.yaml           # Single source of truth for all categories
│   ├── announcements.yaml        # Announcement content for READMEs
│   ├── README_AWESOME.template.md
│   ├── README_CLASSIC.template.md
│   ├── README_EXTRA.template.md
│   ├── footer.template.md
│   └── resource-overrides.yaml   # Per-resource display overrides
│
├── README.md                     # Generated — root style (currently: awesome)
├── README_ALTERNATIVES/          # Generated — all 47+ README variants
├── assets/                       # SVG badges, headers, banners
│
├── data/                         # Repo ticker CSVs (auto-updated by CI)
├── docs/                         # Developer documentation
│   ├── CONTRIBUTING.md
│   ├── README-GENERATION.md      # Detailed generation architecture
│   ├── TESTING.md
│   └── development/              # Architecture notes and gotchas
│
├── resources/                    # Curated content (downloaded/mirrored)
│   ├── claude.md-files/          # Example CLAUDE.md files from listed repos
│   ├── slash-commands/           # Example slash-command files
│   ├── workflows-knowledge-guides/
│   └── official-documentation/
│
├── tests/                        # pytest test suite
└── tools/                        # Dev tools (e.g., readme_tree updater)
```

---

## Core Data Model

### `THE_RESOURCES_TABLE.csv`

Every resource is a single CSV row. Key columns:

| Column | Description |
|---|---|
| `ID` | `{prefix}-{hash}` (e.g., `skill-ca8cbc21`) — never change once set |
| `Display Name` | Human-readable name |
| `Category` | Must match a `name` in `templates/categories.yaml` |
| `Sub-Category` | Must match a subcategory under the parent category |
| `Primary Link` | Main resource URL (must be accessible, 200 OK) |
| `Secondary Link` | Optional secondary URL |
| `Author Name` / `Author Link` | Attribution |
| `Active` | `TRUE` / `FALSE` |
| `Date Added` | Format `YYYY-MM-DD:HH-MM-SS` |
| `License` | SPDX identifier or `No License / Not Specified` |
| `Description` | Markdown-safe description string |

**Rule:** `README.md` and all `README_ALTERNATIVES/` files are 100% generated from this CSV. Never hand-edit the generated READMEs.

### `templates/categories.yaml`

The single source of truth for all categories and subcategories. Adding a category here propagates everywhere — generators, validation, the `add-category` tool, and ID prefixes.

### `acc-config.yaml`

Controls which README style is written to the repo root (`readme.root_style`) and the style-selector badge order and colors.

---

## Development Workflows

### Setup

```bash
python3 -m venv venv
make install        # pip install -e ".[dev]"
```

### Day-to-day commands

```bash
make generate       # Sort CSV then regenerate all READMEs + badges
make test           # Run pytest
make mypy           # Type-check scripts/ and tests/
make format         # Auto-fix with ruff
make format-check   # Check only (used in CI)
make ci             # format-check + mypy + test + docs-tree-check
make validate       # Check all resource URLs in CSV (network-heavy)
make validate-single URL=https://...   # Check one URL
make sort           # Sort CSV (also runs automatically before generate)
make clean          # Remove caches
```

### Adding a Resource (manual)

1. Add a row to `THE_RESOURCES_TABLE.csv` (use `make generate-resource-id` to compute the ID).
2. Run `make generate` — regenerates all READMEs.
3. Commit the CSV change and the generated README files together.

### Adding a Category

```bash
make add-category
# or with args:
make add-category ARGS='--name "My Category" --prefix mycat --icon 🎯'
```

This updates `templates/categories.yaml`. Then run `make generate`.

### README Generation Architecture

Generation runs in two phases:

1. All style variants are written to `README_ALTERNATIVES/`.
2. The configured `root_style` is also written to `README.md`.

Generator hierarchy (`scripts/readme/generators/`):

```
ReadmeGenerator (ABC)          base.py
├── VisualReadmeGenerator      → README_EXTRA.md       (Extra style)
├── MinimalReadmeGenerator     → README_CLASSIC.md     (Classic style)
├── AwesomeReadmeGenerator     → README_AWESOME.md     (Awesome style)
└── ParameterizedFlatListGenerator → README_FLAT_*.md (44 views: 11 cats × 4 sorts)
```

The Flat style simulates sorting/filtering by generating every permutation of `category × sort_type` as a separate file. **Adding a new flat category** requires updating `FLAT_CATEGORIES` in `scripts/readme/generators/flat.py`, then running `make generate`.

---

## CI / Pre-commit

Pre-commit hooks (`.pre-commit-config.yaml`) run on every Python file commit:
- `ruff` linter + formatter (auto-fix)
- `make test` (pytest)
- README regeneration check (`make generate && git diff --exit-code README.md`)

GitHub Actions (`.github/workflows/`) handle:
- `ci.yml` — format-check, mypy, tests, docs-tree-check
- `submission-enforcement-v2.yml` — validates issue form submissions
- `validate-links.yml` — periodic URL health check
- `check-repo-health.yml` — repo health metrics
- `update-repo-ticker.yml` — updates `data/repo-ticker.csv`
- `notify-on-merge.yml` — badge notifications to resource authors

---

## Resource Submission System

Submissions come exclusively through the GitHub Issue form template (`recommend-resource.yml`). The bot pipeline:

1. Issue opened → `resource-submission` label applied automatically.
2. `submission-enforcement-v2.yml` runs validation (URL reachability, duplicates, required fields).
3. Maintainer approves with `/approve` → `create_resource_pr.py` opens a PR automatically.
4. PR merges → badge notification sent to resource author's repo.

**Key labels:** `resource-submission`, `validation-passed`, `validation-failed`, `approved`, `pr-created`, `changes-requested`, `rejected`, `broken-links`, `do-not-disturb`.

---

## Coding Conventions

- **Run scripts with `python -m`**, using dot notation: `python -m scripts.resources.parse_issue_form` (not slash paths, no `.py`).
- All scripts that are directly invocable have `if __name__ == "__main__":`.
- **Imports:** `scripts.*` is the package root; always import from repo root or set `PYTHONPATH`.
- **Ruff line length:** 100 chars. `scripts/archive/` is excluded from linting.
- **Type annotations:** Required on all new code; `mypy` is enforced in CI.
- **Tests:** Place in `tests/`. Temp-directory tests must create a minimal `pyproject.toml` so `find_repo_root()` works correctly.
- `scripts/archive/` is excluded from tests, coverage, and linting — do not touch.
- Never commit generated READMEs without also committing the CSV changes that produced them.
- `GITHUB_TOKEN` env var is required for scripts that call the GitHub API (avoids rate limits).

---

## Resource ID Format

```
{prefix}-{sha256(display_name + primary_link)[:8]}
```

Prefixes are defined per-category in `templates/categories.yaml` (e.g., `skill`, `cmd`, `wf`, `tool`, `claude`, `hook`, `doc`). IDs are **immutable** once assigned.

---

## Key Gotchas

- GitHub Actions with sparse checkout must include `pyproject.toml` in the checkout so `find_repo_root()` can locate the repo root.
- Sparse checkout must also include any data files scripts read: `THE_RESOURCES_TABLE.csv`, `templates/`, etc.
- Running near midnight may cause `test-regenerate` to fail due to date-stamp changes in generated READMEs — rerun when the clock is stable.
- `make test-regenerate` requires a clean working tree (`git status` must be empty). Use `make test-regenerate-allow-diff` to bypass.
- The Flat README system generates 44+ files. Removing a flat category requires manually deleting the orphaned `.md` files from `README_ALTERNATIVES/` after updating `FLAT_CATEGORIES`.
- Resource recommendations via the `gh` CLI are explicitly blocked — only the GitHub web UI issue form is accepted.

---

## Documentation Index

| File | Purpose |
|---|---|
| `docs/CONTRIBUTING.md` | Submission guidelines and process |
| `docs/README-GENERATION.md` | Full generation architecture reference |
| `docs/HOW_IT_WORKS.md` | Submission flow, GitHub labels, stats integration |
| `docs/TESTING.md` | Test setup, coverage, regeneration cycle tests |
| `docs/COOLDOWN.md` | Submission cooldown / ban rules |
| `docs/development/do-not-forget.md` | Critical gotchas for developers |
| `docs/development/toc-anchor-generation.md` | TOC anchor rules |
| `docs/development/summary-rendering-cheatsheet.md` | Rendering reference |
