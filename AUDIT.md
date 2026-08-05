# Local AI Creation Stack — Technical Audit

This audit covers the four repositories cloned as submodules into `local-ai-creation-stack`:

| Tool | Submodule path | Public URL | Shallow clone commit tracked |
|---|---|---|---|
| HeyGem.ai | `HeyGem.ai/` | `https://github.com/suifeng9203/HeyGem.ai` | yes (`.gitmodules`) |
| ComfyUI | `ComfyUI/` | `https://github.com/Comfy-Org/ComfyUI` | yes (`.gitmodules`) |
| OmniVoice | `OmniVoice/` | `https://github.com/k2-fsa/OmniVoice` | yes (`.gitmodules`) |
| Handy | `Handy/` | `https://github.com/cjpais/Handy` | yes (`.gitmodules`) |

> **Visibility note:** This report and the surrounding meta-repo are kept private by default because they reference your local machine paths and audit notes. The submodules themselves point to public repositories. If you want the meta-repo public, create an empty public GitHub repo and add it as `origin`.

---

## 1. HeyGem.ai

### 1.1 Exact Repository Contents & Architecture

#### Directory structure
```
HeyGem.ai/
├── build/                  # Build artifacts / assets
├── deploy/                 # Docker Compose service definitions
│   ├── docker-compose.yml
│   ├── docker-compose-lite.yml
│   └── docker-compose-linux.yml
├── doc/                    # Documentation assets
├── resources/              # Packaged resources
├── src/
│   ├── main/               # Electron main-process code
│   │   └── service/        # API client modules
│   │       ├── model.js
│   │       ├── video.js
│   │       └── voice.js
│   ├── preload/            # Electron preload scripts
│   └── renderer/           # Vue 3 frontend (TDesign UI)
├── package.json            # Electron + Vue dependency manifest
├── electron-builder.yml    # Electron packager config
├── electron.vite.config.mjs # Vite / Electron build config
├── .eslintrc.cjs
├── .prettierrc.yaml
└── README.md / README_zh.md
```

#### Under-the-hood components
- **Frontend framework:** Vue 3 + `tdesign-vue-next` + `vue-router` + `vue-i18n`.
- **Desktop shell:** Electron 33 with `electron-vite` build pipeline.
- **Backend inference:** Three Docker services (not code in this repo, orchestrated from `deploy/`):
  - `guiji2025/fun-asr` — automatic speech recognition (ASR) based on FunASR.
  - `guiji2025/fish-speech-ziming` — text-to-speech / voice cloning (Fish Speech).
  - `guiji2025/heygem.ai` — talking-face / lip-sync video generation.
- **Local API client:** Electron main process calls the three Docker-exposed endpoints via `src/main/service/{model,video,voice}.js`.

#### Primary dependencies
| Category | Key items | Source file |
|---|---|---|
| Runtime | Node.js 18 | `README.md` |
| Electron | `electron ^33.0.0`, `electron-builder ^24.13.3`, `electron-vite ^2.3.0` | `package.json` |
| Frontend | `vue ^3.5.13`, `tdesign-vue-next ^1.10.3`, `vue-router ^4.4.5`, `vue-i18n ^10.0.5` | `package.json` |
| State | `pinia ^2.2.6` | `package.json` |
| Media | `fluent-ffmpeg ^2.1.3` | `package.json` |
| DB | `better-sqlite3 ^11.5.0` | `package.json` |
| HTTP | `axios ^1.7.7` | `package.json` |
| Desktop | `@electron-toolkit/preload`, `@electron-toolkit/utils` | `package.json` |

---

### 1.2 Functional Scope & Key Features

#### Core functionality
Fully offline digital-human video synthesis for Windows/Ubuntu:
1. **Appearance cloning** from a source video.
2. **Voice cloning** from a reference audio clip.
3. **Text- or audio-driven avatar video generation** with lip-sync.
4. Multi-language support: English, Japanese, Korean, Chinese, French, German, Arabic, Spanish.

