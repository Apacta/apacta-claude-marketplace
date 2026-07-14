---
description: Deploy a project to shub.dk (Cloudflare Workers). Works for first deploys and updates/fixes.
argument-hint: "[project-name or path]"
---

Deploy a Service Hub project to Cloudflare Workers on shub.dk.

## When to use

- First deploy of a new project to demo/test
- Deploying a fix or update to an existing project
- The user says "deploy", "put it live", "push to demo", or similar

## Domain structure

All demo/test products live on `{product}.shub.dk`:

| Product | URL | GitHub repo |
|---------|-----|-------------|
| Trading HUB | tradinghub.shub.dk | akrisager/trading-hub |
| APV | apv.shub.dk | akrisager/APV |
| KLS | kls.shub.dk | akrisager/KLS_ISO9001 |
| Intimacy Intelligence | intimiq.shub.dk | (in Apacta org) |
| Demo Portal | demo.shub.dk | akrisager/demo-portal |
| PDW | pdw.shub.dk | (in Apacta org) |

## Infrastructure

- **Platform:** Cloudflare Workers (serverless edge)
- **Database:** Cloudflare D1 (SQLite at the edge)
- **File storage:** KV Namespace (or R2 if enabled)
- **DNS:** Cloudflare manages shub.dk — custom_domain in wrangler.toml auto-provisions DNS + SSL
- **Auth sharing:** All products on shub.dk share the `nuansa_demo` cookie (password: stored as DEMO_PASSWORD secret)
- **Secrets:** Managed via `wrangler secret put <KEY>` — never in code

## Deploy pipeline: Local → GitHub → Cloudflare

**IMPORTANT:** We ALWAYS deploy through GitHub, never directly from local.

```
Local code → git push → GitHub Actions → wrangler deploy → live on shub.dk
```

This gives us:
- Version history and rollback via git
- Code review before deploy (PRs)
- Automated deploys on push to main

### GitHub Actions workflow

Every project needs `.github/workflows/deploy.yml`:

```yaml
name: Deploy to Cloudflare Workers
on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '22'
      - run: npm ci
      - run: npx @tailwindcss/cli -i src/app.css -o dist/css/app.css --minify
        if: hashFiles('src/app.css') != ''
      - run: npx wrangler deploy
        env:
          CLOUDFLARE_API_TOKEN: ${{ secrets.CLOUDFLARE_API_TOKEN }}
```

The repo needs a GitHub secret `CLOUDFLARE_API_TOKEN` — create one at:
https://dash.cloudflare.com/profile/api-tokens → Create Token → "Edit Cloudflare Workers" template

## First-time deploy checklist

### 1. Ensure code is on GitHub

If the project only exists locally:
```bash
cd <project-dir>
git init  # if needed
git add -A
git commit -m "Initial commit"
gh repo create <name> --private --source=. --push  # ALWAYS --private — never --public
```

If the project exists on GitHub but local code is newer:
```bash
cd <local-project-dir>
git remote add origin https://github.com/akrisager/<repo>.git
git push -u origin main --force  # ONLY for initial setup
```

### 2. Ensure project is CF Workers compatible

If the project uses Node.js + better-sqlite3, it MUST be migrated first:
- Drop `@hono/node-server` — use `export default app`
- Drop `better-sqlite3` / Drizzle ORM — use D1 raw SQL
- Drop local file writes — use KV via `c.env.UPLOADS`
- All env via `c.env.*` bindings, not `process.env`
- Hono type: `new Hono<{ Bindings: Env }>()`

See the Migration section below for details.

### 3. Create cloud resources

```bash
cd <project-dir>
npx wrangler d1 create <project>-db        # If needs database
npx wrangler kv namespace create UPLOADS    # If needs file storage
```
Copy the returned IDs into wrangler.toml.

### 4. Create wrangler.toml

