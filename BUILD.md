# DietRight — Claude Code Build & Deploy Instructions

## Project overview
Single-file HTML/JS web app for Aaron & Bridie's personalised keto diet and daily routine tracker.
No framework dependencies. No build step. No backend. All data in localStorage.
Deploy target: `jointcollab.com/dietright`

---

## File structure to create

```
dietright/
├── index.html          ← the entire app (already built)
├── BUILD.md            ← this file
├── .gitignore
└── _redirects          ← Cloudflare Pages routing
```

---

## Step 1 — GitHub repository

```bash
# Run from the directory containing index.html and BUILD.md
git init
git add .
git commit -m "Initial DietRight app"

# Create repo on GitHub (requires gh CLI — install if needed: brew install gh)
gh auth login
gh repo create dietright --public --source=. --push
```

If gh CLI is not available, create the repo manually at github.com and then:
```bash
git remote add origin https://github.com/YOUR_GITHUB_USERNAME/dietright.git
git branch -M main
git push -u origin main
```

---

## Step 2 — .gitignore

Create `.gitignore`:
```
.DS_Store
node_modules/
*.log
```

---

## Step 3 — _redirects file (Cloudflare Pages SPA routing)

Create `_redirects` in the project root:
```
/dietright        /index.html    200
/dietright/*      /index.html    200
/*                /index.html    200
```

---

## Step 4 — Cloudflare Pages setup

**Option A: Via Cloudflare dashboard (recommended)**

1. Log in at dash.cloudflare.com
2. Go to **Workers & Pages** → **Pages** → **Create a project**
3. Select **Connect to Git** → authorise GitHub → select the `dietright` repo
4. Build settings:
   - Framework preset: `None`
   - Build command: *(leave blank)*
   - Build output directory: `/` (root)
5. Click **Save and Deploy**
6. Cloudflare will assign a `*.pages.dev` URL — note it down

**Option B: Via Wrangler CLI**

```bash
npm install -g wrangler
wrangler login
wrangler pages project create dietright
wrangler pages deploy . --project-name=dietright
```

---

## Step 5 — Custom domain: jointcollab.com/dietright

### If jointcollab.com is already on Cloudflare:

The cleanest path for a subdirectory (`/dietright`) on an existing domain is a **Cloudflare Worker route**.

**5a. Create a Worker to proxy the Pages deployment:**

In Cloudflare dashboard → **Workers & Pages** → **Create application** → **Create Worker**

Name it `dietright-proxy`. Replace the default script with:

```javascript
export default {
  async fetch(request, env) {
    const url = new URL(request.url);
    // Only handle /dietright paths
    if (url.pathname.startsWith('/dietright')) {
      // Rewrite to Cloudflare Pages URL
      const pagesUrl = 'https://dietright.pages.dev' + 
        (url.pathname.replace('/dietright', '') || '/');
      return fetch(new Request(pagesUrl, request));
    }
    // Pass everything else through
    return fetch(request);
  }
};
```

Replace `dietright.pages.dev` with your actual Pages URL from Step 4.

**5b. Add a route for the Worker:**

In Cloudflare dashboard → **Workers & Pages** → your Worker → **Triggers** → **Add route**:
- Route: `jointcollab.com/dietright*`
- Zone: `jointcollab.com`

**5c. Test:**
```
curl -I https://jointcollab.com/dietright
# Should return 200
```

---

### If jointcollab.com is NOT yet on Cloudflare:

**Add domain to Cloudflare:**

```bash
# Via Wrangler
wrangler pages project create dietright
# Then add custom domain in dashboard:
# Pages project → Custom domains → Add custom domain → jointcollab.com/dietright
```

**DNS setup (point domain to Cloudflare):**

At your domain registrar, update nameservers to Cloudflare's:
- `ns1.cloudflare.com`
- `ns2.cloudflare.com`

Then in Cloudflare DNS:
- Add `A` record: `@` → `192.0.2.1` (Cloudflare proxied, orange cloud)
- Or add `CNAME`: `@` → your Pages URL

---

### If you want a subdomain instead (simpler):

`dietright.jointcollab.com` is easier to set up than a path.

In Cloudflare Pages → your project → **Custom domains** → **Set up a custom domain**:
- Enter: `dietright.jointcollab.com`
- Cloudflare auto-creates the CNAME record

---

## Step 6 — Auto-deploy on push

Once connected to GitHub, every `git push` to `main` auto-deploys via Cloudflare Pages CI/CD.
No additional config needed.

---

## Step 7 — Verify deployment

```bash
# Check the live URL
curl -s https://jointcollab.com/dietright | grep -o '<title>[^<]*</title>'
# Should return: <title>DietRight — Aaron & Bridie</title>
```

---

## Updating the app

All future changes are made to `index.html` only.

```bash
# Make changes to index.html, then:
git add index.html
git commit -m "Update: describe what changed"
git push
# Cloudflare Pages auto-deploys within ~60 seconds
```

---

## Adding Bridie's data / updating Aaron's preferences

These are stored in the user's browser `localStorage` — no server needed.
Each user sets up their profile and food preferences directly in the app on first visit.
Aaron's preferences are pre-loaded from the quiz results in the code.

---

## Tech notes for Claude Code

- **No npm install needed** — zero dependencies, pure HTML/CSS/JS
- **No build step** — deploy the file as-is
- **Data persistence** — browser `localStorage` only, no database
- **Two users** — Aaron (pre-configured) and Bridie (setup wizard on first visit)
- **Mobile-friendly** — responsive, max-width 680px, bottom nav
- **Dark mode** — respects `prefers-color-scheme`
- **Offline-capable** — works without internet after first load (no external API calls)

---

## If Cloudflare is not available

Deploy to any static host:
- **Netlify**: drag and drop the `dietright/` folder at app.netlify.com
- **Vercel**: `npx vercel --prod` from the project directory
- **GitHub Pages**: enable in repo Settings → Pages → deploy from `main`
- **Any web host**: upload `index.html` to `public_html/dietright/`

---

## Troubleshooting

| Issue | Fix |
|-------|-----|
| 404 at `/dietright` | Check Worker route is `jointcollab.com/dietright*` (with asterisk) |
| App loads but data resets | Check browser allows localStorage (not in private/incognito) |
| CSS icons missing | Check CDN link for Tabler Icons is loading (requires internet) |
| Bridie sees Aaron's plan | Each user's data is scoped to their own localStorage key |
| Pages not auto-deploying | Check GitHub repo connection in Cloudflare Pages dashboard |
