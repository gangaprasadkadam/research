# Sarvam AI Deep Research: Capabilities, Accuracy, Pricing & Tradeoffs

## 📋 What This Document Is For
This document answers: **is Sarvam AI actually good enough to use in production, and is it worth choosing over Deepgram or ElevenLabs?** It goes deep on Sarvam's STT (Saaras v3) and TTS (Bulbul v2/v3) — real accuracy numbers, pricing in INR/USD, latency measured from India, and known limitations.

> Read this after the multilingual research. It's the "due diligence" check on Sarvam before committing to it.

## 📑 What It Covers
- **Section 1** — What Sarvam AI is (company background, India-first mission)
- **Section 2** — Sarvam models for voice agents (Saaras STT, Bulbul TTS)
- **Section 3** — Accuracy benchmarks (WER numbers, IndicVoices dataset, real phone audio)
- **Section 4** — Pricing in INR and USD (Saaras v3, Bulbul v2 vs v3)
- **Section 5** — Sarvam pipeline vs S2S (Gemini Live) — direct comparison
- **Section 6** — Final verdict: is Sarvam worth it?
- **Section 7** — Pipecat integration (install commands + code)
- **Section 8** — Sources

---

## 1. What is Sarvam AI?

Sarvam AI is an Indian full-stack AI company purpose-built for India's linguistic diversity. It is backed by the **IndiaAI Mission** (Government of India) and is the foundational partner for India's sovereign LLM program. Key differentiators:

- Trained exclusively on Indian speech data — urban, rural, noisy, mobile-recorded
- Covers all 22 scheduled Indian languages + English
- India-hosted endpoints (India-local servers for low latency)
- India-first pricing in INR — significantly cheaper than global providers
- Native Pipecat integration for both STT and TTS
- Used by large Indian enterprises: Tata Capital, Meesho, PocketFM, Cars24

---

## 2. Sarvam AI Models for Voice Agents

### 2.1 STT — Saaras v3 (Speech-to-Text)

**Flagship model for Indian language transcription.**

| Feature | Details |
|---|---|
| **Languages** | 22 Indian languages + English |
| **Modes** | `transcribe`, `translate`, `verbatim`, `translit`, `codemix` |
| **Streaming** | ✅ WebSocket-based real-time streaming |
| **VAD (Voice Activity Detection)** | ✅ Built-in |
| **Batch transcription** | ✅ Supported |
| **Noisy audio robustness** | ✅ Trained on real-world noisy Indian audio |
| **Indian accents** | ✅ Trained on diverse Indian regional accents |
| **Pipecat service** | `SarvamSTTService` |

**The `codemix` mode** — unique to Sarvam — is designed exactly for the interview use case:
- Hindi/Marathi/English can switch freely within a sentence
- Output preserves native script for Indian words + Latin for English
- Example: `"मेरा experience 5 साल का है and I worked on Python mainly"` → correctly transcribed as-is

---

### 2.2 TTS — Bulbul v2 / v3 (Text-to-Speech)

| Feature | Bulbul v2 | Bulbul v3 |
|---|---|---|
| **Languages** | 11 Indian + English | 11 Indian + English |
| **Voices** | Multiple regional | 30+ voices |
| **Voice quality** | ★★★★☆ | ★★★★½ |
| **Voice personas** | Conversational, Professional, Expressive, Dramatic | All v2 + more |
| **Streaming** | ✅ | ✅ |
| **Emotion/prosody** | Good | Better |
| **Regional accents** | Mumbai, Delhi, Pune variants | More variants |
| **Pipecat service** | `SarvamTTSService` | `SarvamTTSService` |

---

## 3. Accuracy Benchmarks

### 3.1 STT — Saaras v3 vs Competitors (IndicVoices + Svarah Benchmarks)

| Provider | Indic Languages WER | Notes |
|---|---|---|
| **Sarvam Saaras v3** | **~19%** ⭐ | Best for Indian languages; beats Gemini, GPT-4o, ElevenLabs Scribe |
| Deepgram Nova-3 | ~23–25% | Strong for Hinglish; weaker for pure Marathi |
| AssemblyAI Universal-3 | ~25–28% | English-first; Indian languages are secondary |
| Google Chirp | ~20–22% | Good Marathi; tied into GCP |
| OpenAI Whisper v3 | ~20–24% | Multilingual but no streaming; batch only |

> ✅ **Saaras v3 beats Gemini and GPT-4o on Indian speech benchmarks** — per Sarvam AI's own benchmark release (Business Standard, 2025) and independent evaluators.

### 3.2 Latency from India (Benchmark: Vexyl AI, 2025)

