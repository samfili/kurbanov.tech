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
  blog/                   # all blog articles (translations by filename suffix)
    _index.md             # blog section page (Russian, default lang)
    _index.en.md          # blog section page (English)
    <slug>.md             # Russian article
    <slug>.en.md          # English translation of the same article
```

## Bilingual content rules

- Translations are matched **by filename suffix**, not by separate directories. Russian (default) is `<slug>.md`, English is `<slug>.en.md`. Both live side-by-side in `content/blog/`.
- This is the standard Hugo "translation by filename" pattern. Avoid the alternate "translation by directory" pattern (`content/en/`) — it caused the EN content to leak into the RU RSS feed because the directory was nested inside `content/`.
- Russian is served at `/`, English at `/en/` (via `defaultContentLanguageInSubdir = false`).
- Front matter: `title`, `date` (ISO YYYY-MM-DD), optional `description`, optional `tags`.
- The language switcher uses `.Translations` to find the matching page in the other language and falls back to the language root if no translation exists. See `layouts/partials/header.html`.

## Adding a blog post

1. Create `content/blog/<slug>.md` (Russian, default).
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
4. (Optional) Mirror as `content/blog/<slug>.en.md` for the English version. Same filename stem links them as translations automatically.

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
