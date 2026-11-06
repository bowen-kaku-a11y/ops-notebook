# The Ops Notebook

Source for [notes.aiworkshoplab.com](https://notes.aiworkshoplab.com/): field notes on running
Kubernetes, streaming data platforms (Redpanda), GitOps (Argo CD), Teleport and Terraform in a
home lab and small cloud setups, with the trade-offs and mistakes written down.

## Layout

| Path | What |
|---|---|
| `content/posts/` | articles (Markdown, YAML front matter) |
| `content/*.md` | about, contact, disclaimer, privacy, terms |
| `themes/opsnotes/` | the site's own Hugo theme |
| `static/` | files served as-is (`robots.txt`, `ads.txt`) |
| `hugo.toml` | site config |

## Build locally

Requires [Hugo](https://gohugo.io/) **0.166.0** extended (the version the site is built with).

```bash
hugo server              # preview at http://localhost:1313
hugo --minify            # build into public/
```

## Publishing

Merging to `main` publishes automatically: `.github/workflows/site-deploy.yml` builds with
Hugo, checks that every publishable post is in the sitemap, and uploads `public/` to the
Cloudflare Pages project with Wrangler. Posts dated in the future are skipped and published by
a daily scheduled run once their date arrives.

## Writing posts

Front matter needs `title`, `date` and `description`, followed by an H1 that repeats the title.
Posts are sanitized: real hostnames, IPs and account identifiers become placeholders such as
`ssh.lab.example.com`, documentation IP ranges (`203.0.113.0/24`, `192.0.2.0/24`), or
truncated IDs.

## License

Copyright © the author. All rights reserved unless stated otherwise.
