# Nikita Kibitkin Engineering Blog

Personal technical blog built with [Hugo](https://gohugo.io/) and deployed to GitHub Pages. The site uses custom templates under `layouts/` and does not depend on an external theme.

## Local development

Run a local preview:

```bash
hugo server
```

Build the production site:

```bash
hugo --gc --minify
```

## Content workflow

- Publish articles in `content/posts/`.
- Use `archetypes/default.md` with `hugo new content posts/<slug>.md`.
- Use `draft-template.md` when drafting outside Hugo.

## Deployment

GitHub Actions builds and deploys the site to GitHub Pages on every push to `main`.
