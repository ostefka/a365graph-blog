# a365graph notes

Bilingual (EN / CS) blog about the **Microsoft 365 Copilot Agent &amp; app Package Management API**
(`/beta/copilot/admin/catalog/packages`) and the governance picture it gives you of a real tenant.

- **Site:** published with GitHub Pages from this repo's `main` branch (root).
- **Live demo dashboard:** [a365graph.ai-news.cz](https://a365graph.ai-news.cz/)
- **Theme:** stock Jekyll `minima` (no Gemfile needed — built by GitHub Pages).

## Contents

- `index.md` — landing page listing posts.
- `about.md` — about page.
- `_posts/`
  - `2026-05-24-copilot-package-management-api.md` — EN long-read.
  - `2026-05-24-copilot-package-management-api-cs.md` — CS long-read.
- `assets/images/` — screenshots embedded in the post.
- `_config.yml` — Jekyll config (`minima` + `jekyll-feed` + `jekyll-seo-tag`).

## Local preview

```bash
bundle init
bundle add jekyll github-pages
bundle exec jekyll serve
# → http://127.0.0.1:4000
```

## License

Content under [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/).
