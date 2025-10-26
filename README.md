# Homelab Docs (MkDocs + Material)

## Quick start
```bash
python -m venv .venv && source .venv/bin/activate
pip install mkdocs-material
mkdocs serve
```

Open http://127.0.0.1:8000 to view the site.

## Build & deploy
- Local build: `mkdocs build`
- GitHub Pages: push to `main` with the included workflow enabled in repo Settings → Pages.

## Customize the hero
Replace `docs/assets/images/hero-placeholder.jpg` with your own image and tweak `docs/stylesheets/extra.css`.
You can edit `overrides/home.html` for more control over the homepage.
