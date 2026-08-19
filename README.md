# SpaceCadet

A meeting attention assistant in one self-contained HTML file. Open `spacecadet.html` from
disk — no Node, no npm, no build step, no extensions, no server, no account.

When you snap back from a lapse in focus, it tells you in about two seconds what is being
discussed *right now*, and whether someone just asked you something.

## Quick start

1. Open `spacecadet.html` in Chrome or Edge on Windows.
2. Pick where transcription happens — on-device or Google's speech service. It asks once.
3. Settings → **My name variants**. The name alert is the fastest thing in the app and it
   needs to know what you are called — including the manglings speech-to-text produces.
4. Settings → **Who is in the room**. Add the people; then press `1`–`9` as each one speaks.
5. Click **Start listening**, confirm you have told the room, and allow the microphone.

There is no API key step, because there is no API key requirement.

## Pipeline

```
audio capture → speech-to-text → rolling timestamped buffer ─┬─ fast loop (local, instant)
                                                             └─ slow loop (optional, Claude)
```

Claude cannot accept audio — the Messages API is text/image/PDF only — so speech-to-text
happens in the browser and only text is ever sent anywhere.

The two loops are independent. The **fast loop** is plain regex and keyword maths over the
transcript buffer: name alerts, question detection, keyword chips, topic-change detection.
It works with the API down, with no key ever entered, and in local-only mode. That is the
normal case, not an edge case. The **slow loop**, when you turn it on, adds the part local
code cannot do: resolving what "that" referred to, and what kind of answer is wanted.

## What you get on screen

- **Alert** — fires the moment your name or a question aimed at you appears. It carries the
  question, the exchange it came out of, what the pronoun in it referred to, and the
  direction of a sensible answer. The whole point is that you can answer from it cold.
- **Now** — the current topic in text readable at a glance, with the last stretch of speech
  under it and the recent topics beside it.
- **Level 10 segments** — Check-in, Scorecard, Rocks, Headlines, To-dos, IDS, Conclude.
  Tap one and everything captured from then on is stamped with it, so the export reads as an
  agenda log rather than a word cloud. IDS runs a clock on the issue being discussed.
- **To-dos, issues, decisions, glossary** — captured as they are said, each with an owner
  where one can be worked out. Unticked to-dos carry into your next meeting.
- **Bookmarks** (`B`) — mark a moment you want to come back to, with the surrounding context.

## Summarizer backends

Chosen with the three-way switch in the control strip. Every backend feeds the same
rendering code — the UI cannot tell which one produced what it is showing.

| Backend | Cost | What it does |
|---|---|---|
| **`handoff`** (default) | free | Catch-me-up builds a prompt, copies it, and opens claude.ai. Your Pro subscription pays. This page makes no API call. |
| `api` | metered | Adds the ambient auto-summary and reference resolution via the Anthropic API with your own key. The only thing gated behind a key. |
| `local` | free | Nothing leaves the machine. The outbound choke point is hard-disabled and blocked attempts are logged. |

Every outbound request goes through a single function, so the network boundary is auditable
at a glance — and visible in the app's own Network log panel. A Content-Security-Policy
header bounds it a second time, in the browser rather than in our own code.

## Audio sources

- **Microphone** — hears the room and your own speakers. Echo cancellation is forced off,
  because it is designed to suppress exactly the meeting audio you want.
- **System / tab audio** — hears remote participants directly. On **Windows**, share the
  **entire screen** and tick **"Also share system audio"**. A window share carries no audio.
  Chrome on macOS cannot capture system audio at all.

## Speech-to-text engines

- **Web Speech** — zero dependencies, instant, noticeably more accurate. Microphone only; it
  cannot consume a captured system-audio stream, and the app disables the combination rather
  than silently transcribing the wrong thing. Sends audio to Google's speech service.
- **Local Whisper** — `whisper-tiny.en` via transformers.js, WebGPU with a WASM fallback.
  Works with system audio and **audio never leaves the machine**. Needs a one-off ~40 MB
  model download, which corporate proxies often block; the app falls back and says so.
  Transcript quality is mediocre by design — the intelligence layer compensates.

