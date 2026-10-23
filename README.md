# Secure single-file blog publisher (Cloudflare Worker)

This Worker authenticates the author and updates the published articles marker block inside `index.html` through the GitHub Contents API. The GitHub token and author password remain server-side Cloudflare secrets; neither is stored in the HTML.

## Correct repository coordinates

From the repository URL supplied: `https://github.com/gowribasava98-byte/gowrimallesh/blob/main/index.html`

- Owner: `gowribasava98-byte`
- Repository: `gowrimallesh`
- Branch: `main`
- File path: `index.html`

## 1. Upload the updated HTML first

Copy the generated `index.html` file into the root of the `main` branch of `gowribasava98-byte/gowrimallesh` and commit it. The HTML must include `BEGIN_GITHUB_PUBLISHED_ARTICLES` and `END_GITHUB_PUBLISHED_ARTICLES` markers. Do not configure publishing until this initial upload is done.

## 2. Create a least-privilege GitHub token

In GitHub Settings > Developer settings > Personal access tokens > Fine-grained tokens, create a token limited to **only** the `gowrimallesh` repository. Grant **Contents: Read and write**. Set an expiration. Do not put the token into HTML, Git, or this project folder.

## 3. Configure Cloudflare Worker

Install Node.js, then in this directory run:

```bash
npm install
npx wrangler login
```

Edit `wrangler.toml` and replace `ALLOWED_ORIGIN` with the exact origin of your published website. For GitHub Pages project sites it is typically `https://gowribasava98-byte.github.io` (origin only; no repository path). Once published, confirm this origin in your browser address bar.

Add secrets interactively (values are hidden):

```bash
npx wrangler secret put GITHUB_TOKEN
npx wrangler secret put ADMIN_PASSWORD
npx wrangler secret put SESSION_SIGNING_SECRET
```

Use a long, unique password for `ADMIN_PASSWORD`. For `SESSION_SIGNING_SECRET`, generate a random secret with at least 32 bytes, for example on macOS/Linux:

```bash
openssl rand -hex 32
```

Then deploy:

```bash
npx wrangler deploy
```

Copy the deployed Worker URL, such as `https://gowri-blog-publisher.<your-subdomain>.workers.dev`.

## 4. Connect the HTML to the Worker

In `index.html`, find:

```js
const GITHUB_PUBLISH_ENDPOINT = "https://YOUR-WORKER-NAME.YOUR-SUBDOMAIN.workers.dev";
```

Replace the placeholder URL with the actual deployed Worker URL, then commit `index.html` to GitHub. If the browser is already serving an older copy, hard-refresh it.

## 5. Publish

Open the published website over HTTPS. Use the blog author sign-in or the admin elevation prompt and enter the `ADMIN_PASSWORD` configured in Cloudflare. Then create or edit an article and click **Save & Publish**. The Worker updates the article data block in `index.html`, and GitHub creates a commit. The GitHub Pages build/update can take a short time to become visible.

## 6. Enable GitHub Pages

In the `gowrimallesh` repository, open Settings > Pages and choose deploy from the `main` branch and `/ (root)` if that is how you want to publish. The project site will normally be `https://gowribasava98-byte.github.io/gowrimallesh/` unless you configure a custom domain. Use only the origin (without `/gowrimallesh/`) for `ALLOWED_ORIGIN`.

## Security notes

- The backend rejects requests from origins other than `ALLOWED_ORIGIN` and requires a signed, two-hour author session for publishing.
- GitHub API credentials and the author password must exist only as Cloudflare Worker secrets.
- Configure Cloudflare rate limiting/WAF rules for the Worker route to reduce password guessing. Rotate secrets if exposed.
- Never run a public test with real secrets in committed files.
- This Worker edits only the marker block in `index.html`. The marker block is JSON data; the rest of the portfolio is preserved.
- The browser's `localStorage` remains a local cache. The canonical published articles are the data embedded in `index.html` in GitHub.
- A delete request from the current UI is still local-only unless the UI is wired to call the Worker's `delete` action; use the article edit/publish flow for publishing updates.
