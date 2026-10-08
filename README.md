# Local Voice Agent

A private, low-latency voice assistant that runs entirely on your own hardware.
Speak to it in the browser; speech recognition, language inference, and speech
generation all happen locally — nothing leaves your machine unless you
explicitly point a service at a remote endpoint.

## Key features

- **Fully local voice pipeline** — streaming STT, LLM, and TTS run on your
  machine; no cloud APIs required.
- **Hardware-aware model selection** — the launcher detects CPU, NVIDIA GPU,
  Apple Silicon, or Jetson and picks a model profile that fits available memory.
- **Real-time conversation** — WebRTC audio via LiveKit with streaming (partial)
  transcripts, so replies start while you are still finishing a sentence.
- **Single supervised process** — one Python supervisor spawns, health-checks,
  and restarts every service (LiveKit server, LLM, STT, TTS, agent worker).
- **Wake word (optional)** — "Hey LiveKit" gating so the agent only listens
  after the wake phrase (`WAKE_WORD=1`).
- **Remote-client mode** — run the voice stack on one machine (e.g. Jetson) and
  the browser UI on another over your LAN.
- **CPU, NVIDIA CUDA, Jetson, and Apple Silicon support** — Docker images for
  Linux/Jetson, native runtime for macOS.

## Technology stack

| Technology | Purpose |
| ---------- | ------- |
| LiveKit / LiveKit Agents | Real-time audio (WebRTC) and voice-agent orchestration |
| llama.cpp (`llama-server`) | Local LLM inference (OpenAI-compatible API) |
| NeMo-Speech.cpp (Nemotron) | Streaming speech-to-text (default) |
| faster-whisper | Alternative STT backend |
| Kokoro / kokoro-onnx | Local text-to-speech |
| FastAPI + uvicorn | Token minting, status API, static frontend hosting |
| Next.js (static export) | Browser interface |
| Python supervisor | Process lifecycle, readiness, restarts |
| Docker / Docker Compose | Reproducible CPU, GPU, and Jetson deployments |
| uv / pnpm | Python and frontend dependency management |

## Architecture

```
Browser (Next.js static export)
   │  microphone / speakers
   ▼
LiveKit server ◄──── WebRTC (ports 7880/7881/7882/udp)
   │  room audio in/out
   ▼
LiveKit Agent worker (port 8081)
   │
   ├──► STT service  (Nemotron streaming WebSocket, or OpenAI-compatible HTTP)
   ├──► LLM service  (llama-server, OpenAI-compatible HTTP, port 11434)
   └──► TTS service  (Kokoro, OpenAI-compatible HTTP, port 8880)
```

### The voice processing pipeline

1. **User speaks** into the browser microphone (optionally only after the wake
   phrase is detected).
2. **WebRTC** carries the audio to the local LiveKit server.
3. The **LiveKit Agent worker** receives the room's audio frames.
4. **Speech-to-text**: frames stream over WebSocket to NeMo-Speech.cpp, which
   returns partial transcripts while you speak and a final transcript when
   endpointing (VAD/turn-detector) ends the utterance.
5. **LLM**: the final transcript is sent to `llama-server` over an
   OpenAI-compatible HTTP API; tokens stream back immediately (reasoning output
   is disabled so there is no dead air).
6. **Text-to-speech**: streamed text goes to the Kokoro server
   (OpenAI-compatible `/v1/audio/speech`), which returns PCM/MP3 audio chunks.
7. The agent publishes the audio into the room; the **browser plays it** over
   WebRTC — and the loop repeats for the next turn.

### The supervisor architecture

`python -m local_voice_ai serve` (started by `run.py` or the container
entrypoint) is a single Python process that owns everything:

```
Python supervisor
├── FastAPI web server        :8080  — /api/connection-details (LiveKit token),
│                                       /api/status, static frontend, /healthz
├── livekit-server            :7880/:7881/:7882  (managed dev server)
├── llama-server              :11434 — local LLM
├── STT server                :8000  — Nemotron (native or Python) or Whisper
├── TTS server                :8880  — Kokoro (PyTorch or ONNX)
└── agent worker              :8081  — LiveKit Agents session (STT→LLM→TTS glue)
```

The supervisor spawns each child with `asyncio` subprocesses, polls a
per-service readiness URL, monitors health after startup, restarts crashed
children with linear backoff, and terminates everything cleanly on shutdown.
Because the web server starts *before* the children are ready, the frontend can
poll `/api/status` and show per-model download progress during first boot.

### Communication protocols

