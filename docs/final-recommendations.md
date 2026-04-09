# AI Voice Interview Agent — Technical Recommendations

## 📋 What This Document Is For

This is the **single source of truth** for all technical decisions on this project. It consolidates all the research from the other documents into one place — with the final recommended tool for each component, a runner-up option, comparison tables, code snippets, and clear reasoning for every choice. Start here if you need to understand "what are we using and why."

> This document summarizes all research. Detailed justification and benchmarks are in the linked research docs.

## 📑 What It Covers

- **Quick Reference** — Full stack at a glance (one table)
- **Section 1** — Speech architecture decision: Pipeline vs S2S (and full S2S consequences)
- **Section 2** — Transport layer: Twilio #1 / Plivo #2 (with `FastAPIWebsocketTransport` + serializer explained)
- **Section 3** — STT: Sarvam Saaras v3 #1 / Deepgram Nova-3 #2 (with comparison table)
- **Section 4** — LLM: GPT-4o #1 / Claude 3.5 Sonnet #2 (with comparison table)
- **Section 5** — TTS: Sarvam Bulbul v2 #1 / ElevenLabs Flash v2 #2 (with comparison table)
- **Section 6** — VAD: Silero (built-in Pipecat, no alternative needed)
- **Section 7** — Memory: within-session (GPT-4o native) + cross-round (Mem0 #1 / Supermemory #2)
- **Section 8** — Full pipeline diagram (all components wired together with code)
- **Section 9** — Latency budget (milliseconds per component)
- **Section 10** — Cost estimate per 45-minute interview session
- **Section 11** — What's not yet researched (open questions)
- **Section 12** — References to all research docs and official docs

---

> **Project**: AI voice agent that conducts candidate screening interviews via phone call, triggered by a recruiter dashboard (React Native app).  
> **Framework**: [Pipecat](https://github.com/pipecat-ai/pipecat) · **Language**: Python · **Market**: India (Hindi / Marathi / English)

---

## Quick Reference: Final Stack

| Layer                     | Choice                    | Alternative                 | Why                                   |
| ------------------------- | ------------------------- | --------------------------- | ------------------------------------- |
| **Speech Architecture**   | STT + LLM + TTS Pipeline  | —                           | Transcripts, control, cost            |
| **Transport**             | Twilio (PSTN outbound)    | Plivo (cheaper India rates) | Calls candidate's phone               |
| **STT**                   | Sarvam Saaras v3          | Deepgram Nova-3             | Only model for Indic code-switching   |
| **LLM**                   | GPT-4o (128K)             | Claude 3.5 Sonnet           | Context fits 45 min; strong reasoning |
| **TTS**                   | Sarvam Bulbul v2          | ElevenLabs (premium)        | Natural Indian voice, lowest cost     |
| **VAD**                   | Silero (built-in Pipecat) | —                           | Turn detection, free                  |
| **Single-session Memory** | GPT-4o 128K context       | —                           | Full 45-min interview fits natively   |
| **Cross-round Memory**    | Mem0                      | Supermemory                 | Round 1 → Round 2 recall              |
| **Backend**               | FastAPI (Python)          | —                           | Async, Pipecat-compatible             |
| **DB**                    | PostgreSQL                | —                           | Transcripts, scores, candidates       |

---

## 1. Speech Architecture Decision: Pipeline vs S2S

### What We Evaluated

Two approaches exist for voice AI:

**Option A — STT → LLM → TTS (Modular Pipeline)**

```
Candidate Audio → [Sarvam STT] → Text → [GPT-4o] → Text → [Sarvam TTS] → AI Audio
```

**Option B — Speech-to-Speech (S2S)**

```
Candidate Audio → [OpenAI Realtime / Gemini Live] → AI Audio
```

### Decision: ✅ STT + LLM + TTS Pipeline

| Reason                        | Why It Matters for Interviews                                                  |
| ----------------------------- | ------------------------------------------------------------------------------ |
| **Transcripts are mandatory** | Recruiters must review what candidates said; S2S has no native text access     |
| **Full LLM control**          | Inject job description, resume, scoring rubric into any prompt                 |
| **Cost**                      | S2S streaming costs 5–10× more than modular pipeline at scale                  |
| **Structured output**         | GPT-4o reliably returns evaluation JSON; S2S is weaker for structured tasks    |
| **Debuggability**             | You can inspect every STT/LLM output step; S2S is a black box                  |
| **Latency is acceptable**     | 1.5–2s response gap on a phone interview feels natural                         |
| **Indian language accuracy**  | Sarvam codemix is purpose-built; S2S models under-trained on Marathi+Hindi mix |

### When would S2S make sense (not this project)?

- Extremely latency-sensitive casual voice chat (gaming, social apps)
- Demo or MVP needing the quickest integration
- Primary language is English only, no Marathi code-switching required

---

### 🔴 If We Had Chosen S2S — Full Consequences

**Provider choice**: Gemini Live (best for Indian multilingual S2S) or OpenAI Realtime (best reasoning). Sarvam has **no S2S model** — they only offer Saaras (STT) + Bulbul (TTS).

**What breaks or gets harder:**

#### ❌ No transcripts by default

S2S models do not produce text as output — they produce audio directly. You must explicitly hook into `TranscriptionFrame` events in Pipecat to capture text. Without this, you have no record of what the candidate said — which is unusable for a screening interview.

```python
# Extra code required to capture transcripts from S2S:
class TranscriptSaver(FrameProcessor):
    async def process_frame(self, frame, direction):
        if isinstance(frame, TranscriptionFrame):
            await self.save_to_db(frame.text)  # manually wired
        await self.push_frame(frame, direction)
```

#### ❌ Gemini Live: 15-minute session hard limit

Gemini Live's audio-only sessions **expire after 15 minutes** without context window compression enabled. A 45-minute interview would require:

- Context compression configuration
- At least 3–4 WebSocket reconnections
- Session resumption token management (valid for 2 hours)
- Code to handle mid-interview reconnection gracefully

OpenAI Realtime does not have this specific limit, but its WebSocket connection must be managed carefully over 45 minutes.

#### ❌ Context lives on the provider's server

With S2S, the conversation history is stored on OpenAI's or Google's servers inside the active WebSocket session. When the session ends — context is gone. You cannot:

- Inspect what the LLM is "thinking"
- Modify the context mid-interview
- Use Pipecat's built-in summarization (`enable_context_summarization`)
- Use Mem0 natively for cross-round memory (must be manually injected at session start)

#### ❌ No LLM control for structured scoring

S2S models are optimized for conversational flow, not for following strict interview rubrics or returning structured JSON at the end of a session. GPT-4o in pipeline mode reliably returns:

```json
{
  "overall_score": 7.5,
  "technical_score": 8,
  "recommendation": "Proceed to Round 2"
}
```

S2S models can do this too, but it requires custom function-calling hooks and is significantly less reliable.

#### ❌ Marathi+Hindi+English code-switching is unreliable

OpenAI Realtime and Gemini Live handle Hinglish (Hindi+English) reasonably well. However:

- Rapid Marathi+English script switching causes higher error rates
- Both models default toward English-accented output when uncertain
- No `codemix` mode equivalent — language detection is automatic but less controllable
- Sarvam's purpose-built training on Indian audio cannot be matched by global S2S models

#### ✅ What S2S does better

- **Latency**: 350–500ms vs 1.2–2.2s for pipeline. Noticeably snappier.
- **Naturalness**: Preserves intonation end-to-end; no TTS "flatness"
- **Barge-in**: Native interrupt support — candidate can cut the AI off mid-sentence
- **Integration simplicity**: Fewer components to wire up for a demo or MVP

#### Summary: S2S Trade-off for This Project

|                            | STT+LLM+TTS Pipeline ✅ | S2S ❌                                      |
| -------------------------- | ----------------------- | ------------------------------------------- |
| Transcript access          | ✅ Automatic            | ❌ Requires extra hooks                     |
| 45-min session support     | ✅ Native               | ❌ Gemini: 15-min limit; OpenAI: manageable |
| Marathi code-switching     | ✅ Sarvam codemix       | ⚠️ Inconsistent                             |
| Structured evaluation JSON | ✅ Reliable             | ⚠️ Needs custom function hooks              |
| Cross-round memory (Mem0)  | ✅ Native Pipecat       | ❌ Manual injection at session start        |
| Latency                    | ⚠️ 1.2–2.2s             | ✅ 350–500ms                                |
| Cost at production scale   | ✅ Lower                | ❌ 5–10× higher                             |
| Vendor lock-in             | ✅ Low                  | ❌ Locked to OpenAI or Google               |
| Debuggability              | ✅ Full visibility      | ❌ Black box                                |

> **Bottom line**: S2S would make the interview feel more natural and responsive, but it breaks transcripts, structured scoring, Marathi code-switching reliability, and 45-minute session continuity. For a screening interview product, pipeline is the right choice.

📚 Detailed comparison: [`docs/speech-pipeline-research.md`](./speech-pipeline-research.md) · [Pipecat S2S services](https://deepwiki.com/pipecat-ai/pipecat/4.5-speech-to-speech-services) · [`docs/memory-context-management-research.md §10`](./memory-context-management-research.md)

---

## 2. Transport Layer: How the Audio Gets to Pipecat

### Our Use Case

- Recruiter triggers outbound call via React Native app → REST API
- AI agent calls the candidate's Indian mobile/landline
- Candidate answers → audio streams to Pipecat → AI responds back via phone

---

### ⚠️ Important: Twilio ≠ Pipecat Transport

This is the most common confusion. Let's be precise:

| Term                            | What it is                          | Role                                                                    |
| ------------------------------- | ----------------------------------- | ----------------------------------------------------------------------- |
| **Twilio**                      | A telephony company (cloud service) | Makes the PSTN phone call to the candidate                              |
| **`FastAPIWebsocketTransport`** | A **Pipecat transport class**       | Manages the WebSocket connection between Twilio and your FastAPI server |
| **`TwilioFrameSerializer`**     | A **Pipecat serializer class**      | Translates Twilio's audio format ↔ Pipecat's internal format            |

**Twilio is not a Pipecat transport class.** It is the telephony provider that sends audio to your server via WebSocket. The Pipecat classes that handle this are `FastAPIWebsocketTransport` (the transport) + `TwilioFrameSerializer` (the serializer). Both are always used together for any PSTN phone call.

```
Candidate's phone → Twilio (PSTN) → WebSocket → FastAPIWebsocketTransport
                                                          ↓
                                                  TwilioFrameSerializer
                                                  (decodes Twilio's format)
                                                          ↓
                                                  Pipecat Pipeline
                                                  (VAD → STT → LLM → TTS)
```

**You cannot use `TwilioFrameSerializer` without `FastAPIWebsocketTransport`.**
**You cannot use `FastAPIWebsocketTransport` alone for Twilio** — without the serializer, Pipecat won't understand Twilio's JSON message format.

They are always paired:

```python
# These two always go together for PSTN calls:
transport = FastAPIWebsocketTransport(websocket=ws, params=FastAPIWebsocketParams(
    serializer=TwilioFrameSerializer(...)   # ← plugged in here
))
```

---

### 🥇 #1 — Twilio (Programmable Voice + Media Streams)

**What it does:**

- Twilio dials the candidate's phone via PSTN (any Indian number — Jio, Airtel, Vi, landline)
- Once the call is answered, Twilio opens a WebSocket to your FastAPI server and streams audio in real-time (8kHz µ-law G.711)
- Your Pipecat pipeline processes audio and streams the AI's response back through the same WebSocket
- Twilio plays the audio to the candidate and hangs up when done

**Capabilities:**

- ✅ Outbound PSTN calls to any phone number worldwide
- ✅ WebSocket Media Streams — real-time bidirectional audio
- ✅ REST API to initiate, monitor, and terminate calls
- ✅ India phone number (DID) support — local caller ID
- ✅ Pipecat integration via `TwilioFrameSerializer` (official, documented)
- ✅ DTMF support (touch-tone input if needed)
- ✅ Call recording via Twilio API
- ✅ Webhooks for call status (ringing, connected, completed, failed)

**Why Twilio is #1:**

- Most mature PSTN provider with the widest global coverage and India reliability
- Has the most Pipecat examples and community support
- Single REST call to start an interview: `twilio.calls.create(to=candidate_phone, from_=our_number, url=twiml_url)`
- Handles everything about the call — your backend only processes audio

**Limitations:**

- Higher cost for India outbound ($0.0135–0.091/min) — see Plivo below for savings
- Manual TRAI compliance setup required (AI disclosure, DNC scrubbing)
- Audio quality capped at 8kHz µ-law (telephone quality) — no HD audio

**India pricing:**

| Call Type                  | Rate/min         |
| -------------------------- | ---------------- |
| Outbound to India mobile   | $0.0135 – $0.091 |
| Outbound to India landline | $0.0135          |
| India DID number (monthly) | $2.00/month      |

**Pipecat integration:**

```python
from pipecat.transports.network.fastapi_websocket import FastAPIWebsocketTransport, FastAPIWebsocketParams
from pipecat.serializers.twilio import TwilioFrameSerializer

serializer = TwilioFrameSerializer(
    stream_sid=stream_sid, call_sid=call_sid,
    account_sid=TWILIO_SID, auth_token=TWILIO_TOKEN
)
transport = FastAPIWebsocketTransport(
    websocket=ws,
    params=FastAPIWebsocketParams(serializer=serializer, vad_enabled=True)
)
```

> ⚠️ `TwilioFrameSerializer` is NOT a full Pipecat transport — it's a serializer that sits on top of `FastAPIWebsocketTransport`. It auto-converts Twilio's **8kHz µ-law → 16kHz PCM** (Pipecat's working format) and back.

---

### 🥈 #2 — Plivo (or Exotel for India-native)

**What it does:**
Same core function as Twilio — PSTN outbound calls to candidates, audio streamed via WebSocket to your Pipecat server. Drop-in replacement via `PlivoFrameSerializer` or `ExotelFrameSerializer` in Pipecat.

**Capabilities:**

- ✅ Same PSTN outbound calling as Twilio
- ✅ WebSocket audio streaming (same pattern as Twilio)
- ✅ India data center (Bangalore) — lower latency for India calls
- ✅ Built-in TRAI compliance tools (especially Exotel)
- ✅ Pipecat serializer available (`PlivoFrameSerializer`, `ExotelFrameSerializer`)
- ✅ India toll-free number (1800) faster setup via Exotel vs Twilio
- ✅ 50–70% cheaper India outbound rates

**India pricing (Plivo):**

| Call Type                  | Rate/min         |
| -------------------------- | ---------------- |
| Outbound to India mobile   | $0.004 – $0.012  |
| Outbound to India landline | $0.005           |
| India DID number           | $1.00–1.50/month |

**Why choose Plivo/Exotel over Twilio:**

- Cost: 50–70% cheaper for India outbound — at 1,000 interviews/month this is significant
- Exotel is built specifically for Indian telecoms — handles TRAI registration, KYC, local compliance natively
- India DC means lower call latency from Indian networks

**Why Twilio is still #1:**

- More mature Pipecat tooling and examples
- Better global reach if interviews extend to non-India candidates
- More battle-tested reliability and documentation

**Code change to switch from Twilio to Plivo:** Only the serializer changes — pipeline stays identical.

```python
# Swap this line only:
from pipecat.serializers.plivo import PlivoFrameSerializer  # instead of TwilioFrameSerializer
```

📚 [`docs/transport-layer-research.md`](./transport-layer-research.md) · [Pipecat Twilio Docs](https://docs.pipecat.ai/pipecat/telephony/twilio-websockets)

---

## 3. STT — Speech to Text

### Comparison Table

| Feature                            | Sarvam Saaras v3      | Deepgram Nova-3                    | Whisper                | Google Chirp          |
| ---------------------------------- | --------------------- | ---------------------------------- | ---------------------- | --------------------- |
| Hindi accuracy                     | ★★★★★                 | ★★★★☆                              | ★★★★☆                  | ★★★★☆                 |
| Marathi accuracy                   | ★★★★★                 | ★★★☆☆                              | ★★★☆☆                  | ★★★★☆                 |
| Code-switching (Hindi+Marathi+Eng) | ✅ Native `codemix`   | ⚠️ `language=multi` (weak Marathi) | ❌ No real-time        | ❌ No codemix         |
| Latency from India                 | ~300ms (India DC)     | **150–300ms** (US DC)              | 500ms–1s               | ~400ms                |
| WER on Indian speech               | ~19% (IndicVoices)    | ~20–25% (India)                    | ~22%+                  | ~18–22%               |
| Pipecat native                     | ✅ `SarvamSTTService` | ✅ `DeepgramSTTService`            | ✅ `WhisperSTTService` | ✅ `GoogleSTTService` |
| Pricing                            | ₹15/10K chars         | Usage-based (US pricing)           | Free (self-hosted)     | $0.016/15sec          |

---

### 🥇 #1 — Sarvam Saaras v3 (codemix mode)

**What it does:**

- Streams audio from the phone call → returns transcript in real-time
- `codemix` mode transcribes Hindi, Marathi, and English simultaneously, preserving each script exactly as spoken
- Runs from India servers — ~300ms latency vs 1,830ms for US-based endpoints

**Capabilities:**

- ✅ Native Hindi/Marathi/English code-switching — no other provider has this
- ✅ Preserves Devanagari script for Hindi/Marathi words and Latin for English words
- ✅ India-local inference — ~952ms total E2E (including network) vs Deepgram's 1,830ms from India
- ✅ Real-time streaming (not batch) — word-by-word output while candidate still speaking
- ✅ Telephony-grade audio support (handles 8kHz µ-law from Twilio)
- ✅ Native Pipecat service (`SarvamSTTService`)
- ✅ WER ~19% on IndicVoices benchmark (~25–35% expected on real 8kHz phone audio — normal for telephony)

**The unique advantage — `codemix` output:**

```
Input speech:  "मेरे पास 5 years का experience है in Python"
Saaras output: "मेरे पास 5 years का experience है in Python"
                ↑ Devanagari preserved              ↑ Latin preserved
```

GPT-4o receives the transcript exactly as the candidate spoke it — no translation or script normalization.

**Setup:**

```python
pip install "pipecat-ai[sarvam]"
```

```python
from pipecat.services.sarvam.stt import SarvamSTTService

stt = SarvamSTTService(
    api_key=SARVAM_API_KEY,
    model="saaras:v3",
    mode="codemix"   # handles Hindi/Marathi/English mixing natively
)
```

> ⚠️ WER benchmarks (~19%) are from IndicVoices dataset. Real phone-call performance (8kHz µ-law) will be 25–35% WER — this is normal for telephone audio across all providers.

---

### 🥈 #2 — Deepgram Nova-3 (language=multi)

**What it does:**
Same role — real-time streaming STT from phone audio. Deepgram is US-based but is the fastest STT provider globally and has good Hindi support.

**Capabilities:**

- ✅ Fastest STT available — 150–300ms latency (though from US DC, round-trip from India adds ~200ms)
- ✅ Good Hindi accuracy
- ✅ `language=multi` flag supports multilingual input
- ✅ Very mature streaming API, widely used in production voice agents
- ✅ Native Pipecat service (`DeepgramSTTService`)
- ⚠️ Marathi support is partial — works for common phrases, struggles with regional accents
- ⚠️ No `codemix` mode — code-switched output may be inconsistent in script (sometimes transliterates Hindi to Latin)

**When to choose Deepgram over Sarvam:**

- Candidate speaks primarily English or Hinglish (no pure Hindi/Marathi speakers in your target pool)
- Sarvam downtime or rate limit issues — use Deepgram as fallback
- Absolute minimum latency is the top priority

```python
from pipecat.services.deepgram import DeepgramSTTService
from deepgram import LiveOptions

stt = DeepgramSTTService(
    api_key=DEEPGRAM_API_KEY,
    live_options=LiveOptions(model="nova-3", language="multi", smart_format=True)
)
```

📚 [`docs/multilingual-support-research.md`](./multilingual-support-research.md) · [`docs/sarvam-ai-deep-research.md`](./sarvam-ai-deep-research.md) · [Sarvam Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/stt/sarvam) · [Deepgram Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/stt/deepgram)

---

## 4. LLM — The Interviewer Brain

### Comparison Table

| Feature                          | GPT-4o                | Claude 3.5 Sonnet        | Gemini 1.5 Pro        |
| -------------------------------- | --------------------- | ------------------------ | --------------------- |
| Context window                   | 128K tokens           | 200K tokens              | 1M+ tokens            |
| 45-min interview headroom        | ✅ 3–4×               | ✅ 5–6×                  | ✅ 20×+               |
| Structured JSON output           | ✅ Excellent          | ✅ Excellent             | ✅ Good               |
| Hindi/Marathi comprehension      | ✅ Good               | ✅ Good                  | ✅ Good               |
| Follow-up question quality       | ✅ Excellent          | ✅ Excellent             | ✅ Good               |
| Latency (first token, streaming) | ~200–500ms            | ~300–600ms               | ~300–500ms            |
| Pipecat native service           | ✅ `OpenAILLMService` | ✅ `AnthropicLLMService` | ✅ `GoogleLLMService` |
| Cost (input / output tokens)     | $2.50 / $10 per 1M    | $3 / $15 per 1M          | $1.25 / $5 per 1M     |

---

### 🥇 #1 — GPT-4o (128K context, gpt-4o)

**What it does:**

- Receives the full conversation history (`messages[]`) + system prompt (job description, candidate resume, scoring rubric)
- Generates the next interview question, follow-up, or clarifying probe
- At session end: generates a structured evaluation JSON (scores, reasoning, recommendation)

**Capabilities:**

- ✅ 128K context window — holds entire 45-min multilingual interview with 3–4× headroom
- ✅ Streaming output — starts generating before full response is ready (reduces TTFA)
- ✅ Excellent structured JSON output via `response_format={"type": "json_object"}` — critical for scoring rubrics
- ✅ Strong instruction following — follows complex interview rubrics and branching logic
- ✅ Good comprehension of code-switched Hindi/Marathi/English transcripts
- ✅ Best-documented Pipecat integration (`OpenAILLMService`) with most community examples
- ✅ Function calling — can trigger structured scoring mid-interview if needed

**Setup:**

```python
from pipecat.services.openai import OpenAILLMService
from pipecat.processors.aggregators.openai_llm_context import OpenAILLMContext

llm = OpenAILLMService(api_key=OPENAI_API_KEY, model="gpt-4o")
messages = [{"role": "system", "content": system_prompt}]  # system_prompt has JD + rubric
context = OpenAILLMContext(messages)
context_aggregator = llm.create_context_aggregator(context)
```

**Why GPT-4o is #1:**

- Most mature Pipecat tooling with `OpenAILLMService`
- Most community examples for voice agent use cases
- Proven structured output reliability at production scale

---

### 🥈 #2 — Claude 3.5 Sonnet (claude-3-5-sonnet-20241022)

**What it does:**
Same role as GPT-4o — the interviewer brain. Near-identical output quality for interview tasks.

**Capabilities:**

- ✅ 200K context window — 50% larger than GPT-4o; more headroom for very long sessions or extra context
- ✅ Widely considered slightly better at nuanced instruction following and rubric adherence
- ✅ Strong structured output
- ✅ Good comprehension of multilingual code-switched input
- ✅ Native Pipecat service (`AnthropicLLMService`)
- ⚠️ Slightly higher first-token latency (~300–600ms vs GPT-4o's ~200–500ms)
- ⚠️ Less Pipecat community examples than GPT-4o

**When to choose Claude over GPT-4o:**

- Your interview rubrics are very complex (multi-level branching, detailed follow-up logic) — Claude's instruction following may be marginally better
- Interviews regularly exceed 90 minutes and you need the larger 200K window
- You want to compare quality in an A/B test after building on GPT-4o first

```python
from pipecat.services.anthropic import AnthropicLLMService

llm = AnthropicLLMService(api_key=ANTHROPIC_API_KEY, model="claude-3-5-sonnet-20241022")
```

> ⚠️ **Not yet researched**: Direct benchmark of GPT-4o vs Claude 3.5 Sonnet for interview question generation quality and evaluation scoring accuracy. This is the **recommended next research topic** before production.

📚 [Pipecat OpenAI Docs](https://docs.pipecat.ai/api-reference/server/services/llm/openai) · [Pipecat Anthropic Docs](https://docs.pipecat.ai/api-reference/server/services/llm/anthropic) · [`docs/memory-context-management-research.md`](./memory-context-management-research.md)

---

## 5. TTS — Voice Output

### Comparison Table

| Feature                      | Sarvam Bulbul v2       | ElevenLabs Flash v2        | Google WaveNet        | Azure Neural TTS     |
| ---------------------------- | ---------------------- | -------------------------- | --------------------- | -------------------- |
| Hindi voice quality          | ★★★★★                  | ★★★★☆                      | ★★★★☆                 | ★★★★☆                |
| Marathi voice quality        | ★★★★★                  | ★★★☆☆                      | ★★★★☆                 | ★★★☆☆                |
| Indian accent authenticity   | ✅ Native Indian       | ✅ Good (global-trained)   | ✅ Good               | ✅ Good              |
| Code-mixed output (Hinglish) | ✅ Natural             | ⚠️ Partial                 | ⚠️ Partial            | ⚠️ Partial           |
| Latency (first audio byte)   | ~300–500ms             | **75–300ms** (Flash)       | ~400ms                | ~350ms               |
| Cost                         | ₹15/10K chars (~$0.18) | $0.11/1K chars (expensive) | $0.004/char           | $0.004/char          |
| Pipecat native               | ✅ `SarvamTTSService`  | ✅ `ElevenLabsTTSService`  | ✅ `GoogleTTSService` | ✅ `AzureTTSService` |
| Streaming                    | ✅ Yes                 | ✅ Yes                     | ✅ Yes                | ✅ Yes               |

---

### 🥇 #1 — Sarvam Bulbul v2

**What it does:**

- Takes the LLM's text response → generates Hindi/English/Marathi speech audio
- Streams audio chunks back through Twilio to the candidate's phone
- Bulbul v2 is purpose-trained on Indian speech — sounds natural to Indian candidates

**Capabilities:**

- ✅ Native Hindi and Marathi TTS — purpose-trained on Indian speech patterns, not globally trained
- ✅ Handles code-mixed text naturally ("आपका experience kya है?") without pronunciation errors
- ✅ Regional accent variants — Mumbai, Delhi, Pune-flavored voices available
- ✅ Streaming support — audio starts playing before full text is generated
- ✅ Multiple voice profiles: professional, conversational, formal
- ✅ Native Pipecat service (`SarvamTTSService`)
- ✅ India-local inference — lower latency vs US-based TTS

**Why Bulbul is #1 for phone interviews:**

- Phone calls run at 8kHz µ-law (telephone quality) — this naturally caps perceived audio fidelity. An expensive, high-quality TTS voice is partially "wasted" on a phone call. Bulbul's quality is more than sufficient for this channel.
- Indian candidates will find Bulbul's accent more natural and relatable than a globally-trained voice
- 5–10× cheaper than ElevenLabs at the same audio quality level on phone

```python
from pipecat.services.sarvam.tts import SarvamTTSService

tts = SarvamTTSService(
    api_key=SARVAM_API_KEY,
    model="bulbul:v2",
    language="hi-IN",   # or "mr-IN" — switchable per-response
    voice="anila"        # professional female voice; options: anila, meera, etc.
)
```

---

### 🥈 #2 — ElevenLabs Flash v2

**What it does:**
Same role — TTS from LLM text output. ElevenLabs is the gold standard for voice quality globally.

**Capabilities:**

- ✅ Best voice quality globally — most natural, human-like output of any TTS provider
- ✅ Flash v2 has 75–300ms TTFA — fastest TTS available (vs Bulbul's 300–500ms)
- ✅ Hindi support (good quality, though not India-regional-accent specific)
- ✅ Voice cloning — can clone a specific voice if you want a branded interviewer persona
- ✅ Native Pipecat service (`ElevenLabsTTSService`)
- ⚠️ Marathi support is limited
- ⚠️ Most expensive TTS option — $0.11/1K chars vs Bulbul's ₹15/10K chars
- ⚠️ US-based endpoint — adds ~150–200ms round-trip from India vs Bulbul's India-local

**When to choose ElevenLabs over Bulbul:**

- Premium voice quality is the top priority (e.g., high-value executive interviews, not mass screening)
- You want voice cloning to give the AI agent a distinct brand identity
- TTS latency is the bottleneck and you need Flash v2's speed (check after profiling the real pipeline)

```python
from pipecat.services.elevenlabs import ElevenLabsTTSService

tts = ElevenLabsTTSService(
    api_key=ELEVENLABS_API_KEY,
    voice_id="21m00Tcm4TlvDq8ikWAM",  # Rachel — or a custom cloned voice
    model="eleven_flash_v2_5"           # lowest latency model
)
```

📚 [`docs/sarvam-ai-deep-research.md`](./sarvam-ai-deep-research.md) · [Sarvam TTS Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/tts/sarvam) · [ElevenLabs Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/tts/elevenlabs)

---

## 6. VAD — Voice Activity Detection

### Decision: ✅ Silero VAD (built into Pipecat)

Silero VAD detects when the candidate **starts** and **stops** speaking on the phone call. This is what prevents the AI from talking over the candidate and ensures natural turn-taking.

```python
from pipecat.audio.vad.silero import SileroVADAnalyzer

vad = SileroVADAnalyzer()
# Pipecat integrates this automatically with transport input
```

- **Cost**: Free (runs locally, open-source)
- **Latency added**: ~20–50ms
- **Accuracy**: Production-grade for phone-quality audio
- **No alternative needed** — Silero is the standard for Pipecat voice agents

> ⚠️ **Not yet researched**: VAD fine-tuning for phone call characteristics — phone calls introduce different noise floors and acoustic artifacts vs WebRTC. Silero handles this well in general, but specific threshold tuning for 8kHz Twilio audio may be needed in production.

📚 [Pipecat VAD Docs](https://docs.pipecat.ai/guides/features/vad)

---

## 7. Memory & Context Management

### 7.1 Within a Single 45-Minute Interview

**No special tooling needed.** GPT-4o's 128K context window handles the full session natively.

**Token math:**

```
45-min multilingual interview ≈ 20,000–35,000 tokens
GPT-4o limit: 128,000 tokens → ~3–4× headroom. Limit hit at ~66 min of Hindi/Marathi.
```

Pipecat automatically manages the `messages[]` array — every turn is appended and passed to GPT-4o on each request.

```python
# Pipecat manages context automatically — no extra setup needed
messages = [{"role": "system", "content": system_prompt}]
context = OpenAILLMContext(messages)
context_aggregator = llm.create_context_aggregator(context)
```

> ⚠️ **"Lost in the middle" problem**: LLMs recall information at the start and end of context better than the middle. Mitigate with a running `interview_notes` JSON object in the system prompt that gets updated each turn with key facts extracted.

---

### 7.2 Cross-Round Memory (Round 1 → Round 2)

This is where specialized memory tooling helps. Round 2's agent needs to know what happened in Round 1 without replaying the full 45-min transcript.

### Cross-Round Comparison

| Feature              | Mem0                    | Supermemory                | PostgreSQL + pgvector |
| -------------------- | ----------------------- | -------------------------- | --------------------- |
| Pipecat native       | ✅ `Mem0MemoryService`  | ✅ `SupermemoryService`    | ❌ Manual setup       |
| Semantic search      | ✅ Vector similarity    | ✅ Vector similarity       | ✅ (if pgvector)      |
| Automatic injection  | ✅ Into system prompt   | ✅ Into system prompt      | ❌ Manual             |
| User/session scoping | ✅ `user_id` + `run_id` | ✅ `user_id` + `space_id`  | ✅ Custom SQL         |
| Long-term profiles   | ✅ Good                 | ✅ `profile` mode (better) | ✅ Custom             |
| Self-hosted option   | ❌ Cloud only           | ❌ Cloud only              | ✅ Full control       |

---

### 🥇 #1 Cross-Round Memory — Mem0

**What it does:**

- After Round 1 ends: automatically extracts and stores key facts ("Python expert, 5 years, Infosys, weak on system design")
- Before Round 2 starts: retrieves and injects those memories into the GPT-4o system prompt
- Uses semantic vector search — Round 2 can query "leadership experience" and find relevant memories even if Round 1 used different words

**Capabilities:**

- ✅ Native Pipecat integration — drop into pipeline with no extra backend code
- ✅ Semantic search retrieval — not just keyword matching, meaning-based
- ✅ Scoped by `user_id` (candidate) + `run_id` (interview round) — clean separation
- ✅ Auto-injects retrieved memories into system prompt as a system message
- ✅ Works across interview rounds weeks or months apart
- ✅ Managed cloud service — no vector DB infrastructure to run

```python
pip install "pipecat-ai[mem0]"
```

```python
from pipecat.services.mem0 import Mem0MemoryService

memory = Mem0MemoryService(
    api_key=MEM0_API_KEY,
    user_id=candidate_id,         # same candidate across all rounds
    run_id=f"round_{round_num}",  # unique per session
    params={"search_limit": 10, "add_as_system_message": True}
)
# Pipeline: stt → context_aggregator.user() → memory → llm → tts
```

**What Mem0 does automatically:**

- After Round 1: saves "Python 5 years, Infosys, weak system design, strong communication"
- Before Round 2: injects those facts into GPT-4o's context → Round 2 interviewer is pre-informed

---

### 🥈 #2 Cross-Round Memory — Supermemory

**What it does:**
Same role as Mem0 — cross-round memory retrieval and injection. Supermemory has a `profile` mode that maintains a persistent candidate profile that evolves across all rounds.

**Capabilities:**

- ✅ Native Pipecat integration (`SupermemoryService`)
- ✅ `profile` mode — builds a growing candidate profile across all interactions (better for 3+ round interviews)
- ✅ `space_id` for organizing memories by project/job role
- ✅ Semantic vector search retrieval
- ✅ Auto-injection into system prompt
- ⚠️ Less community examples in Pipecat context than Mem0

**When to choose Supermemory over Mem0:**

- 3+ interview rounds and you want a rich evolving candidate profile
- Need to organize memories by job role or department (`space_id`)
- Want a candidate profile that persists indefinitely (e.g., re-interview same candidate 6 months later)

```python
pip install "pipecat-ai[supermemory]"
```

```python
from pipecat.services.supermemory import SupermemoryService

memory = SupermemoryService(
    api_key=SUPERMEMORY_API_KEY,
    user_id=candidate_id,
    space_id=f"job_{job_role_id}",  # organize by role
    params={"search_limit": 10}
)
```

📚 [`docs/memory-context-management-research.md`](./memory-context-management-research.md) · [Mem0 Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/memory/mem0) · [Supermemory Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/memory/supermemory)

---

## 8. Full Pipeline (Putting It All Together)

```
Recruiter taps "Start Interview" in React Native App
         ↓ REST API call
FastAPI Backend → Twilio: "Call +91-9XXXXXXXXX"
         ↓ Candidate picks up
Twilio WebSocket opens
         ↓
┌────────────────── PIPECAT PIPELINE ──────────────────┐
│                                                       │
│  FastAPIWebsocketTransport (Twilio MediaStream)       │
│      + TwilioFrameSerializer (8kHz µ-law ↔ 16kHz PCM)│
│                     ↓                                 │
│  Silero VAD  (detects when candidate stops speaking)  │
│                     ↓                                 │
│  Sarvam Saaras v3  (STT, codemix mode)               │
│     "नमस्ते! मैंने 5 साल Python में काम किया है"     │
│                     ↓                                 │
│  LLMUserContextAggregator  (builds messages[])        │
│                     ↓                                 │
│  [Mem0]  (retrieves prior round memories if Round 2+) │
│                     ↓                                 │
│  GPT-4o 128K  (generates next question / evaluation)  │
│                     ↓                                 │
│  Sarvam Bulbul v2  (TTS, Hindi/English voice)         │
│                     ↓                                 │
│  FastAPIWebsocketTransport.output()                   │
│      → audio back to Twilio → to candidate's phone   │
│                                                       │
└───────────────────────────────────────────────────────┘
         ↓ Interview ends (45 min)
FastAPI saves transcript + evaluation JSON → PostgreSQL
Mem0 stores key memories for next round
Recruiter sees results in React Native App
```

**Minimal Pipecat code:**

```python
pipeline = Pipeline([
    transport.input(),
    silero_vad,
    sarvam_stt,
    context_aggregator.user(),
    mem0_memory,            # only for multi-round
    gpt4o_llm,
    sarvam_tts,
    transport.output(),
    context_aggregator.assistant(),
])
```

---

## 9. Latency Budget

```
Candidate finishes speaking
        ↓  ~20ms    Silero VAD: detects end of speech
        ↓  ~300ms   Sarvam Saaras v3: transcribes audio (India endpoint)
        ↓  ~300ms   GPT-4o: generates first token (streaming)
        ↓  ~400ms   Sarvam Bulbul v2: first audio chunk
        ↓  ~150ms   Twilio: delivers audio to candidate's phone
        ↓
     Total ≈ 1.2 – 2.2 seconds
```

This is within the natural conversational pause range for a **phone interview** (humans pause 1–3 seconds between turns). Candidates will not notice this as unnatural.

---

## 10. Cost Estimate (Per Interview Session)

Assuming: 45-minute multilingual interview, ~3 turns/minute = ~135 turns

| Component                    | Cost/session        | Basis                                 |
| ---------------------------- | ------------------- | ------------------------------------- |
| Twilio outbound call (India) | $0.61 – $4.10       | 45 min × $0.0135–$0.091/min           |
| Sarvam Saaras v3 STT         | ~₹3–5 (~$0.04–0.06) | ~135 turns × ~300 chars × ₹15/10K     |
| GPT-4o LLM                   | ~$0.10 – $0.25      | ~20K tokens × $5–15/1M tokens         |
| Sarvam Bulbul v2 TTS         | ~₹2–4 (~$0.02–0.05) | ~135 responses × ~200 chars × ₹15/10K |
| **Total/session**            | **~$0.75 – $4.50**  | Twilio dominates; Plivo halves it     |

> 💡 Switching Twilio → Plivo reduces the telephony cost by ~50–70%, making the total closer to **$0.40 – $2.00/session** for India outbound.

---

## 11. What's Not Yet Researched

| Topic                            | Priority  | Notes                                                                    |
| -------------------------------- | --------- | ------------------------------------------------------------------------ |
| **LLM benchmark for interviews** | 🔴 High   | GPT-4o vs Claude 3.5 for interview question quality, evaluation accuracy |
| **VAD tuning for phone audio**   | 🔴 High   | 8kHz Twilio audio characteristics, silence thresholds, barge-in handling |
| **Interview prompt engineering** | 🔴 High   | System prompt structure for structured interviews, follow-up logic       |
| **Scoring & evaluation design**  | 🟡 Medium | Rubric design, structured output schema, scoring consistency             |
| **Deployment & scaling**         | 🟡 Medium | Docker, concurrent Twilio calls, pipeline resource limits                |
| **Error handling**               | 🟡 Medium | Twilio disconnects, STT failures, LLM timeouts mid-interview             |

---

## 12. References

### Our Research Docs

- [`docs/speech-pipeline-research.md`](./speech-pipeline-research.md) — S2S vs Pipeline full comparison
- [`docs/multilingual-support-research.md`](./multilingual-support-research.md) — Hindi/Marathi/English code-switching, Sarvam deep dive
- [`docs/sarvam-ai-deep-research.md`](./sarvam-ai-deep-research.md) — Sarvam accuracy benchmarks, pricing, Pipecat integration
- [`docs/memory-context-management-research.md`](./memory-context-management-research.md) — Token math, context strategies, Mem0, S2S context limits
- [`docs/transport-layer-research.md`](./transport-layer-research.md) — Twilio, Daily.co, LiveKit, React Native transports
- [`docs/master-architecture.md`](./master-architecture.md) — End-to-end flow with component roles

### Official Documentation

- [Pipecat GitHub](https://github.com/pipecat-ai/pipecat) — Framework source
- [Pipecat Docs](https://docs.pipecat.ai/) — Official documentation
- [Pipecat Twilio WebSocket Integration](https://docs.pipecat.ai/pipecat/telephony/twilio-websockets)
- [Pipecat VAD Guide](https://docs.pipecat.ai/guides/features/vad)
- [Sarvam AI Docs](https://docs.sarvam.ai/) — Saaras v3, Bulbul v2 API reference
- [Sarvam STT — Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/stt/sarvam)
- [Sarvam TTS — Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/tts/sarvam)
- [Mem0 Pipecat Integration](https://docs.pipecat.ai/api-reference/server/services/memory/mem0)
- [Twilio Programmable Voice — India Pricing](https://www.twilio.com/en-us/voice/pricing/in)
- [Twilio Media Streams Docs](https://www.twilio.com/docs/voice/media-streams)
- [OpenAI GPT-4o API](https://platform.openai.com/docs/models/gpt-4o)
- [Deepgram Multilingual Code-Switching](https://developers.deepgram.com/docs/multilingual-code-switching)
- [Gemini Live Pipecat Service](https://docs.pipecat.ai/api-reference/server/services/llm/google-gemini-multimodal-live)