#### Input / output specifications
| Step | Input | Output |
|---|---|---|
| Model training | Target video + reference `.wav` | Extracted audio placed at `D:\heygem_data\voice\data` |
| Preprocessing | Reference `.wav` (format field, language) | `asr_format_audio_url` + `reference_audio_text` |
| Audio synthesis | Speaker UUID, text, reference audio/text | Synthesized `.wav` (via `http://127.0.0.1:18180/v1/invoke`) |
| Video synthesis | `audio_url`, `video_url`, task `code` | Async lip-sync video (via `http://127.0.0.1:8383/easy/submit`) |

#### Execution modes
- **Server:** `docker-compose up -d` inside `deploy/` (full version) or `docker-compose -f docker-compose-lite.yml up -d` (lite).
- **Client:** `npm run build` then `electron-builder --win` (or `--linux`); or download the pre-built release.
- **Open API:** Direct `POST/GET` to `127.0.0.1:18180` and `127.0.0.1:8383` once Docker is running.

---

### 1.3 Technical Advantages & Performance Benefits

#### Privacy & security
- Runs entirely offline after Docker images are pulled.
- No cloud inference by default; video/audio data never leaves the machine unless the user opts into the commercial API.
- Electron camera/microphone permissions are declared in `electron-builder.yml` (`NSCameraUsageDescription`, `NSMicrophoneUsageDescription`).

#### Resource efficiency & cost
- Recommended spec: 13th-gen Intel i5, **32 GB RAM**, **NVIDIA RTX 4070**, **>100 GB free disk** (C: for images, 30 GB+ D: for data).
- Docker full setup downloads ~70 GB of container data.
- Saves the cost of HeyGen / D-ID cloud credits but requires a local GPU server.

#### Customizability & extensibility
- Vue/Electron frontend can be restyled and extended.
- Three services expose REST endpoints (`src/main/service/*.js`) so any external script can call them.
- `docker-compose.yml` can be edited to remap data paths and ports.

---

### 1.4 Practical Capabilities & Real-World Use Cases

#### End-to-end capabilities
- Record a 1-minute talking-head video → extract face/appearance → clone voice → generate unlimited localized marketing videos from text.
- Batch avatar video production without subscription fees.

#### Developer / power-user automation
```bash
cd deploy
docker-compose up -d
curl -X POST http://127.0.0.1:18180/v1/invoke \
  -H "Content-Type: application/json" \
  -d '{"speaker":"uuid-123","text":"Hello world","format":"wav","reference_audio":"/data/ref.wav","reference_text":"hello world"}'

curl -X POST http://127.0.0.1:8383/easy/submit \
  -H "Content-Type: application/json" \
  -d '{"audio_url":"/out/uuid-123.wav","video_url":"/data/model.mp4","code":"uuid-123"}'

curl "http://127.0.0.1:8383/easy/query?code=uuid-123"
```

---

### 1.5 Joint Integration Synergy & Co-Working Plan

#### Repository orchestration role
HeyGem.ai sits at the **end of the production pipeline**: it consumes audio from OmniVoice/Handy and produces the final lip-sync avatar video. ComfyUI can generate still/video backgrounds or props that are composited with the avatar clip.

#### Implementation blueprint for this repo
- **Input:** `audio.wav` produced by OmniVoice (or directly by Handy dictation) + a `model.mp4` base avatar video.
- **Transform:** `POST /v1/invoke` for audio synthesis, `POST /easy/submit` + `GET /easy/query` for video synthesis.
- **Output:** `output.mp4` talking-head video.

---

## 2. ComfyUI

### 2.1 Exact Repository Contents & Architecture

