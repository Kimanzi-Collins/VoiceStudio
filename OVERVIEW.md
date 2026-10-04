# VoiceStudio — Comprehensive Project Overview

> **Version 0.5.7** · AGPL-3.0 · [debpalash/VoiceStudio](https://github.com/debpalash/VoiceStudio)

VoiceStudio is an open-source, local-first ElevenLabs alternative — a desktop application for **voice cloning**, **voice design**, **video dubbing**, **real-time dictation**, **transcription**, and **audiobook creation** across **646 languages**. It runs entirely on the user's machine with no API keys, no accounts, and no cloud dependencies.

---

## Table of Contents

- [High-Level Architecture](#high-level-architecture)
- [Repository Structure](#repository-structure)
- [Tech Stack](#tech-stack)
- [Backend (Python / FastAPI)](#backend-python--fastapi)
  - [Entry Points](#entry-points)
  - [API Surface](#api-surface)
  - [Engine Registry](#engine-registry)
  - [ML & Audio Processing](#ml--audio-processing)
  - [Database & Storage](#database--storage)
  - [Worker / Task Architecture](#worker--task-architecture)
- [Frontend (Electron / React)](#frontend-electron--react)
  - [Main Process](#main-process)
  - [Preload Bridge](#preload-bridge)
  - [Renderer](#renderer)
  - [State Management](#state-management)
  - [Routing & Feature Pages](#routing--feature-pages)
  - [Internationalization](#internationalization)
- [MCP Server (AI Agent Integration)](#mcp-server-ai-agent-integration)
- [Native Bridge (Rust)](#native-bridge-rust)
- [Core Features](#core-features)
- [Supported Engines](#supported-engines)
  - [Text-to-Speech (TTS)](#text-to-speech-tts)
  - [Speech Recognition (ASR)](#speech-recognition-asr)
  - [Other Engines](#other-engines)
- [Model Catalog](#model-catalog)
- [Setup & Installation](#setup--installation)
  - [One-Command Install](#one-command-install)
  - [From Source (Development)](#from-source-development)
  - [Docker / Podman](#docker--podman)
  - [Hardware Support](#hardware-support)
- [Configuration](#configuration)
- [Build & Packaging](#build--packaging)
- [Testing Strategy](#testing-strategy)
- [CI/CD Pipelines](#cicd-pipelines)
- [Deployment Channels](#deployment-channels)
- [Licensing & Commercial (Pro)](#licensing--commercial-pro)
- [Telemetry & Privacy](#telemetry--privacy)
- [Governance & Contribution](#governance--contribution)
- [Documentation Map](#documentation-map)

---

## High-Level Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        VoiceStudio Desktop                         │
│  ┌────────────────┐  IPC (contextBridge)  ┌─────────────────────┐  │
│  │  Electron Main │◄────────────────────►│  Electron Renderer   │  │
│  │  (Node.js)     │                       │  (React 19 + Vite)   │  │
│  │  ─ Backend     │                       │  ─ TanStack Router   │  │
│  │    Supervisor   │                       │  ─ Zustand + RTK     │  │
│  │  ─ Auto-Update │                       │  ─ shadcn/ui         │  │
│  │  ─ Tray, Proto │                       │  ─ Tailwind CSS v4   │  │
│  └───────┬────────┘                       └──────────┬──────────┘  │
│          │ spawn / supervise                         │ HTTP / WS   │
│          ▼                                           ▼             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │              FastAPI Backend (Python 3.11+)                   │  │
│  │  ─ 40+ routers  ─ 98 services  ─ Engine adapters             │  │
│  │  ─ SQLite (Alembic)  ─ VRAM-aware GPU pool                   │  │
│  │  ─ Task queue (SSE/WS)  ─ MCP Server  ─ OpenAI compat API   │  │
│  └───────────────────────┬───────────────────────────────────────┘  │
│                          │                                          │
│          ┌───────────────┼───────────────┐                         │
│          ▼               ▼               ▼                         │
│   ┌────────────┐  ┌────────────┐  ┌──────────────┐                │
│   │ OmniVoice  │  │ WhisperX   │  │ Demucs /     │                │
│   │ TTS Engine │  │ ASR Engine │  │ AudioSeal    │                │
│   │ (PyTorch)  │  │ (wav2vec2) │  │ DSP Pipeline │                │
│   └────────────┘  └────────────┘  └──────────────┘                │
│                                                                     │
│  ┌──────────────────┐  ┌──────────────────────────────────────┐    │
│  │ Rust Native      │  │ Remote GPU Workers (gRPC + mTLS)     │    │
│  │ Desktop Bridge   │  │ ─ Distributed task dispatch           │    │
│  └──────────────────┘  └──────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────────────────┘
```

---

## Repository Structure

```
VoiceStudio/
├── backend/                  # Python FastAPI server
│   ├── main.py               #   App entry point, lifecycle, middleware
│   ├── mcp_server.py         #   Model Context Protocol server
│   ├── api/                  #   Schemas, deps, HTTP client
│   │   └── routers/          #     40+ modular route handlers
│   │       └── setup/        #     First-run wizard & model downloads
│   ├── config/               #   models.yaml (HF model catalog)
│   ├── core/                 #   Config, DB, auth, CSRF, tasks, GPU budget
│   ├── engines/              #   Engine wrappers (CosyVoice, GGUF, etc.)
│   ├── migrations/           #   Alembic SQLite migrations
│   ├── services/             #   Business logic & ML runtimes
│   │   └── telephony/        #     Twilio call integration
│   ├── utils/                #   FS containment, atomic I/O, HF helpers
│   └── worker/               #   Remote GPU workers (gRPC + protobuf)
├── electron/                 # Desktop app (sole UI)
│   ├── src/main/             #   Main process (backend supervisor, IPC, updater)
│   ├── src/preload/          #   Secure contextBridge
│   ├── src/renderer/         #   React app (features, routes, components)
│   │   └── src/i18n/locales/ #     21 locale JSON files
│   └── src/shared/           #   Shared components, stores, API clients
├── omnivoice/                # Core OmniVoice TTS model library
├── omnivoice-gallery/        # Community voice gallery (submodule)
├── native/desktop-bridge/    # Rust native helper
├── deploy/                   # Dockerfile, docker-compose.yml
├── docs/                     # 100+ documentation files
│   ├── adr/                  #   Architecture Decision Records
│   ├── engines/              #   Per-engine guides (30 engines)
│   ├── install/              #   Platform-specific install guides
│   ├── specs/                #   Feature specs & roadmap
│   └── features.yaml         #   Canonical feature catalog
├── scripts/                  # Build, setup, and dev scripts
├── tests/                    # Python integration & contract tests
├── examples/                 # API client examples
├── notebooks/                # Jupyter exploration notebooks
├── package.json              # Monorepo root (SINGLE VERSION SOURCE OF TRUTH)
├── pyproject.toml            # Python build config (Hatchling)
├── bun.lock                  # JS dependency lockfile
├── uv.lock                   # Python dependency lockfile
├── CHANGELOG.md              # Release history
├── CLAUDE.md                 # AI agent constitution
├── AGENTS.md                 # Agent operating contract
└── LICENSE                   # AGPL-3.0
```

---

## Tech Stack

| Layer | Technology |
|---|---|
| **Desktop Shell** | Electron 44 (macOS, Windows, Linux) |
| **Frontend Framework** | React 19, TypeScript 7 |
| **Build Tooling** | Vite 8, electron-vite 6, Tailwind CSS v4, Bun 1.4 |
| **Design System** | shadcn/ui, Radix UI primitives, Lucide icons |
| **State Management** | Zustand 5, Redux Toolkit (shared slices), TanStack Query 5 |
| **Routing** | TanStack Router |
| **Backend** | Python 3.11+, FastAPI, Uvicorn |
| **ML Framework** | PyTorch 2.8, torchaudio 2.8, Transformers 5.10+ |
| **ASR** | WhisperX, Faster-Whisper, Sherpa-ONNX, Parakeet, MLX-Whisper |
| **Audio DSP** | Pedalboard, Demucs, AudioSeal, pydub, soundfile |
| **Database** | SQLite + Alembic migrations |
| **Python Packaging** | Hatchling + Astral `uv` |
| **JS Package Manager** | Bun (workspaces) |
| **Desktop Packaging** | electron-builder (NSIS, DMG, AppImage, deb) |
| **Container** | Docker / Podman (CPU, CUDA, ROCm flavors) |
| **Native** | Rust (desktop-bridge crate) |
| **Distributed Compute** | gRPC + Protobuf (remote GPU workers) |
| **AI Agent Protocol** | FastMCP (Model Context Protocol) |
| **Localization** | i18next (21 locales) |
| **Analytics** | PostHog (opt-in, consent-gated) |
| **License** | AGPL-3.0 (+ commercial Pro tier via Lemon Squeezy) |

---

## Backend (Python / FastAPI)

### Entry Points

**`backend/main.py`** — Two-phase early-bind startup:

1. **Phase A** (instant): Uvicorn binds `127.0.0.1:3900`. `/health` and `/startup/progress` respond within 1 second while heavy imports (PyTorch, model managers, 30+ routers) load in a background executor.
2. **Phase B** (deferred): DB migrations, demo project seeding, orphaned job recovery, model warming.

**Middleware stack** (in order):
| Middleware | Purpose |
|---|---|
| `RenderTraceMiddleware` | Request tracing and diagnostic profiling |
| `StartupGateMiddleware` | Returns 503 until deferred startup completes |
| `NetworkAccessMiddleware` | LAN share PIN enforcement for non-loopback |
| `BearerKeyMiddleware` | `OMNIVOICE_API_KEY` authentication |
| `CORSMiddleware` | Dynamic origin allowlisting |
| `BackendMarkerMiddleware` | Injects `x-omnivoice-backend: <version>` header |

**CLI options**: `--diagnose`, `--deep`, `--health-check`.

**`backend/mcp_server.py`** — FastMCP server exposing tools for AI agents via stdio/SSE/HTTP at `/mcp`:
- Tools: `generate_speech`, `clone_voice`, `transcribe`, `describe_voice`, `list_voices`, `list_languages`, `list_personalities`, `check_health`
- Resources: `voice://{profile_id}`, `history://recent`
- Sandboxed filesystem boundary via `OMNIVOICE_MCP_BASE_PATH`

### API Surface

The backend exposes **40+ routers** across these domains:

| Domain | Key Endpoints | Description |
|---|---|---|
| **Generation** | `POST /generate`, `GET /generate/budget` | TTS synthesis with cloning, design, instruct, speed, diffusion |
| **Audio** | `GET /audio/{id}.ogg\|opus`, `POST /convert` | Transcoding, serving, speech-to-speech conversion |
| **History** | `GET /history`, `PUT /history/{id}/starred` | Generation history management |
| **Voice Profiles** | `GET/POST /profiles`, `POST /profiles/{id}/consent` | Voice clone profiles, consent attestation |
| **Voice Design** | `POST /design/describe` | NL prompt → gender, age, pitch, accent mapping |
| **Personas** | `POST /personas/export\|import\|inspect` | Persona bundle format |
| **Dubbing** | `POST /dub/upload`, `/dub/ingest-url`, `/dub/transcribe-stream/{id}` | Upload, yt-dlp ingest, ASR transcription |
| **Dub Generate** | `POST /dub/generate/{id}`, `/dub/preview-segment/{id}` | Per-segment synthesis |
| **Dub Translate** | `POST /dub/translate`, `/dub/agent-fit` | Translation + timing fit |
| **Dub Export** | SRT, WebVTT, ASS karaoke, stem export, MP3/WAV | Multi-format download |
| **Dictation** | `POST /transcribe`, `WS /ws/transcribe` | One-shot and real-time WebSocket ASR |
| **TTS Streaming** | `WS /ws/tts` | Real-time WebSocket speech synthesis |
| **Audiobook** | `POST /audiobook/import`, `/audiobook/plan` | EPUB/PDF/TXT import, chapter planning |
| **Longform** | `POST /longform/render` | Batch render with checkpointing and resume |
| **Engines** | `GET /engines` | TTS, ASR, LLM, translation, diarization registry |
| **System** | `GET /system/info`, `/sysinfo`, `/system/logs/stream` | Hardware profiling, SSE log streaming |
| **Diagnostics** | `POST /system/diagnostic-bundle` | Anonymized debug zip |
| **OpenAI API** | `POST /v1/audio/speech`, `/v1/audio/transcriptions` | Drop-in OpenAI TTS/ASR replacement |
| **Workers** | `/workers/*` | Remote GPU node registration, dispatch, mTLS |
| **Telephony** | `/calls/*` | Twilio call session handling |

### Engine Registry

VoiceStudio supports a plug-in engine architecture with isolated process wrappers for heavier engines:

```
backend/engines/
├── cosyvoice/              # CosyVoice 3 subprocess wrapper
├── dots_tts/               # DotsTTS engine
├── index_tts/              # IndexTTS 2.5
├── moss_tts/               # MOSS TTS (Nano + v15)
├── omnivoice_gguf/         # Quantized GGUF runtime (omnivoice.cpp)
├── supertonic3/            # Supertonic-3
├── voxcpm2/                # VoxCPM2
└── audio_cpp/              # audio.cpp native bundle
```

### ML & Audio Processing

| Component | Implementation |
|---|---|
| **TTS Pool** | Abstract `TTSBackend` with lease-based lifecycle; VRAM-aware GPU worker pool (5 GB budget, 1–4 workers, CUDA/ROCm) |
| **Audio DSP** | Pedalboard + torchaudio: normalization, EQ, compression, de-essing, noise reduction, reverb |
| **Format I/O** | WAV (16/24/32-bit), MP3, FLAC, OGG/Opus |
| **Stem Separation** | Demucs vocal/instrumental isolation |
| **ASR** | WhisperX (wav2vec2 alignment, sub-30ms timestamps), Faster-Whisper, PyTorch Whisper, MLX-Whisper, Parakeet TDT, Sherpa-ONNX |
| **Diarization** | Pyannote-audio speaker segmentation |
| **Watermarking** | Meta AudioSeal neural watermark via `mark_synthetic` chokepoint |
| **Voice Conversion** | RVC (Retrieval-based Voice Conversion), cascaded ASR→TTS |
| **Mastering** | ACX-compliant 2-pass mastering, loudness normalization |

### Database & Storage

- **Engine**: SQLite at `<DATA_DIR>/omnivoice.db`
- **Migrations**: Alembic (13 versions): settings store, voice profiles, consent records, MCP bindings, pronunciation dictionary, history starring, remote workers, call sessions, voice design recipes
- **Encryption**: Fernet-encrypted HF tokens in settings store

**Data directories** (OS-specific):
| OS | Path |
|---|---|
| Windows | `%APPDATA%\OmniVoice` (cache: `%LOCALAPPDATA%\OmniVoice\hf_cache`) |
| macOS | `~/Library/Application Support/OmniVoice` |
| Linux | `~/.omnivoice` |

**Subdirectories**: `voices/` (reference clips), `outputs/` (generated audio), `dub_jobs/` (dubbing projects), `preview/` (temporary previews)

### Worker / Task Architecture

1. **In-Process Task Manager** (`core/tasks.py`): Lightweight async job queue with SSE (`/tasks/stream/{id}`) and WebSocket (`/ws/events`) progress updates
2. **GPU Executor Pool** (`services/model_manager.py`): Custom thread pool with crash recovery, `StopIteration` guards, cancellation
3. **Remote GPU Workers** (`worker/`): gRPC-based distributed execution with TLS pairing, deadline propagation, automatic task migration on node dropout

---

## Frontend (Electron / React)

### Main Process

**`electron/src/main/index.ts`** manages:
- Single-instance lock (`app.requestSingleInstanceLock()`)
- `BackendSupervisor`: spawns/supervises the local Python backend (or proxies remote)
- Custom protocol registration (`APP_ORIGIN` scheme)
- `BrowserWindow` with hidden titlebar (overlay controls on Windows/Linux, traffic lights on macOS), context isolation enabled
- Tray icon, native audio capture, watch folders, auto-updates (`electron-updater`), crash journals (`MainErrorJournal`)
- Graceful shutdown with persistence flushing

**Key main-process modules** (~90 files):
- `backend.ts` / `backend-download.ts` / `backend-port.ts` — Backend lifecycle
- `updater.ts` — Auto-update against GitHub Releases (SHA-512 validation)
- `ipc.ts` — Central IPC handler registration
- `pro-license.ts` — Ed25519 offline license validation
- `watch-folders.ts` — File system watchers for Pro users
- `repair-agents.ts` — Self-repair and translation agent bridge
- `crash-journal.ts` — Error persistence across restarts
- `site-browser.ts` — Embedded browser control
- `dictation-output.ts` — Native dictation widget

### Preload Bridge

**`electron/src/preload/index.ts`** exposes `window.voicestudio` via `contextBridge.exposeInMainWorld()`:

| Namespace | Capabilities |
|---|---|
| `app` | Version, platform, isDev, navigation hooks, persistence flush |
| `backend` | Status, crash ack, runtime setup, remote config, WebSocket URLs |
| `files` | File dialogs, audio/data saving, script extraction (.docx), path reveal |
| `window` | Minimize, maximize toggle, close, maximize change listener |
| `capture` | Native global dictation and floating widget IPC |
| `pro` | License status, activate, deactivate |
| `updates` | Check, download, install, dismiss, list releases |
| `watch` | Folder watcher triggers and queuing |
| `repair` | Repair/translation agent invocation and events |
| `permissions` | OS permission checks (mic, accessibility) |
| `browser` | Embedded site browser control |
| `maintenance` | Disk reset, data relocation, uninstall cleanup |

### Renderer

**`electron/src/renderer/src/main.tsx`** bootstraps:
1. Error handling and console capture
2. Font injection (Inter, Source Serif 4, IBM Plex Mono)
3. i18n initialization
4. Conditional mount: `<CaptureWidget />` (dictation popup) or full `<App />`

**`app.tsx`** sets up:
- `QueryClientProvider` (TanStack Query)
- `RouterProvider` (TanStack Router)
- `GenerationProvider`, `TooltipProvider`, Sonner `Toaster`
- Background syncs: `NativeDictationSync`, `ModelInstallSync`, `RealtimeEventSync`, `GenerateBudgetSync`, `UpdateNotifier`

**Component architecture**:
- `src/components/ui/` — shadcn/ui primitives (Radix-based)
- `src/components/app-shell/` — Layout, top-bar, nav-rail, status-bar
- `src/shared/components/` — Shared domain components (audiobook, clone, dub, gallery, engines, settings, catalogue, profile)
- Specialized: `wavesurfer.js` (waveforms), `@xyflow/react` (workflow editor), `@vidstack/react` (video), `remotion` (video manipulation), `@scalar/api-reference-react` (API docs)

### State Management

Multi-tier approach:

| Layer | Technology | Location | Purpose |
|---|---|---|---|
| **Local UI** | Zustand 5 | `renderer/src/lib/store/` | Clone settings, dictation, output, references, takes, workspace |
| **Reactive App** | TanStack Store | `renderer/src/lib/` | App-level reactive state |
| **Server State** | TanStack Query 5 | Various hooks | Backend data fetching, caching, polling |
| **Shared State** | Redux Toolkit | `shared/store/` | Cross-concern slices: generate, longform, prefs, UI, updater, dub, gallery, glossary, donation, releases |

### Routing & Feature Pages

**TanStack Router** manages routes with these feature modules (`renderer/src/features/`):

| Feature | Description |
|---|---|
| `home` | Main synthesis & workspace dashboard |
| `clone` | Voice cloning workflow |
| `design` | Natural language voice design |
| `dub` | Video dubbing & translation |
| `longform` | Multi-segment audiobook/story editor |
| `transcriptions` | Speech-to-text & dictation capture |
| `gallery` | Voice & model library browser |
| `batch` | Batch generation queue |
| `workflows` | XYFlow-based audio node pipeline editor |
| `projects` | Project management |
| `settings` | System, backend, storage, audio preferences |
| `tools` | Utility tools |
| `calls` | Telephony / call agent |
| `browser` | Embedded site browser |
| `pro` | Pro license management |
| `integrations` | Third-party integration setup |

### Internationalization

- **Library**: i18next + react-i18next + browser language detector
- **21 locales**: ar, de, en, es, fr, hi, id, it, ja, ko, nl, pl, pt, ru, sv, th, tr, uk, vi, zh-CN, zh-TW
- **Enforcement**: `locale:check` script validates encoding, source keys, and coverage; `tests/test_locale_parity.py` enforces parity; `tests/test_no_hardcoded_cjk.py` catches hardcoded CJK

---

## MCP Server (AI Agent Integration)

VoiceStudio ships a native **Model Context Protocol** server (`backend/mcp_server.py`) for integration with Claude Desktop, Cursor, and other MCP-compatible AI agents.

**Transport**: stdio, SSE, or streamable HTTP at `/mcp`

| Type | Name | Description |
|---|---|---|
| Tool | `generate_speech` | Synthesize speech from text |
| Tool | `clone_voice` | Clone a voice from reference audio |
| Tool | `transcribe` | Transcribe audio to text |
| Tool | `describe_voice` | Get voice characteristics |
| Tool | `list_voices` | List available voice profiles |
| Tool | `list_languages` | List supported languages |
| Tool | `list_personalities` | List voice personality presets |
| Tool | `check_health` | Health check |
| Resource | `voice://{profile_id}` | Voice profile data |
| Resource | `history://recent` | Recent generation history |

---

## Native Bridge (Rust)

`native/desktop-bridge/` provides a Rust crate for platform-native integrations that Electron's Node.js layer cannot efficiently handle.

---

## Core Features

| # | Feature | Description |
|---|---|---|
| 1 | **Voice Cloning** | Zero-shot cloning from 5–30s audio across 646 languages |
| 2 | **Voice Design** | Generate synthetic voices from natural language descriptions (age, gender, pitch, accent) |
| 3 | **Video Dubbing** | End-to-end: URL ingest → vocal separation (Demucs) → ASR → translation → prosody matching → subtitle export |
| 4 | **Live Dictation** | Real-time mic capture via Sherpa-ONNX streaming or offline Whisper, with floating widget |
| 5 | **Transcription** | Multi-engine speech-to-text with forced alignment |
| 6 | **Audiobook / Longform** | EPUB/PDF/TXT import, character dialogue assignment, chapter rendering, stateful resume |
| 7 | **Batch Queue** | Bulk generation job management |
| 8 | **Workflow Editor** | Visual XYFlow-based audio processing node graph |
| 9 | **Voice Gallery** | Community and starter voice library (`.ovsvoice` format) |
| 10 | **Vocal Isolation** | Demucs-powered stem separation |
| 11 | **Speaker Diarization** | Pyannote-based speaker segmentation |
| 12 | **AI Watermark** | AudioSeal invisible watermarking on all synthetic audio |
| 13 | **OpenAI API** | Drop-in `/v1/audio/speech` and `/v1/audio/transcriptions` |
| 14 | **MCP Server** | AI agent integration via Model Context Protocol |
| 15 | **Remote Workers** | Distributed GPU compute via gRPC + mTLS |
| 16 | **Telephony** | Twilio call agent integration |
| 17 | **GPU Auto-Detect** | CUDA, MPS, ROCm, DirectML, CPU fallback |
| 18 | **Local-First** | Fully offline, no cloud dependencies |

---

## Supported Engines

### Text-to-Speech (TTS)

| Engine | Model | Size | Notes |
|---|---|---|---|
| `omnivoice` (default) | k2-fsa/OmniVoice 0.6B + Higgs Audio v2 | 2.4 GB | 600+ languages, zero-shot clone |
| `omnivoice-gguf` | Quantized GGUF (Q4_K_M ~659 MB) | 659 MB–4.98 GB | CPU / 4 GB VRAM devices |
| `kittentts` | KittenML/kitten-tts-mini-0.8 | 80 MB | ONNX CPU, lightweight |
| `cosyvoice` | FunAudioLLM/Fun-CosyVoice3-0.5B | 9.8 GB | Subprocess engine |
| `voxcpm2` | openbmb/VoxCPM2 | 5.0 GB | Subprocess engine |
| `audiocpp` | audio-cpp/audio.cpp-gguf | 4.98 GB | Native C++ bundle |
| `indextts2` | IndexTTS 2.5 | — | Subprocess engine |
| `moss-tts-v15` / `nano` | OpenMOSS-Team/MOSS-TTS | — | Nano: 100M params |
| `supertonic3` | Supertonic-3 | — | — |
| `dots-tts` | DotsTTS | — | — |
| `pockettts` | PocketTTS | — | Lightweight |
| `confucius4-tts` | Confucius-4 TTS | — | — |
| `gpt-sovits` | lj1995/GPT-SoVITS | — | — |
| `sherpa-onnx` | csukuangfj/sherpa-onnx | — | CPU int8 |
| `mlx-audio` | Apple Silicon MLX models | — | macOS only: Kokoro, CSM, Qwen3-TTS, Dia, OuteTTS, Chatterbox, MeloTTS |

### Speech Recognition (ASR)

| Engine | Model | Notes |
|---|---|---|
| `whisperx` (default) | Systran/faster-whisper-large-v3 | wav2vec2 forced alignment, sub-30ms timestamps |
| `faster-whisper` | deepdml/faster-whisper-large-v3-turbo-ct2 | CTranslate2 backend |
| `pytorch-whisper` | openai/whisper-large-v3 | PyTorch native (ROCm compatible) |
| `mlx-whisper` | mlx-community/whisper-large-v3-mlx | Apple Silicon |
| `parakeet` | nvidia/parakeet-tdt-0.6b-v3 | NeMo & MLX variants |
| `moonshine` | Moonshine | — |
| `funasr` | FunASR | — |
| `sherpa-onnx` | sherpa-onnx streaming/offline | Live dictation, CPU int8 |
| `openai-compatible` | Any OpenAI-compatible API | External service |

### Other Engines

| Type | Engines |
|---|---|
| **Diarization** | pyannote/speaker-diarization-3.1 (gated via HF token) |
| **Translation** | facebook/nllb-200-distilled-600M (offline), Argos, LLM-based |
| **LLM** | OpenAI, LiteLLM (100+ providers) |

---

## Model Catalog

All supported model weights are declared in `backend/config/models.yaml`, the single source of truth for HuggingFace downloads. Each entry specifies the repo ID, expected size, checksum, and compatibility tags.

---

## Setup & Installation

### One-Command Install

**macOS / Linux:**
```bash
curl -fsSL https://voicestudio.sh/install | sh
```

**Windows (PowerShell):**
```powershell
irm https://voicestudio.sh/install | iex
```

**Options:**
- `--version X.Y.Z` — Install a specific release
- `--main` — Build from current `main` branch
- `--uninstall` — Remove the app (keeps data)

### From Source (Development)

**Prerequisites**: Git, Node.js 22+, Bun, Python 3.11+, `uv` (Astral)

```bash
git clone https://github.com/debpalash/VoiceStudio.git
cd VoiceStudio
bun install
bun run setup:api    # Prepare Python environment and dependencies
bun run dev          # Launch Electron in dev mode
```

**Other dev commands:**
| Command | Description |
|---|---|
| `bun run dev:api` | Start only the backend |
| `bun run dev:frontend` | Start only the web renderer |
| `bun run dev:web` | Backend + frontend concurrently |
| `bun run build` | Production build |
| `bun run dist` | Package desktop installer |
| `bun run smoke-test` | Build and launch isolated packaged app |
| `bun run check:electron` | Typecheck + test + build + packaging contract |
| `bun run test` | Run frontend tests (Vitest) |
| `bun run lint` / `bun run format` | Lint / format |

### Docker / Podman

```bash
# CPU
docker compose up voicestudio

# NVIDIA GPU
docker compose up voicestudio-gpu

# AMD ROCm
docker compose up voicestudio-rocm

# Standalone GPU worker
docker compose up worker
```

**Images** (GHCR + Docker Hub):
| Tag | Channel |
|---|---|
| `:latest` | Rolling preview from `main` |
| `:X.Y.Z` / `:X.Y` / `:stable` | Tagged releases |
| `:rocm` / `:stable-rocm` | AMD ROCm variants |

### Hardware Support

| Hardware | Acceleration |
|---|---|
| NVIDIA GPU (Windows/Linux) | CUDA |
| Apple Silicon (macOS) | Metal (MPS) |
| AMD GPU (Linux) | ROCm |
| No dedicated GPU | CPU (slower but fully functional) |
| Windows on ARM | Experimental (native app, x64 Python under emulation, CPU only) |
| Intel Mac | App UI only; connect to a remote backend |

---

## Configuration

**User config** is stored outside the repo in OS-specific locations, managed via the Settings panel or `backend/core/user_env.py`:

| Variable | Default | Description |
|---|---|---|
| `OMNIVOICE_PORT` | `3900` | Backend API port |
| `OMNIVOICE_UI_PORT` | — | Frontend dev server port |
| `OMNIVOICE_TORCH_VARIANT` | `auto` | `auto` / `cuda` / `cpu` |
| `OMNIVOICE_CPU_DTYPE` | `bfloat16` | CPU inference dtype |
| `HF_HUB_OFFLINE` | `0` | Block all HuggingFace downloads |
| `HF_HUB_CACHE` | OS-specific | HuggingFace model cache path |
| `OMNIVOICE_API_KEY` | — | Bearer key for remote API access |
| `OMNIVOICE_MCP_BASE_PATH` | — | MCP filesystem sandbox |
| `VOICESTUDIO_DISABLE_UPDATER` | `0` | Disable auto-update checks |

---

## Build & Packaging

| Target | Tool | Output |
|---|---|---|
| Electron main + preload + renderer | electron-vite 6 + Vite 8 | `electron/out/` |
| Web standalone build | Vite (`vite.web.config.ts`) | `electron/dist-web/` |
| Desktop installers | electron-builder 26 | Per-platform packages |
| Python backend | Hatchling + `uv` | Wheel / editable install |
| Docker images | `deploy/Dockerfile` | Multi-stage, multi-variant |
| Rust native bridge | Cargo | Platform-specific binary |

**Desktop installer targets:**
- **Windows**: NSIS `.exe` (x64 + ARM64)
- **macOS**: `.dmg` + `.zip` (arm64, x64)
- **Linux**: AppImage (x64), `.deb` (x64)

---

## Testing Strategy

Three hermetic test suites, deliberately separated:

| Suite | Tool | Location | Scope |
|---|---|---|---|
| **Python integration** | pytest | `tests/` | Contract tests, version lockstep, locale parity, changelog style, CJK enforcement |
| **Backend unit** | pytest | `backend/tests/` | Isolated backend tests (no heavy imports), Linux + Windows CI |
| **Frontend** | Vitest + JSDOM | `electron/src/**/*.test.{ts,tsx}` | Component tests, shared API tests |
| **E2E / Smoke** | Playwright | `tests/frontend/` | Workflow smokes: playback, dubbing, etc. |
| **Packaging** | Node.js scripts | `electron/tests/` | `packaging-contract.mjs`, `update-package-contract.mjs` |

**Key contract tests:**
- `test_app_version.py` — Version lockstep (package.json ↔ pyproject.toml ↔ version.py)
- `test_locale_parity.py` — All 21 locales contain every key
- `test_changelog_style.py` — Changelog formatting rules
- `test_no_hardcoded_cjk.py` — No hardcoded CJK outside allowlist

**CI honesty**: Tests run with `HF_HUB_OFFLINE=1` and empty `HF_HUB_CACHE` to prevent dev-cache masking failures.

---

## CI/CD Pipelines

| Workflow | Trigger | Purpose |
|---|---|---|
| `ci.yml` | PR + push | Full gate: installer contract, Python tests, frontend tests, Playwright smokes, multi-OS smoke matrix (macOS ARM64, Windows 2022, Ubuntu 22.04) |
| `electron-build.yml` | — | Cross-platform packaging (Linux x64, Windows x64/ARM64, macOS arm64/x64) |
| `electron-release.yml` | Tag | Desktop release to GitHub Releases |
| `docker.yml` | — | Build + push to GHCR and Docker Hub (`:latest`, `:stable`, `:rocm`) |
| `build-omnivoice-tts.yml` | — | Native C++ sidecar build |
| `docs-drift.yml` | Nightly | Verify `features.yaml` matches README and engine registries |
| `security.yml` | — | Gitleaks secret scanning |
| `commit-identity.yml` | PR | No AI agent attribution in commits |
| `cla.yml` | PR | Contributor License Agreement check |
| `evals.yml` | — | Model evaluation runs |
| `install-smoke.yml` | — | Installer smoke tests |

---

## Deployment Channels

A release is not complete until **every** channel ships:

1. **Electron GitHub Release** — Linux x64, Windows x64 + ARM64, macOS arm64 + x64
2. **GHCR** — `ghcr.io/debpalash/voicestudio` (CUDA + ROCm)
3. **Docker Hub** — Same image variants + synced overview from `deploy/dockerhub-overview.md`
4. **Final Tauri feeds** — Immutable compatibility assets (never rebuilt)

**Update mechanism**: 6-hour polling + launch checks against GitHub Releases. SHA-512 checksum verification. Pre-migration SQLite snapshots. Can be disabled with `VOICESTUDIO_DISABLE_UPDATER=1`.

---

## Licensing & Commercial (Pro)

**Core**: GNU AGPL-3.0 — free local voice creation, cloning, dubbing, audiobooks, batching.

**Desktop Pro** (via Lemon Squeezy):
| Plan | Price |
|---|---|
| Yearly | $99/user/year |
| Lifetime | $299/user |
| Enterprise | Custom |

**Pro features**: Watch folders, remote-device compute, remote workers, encrypted GPU sharing.

**License verification**: Offline Ed25519 certificate validation (`vslc1.` certificate, `vsle1.` endorsement, compiled root keys). Enforced at both Electron main process and backend boundaries.

---

## Telemetry & Privacy

- **Analytics**: Opt-in only via Settings > Privacy. PostHog US with strict sanitization — no file paths, credentials, audio, or text content.
- **First-run consent**: Two equal-weight Yes/No buttons; skipping = off; never default-on.
- **Synthetic watermark**: AudioSeal neural watermark enforced through the `mark_synthetic` chokepoint on all generated audio.
- **Network calls**: Only user-initiated (see [sanctioned calls list in CONTRIBUTING.md](.github/CONTRIBUTING.md)).
- **Bug reports**: Opt-in, prefilled GitHub Issue URLs, submitted from user's own browser.

---

## Governance & Contribution

### Invariant Rules

1. **Local-first**: No required network calls. Everything works offline.
2. **Cross-platform parity**: Default behavior identical on macOS/Windows/Linux. Performance may vary.
3. **Single version source**: Root `package.json` — mirrored to `pyproject.toml` and `backend/core/version.py`.
4. **Localization**: Every user-facing string in all 21 locales.
5. **No AI attribution**: No agent co-authors, session links, or "Generated with" in commits.
6. **Docs-sync**: Documentation updated in the same PR as the code change.
7. **Keep main green**: Never merge something that breaks CI.

### Merge Protocol

1. Harvest CodeRabbit + Greptile bot reviews first
2. Fix all findings on the PR branch pre-merge (no merge-then-fix)
3. Merge current `main` into stale branches before judging CI
4. Gate: "Tests (backend + frontend)" green + MERGEABLE
5. Watch post-merge `main` runs to green

### Review Bots

- **CodeRabbit** — Configured via `.coderabbit.yaml`
- **Greptile** — Configured via `greptile.json`
- Both fed `CLAUDE.md` as context

---

## Documentation Map

| Need | Location |
|---|---|
| Platform install guides | `docs/install/{macos,windows,linux,docker,script}.md` |
| Troubleshooting | `docs/install/troubleshooting.md` |
| Engine guides (30 engines) | `docs/engines/README.md` |
| Feature catalog | `docs/feature-catalog.md` + `docs/features.yaml` |
| MCP integration | `docs/mcp.md` + `docs/mcp.json` |
| API & integrations | `docs/speech-platform.md`, `examples/` |
| Performance & benchmarks | `docs/performance.md`, `docs/benchmarks.md` |
| Architecture decisions | `docs/adr/` |
| Feature specs & roadmap | `docs/specs/`, `docs/ROADMAP.md` |
| Release process | `docs/RELEASING.md` |
| Contributing | `.github/CONTRIBUTING.md` |
| Remote GPU workers | `docs/remote-gpu.md`, `docs/remote-workers.md` |
| Telephony / Twilio | `docs/integrations/twilio.md`, `docs/calls.md` |
| Dubbing pipeline | `docs/dubbing/`, `docs/electron-dubbing.md` |
| Audiobook / longform | `docs/electron-longform.md`, `docs/specs/longform/` |
| Electron migration (from Tauri) | `docs/electron-migration.md` |
| Branding | `docs/branding.md` |
| Security | `.github/SECURITY.md` |
| Sponsorship | `SPONSORS.md`, `docs/playbooks/oss-sponsorship-setup.md` |
