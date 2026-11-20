# Silencia 1.13.29 

**Windows desktop application for call-center agents — real-time AI noise suppression, transcription, and a fully local, document-grounded knowledge base with verified answers.**

Silencia runs 100% on the workstation: the audio never leaves the machine, the imported documents never leave the machine, and the AI models (noise suppression, embeddings, chat LLM) execute locally. There is no cloud service, no telemetry, and no account.

- Current version: **1.13.29 (FIX 82)**
- Platform: Windows 10/11 (Electron desktop application)
- License: MIT (application code)
- Companion documents: `LISEZMOI.md` (French), `Silencia-1.13.29-Rapport-detaille-Guide-et-Securite.pdf` (detailed report, user guide and security report, in English)

---

## 1. What Silencia does

Silencia sits between the agent's headset and the call-center workstation and provides four permanent services:

1. **Real-time noise suppression** — the microphone signal is cleaned by a two-stage local AI chain (RNNoise + DTLN) with three intensity modes, acoustic echo cancellation (AEC), adaptive notch filtering, AGC and a broadcast-grade limiter. Target loudness: −15.6 LUFS; limiter ceiling −2.85 dB (ratio 20:1); added latency 24 ms.
2. **Transcription and voice services** — local speech-to-text (Whisper) with subtitles and a text corrector, and text-to-speech through the Windows SAPI voices.
3. **Virtual cable routing** — a VB-Cable virtual device connects Silencia's cleaned audio to conferencing software; the installer can deploy it, and since FIX 82 this happens **only on an explicit click**, never automatically at startup.
4. **Knowledge base and document chat (Chat RAG)** — import PDF, DOCX, XLSX/XLS, CSV, TXT and MD files; ask questions in French or English; receive answers that are either written from the retrieved passages (when the local LLM is available) or given as verbatim sourced extracts (fallback mode). Every answer cites its source: file name and page (or "document" for single-page kinds, "page N (image)" for OCR'd figures).

## 2. What's new in 1.13.29 (FIX 82)

**Total suppression of first-launch popups.** Since early versions, the virtual cable was installed automatically at startup (two automatic triggers in the renderer), which surfaced a Windows UAC prompt without any user click. FIX 82 removes all three automatic triggers:

- no automatic cable installation 1.2 s after the "cable required" banner;
- no automatic repair of a broken device branch (v1.10.7 behavior);
- no second trigger 2.5 s after boot.

The application now starts **directly** (window + audio pipeline at 350 ms, as before), with **no UAC, no installer window, no popup at startup**. Installation and repair of the cable remain available at any time from the banner buttons in the window — the UAC prompt then appears only after a click. Informative banners inside the window, microphone/headset/listening toasts and honest error boxes (startup impossible, unexpected error) are preserved. All 47 other body files of the asar are bit-identical to 1.13.28.

## 3. Feature overview

### 3.1 Real-time audio engine (verified bit-identical across 1.13.23 → 1.13.29)

| Stage | Detail |
|---|---|
| Pre-cleanup | RNNoise v2 compiled to WASM — 48 kHz, int16 |
| Deep denoising | DTLN, two ONNX stages — 16 kHz, analysis window 512 / hop 128, latency 24 ms |
| Spectral engine | STFT 512 with WOLA synthesis, Wiener filtering |
| Hum control | Adaptive NOTCH filter, 250–600 Hz |
| Dynamics | AGC (targets 0.1117 / 0.1061), EQ, limiter 20:1 at −2.85 dB |
| Output anchors | Makeup gain 0.994, −2.70 dBFS, −15.6 LUFS, −2.6 dBTP |
| Modes | `normal` / `strong` / `max` — default `max`, suppression ON, AEC ON, local listening OFF (anti-Larsen lock at launch) |

The engine files (`dsp-worklet.js`, `nn-engine.js`, `rnnoise.wasm`, `dsp-host.js` outside the asar) are covered by an MD5 integrity baseline and were verified bit-identical between 1.13.23 and 1.13.25 on 10 generated audio files; no FIX since then has touched them.

### 3.2 Knowledge base and Chat RAG

| Component | Behavior |
|---|---|
| Formats | PDF (pdfjs-dist, page by page, max 500 pages), DOCX (mammoth, single page), XLSX/XLS (SheetJS, one page per sheet, max 20 sheets × 5000 rows), CSV/TXT/MD (UTF-8 with Windows-1252 fallback) |
| PDF images (FIX 79) | Embedded images are extracted (XObject/inline → PNG), size-filtered (≥ 32 px, ≤ 8000 px, ≤ 6 Mpx, ≤ 250 per document), deduplicated by SHA1, then OCR'd locally; scanned PDFs without a text layer are no longer rejected — the OCR reconstructs the content. Sources are cited as "page N (image)". |
| Chunking | ~800 characters, 120 overlap, short chunks merged; limits 4000 chunks/document and 20000 per base |
| Indexing | Double index: pure-JS BM25 (k1 = 1.4, b = 0.75, compact French anti-lexicon) + BGE-M3 vectors (1024 dims) stored as a flat Float32 matrix |
| Retrieval | Hybrid fusion 0.6 × vector + 0.4 × normalized BM25, demanding relevance thresholds |
| Guardian (rag-v3.js, FIX 74/75/77/78) | Before writing, the retrieved passages must pass a gatekeeper (cosine + BM25 + coverage + identifier-existence rule: a distinctive token of the question such as a reference code must exist in the retrieved passages, otherwise the answer is refused) |
| Validation (validateAnswer v2) | Post-generation control: contradiction refusal and numeric checks against the source chunks |
| Answer modes | MODE 1 — written answer by Qwen2.5-3B when available; MODE 2 — verbatim sourced extraction fallback (always available, 100% faithful to the index) |
| Reindexing | Instant: OCR texts are cached per document (keyed by image hash), so re-embedding never re-runs OCR |

### 3.3 Local AI models

| Model | Role | Provenance | License |
|---|---|---|---|
| `qwen2.5:3b` | Chat generation (MODE 1) | Alibaba Qwen team, via the Ollama library | Qwen Research License (non-commercial research use) |
| `bge-m3` | Embeddings, 1024 dims | BAAI (Beijing Academy of Artificial Intelligence), via Ollama | MIT |
| `dtln1.onnx` + `dtln2.onnx` | Two-stage speech enhancement | DTLN — N. Westhausen & B. Meyer, University of Hamburg, INTERSPEECH 2020; ONNX export | academic/open source |
| `rnnoise.wasm` | Pre-denoising stage | Xiph.Org / Mozilla (J.-M. Valin) | BSD 3-Clause |

Ollama runs strictly on `127.0.0.1:11434`, is started automatically by Silencia at **BelowNormal** priority so it can never starve the audio thread, and can be installed silently from inside the application (official Ollama build only).

## 4. Installation

1. Run `Silencia-Setup-1.13.29.exe`:
   - double-click for the interactive installation; or
   - `Silencia-Setup-1.13.29.exe /S` for silent installation; or
   - `Silencia-Setup-1.13.29.exe /S /D="D:\Silencia"` for a custom directory.
   The installer closes any running Silencia, writes the files, verifies the engine, then launches Silencia (checkbox on the final page; also in `/S` mode).
2. Verify: title bar = "Silencia v1.13.29", permanent badge bottom-left = "Silencia v1.13.29", `version.txt` next to `Silencia.exe`.
3. **First launch (FIX 82): nothing pops up.** The window and the audio pipeline are up in ~350 ms. If the VB-Cable is not installed yet, an informative banner appears inside the window; installation happens only when you click the banner's button, and only the standard UAC prompt is shown (no CMD/PowerShell window, no driver wizard).

Ollama and the two text models (`qwen2.5:3b`, `bge-m3`) are **not** bundled: the Knowledge Base tab offers a guided silent installation, or a pull of the models, on demand. Until they are present, the chat still answers in MODE 2 (verbatim sourced extraction), so the feature is never dead.

## 5. Quick user guide

### Audio
- Select the headset microphone (Silencia auto-selects it when present) and pick the mode: `normal`, `strong` (strong suppression, default for open spaces) or `max` (maximum suppression, default).
- "I listen to the environment" (local monitoring) stays **off** at launch by design (anti-Larsen): enable it only with headphones, never with speakers. Silencia refuses to monitor to speakers or to the virtual cable.
- If a residual hiss/echo remains, check Windows "Listen to this device" (microphone Properties — Listen tab: uncheck) and the headset hardware sidetone: these bypass Silencia and cannot be disabled by any documented Windows API.

### Transcription and speech
- Speech is transcribed locally (Whisper) with subtitles and the corrector; SAPI voices read texts aloud (`sapi:voices` / `sapi:speak`).

### Knowledge base and Chat RAG
1. Open the "Knowledge base" tab.
2. Click **Import** and choose a PDF, DOCX, XLSX, CSV, TXT or MD file. Progress is shown (extract → index → embed). The status must end at `ok` with the chunk count.
3. Ask questions in French or English in the chat area. Successful answers show the cited source ("file.pdf, page 2"; "page 3 (image)" for OCR'd figures; "document" for DOCX/CSV/TXT). When the guardian refuses (question off-corpus or insufficient evidence), the answer is an explicit refusal — this is the anti-hallucination behavior working, not a failure.
4. Use **Reindex** after changing a document on disk (e.g., after a mammoth/DOCX or parser improvement) — reindexing is instant thanks to the OCR cache. Duplicates are refused by content hash ("already imported").
5. The AI badge shows the Ollama status; buttons offer the guided install or the model pull.

### Installer switches and logs
- `/S` silent, `/D="path"` custom directory.
- Logs: `main.log` in the user profile (interface traces via `diag:log`, cable install log via `cable:openlog`).

## 6. Verification record (highlights)

- **1.13.23 baseline audit (user-run bench, 23 questions)**: 14/23 conformant. It identified the Q15 hallucination class (identifier of one document falsely attributed to an off-corpus question), the over-strict cosine gate on true facts, and the over-permissive 0.451 threshold on traps.
- **FIX 77/78 (1.13.24/1.13.25)**: gatekeeper v2 (identifier-existence rule directly targets the Q15 class), hybrid context guarantee (lexical rescue when BM25/coverage are strong — the Q1–Q3/Q18/Q19 refusal class), validateAnswer v2 (contradiction refusal, numeric checks), and a batch vector-index fix in `rag.js`.
- **36-question bilingual bench (1.13.23/1.13.25 line)**: 36/36, with the Ollama backend exercised for real (no simulation). The trap `CC-7749` (absent from every chunk) is refused.
- **Audio regression control**: 10 generated audio files + `dsp-host.js` MD5-identical between 1.13.23 and 1.13.25; the audio engine has been untouched since.

## 7. Security at a glance (details in the PDF)

- **Context isolation**: the renderer talks to the main process only through the `contextBridge` (`preload.js`); no Node access in the renderer.
- **IPC surface**: ~45 whitelisted channels (`app:*`, `settings:*`, `cable:*`, `sapi:*`, `asr:*`, `nn:*`, `nn2:*`, `ai:*`, `rag:*`, `chat:*`), all app-internal.
- **Network**: the only endpoint is the local Ollama API on `127.0.0.1:11434`; the optional Ollama silent install downloads the official build; external URL opening is whitelist-controlled (`ai:openUrl`). No telemetry, no analytics, no remote content loading, no `eval`.
- **Data locality**: document parsing is 100% local ("aucune donnée envoyée nulle part" — docparse.js); the knowledge base lives in `%APPDATA%\Silencia\rag`.
- **Integrity**: an MD5 baseline of 1,917 delivered files pins the build; the three bundled audio-model files are MD5-pinned; Ollama blobs are sha256-named by the store.
- **VirusTotal**: see Part E of the PDF — method, constraints (650 MB web-upload limit, common Electron false positives, public-sample caveat) and the honest status of per-file verdicts.

## 8. Version history (1.13.x line)

| FIX | Version | Subject |
|---|---|---|
| 72 | 1.13.20 | Knowledge base & Chat RAG restored from the v1.10.64 system (rag.js + docparse.js + ollama.js, IPC `rag:*`/`chat:*`/`ai:*`), replacing the retired 1.13.x kb/* stack |
| 74/75 | — | Guardian V3 (`rag-v3.js`): cosine/BM25/coverage gate |
| 76 | 1.13.23 | Bilingual chat validated (36/36), Ollama audit, no simulation |
| 77 | 1.13.24 | Guardian v2 — ChatRAG hardening |
| 78 | 1.13.25 | Hybrid context guarantee + validateAnswer v2 + batch vector index fix |
| 79 | 1.13.26 | PDF image exploitation: image extraction + local OCR + "page N (image)" citations; scanned PDFs accepted |
| 80 | 1.13.27 | Installer fix |
| 81 | 1.13.28 | Display fix (imported-file display offset) |
| 82 | 1.13.29 | Total suppression of first-launch popups (no UAC/installer at startup; cable install only on explicit click) |

## 9. Repository layout (application)

```
Silencia/
├── Silencia.exe, version.txt          # Electron shell + version marker
├── resources/app.asar                 # application body (see below)
├── resources/app.asar.unpacked/       # files kept outside the asar
└── renderer/dsp/                      # audio engine (bit-identical, MD5-pinned)
    ├── dsp-worklet.js, nn-engine.js, dsp-core.js
    ├── dsp-host.js (unpacked, outside the asar)
    ├── models/dtln1.onnx, models/dtln2.onnx
    └── rnnoise.wasm
```

Inside the asar: `main.js` (Electron main process, IPC, Ollama client, installer helpers), `preload.js` (contextBridge), `renderer/` (UI, `app.js`, `rag-ui.js`, audio worklet wiring), `rag.js` (RagStore: chunking, BM25, vectors, import/reindex), `rag-v3.js` (guardian), `docparse.js` (PDF/DOCX/XLSX/CSV/TXT/MD + PDF image extraction), `ocr.js` (local OCR), `ollama.js` (localhost client), `kb-config.json`, `styles.css`.

## 10. Known limitations (stated honestly)

- **DOCX**: mammoth extracts raw text only — one single "document" page; tables become plain text and embedded images are not extracted (the FIX 79 image/OCR pipeline applies to PDF only). A correction of mammoth affects only the DOCX import/reindex/Chat-RAG path.
- **UI extract length**: source excerpts are truncated for display; on single-passage documents (DOCX), facts at the end of the passage may not be visible in "Sources used" while still being in the index and the answer.
- **Qwen2.5-3B license**: research license — for commercial call-center deployments, review the Qwen license terms or swap to a commercially licensed model via Ollama.
- **VirusTotal**: the multi-GB Ollama models and the large installer exceed the comfort zone of the VirusTotal web upload (650 MB); integrity is instead guaranteed by MD5/sha256 pinning and official-source provenance. See the PDF, Part E.
