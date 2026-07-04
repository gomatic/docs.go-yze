# template.repo-docs

Canonical template for a project's **public docs** repository (`<org>/docs.<project>`). A project's developer and user documentation — contributing, design, rules — lives here as a public [Hugo](https://gohugo.io) site published via GitHub Pages. Private content (ideas, tasks, specs) lives in the project's hub repo (`project.<project>`), never here.

## Layout

| Path | Purpose |
|------|---------|
| [`content/`](content/) | Markdown documentation — the Hugo site content. |
| [`layouts/`](layouts/) | Hugo templates. |
| [`hugo.json`](hugo.json) | Hugo configuration. |
| [`.github/workflows/pages.yml.disabled`](.github/workflows/pages.yml.disabled) | The GitHub Pages build workflow, **disabled** by its `.disabled` suffix (GitHub Actions only runs `*.yml`). |
| [`Makefile`](Makefile) | Local preview and build. Run `make` for help. |

## Public, with a disabled workflow

Everything in this repo is **public** — it exists to be published, so there is no `public/`/`private/` split. Anything private (design notes, tasks, ideas, specs) belongs in the project's private hub repo (`project.<project>`).

GitHub Pages is unavailable on private repos in free orgs, so a docs repo ships ready to go public: the Pages workflow is present but **disabled** until you make the repo public and enable Pages.

## Going public

1. Enable Pages: **Settings → Pages → Source: GitHub Actions**.
2. Activate the workflow:
   ```bash
   git mv .github/workflows/pages.yml.disabled .github/workflows/pages.yml
   ```
3. Push. The whole site builds and publishes.

## Creating a new docs repo from this template

```bash
gh repo create <org>/docs.<project> --public --template nicerobot/template.repo-docs
```
