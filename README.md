# Spacecadet

A meeting attention assistant in one self-contained HTML file. Open `spacecadet.html` from
disk — no Node, no npm, no build step, no extensions, no server.

When you snap back from a lapse in focus, it tells you in about two seconds what is being
discussed *right now*, and whether someone just asked you something.

## Quick start

1. Open `spacecadet.html` in Chrome or Edge on Windows.
2. Settings → **My name variants**. The name alert is the fastest thing in the app and it
   needs to know what you are called — including the manglings speech-to-text produces.
3. Click **Start listening** and allow the microphone.

That is the whole setup. There is no API key step, because there is no API key requirement.

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
normal case, not an edge case.

## Summarizer backends

Chosen with the three-way switch in the control strip. Every backend feeds the same
rendering code — the UI cannot tell which one produced what it is showing.

| Backend | Cost | What it does |
|---|---|---|
| **`handoff`** (default) | free | Catch-me-up builds a prompt, copies it, and opens claude.ai. Your Pro subscription pays. This page makes no API call. |
| `api` | metered | Adds the ambient 90-second auto-summary via the Anthropic API with your own key. The only thing gated behind a key. |
| `local` | free | Nothing leaves the machine. The outbound choke point is hard-disabled and blocked attempts are logged. |

Every outbound request goes through a single function, so the network boundary is auditable
at a glance — and visible in the app's own Network log panel.

## Audio sources

- **Microphone** — hears the room and your own speakers. Echo cancellation is forced off,
  because it is designed to suppress exactly the meeting audio you want.
- **System / tab audio** — hears remote participants directly. On **Windows**, share the
  **entire screen** and tick **"Also share system audio"**. A window share carries no audio.
  Chrome on macOS cannot capture system audio at all.

## Speech-to-text engines

- **Web Speech** (default) — zero dependencies, instant. Microphone only; it cannot consume a
  captured system-audio stream, and the app disables the combination rather than silently
  transcribing the wrong thing. Sends audio to Google's speech service.
- **Local Whisper** — `whisper-tiny.en` via transformers.js, WebGPU with a WASM fallback.
  Works with system audio and **audio never leaves the machine**. Needs a one-off ~40 MB
  model download, which corporate proxies often block; the app falls back to Web Speech and
  says so. Transcript quality is mediocre by design — Claude compensates.

## Look

Styled after the inside of an X-wing, from reference photography of the cockpit.

The organising idea: **grey hull for chrome only, dark screens for content**. The status
strip, switch panel, side rails and bezels are weathered mid-grey plate with white scribed
line-work and salmon-orange outline boxes; everything that carries text is a dark screen
recessed into it. That is how the real cockpit is built, and it keeps every reading surface
high-contrast — which is the whole point of the app.

- **NOW** is the targeting computer: amber CRT, scanlines, reticle corners, one slow sweep.
- **ALERT** is the master caution: hazard striping and a red-lit face.
- Side rails carry illuminated key grids, louvred vents and knurled knobs. They are ornament
  and deliberately carry **no data** — a readout that means nothing is worse than none.
- Peak-emphasis moments use the signature crawl yellow.
- Labels stay in plain words. The theme decorates the chrome, never the content.

Two deliberate exemptions: **discreet mode** drops the rails, glow and colour, and the
**panic screen** is not themed at all — it is a plain white document, because it is the one
screen someone else might see.

Type is Bahnschrift, which ships with Windows 10+ and is the condensed technical face the
design is drawn for. On other platforms it falls back through DIN Alternate and Roboto
Condensed to the system sans, which reads wider and less instrument-like.

Motion is transform/opacity only and honours `prefers-reduced-motion`. Nothing animates a
box-shadow or filter on a surface that repaints, because this runs for hours on a locked-down
laptop.

## Keys

`L` listen · `C` catch me up · `S` settings · `D` dismiss alert · `E` export · `V` discreet ·
`Esc` panic-hide, again to restore.

## Notes

- Settings and in-flight meeting state persist to `localStorage`, so an accidental refresh
  mid-meeting is not fatal.
- An API key, if you use one, lives in `localStorage`. A `file://` page offers no meaningful
  secret protection — use a personal, spend-limited key.
- Chrome cannot remember media permissions for a `file://` page, so it re-prompts each time.
- Not built: speaker diarization, meeting-platform integrations, servers, auth, cloud storage.