#### Directory structure
```
ComfyUI/
├── app/                    # FastAPI/ASGI app layer, DB, assets
├── comfy/                  # Core diffusion / model / sampler modules
├── comfy_api/              # REST + WebSocket API definitions
├── comfy_api_nodes/        # API-compatible node implementations
├── comfy_execution/        # Graph executor and scheduling
├── comfy_extras/           # Extra nodes and utilities
├── custom_nodes/           # Community / user-installed nodes
├── models/                 # Checkpoint / VAE / LoRA / controlnet / etc.
├── input/                  # User-provided input media
├── output/                 # Generated media
├── script_examples/        # API usage examples
├── tests/                  # Integration tests
├── tests-unit/             # Unit tests
├── main.py                 # CLI entry point
├── server.py               # HTTP/WebSocket server
├── execution.py            # Graph execution engine
├── nodes.py                # Built-in node definitions
├── folder_paths.py         # Path resolution helpers
├── requirements.txt        # Python dependencies
└── pyproject.toml          # Project metadata
```

#### Under-the-hood components
- **Core engine:** PyTorch-based node graph executor for diffusion, video, audio, 3D and text models.
- **Frontend:** Separate `Comfy-Org/ComfyUI_frontend` package pulled from PyPI (`comfyui-frontend-package==1.48.6`) and served by `server.py`.
- **API layer:** FastAPI/ASGI with WebSocket `/ws` for real-time progress and OpenAPI spec (`openapi.yaml`).
- **Model formats:** `safetensors`, `ckpt`, LoRA, ControlNet, VAE, CLIP, T2I adapters, plus external nodes for hundreds of models.
- **Acceleration:** CUDA/ROCm/Intel XPU/Apple Metal via PyTorch; optional `--preview-method taesd` for faster latent previews.

#### Primary dependencies
| Category | Key items | Source file |
|---|---|---|
| Deep learning | `torch`, `torchvision`, `torchaudio`, `transformers>=4.50.3`, `safetensors`, `einops`, `kornia` | `requirements.txt` |
| Diffusion / VAE | Custom `comfy/` modules, `k_diffusion`, `ldm`, `taesd`, `clip_*` | `comfy/` tree |
| Server | `aiohttp>=3.11.8`, `yarl>=1.18.0` | `requirements.txt` |
| DB / migrations | `SQLAlchemy>=2.0.0`, `alembic`, `comfy-aimdo==0.4.13` | `requirements.txt` |
| UI packages | `comfyui-frontend-package==1.48.6`, `comfyui-workflow-templates==0.11.31` | `requirements.txt` |
| Python | `>=3.10` | `pyproject.toml` |
| Version | `0.30.0` | `comfyui_version.py`, `pyproject.toml` |

---

### 2.2 Functional Scope & Key Features

#### Core functionality
Generate and manipulate images, videos, 3D assets, audio and text through a node-graph UI or JSON API. Supports Stable Diffusion 1.5/SDXL/SD3.5/Flux, Wan/LTX/HunyuanVideo, CogVideoX, Stable Audio, Hunyuan3D, SAM, Depth Anything, and many more via custom nodes.

#### Input / output specifications
| Input | Format | Notes |
|---|---|---|
| Text prompts | UTF-8 strings | Supports dynamic syntax `(word:1.2)` and wildcards `{a|b}` |
| Images | PNG/JPEG/WebP | Used as seeds, references, inpainting masks, ControlNet sources |
| Models | `.safetensors`, `.ckpt`, LoRA, VAE, etc. | Placed under `models/` subdirectories |
| Workflows | JSON | Save/load full node graphs, including embedded metadata |
| Output | PNG/JPEG/WebP/MP4/WAV/etc. | Written to `output/` |

#### Execution modes
- **CLI:** `python main.py` (with many flags).
- **API server:** `python main.py --listen 0.0.0.0 --port 8188`.
- **Manager mode:** `python main.py --enable-manager`.
- **Desktop:** official ComfyUI Desktop app (separate repo).
- **Container:** community Docker images (not bundled in this repo).

