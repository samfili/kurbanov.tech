# kurbanov.tech

Personal site and blog for Ivan Kurbanov (Head of DevOps & Infrastructure). Hugo static site, no theme, custom layouts and CSS, deployed to Vercel.

## Stack

- **Hugo** (extended not required) — static site generator. Pinned in `vercel.json` via `HUGO_VERSION`.
- **Vercel** — hosting. Build runs `hugo --gc --minify`, output in `public/`.
- **No JS framework, no theme.** Everything is hand-rolled HTML/CSS in `layouts/` and `static/css/`.

## Layout

```
hugo.toml                 # site config — bilingual (ru default, en secondary)
vercel.json               # build config + Hugo version pin
layouts/
  _default/baseof.html    # base HTML shell, head, header/footer partials
  index.html              # homepage (intro + recent posts)
  blog/list.html          # /blog/ index
  blog/single.html        # individual article
  partials/header.html    # nav + lang switcher
  partials/footer.html    # copyright + RSS
static/
  css/style.css           # all styles, single file
content/
  blog/                   # Russian articles (default lang)
    _index.md             # blog section page
    *.md                  # one file per article
content/en/
  blog/                   # English translations
    _index.md
    *.md
```

## Bilingual content rules

- Russian is the default language. Russian content lives in `content/`, served at `/`.
- English content lives in `content/en/`, served at `/en/`.
- To add a new article: create `content/blog/<slug>.md`. If translating, also create `content/en/blog/<slug>.md` with the same filename — Hugo links them automatically as translations.
- Front matter: `title`, `date` (ISO YYYY-MM-DD), optional `description`, optional `tags`.
- The language switcher in the header is a hard-coded RU↔EN toggle — it links to the language root, not the page translation. Keep it that way unless asked to use `.Translations` for per-page translation links.

## Adding a blog post

1. Create `content/blog/<slug>.md` (Russian).
2. Front matter:
   ```yaml
   ---
   title: "Заголовок"
   date: 2026-05-05
   description: "Краткое описание для SEO/OG"
   tags: ["devops", "kubernetes"]
   ---
   ```
3. Body in Markdown. Inline HTML is allowed (`unsafe = true` in goldmark).
4. (Optional) Mirror in `content/en/blog/<slug>.md` for the English version.

## Local development

```bash
hugo server -D            # dev server with drafts at http://localhost:1313
hugo --gc --minify        # production build into public/
```

If `hugo` isn't installed: `brew install hugo`.

## Deploying

Pushes to the connected Git branch on Vercel auto-deploy. Vercel uses `vercel.json` — `HUGO_VERSION` is pinned there; bump it deliberately, not casually.

## Conventions and gotchas

- **No theme.** Don't add one. Don't run `hugo new theme`. Custom layouts live directly in `/layouts/`.
- **CSS is one file** (`static/css/style.css`) using CSS variables and a dark-mode media query. Keep it that way — no preprocessor, no PostCSS, no Tailwind.
- **No build step beyond Hugo.** No npm, no node_modules. If you're tempted to add a `package.json`, ask first.
- **Design language: minimal.** System font stack, generous whitespace, single accent color, max-width ~680px. Inspired by sereja.tech.
- **RSS** is generated automatically by Hugo for the home page and the `blog` section. Don't hand-roll.
- **Meta menus** (header nav) are configured per-language under `[[languages.ru.menu.main]]` / `[[languages.en.menu.main]]` in `hugo.toml`.
- **Permalinks**: blog posts use `/blog/:contentbasename/` (the URL comes from the *filename*, not the title — important for posts with Cyrillic titles, otherwise URLs get percent-encoded). Pinned Hugo: `0.161.1` (in `vercel.json`).
- Don't add tracking, analytics, or third-party JS without being asked.

## When making changes

- For style changes: edit `static/css/style.css` and verify in `hugo server`.
- For layout/template changes: edit files under `layouts/`. Test both `/` and `/en/` to make sure bilingual rendering still works.
- For new sections (beyond `blog`): create `content/<section>/_index.md` (RU), `content/en/<section>/_index.md` (EN), and a layout pair under `layouts/<section>/list.html` + `single.html` if it deviates from defaults.
