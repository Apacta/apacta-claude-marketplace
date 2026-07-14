---
description: Show deployment status of all Service Hub products on shub.dk
---

Check the deployment status of all Service Hub products.

## When to use

- The user asks "what's deployed?", "status", "what's live?", or similar
- Before deploying, to see current state
- To verify a deploy succeeded

## What to check

For each known product, check:

1. **Is it deployed?** — `curl -s -o /dev/null -w "%{http_code}" https://<product>.shub.dk/`
2. **What version?** — `npx wrangler deployments list` in the project directory
3. **Is the database seeded?** — Quick query via `wrangler d1 execute`

## Known products

| Product | Subdomain | Project dir | Status check |
|---------|-----------|-------------|--------------|
| Demo Portal | demo.shub.dk | C:/Users/akr/code/demo-portal | `curl https://demo.shub.dk/` |
| Trading HUB | tradinghub.shub.dk | C:/Users/akr/code/trading-hub | `curl https://tradinghub.shub.dk/` |
| APV | apv.shub.dk | C:/Users/akr/code/apv | Not yet deployed |
| KLS | kls.shub.dk | C:/Users/akr/code/kls-platform | Not yet deployed |
| Intimacy Intelligence | shub.dk/intimiq | C:/Users/akr/code/intimacy-intelligence | `curl https://shub.dk/intimiq/` |
| PDW | pdw.shub.dk | C:/Users/akr/code/product-discovery-wizard | `curl https://pdw.shub.dk/` |

## Output format

Present results as a clean table:

```
Product              URL                          Status    Last deploy
─────────────────────────────────────────────────────────────────────
Demo Portal          demo.shub.dk                 ✅ Live    2026-04-12
Trading HUB          tradinghub.shub.dk           ✅ Live    2026-04-12
APV                  apv.shub.dk                  ⬜ Not deployed
KLS                  kls.shub.dk                  ⬜ Not deployed
Intimacy Intelligence shub.dk/intimiq             ✅ Live    2026-03-xx
PDW                  pdw.shub.dk                  ✅ Live    2026-03-xx
```

## How to run checks

1. Set PATH: `export PATH="/c/Users/akr/AppData/Local/mise/installs/node/22.22.1:$PATH"`
2. For each product, run `curl -s -o /dev/null -w "%{http_code}" <url>` — 200 = live
3. For deployment history, run `npx wrangler deployments list` in each project dir
4. Present results to user
