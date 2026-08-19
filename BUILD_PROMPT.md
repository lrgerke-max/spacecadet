# One-shot build prompt

> Paste everything below the line into a fresh Claude session. It is self-contained.

---

Build **Spacecadet**, a meeting attention assistant, as **one self-contained `spacecadet.html` file**.

I open it from disk (`file://`) on a locked-down corporate Windows laptop. I cannot install anything —
no Node, no Python, no build step, no npm, no extensions. Everything must work from that one file.
Write the complete file, no placeholders and no "TODO" stubs.

## What it's for

I lose focus in meetings. When I snap back I need to look at a second screen and know, in about two
seconds, what is being discussed *right now* — and critically, whether someone just asked me
something. Design every UI decision around a two-second glance by someone who is mildly panicking.

## Pipeline

    audio capture → speech-to-text → rolling timestamped buffer → periodic Claude call → dashboard

**Claude cannot accept audio.** The API is text/image/PDF only. Speech-to-text must happen in the
browser; only text is ever sent to the API. Do not attempt to send audio to Anthropic.

Run two independent loops:

- **Fast loop (local, free, instant):** regex-scan every incoming transcript fragment for my name
  and question patterns. Fires the "you're being spoken to" alert with no network round-trip.
- **Slow loop (Claude, every 90s):** produces topic, bullets, decisions, action items, talking points.

Keep them decoupled. The fast loop must still work when the API is down, the key is missing, or
local-only mode is on.

## Audio capture — two sources, user-selectable

1. **Microphone** — `getUserMedia({audio: {echoCancellation: false, noiseSuppression: false, autoGainControl: true}})`.
   Echo cancellation must be **off**: it actively suppresses meeting audio coming from my own speakers.
2. **System / tab audio** — `getDisplayMedia({video: true, audio: true})`, then keep only the audio track.
   `video: true` is required — Chrome will not offer audio-only display capture. Stop the video track
   immediately after acquiring the stream to save CPU.

Detect and clearly report the case where the user shares a screen but leaves "share system audio"
unchecked — the resulting stream has no audio track. Say exactly that, don't fail silently.

## Speech-to-text — two engines behind one interface

Define a single interface both engines implement:

```js
// start(stream, onFragment) -> stop()
// onFragment({ text: string, isFinal: boolean, t: number /* epoch ms */ })
```

**Engine A — Web Speech API** (default; zero dependencies)
- `webkitSpeechRecognition`, `continuous = true`, `interimResults = true`, `lang = 'en-US'`.
- **Critical gotcha:** it stops on its own after silence. You MUST reattach in `onend` and restart,
  guarded by a `shouldBeRunning` flag so an intentional stop doesn't resurrect it. Without this the
  app silently dies after ~30 seconds and this is the single most common way this build fails.
- Handle `onerror` for `no-speech`, `audio-capture`, `not-allowed`, `network` distinctly.
- **Limitation to surface in the UI:** it only ever listens to the *microphone*. It cannot consume a
  `getDisplayMedia` stream. If the user selects system audio, this engine must be disabled with a
  visible explanation, not silently ignored.
- Note in the UI that this engine sends audio to Google's servers.

**Engine B — Local Whisper** (for system/tab audio, and for keeping audio on-machine)
- `transformers.js` from a CDN, `automatic-speech-recognition`, model `Xenova/whisper-tiny.en`.
- Try WebGPU, fall back to WASM.
- Buffer ~20–30s of audio via `AudioWorklet` or `ScriptProcessorNode`, resample to **16 kHz mono
  Float32** (Whisper requires this — use an `OfflineAudioContext` to resample), transcribe, emit.
- **Verify the current CDN URL and package name yourself** (the package was renamed from
  `@xenova/transformers` to `@huggingface/transformers`) — do not trust a remembered version string.
- The CDN may be blocked by corporate proxy. On any load failure, fail gracefully to Engine A with a
  clear message. Never leave the app in a broken state because a CDN was unreachable.
- Say plainly in the UI: with this engine, audio never leaves the machine.

Transcript quality from `whisper-tiny.en` is mediocre. That is acceptable and expected — Claude
compensates. Do not add error-correction logic.

