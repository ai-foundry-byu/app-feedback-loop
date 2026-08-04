---
name: triage-bugs
description: Pull recent in-app bug reports from the database and produce a fast, DB-only triage, a per-bug brief.json (synthesis + confidence + proposed severity) and one combined HTML dashboard. Use when the user says "triage bugs", "let's look at the recent bugs", "show me the latest bug reports", "help me triage", or names a window ("bugs from the last 2 days", "this week's feedback"). Read-only, it never searches code or fixes anything, it hands each bug off to /fix-bug with a warm-start brief.
---

# triage-bugs, fast DB-only triage of recent reports

Users file bugs through the in-app widget into `bug_reports` (the source of
truth) with a rich context bundle: route, breadcrumbs, console + network rings,
environment, and a screenshot in the private `feedback` bucket. This skill reads
a **window** of recent reports and, for each, writes a `brief.json`, a quick
synthesis grounded ONLY in that captured data, then renders one combined
dashboard for the human and leaves each `brief.json` as the warm-start context
for `fix-bug`.

For an AI-native team this dashboard IS the admin inbox: pair it with the
`bug:resolve` and `bug:ask` CLIs and you may not need the web admin route at
all (see SPEC.md, "Two build profiles").

**This is a lightweight reflex, not an investigation.** Hard rules:

- **DB data only. Never search or read application code.** Everything comes from
  the row + its context bundle + the screenshot. Anything whose answer would
  require the code is a *hypothesis*, flagged `verify_in_code: true`; that is
  `fix-bug`'s job, not this one.
- **One brief per bug, plus one dashboard.** Each `brief.json` is small and
  self-sufficient so `fix-bug` reads exactly one bug, never the aggregate.
- **Honest confidence.** Rate each claim by what the data actually supports.
  Observed-in-screenshot/telemetry = high; reporter's theory or a mechanism =
  lower. The confidence IS the handoff contract.

## Input

An optional time window. **Default: last 72 hours.** Parse plain language,
"last 2 days" is 48h, "this week" is 168h, "since Monday", "today". Call it
`$SINCE` (an interval or absolute timestamp).

## Procedure

1. **Load env** from your env file (e.g. `.env.local`): `DATABASE_URL`,
   `PUBLIC_SUPABASE_URL`, `SUPABASE_SERVICE_ROLE_KEY` or your project's names.
   Read them; don't hardcode.

