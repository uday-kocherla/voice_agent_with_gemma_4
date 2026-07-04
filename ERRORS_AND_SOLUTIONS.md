# Errors & Solutions Log

A running record of every issue hit while building the local voice agent
(Gemma 4 + Ollama + LiveKit + faster-whisper + Qwen3-TTS) on a Kaggle GPU
notebook, and exactly how each one was resolved. Grouped by the phase of the
build where they showed up.

---

## Phase 1 — Missing System Tools

The base container is minimal; several OS-level tools that Ollama and the
audio pipeline assume exist had to be installed by hand.

### 1. Extraction failure — `zstd`
- **Error:** `ERROR: This version requires zstd for extraction.`
- **Root cause:** The Ollama install script ships its binaries compressed
  with Zstandard, but the container only had standard gzip.
- **Fix:** `sudo apt-get install -y zstd` before running the Ollama installer.

### 2. GPU blindness
- **Error:** `systemd is not running.` / `WARNING: Unable to detect NVIDIA/AMD GPU. Install lspci or lshw...`
- **Root cause:** Two issues at once — containers have no `systemd`, so Ollama
  can't register itself as a background service the normal way; and the
  container lacked the hardware-probing tools (`lspci`, `lshw`) needed to see
  the attached Tesla T4 GPUs.
- **Fix:** Installed `pciutils` and `lshw`. Bypassed the missing `systemd` by
  launching the server manually with Python's `subprocess.Popen()` instead of
  relying on a system service.

### 3. Audio processing failure — `sox`
- **Error:** `/bin/sh: 1: sox: not found` / `WARNING:sox:SoX could not be found!`
- **Root cause:** `faster-whisper` and voice-activity detection rely on SoX
  (Sound eXchange) to decode raw audio at the OS level, which wasn't present.
- **Fix:** Added `sox` and `libsox-fmt-all` to the apt-get install list.

---

## Phase 2 — Port Conflicts & Zombie Processes

Jupyter keeps memory and background processes alive even after a cell is
stopped, so re-running "start a server" cells caused collisions.

### 4. Ollama port blocked
- **Error:** `Error: listen tcp 127.0.0.1:11434: bind: address already in use`
- **Root cause:** A previous run of the cell already launched the Ollama
  server. Running it again tried to bind a second server to the same port,
  crashing and interrupting the model download mid-way.
- **Fix:** Force-killed the stuck process with `!fuser -k 11434/tcp`, and
  added a defensive `is_port_in_use()` check so the cell gracefully skips
  re-launching the server if one is already running.

### 5. LiveKit health server blocked
- **Error:** `OSError: [Errno 98] error while attempting to bind on address ('0.0.0.0', 8081): address already in use`
- **Root cause:** LiveKit runs an internal HTTP server on port 8081 for
  health checks. An earlier crashed run left that tiny server running.
- **Fix:** Configured `AgentServer` to explicitly use an alternate port
  (`port=8082`), permanently sidestepping the conflict.

---

## Phase 3 — Thread Blocking & the "Cold Start"

Running a large model on a GPU requires careful orchestration; blocking the
main Python thread makes LiveKit think the whole app crashed.

### 6. Unresponsive executor
- **Error:** `WARNING:livekit.agents:job executor is unresponsive... dropping pass-through signal` (alongside a `pulling manifest` log line)
- **Root cause:** Ollama was still downloading the multi-gigabyte Gemma model
  while LiveKit was trying to establish a real-time connection. The download
  blocked Python's execution thread, so LiveKit's required heartbeats never
  fired and it killed the connection.
- **Fix:** Decoupled the download from the live session entirely — `ollama
  pull gemma4` now runs in its own isolated cell, fully before the agent
  connects to anything.

### 7. Inference timeout
- **Error:** `APITimeoutError` (from `livekit.plugins.openai.llm.LLM`)
- **Root cause:** The "cold start penalty." The first time LiveKit asked the
  agent to say hello, Ollama had to physically move Gemma 4 from disk into
  GPU VRAM — a ~15 second operation that exceeded LiveKit's default patience,
  so it hung up on the LLM.
