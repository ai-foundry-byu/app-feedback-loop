# app-feedback-loop

**An AI-native, in-app feedback loop: one click captures a screenshot and the machine context around it, an agent fixes the bug with that context loaded, and the fix commit itself closes the loop and thanks the reporter.**

This is not a library. It is a **build spec** that a coding agent implements natively in whatever stack your app uses, in your design system, your auth, your database, with no dependency to install or maintain.

## Install it in your app

Open Claude Code (or any capable coding agent) in your app's repo and say:

```
Read https://github.com/ai-foundry-byu/app-feedback-loop/blob/main/SPEC.md
and implement this feedback loop natively in my stack. When you believe you
are done, run the checks in VERIFY.md and show me the results.
```

That is the whole install. The agent builds the widget, the schema, the intake
endpoint, the admin inbox, the reporter page, the `/fix-bug` skill, and the
close-the-loop git hook, then proves each piece works.

## The loop

```
  USER                        SYSTEM                         ADMIN / AGENT
  ----                        ------                         -------------
  click Feedback
  (or Cmd/Ctrl+Shift+F)
        |
        v
  panel: describe +     -->   capture context bundle
  pick kind +                 (route, breadcrumbs, console,
  screenshot (opt)            network, errors, env, screenshot)
        |                     scrub secrets client-side
        v                            |
      Send         -->  POST  -->    insert bug_reports row
                                     upload screenshot (private)
                                     stamp server metadata
                                     log bug_events: created
                                            |
                                            v
                                     admin inbox  -------------->  triage: severity + summary
                                     (/admin/bugs)                 "Open in Claude Code"
                                            |                       => /fix-bug <id>
                                            |                              |
                                            |                              v
                                            |                       agent pulls row + context
                                            |                       + screenshot, verifies,
                                            |                       reproduces, fixes
                                            |                              |
                                            |                       commit: "...(bug <id>)"
                                            |                       Bug-Note: <warm sentence>
                                            v                              |
            notify reporter  <--  resolveBug()  <--  post-commit hook -----+
            "We fixed something                (parses id + trailer,
             you reported"                      calls resolve --via commit)
                  |
                  v
        reporter sees it on /feedback: status "Fixed", the note,
        and they can reply or reopen
```

## Why this exists

Most feedback systems capture a sentence and lose everything around it. By the
time someone looks, the context is gone: what screen, what the user clicked,
what errored, what the network did. Reports end up vague, reproduction is
guesswork, and the reporter never hears back.

This design inverts that, because it assumes **a coding agent does the fix**:

- **Capture is rich and passive.** One click grabs the route, a screenshot
  (with an annotate-and-redact editor), the recent click trail, the console and
  network rings, thrown errors, and the environment, all scrubbed of secrets
  before anything leaves the browser.
- **The handoff is one command.** From the inbox, "Open in Claude Code" copies
  `/fix-bug <id>`. The agent pulls the row, rebuilds the session, downloads the
  screenshot, and starts with the same context the user had. The bundled skill
  makes the agent verify the bug is real before touching code.
- **The loop closes itself.** The fix commit carries a `Bug-Note:` trailer; a
  git hook resolves the report and notifies the reporter with that exact
  sentence. The database refuses to close a report as fixed without one, so
  the loop cannot close silently or rudely.

## What it looks like

| The widget | Annotate and redact | The admin inbox |
|---|---|---|
| ![Feedback widget](examples/grader/widget.png) | ![Annotator](examples/grader/annotator.png) | ![Admin inbox](examples/grader/inbox.png) |

More in [`examples/`](./examples), including the closed-loop reporter view and
the [Eight Faces voice adaptation](./docs/adaptations.md).

## What's in this repo

| File | What it is |
|---|---|
| [`SPEC.md`](./SPEC.md) | The build spec: widget, schema, context bundle, backend, hook, inbox, reporter page. Stack-agnostic, Supabase shown as the worked default. |
| [`VERIFY.md`](./VERIFY.md) | The self-test checklist the implementing agent runs before claiming done. |
| [`skills/fix-bug/`](./skills/fix-bug/SKILL.md) | The agent side: `/fix-bug <id>` loads the report and enforces verify-before-fix. Drop into `.claude/skills/`. |
| [`githooks/post-commit`](./githooks/post-commit) | The keystone hook: a commit tagged `(bug <id>)` with a `Bug-Note:` closes the report and notifies the reporter. |
| [`examples/`](./examples) | Real production implementations with screenshots. |
| [`docs/adaptations.md`](./docs/adaptations.md) | Adapting the loop to real-time media (the Eight Faces voice-coach pattern). |

## Why a spec instead of a package

A widget package would be one stack, one look, one schema migration away from
abandoned. A spec survives all of that: the agent implements it in your
conventions, it looks like your product because it is your product, and there
is nothing to `npm audit` forever after. The hard-won details live in the spec
text: the DOM-clone textarea trap, the 64KB `keepalive` body cap that silently
ate screenshots, the storage-API 404 that masquerades as a PNG, the database
constraint that makes a warm reporter note mandatory. Those lessons came from
production; the spec says which ones did.

## Who runs this

Built and dogfooded by [AI Foundry](https://aifoundry.byu.edu), the AI product
studio at the BYU Marriott School of Business. In production at
[LawGrader](https://lawgrader.ai) and [WriteGrader](https://new.writegrader.com),
adapted for voice in Eight Faces, and part of the standard build for every app
the Foundry ships.

Extracted from the [agent-team-kit](https://github.com/sdmurff/agent-team-kit),
which pairs this loop with agent QA harnesses that file into the same inbox.

## License

[MIT](./LICENSE)
