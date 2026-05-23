# Design Reference

How to change **how things look or render** — date formats, layout structure, colors, labels — on this Chirpy-themed Jekyll site. Hand this doc to Claude alongside a plain-English request like *"show dates as 'March 2026' instead of 'May 23, 2026'"* and Claude has enough to find the right file and make the edit.

---

## 1. The override model (read this first)

The theme files live in the **gem**, not in this repo:

```
/opt/homebrew/lib/ruby/gems/3.3.0/gems/jekyll-theme-chirpy-7.5.0/
├── _data/        ← locale strings, origin config
├── _includes/    ← partials (sidebar, footer, datetime, …)
├── _layouts/     ← page templates (home, post, default, …)
├── _sass/        ← stylesheet sources
└── assets/       ← JS, fonts, CSS bundle entrypoints
```

**To customize any of those, copy the file from the gem into the same relative path inside this repo.** Jekyll prefers local files over gem files. Examples:

- Change the sidebar HTML → copy `<gem>/_includes/sidebar.html` → `_includes/sidebar.html`, edit the copy.
- Change a color variable → copy `<gem>/_sass/addon/variables.scss` → `_sass/addon/variables.scss`, edit.
- Change the home layout → copy `<gem>/_layouts/home.html` → `_layouts/home.html`, edit.

This means: **never edit files in `/opt/homebrew/lib/ruby/gems/...`** — they'll be wiped on the next gem update and aren't versioned with the repo. Always copy → edit.

Restart `jekyll serve` after creating a new override file. Subsequent edits hot-reload.

---

## 2. Date formats (the most common customization)

There are **two** date formats per locale: `strftime` (the server-rendered initial HTML) and `dayjs` (what the client-side JS swaps in once loaded, used for tooltips). To get a consistent look you usually change both to match.

### Where they're defined

`<gem>/_data/locales/en.yml`:

```yaml
df:
  post:
    strftime: "%b %e, %Y"   # → "May 23, 2026"
    dayjs: "ll"             # → "May 23, 2026" (dayjs `localizedFormat`)
  archives:
    strftime: "%b"          # → "May" (archive page month headings)
    dayjs: "MMM"
```

### How to override

Create `_data/locales/en.yml` in this repo. **You must copy the whole file** (not just the `df:` block) — Jekyll doesn't deep-merge data files; a local one fully replaces the gem one. Then edit only the keys you want to change.

### Format reference

- `strftime` codes: <https://strftime.org/>
- `dayjs` tokens: <https://day.js.org/docs/en/display/format>

### Common recipes

| Desired output | `strftime` | `dayjs` |
|---|---|---|
| `May 23, 2026` (default) | `%b %e, %Y` | `ll` |
| `March 2026` | `%B %Y` | `MMMM YYYY` |
| `Mar 2026` | `%b %Y` | `MMM YYYY` |
| `2026-05-23` | `%Y-%m-%d` | `YYYY-MM-DD` |
| `23 May 2026` | `%e %b %Y` | `D MMM YYYY` |
| `Saturday, May 23` | `%A, %b %e` | `dddd, MMM D` |

### Where dates appear (so you know which key to change)

| Page | Renders via | Locale key |
|---|---|---|
| Home post card date | `_includes/datetime.html` | `df.post.*` |
| Post header date | `_layouts/post.html` → `datetime.html` | `df.post.*` |
| Last-modified date in posts | `datetime.html` | `df.post.*` |
| Archives page year/month headings | `_layouts/archives.html` | `df.archives.*` |
| Tooltips on hover | `datetime.html` `data-df` (dayjs) | `df.post.dayjs` |

> If you want one of these (e.g. archive headings) to use a *different* format from post-card dates, you can: (a) override just `df.archives.*` in the locale file, or (b) copy the relevant include/layout and hard-code a `date:` filter call inline.

### The actual rendering snippet

`<gem>/_includes/datetime.html`:

```html
{% assign df_strftime = site.data.locales[include.lang].df.post.strftime | default: '%d/%m/%Y' %}
{% assign df_dayjs    = site.data.locales[include.lang].df.post.dayjs    | default: 'DD/MM/YYYY' %}

<time
  data-ts="{{ include.date | date: '%s' }}"
  data-df="{{ df_dayjs }}"
>
  {{ include.date | date: df_strftime }}
</time>
```