```toml
name = "<project-name>"
main = "src/index.ts"
compatibility_date = "2024-12-18"
compatibility_flags = ["nodejs_compat"]

routes = [
  { pattern = "<project>.shub.dk", custom_domain = true }
]

[[d1_databases]]
binding = "DB"
database_name = "<project>-db"
database_id = "<from wrangler d1 create>"
migrations_dir = "migrations"

[[kv_namespaces]]
binding = "UPLOADS"
id = "<from wrangler kv namespace create>"

[assets]
directory = "./dist"

[vars]
ENVIRONMENT = "production"
```

### 5. Create migrations and apply

```bash
mkdir -p migrations
# Create migrations/0001_init.sql with full schema
npx wrangler d1 migrations apply <db-name> --remote
```

### 6. Set secrets

```bash
npx wrangler secret put ANTHROPIC_API_KEY   # If uses Claude API
npx wrangler secret put DEMO_PASSWORD       # For demo auto-login (value: demo2026)
```

### 7. Set up GitHub Actions

Create `.github/workflows/deploy.yml` (see template above).

Set the `CLOUDFLARE_API_TOKEN` secret on the GitHub repo:
```bash
gh secret set CLOUDFLARE_API_TOKEN  # Paste the token when prompted
```

### 8. Build assets and push

```bash
npx @tailwindcss/cli -i src/app.css -o dist/css/app.css --minify  # If applicable
git add -A
git commit -m "Ready for deploy"
git push origin main  # Triggers GitHub Actions → auto-deploy
```

### 9. Seed database (if needed)

```bash
npx wrangler d1 execute <db-name> --remote --file=migrations/0002_seed.sql
```

### 10. Update demo-portal

Add the product to C:/Users/akr/code/demo-portal/src/index.ts:
- Product card with role links
- Auto-login redirect route (`/goto/<product>/<role>`)
- Push and deploy demo-portal too

### 11. DNS

If using a NEW subdomain, wrangler's `custom_domain = true` auto-provisions it.
If it doesn't resolve within 5 minutes, manually add in Cloudflare dashboard:
AAAA record, name: subdomain, content: `100::`, proxied: on.

## Update/fix deploy

1. Make changes locally
2. Build CSS if templates changed
3. Commit and push:
   ```bash
   git add -A
   git commit -m "Fix: description of fix"
   git push origin main
   ```
4. GitHub Actions auto-deploys
5. Run migrations if schema changed:
   ```bash
   npx wrangler d1 migrations apply <db-name> --remote
   ```

## Migration from Node.js to CF Workers

If a project currently uses Node.js + better-sqlite3 + Drizzle ORM:

1. **Drop Drizzle** — use raw SQL with D1's `.prepare().bind().first()/.all()/.run()`
2. **All DB calls become async** — every `.get()` → `.first()`, every `.all()` → `(await ...all()).results`
3. **Replace file uploads** — disk writes → KV `.put()` with metadata
4. **Replace `@hono/node-server`** — `export default app` instead of `serve()`
5. **Pass env bindings** — `c.env.DB`, `c.env.UPLOADS`, `c.env.ANTHROPIC_API_KEY`
6. **Hono type** — `new Hono<{ Bindings: Env }>()` with proper Env type
7. **Create `src/types.ts`** with Env type and all row types

## Rollback

```bash
npx wrangler rollback    # Rolls back to previous version
```

Or revert the git commit and push again.

## Project locations

All projects live in `C:/Users/akr/code/`:
- `trading-hub/` — Trading HUB (deployed)
- `APV ver 1.3/` — APV latest local version (needs push to GitHub + migration)
- `kls-platform/` — KLS git repo
- `kls-platform-versions/v3.3.1_final/` — KLS latest local version
- `intimacy-intelligence/` — Nuansa (deployed)
- `demo-portal/` — Demo portal (deployed)
- `product-discovery-wizard/` — PDW (deployed)

## Cloudflare account

- **Zone:** shub.dk (zone_id: bc10b996f222cd0b029391823b6aa5fb)
- **Auth:** wrangler uses OAuth (stored in AppData/Roaming/xdg.config/.wrangler/)
- **Node.js:** Installed via mise at C:/Users/akr/AppData/Local/mise/installs/node/22.22.1
- **PATH setup:** `export PATH="/c/Users/akr/AppData/Local/mise/installs/node/22.22.1:$PATH"`
