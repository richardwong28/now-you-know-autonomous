# Now You Know — Autonomous Video Pipeline

An end-to-end automated video production and publishing system. A single structured text submission via Telegram is transformed into a fully rendered, branded YouTube Short — with burned-in subtitles, a custom thumbnail, and a trademark outro — without manual intervention.

## What it does

1. Receives a structured content submission (segment / title / script) via an access-controlled Telegram bot
2. Validates and sanitizes input against a fixed content-category enum in n8n
3. Generates narration audio via self-hosted Edge TTS
4. Extracts per-sentence visual keywords via LLM prompt (subject-identification before query generation)
5. Sources matching stock footage from Pexels API
6. Composites everything with FFmpeg into a vertical 1080×1920 video:
   - Burned-in subtitles
   - Persistent color-coded segment badge per content category
   - Auto-generated custom thumbnail (frame extraction + PIL text compositing)
   - Branded trademark outro (blurred final frame + TTS voiceover, concatenated via re-encoded filter_complex pass)
7. Uploads finished video to YouTube via OAuth2
8. Sets custom thumbnail in the same automated pass
9. Sends upload confirmation back to Telegram

## Tech stack

| Component | Tool |
|---|---|
| Orchestration | n8n (self-hosted, Docker) + Telegram Bot API |
| AI / LLM | DeepSeek via OpenRouter API |
| Media generation | Python, FFmpeg (filter_complex, drawtext, gblur, concat), Pillow (PIL) |
| Text-to-speech | Self-hosted Edge TTS wrapper |
| Stock footage | Pexels API |
| Publishing | YouTube Data API v3 (upload, OAuth2, thumbnails.set) |
| Infrastructure | Ubuntu 24.04 VPS, Docker Compose, cron, Python venv |
| Version control | Git, GitHub |

## Architecture

```
Telegram Bot (access-controlled, chat-ID allowlist)
    └── n8n intake node
        └── Input validation & sanitization (content-category enum)
            └── Python render script
                ├── TTS narration generation (Edge TTS)
                ├── LLM keyword extraction per sentence (DeepSeek)
                ├── Stock footage sourcing (Pexels API)
                ├── FFmpeg video composition
                │   ├── Burned-in subtitles
                │   ├── Color-coded segment badge
                │   ├── Outro segment (blurred frame + TTS)
                │   └── Custom thumbnail generation (PIL)
                └── YouTube upload (OAuth2)
                    └── thumbnails.set
                        └── Telegram confirmation
```

## Project structure

```
├── scripts/        # Render and upload logic (not public)
├── workflows/      # n8n workflow exports (not public)
├── assets/         # Fonts, outro assets
├── output/         # Rendered videos (gitignored)
├── logs/           # Runtime logs (gitignored)
├── .env.example
├── .gitignore
├── docker-compose.example.yml
└── requirements.txt
```

## Setup

```bash
cp .env.example .env
# Fill in your API keys

docker compose up -d
source venv/bin/activate
pip install -r requirements.txt
```

## Environment variables

```
OPENROUTER_API_KEY=
PEXELS_API_KEY=
TELEGRAM_BOT_TOKEN=
TELEGRAM_ALLOWED_CHAT_ID=
YOUTUBE_CLIENT_ID=
YOUTUBE_CLIENT_SECRET=
EDGE_TTS_HOST=
```

## Key technical problems solved

**Shell injection prevention**
User-submitted free text was originally interpolated into shell commands. Redesigned the flow to write payloads to disk and pass only a generated file path as the CLI argument — keeping arbitrary user input out of the shell entirely.

**OAuth2 scope debugging on a headless server**
Diagnosed that YouTube's `thumbnails.set` endpoint requires the broader `youtube` scope, not `youtube.upload`. Built the authorization flow using `InstalledAppFlow` with a local callback server tunneled through SSH, since the VPS has no browser.

**Platform-side constraint vs. code bug**
Diagnosed a YouTube API 403 on thumbnail upload down to channel phone-verification status — a platform-level gate independent of OAuth scope or code correctness. Correctly distinguished this from an implementation bug during debugging.

**Docker volume and permission debugging**
Diagnosed an n8n Code node failing to read a host directory due to a missing bind mount, followed by an `EACCES` error from the container running as a non-root UID against a root-owned host directory — resolved with a narrowly scoped permission fix.

**n8n data-flow bug**
Diagnosed that Execute Command nodes replace the item's JSON payload with only stdout/stderr/exitCode, silently dropping upstream fields. Fixed by switching to explicit cross-node field references throughout the workflow.

**Prompt engineering for a measurable failure mode**
Iterated on the stock-footage keyword-extraction prompt after observing that abstract sentences produced weak Pexels matches. Added an explicit subject-identification step before query generation — significantly improving footage relevance.

**Secrets hygiene and access control**
Refactored hardcoded API keys into environment-variable configuration with a fail-fast startup check. Added Telegram chat-ID allowlist at the intake node to prevent unauthorized use of a publicly reachable bot endpoint that would otherwise burn paid API quota.
