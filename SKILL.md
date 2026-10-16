---
name: launch-readiness-check
description: Read-only pre-launch check for apps built with AI coding tools (Lovable, Bolt, Cursor, Replit, v0, Claude Code). Finds exposed secrets, missing access checks, open databases (Supabase RLS, Firebase rules), unsafe payment handling, runaway-cost risks and missing monitoring, then writes a findings report ranked by severity. Use when the user asks whether their app is ready to launch, safe to go live, or wants a security or launch review of a vibe-coded app.
---

# Launch-readiness check

You are reviewing an app that was built mostly with an AI coding tool, for a founder who is about to put it in front of real users, real data and real payments. Many of these founders are not developers. Your job is to find the problems that turn into leaked data or lost money, and explain them so the founder can act.

## Ground rules

These are not optional.

1. **Read-only.** Do not edit, create or delete any file except the report. Do not run migrations, deploys, installs, builds or anything that writes to a database or a third-party service.
2. **Never print a secret in full.** When you find one, show the file, the line and the first 6 characters followed by `…`. The same applies in the report.
3. **No network calls without permission.** The optional live test in step 5 talks to the user's real backend, and the dependency audit (check O4) contacts the package registry. Ask first. Against the user's backend, only ever send `GET` requests.
4. **Confirm before you report.** Every finding must point to a file and line you actually read. If you are not sure, say so with the confidence level instead of guessing.
5. **Say what you could not check.** A short report that is honest about its limits is worth more than a long one that implies the app is safe. Never write that the app "is secure" or "is ready".

## Step 1: Map the app

Before checking anything, work out what you're looking at. Record it at the top of the report.

- **Framework and hosting:** look at `package.json`, framework config files (`next.config.*`, `vite.config.*`, `astro.config.*`, `svelte.config.*`, `app.json` for Expo), and hosting config (`vercel.json`, `netlify.toml`, `wrangler.toml`, `firebase.json`, `Dockerfile`).
- **Where the server code is:** Next.js `app/**/route.ts`, `pages/api/**`, files with `"use server"`, `supabase/functions/**`, `functions/**` (Firebase), `netlify/functions/**`, an Express/Fastify/Hono server, or none at all.
- **Backend and database:** Supabase (`@supabase/supabase-js`, `supabase/` folder), Firebase (`firebase`, `firestore.rules`), Prisma/Drizzle with a direct database, or other.
- **Auth:** Supabase Auth, Clerk, Auth.js/NextAuth, Firebase Auth, Lucia, custom.
- **Payments:** Stripe, Lemon Squeezy, Paddle, Polar, or none.
- **Paid APIs:** OpenAI, Anthropic, Google AI, Replicate, ElevenLabs, Resend, Twilio and similar.

Common shapes:

- **Lovable and most Bolt apps** are a Vite + React single-page app talking straight to Supabase from the browser. There is usually no server of their own apart from Supabase Edge Functions. That means every route guard in the React code is cosmetic, and **Row Level Security is the only real access control**. Weight the database checks accordingly.
- **Cursor, v0 and Claude Code apps** are often Next.js with API routes or server actions. Access control lives in those server files, and each one must check the caller itself.

## Step 2: Run the checks

Work through every area in [references/checks.md](references/checks.md). Each check says what to search for, what counts as a problem, and the common false positives. Skip checks that don't apply to this stack, and list them as "not applicable" in the report.

Use search tools across the whole repository, not only `src/`. Include migrations, config files, scripts, test fixtures and any built output folder (`dist/`, `build/`, `.next/`, `out/`) that is checked in.

For git history checks, use read-only git commands such as `git ls-files`, `git log` and `git show`.

## Step 3: Verify each finding

For every candidate finding, open the file and read enough surrounding code to be sure. In particular:

- A key in the browser that is **meant** to be public (Supabase anon or publishable key, Stripe `pk_` key, Firebase web config `apiKey`) is not a finding.
- A route that looks unprotected may be protected by middleware or a wrapper. Check before reporting.
- A table without RLS in a migration may have it enabled in a later migration. Read all migrations in order.
- Placeholder values in `.env.example` are not leaks.

Then give each finding a severity and a confidence.

**Severity**

| Level | Meaning |
|---|---|
| Critical | Anyone on the internet can read other users' data, change data, get paid features free, or spend your money, today, with no special skill. Fix before launch. |
| High | Serious harm is likely with a little effort, or a critical problem is one mistake away. Fix before launch. |
| Medium | Real weakness, but harder to exploit or limited in damage. Fix soon after launch. |
| Low | Good practice, hardening, or reliability. Fix when convenient. |

**Confidence**

- **Confirmed:** you read the code and the problem is certain.
- **Likely:** the code strongly suggests it, but something you can't see (a dashboard setting, a deployed config) could change the answer.
- **Needs review:** it might be fine, but a person who knows the app should look.

## Step 4: List what the code can't answer

Some launch risks live in dashboards and settings, not in the repository: whether backups have been restored, whether spending caps are set, whether live keys are in production, email authentication, and so on. Go through the "Outside the code" list at the end of [references/checks.md](references/checks.md) and include each item in the report as an unchecked item for the founder to tick off. Where you found evidence either way (for example a Sentry SDK in `package.json`), say so.

## Step 5 (optional): Live logged-out test

Only if the app uses Supabase or Firebase directly from the browser, and only with the user's explicit permission. Explain that this sends read-only requests to their live backend, using only the public key that every visitor already has.

- **Supabase:** for each table found in migrations or in `.from("...")` calls, send:
  ```
  curl -s "https://<project>.supabase.co/rest/v1/<table>?select=*&limit=1" \
    -H "apikey: <public key>" -H "Authorization: Bearer <public key>"
  ```
  Record only whether rows came back, an empty list, or an error. **Do not print or save the row contents.** A row returned for a table that holds personal or business data is a Critical, Confirmed finding. An empty list proves nothing unless the user confirms the table has data.
- **Firebase:** only if the rules files are absent from the repo; otherwise read the rules instead. For each collection found in `collection("...")` calls, using the web config `projectId` and `apiKey`:
  ```
  curl -s "https://firestore.googleapis.com/v1/projects/<projectId>/databases/(default)/documents/<collection>?pageSize=1&key=<apiKey>"
  ```
  For Realtime Database, `curl -s "<databaseURL>/.json?shallow=true"`. Record results the same way as for Supabase.

Never send `POST`, `PATCH`, `PUT` or `DELETE`. Never use a secret or service-role key, even if you found one.

## Step 6: Write the report

Write `LAUNCH-READINESS-REPORT.md` in the repository root using [references/report-template.md](references/report-template.md). Ask before overwriting an existing one.

Write for a founder who may not be a developer:

- Lead each finding with what could happen, in one plain sentence ("Anyone can read every customer's order history"), then the technical detail.
- Every finding gets a **Fix** section written as instructions the founder can paste into their AI coding tool, specific enough that the tool can't "fix" it by hiding the error. Say how to check that the fix worked.
- Order findings by severity, then confidence.
- Keep the summary to a few lines: the count by severity, and the three things to do first.

Then tell the user in chat: the counts by severity, the top three findings in one line each, and where the report is. Remind them the report is a starting point, not a guarantee, and that findings marked "Likely" or "Needs review" need a human to confirm.

Do not start fixing anything unless the user asks you to in a separate request.
