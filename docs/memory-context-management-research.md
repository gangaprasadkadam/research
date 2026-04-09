# Memory & Context Management Research: 45-Minute AI Interview Agent

## 📋 What This Document Is For
This document answers two questions: **(1) Will the LLM "forget" earlier parts of the interview in a 45-minute conversation? (2) How does the AI agent in Round 2 know what happened in Round 1?** It covers token math, context window limits, and the tools available in Pipecat for memory management.

> Read this after the LLM section of final-recommendations. It explains the memory strategy in detail.

## 📑 What It Covers
- **Section 1** — The problem: what context challenges a 45-min interview creates
- **Section 2** — Token math: how many tokens does a 45-min multilingual interview use?
- **Section 3** — LLM context window comparison (GPT-4o, Claude, Gemini)
- **Section 4** — Types of memory needed (within-session vs. cross-round vs. long-term)
- **Section 5** — Context management strategies (full context, sliding window, summarization, RAG)
- **Section 6** — Pipecat memory tools (Mem0 and Supermemory — code included)
- **Section 7** — Recommended architecture for the interview agent
- **Section 8** — Context capacity planning (how long before the window fills up)
- **Section 9** — The "lost in the middle" problem and how to mitigate it
- **Section 10** — Context management in S2S mode (Gemini Live 15-min limit, OpenAI Realtime)
- **Section 11** — Summary and recommendations

---

## 1. The Problem: Context in a 45-Minute Interview

A 45-minute interview with an AI agent creates these memory challenges:

1. **Within-session continuity** — Agent must remember what was asked and answered earlier in the same interview (e.g., "You mentioned Python earlier — tell me more about that project")
2. **Context window limits** — LLMs have finite memory; at some point older messages get cut off
3. **Cross-session memory** — For multi-round interviews (technical + HR + culture fit), the agent in round 2 must know what was discussed in round 1
4. **Real-time retrieval latency** — Memory lookups must be fast enough to not break the voice latency budget

---

## 2. How Much Context Does a 45-Minute Interview Actually Use?

### Token estimation for a 45-minute voice interview:

```
Average interview speech rate: ~140 words/minute
Both sides (candidate + agent): ~200–250 words/minute combined

45 minutes × 225 words/min = ~10,125 words

Token conversion: 1 token ≈ 0.75 words
→ ~10,125 ÷ 0.75 = ~13,500 tokens from raw dialogue

Add system prompt (job description, resume, rubric): ~2,000–4,000 tokens
Add LLM responses already generated: included above

TOTAL ESTIMATE: 15,000 – 20,000 tokens for a full 45-min interview
```

### Hindi/Marathi adds overhead:
- Hindi tokens: GPT-4o uses ~1.5–2× tokens per word vs English (script complexity)
- Marathi tokens: Similar to Hindi
- Code-mixed content: ~1.3× multiplier
- **Revised estimate for multilingual interview: 20,000 – 35,000 tokens**

---

## 3. LLM Context Window Capacity

| Model | Context Window | Estimated 45-min interviews fit |
|---|---|---|
| **GPT-4o** | 128,000 tokens | **4–6 interviews** — plenty of headroom |
| **GPT-4.1** | 1,000,000 tokens | 30+ interviews |
| **Claude 3.5 Sonnet** | 200,000 tokens | 6–10 interviews |
| **Claude 3 Opus** | 1,000,000 tokens (beta) | 30+ |
| **Gemini 2.5 Pro** | 1,000,000–2,000,000 tokens | 30–60+ |
| **Llama 4 Scout** | 10,000,000 tokens | Self-hosted only |

### ✅ Key Finding:
**A single 45-minute interview (20,000–35,000 tokens) fits comfortably within GPT-4o's 128K context window.** You have ~4× headroom before context window management is even needed for a single session.

**The real challenge is:**
1. Cross-session memory (round 1 → round 2 → round 3)
2. "Lost-in-the-middle" degradation as context fills up
3. Efficient retrieval without adding latency to voice turns

---

## 4. Types of Memory Needed for an Interview Agent

