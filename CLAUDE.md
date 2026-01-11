# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Changelog Master is a changelog tracking and notification system powered by AI. It monitors markdown changelogs from any source (GitHub projects, etc.), provides AI-powered analysis using Gemini 3 Flash, generates audio summaries with Gemini TTS, and sends email notifications via Resend.

## Development Commands

```bash
# Start both frontend and backend (recommended for development)
npm run dev:all

# Start only frontend (http://localhost:5173)
npm run dev

# Start only backend (http://localhost:3001)
npm run dev:server

# Build for production
npm run build

# Preview production build
npm run preview
```

## Architecture

### Two-Process Architecture
- **Frontend**: React 19 + Vite on port 5173 (proxies `/api` to backend)
- **Backend**: Express.js on port 3001 with SQLite database

The frontend cannot function independently - it requires the backend for audio caching, analysis storage, chat persistence, and email functionality.

### Key Data Flows

**Changelog Analysis Flow:**
1. `useChangelog` hook fetches markdown from source URL
2. `changelogService.parseChangelog()` extracts versions and items
3. `geminiService.analyzeChangelog()` calls Gemini API for AI analysis
4. `cacheService` stores analysis in SQLite via backend API

**Audio Generation Flow:**
1. `useAudio` hook checks SQLite cache via `/api/audio/:hash/:voice`
2. If not cached, `ttsService` calls Gemini TTS API
3. PCM audio converted to WAV via `pcmToWav()` function
4. Audio cached to SQLite and played via HTML5 Audio

**Monitoring Flow (Backend):**
1. `node-cron` scheduler triggers `checkForNewChangelog()`
2. Iterates all active sources from `changelog_sources` table
3. Compares detected version against `changelog_history`
4. If new version, triggers analysis, TTS generation, and email notification

### Frontend Structure (src/)

```
hooks/
  useChangelog.ts  - Source selection, fetching, parsing, analysis orchestration
  useAudio.ts      - TTS generation, playback controls, caching
  useTheme.ts      - Dark/light mode persistence

services/
  changelogService.ts  - Fetch/parse markdown, source API calls
  geminiService.ts     - Frontend AI analysis API calls
  ttsService.ts        - Frontend TTS generation
  cacheService.ts      - IndexedDB + backend API for persistence
  emailService.ts      - Email sending via backend

components/
  App.tsx            - Main orchestrator, state management
  Header.tsx         - Source selector, toolbar actions
  MattersView.tsx    - AI analysis display with category sections
  ChangelogView.tsx  - Raw markdown display by version
  AudioPlayer.tsx    - Seekable player with speed/voice controls
  ChatPanel.tsx      - AI assistant for changelog Q&A
  SettingsPanel.tsx  - Notification and monitoring settings
  SourcesPanel.tsx   - CRUD for changelog sources
```

### Backend Structure (server/index.ts)

Single file containing:
- SQLite schema initialization and migrations
- Cron job management (`startMonitoring`, `stopMonitoring`)
- Changelog parsing (`parseLatestVersion`)
- AI analysis (`analyzeChangelog`) and TTS (`generateTTSAudio`)
- Email generation (`generateEmailHtml`, `sendEmailWithAttachment`)
- REST API routes for all features

### Database (data/audio.db)

SQLite tables:
- `changelog_sources` - Monitored changelog URLs
- `changelog_history` - Detected versions per source
- `analysis_cache` - Cached AI analyses by version
- `audio_cache` - Cached TTS audio blobs by hash+voice
- `chat_conversations` / `chat_messages` - Chat persistence
- `settings` - User preferences

## Environment Variables

```env
# Required
VITE_GEMINI_API_KEY=your-gemini-api-key

# Optional - Email notifications
RESEND_API_KEY=re_xxxxxxxxxxxx
NOTIFY_EMAIL=you@example.com

# Optional - Defaults
VITE_CHANGELOG_CACHE_DURATION=3600000
VITE_VOICE_PREFERENCE=Charon
```

## API Endpoints

The backend exposes REST endpoints under `/api/`:
- `/api/sources/*` - CRUD for changelog sources
- `/api/analysis/:version` - Cached AI analysis
- `/api/audio/:hash/:voice` - Cached TTS audio
- `/api/chat` - Gemini-powered Q&A
- `/api/conversations/*` - Chat history persistence
- `/api/monitor/*` - Cron status and manual triggers
- `/api/settings/*` - User preferences
- `/api/send-*` - Email endpoints

## Key Patterns

**Gemini API Integration:**
- Analysis uses `gemini-3-flash-preview` with `responseMimeType: 'application/json'`
- TTS uses `gemini-2.5-flash-preview-tts` with `responseModalities: ['AUDIO']`
- PCM audio from TTS must be converted to WAV format for browser playback

**Version Detection:**
- Regex pattern: `/^##\s+\[?(\d+\.\d+\.\d+(?:-[a-zA-Z0-9.]+)?)\]?/`
- Captures semver with optional prerelease tags

**Caching Strategy:**
- Audio keyed by SHA-256 hash of text + voice name
- Analysis keyed by version string
- Frontend checks backend cache before generating new content