#### Key CLI flags (from README and `main.py` / `comfy/cli_args`)
```bash
python main.py --listen 0.0.0.0 --port 8188
python main.py --preview-method auto
python main.py --preview-method taesd
python main.py --disable-api-nodes        # force offline, disable paid API nodes
python main.py --enable-manager
python main.py --tls-keyfile key.pem --tls-certfile cert.pem
```

---

### 2.3 Technical Advantages & Performance Benefits

#### Privacy & security
- Core ComfyUI runs fully offline; it does not download models unless explicitly requested.
- `--disable-api-nodes` blocks the optional paid cloud Comfy API nodes.
- Local model storage means prompts and generated media never leave the workstation.

#### Resource efficiency & cost
- Smart VRAM/RAM management: model offloading, partial graph re-execution, async queueing.
- Quantized model support lowers VRAM requirements.
- TAESD fast latent preview reduces preview compute.
- Cost avoidance vs. Midjourney / Runway / RunDiffusion cloud credits.

#### Customizability & extensibility
- **Custom nodes:** drop-in Python packages under `custom_nodes/`; the `ComfyUI-Manager` can install/update them.
- **JSON workflows:** every graph is a JSON file that can be versioned, shared and loaded.
- **Subgraphs:** reusable node groups and templates.
- **Model paths:** configurable via `extra_model_paths.yaml` (example in `extra_model_paths.yaml.example`).

---

### 2.4 Practical Capabilities & Real-World Use Cases

#### End-to-end capabilities
- Text-to-image / image-to-image / inpainting / outpainting / video-to-video.
- Product photography variations, marketing asset generation, character concepts, background plates.
- Compositing: green-screen replacement, depth-based relighting, face restoration, upscaling.

#### Developer / power-user automation
```bash
# Start server with API
python main.py --listen 0.0.0.0 --port 8188 --enable-manager

# Queue a workflow via the API
python script_examples/basic_api_example.py
# or POST /prompt with a JSON workflow payload
```
Saved PNGs contain the full workflow metadata; dropping a generated image onto the canvas restores the graph.

---

### 2.5 Joint Integration Synergy & Co-Working Plan

#### Repository orchestration role
ComfyUI is the **visual asset factory** of the stack. It produces backgrounds, props, stylized frames and even audio/video clips that feed into HeyGem.ai or are combined with OmniVoice/Handy outputs.

#### Implementation blueprint for this repo
- **Input:** Text/image prompts or a JSON workflow.
- **Transform:** `POST /prompt` (when `server.py` is running) or `python main.py --workflow workflow.json`.
- **Output:** `output/image_00001.png` or `output/video_00001.mp4`.
- **Downstream:** Use the generated frame as a background for a HeyGem avatar clip, or generate a style reference image for a ComfyUI video-to-video workflow.

---

## 3. OmniVoice

### 3.1 Exact Repository Contents & Architecture

#### Directory structure
```
OmniVoice/
├── omnivoice/
│   ├── __init__.py
│   ├── cli/                # Command-line entry points
│   │   ├── demo.py         # Gradio web demo
│   │   ├── infer.py        # Single inference
│   │   ├── infer_batch.py  # Multi-GPU batch
│   │   └── merge_lora.py   # LoRA merge helper
│   ├── data/               # Dataset helpers
│   ├── models/             # Model definitions and hub helpers
│   ├── scripts/            # Training / evaluation scripts
│   ├── training/           # Trainer code
│   └── utils/              # Audio / text utilities
├── examples/               # Example pipelines
├── docs/                   # Language lists, tips, parameters
├── tests/                  # Unit tests
├── pyproject.toml          # Package metadata and deps
└── uv.lock                 # Locked dependency tree
```

#### Under-the-hood components
- **Model:** Massively multilingual zero-shot TTS based on a **diffusion language model** architecture (arXiv:2604.00688).
- **Vocoder / waveform:** `torchaudio` + `soundfile` for 24 kHz output.
- **ASR (optional):** Uses a Whisper model to auto-transcribe `ref_audio` when `ref_text` is omitted.
- **Packaging:** `hatchling` build backend; installable via `pip install omnivoice` or `pip install -e .`.
- **Lockfile:** `uv.lock` indicates uv is the recommended workflow.