```
┌─────────────────────────────────────────────────────────┐
│                    MEMORY LAYERS                         │
│                                                         │
│  [Working Memory]     Current turn context              │
│       ↕                                                 │
│  [Short-term Memory]  Current session (45 min)         │
│       ↕                                                 │
│  [Long-term Memory]   Cross-session (round 1→2→3)      │
│       ↕                                                 │
│  [External Knowledge] Candidate resume, JD, rubric     │
└─────────────────────────────────────────────────────────┘
```

### 4.1 Working Memory
- The last 2–3 conversation turns in the immediate LLM prompt
- Handled automatically by Pipecat's context aggregators
- Zero additional latency

### 4.2 Short-term Memory (Within Session)
- Full conversation history of the current 45-minute interview
- Stored in the `messages[]` array passed to LLM
- GPT-4o handles this natively — no extra tooling needed for a single session

### 4.3 Long-term Memory (Cross-session)
- Persist key facts, answers, and impressions from previous interview rounds
- Requires external memory storage (Mem0, Supermemory, or custom DB)
- Retrieved and injected into system prompt at session start

### 4.4 External Knowledge (Injected Context)
- Candidate's resume
- Job description
- Interview rubric/scoring criteria
- Previous round notes
- These are injected once at session start as structured system prompt content

---

## 5. Context Management Strategies

### 5.1 Full Context (Default — Recommended for Single Session)

```
[System Prompt: JD + Resume + Rubric]
[Turn 1] User: "..." | Agent: "..."
[Turn 2] User: "..." | Agent: "..."
...
[Turn N] User: current question
→ LLM Response
```

**Works for:** A single 45-minute session with GPT-4o (128K context).
**Does NOT work for:** Multi-session interviews where earlier sessions must be recalled.

---

### 5.2 Sliding Window (Simple, for very long sessions)

Keep only the last N turns in context. Older turns are dropped.

```
Window = Last 15–20 turns (current session only)
Older turns = Discarded
```

| Pros | Cons |
|---|---|
| Simple to implement | Loses early interview context |
| Fast | "You mentioned X 30 min ago" impossible |
| No added latency | Not suitable for follow-up on early answers |

**Verdict**: ❌ Not suitable for an interview agent — we need to recall answers from the beginning of the interview.

---

### 5.3 Summarization (Recommended for very long sessions / Multi-session)

Periodically compress older turns into a summary and replace them.

```
Every 15 minutes or N turns:
  [Old turns 1-20] → [Summary: "Candidate discussed Python, 5yr experience, led team of 4..."]
  [Turns 21-30 remain full]
  [Turn 31+] Current conversation
```

**Implementation in Pipecat:**
```python
# Pipecat has built-in summarization support
# Triggered automatically when context exceeds threshold
from pipecat.processors.aggregators.llm_response import LLMAssistantContextAggregator
```

| Pros | Cons |
|---|---|
| Preserves key facts across full session | Summary may lose detail/nuance |
| Works for multi-hour sessions | Adds one LLM call per summarization |
| Good recall of topics | Code-mixed text may summarize less accurately |

**Verdict**: ✅ Good for single long sessions; combine with full context for single 45-min sessions.

---

### 5.4 RAG + Vector Store (Cross-session memory)

Store conversation facts as embeddings; retrieve on next session.

```
Round 1 Interview ends:
  → Key answers extracted → stored as embeddings in vector DB

Round 2 Interview starts:
  → Query: "What did candidate say about system design?" 
  → Retrieve top-k relevant memories
  → Inject into system prompt
```

| Pros | Cons |
|---|---|
| Unlimited long-term memory | Adds retrieval latency (30–100ms) |
| Cross-session recall | Semantic search may miss exact quotes |
| Scales to years of interactions | Requires vector DB infrastructure |

**Verdict**: ✅ Ideal for multi-round interviews (3 rounds across days).

---

### 5.5 Hybrid Architecture (Recommended for Production)

```
┌──────────────────────────────────────────────────────┐
│                 HYBRID MEMORY STACK                  │
│                                                      │
│  [System Prompt]                                     │
│    → Job Description                                 │
│    → Candidate Resume (extracted key points)         │
│    → Interview Rubric                                │
│    → Injected memories from previous rounds (Mem0)  │
│                                                      │
│  [Session History — Full, GPT-4o 128K]              │
│    → All turns from current 45-min session          │
│    → Auto-summarized if session goes >2 hours       │
│                                                      │
│  [Persistent Memory — Mem0]                         │
│    → Stored after each session                      │
│    → Retrieved at start of next session             │
│    → Semantic search by topic                       │
└──────────────────────────────────────────────────────┘
```

