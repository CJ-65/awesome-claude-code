# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

Awesome Claude Code is a curated list of Claude Code resources (slash-commands, CLAUDE.md files, hooks, agent skills, tooling, workflows, official docs). It is implemented as a small "full-stack" system: `THE_RESOURCES_TABLE.csv` is the single source of truth, and every README variant (and most SVG assets) is generated from it. **Do not hand-edit generated files** — edit the CSV/templates/generators and regenerate instead.

## Commands

```bash
make install              # Install package + dev deps (pip install -e ".[dev]")
make test                 # Run pytest suite (tests/)
make coverage             # pytest with term/html/xml coverage reports
make mypy                 # mypy scripts tests
make format                # ruff check --fix + ruff format
make format-check         # ruff check + ruff format --check (no changes)
make ci                   # format-check + mypy + test + docs-tree-check (mirrors CI)
make generate             # sort CSV, then regenerate all README styles + assets
make generate-toc-assets  # regenerate subcategory TOC SVGs (after categories.yaml changes)
make add-category         # interactive tool to add a category (edits categories.yaml)
make validate             # validate all resource links in the CSV
make validate-single URL=<url>       # validate one URL
make docs-tree             # refresh the file-tree block in docs/README-GENERATION.md
make docs-tree-check       # fail if that tree block is stale
make test-regenerate       # regenerate READMEs from a clean tree; fail on any diff
make test-regenerate-cycle # full regen integration test (root_style + selector order)
make clean / clean-all     # remove caches/artifacts (clean-all also removes venv/)
```

Run a single test: `venv/bin/python3 -m pytest tests/test_generate_readme.py::test_name -v` (or `python3 -m pytest ...` in CI, where `CI=true` selects system Python — see the `Makefile` `PYTHON` variable).

Pre-commit hooks (`.pre-commit-config.yaml`) run ruff lint/format, `make test`, and a check that `README.md` matches what `make generate` produces — expect these to run on commit.

## Architecture

### The multi-list generation pipeline

1. **`THE_RESOURCES_TABLE.csv`** (repo root) — master data: Display Name, Primary Link, Author Name/Link, Description, Category, Sub-Category, Active, Removed From Origin.
2. **`templates/categories.yaml`** — category/subcategory definitions (id, name, icon, order).
3. **`acc-config.yaml`** — which README style is written to root `README.md` (`readme.root_style`), plus style-selector metadata/order.
4. **`scripts/readme/generate_readme.py`** — entrypoint (`make generate`). Runs generator classes for every style, always writing all styles under `README_ALTERNATIVES/`, then writes the configured `root_style` an extra time to `README.md`.
5. Generator classes (`scripts/readme/generators/`) extend `ReadmeGenerator` (ABC, Template Method pattern) in `generators/base.py`:
   - `VisualReadmeGenerator` → Extra style (`README_EXTRA.md`, SVG-heavy, CRT/vintage themes)
   - `MinimalReadmeGenerator` → Classic style (`README_CLASSIC.md`)
   - `AwesomeReadmeGenerator` → Awesome-list style (`README_AWESOME.md`) — currently the root style
   - `ParameterizedFlatListGenerator` → Flat style, parameterized by `category_slug` × `sort_type`, producing 44 `README_FLAT_*.md` table views from one class
   - Registered in `STYLE_GENERATORS` inside `generate_readme.py`; adding a style means: new template in `templates/`, new generator class, register it, add a style badge, add an `acc-config.yaml` entry.
6. Supporting modules:
   - `scripts/readme/helpers/` — config loading (`readme_config.py`), shared parsing/anchor utilities (`readme_utils.py`), SVG asset writers (`readme_assets.py`), path token resolution (`readme_paths.py`).
   - `scripts/readme/markup/` — per-style Markdown/HTML renderers (`awesome.py`, `flat.py`, `minimal.py`, `visual.py`, `shared.py`).
   - `scripts/readme/svg_templates/` — SVG renderers for badges, headers, dividers, TOC elements.
   - Templates use `{{PLACEHOLDER}}` tokens (e.g. `{{ASSET_PATH('x.svg')}}`, `{{STYLE_SELECTOR}}`, `{{TABLE_OF_CONTENTS}}`, `{{BODY_SECTIONS}}`); content outside placeholders is manual copy, untouched by generation. Generated files are prefixed with `<!-- GENERATED FILE: do not edit directly -->`.

### Path resolution (fragile — see `docs/README-GENERATION.md`)

- Repo root is discovered by walking up from a file until `pyproject.toml` is found (`scripts/utils/repo_root.py`; tests replicate this in `tests/conftest.py`). Any test using temp directories with path resolution needs a stub `pyproject.toml` in the temp root.
- Exactly one style is `root_style` and lives at `README.md`; every style (including root) also lives under the fixed `README_ALTERNATIVES/` directory; `assets/` is a fixed direct child of repo root. Asset/link prefixes differ between root (`assets/`, `README_ALTERNATIVES/file.md`) and alternatives (`../assets/`, bare `file.md`). Don't rename `README_ALTERNATIVES/`, nest alternative folders, or introduce a second root README — these assumptions are asserted by `tests/test_style_selector_paths.py`.

### Resource submission automation (GitHub-driven, not PR-driven)

New resources are **not** added via hand-written PRs. Users submit via the `recommend-resource.yml` issue form; GitHub Actions (`.github/workflows/*`, `scripts/resources/`) parse the issue, validate it (`scripts/resources/detect_informal_submission.py`, `scripts/validation/validate_links.py`), and — once a maintainer runs `/approve` — auto-append a row to the CSV, regenerate READMEs, and open the PR (`scripts/resources/create_resource_pr.py`, `resource_utils.py`). State is tracked via issue labels (`resource-submission`, `validation-passed/failed`, `approved`, `pr-created`, `changes-requested`, `rejected`). Resource IDs are `{category-prefix}-{sha256(name+link)[:8]}` (`scripts/ids/`). See `docs/HOW_IT_WORKS.md` for the full label/state machine.

### Other scripts

- `scripts/categories/` — category management (`category_utils.py`, `add_category.py`, both marked experimental in docs).
- `scripts/ticker/` — fetches GitHub stats for featured repos (`fetch_repo_ticker_data.py`, needs `GITHUB_TOKEN`) and renders animated scrolling SVG tickers (`generate_ticker_svg.py`) into `data/repo-ticker*.csv` / `assets/repo-ticker*.svg`.
- `scripts/maintenance/` — repo health checks, GitHub release data updates.
- `scripts/badges/` — notifies resource authors' repos when their resource is merged in (skippable via `do-not-disturb` label).
- `tools/readme_tree/` — regenerates the file-tree block embedded in `docs/README-GENERATION.md` from `tools/readme_tree/config.yaml`.

### Tests

`tests/` mirrors `scripts/`; `tests/conftest.py` provides a `repo_root` fixture and a `github_stub` fixture (monkeypatches `PyGithub`'s `Github` client with dummy objects to avoid network calls in tests exercising `scripts/utils/github_utils.py` or badge notifications). `scripts/archive/` is excluded from test discovery, mypy, and coverage.

## Key docs

- `docs/README-GENERATION.md` — authoritative reference for the generation pipeline, adding categories/subcategories/styles/sort-types, asset types, and path-resolution rules. Read this before touching anything under `scripts/readme/`.
- `docs/HOW_IT_WORKS.md` — submission/validation/PR-automation flow and label state machine.
- `docs/TESTING.md` — test/coverage/regeneration-cycle commands.
- `docs/CONTRIBUTING.md` — resource submission policy (submissions must go through the issue form, not PRs).
