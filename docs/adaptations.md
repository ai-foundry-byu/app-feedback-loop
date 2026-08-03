# Adapting the loop beyond page apps

The spec assumes a page-shaped web app: the bug is a page state, the screenshot
captures it, the rings explain it. Some products are not page-shaped. This doc
records a production adaptation, the Eight Faces voice-coaching app (a
real-time speech-to-speech coach), and the patterns it adds. Use them when your
"page" is a moment in time rather than a layout.

## Freeze the moment before you capture

In a real-time app the bug is an instant: the coach interrupted, the model
misheard. By the time a panel opens, that instant is gone. Eight Faces pauses
the live session, the audio playback, and the recorder the moment the feedback
button is pressed, then captures. If your app has live state (audio, video, a
running simulation, a timer), freeze it first and resume on cancel.

## Anchor the report to a moment, not just a route

A page app anchors on `route`. A real-time app anchors on time and turn:
Eight Faces stamps every report with `at_ms` (elapsed session time) and `turn`
(which coach utterance), and, with consent, keeps the transcript and audio
recording on the same row. The agent fixing the bug can *listen* to it. The
general rule: whatever your product's unit of experience is (a turn, a video
timestamp, a game tick), capture it as a first-class field.

## Reason chips beat free text for repeated failure modes

When the same five things go wrong repeatedly, one tap on a chip ("interrupted",
"too generic", "missed the point") outperforms a textarea. Chips make reports
aggregatable: the review CLI counts failure modes instead of parsing prose.
Keep the free-text field, but let chips carry the classification.

## Version the thing you are tuning

Eight Faces hashes the coach prompt into a `prompt_version` stamped on every
session. That single field turns the feedback store into an experiment log:
"effectiveness by prompt version" is a query, not a guess, and the tuning
skill (/tune-coach, its /fix-bug analog) closes the loop with a `Coach-Note:`
trailer exactly like the bug hook. If your AI behavior is steered by a prompt
or config, hash it and stamp it.

## Upload big blobs directly, sign server-side

Screenshots and audio go straight from the browser to the private bucket via
short-lived signed upload URLs minted by the server. The service key never
ships to the client, and large blobs never transit your serverless function.
The intake endpoint then stores only the path.

## The pattern behind the pattern

Every adaptation above is the same move: identify what context an agent would
need to relive the moment, capture it passively at report time, and keep one
close path that ends with a warm note bound to a commit. The medium changes;
the loop does not.
