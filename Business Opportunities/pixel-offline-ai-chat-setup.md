# Offline AI Chat on Pixel 10 Pro XL — Setup Guide

A practical guide to running a real chat AI on your Pixel 10 Pro XL with no internet required. Three tiers, from easiest to most powerful.

**Hardware target:** Pixel 10 Pro XL (Tensor G5, 16 GB RAM)
**Recommended model:** Gemma 4 E4B (Q4_K_M quant, ~5 GB)

---

## What This Gets You

A chat assistant that:
- Runs fully offline (airplane mode, hiking, power outages, flights)
- Costs $0 in API fees
- Keeps everything on-device (nothing leaves your phone)
- Performs roughly like "ChatGPT 3.5 in your pocket" — real assistant, not magic

**What it won't do:** match Claude or Gemini Pro on hard reasoning, long-context work, or current-events questions (training cutoff applies).

---

## Tier 1 — What You Already Have (Offline-Capable)

These work right now in airplane mode, no setup needed:

- **Pixel Screenshots** — OCR + Q&A on anything you've screenshotted
- **Recorder app** — transcription + summaries of audio
- **Live Translate** — pre-download language packs first (Settings → System → Languages → Live Translate)
- **Gboard proofread / rewrite** — long-press Gemini icon on keyboard
- **Call Notes, Call Screen, Hold for Me** — all Gemini Nano powered

**Not offline:** Gemini app, Claude.ai, Claude apps. These require cloud servers — no way around it.

---

## Tier 2 — Real Offline Chat (Recommended — 15 minutes)

### Option A: PocketPal AI (best general experience)

1. Install **PocketPal AI** from Google Play Store (free, open-source)
2. Open the app → tap "Download Model" or model picker
3. Search for `Gemma 4 E4B` or `Gemma 3 4B Instruct`
   - If E4B isn't in the catalog, add a Hugging Face URL directly
4. Pick the **Q4_K_M** quant (~3–5 GB download — do this on WiFi)
5. Load the model → start chatting

**Why Q4_K_M:** Best balance of quality and size for phone hardware. Higher quants (Q8, BF16) eat too much RAM and throttle the phone.

### Option B: Google AI Edge Gallery (best performance)

Google's official sample app — uses MediaPipe LLM Inference API and hits the Tensor G5 TPU properly (faster than CPU-based runtimes).

1. Sideload APK from Google's GitHub: `github.com/google-ai-edge/gallery`
2. Download a Gemma model from inside the app
3. Chat

**Tradeoff:** Bare-bones UI compared to PocketPal, but the most efficient on your hardware.

### Option C: Layla (paid, polished)

Persona support, voice input/output, nicer UI. Worth the money if you want a daily-driver feel.

### Realistic Performance Expectations

| Metric | What to expect |
|---|---|
| Speed (E4B Q4) | 15–25 tokens/sec |
| First-token latency | 1–3 seconds |
| Battery drain | ~5–8% per 10 min active use |
| Heat | Warm after 5 min, throttles after ~10 min sustained |
| Effective context | 8K–16K tokens before quality drops |

---

## Tier 3 — Termux + llama.cpp (Builder Path)

For when you want a real local API server on your phone — integrates with Hermes, Tasker, scripts, or anything that speaks OpenAI's API format.

### Install Termux

