# usesextant.com — static site

Zero build step, zero dependencies, zero trackers. Fonts are self-hosted (OFL).

## Before you ship
1. **Newsletter:** the signup is an interim mailto link (hello@protechtus.com). Restore the Buttondown form once the account exists; the original form markup is in git history (commit 487256d).
2. The proof strip (tests, OS count, tool count, Python floor) is generated: the values sit in
   `data-fact="..."` spans that `C:\Tools\protechtus\facts\sync.py` refreshes from the Sextant
   repo and CI, commits and pushes here hourly when they move (task `Facts-Sync`). Never retype a
   value inside a marker; a re-export that drops the spans fails the sync loudly.

## Deploy — GitHub Pages (LIVE since 2026-11-09 at https://usesextant.com)
1. Create repo `protechtus/usesextant.com` (or a `gh-pages` branch) and push the contents of this folder to its root.
2. Repo → Settings → Pages → Source: *Deploy from a branch*, `main` / root.
3. `CNAME` already contains `usesextant.com`. At your DNS provider add:
   - `A` @ → 185.199.108.153, 185.199.109.153, 185.199.110.153, 185.199.111.153
   - `CNAME` www → protechtus.github.io
4. Back in Settings → Pages, tick **Enforce HTTPS** once the cert issues.

`404.html` is picked up automatically. `_headers` is ignored by GitHub Pages (harmless).

## Alternative — Cloudflare Pages (also free)
Connect the repo, build command: *none*, output dir: `/`. `_headers` adds long-cache + security headers. Delete `CNAME` and `.nojekyll` if you like (ignored).

## Any other server
It's static files — copy the folder to any web root (nginx/Caddy). Serve `404.html` as the not-found page.

## Also in the kit
- `brand/social/github-social-preview-1280x640.png` → upload in **protechtus/sextant → Settings → Social preview**.
- README banner snippet (dark/light):
```html
<picture>
  <source media="(prefers-color-scheme: dark)" srcset="https://usesextant.com/brand/social/readme-banner-dark-2560x640.png">
  <img alt="Sextant" src="https://usesextant.com/brand/social/readme-banner-light-2560x640.png">
</picture>
```
