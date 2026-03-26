# anthonymineer.me

Personal site for [anthonymineer.me](https://anthonyminear.me) — built with [Hugo](https://gohugo.io/) and the [PaperMod](https://github.com/adityatelange/hugo-PaperMod) theme.

Cloud engineering, AI infrastructure, and platform thinking.

## Tech Stack

- **Static Site Generator:** Hugo
- **Theme:** PaperMod
- **Hosting:** GitHub Pages

## Local Development

**Prerequisites:** [Hugo](https://gohugo.io/installation/) installed

```bash
git clone https://github.com/AMineer/anthonymineer.me.git
cd anthonymineer.me
hugo server -D
```

The site will be available at `http://localhost:1313`.

## Content

Posts live under `content/field-notes/`. New posts follow the Hugo front matter convention:

```yaml
---
title: "Post Title"
date: 2026-01-01
tags: ["cloud", "ai"]
draft: false
---
```

## Project Structure

```
.
├── archetypes/       # Content templates
├── assets/           # Processed assets (CSS, JS)
├── content/          # Markdown content
│   ├── field-notes/  # Blog posts
│   ├── about/
│   └── contact/
├── layouts/          # Custom Hugo templates
├── static/           # Static files (images, favicon)
├── themes/PaperMod/  # Theme
└── hugo.toml         # Site configuration
```

## License

Content © Anthony Mineer. All rights reserved.