## Calling Claude

Exact request. These headers are verified working from a `file://` origin:

```js
await fetch("https://api.anthropic.com/v1/messages", {
  method: "POST",
  headers: {
    "content-type": "application/json",
    "x-api-key": apiKey,
    "anthropic-version": "2023-06-01",
    "anthropic-dangerous-direct-browser-access": "true"
  },
  body: JSON.stringify({
    model: selectedModel,              // default "claude-opus-5"
    max_tokens: 2000,
    system: SYSTEM_PROMPT,
    output_config: {
      effort: "low",                   // keep latency down; this is a live loop
      format: { type: "json_schema", schema: RESPONSE_SCHEMA }
    },
    messages: [{ role: "user", content: userContent }]
  })
})
```

Model dropdown: `claude-opus-5` (default), `claude-sonnet-5`, `claude-haiku-4-5`.
Use those exact IDs. Never append date suffixes.

Send **incrementally**: previous state + only transcript since the last call. Never resend the
whole window. Skip the call entirely if fewer than ~15 new words accumulated — silence should cost
nothing.

### RESPONSE_SCHEMA

```json
{
  "type": "object",
  "additionalProperties": false,
  "required": ["topic","topic_changed","bullets","user_addressed","talking_points","decisions","actions","glossary"],
  "properties": {
    "topic": { "type": "string" },
    "topic_changed": { "type": "boolean" },
    "bullets": { "type": "array", "items": { "type": "string" } },
    "user_addressed": {
      "type": ["object","null"],
      "additionalProperties": false,
      "required": ["asked_by","question","kind"],
      "properties": {
        "asked_by": { "type": "string" },
        "question": { "type": "string" },
        "kind": { "type": "string", "enum": ["direct_question","mentioned","assigned"] }
      }
    },
    "talking_points": { "type": "array", "items": { "type": "string" } },
    "decisions": { "type": "array", "items": { "type": "string" } },
    "actions": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["text","owner","mine"],
        "properties": {
          "text": { "type": "string" },
          "owner": { "type": "string" },
          "mine": { "type": "boolean" }
        }
      }
    },
    "glossary": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "required": ["term","definition"],
        "properties": { "term": { "type": "string" }, "definition": { "type": "string" } }
      }
    }
  }
}
```

### SYSTEM_PROMPT

Use this text, with the user profile and pre-meeting context interpolated in:

```
You help someone who has lost focus in a live meeting re-enter the conversation quickly.

You receive your own previous state, plus a new chunk of rough speech-to-text transcript.
The transcript is machine-generated: expect misheard words, missing punctuation, no speaker
labels, and dropped words. Infer intent. Never quote transcription errors back verbatim.

USER PROFILE:
{profile}

MEETING CONTEXT (agenda, attendees, projects — may be empty):
{context}

Rules:
- topic: 3-8 words naming what is being discussed at the END of the chunk. Not a summary of the
  whole chunk — what they are on RIGHT NOW.
- topic_changed: true only if meaningfully different from the previous topic. Ignore drift.
- bullets: 3-5 bullets covering the recent window, most recent first, each under 20 words.
  Plain language. No preamble, no "the team discussed".
- user_addressed: non-null ONLY if the user was directly asked something, assigned something, or
  named in a way that expects a response. Someone merely saying their name in passing is
  "mentioned". Be conservative — a false alarm is worse than a miss here.
- talking_points: only when user_addressed is non-null. 2-3 short things the user could actually
  say, grounded in what was discussed. Not generic filler.
- decisions / actions / glossary: only NEW items from this chunk. Never repeat items you have
  already reported. Empty arrays are correct and expected most of the time.
- glossary: only genuinely non-obvious acronyms, jargon, or proper nouns.
- If the chunk is silence, filler, or crosstalk with no substance: repeat the previous topic
  unchanged and return empty arrays.
```

## UI

Dark, high contrast, large type. Three zones, in visual priority order:

1. **NOW** — the current topic in very large text (~48px+), unmissable. Plus a subtle "updated 12s
   ago" and a flash animation when `topic_changed` fires. This is the two-second-glance element.