---

## 6. Pipecat Memory Tools

### 6.1 Default Context Management (Built-in)

Pipecat automatically manages conversation history via `LLMContext`:
- User messages appended after STT
- Assistant messages appended after TTS
- System prompt injected at conversation start
- Summarization available when context approaches limits

```python
# Pipecat context is automatically managed
messages = [
    {"role": "system", "content": system_prompt},
    # turns are added automatically by Pipecat aggregators
]

context = OpenAILLMContext(messages)
context_aggregator = llm.create_context_aggregator(context)
```

### 6.2 Mem0 — Persistent Memory Service (Native Pipecat Integration) ⭐

**Best for cross-session memory in interview agents.**

- Open-source + commercial
- Semantic memory search (vector similarity)
- Scoped by user_id, agent_id, session_id
- Asynchronous writes (no blocking latency)
- Graph-based memory option for relational facts

```python
pip install "pipecat-ai[mem0]"
```

```python
from pipecat.services.mem0 import Mem0MemoryService

memory = Mem0MemoryService(
    api_key=os.getenv("MEM0_API_KEY"),
    user_id=candidate_id,           # unique per candidate
    agent_id="interview_agent",
    run_id=session_id,              # unique per interview round
    params={
        "search_limit": 10,         # retrieve top 10 relevant memories
        "search_threshold": 0.1,    # minimum relevance score
        "system_prompt": "Previous interview context:\n",
        "add_as_system_message": True,
        "position": 1,             # inject after system, before history
    }
)

pipeline = Pipeline([
    transport.input(),
    stt,
    context_aggregator.user(),
    memory,          # retrieves past memories, writes new ones
    llm,
    tts,
    transport.output(),
    context_aggregator.assistant(),
])
```

