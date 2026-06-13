---
title: "How This Blog Is Deployed"
date: 2026-06-13
draft: false
tags: ["meta", "hugo", "cloudflare", "homelab", "ci-cd"]
---

This blog is a static site. There's no database, no server-side rendering, no PHP, no Node process keeping the lights on. Just files. Here's how it gets from a Markdown file on my machine to your browser.

## The Stack

- **Hugo** — static site generator. Takes Markdown, spits out HTML.
- **GitHub** — source of truth. The blog content lives in a monorepo alongside the rest of my homelab config.
- **GitHub Actions** — CI/CD. Watches for changes and triggers the build.
- **Cloudflare Workers** — hosts and serves the built site globally.

## How It's Structured

The blog source lives under `blog/` in the repo. Hugo's layout is pretty standard:

```
blog/
├── config/          # site settings (title, params, menus)
├── content/
│   ├── _index.md    # homepage content
│   ├── about.md
│   └── posts/       # one .md file per post
├── themes/
│   └── PaperMod/    # git submodule
├── wrangler.toml    # Cloudflare Workers config
└── hugo.toml        # Hugo entrypoint config
```

Posts are plain Markdown with a small block of front matter at the top:

```markdown
---
title: "How This Blog Is Deployed"
date: 2026-06-13
draft: false
tags: ["meta", "hugo", "cloudflare"]
---

Your content starts here.
```

That's it. Hugo reads the front matter, applies the theme template, and generates a static HTML file for each post. The `draft: false` line is the only thing standing between a post and the public internet.

## The Deploy Pipeline

When I push a commit that touches anything under `blog/`, a GitHub Actions workflow kicks off automatically:

```
git push
    |
    v
GitHub detects changes in blog/**
    |
    v
GitHub Actions runner (ubuntu-latest)
    |
    +-- actions/checkout@v4 (with submodules: true)
    |       fetches repo + PaperMod theme
    |
    +-- peaceiris/actions-hugo@v3
    |       installs Hugo extended
    |
    +-- hugo --minify --source blog
    |       builds static site into blog/public/
    |
    +-- cloudflare/wrangler-action@v3
            reads blog/wrangler.toml
            uploads blog/public/ to Cloudflare Workers
```

The whole thing takes about 30 seconds. By the time I've switched tabs, the post is live.

## The Cloudflare Side

Cloudflare Workers serves the static files from their global edge network. The config is minimal — `wrangler.toml` just tells Cloudflare where the built files are:

```toml
name = "bitrot-me"
compatibility_date = "2026-06-13"

[assets]
directory = "./public"
```

No Worker script. No custom routing logic. Just "here are some files, serve them." Cloudflare handles caching, HTTPS, and HTTP→HTTPS redirects automatically.

The domain (`bitrot.me`) is registered with Cloudflare, so adding it as a custom domain on the Worker is a one-click operation — no DNS propagation delays, no certificate provisioning drama.

## Why Not Just Host It on the Pi?

The homelab runs several services on a cluster of Raspberry Pi 5s, and the original plan was to serve the blog from there too — Hugo building on the Pi, nginx serving the output, traffic routed through a Cloudflare Tunnel. That setup worked, but it was solving the wrong problem.

A static site doesn't benefit from living close to its data. It has no data. What it benefits from is being *everywhere at once*, which is exactly what a CDN does and what a Pi in a closet does not.

Offloading to Cloudflare Workers also means the blog stays up even if the homelab goes down for maintenance, a Pi falls over, or I'm flashing new firmware at 2am. The rest of the lab can restart around it.

## The Authoring Workflow

Writing a post looks like this:

1. Create a new file in `blog/content/posts/`
2. Write Markdown
3. `git commit && git push`
4. Done

No build step locally. No deployment scripts. No SSHing into anything. If I want to preview before publishing, `hugo serve --source blog` runs a local dev server with live reload. The `draft: true` flag keeps something out of production until I'm ready.

The whole pipeline optimises for the thing that should be easy — writing — and keeps the infrastructure out of the way.