| Provider | Latency from India | Latency from US | Notes |
|---|---|---|---|
| **Sarvam AI** | **~952 ms** ⭐ | ~2,292 ms | India-local servers; optimal for India |
| Deepgram | ~1,830 ms | ~436 ms | US-only endpoints; high latency from India |
| AssemblyAI | ~1,100 ms | ~400 ms | US/NA optimised |
| Azure Speech | ~363 ms | ~1,876 ms | India AZ available but weak Indic accuracy |

> ⚠️ **Critical insight**: Deepgram, which appears faster globally, is actually **nearly 2× slower than Sarvam from India** due to US-only endpoints. For an India-deployed interview agent, Sarvam's latency advantage is significant.

### 3.3 Real-time factor

Saaras v3 claims an **8.5× real-time factor** — it can process 8.5 seconds of audio in 1 second on server hardware. This enables near-instantaneous streaming transcription.

---

## 4. Pricing

### 4.1 TTS — Bulbul Pricing

| Model | INR / 10K chars | USD / 10K chars | 1M chars cost |
|---|---|---|---|
| **Bulbul v2** | ₹15 | ~$0.18 | ₹1,500 (~$18) |
| **Bulbul v3** | ₹30 | ~$0.36 | ₹3,000 (~$36) |

**Comparison to ElevenLabs**: ElevenLabs charges ~$0.30–$0.60 per 10K chars for their standard tier — Bulbul v2 is **40–70% cheaper** for comparable quality in Indian languages.

### 4.2 STT — Saaras Pricing

| Plan | Cost | Rate Limit | Support |
|---|---|---|---|
| Starter | Pay-as-you-go | 60 req/min | — |
| Pro | ₹10,000/mo + ₹1,000 credits | 200 req/min | Email |
| Business | ₹50,000/mo + ₹7,500 credits | 1,000 req/min | Advanced |

- Estimated: ~**₹1 per audio minute** for voice AI use cases
- **Startup program**: Free credits available for early-stage startups

### 4.3 vs Global Providers (STT)

| Provider | Cost per 1000 minutes | India latency |
|---|---|---|
| **Sarvam Saaras** | ~₹1,000 (~$12) | 952 ms |
| Deepgram Nova-3 | ~$15–20 | 1,830 ms |
| AssemblyAI | ~$20–35 | ~1,100 ms |
| Google Speech v2 | ~$16 | 363 ms (but lower Indic accuracy) |

> Sarvam is **30–60% cheaper** than Deepgram and AssemblyAI for Indian deployments, with better accuracy.

---

## 5. The Big Tradeoff: Sarvam Pipeline vs S2S (Gemini Live)

### If you choose Sarvam, you MUST use STT+LLM+TTS pipeline (no S2S)

Here's exactly what you gain and lose:

---

### ✅ What You GAIN with Sarvam (Pipeline)