The `<time>` tag is hydrated by `assets/js/dist/page.min.js` (built from the theme's JS sources) which reads `data-ts` (Unix timestamp) + `data-df` (dayjs format) and replaces the inner text on the client.

---

## 3. Other UI text (labels, copy)

All visible strings live in `_data/locales/<lang>.yml`. Override pattern is the same as §2 (copy whole file, edit keys). Highlights:

| Key | Where it shows | Default |
|---|---|---|
| `tabs.home` | "HOME" in sidebar | `Home` |
| `tabs.categories` / `.tags` / `.archives` / `.about` | Sidebar tab labels | `Categories`, etc. |
| `search.hint` | Placeholder text in the search box | `search` |
| `search.no_results` | When search finds nothing | `Oops! No results found.` |
| `panel.lastmod` | Right-side panel title above recent posts | `Recently Updated` |
| `panel.trending_tags` | Right-side panel title | `Trending Tags` |
| `panel.toc` | TOC heading on posts | `Contents` |
| `copyright.brief` / `.verbose` | Footer "Some rights reserved." + tooltip | — |
| `meta` | "Using the :THEME theme for :PLATFORM." in the footer | — |
| `post.written_by`, `posted`, `updated`, `share`, `relate_posts` | Post page labels | `By`, `Posted`, `Updated`, `Share`, `Further Reading` |
| `post.read_time.unit` / `.prompt` | "5 min read" | `min` / `read` |
| `post.button.next` / `.previous` | Post nav buttons | `Newer` / `Older` |
| `not_found.statement` | 404 page message | — |
| `notification.update_found` / `update` | "New version available" toast | — |

Sidebar labels are upper-cased in the template (`| upcase` filter in `_includes/sidebar.html`) — you write `Home` in YAML, it renders `HOME`. To keep the casing as written, you'd need to override `sidebar.html` and remove `| upcase`.

---

## 4. Colors, fonts, spacing

Chirpy uses SCSS. Variables live in `<gem>/_sass/`:

```
_sass/
├── addon/
│   ├── variables.scss      ← color palette, sizes (light mode + structural)
│   ├── module.scss
│   ├── syntax.scss
│   └── commons.scss
├── colors/
│   ├── light-typography.scss   ← light-mode color tokens
│   ├── dark-typography.scss    ← dark-mode color tokens
│   └── syntax-light.scss / syntax-dark.scss
├── layout/                 ← per-page layout styles (post, home, sidebar, …)
└── main.bundle.scss        ← entry point
```

### To change a color

1. Open `<gem>/_sass/colors/light-typography.scss` (and `dark-typography.scss` for dark mode), find the SCSS variable (e.g. `--link-color`, `--site-title-color`).
2. Copy the whole `light-typography.scss` (or `dark-`) to `_sass/colors/light-typography.scss` in this repo.
3. Edit the variable in the copy.

### To override only a few variables without copying the whole file

Create `assets/css/jekyll-theme-chirpy.scss` in this repo with:

```scss
---
---

// Your overrides FIRST
:root {
  --link-color: #c44d56;
  --site-title-color: #222;
}

// Then import the theme
@import "main";
```

> The two `---` lines at the top are required by Jekyll's front matter for SCSS files. Don't remove them.

### To change fonts

Same pattern. Look for `font-family` declarations in `_sass/addon/commons.scss` and `_sass/addon/module.scss`. Easiest path: add a `@import url(...)` to your custom `assets/css/jekyll-theme-chirpy.scss` and override `--bs-font-sans-serif` / `--bs-font-monospace`.

---

## 5. Layout structure (where things sit on the page)

| Layout file | What it controls |
|---|---|
| `_layouts/default.html` | The shell: sidebar + topbar + content + footer wrapper. Edit to restructure the whole page frame. |
| `_layouts/home.html` | The post-list home page (the post cards). |
| `_layouts/post.html` | A single post (header, body, TOC, share, related, pagination). |
| `_layouts/page.html` | Used by the About tab and any custom tab without an explicit layout. |
| `_layouts/archives.html` | Archives tab (chronological list). |
| `_layouts/categories.html` / `category.html` | Categories tab + per-category pages. |
| `_layouts/tags.html` / `tag.html` | Tags tab + per-tag pages. |
| `_layouts/compress.html` | HTML minification wrapper. Don't touch. |

Common partials in `_includes/`:

| Include | What it is |
|---|---|
| `sidebar.html` | Left sidebar: avatar, title, tagline, nav, social icons |
| `topbar.html` | Top bar with breadcrumbs + search |
| `footer.html` | Bottom copyright + "Using the Chirpy theme for Jekyll" line |
| `datetime.html` | The `<time>` element used everywhere a date renders (§2) |
| `post-nav.html` | Older / Newer buttons under a post |
| `post-sharing.html` | Share buttons row under a post |
| `post-paginator.html` | Numbered pagination on the home list |
| `post-summary.html` | The card excerpt on the home page |
| `read-time.html` | "X min read" label |
| `related-posts.html` | "Further Reading" block under a post |
| `toc.html` / `toc-status.html` | Right-side table of contents |
| `trending-tags.html`, `update-list.html` | Right-side panels |
| `comments/*` | Disqus / utterances / giscus partials |
| `analytics/*` / `pageviews/*` | Per-provider tracker snippets |
| `head.html` | `<head>` block (meta tags, theme color, favicons) |
| `favicons.html` | Favicon link tags |

To change one of these: copy from gem → edit locally.

---

## 6. Front-page card behavior — `_layouts/home.html`

Currently each post card shows: cover image (if set) | title | excerpt | `📅 date  📁 categories  📌 pin`.

If you want different fields (e.g. show tags, show read time, hide categories), copy `<gem>/_layouts/home.html` → `_layouts/home.html` and adjust the `<div class="post-meta">` block. The variables you have access to per post: `post.title`, `post.date`, `post.last_modified_at`, `post.categories`, `post.tags`, `post.excerpt`, `post.image`, `post.pin`, `post.url`.

To change the date format **only on the home cards** (without touching the locale file), replace the `{% include datetime.html date=post.date lang=lang %}` call with something like:

```liquid
{{ post.date | date: "%B %Y" }}
```

---

## 7. Footer — `_includes/footer.html`

Currently shows `© <year> <social.name>. Some rights reserved.` on the left, and `Using the Chirpy theme for Jekyll.` on the right.

To change either side: copy → edit. The right side comes from the locale's `meta:` key with `:THEME` and `:PLATFORM` placeholders. To remove the attribution entirely, replace the `<p>` containing `{{ … meta … }}` with empty markup (but Chirpy's MIT license still requires attribution somewhere visible — putting it in About is acceptable).