#### Primary dependencies
| Category | Key items | Source file |
|---|---|---|
| Deep learning | `torch>=2.4`, `torchaudio>=2.4`, `transformers>=5.3.0`, `accelerate` | `pyproject.toml` |
| Audio | `pydub`, `soundfile`, `librosa` | `pyproject.toml` |
| UI | `gradio` | `pyproject.toml` |
| Training | `tensorboardX`, `webdataset` | `pyproject.toml` |
| Text norm (optional) | `WeTextProcessing`, `num2words` (`pip install "omnivoice[tn]"`) | `pyproject.toml` |
| LoRA (optional) | `peft>=0.20.0` (`pip install "omnivoice[lora]"`) | `pyproject.toml` |
| Python | `>=3.10` | `pyproject.toml` |

---

### 3.2 Functional Scope & Key Features

#### Core functionality
- **600+ language** zero-shot text-to-speech.
- **Voice cloning** from a 3–10 second reference audio.
- **Voice design** from speaker attributes (gender, age, pitch, accent, whisper style).
- **Auto voice** — model picks a random suitable voice.
- **Non-verbal symbols** and phoneme/pinyin pronunciation control.

#### Input / output specifications
| Mode | Input | Output |
|---|---|---|
| Voice cloning | `text`, `ref_audio` (`ref_text` optional) | `np.ndarray` at 24 kHz mono PCM |
| Voice design | `text`, `instruct="female, low pitch, british"` | `np.ndarray` at 24 kHz mono PCM |
| Auto voice | `text` | `np.ndarray` at 24 kHz mono PCM |
| Prompt reuse | `omnivoice.create_voice_clone_prompt(...).save("voice.pt")` | `.pt` prompt file |
| Batch | JSONL with `id`, `text`, optional `ref_audio`/`ref_text`/`instruct` | Folder of `.wav` files |

#### Execution modes
- **Python API:** `from omnivoice import OmniVoice; model.from_pretrained(...)`.
- **CLI single:** `omnivoice-infer`.
- **CLI batch:** `omnivoice-infer-batch`.
- **Web UI:** `omnivoice-demo --ip 0.0.0.0 --port 8001`.
- **Notebook:** `docs/OmniVoice.ipynb`.

#### Key CLI flags
```bash
omnivoice-demo --ip 0.0.0.0 --port 8001
omnivoice-infer --model k2-fsa/OmniVoice --text "Hello" --ref_audio ref.wav --ref_text "hello" --output out.wav
omnivoice-infer-batch --model k2-fsa/OmniVoice --test_list test.jsonl --res_dir results/
```

---

### 3.3 Technical Advantages & Performance Benefits

#### Privacy & security
- Models and inference run locally once downloaded from HuggingFace.
- No cloud TTS provider receives text or voice samples.
- Optional `asr_model_name` and `asr_device` allow pinning Whisper on a specific local GPU/CPU.

#### Resource efficiency & cost
- RTF as low as **0.025** on an H100 (40x real-time); realistically fast on modern NVIDIA GPUs.
- FlashInfer kernels can accelerate **2–2.9x** for batched inference.
- Saves per-character cloud TTS/ voice-cloning charges (ElevenLabs, Azure, etc.).

#### Customizability & extensibility
- **LoRA fine-tuning:** `omnivoice[lora]` extra + `omnivoice-merge-lora` CLI.
- **Voice design attributes** are free-form strings.
- **Custom reference prompts** can be saved/loaded (`VoiceClonePrompt`) to avoid re-encoding.
- **Control tags** like `[laughter]`, `[sigh]`, and phoneme overrides can be embedded in text.

---

### 3.4 Practical Capabilities & Real-World Use Cases

