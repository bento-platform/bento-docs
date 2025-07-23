# Bento Docs

This repository hosts the versioned documentation for the **Bento** portal, built with [MkDocs](https://www.mkdocs.org/) and [Mike](https://github.com/jimporter/mike), and deployed to GitHub Pages.

## Installation

Install dependencies with Poetry:

```bash
poetry install
```

(Optional) Activate the Poetry shell:

```bash
poetry shell
```

## Local Development

Preview the docs with live reload:

```bash
mkdocs serve
```

Build the static site:

```bash
mkdocs build --strict
```

## Deployment

Manual deploy with Mike:

```bash
mike deploy <version> --push            # deploys docs under /<version>/ on gh-pages
mike alias <version> latest --update-aliases --push  # update the latest alias
```

## CI/CD

- **Release deploy**: GitHub Actions workflow (`.github/workflows/deploy-docs.yml`) runs on GitHub Release to deploy the tagged version and update `latest`.
- **Docs build & test**: CI workflow (`.github/workflows/ci.yml`) on push/PR builds docs (`mkdocs build --strict`) and runs tests.
