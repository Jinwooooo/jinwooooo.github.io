# Multi-language path-based deployment for jinwooooo.github.io

## Context

The blog at `/Users/jinwooooo/Documents/GitHub/jinwooooo.github.io` is a Jekyll Chirpy 7.5 site, currently single-build, single-language (`lang: en`), with 11 posts mixed in English and Korean in `_posts/`. The user is purchasing `jinwooooo.dev` next week (DNS/CNAME setup already drafted in `~/Documents/GitHub/customer-domain-plan.md`) and wants the same site to serve two language sections:

- `jinwooooo.dev/en/*` — English posts (the canonical/main version)
- `jinwooooo.dev/kr/*` — Korean translated posts
- A sidebar toggle (visual style: like the old light/dark mode button) that navigates between the same post in the other language, accepting full-page reload

After weighing two-repo, dual-build/dual-domain, and single-build/path-based options, **we picked single repo + single build + path prefixes**. Both languages share one Pages deployment, one CNAME, one Jekyll build, one Gemfile, one Chirpy version. Translations live as side-by-side files in one PR. The "no reload across language switch" goal would require adding a JS SPA layer (Turbo/htmx/View Transitions) — explicitly out of scope for now; user accepted the reload.

Pre-existing branch `8-language-subdomain-implementation` is the work branch — despite the name, the implementation is path-based, not subdomain-based.

## Architecture summary

```
_posts/
  en/                              # English posts (Jekyll picks up nested _posts/*)
    2024-01-12-tree-rotation-….md
    …
  kr/                              # Korean posts
    2024-03-28-map-gerrymandering.md
    …
_tabs/                             # duplicated per language (en-*.md, kr-*.md)
_includes/
  sidebar.html                     # Chirpy override: adds lang-toggle button
  lang-toggle.html                 # new: renders the button
  head.html                        # Chirpy override: html[lang] + hreflang tags
_layouts/
  home.html                        # Chirpy override: filters site.posts by page.lang
index.html                         # root → meta-refresh redirect to /en/
en/index.html                      # English home (layout: home, lang: en)
kr/index.html                      # Korean home (layout: home, lang: ko)
assets/img/posts/<slug>/           # image folders stay where they are
_config.yml                        # defaults block scopes lang+permalink per _posts/<lang>/
CNAME                              # contents: jinwooooo.dev
```

Chirpy is a gem-based theme: any `_includes/` or `_layouts/` file in the repo shadows the gem's same-named file. We override the minimum needed.

## Domain prerequisites (do these when buying `jinwooooo.dev`)

Follow `~/Documents/GitHub/customer-domain-plan.md` verbatim for the apex domain — that work is independent of the language split and already documented (TXT verification, A/AAAA records, CNAME file, Pages settings, HTTPS enforcement). The only adjustment: the `_config.yml url:` change there sets `https://jinwooooo.dev` once for the whole site, which is correct for the path-based architecture (no per-language url needed).

The language work in this plan can be developed and tested locally before the domain switchover; deploy to `jinwooooo.github.io` first, verify everything works at `https://jinwooooo.github.io/en/` and `/kr/`, then do the domain swap.

## Step 1 — Reorganize existing posts into `_posts/en/` and `_posts/kr/`

Audit each of the 11 posts in `_posts/` by reading the body and classifying by primary language:

| Current file | Target |
|---|---|
| `2019-07-01-two-level-cnn-lung-cancer-histology.md` | `_posts/en/` (verify) |
| `2019-10-01-hadoop-inmapper-combiner.md` | `_posts/en/` (verify) |
| `2019-12-01-deduplication.md` | `_posts/en/` (verify) |
| `2024-01-12-tree-rotation-avl-red-black-self-balance.md` | `_posts/en/` (verify) |
| `2024-02-22-repurposing-slack-bits-and-freed-blocks.md` | `_posts/en/` (verify) |
| `2024-03-28-map-gerrymandering.md` | `_posts/kr/` (MAP-series, Korean per memory) |
| `2024-06-22-map-surveillance.md` | `_posts/kr/` (MAP-series, Korean per memory) |
| `2024-07-15-map-ball-chess.md` | `_posts/kr/` (MAP-series, Korean per memory) |
| `2024-08-21-thinking-like-a-computer.md` | audit body — memory suggests Korean origin, may have been translated |
| `2025-08-30-nodejs-mongodb-multi-document-transactions.md` | audit body |
| `2026-04-23-cicd-acr-interaction.md` | audit body |

For each post: read the first ~50 lines of the body. If predominantly Korean → `_posts/kr/`. If predominantly English → `_posts/en/`. Move only the `.md` file; **do not move the `assets/img/posts/<slug>/` folders** — they stay at their current paths and continue to be referenced via `media_subpath` (per existing convention in `jinwooooo-github-io-migration.md` memory).

