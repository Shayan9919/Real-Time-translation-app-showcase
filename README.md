# Zabanak (زبانک)

Real-time English / Farsi translation for phone calls, face-to-face conversation, and video.

This is a personal project I have been building since early 2026. The source code lives in private repositories (Flutter app, Python backend, admin dashboard). This repo is only an overview so people can see what the project is and how it is put together. I'm happy to walk through the actual code in an interview.

## What it does

**Phone call translation.** Two people call each other through a normal phone number, one speaking English and the other Farsi. Each person hears the other side translated and spoken back in their own language. No app is needed on the other end, just a phone.

**In-place translation.** Two people in the same room share one phone. The screen is split into an English half and a Farsi half, and each side shows a live transcript of what the other person is saying.

**Live video captions.** Play a video (YouTube, a lecture, a voice message) and get Farsi subtitles on top of it in real time. On Android this captures the playback audio directly and shows a draggable subtitle strip over any app. On iOS it listens through the microphone and shows the captions in a Picture in Picture window so the user can switch to the video app.

The app is fully bilingual. The whole UI works in English (LTR) and Farsi (RTL), in both dark and light mode.

## How it is built

```
  Flutter app (iOS / Android)        Phone network
          │                               │
          │ WebSocket (PCM audio)         │ Telnyx / Twilio media streams
          ▼                               ▼
  ┌─────────────────────────────────────────────────┐
  │           FastAPI backend (Python 3.11)          │
  │  session manager · turn-taking · audio mixer     │
  │  latency tracking · metrics recorder             │
  └───────────────┬──────────────────┬──────────────┘
                  │                  │
                  ▼                  ▼
        Soniox real-time         Soniox TTS
        STT + translation        (speech output)

  Next.js admin dashboard  ──►  backend /admin API
```

| Part | Stack |
|---|---|
| Mobile app | Flutter / Dart, with native Swift (iOS Picture in Picture caption renderer) and Kotlin (Android audio capture foreground service and overlay) |
| Backend | Python 3.11, FastAPI, asyncio, WebSockets, Docker |
| Speech | Soniox real-time speech-to-text with built-in translation, Soniox TTS (Azure TTS was used earlier) |
| Telephony | Telnyx and Twilio Programmable Voice with bidirectional media streams. Both are integrated behind the same session and pipeline code. |
| Dashboard | Next.js, React, Tailwind, Recharts |
| Storage | SQLite for call metrics and transcripts |

Rough size: about 10k lines of Dart in the app and 7.5k lines of Python in the backend, plus the native iOS/Android pieces and the dashboard.

## The parts that took real work

**Turn-taking on phone calls.** Two people talking over each other through a translator is a mess, so the call works like a hands-free walkie-talkie. The backend does voice activity detection on each leg, gives the floor to one speaker at a time, buffers the translation as it streams in, and starts synthesising speech in the background as soon as the sentence is stable. Playback begins after a short silence window. The gap from "stopped talking" to "hearing the translation" is around two seconds.

**Keeping latency honest.** Every audio chunk is timestamped through the whole pipeline, and each call records per-turn "speech to playback" timings and a realtime lag timeseries per leg. The dashboard flags calls where the audio stream starved. This is how I found out that a latency drift I'd been chasing for weeks was actually an inbound media quality problem on the carrier side, which their engineering team later confirmed, and not something in my code.

**Audio mixing on the outbound leg.** Forwarding the speaker's raw voice and the translated TTS to the listener at the same time originally produced overlapping audio. I replaced that with a small mixer that owns the outbound stream and schedules raw audio and TTS cleanly.

**iOS captions.** iOS does not let a normal app capture another app's audio or draw over it. The workaround was a native Picture in Picture window driven from Flutter through a platform channel, with the caption text rendered natively so it stays readable while the user is inside the video app.

**Android captures.** Playback capture on Android 10+ needs a MediaProjection consent flow, a foreground service, and an overlay window. The subtitle strip crossfades between complete cues, can be dragged anywhere in the safe area, remembers its position across rotation, and hides itself when speech stops.

**Caption stream protocol.** The backend separates translation text into a stable part that never changes and a provisional tail that can be replaced. The app builds its history from the stable text only, so subtitles don't flicker or duplicate when the recogniser revises itself. Shutdown is also handled properly: the backend waits for the recogniser's final tokens before closing the socket so the last sentence is never lost.

**Security basics.** Webhook signature validation on all telephony endpoints, per-connection auth on the media WebSockets, phone numbers redacted from logs, a fail-closed token for the captions endpoint in production, and the speech provider key never leaves the backend.

**Small things that turned out to be interesting.** A "stutter" on view transitions that instrumentation said was fine turned out to be a 28% brightness dip caused by stacked semi-transparent layers during the crossfade, not a dropped frame. Found it by screen recording and stepping through frames.

## Status

Working end to end: phone calls, in-place translation, live captions on both platforms, and the admin dashboard with call history, quality flags, and per-call latency breakdowns.

Still to do: user accounts and credits, offline video dubbing (YouTube and local files), and a proper performance pass on physical devices before any public release.

## Screenshots

Coming soon.

## Contact

Shayan Sharif Nia · ssharyfnia9919@gmail.com
