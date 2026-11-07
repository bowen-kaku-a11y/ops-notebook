# CLAUDE.md

Source of the public Hugo site notes.aiworkshoplab.com ("The Ops Notebook"). **This repo is
public**: everything committed here is published, history included.

- Posts: `content/posts/*.md`, YAML front matter `title`, `date`, `description`, then an H1
  repeating the title. Match the language of the file you edit (posts are English).
- When any content file (post or page such as privacy/terms) gets a substantive update,
  add `lastmod: YYYY-MM-DD` to its front matter. It feeds the sitemap and, for posts,
  dateModified / article:modified_time;
  without it the publish date is used (git dates are not, the repo history starts at the
  2026-11-06 import).
- **Sanitize everything** before committing: no real hostnames or internal domains (use
  `*.example.com`), no real IPs (use 203.0.113.0/24, 192.0.2.0/24, 198.51.100.0/24), no cloud
  account/resource IDs (truncate like `ocid1.user.oc1..aaaa…wxyz`), no tokens, no private repo
  names or links, no employer names. Don't describe how any employer's real systems are set up.
- Theme: `themes/opsnotes/` (own theme). `hugo.toml` holds site params; Goldmark `unsafe = true`.
- Build: Hugo 0.166.0 extended. `hugo --minify --cleanDestinationDir`.
- Deploy: merge to `main` → `.github/workflows/site-deploy.yml` (environment `cloudflare-pages`,
  secrets `CLOUDFLARE_API_TOKEN` Pages:Edit + `CLOUDFLARE_ACCOUNT_ID`) → Cloudflare Pages project
  `ops-notebook` (Direct Upload). Manual fallback from the repo root:
  `hugo --minify --cleanDestinationDir && npx wrangler@4.86.0 pages deploy public --project-name ops-notebook --branch main`
- Never commit `.wrangler/` (Wrangler's cache holds account metadata) or `public/`.
