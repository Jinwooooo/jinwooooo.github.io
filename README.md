# Jinwooooo's Doodling Space

Personal blog at **[jinwooooo.dev](https://jinwooooo.dev)** — software engineering notes, write-ups, and things I'm learning. Built on the [Chirpy][chirpy] Jekyll theme.

## Local development

```shell
bundle install
bundle exec jekyll serve
```

Site is then available at `http://localhost:4000`.

Pre-push check (matches CI):

```shell
bundle exec htmlproofer _site --disable-external --enforce-https
```

## Deployment

`main` branch deploys automatically via `.github/workflows/pages-deploy.yml` (GitHub Actions → GitHub Pages → `jinwooooo.dev`).

## Layout

| Path | Purpose |
|---|---|
| `_posts/` | Published posts (`YYYY-MM-DD-slug.md`) |
| `_tabs/` | Sidebar tabs (about, archives, categories, tags) |
| `assets/img/posts/<slug>/` | Per-post images, referenced via the post's `media_subpath` |
| `assets/css/jekyll-theme-chirpy.scss` | User SCSS additions (lives outside the gem's PurgeCSS bundle) |
| `_layouts/`, `_includes/` | Repo-level overrides that shadow files from the Chirpy gem |
| `docs/` | Internal reference: post syntax, front matter, design notes |
| `plans/` | In-flight planning documents |

## Post conventions

See `docs/post-syntax-reference.md` and `docs/post-config-reference.md` for the full reference. Highlights:

- Filename: `_posts/YYYY-MM-DD-english-slug.md` (ASCII, kebab-case)
- Image folder: `assets/img/posts/<slug>/`, referenced via the post's `media_subpath`
- Image captions: italic line directly under the image (`_caption_`)
- Callouts: kramdown blockquotes followed by `{: .prompt-info }`, `prompt-tip`, `prompt-warning`, or `prompt-danger`

## Notable customizations from stock Chirpy

- Light mode only — dark-mode toggle removed (`_config.yml` `theme_mode: light`)
- Full-row-clickable category cards (`_layouts/categories.html` + `.stretched-link` / `.position-relative` re-declared in user SCSS to survive PurgeCSS)
- Next/prev post navigation hidden
- Custom domain `jinwooooo.dev` via repo `CNAME` + GoDaddy DNS

## License

Posts and post images (`_posts/`, `assets/img/posts/`) are © Jinwoo Chung, licensed under [CC BY 4.0][cc-by]. The Chirpy theme and starter scaffolding are MIT-licensed by the upstream [Chirpy][chirpy] project — see [its license][mit].

[chirpy]: https://github.com/cotes2020/jekyll-theme-chirpy/
[mit]: https://github.com/cotes2020/chirpy-starter/blob/master/LICENSE
[cc-by]: https://creativecommons.org/licenses/by/4.0/
