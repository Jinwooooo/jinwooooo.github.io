# Content Reference

What to edit when changing **content** on the site — your name, tabs, links, posts, etc. Everything below lives inside the repo (`jinwooooo.github.io/`). Edit, save, and the running `bundle exec jekyll serve` will hot-reload (with the exception of `_config.yml`, which needs a server restart).

---

## 1. Site identity — `_config.yml`

This is the single most important file. All keys are referenced by location (line numbers are the current defaults).

| Key | Line | What it controls | Example |
|---|---|---|---|
| `lang` | 9 | Site language (also picks locale file from `_data/locales/`) | `en` |
| `timezone` | 12 | IANA TZ name, used for post dates | `Asia/Seoul` |
| `title` | 17 | Main title in sidebar + browser tab | `Jinwoo Park` |
| `tagline` | 19 | Subtitle under the title in the sidebar | `Notes on X and Y` |
| `description` | 21 | SEO meta description + Atom feed description | — |
| `url` | 26 | Full site URL, no trailing `/` | `https://jinwooooo.github.io` |
| `github.username` | 29 | Auto-builds the GitHub social icon URL | `jinwooooo` |
| `twitter.username` | 32 | Auto-builds the Twitter/X social icon URL | (leave blank to hide) |
| `social.name` | 37 | Default post author + footer copyright owner | `Jinwoo Park` |
| `social.email` | 38 | Used by the email contact icon | `you@example.com` |
| `social.links` | 40–46 | List of profile URLs. **First entry is the copyright link in the footer.** Uncomment lines to add LinkedIn, Facebook, etc. | — |
| `webmaster_verifications.*` | 49–55 | Google/Bing/etc. site verification codes | — |
| `analytics.*` | 61–75 | GA/Goatcounter/Umami/Matomo/Cloudflare/Fathom IDs | — |
| `pageviews.provider` | 79 | Show view counts (currently only `goatcounter`) | — |
| `theme_mode` | 92 | `light`, `dark`, or empty (follow system + show toggle) | — |
| `avatar` | 102 | Avatar shown in the sidebar — local path under `assets/` or a full URL | `/assets/img/avatar.png` |
| `social_preview_image` | 106 | Default OG share image | — |
| `toc` | 109 | Global on/off for the right-side Table of Contents on posts | `true` / `false` |
| `comments.provider` | 113 | `disqus`, `utterances`, or `giscus` (blank = disabled) | — |
| `pwa.enabled` | 142 | Install-as-app feature | `true` |
| `pwa.cache.enabled` | 144 | PWA offline cache | `true` |
| `paginate` | 151 | Posts per page on the home list | `10` |
| `baseurl` | 154 | Subpath if the site isn't served at the domain root. **Leave empty** for `jinwooooo.github.io` (root). | — |

> **Restart required after editing `_config.yml`.** Stop `jekyll serve` with Ctrl-C, then re-run.

---

## 2. Tabs (the sidebar nav items) — `_tabs/`

Each `.md` file in `_tabs/` becomes one item in the left sidebar nav, **below "HOME"**. Order is controlled by the `order:` front-matter key (lower = higher in the list). HOME is always first and uses no file.

Current state:

| File | Layout | Icon (Font Awesome) | Order |
|---|---|---|---|
| `categories.md` | `categories` | `fas fa-stream` | 1 |
| `tags.md` | `tags` | `fas fa-tags` | 2 |
| `archives.md` | `archives` | `fas fa-archive` | 3 |
| `about.md` | `page` (default) | `fas fa-info-circle` | 4 |

### Editing an existing tab

- **Rename the visible label** — by default the label comes from `_data/locales/<lang>.yml` (`tabs.<filename>` key). To override per-tab, add a `title:` to the file's front matter, e.g. in `about.md`:
  ```yaml
  ---
  icon: fas fa-info-circle
  order: 4
  title: About Me
  ---
  ```
- **Change the icon** — pick any free Font Awesome class from <https://fontawesome.com/search?ic=free> and put it in `icon:`.
- **Change order** — bump the `order:` number.
- **Change content (About only)** — just write Markdown after the front-matter `---`. The other tabs (`archives`, `categories`, `tags`) use auto-generated layouts; don't add a body.

