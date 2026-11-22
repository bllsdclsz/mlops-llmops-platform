# Development

## Setup

```bash
make install     # .venv with Python 3.14 and every locked package, plus pre-commit hooks
make check       # ruff, mypy, pytest: what CI runs for the Python code
```

The repository is one uv workspace (`pyproject.toml` at the root, members `churn` and `support-rag`) with a single `uv.lock`. Services generated from the [template](platform/service-template.md) are separate uv projects with their own lock files.

## Conventions

- **Latest stable versions.** Every tool uses its latest stable release, checked on PyPI or the release page; exceptions and their reasons are listed in [Versions](versions.md). Container images use exact tags, and CI actions are pinned by commit SHA.
- **Line endings are LF.** `.gitattributes` enforces it and the `mixed-line-ending` pre-commit hook checks it. On Windows, Python's `Path.write_text` writes CRLF, so generated files need `newline="\n"`.
- **Pre-commit hooks:** trailing whitespace, end of file, mixed line endings, YAML and TOML syntax, large files, ruff lint and format.
- **Verify against the real thing.** Unit tests use fakes (a hashing encoder, fake LLMs) so CI stays fast, but every feature is also checked end to end against the real containers, models or cluster, and the docs record what that showed.

## CI

One workflow per component. Path filters keep a change from running unrelated jobs.

| Workflow | Runs on changes to | Jobs |
| --- | --- | --- |
| `churn-ci` | `churn/`, `monitoring/`, `pyproject.toml`, `uv.lock` | lint, types, tests; image build (published to GHCR on `main`); Prometheus rules and Grafana dashboards |
| `support-rag-ci` | `support-rag/`, `pyproject.toml`, `uv.lock` | lint, types, tests; retrieval and safety evaluation gates |
| `platform-ci` | `platform/` | chart lint and schema checks, promotion rules, Terraform fmt/validate |
| `template-ci` | `templates/`, `platform/scripts/` | generates a service and runs its checks, chart validation and image build |
| `docs` | `docs/`, `zensical.toml`, `pyproject.toml`, `uv.lock` | builds this site in strict mode; deploys it to GitHub Pages from `main` |

Dependabot opens weekly update PRs for Python packages, GitHub Actions and Terraform providers.

## Documentation

This site is built with [Zensical](https://zensical.org/) from the Markdown files in `docs/`; the navigation is in `zensical.toml`.

```bash
make docs-serve     # live preview on http://127.0.0.1:8200
make docs-build     # strict build into site/, as CI does
```

Each component folder keeps a short `README.md` that links here, so the docs live in one place.

## Upgrading dependencies

```bash
uv lock --upgrade && make check
```

Feast holds back two packages (pandas, prometheus-client). Re-check both caps whenever Feast releases a new version.
