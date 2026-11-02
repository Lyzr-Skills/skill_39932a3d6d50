---
name: accessibility-audit
description: >-
  Audits web app pages for accessibility (a11y) issues against WCAG 2.1/2.2 A & AA and generates a
  prioritized Markdown audit report with concrete fixes. Use this whenever the user wants to check,
  test, scan, or audit a website or web app for accessibility, WCAG compliance, ADA/Section 508
  conformance, screen-reader friendliness, color contrast, keyboard navigation, ARIA, or alt text —
  even if they just paste a URL and say "is this accessible?" or "find the a11y problems". Also use it
  when the user asks to "generate an accessibility report" or "fix accessibility issues" on rendered pages.
---

# Accessibility Audit

Audit one or more live web pages for accessibility problems and produce a clear, prioritized
report that an engineer can act on. The skill runs the industry-standard **axe-core** engine in a
real headless browser (so it catches issues in the *rendered* DOM, including JS-driven content),
then layers on a manual-review checklist for the things automation provably cannot detect.

Automated tools reliably catch only ~30-40% of WCAG issues. The goal here is to catch every
machine-detectable issue **and** hand the user a focused list of the manual checks that matter, so
the audit is honest about its coverage rather than giving false confidence.

## When to use this

Trigger on any request to evaluate rendered web pages for accessibility: a URL plus "audit this",
"check WCAG compliance", "find contrast/keyboard/ARIA/alt-text issues", or "make this report so we
can fix it". If the user only has source files and no running server, you can still apply the
review checklist in `references/wcag-checklist.md`, but the automated scan needs a reachable URL.

## Workflow

1. **Get the targets.** Ask for the URL(s) to audit if not given. Multiple pages are fine — audit
   the key user journeys (home, login, a form, a content page), not just the landing page, because
   issues differ per template. If pages sit behind auth, ask the user how to reach them (a public
   staging URL is simplest; otherwise see "Authenticated pages" below).

2. **Run the automated scan.** Use the bundled script. On first run it creates a self-contained
   virtualenv under the skill's own `.venv/`, installs Playwright + the Chromium browser, and
   re-execs itself — so it never touches the user's project or global Python. It then drives a
   headless Chromium, injects the vendored axe-core engine, and tests against the WCAG 2.0/2.1/2.2
   Level A and AA rule sets.

   ```bash
   python3 scripts/audit.py --url "https://example.com" --out ./a11y-report
   ```

   Audit several pages at once by repeating `--url`:

   ```bash
   python3 scripts/audit.py \
     --url "https://example.com" \
     --url "https://example.com/login" \
     --url "https://example.com/contact" \
     --out ./a11y-report
   ```

   Useful flags: `--standard wcag21aa|wcag22aa|wcag2a` (default `wcag22aa`),
   `--include-best-practice` (adds axe best-practice rules beyond strict WCAG),
   `--wait <ms>` (extra settle time for slow SPAs), `--help` for the full list.

   The script writes `<out>/results.json` (raw findings) and `<out>/accessibility-report.md`
   (the human-readable report). If it can't reach a URL or the browser fails to launch, read the
   error it prints — the most common cause is the Chromium binary not being installed, which the
   script installs automatically on first run; surface any remaining error to the user.

3. **Add the manual-review layer.** The automated report ends with a checklist of items axe cannot
   judge (keyboard-only operation, focus order, meaningful reading sequence, caption accuracy,
   sensible link text, etc.). Walk `references/wcag-checklist.md` and, where you reasonably can,
   verify these yourself — e.g. read the page source/structure for heading order, check that
   interactive elements are real buttons/links, look for `lang`, `<title>`, and form-label
   associations. Mark each manual item as Pass / Fail / Needs human review in the report.

4. **Deliver the report.** Present a short summary in chat (counts by severity + the top few
   must-fix items) and point the user to the full Markdown file. Offer to fix the issues in their
   codebase if you have access to it — the report's per-issue "How to fix" guidance is written to
   be directly actionable.

## Report shape

The generated `accessibility-report.md` follows this structure (the script produces it; don't
hand-write it unless the script is unavailable):

```
# Accessibility Audit Report
## Summary            — pages scanned, totals by severity, overall pass/fail per page
## Critical & Serious — must-fix issues first, each with WCAG ref, impact, locations, how to fix
## Moderate & Minor   — lower-priority issues, same per-issue detail
## Per-page breakdown — issues grouped by URL
## Manual review checklist — what automation can't verify, with Pass/Fail/Review status
## Methodology & coverage — tools, standard, and the ~30-40% automation caveat
```

Each automated issue includes: the WCAG success criterion (e.g. `1.4.3 Contrast (Minimum)`),
severity, a plain-language description of the impact on users, every affected element (CSS
selector + HTML snippet), and a specific remediation with a link to the relevant Deque rule page.

## Severity model

Use axe's impact levels, surfacing the most painful issues first so limited engineering time goes
where it helps users most:

- **Critical** — blocks a task entirely for some users (e.g. an unlabeled control they can't operate).
- **Serious** — major barrier with a workaround (e.g. failing color contrast on body text).
- **Moderate** — noticeable friction (e.g. a skipped heading level).
- **Minor** — best-practice / polish.

## Authenticated pages

If a journey requires login, prefer a shared public staging URL. If that's not possible, the script
accepts `--storage-state <file.json>` (a Playwright storage-state export with cookies/localStorage)
so an already-authenticated session can be reused without handling credentials directly. Ask the
user to provide one rather than entering passwords yourself.

## Reference

- `references/wcag-checklist.md` — the full WCAG 2.1/2.2 A & AA checklist with, for each criterion,
  whether it's automatable or manual and what to look for. Read this when doing the manual-review
  layer or when the user asks what a specific criterion means.