| Hop | Protocol |
| --- | -------- |
| Browser ↔ LiveKit | WebRTC (UDP media, TCP fallback) + WebSocket signaling |
| Browser ↔ FastAPI | HTTP (`POST /api/connection-details`, `GET /api/status`) |
| Supervisor ↔ children | subprocess spawn + HTTP readiness/health probes |
| Agent ↔ STT | WebSocket (`/realtime`) for streaming; HTTP for offline recognize |
| Agent ↔ LLM / TTS | OpenAI-compatible HTTP (streaming SSE/chunked audio) |
| FastAPI ↔ LLM/STT/TTS | Optional OpenAI-compatible `/v1/*` gateway (off by default) |

## Model architecture

Model selection is data-driven: `local_voice_ai/profiles.json` maps **platforms**
(runtime + GPU flags) to **model profiles** (memory budget → model set), and the
launcher resolves them against detected hardware.

| Profile | Memory target | LLM | STT | TTS |
| ------- | ------------: | --- | --- | --- |
| `lean` | ~4.7 GB | Qwen3 1.7B Q4 (4K ctx) | Nemotron Q8 streaming | Kokoro ONNX FP16 |
| `jetson-realtime` | ~4.7 GB | Qwen3 1.7B Q4 (4K ctx) | Nemotron Q8 streaming | Kokoro ONNX FP16 |
| `compact` | ~5.5 GB | Gemma 4 E2B QAT Q4 (4K ctx) | Nemotron Q8 streaming | Kokoro 82M |
| `balanced` | ~6.5 GB | Gemma 4 E2B QAT Q4 (16K ctx) | Nemotron Q8 streaming | Kokoro 82M |

Key environment variables:

| Variable | Role |
| -------- | ---- |
| `LLAMA_MODEL` / `LLAMA_MODEL_ALIAS` | Model name reported by the agent |
| `LLAMA_HF_REPO` | GGUF repository + quantization tag (e.g. `…:UD-Q4_K_XL`) |
| `LLAMA_MODEL_PATH` | Load a local `.gguf` directly (fully offline) |
| `LLAMA_N_GPU_LAYERS` | GPU offload (CPU `0`, GPU/Metal `999`) |
| `LLAMA_OFFLINE` | Force cache-only startup once the model is downloaded |
| `STT_PROVIDER` | `nemotron-cpp` (default), `nemotron`, or `whisper` |
| `STT_LANGUAGE` | `en` uses the English Nemotron model; other codes use multilingual |
| `TTS_PROVIDER` | `kokoro` (PyTorch) or `kokoro-onnx` (low-memory) |
| `TTS_VOICE` | Kokoro voice (default `af_nova`) |
| `WAKE_WORD` | `1` enables the "Hey LiveKit" wake gate |

Model weights download on first start into a cache volume (`/models` in Docker,
`~/.cache` natively), are verified by checksum where pinned, and are reused on
every later start — after the first run the stack starts offline.

## Hardware support

| Platform | Runtime | Notes |
| -------- | ------- | ----- |
| Linux CPU | Docker Compose | Base image, `LLAMA_N_GPU_LAYERS=0` |
| Desktop NVIDIA | Docker Compose + GPU overlay | Requires NVIDIA Container Toolkit |
| Jetson Orin Nano | `Dockerfile.jetson` | JetPack 6.2 / L4T 36.4; builds llama.cpp & NeMo-Speech.cpp for SM 87 |
| Apple Silicon | Native (`uv`) | llama.cpp Metal + `nemo-speech` macOS build |

## Privacy and local inference

By default **no audio, transcript, or prompt ever leaves your machine**: LiveKit,
the LLM, STT, and TTS all run on localhost, and the browser talks only to your
server. You can *optionally* redirect any single service to a remote endpoint
(LiveKit Cloud, an OpenAI-compatible LLM API, a hosted STT/TTS) by setting its
`*_BASE_URL` — pointing a base URL off-box automatically disables the matching
local process.

## Project structure

```
run.py                         # hardware-aware setup launcher (stdlib only)
local_voice_ai/
├── __main__.py                # CLI: serve / download-models / console
├── supervisor.py              # child process orchestration
├── api.py                     # FastAPI: token, status, optional gateway, static UI
├── config.py                  # environment-driven configuration
├── agent.py                   # LiveKit Agents worker (STT→LLM→TTS session)
├── nemotron_stt.py             # streaming STT adapter (WebSocket realtime)
├── profiles.py / profiles.json# hardware detection + model/platform catalog
├── launcher.py / native_runtime.py  # wizard UI + pinned runtime installer
├── wakeword.py                # wake-word gate
└── services/                  # STT/TTS OpenAI-compatible servers
    ├── nemotron_cpp/          # native streaming STT launcher (default)
    ├── nemotron/              # Python NeMo STT server (alternative)
    ├── whisper/               # faster-whisper STT server (fallback)
    ├── kokoro/                # PyTorch Kokoro TTS server
    └── kokoro_onnx/           # ONNX Kokoro TTS server (low memory)
frontend/                      # Next.js static-export browser interface
tests/                         # pytest suite
Dockerfile                     # CPU/GPU multi-stage image
Dockerfile.jetson              # Jetson image (compiles native tools)
docker-compose*.yml            # base, GPU, and Jetson overlays
```