2. **ALERT** — when `user_addressed` is set, or the local regex fires: a loud banner naming who
   asked, what they asked, how long ago, and the talking points. Optional soft audio chime
   (default off). This must be visually dominant when active and absent when not.
3. **CONTEXT** — the last-10-minutes bullets, plus collapsible panels for Decisions, My Action
   Items, Glossary, and Topic Timeline.

Persistent slim status bar: engine in use, capture source, connection state, last-call latency,
running session cost estimate.

## Features

Implement all of these:

- **Rolling buffer** — timestamped fragments, prune older than the window. Window configurable
  5–20 min, default 10.
- **Catch me up** — button and `C` hotkey. One-off richer call over the *entire* buffer, not the
  incremental delta. Renders a longer narrative summary in a modal.
- **While you were away** — track `document.visibilityState`. When the tab regains focus, show
  "3 topics changed while you were away" with the topics I missed.
- **Local name alert (fast loop)** — configurable list of name variants. Regex over incoming
  fragments, plus patterns like "what do you think", "any thoughts", "over to you", "can you".
  Fires instantly, independent of the API.
- **Decisions & my action items** — accumulate for the whole meeting, not just the window.
  Action items assigned to me are visually distinct.
- **Topic timeline** — list of topics with start times, so I can see the arc of the meeting.
- **Glossary** — accumulating acronym/jargon decoder.
- **Export** — download the full meeting as Markdown (transcript + topics + decisions + actions)
  via a `Blob` + object URL. Must work from `file://`.
- **Cost meter** — accumulate `usage.input_tokens` / `usage.output_tokens` from responses, price
  per the selected model, show running session total.
- **Local-only mode** — a prominent toggle that hard-disables every outbound call. The app degrades
  to raw transcript plus local keyword-frequency topic detection. Must be a single choke-point
  function, so the network boundary is auditable at a glance.
- **Discreet mode** — compact low-key layout, and an `Esc` panic-hide that instantly blanks the
  screen to something innocuous. Second `Esc` restores.
- **Settings panel** — API key, model, capture source, STT engine, window size, call interval,
  name variants, user profile (free text), pre-meeting context (free text, for agenda/attendees).
- **Persistence** — all settings plus in-flight meeting state to `localStorage`, auto-restored on
  reload so an accidental refresh mid-meeting is not fatal.
- **Resilience** — API failures retry with exponential backoff, never kill the capture loop, and
  surface as a status-bar state rather than an alert box.

## Non-goals — do not build these

Speaker diarization. Meeting-platform integrations or bots. Server components. Auth. Cloud storage.
Multi-user anything. A build step or package manager.

## Constraints

- One file. Inline all CSS and JS. The only permitted network calls at runtime are the Anthropic API
  and the optional transformers.js CDN.
- API key lives in `localStorage`, entered at runtime, never hardcoded. Warn in the UI that a
  `file://` page offers no real secret protection and that a spend-limited personal key is advised.
- Must run from `file://` in Chrome/Edge on Windows. Note in a comment that Chrome on macOS cannot
  capture system audio at all (tab audio only).
- Plain vanilla JS. No frameworks, no bundler, no CDN UI libraries.
- Comment the non-obvious parts — especially the `onend` restart loop, the 16 kHz resampling, and
  the incremental-summarization state machine.

## Priority, if you must cut

Core, never cut: capture → STT → buffer → Claude cycle → NOW zone → ALERT zone → settings →
local-only toggle.
Cut first, in this order: glossary, topic timeline, while-you-were-away, cost meter, export.

## Acceptance checklist

Before you finish, verify each:

1. Web Speech engine survives 5+ minutes of intermittent silence without dying (`onend` restart).
2. Selecting system audio disables the Web Speech engine with a visible explanation.
3. A display-capture stream with no audio track produces a clear, specific error.
4. Silence produces no API calls.
5. Local-only mode makes zero network requests — verifiable in the Network tab.
6. `Esc` hides the screen instantly.
7. Reloading mid-meeting restores state.
8. A malformed or failed API response does not stop audio capture.
9. The current topic is readable from arm's length, at a glance.