---

## 8. Favicons & app icons

Lives at `assets/img/favicons/` (you'd create this if you want custom ones). The theme expects this exact filename set:

```
favicon.ico, favicon-16x16.png, favicon-32x32.png,
apple-touch-icon.png, android-chrome-192x192.png, android-chrome-512x512.png,
mstile-150x150.png, browserconfig.xml, site.webmanifest
```

Generate the set with <https://realfavicongenerator.net/> and drop into `assets/img/favicons/`. The theme picks them up automatically via `_includes/favicons.html`.

---

## 9. Mental model for telling Claude what to change

When you describe a change, Claude needs to know **which of the four layers** to touch:

1. **Data / strings** → `_data/locales/<lang>.yml` (labels, date formats)
2. **Markup / structure** → `_includes/*.html` and `_layouts/*.html` (what HTML gets emitted)
3. **Styling** → `_sass/**/*.scss` or `assets/css/jekyll-theme-chirpy.scss` (colors, spacing, fonts)
4. **Config** → `_config.yml` (site-wide knobs — title, theme mode, comments provider, etc.; see content-reference.md)

Examples of how to phrase a request and where it lands:

| Request | Layer | File |
|---|---|---|
| "Show dates as 'March 2026'" | Data | `_data/locales/en.yml` (`df.post.strftime` + `df.post.dayjs`) |
| "Make the sidebar title not all-caps" | Markup | `_includes/sidebar.html` (remove `\| upcase`) |
| "Use a navy blue accent color" | Styling | `assets/css/jekyll-theme-chirpy.scss` (override `--link-color` etc.) |
| "Hide the 'Using Chirpy theme' footer line" | Markup | `_includes/footer.html` |
| "Remove the categories tab" | Content | delete `_tabs/categories.md` (see content-reference.md) |
| "Show 5 posts per page instead of 10" | Config | `_config.yml` `paginate` |
| "Default to dark mode" | Config | `_config.yml` `theme_mode: dark` |
| "Show read-time on home cards" | Markup | `_layouts/home.html` (add `{% include read-time.html %}`) |
| "Change 'Recently Updated' panel title" | Data | `_data/locales/en.yml` `panel.lastmod` |

---

## 10. Workflow checklist

When making any design change:

1. **Identify the layer** (data / markup / styling / config) — see §9.
2. **If touching theme files** (markup, styling beyond simple variables), **copy from gem first** — never edit in `/opt/homebrew/lib/ruby/...`.
3. **Restart `jekyll serve`** if you (a) edited `_config.yml`, or (b) added a brand-new override file. Otherwise, hot-reload covers it.
4. **Verify in both light and dark mode** (toggle on the sidebar) — color changes often only affect one.
5. **Verify on mobile width** — the sidebar collapses below ~850px; some changes only show at one breakpoint.