## Configuration

Settings merge with this precedence (first wins):

```
real environment  >  .env.local  >  saved profile (.local-voice-ai.toml)  >  .env
```

- `.env` — repository defaults (tracked, non-secret).
- `.env.local` — per-machine overrides and any real API keys (**never
  committed**; ignored by Git).
- `.local-voice-ai.toml` — profile chosen by the launcher (local, untracked).

See [`.env`](./.env) for the full option list.

## Getting started

### Requirements

| Platform | Requirement |
| -------- | ----------- |
| Linux CPU / NVIDIA | Docker Engine + Docker Compose (NVIDIA also needs the container toolkit) |
| Jetson Orin | JetPack 6.2, L4T 36.4, NVIDIA Docker runtime |
| Apple Silicon | Python 3.11–3.13, `uv`, `livekit-server`, `llama-server` (Homebrew) |

The first start needs an internet connection to fetch model weights (≈2–4 GB
depending on the profile); later starts reuse the cache.

### Start

```bash
python3 run.py                 # interactive: detects hardware, picks a profile
# or non-interactive:
python3 run.py start --profile auto --yes
```

When the stack is ready, open <http://localhost:8080> and allow microphone
access.

Common commands:

| Command | Purpose |
| ------- | ------- |
| `python3 run.py configure` | Re-run the profile wizard |
| `python3 run.py plan` | Show detected hardware and resolved models |
| `python3 run.py status` | Per-service readiness |
| `python3 run.py logs` | Follow container logs |
| `python3 run.py down` | Stop the Docker stack |
| `python3 run.py client --server <host>` | Run the UI on this machine against a remote server |

### Development

```bash
uv sync --extra ml --extra dev     # Python env
pnpm --dir frontend install        # frontend deps
.venv/bin/python -m pytest -q      # tests
pnpm --dir frontend build          # static frontend build
```

## Security

The default configuration targets **local development and trusted private
networks**:

- LiveKit uses the public development credentials (`devkey`/`secret`); model
  endpoints have **no authentication**.
- Keep `.env.local` (and any real API keys) out of Git — it is ignored.
- Do not publish ports `8080`, `7880`, `7881`, `7882`, `11434`, `8000`, or
  `8880` to the internet; restrict firewall rules to your local subnet.
- The token endpoint intentionally allows only localhost origins (plus any
  origins you add via `CLIENT_ORIGINS`).
- The OpenAI-compatible gateway (`GATEWAY=1`) is **off by default** because it
  would expose unauthenticated inference on the web port.
- Add authentication and TLS before exposing the application publicly.

## License

This project is licensed under the MIT License — see [LICENCE](./LICENCE).
Copyright (c) 2025 LiveKit.

## Acknowledgements and third-party components

This is a derivative of [ShayneP/local-voice-ai](https://github.com/ShayneP/local-voice-ai)
(MIT), whose author's work is preserved under the original license; the browser
interface is derived from LiveKit's agent-starter templates. The following
third-party libraries and models make the stack possible — each model carries
its own license, listed on its model card:

- [LiveKit](https://livekit.io/) and [LiveKit Agents](https://docs.livekit.io/agents/)
- [llama.cpp](https://github.com/ggml-org/llama.cpp)
- [NVIDIA Nemotron Speech](https://huggingface.co/nvidia/nemotron-speech-streaming-en-0.6b),
  [Nemotron 3.5 ASR](https://huggingface.co/nvidia/nemotron-3.5-asr-streaming-0.6b), and
  [NeMo-Speech.cpp](https://github.com/NVIDIA/NeMo-Speech.cpp)
- [Gemma 4 E2B (GGUF, QAT)](https://huggingface.co/unsloth/gemma-4-E2B-it-qat-GGUF) — Google Gemma license
- [Qwen3](https://huggingface.co/unsloth/Qwen3-1.7B-GGUF) — see model card for license
- [Kokoro](https://github.com/hexgrad/kokoro) and [kokoro-onnx](https://github.com/thewh1teagle/kokoro-onnx)
- [faster-whisper](https://github.com/SYSTRAN/faster-whisper)
- [Silero VAD](https://github.com/snakers4/silero-vad) and the
  [hello-wakeword](https://github.com/livekit-examples/hello-wakeword) wake-word model
