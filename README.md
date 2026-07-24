# linxiaocong.github.io

Personal blog built with [Jekyll](https://jekyllrb.com/) and the
[Chirpy](https://github.com/cotes2020/jekyll-theme-chirpy) theme, hosted on
GitHub Pages.

## Publishing a new post (no code required)

1. Add a Markdown file to the `_posts/` folder named
   `YYYY-MM-DD-your-title.md`.
2. Start it with front matter:

   ```yaml
   ---
   title: My Post Title
   date: 2026-08-01 09:00:00 +0800
   categories: [Topic]
   tags: [tag1, tag2]
   ---
   ```
3. Write the body in Markdown, commit, and push. GitHub Actions rebuilds and
   deploys automatically.

See the post **"How to Write a New Post"** on the live site for the full guide.

## One-time setup on GitHub

In the repository's **Settings → Pages**, set **Source** to
**"GitHub Actions"**. The included workflow (`.github/workflows/pages-deploy.yml`)
handles building and deploying on every push to the default branch.

## Personalizing

- Site title, description, social links, timezone: edit `_config.yml`.
- About page: edit `_tabs/about.md`.
- Avatar: add an image and set `avatar:` in `_config.yml`.

## Running locally (optional)

```bash
bundle install
bundle exec jekyll serve
```

Then open <http://127.0.0.1:4000>.
