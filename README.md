# Signpost Audio — SEO + LLM Index Files

This repo hosts machine-readable index files for [signpostaudio.com](https://www.signpostaudio.com):

- [`sitemap.xml`](sitemap.xml) — XML sitemap for Google and Bing
- [`llms.txt`](llms.txt) — concise AI-readable site index ([spec](https://llmstxt.org))
- [`llms-full.txt`](llms-full.txt) — full markdown mirror of site content
- [`robots.txt`](robots.txt) — crawler directives + index file references
- [`index.html`](index.html) — landing page for the subdomain root

**Main site:** runs on Wix at [signpostaudio.com](https://www.signpostaudio.com)
**This repo:** GitHub Pages, deployed at [seo.signpostaudio.com](https://seo.signpostaudio.com)

---

## Setup (one-time, ~10 minutes)

### Step 1 — Create the GitHub repo

```bash
# From this directory (seo-content/github-pages-deploy/):
git init
git branch -M main
git add .
git commit -m "Initial SEO + LLM index files"

# Create the repo on GitHub (via web or gh CLI):
gh repo create signpost-audio-seo-files --public --source=. --remote=origin --push
```

If you don't have the `gh` CLI: create an empty repo at github.com manually, then:

```bash
git remote add origin git@github.com:isthisbray/signpost-audio-seo-files.git
git push -u origin main
```

### Step 2 — Enable GitHub Pages

1. Go to the repo on GitHub
2. **Settings** → **Pages**
3. **Source:** Deploy from a branch
4. **Branch:** `main` → `/ (root)`
5. **Save**

Within ~60 seconds, the site is live at `https://isthisbray.github.io/signpost-audio-seo-files/`.

### Step 3 — Point a subdomain at it (DNS)

The `CNAME` file in this repo is already set to `seo.signpostaudio.com`. To activate it:

#### If your DNS lives at your domain registrar (GoDaddy, Namecheap, etc.):

Add a CNAME record:

| Type | Name | Value | TTL |
|---|---|---|---|
| CNAME | `seo` | `isthisbray.github.io` | Auto (or 300) |

#### If your DNS lives at Wix:

1. Wix Dashboard → **Domains**
2. Manage DNS Records → Add Record
3. Type: `CNAME`, Host: `seo`, Value: `isthisbray.github.io`, TTL: 1 hour

#### If your DNS lives at Cloudflare (recommended — see Step 5):

| Type | Name | Target | Proxy status |
|---|---|---|---|
| CNAME | `seo` | `isthisbray.github.io` | DNS-only (gray cloud) |

Wait 5–60 minutes for DNS propagation. Verify with:

```bash
dig seo.signpostaudio.com
```

### Step 4 — Verify

In a fresh browser window, visit:

- `https://seo.signpostaudio.com/` (lands on the index page)
- `https://seo.signpostaudio.com/sitemap.xml` (XML loads)
- `https://seo.signpostaudio.com/llms.txt` (markdown loads)
- `https://seo.signpostaudio.com/llms-full.txt` (large markdown loads)
- `https://seo.signpostaudio.com/robots.txt` (text loads)

### Step 5 — Reference from the main site (Wix)

In Wix, the main site at `signpostaudio.com` should point search engines and AI crawlers to your subdomain:

#### Update Wix's robots.txt

1. Wix Dashboard → **Marketing & SEO** → **SEO Tools**
2. **Robots.txt Editor** → Edit
3. Add/replace contents with:

```
User-agent: *
Allow: /

# Traditional XML sitemap (use the hand-curated one from this repo)
Sitemap: https://seo.signpostaudio.com/sitemap.xml

# Wix's auto-generated sitemap (keep as fallback)
Sitemap: https://www.signpostaudio.com/sitemap.xml

# AI / LLM index files
# Primary index (concise): https://seo.signpostaudio.com/llms.txt
# Full markdown mirror: https://seo.signpostaudio.com/llms-full.txt
```

#### Submit to Google Search Console

1. Search Console → **Sitemaps** (left sidebar)
2. Add: `https://seo.signpostaudio.com/sitemap.xml`
3. Submit

---

## Better setup: serving files at the root domain (Cloudflare in front of Wix)

The above gets you `seo.signpostaudio.com/llms.txt`, which works but isn't optimal — AI engines and search bots traditionally look at the **root domain** for these files (e.g., `signpostaudio.com/llms.txt`).

To get files at the root path while keeping Wix as your main site host:

### Cloudflare setup (~30 minutes, one-time)

1. **Sign up for Cloudflare** (free plan is fine): cloudflare.com
2. **Add your domain** `signpostaudio.com`
3. **Point your nameservers to Cloudflare** (replace whatever Wix or your registrar uses)
   - Wait 24–48 hours for full propagation (the site stays up via Wix the whole time)
4. **Set up Cloudflare Workers** (free tier covers this) — a Worker script that:
   - Catches requests to `signpostaudio.com/llms.txt`, `/llms-full.txt`, `/sitemap.xml`, `/robots.txt`
   - Fetches the file from this GitHub Pages repo
   - Returns it with correct headers
   - Lets all other requests pass through to Wix normally

#### Cloudflare Worker script (paste into Workers dashboard):

```js
const ROUTES = {
  '/llms.txt':       'https://seo.signpostaudio.com/llms.txt',
  '/llms-full.txt':  'https://seo.signpostaudio.com/llms-full.txt',
  '/sitemap.xml':    'https://seo.signpostaudio.com/sitemap.xml',
  '/robots.txt':     'https://seo.signpostaudio.com/robots.txt',
};

export default {
  async fetch(request) {
    const url = new URL(request.url);
    const target = ROUTES[url.pathname];

    if (target) {
      const upstream = await fetch(target, { cf: { cacheTtl: 3600 } });
      const headers = new Headers(upstream.headers);
      // Force correct content types
      if (url.pathname.endsWith('.xml')) headers.set('content-type', 'application/xml');
      else if (url.pathname.endsWith('.txt')) headers.set('content-type', 'text/plain; charset=utf-8');
      return new Response(upstream.body, { status: upstream.status, headers });
    }

    return fetch(request);
  }
};
```

5. **Bind the Worker** to a route: `signpostaudio.com/*` (or specific paths)
6. **Test:** `curl https://signpostaudio.com/llms.txt` should now return the file content

Now AI engines and crawlers find your files at the canonical root URL — and you didn't have to migrate from Wix.

---

## Updating the files

### When you publish a new blog post

1. Edit `sitemap.xml` — add the new URL with a recent `<lastmod>`
2. Edit `llms.txt` — add a link under the relevant section
3. Edit `llms-full.txt` — add a Quick Answer + summary for the new post
4. Commit and push:

```bash
git add .
git commit -m "Add new post: [post title]"
git push
```

GitHub Pages re-deploys automatically within ~60 seconds.

### Automated sitemap updates (optional)

If you want a GitHub Action that auto-updates `<lastmod>` dates daily, see `.github/workflows/touch-sitemap.yml` (not included by default — add if useful).

---

## Why this approach works

| Concern | Wix-only | GitHub + subdomain | GitHub + Cloudflare (root) |
|---|---|---|---|
| Files at root URL | ❌ ugly paths | ❌ subdomain | ✅ canonical |
| Version control | ❌ | ✅ | ✅ |
| Easy to update | ⚠️ awkward | ✅ git push | ✅ git push |
| Setup time | 5 min | 10 min | 30 min |
| Recommended | — | starter | optimal |

The GitHub-based setup gives you:
- **Version control** for your SEO files (rollback bad changes, see history)
- **Free hosting** (GitHub Pages is free for public repos)
- **Easy updates** (edit + git push)
- **Optional automation** (GitHub Actions for sitemap regeneration)

---

## Verifying AI engines can read llms.txt

After deployment, test with these prompts in ChatGPT, Claude, or Perplexity:

> "Fetch https://www.signpostaudio.com/llms.txt and tell me what Signpost Audio offers."

> "Visit https://seo.signpostaudio.com/llms.txt — what services does this business provide?"

The AI should be able to summarise your services and offerings. If it says it can't read the file, double-check:

- File is served as `text/plain` (some hosts serve as `application/octet-stream` which breaks AI parsing)
- File is reachable via curl without authentication
- robots.txt isn't blocking the path

---

*Setup created 2026-05-08. Files updated as content changes.*
