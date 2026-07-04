# Local Voice Agent — Gemma 4 + LiveKit + Whisper + Qwen3-TTS

A real-time, fully local voice assistant built on a single Kaggle GPU
notebook. It listens over a live LiveKit room, transcribes your speech
locally, generates a reply with a locally-served Gemma 4 model, and speaks
the reply back — no cloud LLM or cloud TTS API required.

```
 Your voice
     │
     ▼
┌─────────────┐     ┌──────────────┐     ┌───────────────┐     ┌──────────────┐
│  LiveKit    │────▶│ faster-      │────▶│  Gemma 4       │────▶│  Qwen3-TTS    │
│  (transport,│     │ whisper      │     │  via Ollama    │     │  (local voice)│
│  turn-taking)│    │ (local STT)  │     │  (local LLM)   │     │               │
└─────────────┘     └──────────────┘     └───────────────┘     └──────────────┘
                                                                        │
                                                                        ▼
                                                                  Spoken reply
```

## Stack

| Layer | Tool | Role |
|---|---|---|
| Transport / orchestration | [LiveKit Agents](https://docs.livekit.io/agents/) | Real-time audio room, turn-taking, worker dispatch |
| Speech-to-Text | [faster-whisper](https://github.com/SYSTRAN/faster-whisper) | Your voice → text, running locally on GPU |
| LLM | [Gemma 4](https://ai.google.dev/gemma) served via [Ollama](https://ollama.com) | Text reply generation, fully local |
| Text-to-Speech | [Qwen3-TTS](https://github.com/QwenLM) | Text → natural voice audio, running locally on GPU |

Everything runs inside one Kaggle notebook process — the LLM, STT, and TTS
models all share the same GPU and the same Python thread pool, so there's no
separate backend to deploy.

## Requirements

- A Kaggle notebook with a **GPU accelerator enabled** (tested on a T4).
- A [LiveKit Cloud](https://cloud.livekit.io) project (free tier is enough)
  to get a `LIVEKIT_URL`, `LIVEKIT_API_KEY`, and `LIVEKIT_API_SECRET`.
- Internet access enabled on the notebook (to install packages and pull the
  Gemma 4 model on first run).

## Setup

1. **Add your LiveKit credentials as Kaggle secrets:**
   Notebook → Add-ons → Secrets, then add:
   - `LIVEKIT_URL` (e.g. `wss://your-project.livekit.cloud`)
   - `LIVEKIT_API_KEY`
   - `LIVEKIT_API_SECRET`
2. **Turn on the GPU accelerator** for the notebook session.
3. **Run all cells from top to bottom.** The first run will take a few
   minutes: it installs system packages, installs Ollama, and downloads the
   Gemma 4 model.
4. Once the last cell prints `Starting agent server to listen for LiveKit
   Console...`, open your LiveKit project's Sandbox/Console and connect to a
   room — the agent will join and greet you.

## Project structure

```
.
├── voice-agent-with-gemma-4.ipynb   # the full notebook, cell-by-cell
├── ERRORS_AND_SOLUTIONS.md          # every error hit during development + the fix
└── README.md                        # this file
```

### What each notebook cell does

1. **Install dependencies** — LiveKit Agents SDK, faster-whisper, Qwen3-TTS,
   and the system packages (`zstd`, `pciutils`, `lshw`, `sox`) Ollama and the
   audio pipeline need under the hood, then installs Ollama itself.
2. **Load LiveKit secrets** from Kaggle into environment variables.
3. **Port-check helper** — a small utility so re-running cells doesn't crash
   with "address already in use."
4. **Start Ollama & pull Gemma 4** — launches the Ollama server manually
   (containers have no `systemd`), enables Flash Attention, and downloads the
   model in its own isolated step.
5. **Qwen3-TTS wrapper** — a LiveKit-compatible TTS engine around the local
   Qwen3-TTS model.
6. **Whisper STT wrapper** — a LiveKit-compatible STT engine around
   `faster-whisper`.
7. **Pre-warm Gemma 4** — sends one dummy request to force the model into GPU
   memory ahead of time, avoiding a slow/timed-out first reply.
8. **Assemble the `AgentServer`** — wires STT, LLM, and TTS together into one
   `AgentSession`, on an alternate health-check port to avoid conflicts.
9. **Run the server** — starts listening for LiveKit to dispatch a room.

## Troubleshooting

Every error hit while building this — and the exact fix — is logged in
[`ERRORS_AND_SOLUTIONS.md`](./ERRORS_AND_SOLUTIONS.md). The short version:

- **`worker is already running`** → restart the kernel, then Run All.
- **`address already in use` (port 11434 or 8081)** → re-run the cell (the
  built-in port check should skip re-launching), or manually run
  `!fuser -k <port>/tcp`.
- **Slow or timed-out first reply** → make sure the pre-warm cell ran before
  you connect from the LiveKit Sandbox.
- **Agent doesn't show up in the Sandbox** → confirm `agent_name=` in
  `@server.rtc_session(...)` matches what the Sandbox is dispatching to.

## Customizing

- **Swap the LLM:** change `GEMMA_MODEL` to any model pulled by Ollama.
- **Swap the TTS voice:** change `TTS_SPEAKER` in the Qwen3-TTS cell (options
  include Eric, Aiden, Vivian, Serena, Dylan, Ryan, and others).
- **Swap the STT size/accuracy tradeoff:** change the model size passed to
  `LocalWhisperSTT(...)` — options are `base`, `small`, `medium`, `large-v3`,
  `turbo`.
- **Change the assistant's personality:** edit the `instructions=` string
  passed to `Agent(...)`.

## License

Add a license of your choice (e.g. MIT) before publishing this repository.
