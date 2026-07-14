---
description: Create a new project with name and description and a somewhat fixed tech stack based on user's requirements
---

Setup a new project that follows a somewhat fixed tech stack, depending on user requirements for it.

## Tech Stack

- **Backend:** TypeScript on Cloudflare Workers with Hono framework
- **Database:** Cloudflare D1 (SQLite at the edge) — use raw SQL with `.prepare().bind().first()/.all()/.run()`, NOT Drizzle ORM
- **Frontend:** As simple as possible. HTMX + Alpine.js for SSR apps. React only if absolutely necessary (SPA required).
- **Styling:** Tailwind CSS v4 with design system tokens
- **File storage:** Cloudflare KV Namespace (or R2 if enabled on account)
- **Auth:** Magic link for real users + `/auth/demo` shared-secret route for auto-login from demo.shub.dk
- If mobile apps are part of the requirements use React Native with Expo

## Project Setup

- Create a new git repo at `C:/Users/akr/code/$projectName` (NOT in any subfolder, NOT alongside other version snapshots)
- Create README.md, AGENTS.md, CONTRIBUTING.md and document the project
- Create `wrangler.toml` with D1 binding, KV namespace, and custom_domain for `{project}.shub.dk`
- Create `migrations/0001_init.sql` with full database schema
- Create `migrations/0002_seed.sql` (or product-specific name) with initial content
- Create `migrations/0003_demo_users.sql` seeding `super@demo.shub.dk` (admin role) and `bruger@demo.shub.dk` (user role) — see Demo Auth section below
- Create `src/types.ts` with Env type: `{ DB: D1Database; UPLOADS: KVNamespace; ANTHROPIC_API_KEY: string; DEMO_PASSWORD: string; }`
- Create `src/lib/crypto.ts` with `timingSafeEqual` helper (used by /auth/demo)
- Create `src/routes/demo.ts` with /auth/demo route — see Demo Auth section
- Export Hono app as `export default app` (NOT `serve()` — Workers, not Node.js)
- Build Tailwind CSS to `dist/css/app.css`
- Add `[assets] directory = "./dist"` in wrangler.toml

## Versioning — git tags, not folders

Do NOT create version-snapshot folders (`v1.0/`, `v2.0/`, `v2.0-final/`...). Use git tags:

- On first commit, tag `v0.1.0`: `git tag -a v0.1.0 -m "Initial scaffold"`
- For each subsequent milestone: `git tag -a vX.Y.Z -m "..."` and `git push origin vX.Y.Z`
- To roll back temporarily: `git checkout v3.2` → inspect → `git checkout main`
- GitHub's Tags page gives the same visual reassurance as version folders, with date and message

Why: version folders bloat disk (each ~hundreds of MB of node_modules), make diffs hard, and duplicate what git already does. Tags give the same visual rollback story without any of that.

## Deploy-Ready from Day 1

Every new project MUST be deployable to Cloudflare Workers from the first commit. This means:
- No `@hono/node-server` — use `export default app`
- No `better-sqlite3` — use D1 via `c.env.DB`
- No local file writes — use KV via `c.env.UPLOADS`
- All environment values via `c.env.*` (Workers bindings), not `process.env`

After building, the user should be able to run `/product-deploy` to push it live.

## Design System

- **IMPORTANT**: Copy the `DESIGN_SYSTEM.md` file from this skill's directory into the new project root.
- All UI/UX decisions MUST follow DESIGN_SYSTEM.md — colors, typography, spacing, components, layout rules.
- Map the design tokens to the project's styling system (Tailwind config, CSS variables, etc.).
- The design system is the single source of truth for visual decisions across all projects.
- Never hardcode hex values — always use the named tokens from the design system.

## Hono App Pattern

```typescript
import { Hono } from "hono";

type Env = {
  DB: D1Database;
  UPLOADS: KVNamespace;
  ANTHROPIC_API_KEY: string;
  DEMO_PASSWORD: string;
};

const app = new Hono<{ Bindings: Env }>();

// Serve uploaded files from KV
app.get("/uploads/*", async (c) => {
  const key = c.req.path.replace("/uploads/", "");
  const file = await c.env.UPLOADS.getWithMetadata<{ mimeType?: string }>(key, "arrayBuffer");
  if (!file.value) return c.text("Not found", 404);
  return new Response(file.value as ArrayBuffer, {
    headers: { "Content-Type": file.metadata?.mimeType || "application/octet-stream" },
  });
});

// Routes
app.route("/auth", authRoutes);
app.route("/auth", demoRoutes);  // /auth/demo for auto-login from demo.shub.dk
app.route("/admin", adminRoutes);
app.route("/", publicRoutes);

export default app;
```

