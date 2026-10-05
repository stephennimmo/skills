---
name: mkdocs
description: Material for MkDocs documentation sites. Use when creating or changing an MkDocs site, docs content, mkdocs.yaml, GitHub Pages deploy workflows, docs assets, or organization github.io documentation repositories.
---

Follow the `markdown` skill for page structure and tables.

Do not apply Red Hat branding unless the user asks for it. When they do, follow the `red-hat-branding` skill.

## Stack

- Theme: [Material for MkDocs](https://squidfunk.github.io/mkdocs-material)
- Config file: `mkdocs.yaml` (never `.yml`)
- Dev server: port `8000`, always with `--livereload`
- Python env: `.venv` at the repo root; install with `pip install -r requirements.txt`
- Hosting: GitHub Pages via GitHub Actions
- Org docs repos are named `${organization_name}.github.io` under `~/projects/github/${organization_name}/`

Prefer markdown extensions and Material theme features over custom HTML/JS/CSS for site functionality.

## Creating a Site

Follow [Creating your site](https://squidfunk.github.io/mkdocs-material/creating-your-site) and [Publishing with GitHub Actions](https://squidfunk.github.io/mkdocs-material/publishing-your-site/#with-github-actions).

```shell
ORG=exarep
SITE_DIR="${ORG}.github.io"
mkdir -p ~/projects/github/${ORG}
cd ~/projects/github/${ORG}
# create repo ${SITE_DIR}, then:
python3 -m venv .venv
source .venv/bin/activate
echo "mkdocs-material" > requirements.txt
pip install -r requirements.txt
# initialize mkdocs per Material docs, then rename mkdocs.yml → mkdocs.yaml
```

Always add a `README.md` covering clone, create the venv, install deps, and start the dev server.

## Local Development

```shell
python3 -m venv .venv
source .venv/bin/activate
pip install -r requirements.txt
mkdocs serve --livereload
```

Site: http://localhost:8000

## Layout

```
${org}.github.io/
├── docs/
│   ├── assets/                 # site chrome only (images, stylesheets, javascript)
│   │   ├── images/
│   │   └── stylesheets/
│   │       └── extra.css
│   ├── index.md
│   └── ...                     # everything outside assets/ is content
├── .github/workflows/ci.yaml
├── .gitignore
├── mkdocs.yaml
├── README.md
└── requirements.txt
```

- Put images, stylesheets, and javascript under `docs/assets/`. Content pages stay outside that folder.
- Ignore `.venv/`, `site/`, `__pycache__/`, `*.pyc`, and `.cache/`.

## `mkdocs.yaml` Baseline

Mirror this shape; adjust `site_*`, `repo_*`, logo, palette, and `nav` for the project. Default theme is plain Material — not Red Hat branded.

```yaml
site_name: Example
site_url: https://example.github.io
site_description: Short description of the documentation site.

repo_name: example/example.github.io
repo_url: https://github.com/example/example.github.io

theme:
  name: material
  logo: assets/images/logo.png
  favicon: assets/images/logo.png
  palette:
    - scheme: default
      primary: indigo
      accent: indigo
      toggle:
        icon: material/brightness-7
        name: Switch to dark mode
    - scheme: slate
      primary: indigo
      accent: indigo
      toggle:
        icon: material/brightness-4
        name: Switch to light mode
  features:
    - navigation.tabs
    - navigation.sections
    - navigation.expand
    - navigation.path
    - navigation.top
    - navigation.footer
    - toc.follow
    - search.suggest
    - search.highlight
    - content.code.copy
    - content.code.annotate
  icon:
    repo: fontawesome/brands/github

extra_css:
  - assets/stylesheets/extra.css

markdown_extensions:
  - admonition
  - attr_list
  - def_list
  - footnotes
  - md_in_html
  - tables
  - toc:
      permalink: true
  - pymdownx.details
  - pymdownx.highlight:
      anchor_linenums: true
      line_spans: __span
      pygments_lang_class: true
  - pymdownx.inlinehilite
  - pymdownx.superfences:
      custom_fences:
        - name: mermaid
          class: mermaid
          format: !!python/name:pymdownx.superfences.fence_code_format
  - pymdownx.tabbed:
      alternate_style: true
  - pymdownx.tasklist:
      custom_checkbox: true

plugins:
  - search

nav:
  - Home: index.md
```

`docs/assets/stylesheets/extra.css` is for site-specific overrides. Leave it empty unless the project needs custom CSS (including Red Hat branding when requested).

## GitHub Actions Deploy

`.github/workflows/ci.yaml` — deploy on push to `main` with `mkdocs gh-deploy --force` (same pattern as Material's GitHub Actions guide):

```yaml
name: ci

on:
  push:
    branches:
      - main

permissions:
  contents: write

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Configure Git Credentials
        run: |
          git config user.name github-actions[bot]
          git config user.email 41898282+github-actions[bot]@users.noreply.github.com
      - uses: actions/setup-python@v5
        with:
          python-version: 3.x
      - run: echo "cache_id=$(date --utc '+%V')" >> $GITHUB_ENV
      - uses: actions/cache@v4
        with:
          key: mkdocs-material-${{ env.cache_id }}
          path: .cache
          restore-keys: |
            mkdocs-material-
      - run: pip install -r requirements.txt
      - run: mkdocs gh-deploy --force
```

## Writing Docs

- Prefer admonitions, tabs, details, code annotations, and Mermaid fences over custom widgets.

## `requirements.txt`

Keep dependencies in `requirements.txt` (at least `mkdocs-material`). Never `pip install` packages ad hoc without recording them there.
