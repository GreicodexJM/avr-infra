# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**avr-infra** is a Docker Compose-based infrastructure project for deploying the **Agent Voice Response (AVR)** system — a modular voice bot that integrates with Asterisk PBX to handle VoIP calls using AI services. The core pipeline is: **Asterisk → avr-core → ASR → LLM → TTS → Asterisk**.

## Commands

```bash
npm run dc:up    # Start default docker-compose services
npm run dc:down  # Stop services
npm run dc:pull  # Pull latest images
```

To start a specific deployment profile:
```bash
docker-compose -f docker-compose-openai.yml up -d
docker-compose -f docker-compose-app.yml up -d   # Full web GUI deployment
```

## Architecture

### Pipeline Modes

**Modular pipeline (ASR + LLM + TTS as separate services):**
```
Asterisk → avr-core (5001) → avr-asr (6010) → avr-llm (6002) → avr-tts (6011) → avr-core → Asterisk
```

**Speech-to-Speech unified pipeline:**
```
Asterisk → avr-core (5001) → avr-sts (WebSocket) → avr-core → Asterisk
```

### Services

| Service | Port | Role |
|---------|------|------|
| avr-core | 5001 (AudioSocket), 6001 (HTTP) | Central orchestrator |
| avr-asterisk | 5060 (SIP), 8088 (ARI), 5038 (AMI) | PBX / call routing |
| avr-ami | 6006 | AMI wrapper for Asterisk control |
| avr-phone | 8080 | Browser-based SIP phone for testing |
| avr-app-backend | 3001 | REST API for web dashboard |
| avr-app-frontend | 3000 | Next.js admin dashboard |

All services share Docker bridge network `avr` (172.20.0.0/24).

### Supported Providers

- **ASR:** Deepgram, Google, Vosk (local), OpenAI, Gemini, HumeAI, Sarvam, Speechmatics
- **LLM:** OpenAI, Anthropic, OpenRouter, N8N, Gemini, HumeAI, Sarvam, Speechmatics
- **TTS:** Deepgram, Google, ElevenLabs, Kokoro (local), Gemini, HumeAI, Sarvam, Speechmatics
- **STS (unified):** OpenAI Realtime, Ultravox, Deepgram, ElevenLabs, Gemini, HumeAI, Sarvam, Speechmatics

### Docker Compose Files

Each `docker-compose-<provider>.yml` file is a self-contained headless deployment for a specific provider combination. `docker-compose-app.yml` adds Traefik, MySQL, and the web GUI. `docker-compose-local.yml` runs fully offline (Vosk + Ollama + Kokoro).

## Configuration

Copy `.env.example` to `.env` and fill in relevant API keys and service URLs before running. Key variables:

- `ASR_URL`, `LLM_URL`, `TTS_URL`, `STS_URL` — service endpoints consumed by avr-core
- `DEEPGRAM_API_KEY`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc. — provider credentials
- `AMBIENT_NOISE_FILE`, `AMBIENT_NOISE_LEVEL` — optional background audio injection
- `HTTP_PORT` — enables HTTP webhook interface on avr-core (extension 5003)

### Asterisk Configuration (`asterisk/conf/`)

- `extensions.conf` — Dialplan: ext 5001 (AudioSocket to avr-core), 5002 (alt host routing), 5003 (HTTP webhook)
- `pjsip.conf` — SIP/WebRTC endpoints; user 1000 (SIP) and 2000 (WebRTC) pre-configured
- `manager.conf` — AMI credentials (user: avr, pass: avr)
- `ari.conf` — ARI credentials for REST interface

### Ambient Sounds

Audio files in `ambient_sounds/` must be **8kHz, 16-bit, mono, raw PCM** format. See `ambient_sounds/README.md` for conversion commands.
