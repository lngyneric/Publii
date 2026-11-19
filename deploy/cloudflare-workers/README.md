# Deploy a Publii site to Cloudflare Workers (free plan)

> **Scope — read this first.** This deploys the *static website Publii
> generates*, not Publii itself. Publii is an Electron desktop app
> (`app/main.js`, Electron 43 / Node 22); Workers has no Electron, no
> persistent filesystem and no GUI, so the app cannot run there. What
> *can* run there is the output: Publii renders a site to plain
> `index.html` + CSS/JS/images, and Workers Static Assets serves exactly
> that kind of folder.
>
> Publii has no built-in Cloudflare deploy target. Its deploy modules
> (`app/back-end/modules/deploy/`) are FTP/SFTP, S3, GitHub Pages,
> GitLab Pages, Netlify and Google Cloud only — no Cloudflare, no
> Wrangler. The two routes below are workarounds, not a new deploy
> module.

## Free-plan limits (checked 2026-11-19, Cloudflare Workers docs)

- **Static asset requests: free and unlimited** — they do not count
  toward the Worker request quota.
- **Worker (dynamic) requests: 100,000/day** on the free plan, reset at
  midnight UTC; over the limit Cloudflare returns Error 1027. A pure
  static Publii site makes no dynamic requests, so this normally never
  bites.
- **20,000 files per Worker version, 25 MiB per file** on the free plan.
  A typical Publii site is far below both; a very large media library
  could hit the file count — check before deploying.
- **No credit card** is needed for the free tier.

## Route A — manual deploy with Wrangler (quickest)

1. In Publii, render your site. The full render writes to
   `<site>/output/` (see `app/back-end/modules/render-html/renderer.js`:
   `this.outputDir = path.join(basePath, 'output')`).
2. Make a deploy folder, e.g. `my-site/`, and copy the files there:
   ```sh
   mkdir -p my-site/public
   cp -r <site>/output/* my-site/public/
   cp deploy/cloudflare-workers/wrangler.toml my-site/wrangler.toml
   ```
3. Edit `my-site/wrangler.toml`: change `name = "my-publii-site"` to
   your site name.
4. Deploy:
   ```sh
   cd my-site
   npx wrangler login        # once, opens the browser
   npx wrangler deploy
   ```
   Wrangler prints a `https://<name>.<subdomain>.workers.dev` URL.
   Re-render in Publii and repeat step 2 + `npx wrangler deploy` to
   update the site. A custom domain is added in the Cloudflare
   dashboard (Workers & Pages → your Worker → Settings → Domains).

## Route B — Git-connected deploy (automatic on push)

This reuses Publii's existing GitHub sync, which the app already
documents as useful for external hosts such as Cloudflare Pages.

1. In Publii, set the site's deployment to **GitHub Pages** and sync
   the generated site to a dedicated site repository (not this Publii
   source repo).
2. In that site repo, add this `wrangler.toml` at the root and put the
   generated files under `public/` (same layout as Route A).
3. In the Cloudflare dashboard: Workers & Pages → Create → connect
   that repository (Workers Builds). No build command is needed — the
   files are already static. Every push then deploys automatically.

## Notes

- **404 handling:** `not_found_handling = "404-page"` in the template.
  Do not switch it to `"single-page-application"` — Publii sites are
  multi-page, and the SPA setting would serve `index.html` with a 200
  for every broken link.
- **No Worker code:** the template has no `main` field, so no Worker
  runs. If you later add API routes, add a Worker entry point and an
  `ASSETS` binding; without the binding, `env.ASSETS` is undefined
  (static files are still served directly by the edge).
- **Cache:** Cloudflare caches static assets at the edge; a new deploy
  creates a new version with a new asset manifest, so visitors pick up
  the new files without manual cache purging.
