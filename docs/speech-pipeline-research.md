# Speech Pipeline Research: Speech-to-Speech vs STT + LLM + TTS

## 📋 What This Document Is For
This document answers one question: **should we use a Speech-to-Speech (S2S) model, or a classic STT → LLM → TTS pipeline?** It compares both approaches for our specific use case (AI phone interview agent) and gives a final recommendation with reasoning.

> Read this first before any other research document. It sets the foundation for all other component decisions.

## 📑 What It Covers
- **Section 1** — The two architectures explained (S2S vs pipeline)
- **Section 2** — Head-to-head comparison (transcripts, latency, cost, control)
- **Section 3** — STT services compared (Sarvam, Deepgram, Whisper, Google, AssemblyAI)
- **Section 4** — TTS services compared (Sarvam Bulbul, ElevenLabs, Google, Azure)
- **Section 5** — S2S services compared (OpenAI Realtime, Gemini Live, AWS Nova Sonic)
- **Section 6** — Final recommendation for this project
- **Section 7** — Pipecat integration notes

---

## 1. The Two Architectures

### Option A — STT → LLM → TTS (Modular Pipeline)

```
Candidate Audio → [STT] → Text → [LLM] → Text → [TTS] → Agent Audio
```

Each stage is a separate service that runs in sequence. Pipecat was originally designed around this model and gives first-class support for it.

### Option B — Speech-to-Speech (S2S / End-to-End)

```
Candidate Audio → [S2S Model] → Agent Audio
```

A single unified model (or tightly coupled API) handles listening, reasoning, and speaking together — often over a persistent WebSocket/WebRTC connection.

---

## 2. Head-to-Head Comparison

| Dimension | STT → LLM → TTS | Speech-to-Speech (S2S) |
|---|---|---|
| **Latency** | 700 ms – 2 s (typical) | 200 – 500 ms |
| **Turn-taking / Barge-in** | Good with streaming tricks | Excellent — native interrupt support |
| **Emotional / Prosodic quality** | Depends on TTS model | Higher — preserves intonation end-to-end |
| **Transcript / text access** | ✅ Full intermediate text | ❌ No text checkpoint by default |
| **Debuggability** | High — inspect at every stage | Low — mostly opaque |
| **Compliance / Audit trail** | Easy — log transcripts | Hard — requires extra hooks |
| **Component flexibility** | Swap STT/LLM/TTS independently | Locked to provider bundle |
| **Vendor lock-in** | Low | High |
| **Cost** | Lower at scale | Up to 10× higher for real-time streaming |
| **Pipecat support** | Native, mature | Supported (OpenAI Realtime, Gemini Live, AWS Nova Sonic) |

---

## 3. STT Services Compared

### 3.1 Deepgram Nova-3 ⭐ Recommended for real-time
- **Latency**: 150 – 300 ms (best in class for streaming)
- **WER**: ~12 – 13% on conversational English
- **Strengths**: Low latency, robust in noisy conditions, custom vocabulary, enterprise SLAs
- **Weaknesses**: Slightly higher WER than batch-optimised models in clean environments
- **Pipecat**: Native service (`DeepgramSTTService`)
- **Pricing**: Usage-based, volume discounts available

### 3.2 AssemblyAI
- **Latency**: ~400 ms+
- **WER**: ~8 – 9% (higher accuracy, slower)
- **Strengths**: Strong accuracy, sentiment/topic analysis, developer-friendly
- **Weaknesses**: Slower for real-time; not ideal for sub-500 ms agents
- **Pipecat**: Supported

### 3.3 OpenAI Whisper (via API or Groq)
- **Latency**: 500 ms – 1 s+ (OpenAI API); ~300 ms via Groq
- **WER**: ~10 – 12% (API)
- **Strengths**: Multilingual, open-source, free if self-hosted
- **Weaknesses**: Not designed for real-time streaming; API adds latency
- **Pipecat**: Supported (via OpenAI or self-hosted)

### STT Summary Table

