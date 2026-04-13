# Bento Docs

This repository hosts the versioned documentation for the **Bento** portal, built with [MkDocs](https://www.mkdocs.org/) and [Mike](https://github.com/jimporter/mike), and deployed to GitHub Pages.

## Installation

Install dependencies with uv:

```bash
uv sync
```

## Local Development

Preview the docs with live reload:

```bash
uv run mkdocs serve
```

Build the static site:

```bash
uv run mkdocs build --strict
```

## Deployment

Manual deploy with Mike:

```bash
mike deploy <version> --push            # deploys docs under /<version>/ on gh-pages
mike alias <version> latest --update-aliases --push  # update the latest alias
```

## Versioning

Docs are versioned using [Mike](https://github.com/jimporter/mike) and deployed automatically based on branch:

| Branch | Deployed version |
|---|---|
| `main` | `main` (aliased as `latest`) |
| `docs/<version>` | `<version>` |

For example, pushing to `docs/pcgl-bento-docs` deploys a `pcgl-bento-docs` version in the version selector.

To remove an old version:

```bash
uv run mike delete <version> --push
```

## CI/CD

- **Active version deploy**: pushes to `main` or `docs/**` branches automatically deploy the corresponding version to GitHub Pages.
- **Docs build & test**: CI runs on all PRs to `main`, building docs with `mkdocs build --strict` to catch errors before merge.