## D1 Query Pattern

```typescript
// Single row
const user = await c.env.DB.prepare('SELECT * FROM users WHERE id = ?').bind(id).first<UserRow>();

// Multiple rows
const products = (await c.env.DB.prepare('SELECT * FROM products WHERE producer_id = ?').bind(producerId).all<ProductRow>()).results;

// Insert
await c.env.DB.prepare('INSERT INTO users (id, email, name) VALUES (?, ?, ?)').bind(id, email, name).run();

// Update
await c.env.DB.prepare('UPDATE users SET name = ? WHERE id = ?').bind(name, id).run();
```

## Demo Auth — `/auth/demo` (REQUIRED for every product)

Every new product on shub.dk MUST expose `/auth/demo` so the demo-portal at demo.shub.dk can auto-login authenticated testers without each tester needing to know per-product credentials.

**Contract:** `GET /auth/demo?role=admin|user&secret=<DEMO_PASSWORD>`

- Validates `secret` against the shared `DEMO_PASSWORD` env var using `timingSafeEqual`
- Looks up the seeded demo user matching the role:
  - `role=admin` → `super@demo.shub.dk`
  - `role=user` (default) → `bruger@demo.shub.dk`
- Creates a session and redirects into the app
- Returns 401 on bad secret, 404 if demo user not seeded

**Why this exists:** demo-portal authenticates the tester (its own magic-link login + per-tester product-access checks), then forwards them to `/goto/<product>/<role>` which 302s to `<product>.shub.dk/auth/demo?role=<role>&secret=<DEMO_PASSWORD>`. The shared secret only proves "this request came from demo-portal" — testers never see it. This pattern coexists with the product's real auth (magic-link, etc.) which is unaffected.

`src/lib/crypto.ts`:

```typescript
export function timingSafeEqual(a: string, b: string): boolean {
  if (a.length !== b.length) return false;
  let diff = 0;
  for (let i = 0; i < a.length; i++) {
    diff |= a.charCodeAt(i) ^ b.charCodeAt(i);
  }
  return diff === 0;
}
```

`src/routes/demo.ts`:

```typescript
import { Hono } from "hono";
import type { Env, Vars, UserRow } from "../types.js";
import { createSession } from "../lib/auth.js";
import { timingSafeEqual } from "../lib/crypto.js";

const app = new Hono<{ Bindings: Env; Variables: Vars }>();

app.get("/demo", async (c) => {
  const role = (c.req.query("role") || "user").toLowerCase();
  const secret = c.req.query("secret") || "";
  if (!c.env.DEMO_PASSWORD || !timingSafeEqual(secret, c.env.DEMO_PASSWORD)) {
    return c.text("Unauthorized", 401);
  }
  const email = role === "admin" ? "super@demo.shub.dk" : "bruger@demo.shub.dk";
  const user = await c.env.DB.prepare("SELECT * FROM users WHERE email = ?")
    .bind(email).first<UserRow>();
  if (!user) return c.text(`Demo user ${email} not seeded`, 404);
  await createSession(c, user.id);
  return c.redirect(user.role === "admin" ? "/admin" : "/app");
});

export default app;
```

`migrations/0003_demo_users.sql`:

```sql
INSERT INTO users (email, name, role) VALUES
  ('super@demo.shub.dk',  'Demo Admin',  'admin'),
  ('bruger@demo.shub.dk', 'Demo Bruger', 'user');
-- Add product-specific required fields (cvr, company_name, etc.) as needed.
```

After deploy, set the secret: `npx wrangler secret put DEMO_PASSWORD` (value: `demo2026` for current test environment).

## Demo-portal entry — add product card

After scaffolding, add the new product to `C:/Users/akr/code/demo-portal/src/index.ts`:

1. Append a `DemoProduct` entry to the `products` array with:
   - `name`, `description`, `status: "testing"`, `icon`
   - `roles`: at minimum `Admin` (`/goto/<project>/admin`) and `Bruger` (`/goto/<project>/user`)
2. (No `/goto/<project>/...` route handler needed — demo-portal uses a generic handler that maps `/goto/:product/:role` → `https://<product>.shub.dk/auth/demo?role=<role>&secret=<DEMO_PASSWORD>`)
3. Tell the user to redeploy demo-portal with `npm run deploy` from `C:/Users/akr/code/demo-portal/`

## What This Skill Does NOT Handle

- Product decisions (WHAT to build) — that's the Product Discovery Wizard's job
- Deployment to Cloudflare — use `/product-deploy` for that
- Checking deployment status — use `/product-status` for that
- Demo-portal user management (inviting testers, granting per-product access) — that's a demo-portal admin task; new products just expose `/auth/demo` per the contract above