1. Install **Termux** from F-Droid (NOT Play Store version — it's outdated)
   - F-Droid: `f-droid.org/packages/com.termux/`
2. Open Termux, run:

```bash
pkg update && pkg upgrade -y
pkg install -y clang cmake git wget
```

### Build llama.cpp

```bash
cd ~
git clone https://github.com/ggerganov/llama.cpp
cd llama.cpp
make -j$(nproc)
```

Build takes 5–15 minutes depending on phone temperature.

### Download a Model

Get a GGUF file. Example with Gemma 4 E4B Q4_K_M (replace URL with actual HF link when E4B GGUFs are available):

```bash
cd ~/llama.cpp
mkdir models && cd models
wget https://huggingface.co/[publisher]/gemma-4-e4b-gguf/resolve/main/gemma-4-e4b-q4_k_m.gguf
```

For Gemma 3 4B (available now):

```bash
wget https://huggingface.co/google/gemma-3-4b-it-qat-q4_0-gguf/resolve/main/gemma-3-4b-it-q4_0.gguf
```

### Run the Server

```bash
cd ~/llama.cpp
./llama-server \
  -m models/gemma-3-4b-it-q4_0.gguf \
  --host 127.0.0.1 \
  --port 8080 \
  -c 8192
```

Server is now live at `http://localhost:8080` on your phone with an OpenAI-compatible API.

### Test It

In another Termux session (swipe from left edge → New session):

```bash
curl http://localhost:8080/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{
    "model": "gemma",
    "messages": [{"role": "user", "content": "Hello, who are you?"}]
  }'
```

### Keep It Running in Background

```bash
pkg install -y termux-services
# Acquire wake lock so Android doesn't kill the process
termux-wake-lock
```

To stop: `termux-wake-unlock`

---

## Integration Ideas (Tier 3+)

### Hermes Fallback Pattern

When Hostinger VPS is unreachable, route Hermes-bound prompts to the phone's local model instead:

- Set up Tailscale on the phone (already a builder pattern you use)
- Expose llama.cpp's port over Tailscale
- Add a fallback rule in Hermes: if VPS unreachable → try `http://[phone-tailscale-ip]:8080`
- Queue inputs locally when both are down, sync when one comes back

### Voice Input via Gboard

Gboard's voice typing works offline on Pixel. Dictate into PocketPal (or a Termux frontend) for Gemini Live-style voice chat, fully local.

### Pixel Screenshots → Local Model Pipeline

1. Screenshot a document
2. Pixel Screenshots OCRs it (on-device)
3. Copy text → paste into PocketPal
4. Ask deep questions about it
5. Everything offline, nothing on a server

### Tasker / Shortcut Hooks

With the llama.cpp server running, any Tasker action or HTTP-capable app can hit it. Build a home-screen shortcut that fires a prompt and reads the response aloud via Android TTS.

---

## What I'd Actually Do (Recommendation)

**Week 1:** Install PocketPal AI, download Gemma 4 E4B Q4_K_M, use it for a few days. Free, takes 15 minutes. Get a feel for capability.

**Week 2:** If you like it, install Google AI Edge Gallery alongside it to compare TPU-accelerated performance. The MediaPipe runtime is genuinely faster on E4B than CPU llama.cpp.

**Week 3+:** If you want integration, do the Termux setup. Wire it into Hermes as a fallback, or use it as an edge inference node when offline.

---

## Troubleshooting Notes

- **App says "not enough RAM":** Close other apps, or pick a smaller quant (Q4_0 instead of Q4_K_M, or drop to E2B)
- **Slow generation:** Phone is throttling — let it cool, or use Google AI Edge Gallery for TPU acceleration
- **Termux build fails:** Make sure you're using the F-Droid version, not Play Store. Run `pkg upgrade` first.
- **Model gives bad output:** Try a different quant (QAT Q4 from Google is highest quality for size), or a different model entirely (Qwen 2.5 3B is competitive)
- **Battery dies fast:** Expected. Plug in for anything serious.

---

## Reference: Memory Footprint Cheat Sheet

| Model | Quant | RAM Used | Fit on 16GB Pixel? |
|---|---|---|---|
| Gemma 4 E2B | Q4 | ~3.2 GB | Easily |
| Gemma 4 E2B | SFP8 | ~4.6 GB | Yes |
| Gemma 4 E4B | Q4 | ~5 GB | **Sweet spot** |
| Gemma 4 E4B | SFP8 | ~7.5 GB | Yes, slower |
| Gemma 4 E4B | BF16 | ~15 GB | Technically yes, miserable |

Leave at least 4–6 GB free for Android itself.

---

*Created: May 13, 2026*