#### End-to-end capabilities
- Clone a CEO's voice once, then generate localized earnings-call audio in 10 languages.
- Produce consistent audiobook narration by saving and reusing voice prompts.
- Generate expressive character voices with inline `[laughter]` tags for games/animation.

#### Developer / power-user automation
```python
from omnivoice import OmniVoice
import soundfile as sf
import torch

model = OmniVoice.from_pretrained("k2-fsa/OmniVoice", device_map="cuda:0", dtype=torch.float16)
prompt = model.create_voice_clone_prompt(ref_audio="voice.wav", ref_text="Hello there")
prompt.save("brand_voice.pt")

audio = model.generate(text="Welcome to the demo.", voice_clone_prompt=prompt, speed=1.0, num_step=32)
sf.write("welcome.wav", audio[0], 24000)
```

---

### 3.5 Joint Integration Synergy & Co-Working Plan

#### Repository orchestration role
OmniVoice is the **speech generator** of the stack. It consumes text (from Handy STT or a script) and a reference voice, then emits the `.wav` that drives HeyGem.ai's avatar.

#### Implementation blueprint for this repo
- **Input:** `transcript.txt` (from Handy) + `ref.wav` or a saved `voice.pt`.
- **Transform:** `omnivoice-infer --text "..." --ref_audio ref.wav --output narration.wav`.
- **Output:** `narration.wav` (24 kHz).
- **Downstream:** Send `narration.wav` to HeyGem.ai `http://127.0.0.1:8383/easy/submit` together with the target avatar video.

---

## 4. Handy

### 4.1 Exact Repository Contents & Architecture

#### Directory structure
```
Handy/
├── src/                    # React + TypeScript frontend
│   ├── components/
│   ├── hooks/
│   ├── stores/
│   └── types/
├── src-tauri/              # Rust backend (Tauri v2)
│   ├── src/                # Rust source
│   ├── Cargo.toml          # Rust dependencies
│   ├── tauri.conf.json     # Tauri app config
│   ├── build.rs            # Build script for ONNX/Whisper libs
│   └── capabilities/
├── scripts/                # Build / translation checks
├── tests/                  # Playwright tests
├── public/                 # Static web assets
├── package.json            # Node frontend dependencies
├── BUILD.md                # Build instructions
├── flake.nix               # Nix flake for reproducible builds
└── README.md
```

#### Under-the-hood components
- **Frontend:** React 18 + TypeScript + Vite + Tailwind CSS v4.
- **Desktop shell:** Tauri v2 (Rust) with `tauri-specta` for type-safe commands.
- **Speech recognition backends:**
  - `transcribe-cpp` (Rust crate) — Whisper-family models via `ggml/gguf` with Vulkan/Metal/CPU backends.
  - `transcribe-rs` (Rust crate) — ONNX runtime for Parakeet, Moonshine, SenseVoice, GigaAM, Canary, Cohere.
- **Audio pipeline:** `cpal` capture, `vad-rs` voice-activity detection, `rubato` resampling, `hound` WAV writing.
- **System integration:** `rdev` global hotkeys, `enigo` text injection, `rodio` audio playback.

#### Primary dependencies
| Category | Key items | Source file |
|---|---|---|
| Frontend | `react ^18.3.1`, `vite ^6.4.1`, `tailwindcss ^4.1.16`, `@tauri-apps/api ^2.11.0` | `package.json` |
| Backend | `tauri = "2.11.5"`, `tokio`, `cpal`, `rdev`, `enigo`, `rubato`, `hound` | `src-tauri/Cargo.toml` |
| ASR | `transcribe-cpp`, `transcribe-rs` (ONNX) | `src-tauri/Cargo.toml` |
| Storage | `rusqlite ^0.37` (bundled), `tauri-plugin-store` | `src-tauri/Cargo.toml` |
| Download | `hf-hub` (cancellable downloads), `reqwest` | `src-tauri/Cargo.toml` |
| Models | Whisper GGML `.bin`, Parakeet `.gguf`/`.tar.gz` | `README.md` |
| Build | `bun` or `pnpm` + Rust stable + `cargo tauri` | `BUILD.md` |