Add `lang_ref:` front-matter field where a translation counterpart exists (slug-matched). Leave it out otherwise — the sidebar toggle falls back to the other section's homepage when missing.

## Step 2 — Update `_config.yml`

Existing `defaults` block (lines 171–192) sets one permalink `/posts/:title/` for all posts. Replace with per-language scopes:

```yaml
defaults:
  - scope: { path: "_posts/en", type: posts }
    values:
      layout: post
      comments: true
      toc: true
      lang: en
      permalink: /en/posts/:title/
  - scope: { path: "_posts/kr", type: posts }
    values:
      layout: post
      comments: true
      toc: true
      lang: ko
      permalink: /kr/posts/:title/
  - scope: { path: _drafts }
    values: { comments: false }
  - scope: { path: "", type: tabs }
    values: { layout: page, permalink: /:title/ }
```

Leave top-level `lang: en` (line 9), `url:`, `baseurl: ""` as-is. The `lang:` field affects only theme UI strings on pages that don't override it (we override per-page below).

**Note on `jekyll-archives`** (lines 216–223): for v1, keep categories/tags pages as mixed-language (`/categories/python/` lists posts from both langs, each post rendered with a small lang badge). Per-language category/tag pages are a follow-up — `jekyll-archives` doesn't filter by language natively and would need a custom plugin or layout override that hand-rolls archive generation. Documented as deferred work.

## Step 3 — Lang-aware home pages

Create two homepages, each with `layout: home`:

- `en/index.html` — `layout: home`, `lang: en`, `permalink: /en/`
- `kr/index.html` — `layout: home`, `lang: ko`, `permalink: /kr/`

Shadow Chirpy's `_layouts/home.html` (copy from gem at `$(bundle info --path jekyll-theme-chirpy)/_layouts/home.html`) and replace `site.posts` references with `site.posts | where: "lang", page.lang`. This filters the post list, pagination, and pinned post logic.

For the root `/`, write `index.html` with a meta-refresh + JS fallback redirecting to `/en/`:

```html
---
permalink: /
sitemap: false
---
<!DOCTYPE html>
<meta http-equiv="refresh" content="0; url=/en/">
<script>location.replace('/en/');</script>
<link rel="canonical" href="/en/">
```

## Step 4 — Duplicate `_tabs/` per language

Current `_tabs/` has `about.md`, `archives.md`, `categories.md`, `tags.md`. Duplicate into language-prefixed versions:

- `_tabs/en-about.md`, `_tabs/en-archives.md`, `_tabs/en-categories.md`, `_tabs/en-tags.md` — each with `lang: en`, `permalink: /en/about/`, etc.
- `_tabs/kr-about.md`, `_tabs/kr-archives.md`, `_tabs/kr-categories.md`, `_tabs/kr-tags.md` — each with `lang: ko`, `permalink: /kr/about/`, etc.

Override `_layouts/page.html`, `_layouts/archives.html`, `_layouts/categories.html`, `_layouts/tags.html` (shadow from gem) and filter `site.posts` by `page.lang` — same pattern as home.

Chirpy's sidebar navigation lists `_tabs/` entries; it must show only the current language's tabs. Override `_includes/sidebar.html` to filter: `{% assign current_lang = page.lang | default: 'en' %}{% assign tabs = site.tabs | where: "lang", current_lang %}`.

## Step 5 — Sidebar language toggle button

Create `_includes/lang-toggle.html`:

```liquid
{% assign current = page.lang | default: 'en' %}
{% assign other = 'en' %}{% if current == 'en' %}{% assign other = 'ko' %}{% endif %}
{% assign other_url = '/kr/' %}{% if other == 'en' %}{% assign other_url = '/en/' %}{% endif %}

{% if page.lang_ref %}
  {% assign counterpart = site.posts | where: "lang", other | where: "slug", page.lang_ref | first %}
  {% if counterpart %}{% assign other_url = counterpart.url %}{% endif %}
{% endif %}

<a href="{{ other_url | relative_url }}" class="lang-toggle" aria-label="Switch language">
  <i class="fa-solid fa-globe"></i>
  <span>{% if current == 'en' %}한국어{% else %}EN{% endif %}</span>
</a>
```