2. **List the window**, newest first (adjust the timezone to the team's):

   ```bash
   psql "$DATABASE_URL" -P pager=off -c "
     SELECT substring(id::text,1,8) AS id, kind, status,
            COALESCE(severity,'-') AS sev, reporter_role, route,
            created_at AT TIME ZONE 'America/Denver' AS when_local
     FROM bug_reports
     WHERE created_at > now() - interval '72 hours'   -- or \$SINCE
       AND source = 'user'                            -- people first; agent queues on request
     ORDER BY created_at DESC;"
   ```

   Skim it with the user. Many reports in a burst are usually one tester's
   session; note that. Triage every `open` one (and recently-resolved if asked).

3. **Per bug, materialize `.bugs/<short-id>/`** (gitignored, screenshots carry
   user PII; never commit):
   - Pull the full row as JSON (`row_to_json`), joining your profiles table for
     the reporter name.
   - **Download the screenshot** if `screenshot_path` is set. The Supabase
     Storage API needs BOTH headers or it returns a JSON 404 body that gets
     written as the PNG; verify with `file`:

     ```bash
     curl -s \
       -H "Authorization: Bearer $SUPABASE_SERVICE_ROLE_KEY" \
       -H "apikey: $SUPABASE_SERVICE_ROLE_KEY" \
       "$PUBLIC_SUPABASE_URL/storage/v1/object/feedback/<screenshot_path>" \
       -o .bugs/<short-id>/screenshot.png
     file .bugs/<short-id>/screenshot.png
     ```

4. **View the screenshot and read the narrative FIRST** (you read PNGs
   natively, the marked-up screen is real context). Note what they circled and
   what the UI shows. Then read the `expectation` verbatim, then the telemetry
   (breadcrumbs, console + network rings, env) to corroborate and localize.

5. **Author `.bugs/<short-id>/brief.json`**, the single source of truth (schema
   below). Inline the facts so the brief is self-sufficient. Separate the
   **observation** (what happened, anchored in screenshot + telemetry) from the
   **reporter's theory** (why they think it happened, a hypothesis). Propose a
   **severity**. Where the data can't answer something you'd want, add a
   **data gap** (a candidate to add to feedback capture) rather than guessing or
   going to the code.

6. **Render the dashboard yourself.** Write one self-contained HTML file at
   `.bugs/triage-<YYYY-MM-DD>.html` (no external assets, inline CSS): a left
   nav listing every triaged bug (short id, kind, proposed severity, headline)
   and a main pane showing the selected brief, its screenshot (relative path,
   which is why the file lives at the `.bugs/` root), the claims table with
   confidence, and the disposition. Plain, readable, dependency-free.

7. **Hand off.** Print the clickable dashboard path
   (`file:///.../.bugs/triage-<date>.html`) and a one-paragraph chat
   summary: counts by proposed severity, the standouts, anything cross-linked.
   To act on one: `/fix-bug <id>` reads that bug's `brief.json` directly.

## brief.json schema (`bug-brief/1`)

```jsonc
{
  "id": "<full uuid>", "short_id": "<8 char>", "schema": "bug-brief/1",
  "proposed_severity": "low|med|high",        // triage's call (intake leaves it unset)
  "proposed_severity_why": "one line",
  "facts": {                                   // INLINED from the DB, fix-bug needs only this
    "kind": "bug|confusing|idea", "status": "open|triaged|resolved", "severity": null,
    "reporter": { "name": "...", "role": "...", "impersonating": true },
    "created_local": "Mon DD, HH:MM",
    "build": { "app_version": "...", "app_commit": "..." },
    "route": "...", "page_title": "...",
    "verbatim": "the expectation, exactly as typed",
    "screenshot": "screenshot.png",            // path, relative to this file
    "markup": "what the user drew / circled",
    "screen_state": "key state legible in the screenshot",
    "breadcrumbs_tail": [ { "type": "click", "detail": "..." } ],
    "console": [ { "level": "error", "text": "..." } ],
    "network_tail": [ { "status": 200, "method": "GET", "ms": 0, "ok": true, "url": "..." } ]
  },
  "synthesis": {
    "what_happened": "the observation, plain",
    "telemetry": [ { "text": "...", "source": "breadcrumb|network ring|console ring|screenshot" } ],
    "likely_reading": "best inference, explicitly NOT code-verified"
  },
  "claims": [                                  // becomes the confidence table
    { "text": "...", "conf": 1-5, "source": "DB | screenshot | inferred",
      "verify_in_code": true }                 // set on any claim a code read must confirm
  ],
  "overall_confidence": { "label": "High|Mixed|Low", "note": "one line" },
  "reporter_theory": "their stated cause, a hypothesis to disprove, never inherit",
  "data_gaps": [ "what the DB couldn't answer + the capture field that would fix it" ],
  "disposition": {
    "verdict": "real_bug|by_design|cant_repro|misunderstanding",
    "headline": "short", "next": "what fix-bug (or close) should do"
  }
}
```

## Notes

- The DB row is canonical and always current; re-query rather than trust a stale
  local copy.
- If a screenshot 404s or capture failed, proceed without it, and note it as a
  gap.
- This skill does not change any DB state and does not touch git.
- Handoff: `fix-bug` trusts `facts`, but treats every `verify_in_code` claim and
  the `reporter_theory` as something to reproduce and disprove, not inherit.
