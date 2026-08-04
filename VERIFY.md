# VERIFY, the self-test for an implementation

You (the implementing agent) run this after building the loop from `SPEC.md`.
Every line is checkable without human help except the last section. Report
results as a table: check, PASS/FAIL, evidence (a command output, a row id, a
screenshot path). Do not report done until everything passes or the failures
are listed explicitly.

## 1. Capture

- [ ] The widget mounts on every page (root layout), docked bottom-right, and
      opens via click AND the keyboard chord.
- [ ] Opening the panel shows a screenshot preview of the current page. The
      widget itself does not appear in its own screenshot, and neither does
      any element marked `data-feedback-ignore`.
- [ ] Type into a `<textarea>` on the page, then capture: the typed text
      appears in the screenshot (the DOM-clone textarea trap from SPEC Part A).
- [ ] Fill a report on a page that just logged a console error and a failed
      fetch: submit, then confirm the stored `context` contains both, plus
      breadcrumbs, viewport, route, and perf fields (SPEC Part C shape).
- [ ] Put a fake JWT (`eyJx.eyJy.zzz`) and a 40+ char token into the URL query
      and console before capturing: the stored context shows `[redacted-jwt]`
      and `[redacted]`, never the raw values.

## 2. Storage and intake

- [ ] The `bug_reports` row has server-stamped `app_commit`, and the reporter's
      role/org fields when a session exists.
- [ ] The screenshot exists in the PRIVATE bucket at `{id}/screenshot.png`; an
      unauthenticated GET of the raw storage URL fails.
- [ ] A `bug_events` row exists: `kind='created'`, `to_status='open'`,
      `via='reporter'`.
- [ ] Submit 7 reports inside 2 minutes as one user: the 7th returns
      `{ok:true, throttled:true}` and inserts no row.
- [ ] Try to set `status='resolved'` with `resolution='fixed'` and NO
      `reporter_note` directly in SQL: the CHECK constraint rejects it.

## 3. The loop

- [ ] `/fix-bug <short-prefix>` materializes `.bugs/<short-id>/` with
      `context.json`, `report.md`, and a real PNG (run `file` on it).
- [ ] A commit with subject `... (bug <id>)` and a `Bug-Note:` trailer flips
      the row to `status='resolved', resolution='fixed'`, records a
      `bug_events` `resolved` row with `via='commit'`, and stores the note.
- [ ] A commit tagged `(bug <id>)` WITHOUT a `Bug-Note:` prints the loud
      warning and leaves the row open.
- [ ] The reporter's `/feedback` page shows the report as "Fixed" with the
      note, and never shows internal words (`open`, `triaged`, `noise`,
      `wontfix`).
- [ ] Reporter reply and reopen both work, and reopen appends a `reopened`
      event without erasing the prior resolution from the event history.

## 4. Triage (run the block for the profile you built)

Profile A, agent-first:

- [ ] `/triage-bugs` over the test window writes a `brief.json` per report
      and renders `.bugs/triage-<date>.html` with every report in the nav.
- [ ] Each brief separates `facts` / `synthesis` / `claims` and flags at
      least one `verify_in_code` claim where the data warrants it.
- [ ] `bug:ask <id> "..."` posts a `team` message, notifies the reporter,
      and flips the reporter's card to "Needs your input".

Profile B, admin inbox:

- [ ] `/admin/bugs` lists the report; non-admin access is refused.
- [ ] The detail page shows the screenshot (via signed URL or the proxy
      endpoint, and the proxy re-checks admin inside the endpoint).
- [ ] "Open in Claude Code" copies `/fix-bug <id>`; the second button copies
      the full repair markdown.
- [ ] Triage sets severity + summary, flips status to `triaged`, stamps
      `triaged_at` once, and logs a `triaged` event.

## 5. Human checks (ask your user)

- [ ] The panel look and placement fit the app's design system.
- [ ] The Bug-Note voice: show your user one example note and confirm the tone.