## Who said what

Two mechanisms, and they stack:

- **Manual** — click a name in the roster bar, or press its number key. Everything after that
  is attributed to them until you change it. This is the reliable one, and it is what gives
  a to-do a real owner.
- **Voice memory** (opt-in) — record about eight seconds per person once, and the app will
  guess who is speaking in later meetings. It stores a short list of numbers describing pitch
  and tone, never recorded audio, and never sends any of it anywhere. Guesses are marked with
  a **?** and never overwrite a name you picked. Microphone source only: a shared
  system-audio stream is one mixed channel and cannot be separated this way.

Voice matching is approximate. It handles a handful of clearly different voices in decent
audio, and gets confused by similar voices, crosstalk and speakerphones.

## Governance

Built for someone whose meetings include personnel matters, and reviewed by people who would
have to answer for it:

- **Consent gate** before any capture, once per session, every session — never remembered.
- **Persistent recording indicator** that discreet mode cannot suppress.
- **Sensitive meeting** switch hard-disables export, the claude.ai handoff and every API call
  for as long as it is on. Use it for personnel matters, investigations, board executive
  session, or anything involving a minor.
- **Retention** — stored meetings are purged after a set number of days, enforced on load and
  while running, rather than waiting for someone to remember.
- **Privacy panel** in Settings states where data goes under *your current settings*, not as a
  general claim, and updates as you change them.
- Attention alerts are deliberately **not** exported. A log of the moments one person's focus
  lapsed is a surveillance artifact, not meeting notes.
- Settings export/import as JSON so a whole team can run one configuration. The API key is
  never included, and an imported config cannot touch the governance settings — sensitivity,
  backend, retention, spend cap and the model download are changeable only at the keyboard of
  the machine they apply to.

## Exports

`E` writes Markdown. The export menu and the meeting snapshot also offer plain text, a
structured `.json` record, and **Copy for Strety** — a block shaped to paste straight into the
matching To-dos, Issues and Headlines lists rather than making someone retype the meeting.

A saved calendar `.ics` invite can be loaded from disk to fill in the title and attendees.
Nothing is uploaded and no calendar account is connected — that would need a server this app
deliberately does not have.

## Look

Aligned to Strety, because that is what is already open on the screen next to it: charcoal
top bar, light page, white cards, hairline borders, one orange primary action, the signature
multicolour rule, agenda segments as coloured pills.

It is laid out for a column about **one third of a 1920px screen** (~600px), docked beside
Strety and Teams. Everything is single-column and vertical, and holds from 380px to full
width. Vertical space is the scarce resource, so the transcript takes whatever is left and
everything else is sized to get out of its way.

Two deliberate exemptions: **discreet mode** drops the colour and compacts everything, and
the **panic screen** is not themed at all — it is a plain white document, because it is the
one screen someone else might see.

Motion is transform/opacity only and honours `prefers-reduced-motion`. Nothing animates a
box-shadow or filter on a surface that repaints, because this runs for hours on a locked-down
laptop.

## Keys

`L` listen · `C` catch me up · `B` bookmark · `T` full transcript · `R` meeting snapshot ·
`E` export · `D` dismiss alert · `V` discreet · `S` settings · `1`–`9` who is speaking ·
`Esc` panic-hide, again to restore.

`Esc` blanks the screen instantly, always, from anywhere including a text field. Whether it
*also* stops the microphone is a setting, off by default.

## Notes

- Settings and in-flight meeting state persist to `localStorage`, so an accidental refresh
  mid-meeting is not fatal.
- An API key, if you use one, lives in `localStorage`. A `file://` page offers no meaningful
  secret protection — use a personal, spend-limited key.
- Chrome cannot remember media permissions for a `file://` page, so it re-prompts each time.
  Serving the file over https fixes that permanently.
- Not built: meeting-platform integrations, servers, auth, cloud storage, calendar OAuth.