| Provider | WER | Latency (TTFT) | Real-time? | Best For |
|---|---|---|---|---|
| **Deepgram Nova-3** | ~12–13% | 150–300 ms | ✅ Yes | Live agents, telephony |
| AssemblyAI v2 | ~8–9% | ~400 ms+ | ⚠️ Partial | High accuracy, batch |
| Whisper v3 | ~10–12% | 500 ms – 1 s | ❌ No | Multilingual, offline |

---

## 4. TTS Services Compared

### 4.1 Cartesia (Sonic) ⭐ Recommended for real-time agents
- **Latency**: 40 – 90 ms TTFA (industry best)
- **Voice Quality**: ★★★★☆ — natural, low robotic artifacts
- **Voice Cloning**: Yes (from 3 seconds of audio)
- **Languages**: 15 – 20
- **Voices**: ~130
- **Strengths**: Ultra-low latency, lowest cost (up to 73% cheaper than ElevenLabs at volume)
- **Pipecat**: Native service (`CartesiaTTSService`)

### 4.2 ElevenLabs ⭐ Recommended for voice quality
- **Latency**: 75 ms (Flash v2.5) – 300 ms (quality models)
- **Voice Quality**: ★★★★★ — most human-like, best prosody
- **Voice Cloning**: Advanced (instant + professional tiers)
- **Languages**: 70+
- **Voices**: 3,000+
- **Strengths**: Best naturalness, broadest language support, rich voice library
- **Weaknesses**: Most expensive at volume; 40,000 char/request cap
- **Pipecat**: Native service (`ElevenLabsTTSService`)

### 4.3 PlayHT
- **Latency**: 200 ms+
- **Voice Quality**: ★★★☆☆
- **Languages**: 140+ (broad accent coverage)
- **Strengths**: Wide voice variety, budget-friendly
- **Weaknesses**: Less expressive, slower than Cartesia/ElevenLabs
- **Pipecat**: Supported

### TTS Summary Table

| Provider | Quality | Latency (TTFA) | Languages | Cost | Best For |
|---|---|---|---|---|---|
| **Cartesia Sonic** | ★★★★☆ | 40–90 ms | 15–20 | Lowest | Real-time agents |
| **ElevenLabs** | ★★★★★ | 75–300 ms | 70+ | Highest | Premium voice quality |
| PlayHT | ★★★☆☆ | 200 ms+ | 140+ | Budget | Variety at scale |

---

## 5. Speech-to-Speech (S2S) Services Compared

### 5.1 OpenAI Realtime API (GPT-4o Realtime)
- **Latency**: ~400 – 500 ms end-to-end
- **Voices**: 8+ (with on-the-fly switching)
- **Languages**: ~10
- **Tool Calling**: ✅ Full function calling support
- **Strengths**: Best reasoning, GPT-4o level intelligence, mature tooling, familiar DX
- **Weaknesses**: Fewer voices, less emotional nuance, higher cost
- **Pipecat**: `OpenAIRealtimeBetaLLMService`

### 5.2 Gemini Live (Google)
- **Latency**: ~350 – 400 ms
- **Voices**: 30+ HD, studio-quality, emotional adaptation
- **Languages**: 24+
- **Tool Calling**: ✅ Yes
- **Strengths**: Fastest S2S, best voice variety, emotional/affective dialog, multimodal
- **Weaknesses**: Requires Google Cloud buy-in for production; less developer tooling maturity
- **Pipecat**: `GeminiMultimodalLiveLLMService`

### 5.3 AWS Nova Sonic
- **Latency**: ~400 – 600 ms
- **Strengths**: AWS ecosystem integration, enterprise compliance
- **Pipecat**: `AWSNovaSonicLLMService`

### S2S Summary Table

| Platform | Latency | Voices | Languages | Tool Calling | Best For |
|---|---|---|---|---|---|
| **Gemini Live** | 350–400 ms | 30+ HD | 24+ | ✅ | Low latency, emotional, multilingual |
| **OpenAI Realtime** | 400–500 ms | 8+ | ~10 | ✅ | Advanced reasoning, complex logic |
| AWS Nova Sonic | 400–600 ms | Limited | ~8 | ✅ | AWS-native deployments |

