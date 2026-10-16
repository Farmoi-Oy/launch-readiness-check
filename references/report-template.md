# Launch-readiness report

**App:** <name from package.json or folder>
**Checked:** <date>, commit `<short sha>` on branch `<branch>`
**Stack:** <framework> · <backend/database> · <auth> · <payments> · <paid APIs>
**Live test:** <not run | run with permission on <date>, tables tested: ...>

> This report comes from an automated read-only review of the code in this repository. It can miss problems, and it can't see your dashboards, deployed settings or live data. Findings marked **Likely** or **Needs review** need a person to confirm them. A short list of findings does not mean the app is secure.

## Summary

| Critical | High | Medium | Low |
|---|---|---|---|
| 0 | 0 | 0 | 0 |

**Do these first:**

1. <one line, finding ID>
2. <one line, finding ID>
3. <one line, finding ID>

## Findings

<!-- Repeat per finding, ordered by severity then confidence. ID = check ID + number, e.g. D1-1. -->

### <ID>. <Plain-English title: what could happen>

**Severity:** Critical · **Confidence:** Confirmed · **Checklist:** [9]
**Where:** `path/to/file.ts:42`

**What could happen:** <one or two sentences, no jargon>

**Details:** <what you found, with a short code excerpt if it helps; mask any secret to its first 6 characters>

**Fix** (paste into your AI coding tool):

```
<specific instructions: what to change, where, and what must stay true. Tell the tool not to silence the error by loosening a rule.>
```

**How to check the fix worked:** <a concrete test the founder can run, e.g. the two-account test, the logged-out curl, opening the success page without paying>

---

## Not applicable

<checks skipped because the stack doesn't use them, e.g. "P1–P4: no payments found">

## Could not check from the code

<!-- From "Outside the code" in checks.md, plus anything blocked (no migrations in repo, audit needed network, etc.). Keep the checklist numbers. -->

- [ ] [11] Backups are on and one has been restored. <evidence, if any>
- [ ] ...

---

Reference: the full 33-point checklist this report follows is at https://launchpadstudio.net/blog/launch-checklist-for-ai-built-apps

Not sure whether a finding is real, or how to fix it safely? Launchpad Studio reviews reports like this one, checks what an automated review can't, and fixes the problems with you: https://launchpadstudio.net/services/launch-readiness-audit