### Adding a new tab

Create `_tabs/<slug>.md`:
```yaml
---
icon: fas fa-briefcase
order: 5
title: Projects
---

Markdown body here.
```
The URL will be `/<slug>/` (controlled by `_config.yml` line 195).

### Removing a tab

Delete the file. (E.g. if you don't want a Tags tab at all, delete `_tabs/tags.md`.)

---

## 3. Social / contact icons (sidebar bottom) — `_data/contact.yml`

The icon row at the bottom of the sidebar. Each entry is one icon.

- `github`, `twitter` — auto-built from `_config.yml` `github.username` / `twitter.username`. Leaving those empty hides the icon.
- `email` — uses `_config.yml` `social.email`.
- `rss` — always works once the site is built.
- Any other type needs a `url:` field. Uncomment the templates at the bottom of the file (`linkedin`, `mastodon`, `stack-overflow`, `bluesky`, `reddit`, `threads`) and fill the URL.

To add a custom platform not on the template list, add:
```yaml
- type: youtube
  icon: "fab fa-youtube"
  url: "https://youtube.com/@yourchannel"
```

---

## 4. Post share buttons — `_data/share.yml`

Controls the share-this-post buttons under each post. Twitter / Facebook / Telegram are on by default. Uncomment any of LinkedIn, Weibo, Mastodon, Bluesky, Reddit, Threads to enable. Remove entries to hide them.

---

## 5. Posts — `_posts/`

Currently empty. To add a post, create a file named **`YYYY-MM-DD-slug.md`** (the date prefix is required), e.g.:

```
_posts/2026-05-23-hello-world.md
```

Minimal front matter:
```yaml
---
title: Hello World
date: 2026-05-23 14:30:00 +0900
categories: [Diary]
tags: [intro]
---

Body in Markdown.
```

Useful per-post front-matter knobs (all optional):

| Key | What it does |
|---|---|
| `pin: true` | Pins the post to the top of the home page |
| `hidden: true` | Hides from home/archives but still reachable by direct URL |
| `image:` (`path`, `alt`, `lqip`) | Cover image shown on the home card |
| `media_subpath` | Prefix prepended to relative image paths in the post |
| `toc: false` | Disable TOC for this post only |
| `comments: false` | Disable comments for this post only |
| `math: true` | Enable MathJax for this post |
| `mermaid: true` | Enable Mermaid diagrams |

Drafts: put files under `_drafts/` (no date prefix needed). They're invisible unless you run `bundle exec jekyll serve --drafts`.

---

## 6. Where each piece of content shows up

| Where on the page | Source |
|---|---|
| Sidebar title | `_config.yml` → `title` |
| Sidebar subtitle | `_config.yml` → `tagline` |
| Sidebar avatar | `_config.yml` → `avatar` |
| Sidebar nav items | `_tabs/*.md` (labels: `_data/locales/<lang>.yml` `tabs.*`) |
| Sidebar social icons | `_data/contact.yml` (+ `_config.yml` social fields) |
| Footer copyright name | `_config.yml` → `social.name` (linked to `social.links[0]`) |
| Footer copyright year | Auto (current year) |
| Home page post cards | Posts under `_posts/` |
| Browser tab title | `_config.yml` → `title` (+ post title on post pages) |
| `<meta>` SEO description | `_config.yml` → `description` |
| Atom feed | `/feed.xml` (auto, uses `title` + `description`) |

---

## 7. Things you almost never need to touch

- `Gemfile`, `Gemfile.lock` — dependencies. Only edit if you're updating the theme version.
- `_plugins/posts-lastmod-hook.rb` — auto-populates each post's "last modified" date from git history.
- `.github/workflows/` — GitHub Pages deploy automation.
- `tools/` — repo helper scripts.
- `assets/lib/` — git submodule pointing at Chirpy's static-asset CDN mirror. Don't edit by hand.
- `_site/`, `.jekyll-cache/` — build output. Ignore.

