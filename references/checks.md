# Checks

Each check has an ID used in the report. The numbers in brackets refer to the public 33-point checklist at
https://launchpadstudio.net/blog/launch-checklist-for-ai-built-apps, so a founder can read the longer explanation.

Search patterns are starting points. Always read the code around a match before reporting it.

---

## S: Secrets and API keys

### S1. Secret keys reachable from the browser [1]

Anything in a client bundle is public. Look for:

- **Environment variables with a client prefix** that hold secrets. Prefixes that ship to the browser: `NEXT_PUBLIC_`, `VITE_`, `REACT_APP_`, `EXPO_PUBLIC_`, `PUBLIC_` (SvelteKit, Astro), `NUXT_PUBLIC_`, `GATSBY_`. Flag names containing `SECRET`, `SERVICE_ROLE`, `PRIVATE`, `OPENAI`, `ANTHROPIC`, `STRIPE_SECRET`, `SK_`, `WEBHOOK`, `DATABASE_URL`, `ADMIN`.
- **Hardcoded secrets** in source, config or checked-in build output. Patterns:
  - Stripe: `sk_live_`, `rk_live_`, `whsec_` (`sk_test_` is lower severity but still should not be in the browser)
  - Supabase: `sb_secret_`, or a JWT (`eyJ...`) whose decoded payload has `"role":"service_role"`. Decode the middle part (base64) to check; don't assume.
  - OpenAI `sk-proj-` / `sk-`, Anthropic `sk-ant-`, Google `AIza` (only a problem if it's a Gemini/Maps key without restrictions; Firebase web `apiKey` is public by design), AWS `AKIA`, GitHub `ghp_` / `github_pat_`, Slack `xoxb-`, Resend `re_`, Twilio `SK` + 32 hex, SendGrid `SG.`
  - Private keys: `-----BEGIN` ... `PRIVATE KEY-----`
  - Database connection strings with a password: `postgres://user:pass@`, `mongodb+srv://user:pass@`
- **AI SDKs called from the browser**: `dangerouslyAllowBrowser: true`, or `new OpenAI(` / `new Anthropic(` in a file that runs client-side (Vite SPA source, `"use client"` files, React components).
- **Supabase client created with the service-role key** in any file that can reach the browser.

Not findings: Supabase anon/publishable key, Stripe `pk_live_` / `pk_test_`, Firebase web config, Clerk publishable key, PostHog / Sentry public DSNs or project keys.

Severity: live secret key in browser code or bundle = Critical. Service-role key in browser = Critical.

### S2. Secrets committed to the repository [2]

- `git ls-files | grep -iE '(^|/)\.env($|\.)'` and exclude `.env.example`, `.env.sample`, `.env.template`.
- `.env` and `.env.local` should be listed in `.gitignore`.
- History: `git log --all --oneline --diff-filter=A -- '*.env' '*.env.*'` and, if anything shows up, `git show <commit> --stat`. A secret that was ever committed is exposed even if it has since been deleted.
- Search tracked files for the patterns in S1, including scripts, seed files, tests and docs.

Severity: live secret in a public repo = Critical; private repo = High (anyone with access, including any tool connected to the repo, has it).

### S3. Exposed keys have been rotated [3]

You can't verify rotation from code. For every key found in S1 or S2, the Fix must say: create a new key in the provider's dashboard, update it where the app reads it, then revoke the old one. Deleting it from the code is not enough.

### S4. Test and production keys are separated [4]

Look for live keys used in development config, or `sk_test_` / test project URLs hardcoded in code that runs in production. Usually Low/Needs review; the real check is in the dashboard.

---

## A: Accounts and access

### A1. Protected pages require a login [5]

- Next.js: check `middleware.ts` (and its `matcher`) and server-side checks in layouts/pages. Pages that render private data must check the session on the server.
- SPA (Vite/Lovable/Bolt): route guards in React (`<ProtectedRoute>`, `if (!user) navigate(...)`) only hide the UI. Not a finding on its own, but note that the data behind those pages must be protected by RLS or the API (see D1, A2).

### A2. Every server route checks who is calling [6]

List every server entry point: `app/**/route.ts`, `pages/api/**`, `"use server"` functions, `supabase/functions/*/index.ts`, Firebase `functions`, Express/Hono routes. For each, check that it:

1. Gets the current user from a trusted source (session cookie verified on the server, `supabase.auth.getUser()` with the caller's token, Clerk `auth()`, verified JWT). `getSession()` alone on the server reads an unverified cookie in older Supabase setups; prefer `getUser()` or `getClaims()`.
2. Rejects the request when there is no user, before doing any work.

Server actions are public HTTP endpoints. A `"use server"` function without an auth check can be called by anyone.

Supabase Edge Functions: check whether `verify_jwt` is disabled in `supabase/config.toml` for that function, and if so whether the function checks auth itself.

Severity: a route that reads or changes user data without an auth check = Critical.

### A3. One user can't reach another user's data [7]

The most common serious bug in AI-built apps. In each server route and server action, look at how records are fetched or changed:

- An ID taken from the URL, query string or body (`params.id`, `req.body.userId`, `searchParams.get("id")`) and used in a query **without** also filtering by the logged-in user's ID, or checking ownership after loading.
- `userId` accepted from the client instead of taken from the session.
- Supabase queries made with a service-role client inside a route: RLS is skipped, so the route must filter by the user itself.

Severity: Critical when Confirmed.

### A4. Admin powers are checked on the server [8]

- Admin checks based on `user_metadata` (users can edit their own), `localStorage`, a client-side flag, a hidden route, or an email compared in browser code.
- Admin API routes or server actions that check only that someone is logged in, not that they're an admin.

Correct: role in `app_metadata`, a roles table only the server can write, or the auth provider's org/role system, checked on the server.

---

## D: Database and storage

### D1. Access rules are on for every table [9]

**Supabase.** Read every file in `supabase/migrations/` in order (also any `.sql` files elsewhere).

- Every `create table` in the `public` schema (or any schema exposed by the API) needs `alter table ... enable row level security` somewhere later. Tables without it are readable and writable by every visitor.
- Read every `create policy`:
  - `using (true)` or `with check (true)` on a table that holds anything personal or business-sensitive. Public content (blog posts, product catalogue) may be fine as `select ... using (true)`; writes almost never are.
  - Policies `to anon` or with no `to` clause (defaults to `public`, which includes anon) that allow `insert`, `update` or `delete`.
  - `for update` whose `with check` is looser than its `using` (e.g. `with check (true)`): a user can move a row to someone else, or change the owner column. A missing `with check` is not a finding on its own; Postgres then reuses the `using` condition.
  - RLS can't limit which columns change. A user who can update their own row can change every column in it (`role`, `plan`, `credits`) unless column grants or a trigger stop them.
  - `for all` policies: check that both `using` and `with check` are right.
  - Conditions that read `auth.jwt() -> 'user_metadata'` or `raw_user_meta_data`: users can edit this. Never base permissions on it.
  - Policies that compare against a column the user can update themselves (e.g. a `role` column on `profiles` with an update policy that doesn't exclude it).
- `security definer` functions in an exposed schema: callable via `rpc()` by anyone with the public key, and they skip RLS. Check each one checks `auth.uid()` itself and sets `search_path`.
- Views: without `with (security_invoker = true)` they run as their owner and skip RLS.
- `grant ... to anon` on sensitive tables or functions.

**If there are no migrations in the repo** (common with Lovable when changes were made in the dashboard, or projects that never pulled the schema): say clearly that RLS could not be verified from code, mark it as an unchecked Critical risk area, and give the founder these queries to run in the Supabase SQL editor:

```sql
select tablename, rowsecurity from pg_tables where schemaname = 'public' order by tablename;

select tablename, policyname, cmd, roles, qual, with_check
from pg_policies where schemaname = 'public' order by tablename;
```

and point to https://launchpadstudio.net/blog/supabase-row-level-security-check-for-ai-built-apps for how to read the results. Offer the optional live test (SKILL.md step 5).

**Firebase.** Read `firestore.rules`, `database.rules.json`, `storage.rules`:

- `allow read, write: if true;` or `if request.auth != null;` on collections that hold per-user data (any logged-in user can read everyone's).
- Test-mode rules: `if request.time < timestamp.date(...)`.
- Rules files missing from the repo: can't verify; add to unchecked list.

**Direct database (Prisma, Drizzle, Mongoose, raw SQL).** Access control is in the server code; covered by A2 and A3. Also check for SQL built with string concatenation or template literals from user input (`$queryRawUnsafe`, `sql.raw`, `` `... ${req.query.x}` ``).

### D2. File storage is private unless meant to be public [10]

- Supabase: `insert into storage.buckets ... public = true`, or `createBucket(..., { public: true })`, for buckets holding uploads, documents, invoices or avatars of private users. Policies on `storage.objects` with the same problems as D1.
- `getPublicUrl()` used for private documents instead of `createSignedUrl()`.
- Firebase `storage.rules` as above. S3: `public-read` ACLs or a bucket policy with `"Principal": "*"`.

### D3. Backups [11] and D4. Separate development and production databases [12]

Mostly outside the code. Evidence worth noting: a single Supabase URL used everywhere with no dev/staging config; migration or seed scripts that `drop`/`truncate` pointed at the production URL; an MCP or AI-tool config file (`.cursor/mcp.json`, `.mcp.json`) connecting the AI tool to the production database with write access.

---

## P: Payments

Skip this section if there are no payments.

### P1. Live mode only in production [13]

Live keys in development config, or test keys hardcoded where production reads them. Usually Needs review.

### P2. Webhooks are verified [14]

Find the webhook handler (search `webhook`, `stripe.webhooks`, `constructEvent`, `checkout.session.completed`).

- Stripe: must call `stripe.webhooks.constructEvent(rawBody, signature, secret)` (or `constructEventAsync`) on the **raw** body. Parsing JSON first (`await req.json()`, `express.json()` before the route) breaks verification, and AI tools sometimes "fix" the resulting error by removing verification.
- A `try/catch` that logs a signature error and carries on anyway is the same as no verification.
- Lemon Squeezy / Paddle / Polar: HMAC signature checked with a timing-safe comparison.

Severity: unverified webhook that grants access or credits = Critical.

### P3. Paid features are unlocked by the server [15]

- Access granted on the success/thank-you page, from a URL parameter (`?success=true`, `session_id` not checked server-side), or by browser code writing `plan: 'pro'` / `is_paid: true`.
- Users able to update their own plan/credits/subscription columns through an RLS update policy or an unprotected route.
- Price or amount taken from the request body when creating a checkout session instead of a server-side price ID.

Severity: Critical when a user can get paid features without paying.

### P4. Cancellations, failed payments, refunds [16]

Check which events the webhook handles. Missing `customer.subscription.deleted`, `customer.subscription.updated`, `invoice.payment_failed` (or provider equivalents) usually means access never ends. Also check the handler is idempotent (the same event can arrive twice). Usually Medium.

---

## C: Abuse and runaway costs

### C1. Rate limits on expensive actions [17]

Routes that call a paid AI or API, send email or SMS, or handle signup/login/password reset. Look for a rate limiter (Upstash `@upstash/ratelimit`, `express-rate-limit`, Arcjet, Cloudflare/Vercel firewall rules in config, a usage counter in the database). None on an AI endpoint = High; open AI endpoint with no auth and no limit = Critical.

### C2. Spending caps [18]

Outside the code: list each paid provider you found and tell the founder to set a monthly cap and billing alert in each dashboard. In code: AI calls without `max_tokens` / `max_output_tokens`, or where the client chooses the model, prompt length or number of generations.

### C3. Input is checked on the server [19]

File uploads without a server-side size and type limit; text fields with no length limit passed to AI calls or stored; validation only in the form (zod schema used client-side but not in the route). Also `dangerouslySetInnerHTML` / `v-html` with user content (stored XSS).

### C4. Bot protection on public forms [20]

Signup, contact, waitlist and password-reset forms without Turnstile, hCaptcha, reCAPTCHA or an equivalent. Low/Medium.

---

## O: Knowing when it breaks

### O1. Error tracking [21]

Sentry, PostHog error tracking, Bugsnag, Highlight or similar in dependencies and initialised. Absent = Low/Medium.

### O2. Uptime monitoring [22] and O3. Rollback [23]

Outside the code; list for the founder.

### O4. Dependencies [24]

If a lockfile exists, ask the user before running the read-only audit for the package manager, because it sends the dependency list to the registry (`npm audit --omit=dev`, `pnpm audit --prod`, `yarn audit --groups dependencies` for Yarn 1, `yarn npm audit --environment production` for Yarn 2+, `bun audit`). If the user declines or the network call fails, list it as unchecked. Report high and critical advisories in packages the app actually uses at runtime. Also flag a framework version with a known critical advisory you are aware of.

### O5. Other hardening

- CORS `Access-Control-Allow-Origin: *` together with credentials, or reflecting any origin, on routes that use cookies.
- Secrets, tokens or full user records written to logs or `console.log`.
- Error responses that return stack traces or raw database errors to the client.
- Debug or seed routes left in (`/api/debug`, `/api/seed`, `/api/test`, `/admin/reset`).

---

## Outside the code

These can't be answered from the repository. Include each in the report's checklist for the founder, with any evidence you found.

- [3] Every key that was ever exposed has been replaced and the old one revoked
- [4] Live keys only in production; test keys everywhere else
- [11] Automatic backups are on, and one has been restored into a test project
- [12] The AI tool and local development are not connected to the production database
- [13] One real purchase made and refunded on the live site
- [18] Monthly spending cap and billing alert set on every paid AI and API account
- [22] An uptime monitor that alerts you
- [23] You know how to roll back a release, and have tried it once
- [25] Own domain, HTTPS, `www` and non-`www` both work
- [26] SPF, DKIM and DMARC set for the domain the app sends email from
- [27] Signup and password-reset emails arrive at Gmail and Outlook, and their links work
- [28] Privacy policy and terms that name the data you collect and the services you send it to, including AI providers
- [29] Cookie consent before analytics/ads load, if you have EU or UK visitors
- [30] Users can delete their account and data
- [31] The whole journey works on a phone with a fresh account
- [32] Analytics can tell you signups and activation on day one
- [33] Users can reach you from inside the app
