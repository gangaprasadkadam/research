# Multilingual Support Research: Hindi, English & Marathi Code-Switching

## 📋 What This Document Is For
This document answers: **which STT and TTS services best handle candidates who switch between Hindi, Marathi, and English mid-sentence?** Indian candidates rarely speak in a single language — this is called "code-switching." This doc finds which tools handle that natively, and which ones struggle.

> Read this after the speech pipeline research. It goes deeper on the multilingual dimension.

## 📑 What It Covers
- **Section 1** — What code-switching is and why it's a challenge
- **Section 2** — How code-switching is handled technically (transliteration vs. native support)
- **Section 3** — STT options for Hindi + Marathi + English (with comparison table)
- **Section 4** — TTS options for Hindi + Marathi + English
- **Section 5** — Recommended STT + TTS architecture
- **Section 6** — How to handle language switching inside a Pipecat pipeline
- **Section 7** — Sarvam Saaras v3 codemix mode in action (real examples)
- **Section 8** — Key decisions and trade-offs
- **Section 9** — S2S models for Indian multilingual (OpenAI Realtime, Gemini Live)
- **Section 10** — Sources

---

## 1. The Problem: Code-Switching in Indian Interviews

Indian candidates rarely stay in a single language. A typical response might look like:

> *"My experience in backend development hai, mainly Python aur Node.js mein kaam kiya hai. Project deadlines meet karne mein I always try to prioritize."*

This is **code-switching** (or code-mixing) — seamlessly blending Hindi, Marathi, and English within a sentence. A production-ready interview agent must:

1. **Accurately transcribe** mixed-language speech
2. **Understand** the full meaning regardless of language mix
3. **Respond** in a natural voice that matches the candidate's language context
4. **Preserve context** — don't lose conversation history when language changes

---

## 2. How Code-Switching is Handled Technically

### Pipeline flow for multilingual voice agents:

```
Candidate Audio (Hindi/English/Marathi mixed)
    ↓
[Multilingual STT with codemix mode]
    → Transcript with language tags per word/segment
    ↓
[LLM with multilingual prompt]
    → Response text in appropriate language mix
    ↓
[TTS with Indian language voice]
    → Natural agent audio
```

### Key challenges:
- **Per-word language detection** — "मेरा phone number hai" needs word-level tags
- **Acoustic similarity** — Hindi and Marathi share Devanagari script but differ phonetically
- **Accent diversity** — Mumbai Hindi ≠ Delhi Hindi ≠ Pune Marathi
- **Context preservation** — Don't reset dialogue state on language switch

---

## 3. STT Options for Hindi + Marathi + English Code-Switching

### 3.1 Sarvam AI — Saaras v3 ⭐ Top Pick for Indian Languages

**The only purpose-built Indian-language STT with native Pipecat integration.**

- **Languages**: 22 Indian languages + English (Hindi, Marathi, Tamil, Telugu, Bengali, etc.)
- **Modes** (unique feature):
  - `codemix` — preserves mixed-language input as spoken (e.g., "मेरा phone number है 9840950950")
  - `transcribe` — source language transcription
  - `translate` — translates to English
  - `verbatim` — word-for-word, including fillers (umm, haan, etc.)
  - `translit` — Romanized/Latin script output
- **Accuracy**: Trained on real-world Indian audio (urban, rural, noisy, mobile)
- **Streaming**: WebSocket-based, real-time with built-in VAD
- **Pipecat Native**: `SarvamSTTService` — official service in pipecat package

```python
pip install "pipecat-ai[sarvam]"
```

```python
from pipecat.services.sarvam.stt import SarvamSTTService

stt = SarvamSTTService(
    api_key="YOUR_SARVAM_API_KEY",
    model="saaras:v3",
    mode="codemix",          # handles Hindi/English/Marathi mixing
    input_audio_codec="wav"  # or "pcm_s16le" for streaming
)
```