---

### 4.2 Functional Scope & Key Features

#### Core functionality
Offline desktop speech-to-text. Press a global shortcut, speak, release, and Handy pastes the transcription into the active text field.

#### Input / output specifications
| Step | Input | Output |
|---|---|---|
| Audio capture | Microphone via `cpal` | Raw PCM buffer |
| VAD | PCM buffer | Voice-segmented audio |
| Transcription | Whisper/Parakeet model | Plain text |
| Paste | System keyboard focus | Typed text via `enigo` or `xdotool`/`wtype` |

#### Execution modes
- **Desktop GUI:** double-click the built `.app`/`.exe`/`.AppImage`.
- **CLI remote control:** `handy --toggle-transcription`, `handy --toggle-post-process`, `handy --cancel`.
- **Unix signal:** `pkill -USR2 -n handy` toggles recording.
- **Dev mode:** `bun run dev` (frontend) + `cargo tauri dev` (backend).

#### Key CLI flags
```bash
handy --toggle-transcription
handy --toggle-post-process
handy --cancel
handy --start-hidden
handy --no-tray
handy --debug
```

---

### 4.3 Technical Advantages & Performance Benefits

#### Privacy & security
- All audio processing and model execution are local.
- No cloud ASR service receives voice data.
- Voice Activity Detection runs locally with Silero.

#### Resource efficiency & cost
- **Parakeet V3** is CPU-optimized; runs ~5x real-time on mid-range i5 hardware.
- **Whisper models** can use Vulkan/Metal/CUDA (via `ggml` backends); 487 MB–1.6 GB model sizes.
- Saves per-minute cloud transcription costs (Whisper API, AssemblyAI, etc.).

#### Customizability & extensibility
- Drop custom Whisper GGML `.bin` files into the models directory.
- Configurable global shortcuts, paste behavior, overlay position.
- Raycast extension for external control.
- Rust backend can be extended with new Tauri commands.

---

### 4.4 Practical Capabilities & Real-World Use Cases

#### End-to-end capabilities
- Hands-free dictation into any editor, IDE, chat app, or form.
- Meeting transcription shortcut for note-taking apps.
- Trigger voice commands that paste standard phrases.

#### Developer / power-user automation
```bash
# Toggle recording from a window-manager binding
bindsym $mod+o exec pkill -USR2 -n handy
```
Or programmatically:
```bash
handy --toggle-transcription
# After recording, text is injected into the focused text field.
```

---

### 4.5 Joint Integration Synergy & Co-Working Plan

#### Repository orchestration role
Handy is the **voice trigger / transcript source** of the stack. It converts microphone input into text, which is then fed to OmniVoice for narration or used as a prompt for ComfyUI/HeyGem.

#### Implementation blueprint for this repo
- **Input:** Microphone audio.
- **Transform:** Global hotkey → VAD → ASR → clipboard / keystroke injection.
- **Output:** Plain text (e.g., "generate a space background" or the script for a video).
- **Downstream:** Write the transcript to a file; a watcher script forwards it to OmniVoice (`omnivoice-infer`) or to a ComfyUI `POST /prompt` request.

---

## 5. Cross-Repo Integration Blueprint (End-to-End Pipeline)

### 5.1 Conceptual flow
```
Microphone
    |
    v
[ Handy ]  --(plain text)-->  [ OmniVoice ]
    |                                    |
    | (text prompt for image/video)      v
    v                                 [ .wav audio ]
[ ComfyUI ]  --(background asset)-->   |
    |                                    v
    +--------------------------------> [ HeyGem.ai ]
                                           |
                                           v
                                    [ final avatar video ]
```

### 5.2 Step-by-step wiring

