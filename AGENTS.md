# AGENTS.md

Jekyll blog on the `jekyll-theme-chirpy` gem (`~> 4.3`), deployed to `blog.tiansen.me` (custom domain) via GitHub Actions to `gh-pages`. No app code, no test suite — content + config only.

## Commands

- Install: `bundle`
- Local dev: `bundle exec jekyll serve` (incremental: `bundle exec jekyll serve --livereload`)
- Production build (mirrors CI): `JEKYLL_ENV=production bundle exec jekyll build`
- Build + htmlproofer check without deploying: `bash tools/deploy.sh --dry-run`
- Never run `bash tools/deploy.sh` without `--dry-run` locally — the script exits with an error outside GitHub Actions by design, and with deploy intent it force-pushes `_site` to `gh-pages`.

## CI / deploy (`tools/deploy.sh`, `.github/workflows/pages-deploy.yml`)

- Triggers on push to `main` only; runs Ruby 2.7 with `bundler-cache: true`.
- Flow: `JEKYLL_ENV=production bundle exec jekyll build` → `htmlproofer --disable-external --check-html --allow_hash_href _site` → force-push `_site` to `gh-pages`.
- Checkout uses `fetch-depth: 0` — required by `_plugins/posts-lastmod-hook.rb`, which derives `last_modified_at` from `git log`. Don't shallowen or remove it.
- `CNAME` lives only on `gh-pages`; don't add one on `main`.

## Structure

- `_posts/YYYY-MM-DD-slug.md` — published posts. Filename slug defines the URL via `permalink: /posts/:title/`. Never rename a published file without a redirect plan.
- `_drafts/` — unpublished; `comments: false` default.
- `_tabs/` — fixed nav pages (`about/archives/categories/slides/tags.md`). Tab front matter uses `icon` + `order`, `layout: page`, `permalink: /:title/`.
- `_layouts/home.html` — the only theme override; all other layouts/includes come from the installed gem (`bundle info --path jekyll-theme-chirpy`). Override by copying from the gem, never edit the gem.
- `_plugins/posts-lastmod-hook.rb` — sole plugin (lastmod hook above). Keep it.
- `assets/` (images, `ts.jpg` avatar), `slides/` — standalone HTML decks, deliberately excluded from htmlproofer in `_config.yml` (`alt_ignore`/`file_ignore`/`exclude`). Don't "fix" those exclusions.
- `tools/`, `docs/`, `README.md`, `LICENSE`, `*.json`, `*.gem*` are in Jekyll `exclude:` — not part of the built site.
- `_site/`, `vendor/`, `.history/` are build output / local clutter. Never edit or commit `_site/`.

## Conventions

- Post front matter: `title`, `date: YYYY-MM-DD HH:MM:SS +0800` (timezone is `Asia/Shanghai`), `categories: ['x']`, `tags: ['x']`, optional `slug:`. `lang` is `zh-CN`; most posts are Chinese. Match the existing style.
- `baseurl: ''` is intentional (custom domain). Don't set it to a project path.
- `.gitignore` ignores all dotfiles (`.*`) except `.git*`, `.editorconfig`, `.nojekyll` — new dotfiles (e.g. `.cursor/`, opencode config) will be silently ignored unless allowlisted.
- Formatting: 2-space indent, LF, trim trailing whitespace (`.editorconfig`).
