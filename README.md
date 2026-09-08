# Shellingford — Bisaya Voice Journal Pipeline

An end-to-end personal system that turns a spoken voice journal in Bisaya/Cebuano into a fully translated, AI-tagged entry in my Obsidian vault — spanning a native iOS app, a cloud automation platform, a home server, and a private network bridge between them.

## Overview

Shellingford started as a simple question: could I keep a daily journal in my first language, Bisaya/Cebuano, without the friction of writing in English? It grew into a full pipeline: I speak into a native iPhone app throughout the day; when I upload, the audio is transcribed, translated to English, automatically tagged with metadata (mood, people, companies, events) against a fixed schema, and pushed as a formatted Markdown note directly into my Obsidian journal — which then syncs to every device I own. A Discord notification confirms when it's done.

## Problem / Motivation

Two constraints shaped this project:

1. **Apple's Voice Memos has no public API.** Unlike Photos, there's no way for a third-party app or script to programmatically read its recordings — so any automation had to start with a purpose-built recording app instead.
2. **My home automation server has no public IP.** It sits behind home NAT, and I didn't want to expose it to the internet just to receive data from a cloud workflow. Solving this properly (rather than punching a hole in my router) became one of the more interesting technical problems in the project.

## Tech Stack

**iOS App**
- **Swift** and **SwiftUI** — native app, declarative UI
- **AVFoundation** (`AVAudioSession`, `AVAudioRecorder`, `AVAudioPlayer`) — recording and playback
- **URLSession** (async/await, multipart form-data) — batches a day's recordings into one upload

**Automation / Orchestration**
- **n8n** (self-hosted on **Fly.io**) — the workflow engine tying everything together: webhook ingestion, OpenAI calls, branching logic, notifications
- **JavaScript** (n8n Code nodes) — splits multipart audio into individual items, joins per-file transcripts into one timestamped text block, base64-encodes payloads for safe transport over SSH
- **OpenAI API** — `gpt-4o-transcribe` for speech-to-text, a chat completion model for Bisaya→English translation, and a second structured-output call that classifies each entry against a fixed metadata schema (tags, mood, people, companies, events, a 1–5 rating)

**Home Server / Persistence**
- **Ubuntu server** (homelab, 24/7) — receives the finished entry over SSH and writes it into a local git-tracked Obsidian vault mirror
- **Python** — the receiving script: builds the correct year/month file path, writes the Markdown file with YAML frontmatter, and safely commits + pushes to GitHub (with automatic rebase-and-retry if the remote has moved on)
- **WireGuard** — bridges the Ubuntu server into Fly.io's private network (6PN), so n8n can reach it over SSH with zero public ports exposed on my home router
- **GitHub** — the private repo acting as the sync bridge between the Ubuntu server and my Mac
- **Discord webhooks** — a completion notification with the final formatted entry, once the whole pipeline finishes

**Sync back to Obsidian**
- **Obsidian** (Mac + iOS), vault stored in **iCloud Drive**
- A local Python sync script (reused from an earlier RAG-pipeline project) pulls the latest commits into the vault, which iCloud then propagates automatically to iPhone/other devices — no code runs on the phone itself

## Architecture / Flow

```
iPhone app (record, consolidate by day)
        │  multipart upload
        ▼
n8n webhook (Fly.io)
        │
        ├─ Split into per-file items (JS)
        ├─ OpenAI: transcribe each file (Bisaya text)
        ├─ Join transcripts with real timestamps (JS)
        ├─ OpenAI: translate to English
        ├─ OpenAI: classify against metadata schema (JSON)
        ├─ Build final Markdown (frontmatter + entry) (JS)
        │
        ├──────────────► Discord notification (HTTP)
        │
        └─ Base64-encode payload (JS)
                │  SSH, over a WireGuard-bridged private network
                ▼
        Ubuntu server: Python script
                │  writes file, git commit, git push
                ▼
        GitHub (private repo)
                │  pulled manually / on demand
                ▼
        Mac: local Obsidian vault (iCloud Drive)
                │  iCloud sync (automatic)
                ▼
        iPhone: Obsidian shows the new entry
```

## Interesting Problems Solved

A few things that came up along the way, which I think show real engineering rather than just wiring APIs together:

- **Timezone bugs across three systems.** The app, n8n, and the server all defaulted to UTC in different spots, which quietly shifted both the calendar day and the displayed time of each entry. Fixed by forcing explicit timezone handling (device-local on iOS, explicit `Pacific/Auckland` in JS) at each boundary rather than trusting defaults.
- **JSON safety with real, messy text.** Journal entries contain line breaks, quotes, and apostrophes — all of which can silently break hand-built JSON strings. Solved with `JSON.stringify()` at the point of construction rather than manual escaping.
- **Bridging a private home server into a cloud workflow, without exposing it.** Rather than port-forwarding SSH to the public internet, the Ubuntu server joins Fly.io's private network directly via a WireGuard peer connection — meaning the cloud-hosted automation can reach it over an encrypted tunnel that the home server itself initiates outbound, with no inbound firewall rule needed at all.
- **SSH key format incompatibility.** n8n's SSH node couldn't parse the modern OpenSSH key format `ssh-keygen` generates by default for ed25519 keys — resolved by generating an RSA key in the older PEM format instead.
- **git and iCloud actively conflict.** Running git commands directly inside an iCloud-synced folder produces intermittent `mmap failed: Resource deadlock avoided` errors, because git's low-level file-mapping collides with iCloud's background sync daemon touching the same files. Worked around by keeping the automated sync as a manual/on-demand step rather than a tight background loop, avoiding the collision window.

## What's Next

- Apple Watch companion target, using WatchConnectivity to record from the wrist
- Automating the final Mac-side pull step in a way that's resilient to the iCloud/git conflict above
- Surfacing transcription/translation history back inside the iOS app for review

## Background

This project followed an earlier Python-based MVP (Mac-only, terminal-driven) that proved out the OpenAI transcription + translation pipeline for Bisaya/Cebuano before any of the app or automation work began.
