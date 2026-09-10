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

## The parts that took real work

**Turn-taking on phone calls.** Two people talking over each other through a translator is a mess, so the call works like a hands-free walkie-talkie. The backend decides who has the floor, prepares the translated speech while the person is still talking, and plays it once they stop. The gap between finishing a sentence and hearing it translated is around two seconds.

**Knowing where the time goes.** Audio is timestamped through the whole pipeline, and every call is recorded with per-turn timings and quality flags that the dashboard can show. That instrumentation is how I found out that a latency drift I'd been chasing for weeks was a carrier-side media problem, later confirmed by their engineering team, and not something in my code.

**Captions on iOS and Android.** Neither platform makes this easy. Android needs an explicit screen-capture consent flow, a foreground service, and an overlay window drawn over other apps. iOS does not allow either, so the captions run in a native Picture in Picture window instead, which keeps them visible while the user is inside the video app.

**Subtitles that don't flicker.** Live speech recognition constantly revises what it just heard. The backend splits its output into a settled part and a provisional tail, so the app can build a stable caption history instead of redrawing text that keeps changing.

**Security.** Signature validation on the telephony webhooks, authentication on the media connections, phone numbers kept out of logs, and speech provider keys that never leave the backend.

## Status

Working end to end: phone calls, in-place translation, live captions on both platforms, and the admin dashboard with call history, quality flags, and per-call latency breakdowns.

Still to do: user accounts and credits, offline video dubbing (YouTube and local files), and a proper performance pass on physical devices before any public release.

## Screenshots

Coming soon.

## Contact

Shayan Sharif Nia · ssharyfnia9919@gmail.com