In overridden `_includes/sidebar.html`, place `{% include lang-toggle.html %}` next to the social icons block (the same location the dark/light mode button used to occupy — see Chirpy's `_includes/sidebar.html` in the gem for the exact insertion point, near the `sidebar-bottom` div).

Add minimal CSS in `assets/css/jekyll-theme-chirpy.scss` (which Chirpy already imports) or a new `_sass/addon/lang-toggle.scss` — match the visual weight of Chirpy's existing sidebar icons.

## Step 6 — SEO: `html[lang]` and `hreflang`

Shadow `_includes/head.html` (copy from gem). Two changes:

1. Replace `<html lang="{{ site.lang }}">` (which is in `_layouts/default.html` actually — check both) with `<html lang="{{ page.lang | default: site.lang }}">`.
2. Inside `<head>`, add hreflang tags pointing to the counterpart:
   ```html
   {% if page.lang_ref %}
     {% assign other_lang = 'ko' %}{% if page.lang == 'ko' %}{% assign other_lang = 'en' %}{% endif %}
     {% assign counterpart = site.posts | where: "lang", other_lang | where: "slug", page.lang_ref | first %}
     {% if counterpart %}
       <link rel="alternate" hreflang="{{ other_lang }}" href="{{ counterpart.url | absolute_url }}">
       <link rel="alternate" hreflang="{{ page.lang }}" href="{{ page.url | absolute_url }}">
     {% endif %}
   {% endif %}
   ```

## Step 7 — PWA cache scoping

`_config.yml` line 137–146 has `pwa.enabled: true`. The service worker caches per-origin, so it'll naturally cover both `/en/` and `/kr/`. No config change needed. After switching domains, **users with the old PWA installed will need to refresh** to pick up the new manifest — document this as a one-time post-launch caveat.

## Critical files to modify

| File | Change |
|---|---|
| `_config.yml` | Replace `defaults` block (lines 171–192) with per-lang scopes |
| `_posts/en/*.md`, `_posts/kr/*.md` | Move existing posts here, classified by language |
| `_posts/*.md` (original) | Removed after move |
| `en/index.html` (new) | Home page, `lang: en`, `permalink: /en/` |
| `kr/index.html` (new) | Home page, `lang: ko`, `permalink: /kr/` |
| `index.html` | Replace with meta-refresh to `/en/` |
| `_tabs/en-*.md`, `_tabs/kr-*.md` (new) | Per-language tabs, language-prefixed permalinks |
| `_tabs/about.md`, `archives.md`, `categories.md`, `tags.md` | Remove (replaced by per-lang versions) |
| `_layouts/home.html` (new override) | Filter by `page.lang` |
| `_layouts/page.html`, `archives.html`, `categories.html`, `tags.html` (new overrides) | Filter by `page.lang` |
| `_includes/sidebar.html` (new override) | Filter tabs by `page.lang`; insert lang-toggle |
| `_includes/lang-toggle.html` (new) | The toggle button |
| `_includes/head.html` (new override) | `html[lang]` + hreflang tags |
| `CNAME` (new, when domain ready) | `jinwooooo.dev` — per `customer-domain-plan.md` Step 3 |

## Verification

Local build:
1. `bundle install` then `bundle exec jekyll serve` — open `http://localhost:4000`
2. `/` should redirect to `/en/`
3. `/en/` shows only English posts; `/kr/` shows only Korean posts
4. Sidebar nav on `/en/` shows `/en/about/`, `/en/archives/`, etc.; same for `/kr/`
5. Click any post → land on `/en/posts/<slug>/` or `/kr/posts/<slug>/`
6. Sidebar lang-toggle button visible at the social-icons position; click on `/en/posts/X` (with `lang_ref` set) → lands on `/kr/posts/X-translated/`; without `lang_ref` → lands on `/kr/`
7. View source on any post: `<html lang="en">` or `<html lang="ko">` matches; `<link rel="alternate" hreflang="...">` present when counterpart exists
8. `bundle exec htmlproofer _site --disable-external` passes

Deploy verification (after pushing to `8-language-subdomain-implementation` and merging to `main`):
9. Visit `https://jinwooooo.github.io/en/` and `https://jinwooooo.github.io/kr/` — same behavior as local
10. After domain switchover (separate `customer-domain-plan.md` execution): `https://jinwooooo.dev/en/`, `https://jinwooooo.dev/kr/` work; HTTPS green

## Deferred (not in this plan)

- **Per-language category/tag pages** — `jekyll-archives` doesn't filter by language; v1 ships with mixed-language category/tag pages. Revisit if it becomes confusing.
- **Translation of existing untranslated posts** — separate ongoing task; this plan only sorts what exists.
- **Browser-language-based auto-redirect at `/`** — current redirect is unconditional to `/en/`. If desired later, can add JS that checks `navigator.language` before redirecting.
- **No-reload SPA toggle** — Turbo/htmx/View Transitions integration. Architectural change, not bundled in.