- **Fix:** Built a "pre-warmer" — a script that sends one silent, throwaway
  HTTP request to Ollama before LiveKit initializes, forcing the model into
  VRAM ahead of time so it's instantly ready when the live session starts.

### 8. Suboptimal VRAM performance
- **Log notice:** `OLLAMA_FLASH_ATTENTION:false`
- **Root cause:** Flash Attention — which noticeably speeds up token
  generation and reduces VRAM use — is disabled in Ollama by default.
- **Fix:** Set `os.environ["OLLAMA_FLASH_ATTENTION"] = "1"` in Python
  *before* spinning up the Ollama server subprocess (it has no effect if set
  after the server has already started).

---

## Phase 4 — LiveKit API & Syntax Quirks

The final round of issues came from version-specific quirks in the LiveKit
SDK itself.

### 9. Silero VAD deprecation
- **Error:** `DeprecationWarning: livekit-plugins-silero is deprecated...`
- **Root cause:** The LiveKit API now bundles voice-activity detection
  directly into `AgentSession`; explicitly importing and loading it
  separately is obsolete.
- **Fix:** Removed the standalone `silero` import from `livekit.plugins` and
  deleted the `vad=silero.VAD.load()` argument.

### 10. The ghost worker registry
- **Error:** `Exception: worker is already running`
- **Root cause:** The LiveKit SDK keeps a strict global registry in Python's
  process memory to guarantee only one worker runs per process. Stopping a
  notebook cell doesn't clear that in-memory flag.
- **Fix:** Full kernel restart (Kernel → Restart) to wipe the stuck process
  state before registering a fresh worker. Added `nest_asyncio.apply()`
  proactively to avoid related event-loop conflicts.

### 11. Sandbox connection failure
- **Issue:** The agent never appeared in the LiveKit Sandbox's participant
  list, with no console errors to explain why.
- **Root cause:** Two things at once — the LiveKit Sandbox uses *explicit
  dispatch* (it calls out for a specific agent name), but the code used an
  empty `@server.rtc_session()` decorator, which defaults to auto-dispatch.
  Separately, `await server.run()` was missing from the bottom of the script
  entirely, so the worker never actually started listening.
- **Fix:** Updated the decorator to `@server.rtc_session(agent_name="my-agent")`
  and added the explicit `await server.run()` call at the bottom of the
  execution block.

### 12. The bad parameter
- **Error:** `TypeError: LLM.with_ollama() got an unexpected keyword argument 'timeout'`
- **Root cause:** A `timeout` argument was passed into LiveKit's
  `with_ollama()` shortcut wrapper, but that wrapper is a thin convenience
  helper and doesn't accept raw HTTP-client kwargs like `timeout`.
- **Fix:** Removed the `timeout` parameter entirely. The "pre-warmer" from
  Error #7 already makes Ollama respond instantly, so a custom timeout was
  never actually necessary.

---

## Quick Reference

| # | Symptom | One-line fix |
|---|---|---|
| 1 | `requires zstd for extraction` | `apt-get install -y zstd` |
| 2 | GPU not detected | `apt-get install -y pciutils lshw`, launch via `subprocess.Popen` |
| 3 | `sox: not found` | `apt-get install -y sox libsox-fmt-all` |
| 4 | Ollama port 11434 in use | `is_port_in_use()` check / `fuser -k 11434/tcp` |
| 5 | LiveKit health port 8081 in use | `AgentServer(port=8082, ...)` |
| 6 | Job executor unresponsive during model pull | Pull the model in its own cell, before connecting |
| 7 | `APITimeoutError` on first reply | Pre-warm the model with a dummy request |
| 8 | Flash Attention disabled | `os.environ["OLLAMA_FLASH_ATTENTION"] = "1"` before server start |
| 9 | Silero deprecation warning | Drop the standalone `silero` import/`vad=` arg |
| 10 | `worker is already running` | Restart the kernel; add `nest_asyncio.apply()` |
| 11 | Agent missing from Sandbox | Set `agent_name=` explicitly; add `await server.run()` |
| 12 | `with_ollama() got an unexpected keyword argument 'timeout'` | Remove `timeout=`; rely on the pre-warmer instead |
