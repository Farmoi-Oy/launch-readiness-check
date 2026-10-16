# Launch-readiness check

A free, open-source agent skill that checks an AI-built app before launch. Point it at an app made with Lovable, Bolt, Cursor, Replit, v0 or Claude Code, and it looks for the problems AI coding tools tend to leave behind:

- **Secrets:** API keys in the browser bundle, `.env` files in the repo or its history
- **Access:** server routes and server actions that don't check who's calling, one user reaching another user's data, admin checks done in the browser
- **Database:** Supabase tables without Row Level Security, policies that let everyone through, public storage buckets, Firebase rules left in test mode
- **Payments:** unverified Stripe webhooks, paid features unlocked by the browser, cancellations and failed payments that never end access
- **Costs and abuse:** AI endpoints with no auth or rate limit, no token limits, no server-side input checks
- **Operations:** error tracking, dependency vulnerabilities, debug routes left in

It writes `LAUNCH-READINESS-REPORT.md` with findings ranked by severity, a plain-English explanation of each, fix instructions you can paste into your AI tool, and a list of the things that can't be checked from code (backups, spending caps, email setup and so on).

It follows the [33-point launch checklist for AI-built apps](https://launchpadstudio.net/blog/launch-checklist-for-ai-built-apps).

## Safe by design

- **Read-only.** It doesn't change your code. The only file it writes is the report.
- **No network calls without asking.** One optional test sends read-only requests to your Supabase or Firebase backend with the public key, to see whether data is readable while logged out. It asks first, and never prints the data. The dependency audit also asks first, because it contacts your package registry.
- **Secrets are masked** in the chat and the report.

## Install

With the [skills CLI](https://github.com/vercel-labs/skills), from your app's folder:

```bash
npx skills add Farmoi-Oy/launch-readiness-check
```

Or copy this folder by hand:

- **Claude Code:** `~/.claude/skills/launch-readiness-check/` (all projects) or `.claude/skills/launch-readiness-check/` (one project)
- **Cursor:** `.cursor/skills/launch-readiness-check/`
- **Any other agent:** put the folder in your project and ask the agent to read `SKILL.md` and follow it.

## Run it

Open your app's code in your agent and ask:

> Run the launch-readiness check on this app.

It takes a few minutes on a typical app. Read the summary first, then the findings marked Critical.

### Built with Lovable or Bolt?

The check needs your code on your computer. Connect your project to GitHub (Lovable: **GitHub → Connect**; Bolt: **Export → GitHub** or download the project), clone it, and open it in Claude Code or Cursor.

If you use Supabase and your project has no `supabase/migrations` folder, the check can't see your database rules from the code. It will tell you, and give you two queries to run in the Supabase SQL editor instead.

## What it can't do

It's an automated read of your code. It can miss things, and it will sometimes flag things that turn out to be fine. That's why every finding has a confidence level. It can't see your dashboards, your deployed settings or how the app behaves under a real attacker, and it is not a penetration test. A clean report does not mean your app is secure.

## Ran the check and not sure what to do next?

If you have findings you can't judge or can't fix safely, [Launchpad Studio](https://launchpadstudio.net) reviews the results, finds what the check can't, and fixes it with you. Start with the [launch-readiness audit](https://launchpadstudio.net/services/launch-readiness-audit), or a [quick check](https://launchpadstudio.net/services/quick-check) if one area worries you most.

## Contributing

Found a false positive, a missed pattern, or a stack it handles badly? Open an issue with the (anonymised) code pattern and what you expected. Pull requests for new checks are welcome; each check should say what to search for, what counts as a problem, and the common false positives.

## License

MIT