---

## 6. Recommendation for AI Interview Agent

### For an interview bot, these factors matter most:
- **Transcripts are critical** — you need accurate text records of what candidates said
- **Moderate latency is acceptable** — interviews are not rapid-fire chats; a 700 ms pause feels natural
- **Voice quality matters** — candidates are judged on experience quality; professional, natural voice builds trust
- **Control & debuggability** — you need to review, log, and potentially replay conversations
- **Compliance** — storing conversation data requires a clear text audit trail

### ✅ Recommended Architecture: **STT + LLM + TTS Pipeline**

| Component | Recommended | Alternative |
|---|---|---|
| **STT** | Deepgram Nova-3 | Whisper via Groq (budget) |
| **LLM** | GPT-4o / Claude 3.5 Sonnet | Gemini 1.5 Pro |
| **TTS** | ElevenLabs (quality) or Cartesia (speed) | PlayHT (budget) |

**Why STT → LLM → TTS over S2S for interviews:**
1. **Transcripts** — Interview responses must be stored and reviewable. Modular pipeline gives full text at every step.
2. **LLM control** — You can use any LLM, craft detailed system prompts, inject candidate context (resume, job description), and run structured scoring logic.
3. **Cost** — At scale, modular pipeline is significantly cheaper. S2S streaming costs can be 10× higher.
4. **Debuggability** — When a candidate says something unexpected, you can trace exactly where the agent went wrong.
5. **Latency is fine** — A ~700 ms response gap in an interview is completely natural. S2S's advantage matters most for rapid conversational banter, not structured Q&A.

### When to consider S2S instead:
- You want the most natural, human-like conversation feel with barge-in
- You're building a demo or MVP and want minimal integration complexity
- Your candidate pool expects a highly fluid, real-time voice experience

---

## 7. Pipecat Integration Notes

Pipecat supports both architectures natively:

```python
# Modular Pipeline (recommended for interviews)
from pipecat.services.deepgram import DeepgramSTTService
from pipecat.services.openai import OpenAILLMService
from pipecat.services.cartesia import CartesiaTTSService

pipeline = Pipeline([
    transport.input(),
    DeepgramSTTService(api_key=DEEPGRAM_KEY),
    LLMUserResponseAggregator(),
    OpenAILLMService(api_key=OPENAI_KEY, model="gpt-4o"),
    CartesiaTTSService(api_key=CARTESIA_KEY, voice_id="..."),
    transport.output(),
])

# S2S Pipeline (for low-latency / demo)
from pipecat.services.openai_realtime_beta import OpenAIRealtimeBetaLLMService
```

---

## 8. Sources

- [Pipecat GitHub](https://github.com/pipecat-ai/pipecat)
- [Pipecat S2S Services — DeepWiki](https://deepwiki.com/pipecat-ai/pipecat/4.5-speech-to-speech-services)
- [Real-Time vs Turn-Based Voice Agents — Softcery](https://softcery.com/lab/ai-voice-agents-real-time-vs-turn-based-tts-stt-architecture)
- [Voice Pipelines vs S2S Models — Field Journal](https://fieldjournal.ai/blog/voice-pipelines-vs-speech-to-speech-models/)
- [STT Benchmarks for Voice Agents — Daily.co](https://www.daily.co/blog/benchmarking-stt-for-voice-agents/)
- [ElevenLabs vs Cartesia — Coval](https://www.coval.dev/blog/elevenlabs-vs-cartesia-which-tts-provider-is-right-for-your-voice-ai-project)
- [Gemini Live vs OpenAI Realtime — Edesy](https://edesy.in/ai-voice-assistant/blog/gemini-live-vs-openai-realtime)
- [How to Choose STT/TTS for Voice Agents — Softcery](https://softcery.com/lab/how-to-choose-stt-tts-for-ai-voice-agents-in-2025-a-comprehensive-guide)
- [Realtime Models vs Pipeline — SoftwareThug](https://www.softwarethug.com/posts/realtime-vs-pipeline-voice-ai/)