| Gain | Why it matters for interviews |
|---|---|
| **Accurate transcripts** | Store exactly what each candidate said — mandatory for review, scoring, legal |
| **Best Indian language accuracy** | Saaras v3 beats all global models on Indic benchmarks |
| **Genuine Marathi+Hindi codemix** | Native `codemix` mode — no other provider handles this as well |
| **LLM control** | Inject candidate's resume, job description, scoring rubric into system prompt |
| **Structured output** | LLM can return scores, follow-up questions, evaluation JSON alongside speech |
| **Full debuggability** | If agent says something wrong, you can see exactly which stage failed |
| **Compliance** | DPDP Act (India's data protection law) — easier with text audit trail |
| **Cost** | 30–60% cheaper than global alternatives |
| **India data residency** | Sarvam runs India-local servers; data doesn't leave India |
| **Vendor diversity** | Can swap STT or TTS independently without rebuilding everything |

---

### ❌ What You LOSE vs S2S (Gemini Live)

| Loss | Real-world impact for interviews |
|---|---|
| **Higher latency** | 800ms–2s total vs 350–500ms for Gemini Live |
| **No mid-speech barge-in** | Natural interruptions harder (Pipecat VAD helps, but not as seamless) |
| **Less natural prosody carry-through** | Emotional tone of candidate's speech is "reset" through the text bottleneck |
| **More moving parts** | Three services to maintain, monitor, and debug vs one |
| **No paralinguistic preservation** | Hesitations, sighs, emotional stress don't transfer through text |

---

### Latency Breakdown Comparison

```
SARVAM PIPELINE (STT+LLM+TTS):
  [STT Saaras v3]   → ~300–500ms (streaming TTFT from India)
  [LLM GPT-4o]      → ~200–600ms (streaming, first token)
  [TTS Bulbul v2]   → ~200–400ms (streaming TTFA)
  ─────────────────────────────────────────
  TOTAL: ~700ms – 1.5s end-to-end

GEMINI LIVE (S2S):
  [End-to-end]      → ~350–500ms
  ─────────────────────────────────────────
  TOTAL: ~350–500ms end-to-end
```

> **Reality check**: A 700ms–1.5s pause in an interview feels completely natural — it mirrors how a human interviewer pauses to think before asking the next question. This latency gap is **NOT a problem** for interview use cases. It becomes a problem only in fast, banter-style conversations.

---

## 6. Is Sarvam Worth It? Verdict

### For an AI Interview Agent in India: YES — Strongly Recommended

Here's why:

| Factor | Score | Reasoning |
|---|---|---|
| Indian language accuracy | ✅ Best | Beats all global providers on Indic benchmarks |
| Marathi+Hindi codemix | ✅ Native | `codemix` mode — unique capability |
| Transcript access | ✅ Full | Essential for interview review |
| LLM control | ✅ Full | Scoring, rubric injection, structured eval |
| Cost | ✅ Cheapest | 30–60% cheaper than Deepgram/AssemblyAI in India |
| Latency | ⚠️ ~1s | Acceptable for interviews; not for rapid chats |
| India data residency | ✅ Yes | Sarvam runs India-local; important for DPDP compliance |
| Pipecat native | ✅ Yes | `SarvamSTTService` + `SarvamTTSService` |
| S2S available | ❌ No | No true end-to-end S2S — pipeline only |

### When to reconsider and use Gemini Live instead:

- You want to build an MVP/demo in < 1 week (Gemini Live is simpler to integrate)
- The interview is very conversational/banter-heavy and 1s latency feels awkward
- Marathi support is low priority (mostly Hindi + English)
- You're already deep in Google Cloud ecosystem

---

## 7. Pipecat Integration

```python
pip install "pipecat-ai[sarvam]"
```

```python
from pipecat.services.sarvam.stt import SarvamSTTService
from pipecat.services.sarvam.tts import SarvamTTSService
from pipecat.services.openai import OpenAILLMService

# STT
stt = SarvamSTTService(
    api_key=os.getenv("SARVAM_API_KEY"),
    model="saaras:v3",
    mode="codemix",           # key for Hindi+Marathi+English mixing
    input_audio_codec="wav"
)

# LLM
llm = OpenAILLMService(
    api_key=os.getenv("OPENAI_API_KEY"),
    model="gpt-4o"
)

# TTS
tts = SarvamTTSService(
    api_key=os.getenv("SARVAM_API_KEY"),
    model="bulbul:v2",        # v3 for higher quality
    language="hi-IN",         # dynamically switchable
    voice="anila"             # professional female voice
)

pipeline = Pipeline([
    transport.input(),
    stt,
    LLMUserResponseAggregator(),
    llm,
    tts,
    transport.output(),
])
```

📚 Official docs: [Sarvam STT on Pipecat](https://docs.pipecat.ai/api-reference/server/services/stt/sarvam) | [Build Voice Agent with Pipecat](https://docs.sarvam.ai/api-reference-docs/integration/build-voice-agent-with-pipecat)

---

## 8. Sources

- [Sarvam AI — Models](https://www.sarvam.ai/models)
- [Sarvam AI — API Pricing](https://www.sarvam.ai/api-pricing)
- [Sarvam API Pricing Docs](https://docs.sarvam.ai/api-reference-docs/pricing)
- [Saaras v3 Blog — Sarvam AI](https://www.sarvam.ai/blogs/asr)
- [Saaras v3 Beats Gemini, GPT-4o on Indian Benchmarks — Business Standard](https://www.business-standard.com/technology/tech-news/saaras-v3-beats-gemini-gpt-4o-on-indian-speech-benchmarks-says-sarvam-ai-126021200384_1.html)
- [Bulbul v2 — Sarvam API Docs](https://docs.sarvam.ai/api-reference-docs/getting-started/models/bulbul)
- [Sarvam STT — Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/stt/sarvam)
- [TTS Latency Benchmark 2025 — Vexyl AI](https://vexyl.ai/tts-latency-benchmark-2025/)
- [Multilingual Voice AI India: Pipeline vs S2S — SubverseAI](https://subverseai.com/blogs/multilingual-voice-ai-in-india-stt-llm-tts-pipeline-vs-speech-to-speech-s2s-what-should-you-choose)
- [Recommendations for Indian Language Pipeline — LiveKit Community](https://community.livekit.io/t/recommendations-for-indian-language-tts-stt-and-llm-pipeline/173)
- [Sub-Second Voice Agent Latency Guide — Sayna AI](https://sayna.ai/blog/sub-second-voice-agent-latency-practical-architecture-guide)
- [Sarvam Audio: Speech Recognition Beyond Transcription](https://www.sarvam.ai/blogs/sarvam-audio)
- [Sarvam Edge — On-Device AI](https://www.sarvam.ai/blogs/sarvam-edge)
