# CLAUDE.md

Guidance for working in this repo.

## What this is

Rohin's personal site: a minimal **Jekyll** blog hosted on **GitHub Pages** at
`https://gargrohin.github.io`. It is a GitHub Pages *user site*, so the live
site deploys from the **`master`** branch. Pushing to `master` publishes.

## Design references

The site's look is modeled on a few minimal personal blogs. Use these when
adjusting layout, typography, or spacing:

- **evjang.com**: primary style inspiration (also a Jekyll + GitHub Pages
  site). Referenced by URL, not mirrored locally.
- **rajan.sh**: inspiration for the home page layout (intro paragraphs plus a
  "Selected work" list). Local mirror downloaded for offline reference.
- **ericyuegu.com**: style reference. Local mirror downloaded for offline
  reference.

The two local mirrors live **outside this repo** at `../downloaded_blogs/`
(i.e. `blogsite/downloaded_blogs/{rajan.sh,ericyuegu.com}`), so they are never
committed or published. The original build plan and agent prompt also sit in
the parent `blogsite/` dir as `PLAN.md` and `AGENT_PROMPT.md` (not in the repo).

## Structure

- `_config.yml` — site config. Holds `title`, `description`, and social link
  vars (`github`, `email`, `linkedin`, `scholar`). Empty social
  vars are hidden in the footer.
- `_layouts/default.html` — base layout (nav, `{{ content }}`, footer, and a
  script that opens external links in a new tab).
- `_layouts/post.html` — blog post layout.
- `_includes/head.html` — `<head>`: meta, CSS, KaTeX for math.
- `_includes/footer.html` — social links, driven by `_config.yml` vars.
- `index.html` — home: intro (bio) plus a hand-written "Selected work" list.
- `writing.html` — `/writing/`, the blog archive.
- `projects.html` — `/projects/`, renders `_data/projects.yml`.
- `_data/projects.yml` — project entries (see the header comment for fields).
- `_posts/` — blog posts, named `YYYY-MM-DD-slug.md`.
- `assets/css/style.css` — all site styling. `syntax.css` is Rouge highlighting.

Post URLs use the permalink `/:year/:month/:day/:title.html`, where `:title`
is the **filename slug**, not the front-matter title. So
`_posts/2026-02-22-reward-hacking.md` -> `/2026/02/22/reward-hacking.html`.

## Conventions

- **No em-dashes** in any prose or content. Use commas, colons, periods, or
  parentheses.
- **External links open in a new tab** automatically (script in
  `default.html`); do not add `target="_blank"` by hand. Internal links stay in
  the same tab.
- Math renders via KaTeX using `$...$` (inline) and `$$...$$` (display).

## Adding content

- **Blog post**: add `_posts/YYYY-MM-DD-slug.md` with front matter
  `layout: post`, `title:`, `date:`. It appears on `/writing/` and the home
  page automatically.
- **Project**: add a block to `_data/projects.yml`.
- **Social link**: set the corresponding var in `_config.yml`.

## Running locally

Needs **Ruby >= 3.x** (system Ruby 2.6 on macOS is too old for Jekyll's
`rouge`). Install Ruby via Homebrew (`brew install ruby`) if needed.

Normal path:

```sh
bundle install
bundle exec jekyll serve --livereload
```

If `bundle` fails because `Gemfile.lock` is stale, bypass Bundler and run
Jekyll directly against locally installed gems:

```sh
export PATH="/opt/homebrew/opt/ruby/bin:$PATH"
gem install jekyll -v 4.3.4 jekyll-seo-tag --no-document
JEKYLL_NO_BUNDLER_REQUIRE=true jekyll serve --livereload
```

Preview at `http://127.0.0.1:4000/`. Editing `_config.yml` requires a server
restart (Jekyll does not hot-reload config).

## Deploying

Commit and push to `master`. GitHub Pages rebuilds and publishes within a
minute or two. There is no separate build step or CI.