📚 [Mem0 Pipecat Docs](https://docs.pipecat.ai/api-reference/server/services/memory/mem0) | [Mem0 Integration](https://docs.mem0.ai/integrations/pipecat)

---

### 6.3 Supermemory — Alternative Persistent Memory (Native Pipecat)

- Commercial + open-source
- Three modes: `profile` (user facts), `query` (semantic), `full` (both)
- Focused on personalization and cross-session recall

```python
pip install supermemory-pipecat
```

```python
from supermemory_pipecat import SupermemoryPipecatService

memory = SupermemoryPipecatService(
    api_key=os.getenv("SUPERMEMORY_API_KEY"),
    user_id=candidate_id,
    session_id=session_id,
    params=InputParams(
        mode="full",          # profile + semantic search
        search_limit=10,
        system_prompt="Based on previous interviews:\n\n"
    )
)
```

📚 [Supermemory Pipecat Docs](https://supermemory.ai/docs/integrations/pipecat) | [GitHub](https://github.com/supermemoryai/pipecat-memory)

---

## 7. Recommended Architecture for Interview Agent

### Single-session (45 minutes) — No special tooling needed

```python
# GPT-4o 128K easily holds a full 45-min interview
# Just use Pipecat's default full context

system_prompt = f"""
You are an AI interviewer.

JOB DESCRIPTION:
{job_description}

CANDIDATE RESUME:
{resume_summary}

INTERVIEW RUBRIC:
{scoring_rubric}

INTERVIEW INSTRUCTIONS:
- Ask one question at a time
- Probe follow-up based on the candidate's answer
- Respond in the same language the candidate uses (Hindi/Marathi/English/mix)
"""

messages = [{"role": "system", "content": system_prompt}]
context = OpenAILLMContext(messages)
# All 45-min turns fit comfortably in 128K window
```

---

### Multi-round (Round 1 → Round 2 → Round 3)

```python
# Session end: Mem0 auto-saves key memories
# Session start: Mem0 retrieves relevant past context

# At start of Round 2:
# Mem0 injects: "In Round 1, candidate mentioned 5 years of Python,
#  led a team of 4, built microservices for FinTech client..."

memory = Mem0MemoryService(
    api_key=MEM0_KEY,
    user_id=candidate_id,      # same candidate across rounds
    run_id=f"round_{round_number}",
    params={"search_limit": 15, "add_as_system_message": True}
)
```

---

## 8. Context Capacity Planning

### How long until you hit GPT-4o's 128K limit?

```
System prompt (JD + resume + rubric): ~3,000 tokens
Per interview turn: ~180 tokens (English) / ~300 tokens (Hindi/Marathi)

For multilingual interview:
  128,000 total ÷ 300 tokens/turn = ~420 turns before hitting limit
  At 3 turns/minute: ~140 minutes = 2.3 hours of conversation

A 45-minute interview uses: ~45 min × 3 turns/min × 300 tokens = ~40,500 tokens
→ GPT-4o 128K limit reached at ~66 minutes of Hindi/Marathi conversation
```

### ✅ Bottom line for 45-minute interviews:
- **GPT-4o 128K** is sufficient with ~30–40% buffer for multilingual
- **Claude or Gemini** give extra headroom if interviews run long
- **No summarization needed** for a typical 45-min session
- **Mem0 required** only if you need cross-session (multi-round) memory

---

## 9. The "Lost in the Middle" Problem

LLMs have a known weakness: **information in the middle of a long context is recalled less accurately than the beginning or end.**

For an interview, this means:
- Candidate's early answers may be partially "forgotten" by the LLM near the end
- GPT-4o mitigates this better than older models, but it's not fully solved

**Mitigation strategies:**
1. **Periodic summaries**: Every 15 min, summarize previous answers and re-inject as structured notes
2. **Structured tracking**: Maintain a running `interview_notes` JSON that gets updated each turn
3. **Explicit callbacks**: Prompt the LLM to explicitly reference the candidate's earlier answer when asking follow-ups

```python
# Structured tracking approach
interview_tracker = {
    "technical_skills_mentioned": ["Python", "Node.js", "PostgreSQL"],
    "experience_level": "5 years",
    "projects_discussed": ["FinTech microservices", "Mobile app"],
    "red_flags": [],
    "follow_ups_needed": ["team leadership experience"]
}

# Inject this as part of system prompt, updated each turn
```

---

## 10. Context Management in S2S (Speech-to-Speech) Mode

When using S2S services like OpenAI Realtime or Gemini Live instead of the STT+LLM+TTS pipeline, context management works fundamentally differently — and has **critical limitations** for a 45-minute interview.

---

### 10.1 How S2S Context Works (vs Pipeline)

```
PIPELINE (STT + LLM + TTS):
──────────────────────────────────────────────────────────
  Pipecat OWNS the messages[] array locally in memory:
  [system], [user turn 1], [assistant turn 1], ...
  
  • You can inspect, modify, summarize it anytime
  • Mem0 injects cross-session memories into it
  • Full text of every turn is always accessible
  • Pipecat auto-summarizes when context gets large

S2S (OpenAI Realtime / Gemini Live):
──────────────────────────────────────────────────────────
  The PROVIDER's server owns the session state:
  • Context lives inside a persistent WebSocket connection
  • Provider manages conversation history internally
  • You send audio → get audio back
  • Text transcript is optional — must be explicitly enabled
  • Context is lost when session/WebSocket ends
```

---

### 10.2 OpenAI Realtime API — Context Details

**How context is managed:**
- Each session maintains a full conversation history internally on OpenAI's servers
- `OpenAIRealtimeLLMContext` is Pipecat's context object for Realtime — extends base `LLMContext`
- Context is initialized at session start (system prompt, pre-loaded messages)
- Session properties (instructions, voice) can be updated mid-session via `session.update` events
- **Transcripts**: Emitted as `TranscriptionFrame` objects — can be captured and stored by your app

**Token/context limits:**
- Inherits the underlying model's context window
- GPT-4o Realtime: **128,000 tokens** (same as text API)
- GPT-4.1 Realtime: up to **1,000,000 tokens**
- When limit is hit: **oldest messages are automatically truncated** (no warning)

**Session lifecycle:**
- Session context lives in the WebSocket connection
- When WebSocket closes → context is gone from provider's side
- You must capture `TranscriptionFrame` events to preserve the transcript externally
- No built-in cross-session memory — must be implemented manually (or with Mem0)

**Transcript capture in Pipecat (OpenAI Realtime):**
```python
# Enable transcription in OpenAI Realtime settings
from pipecat.services.openai.realtime import OpenAIRealtimeLLMService

llm = OpenAIRealtimeLLMService(
    api_key=OPENAI_KEY,
    settings=OpenAIRealtimeLLMSettings(
        model="gpt-4o-realtime-preview",
        input_audio_transcription={"model": "whisper-1"},  # enable STT transcription
        instructions="You are an AI interviewer..."
    )
)

# Capture TranscriptionFrames in a pipeline processor to store them
class TranscriptSaver(FrameProcessor):
    async def process_frame(self, frame, direction):
        if isinstance(frame, TranscriptionFrame):
            await self.save_to_db(frame.text, frame.user_id)
        await self.push_frame(frame, direction)
```

**⚠️ Mem0 with OpenAI Realtime:**
- Mem0 is a pipeline processor that sits BETWEEN context aggregation and the LLM
- In S2S mode, there is no separate "before LLM" step — audio goes directly to OpenAI
- You **cannot** use Mem0's auto-injection with Realtime API natively
- Workaround: Manually inject past memories into `instructions` at session start

---

### 10.3 Gemini Live — Context Details

**How context is managed:**
- Entire conversation history is stored within the active WebSocket session
- Context window: **128K tokens** (standard) / up to **1M+ tokens** (advanced models)
- Context compression: Built-in sliding window + summarization when limit is approached

**⚠️ CRITICAL SESSION DURATION LIMITS:**

| Mode | Max Session Duration | Notes |
|---|---|---|
| Audio-only (without compression) | **15 minutes** | ⚠️ Hard limit — critical for 45-min interviews |
| Audio + video | **2 minutes** | Even shorter |
| With context window compression | Extendable | Requires explicit configuration |
| WebSocket connection limit | **10 minutes** | Must reconnect and resume |
| Session resumption token validity | **2 hours** | After previous session end |

> ⚠️ **This is a critical limitation**: A 45-minute interview with Gemini Live S2S requires:
> 1. Context window compression enabled
> 2. At least 3–4 WebSocket reconnections and session resumptions
> 3. Session resumption token management
> 4. Risk of context loss on reconnection failure

**Context compression (required for 45-min sessions):**
```python
# Gemini Live requires context window compression for sessions > 15 min
from pipecat.services.google.gemini_live import GeminiLiveLLMService

llm = GeminiLiveLLMService(
    api_key=GOOGLE_KEY,
    settings=GeminiLiveLLMSettings(
        model="gemini-2.0-flash-live-001",
        context_window_compression={
            "trigger_tokens": 100000,  # compress when 100K tokens accumulated
            "sliding_window": {"target_tokens": 80000}
        }
    )
)
```

**Session resumption (required for 45-min sessions):**
```python
# Must handle WebSocket reconnection every ~10 minutes
session_resumption_config = {
    "handle": previous_resumption_token,  # from previous WebSocket connection
}
# On each reconnection, use the token to restore session state
```

---

### 10.4 Comparison: Pipeline vs S2S Context for 45-Min Interviews

| Feature | Pipeline (Sarvam + GPT-4o) | OpenAI Realtime S2S | Gemini Live S2S |
|---|---|---|---|
| **Context owner** | Pipecat (local) | OpenAI's server | Google's server |
| **Transcript access** | ✅ Always — full text | ⚠️ Must enable TranscriptionFrames | ⚠️ Must extract |
| **Context inspection/modification** | ✅ Any time | ⚠️ Limited (session.update) | ⚠️ Limited |
| **45-min session support** | ✅ Native, no config | ✅ Yes (128K window handles it) | ⚠️ Requires compression + reconnection |
| **Session duration limit** | None | None (WebSocket can persist) | 15 min without compression |
| **Pipecat auto-summarization** | ✅ `enable_context_summarization=True` | ❌ Not applicable | ❌ Not applicable |
| **Mem0 cross-session memory** | ✅ Native Pipecat service | ⚠️ Manual injection at session start | ⚠️ Manual injection at session start |
| **Context lost on disconnect** | ❌ No — Pipecat holds it | ✅ Yes — must save externally | ✅ Yes — use resumption tokens |
| **Transcript storage** | ✅ Auto — always in messages[] | ⚠️ Custom TranscriptSaver needed | ⚠️ Custom extraction needed |
| **Context size for 45-min interview** | ~20–35K tokens (comfortable) | ~20–35K tokens (comfortable) | ~20–35K tokens + audio overhead |
| **Compliance/audit trail** | ✅ Easy | ⚠️ Requires extra work | ⚠️ Requires extra work |

---

### 10.5 Verdict: S2S Context Management for Interview Agent

**❌ Both S2S options add engineering complexity that the pipeline approach handles natively.**

For OpenAI Realtime:
- You can do 45 minutes, but transcript storage and cross-session memory require custom builds
- No Mem0 auto-injection — must manually inject prior round notes into `instructions`

For Gemini Live:
- **15-minute session limit** is a hard constraint — you MUST implement:
  - Context window compression (configuration)
  - Session resumption with token management (code)
  - Reconnection handling (reliability engineering)
- This adds significant complexity for an interview that "just needs to run for 45 minutes"

For STT+LLM+TTS pipeline:
- 45 minutes fits comfortably in GPT-4o 128K with ~3× buffer
- Transcripts are automatic — no extra code
- Mem0 works natively as a Pipecat service
- Pipecat summarization handles >60 min sessions

> **Final recommendation**: For the 45-minute AI interview agent, the **STT+LLM+TTS pipeline** approach avoids all these S2S context complications, while delivering better transcript access, memory management, and compliance — at lower cost.

---

## 11. Summary & Recommendations

| Scenario | Recommended Approach |
|---|---|
| Single 45-min interview | GPT-4o 128K + full context (no extra tools) |
| Multilingual (Hindi/Marathi) 45-min | GPT-4o 128K + Sarvam codemix + full context |
| Interview goes >60 min | Add periodic summarization via Pipecat built-in |
| Multi-round (3 rounds) | GPT-4o + Mem0 for cross-session memory |
| Multi-round + multilingual | GPT-4o + Sarvam + Mem0 (full recommended stack) |
| Recall specific past answers | Mem0 semantic search on candidate_id |

### Final recommended stack for production interview agent:

```
Session start:
  1. Inject system prompt (JD + resume + rubric)
  2. Mem0: retrieve previous round memories (if round 2+)
  3. Begin interview

During session:
  4. Full context maintained in GPT-4o 128K window
  5. Structured interview_notes JSON updated each turn
  6. Auto-summarize if context > 80K tokens (Pipecat built-in)

Session end:
  7. Mem0: store key facts and scores from this session
  8. Save full transcript to DB for review/compliance
  9. Generate structured evaluation report
```

---

## 11. Sources

- [Pipecat Context Management Docs](https://docs.pipecat.ai/pipecat/learn/context-management)
- [Pipecat Memory & Persistent Context — DeepWiki](https://deepwiki.com/pipecat-ai/pipecat/8.6-memory-and-persistent-context)
- [Mem0 Pipecat Service Docs](https://docs.pipecat.ai/api-reference/server/services/memory/mem0)
- [Mem0 Integration Guide](https://docs.mem0.ai/integrations/pipecat)
- [Supermemory Pipecat Integration](https://supermemory.ai/docs/integrations/pipecat)
- [Supermemory GitHub](https://github.com/supermemoryai/pipecat-memory)
- [Beyond the Context Window — Daily.co Blog](https://www.daily.co/blog/beyond-the-context-window-why-your-voice-agent-needs-structure-with-pipecat-flows/)
- [Mem0 Research Paper — arxiv](https://arxiv.org/html/2504.19413v1)
- [VoiceAgentRAG — Salesforce AI Research](https://arxiv.org/html/2603.02206v1)
- [Context Window Management Strategies — Maxim](https://www.getmaxim.ai/articles/context-window-management-strategies-for-long-context-ai-agents-and-chatbots/)
- [Context Window Management LLM Apps — Redis](https://redis.io/blog/context-window-management-llm-apps-developer-guide/)
- [LLM Context Window Comparison 2026 — Morph](https://www.morphllm.com/llm-token-limit)
- [Claude 1M Token Context Window for Agents](https://www.mindstudio.ai/blog/claude-1m-token-context-window-agents)
- [Memory for Voice Agents — Mem0 Blog](https://mem0.ai/blog/ai-memory-for-voice-agents)
