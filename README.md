# Tejas Space

**Infrastructure engineering · Cloud architecture · Applied AI systems**

Tejas Space is my personal engineering website and technical blog. It brings together my professional background, selected projects, and notes from the systems I build, operate, investigate, and learn from.

**Website:** https://nio707ng-arch.github.io/tejas-space/

## About this site

My professional foundation is in engineering and IT infrastructure: Linux and Windows systems, networking, virtualization, storage, backup and recovery, and business-critical operations.

I came to artificial intelligence later in that journey. I am developing this area through hands-on experimentation and system building, extending my experience into cloud architecture, infrastructure as code, and AI systems engineering.

The writing here is intended to be practical and evidence-led. Articles document real engineering work: the problem, the approach, the decisions, what was tested, what did not work, and what I learned. Experiments and work in progress are identified as such rather than presented as finished products.

## Areas of interest

- **Infrastructure and operations** — Linux, Windows Server, networking, virtualization, storage, backup and recovery.
- **Cloud and automation** — AWS, Terraform, repeatable deployment, reliability and operational discipline.
- **AI systems engineering** — modular architectures, agents and capabilities, deterministic planning, knowledge continuity, integration boundaries and verification.

## Repository structure

This repository contains the Hugo source for the website.

- `content/` — site pages and blog posts.
- `static/` — static assets such as images and other files.
- `layouts/` — site templates and presentation.
- `hugo.toml` — Hugo site configuration.
- `.github/workflows/hugo.yml` — GitHub Actions workflow that builds and deploys the site to GitHub Pages.

## Local development

The site is built with [Hugo](https://gohugo.io/).

Run the development server:

```bash
hugo server
```

To preview draft and future-dated content locally:

```bash
hugo server --buildDrafts --buildFuture
```

Build the production site:

```bash
hugo --minify
```

## Publishing

The GitHub Actions workflow builds and deploys the site to GitHub Pages when changes are pushed to the `main` branch. It can also be run manually through the Actions tab.

Blog posts are written in Markdown under `content/posts/`. Posts can remain unpublished while being developed by setting `draft: true` in their front matter. Before publication, I review the content, set `draft: false`, and push the change to `main`.

## Editorial approach

This is a working portfolio and an ongoing engineering record—not a claim to have finished learning. The goal is to share useful technical work with enough context and evidence that readers can understand both the result and the reasoning behind it.

Some projects are active, some are experiments, and some are learning tracks. Their status should be understood from the individual project descriptions.