#### Step 1 — Trigger and transcribe with Handy
1. Press the configured global shortcut (or `handy --toggle-transcription`).
2. Speak the command, e.g. "Generate a 10-second product demo. Speaker is John. Script: welcome to our launch."
3. Handy transcribes and injects the raw text into a target file or clipboard.

#### Step 2 — Extract script and voice identity
Parse the transcript:
- Use the first sentence as a **system intent**.
- Extract the `Speaker is <name>` reference (a previously saved `.wav` sample).
- Extract the `Script: <text>` part as the TTS input.

#### Step 3 — Generate narration with OmniVoice
```bash
omnivoice-infer \
  --model k2-fsa/OmniVoice \
  --text "welcome to our launch" \
  --ref_audio speakers/john.wav \
  --ref_text "this is john speaking" \
  --output narration.wav
```

#### Step 4 — Generate the visual asset with ComfyUI
Create a workflow JSON that produces:
- A background image (`output/background_00001.png`), or
- A product/character image, or
- A short ambient video clip.

Queue it via the API:
```bash
curl -X POST http://127.0.0.1:8188/prompt \
  -H "Content-Type: application/json" \
  -d @workflow.json
```

#### Step 5 — Synthesize the avatar with HeyGem.ai
1. Place `narration.wav` in the HeyGem data folder (mapped in `deploy/docker-compose.yml`).
2. Pick or train a base avatar `model.mp4`.
3. Call the HeyGem services:
```bash
curl -X POST http://127.0.0.1:8383/easy/submit \
  -H "Content-Type: application/json" \
  -d '{"audio_url":"/data/narration.wav","video_url":"/data/model.mp4","code":"demo-001"}'

curl "http://127.0.0.1:8383/easy/query?code=demo-001"
```

#### Step 6 — Optional post-production
Combine the HeyGem `avatar.mp4` with the ComfyUI `background.png` in a video editor or using `ffmpeg` (ComfyUI/ffmpeg node).

---

## 6. Storage, RAM & GPU Cheat Sheet

| Tool | Min disk | Min RAM | Ideal GPU | Notes |
|---|---|---|---|---|
| Handy | ~1 GB (model cache) | 8 GB | Optional (Vulkan/Metal helps Whisper) | Parakeet V3 runs on CPU |
| OmniVoice | ~2–5 GB (model) | 16 GB | NVIDIA GPU (FP16) | 24 kHz mono output |
| ComfyUI | Model-dependent (10–100+ GB) | 16–32 GB | NVIDIA RTX with 8–24 GB VRAM | Quantized models reduce VRAM |
| HeyGem.ai | ~100+ GB Docker images | 32 GB | NVIDIA RTX 4070 class | Windows primary; Ubuntu beta |

---

## 7. File Index of Key References

| Repo | File | Why it matters |
|---|---|---|
| HeyGem.ai | `README.md` | API endpoints, Docker setup, hardware requirements |
| HeyGem.ai | `package.json` | Electron + Vue dependencies and build scripts |
| HeyGem.ai | `electron-builder.yml` | App packaging and platform settings |
| HeyGem.ai | `deploy/docker-compose.yml` | Server orchestration |
| HeyGem.ai | `src/main/service/{model,video,voice}.js` | Local API client code |
| ComfyUI | `README.md` | Features, install, CLI flags, model support |
| ComfyUI | `pyproject.toml` / `comfyui_version.py` | Version `0.30.0`, Python `>=3.10` |
| ComfyUI | `requirements.txt` | Python package list |
| ComfyUI | `main.py` / `server.py` | Entry points and HTTP/WebSocket server |
| OmniVoice | `README.md` | API, CLI, voice design, FlashInfer benchmarks |
| OmniVoice | `pyproject.toml` | Entry scripts and dependencies |
| Handy | `README.md` | Architecture, models, CLI flags, build notes |
| Handy | `package.json` | Frontend dependencies |
| Handy | `src-tauri/Cargo.toml` | Rust/Tauri backend dependencies |
| Handy | `src-tauri/tauri.conf.json` | App metadata and bundle config |