📚 [Pipecat Sarvam STT Docs](https://docs.pipecat.ai/api-reference/server/services/stt/sarvam) | [Build voice agent with Pipecat — Sarvam Docs](https://docs.sarvam.ai/api-reference-docs/integration/build-voice-agent-with-pipecat)

---

### 3.2 Deepgram Nova-3 — Multilingual Code-Switching

- **Languages**: Hindi, English, Marathi supported via `language=multi` flag
- **Code-switching**: Per-word language tags in transcripts
- **Latency**: 150 – 300 ms (fastest among general STT providers)
- **Accuracy**: 88 – 92% for Hindi/Hinglish in real environments
- **Limitation**: Not as deep on Marathi as Sarvam; more optimised for Hinglish than Marathi-English mix
- **Pipecat Native**: `DeepgramSTTService`

```python
# Deepgram with multilingual code-switching
wss_url = "wss://api.deepgram.com/v1/listen?language=multi&model=nova-3&endpointing=100"
```

---

### 3.3 AssemblyAI Universal-3 Pro

- **Languages**: 99+ languages including Hindi and Marathi
- **Code-switching**: Supported via Universal streaming model
- **Latency**: Sub-500 ms (suitable for voice agents, but slower than Deepgram)
- **Extras**: Sentiment analysis, topic detection, structured data extraction
- **Best when**: You need post-interview analytics (sentiment, topic coverage)
- **Pipecat**: Supported

---

### 3.4 Google Chirp / Speech-to-Text v2

- **Languages**: Excellent Hindi and Marathi support
- **Accuracy**: One of the best for Indian languages, especially Marathi
- **Latency**: Moderate (~400 ms)
- **Code-switching**: Supported via multi-language model config
- **Pipecat**: Supported via Google Cloud STT service
- **Note**: Ties you into Google Cloud ecosystem

---

### STT Comparison Table (India-focused)

| Provider | Hindi | Marathi | Code-mix | Latency | Pipecat Native | Best For |
|---|---|---|---|---|---|---|
| **Sarvam Saaras v3** | ★★★★★ | ★★★★★ | ✅ Native `codemix` mode | ~300 ms | ✅ Yes | Indian language-first |
| **Deepgram Nova-3** | ★★★★☆ | ★★★☆☆ | ✅ `language=multi` | 150–300 ms | ✅ Yes | Low latency, Hinglish |
| AssemblyAI Universal-3 | ★★★★☆ | ★★★★☆ | ✅ Yes | ~400 ms | ✅ Yes | Analytics + accuracy |
| Google Chirp | ★★★★☆ | ★★★★☆ | ✅ Yes | ~400 ms | ✅ Yes | Marathi accuracy |

---

## 4. TTS Options for Hindi + Marathi + English

### 4.1 Sarvam AI — Bulbul v2/v3 ⭐ Top Pick for Indian Languages

- **Languages**: 11 Indian languages including Hindi, Marathi + English
- **Voice quality**: Natural, regionally authentic accents (Mumbai, Delhi, Pune variants)
- **Voice personas**: Conversational, Professional, Narrational, Expressive, Dramatic
- **Code-mixing**: Handles Hinglish/Marathi-English output naturally
- **Latency**: Low (streaming supported)
- **Price**: Optimised for Indian market — significantly cheaper than ElevenLabs
- **Pipecat**: Native `SarvamTTSService`

```python
from pipecat.services.sarvam.tts import SarvamTTSService

tts = SarvamTTSService(
    api_key="YOUR_SARVAM_API_KEY",
    model="bulbul:v2",
    language="hi-IN",     # or "mr-IN" for Marathi, dynamically switchable
    voice="anila"         # professional female voice
)
```

---

### 4.2 ElevenLabs — Premium Multilingual Quality

- **Indian Languages**: Hindi, Marathi, Indian-accented English
- **Voice quality**: ★★★★★ — most human-like globally
- **Languages**: 70+ including Indian languages
- **Code-switching output**: Yes (can produce Hinglish-style TTS)
- **Enterprise use**: Used by Meesho, PocketFM, Cars24
- **Latency**: 75 – 300 ms
- **Price**: Highest among all options
- **Pipecat Native**: `ElevenLabsTTSService`

---

### 4.3 Bhashini (Government-backed)

- **Languages**: 22 official Indian languages (widest coverage)
- **Best for**: Accessibility, government, education projects
- **Voice quality**: ★★★★☆
- **Cost**: Very low / subsidised
- **Limitation**: Less enterprise-grade API maturity; fewer telephony integrations
- **Pipecat**: Community-supported (not native)

---

### 4.4 Google WaveNet / Azure Neural TTS

- **Languages**: Hindi, Marathi, English with good Indian accent options
- **Voice quality**: ★★★★☆
- **Cost**: Budget-friendly
- **Pipecat**: Supported (Google TTS service)
- **Best for**: Cost-sensitive, reliable fallback

---

### TTS Comparison Table (India-focused)

| Provider | Hindi | Marathi | Indian Accent | Voice Quality | Latency | Cost | Pipecat |
|---|---|---|---|---|---|---|---|
| **Sarvam Bulbul v2** | ★★★★★ | ★★★★★ | Excellent | ★★★★½ | Low | $ | ✅ Native |
| **ElevenLabs** | ★★★★☆ | ★★★★☆ | Excellent | ★★★★★ | 75–300 ms | $$$ | ✅ Native |
| Bhashini | ★★★★★ | ★★★★★ | Very Good | ★★★★☆ | Moderate | $ | Community |
| Google WaveNet | ★★★★☆ | ★★★★☆ | Good | ★★★★☆ | Moderate | $$ | ✅ Yes |

---

## 5. Recommended Architecture for Multilingual Interview Agent

### Strategy: Sarvam-First + ElevenLabs Fallback

```
Candidate Audio
    ↓
[Sarvam Saaras v3 STT — codemix mode]
    → Accurate Hindi/Marathi/English transcript with mixing
    ↓
[LLM — GPT-4o or Claude 3.5]
    → System prompt instructs: respond in candidate's language
    → Context preserved across language switches
    ↓
[Sarvam Bulbul v2 TTS — dynamic language]
    → Natural Indian-accented voice
    → Switchable per response
```

### Component Recommendations

| Component | Primary Choice | Alternative |
|---|---|---|
| **STT** | Sarvam Saaras v3 (`codemix`) | Deepgram Nova-3 (`language=multi`) |
| **LLM** | GPT-4o / Claude 3.5 Sonnet | Gemini 1.5 Pro (multilingual) |
| **TTS** | Sarvam Bulbul v2 | ElevenLabs (if premium voice needed) |
| **Fallback TTS** | Google WaveNet | Azure Neural |

---

## 6. Handling Language Switching in Pipecat Pipeline

### The Challenge
When a candidate switches language mid-interview:
1. STT must detect and transcribe correctly (✅ handled by Sarvam codemix)
2. LLM must respond in the appropriate language (✅ handled via prompt)
3. TTS must speak in the right language voice (⚠️ needs dynamic switching)
4. Conversation history must not reset (✅ handled by Pipecat session state)

### Dynamic TTS Language Switching Pattern

```python
# Pipecat processor to detect language and switch TTS dynamically
class LanguageSwitchProcessor(FrameProcessor):
    async def process_frame(self, frame, direction):
        if isinstance(frame, LLMFullResponseEndFrame):
            detected_lang = detect_language(frame.text)
            if detected_lang != self.current_lang:
                self.current_lang = detected_lang
                # Update TTS voice/language
                await self.tts.set_language(detected_lang)
        await self.push_frame(frame, direction)
```

### LLM Prompt Strategy

```
System: You are an AI interviewer. The candidate may speak in Hindi, English, or Marathi, 
or mix them freely. Always respond in the same language or mix the candidate used. 
Never ask them to switch languages. Maintain natural conversation continuity.
```

---

## 7. Code-Switching: Sarvam Saaras v3 in Action

| Input Speech | Saaras v3 `codemix` Output |
|---|---|
| "मेरा experience 5 साल का है" | "मेरा experience 5 साल का है" |
| "I worked on backend, mainly Python mein" | "I worked on backend, mainly Python मैं" |
| "Deadline meet karna maza aata hai" | "Deadline meet करना मज़ा आता है" |
| "माझा experience cloud platforms मध्ये आहे" | "माझा experience cloud platforms मध्ये आहे" |

✅ Hindi words stay in Devanagari, English words remain in English — perfect for LLM input.

---

## 8. Key Decisions & Trade-offs

### Option A: Sarvam-only Stack (Cost-effective, India-first)
- STT: Sarvam Saaras v3
- TTS: Sarvam Bulbul v2
- **Pros**: Best Indian language accuracy, lowest cost, single vendor for STT+TTS, purpose-built
- **Cons**: Less globally recognised; fewer voice options than ElevenLabs

### Option B: Deepgram STT + Sarvam TTS (Best latency + Indian voice)
- STT: Deepgram Nova-3 (`language=multi`) — fastest real-time
- TTS: Sarvam Bulbul v2 — authentic Indian voice
- **Pros**: Ultra-low latency STT + natural Indian TTS
- **Cons**: Two vendors; Deepgram's Marathi accuracy is lower

### Option C: Sarvam STT + ElevenLabs TTS (Premium experience)
- STT: Sarvam Saaras v3
- TTS: ElevenLabs (Hindi/Marathi voices)
- **Pros**: Best-in-class voice naturalness; Sarvam's deep Indian language accuracy
- **Cons**: Highest cost; ElevenLabs has fewer Indian-specific voice options vs Sarvam

---

## 9. S2S (Speech-to-Speech) for Indian Multilingual: Deep Research

### 9.1 Does S2S Support Hindi / Marathi / English Code-Switching?

This is a critical question. The short answer: **partially, and with important caveats.**

S2S models like OpenAI Realtime and Gemini Live do support Indian languages, but their handling of real-world code-switching (especially rapid Hindi-Marathi-English mixing) is significantly weaker than purpose-built pipelines with Sarvam.

---

### 9.2 OpenAI Realtime API (GPT-4o) — Multilingual S2S

| Feature | Details |
|---|---|
| **Hindi support** | ✅ Yes — well-supported |
| **Marathi support** | ✅ Yes — listed, but less robust than Hindi |
| **Code-switching (mid-sentence)** | ✅ Yes — can follow language switches |
| **Indian accent** | ⚠️ Inconsistent — may drift to English-accented output |
| **Latency** | 400–500 ms |
| **Pipecat service** | `OpenAIRealtimeBetaLLMService` |

**Key facts:**
- GPT-4o optimised token efficiency for Indian languages by **2.9×** vs previous models — cheaper and faster for Hindi/Marathi
- Handles mid-sentence code-switching reasonably well
- ⚠️ **Limitation**: In ambiguous cases, tends to default to English-accented speech output
- No fine-grained control over TTS voice, prosody, or regional accent in S2S mode
- Function calling works in Hindi/Marathi — viable for interview scoring tools

---

### 9.3 Gemini Live (Google) — Multilingual S2S ⭐ Strongest for Indian S2S

| Feature | Details |
|---|---|
| **Hindi support** | ✅ Yes — excellent |
| **Marathi support** | ✅ Yes — officially listed |
| **Code-switching (mid-sentence)** | ✅ Yes — auto-detects without manual settings |
| **Indian accent** | ✅ Good — better than OpenAI for Indian context |
| **Languages (India)** | Hindi, Marathi, Bengali, Gujarati, Kannada, Malayalam, Tamil, Telugu, Urdu + English |
| **Latency** | 350–400 ms |
| **Pipecat service** | `GeminiMultimodalLiveLLMService` |

**Key facts:**
- As of March 2025, Gemini Live can **auto-switch languages mid-conversation** without any manual settings change
- Users can freely mix Hindi + Marathi + English and Gemini Live follows and responds in kind
- 30+ HD studio voices with emotional adaptation
- ⚠️ **Limitation**: Rapid script-switching (Devanagari ↔ Latin within one sentence) can cause higher error rates
- ⚠️ Requires Google Cloud buy-in for production deployments

---

### 9.4 Does Sarvam AI Have an S2S (End-to-End) Model?

**❌ No — Sarvam does NOT have a true end-to-end S2S neural model (as of 2025).**

Sarvam offers:
- `Saaras v3` — best Indian STT
- `Bulbul v2/v3` — best Indian TTS
- `Mayura` / `Sarvam-Translate` — speech-to-text translation (not audio-to-audio)

They can **simulate S2S** by chaining `STT → Translation → TTS`, but that's still a pipeline under the hood — not a unified neural S2S model. The Sarvam Edge project hints at more on-device integration in the future.

> **Bottom line**: For Indian language S2S, Sarvam is a pipeline player — excellent for STT+TTS but not a direct S2S competitor to OpenAI Realtime or Gemini Live.

---

### 9.5 Other S2S Options for Indic Languages

| Platform | Hindi | Marathi | S2S Type | Notes |
|---|---|---|---|---|
| **Gemini Live** | ✅ | ✅ | True end-to-end | Best for Indian S2S |
| **OpenAI Realtime** | ✅ | ✅ | True end-to-end | Good but less Indian-accent tuned |
| **AI4Bharat** | ✅ | ✅ | Research/Experimental | Open-source; IndicConformer + IndicTrans2 |
| **BharatGen** | ✅ | ✅ | TTS/ASR pipeline | Not true S2S |
| **Sarvam** | ✅ | ✅ | Pipeline only | No end-to-end S2S model |

**AI4Bharat** (IIT Madras) is notable — they have the Indic-S2ST dataset (600 hours across 14 Indian languages including Hindi and Marathi) and experimental S2S interfaces, but these are research-grade, not production-ready APIs.

---

### 9.6 S2S Code-Switching Limitations for Indian Languages

Even the best S2S models struggle with Indian code-switching. Key limitations:

1. **Rare training data** — large annotated datasets of spontaneous Hindi-Marathi-English code-switched audio are scarce; models are undertrained for this specific mix
2. **Accent inconsistency** — Mumbai Hindi ≠ Delhi Hindi ≠ Pune Marathi; S2S models often flatten these into generic accents
3. **Script boundary confusion** — rapid switches between Devanagari and Latin script within one sentence cause higher error rates in both transcription and synthesis
4. **Default-to-English bias** — when uncertain, S2S models (especially OpenAI) tend to output English-accented speech even if input was Hindi/Marathi
5. **No transcript access** — unlike STT+LLM+TTS, you can't inspect or store what was understood — critical for interview replay and analysis
6. **Structured output limitations** — S2S models are weaker at following strict interview templates, scoring rubrics, or multi-step workflows

---

### 9.7 S2S vs STT+LLM+TTS: Indian Language Decision Matrix

| Scenario | S2S (Gemini Live / OpenAI) | STT+LLM+TTS (Sarvam Stack) |
|---|---|---|
| Casual, conversational tone | ✅ Excellent | ✅ Good |
| Regional accent handling | ⚠️ Inconsistent | ✅ Purpose-built |
| Hinglish (Hindi+English mix) | ✅ Good | ✅ Excellent (codemix) |
| Marathi+English mix | ⚠️ Partial | ✅ Native support |
| Interview transcript storage | ❌ No native access | ✅ Full text at every step |
| Structured Q&A / scoring | ⚠️ Weaker | ✅ Full LLM control |
| Latency | ✅ 350–500ms | ⚠️ 700ms–1s |
| Cost at scale | ❌ 5–10× higher | ✅ Lower |
| Vendor lock-in | ❌ High | ✅ Low |

### 9.8 Verdict for Interview Agent

**For an AI interview agent with Hindi/English/Marathi support:**

> **Recommended: STT+LLM+TTS with Sarvam Saaras v3 (STT) + Sarvam Bulbul v2 (TTS)**

**Why not S2S:**
- You need transcripts to review what candidates said
- Marathi+English code-switching is not reliably handled by OpenAI Realtime or Gemini Live
- You need structured output (scoring, follow-up questions based on answer content)
- Cost at production interview scale would be 5-10× higher with S2S

**When S2S makes sense:**
- You want a demo-ready prototype quickly (Gemini Live is easiest to integrate)
- The interview is primarily Hindi or English (not Marathi-heavy)
- Naturalness/fluency is the #1 priority over accuracy and control

---

## 10. Sources

## 10. Sources

- [Sarvam AI — Official Site](https://www.sarvam.ai/)
- [Saaras v3 — Sarvam API Docs](https://docs.sarvam.ai/api-reference-docs/getting-started/models/saaras)
- [Sarvam STT — Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/stt/sarvam)
- [Build Voice Agent with Pipecat — Sarvam Docs](https://docs.sarvam.ai/api-reference-docs/integration/build-voice-agent-with-pipecat)
- [Bulbul-V2 Launch — Analytics Vidhya](https://www.analyticsvidhya.com/blog/2025/05/bulbul-v2-by-sarvam/)
- [Sarvam Audio: Speech Recognition beyond Transcription](https://www.sarvam.ai/blogs/sarvam-audio)
- [Build Multilingual Voice Agent — LiveKit Blog](https://livekit.com/blog/build-multilingual-voice-agent-automatic-language-switching)
- [Deepgram Multilingual Code-Switching Docs](https://developers.deepgram.com/docs/multilingual-code-switching)
- [Hindi-Marathi Code-Switch Speech Recognition Research](https://www.preprints.org/manuscript/202503.2278/v1)
- [Complete Guide to Indian Language Voice AI — Edesy](https://edesy.in/ai-voice-assistant/blog/indian-language-voice-ai-guide)
- [Multilingual Voice Agents — AssemblyAI Blog](https://www.assemblyai.com/blog/multilingual-voice-agent)
- [Gemini Live's latest update makes it a true polyglot — Android Authority](https://www.androidauthority.com/gemini-live-multilingual-conversations-3531848/)
- [Building a Real-Time Multilingual Voice Agent with Gemini Live API](https://dev.to/vinayak_maskar_5e6d6c60c2/building-a-real-time-multilingual-voice-agent-with-gemini-live-api-on-google-cloud-2jfh)
- [Gemini Live in 9 Indian Languages — FusionChat](https://fusionchat.ai/news/gemini-live-by-google-multilingual-conversations-in-hindi-and-8-other-languages)
- [OpenAI Realtime API — Official Docs](https://developers.openai.com/api/docs/guides/realtime)
- [GPT-4o Hindi token efficiency — Analytics India Magazine](https://analyticsindiamag.com/ai-news-updates/openai-just-killed-google-translate-with-gpt-4o/)
- [Multilingual Voice AI in India: Pipeline vs S2S — SubverseAI](https://subverseai.com/blogs/multilingual-voice-ai-in-india-stt-llm-tts-pipeline-vs-speech-to-speech-s2s-what-should-you-choose)
- [AI4Bharat Models](https://models.ai4bharat.org/)
- [Indic-S2ST Dataset — IJCNLP 2025](https://aclanthology.org/2025.ijcnlp-long.196/)
- [Code-Switching ASR for Hindi-Marathi — IEEE](https://ieeexplore.ieee.org/document/10835062)
