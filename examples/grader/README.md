# LawGrader / WriteGrader, the canonical implementation

[LawGrader](https://lawgrader.ai) and [WriteGrader](https://new.writegrader.com)
run the loop exactly as specified: SvelteKit + Supabase, the widget in the root
layout, `bug_reports` / `bug_messages` / `bug_events`, the `/admin/bugs` inbox
with the "Open in Claude Code" handoff, the `/feedback` reporter page, and the
post-commit hook. Most of the production lessons in `SPEC.md` were learned
here, including the textarea capture trap, the `keepalive` 64KB cap, and the
storage 404-as-PNG trap.

![The feedback widget](widget.png)

The widget: docked launcher, auto-captured screenshot with "Include this
screenshot" on by default, one textarea with dictation, the three-kind chip
row, optimistic send.

![The annotator](annotator.png)

The annotator: box, arrow, pen, text, and redact over the captured PNG,
flattened client-side before upload.

![The admin inbox](inbox.png)

The inbox and detail view: status tabs, severity triage, the copyable
`/fix-bug <id>` command, and the full repair markdown one button away.

![The reporter view](feedback-page.png)

The reporter's `/feedback` page after a fix ships: reporter-safe status, the
warm `Bug-Note:` from the fix commit, the two-way thread, and reopen.
