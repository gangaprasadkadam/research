# Master Architecture: AI Voice Interview Agent

## 📋 What This Document Is For
This document shows the **complete end-to-end system** in one place — every component, how they connect, who does what, and what the candidate and recruiter experience at each step. If you want to understand the full picture without reading all the research docs, start here.

> This is the architecture blueprint. The "why" behind each choice is in `final-recommendations.md`.

## 📑 What It Covers
- **Section 1** — The big picture (system diagram with all components)
- **Section 2** — What each tool does (role of every component explained simply)
- **Section 3** — End-to-end flow for one interview session (step by step)
- **Section 4** — Technology stack summary table
- **Section 5** — Latency budget (what the candidate actually experiences, in milliseconds)
- **Section 6** — Data flow and storage (where transcripts, scores, and memories are saved)
- **Section 7** — What still needs research (open questions before production)

---

**Framework:** Pipecat · **Language:** Python · **Market:** India (Hindi / Marathi / English)

---

## 1. The Big Picture

```
┌─────────────────────────────────────────────────────────────────────┐
│                        RECRUITER SIDE                               │
│  ┌──────────────────────┐       ┌──────────────────────────────┐   │
│  │  React Native App     │──────▶│   Backend API (FastAPI)      │   │
│  │  (Recruiter Dashboard)│  REST │   - Trigger interviews        │   │
│  │  - Create job roles   │       │   - Store results            │   │
│  │  - View transcripts   │◀──────│   - Manage candidates        │   │
│  │  - See scores         │  JSON │                              │   │
│  └──────────────────────┘       └────────────┬─────────────────┘   │
└───────────────────────────────────────────────│─────────────────────┘
                                                │ Trigger Twilio call
                                                ▼
┌─────────────────────────────────────────────────────────────────────┐
│                        PHONE CALL LAYER                             │
│  ┌───────────────────────────────────────────────────────────────┐  │
│  │                          TWILIO                               │  │
│  │  - Outbound call to candidate's phone number                  │  │
│  │  - Streams audio in/out via WebSocket (Media Streams)         │  │
│  │  - Audio format: 8kHz, µ-law (G.711)                         │  │
│  └──────────────────────┬──────────────────────────────────────-─┘  │
└─────────────────────────│───────────────────────────────────────────┘
                          │ WebSocket (audio stream)
                          ▼
┌─────────────────────────────────────────────────────────────────────┐
│                     AI PIPELINE (Pipecat)                           │
│                                                                     │
│  Candidate's Voice                                                  │
│       │                                                             │
│       ▼                                                             │
│  [TwilioFrameSerializer] ──── Decodes µ-law audio → PCM            │
│       │                                                             │
│       ▼                                                             │
│  [VAD: Silero] ──── Detects when candidate starts/stops talking    │
│       │                                                             │
│       ▼                                                             │
│  [Sarvam Saaras v3] ──── STT: Speech → Text (Hindi/Marathi/English)│
│       │                                                             │
│       ▼                                                             │
│  [GPT-4o 128K] ──── LLM: Thinks, generates next question/response  │
│       │                                                             │
│       ▼                                                             │
│  [Sarvam Bulbul v2] ─── TTS: Text → Voice (natural Hindi/English)  │
│       │                                                             │
│       ▼                                                             │
│  [TwilioFrameSerializer] ──── Encodes back → µ-law                 │
│       │                                                             │
│       ▼                                                             │
│  Audio plays to candidate on phone                                  │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 2. What Each Tool Does (Role of Every Component)

### 🔵 React Native App — Recruiter Dashboard
**Role:** Management interface for recruiters and HR team  
**NOT used by candidates**

What it does:
- Create job roles with description and interview rubric
- Upload candidate profiles (name, phone, resume)
- **Trigger the AI interview** with one button (calls the backend API)
- View real-time interview status (in-progress / completed)
- Read the full interview transcript after the call
- See the AI-generated candidate evaluation and score
- Schedule interviews, manage multi-round workflows

**Connects to:** Backend FastAPI via REST API

---

### 🔵 FastAPI Backend — The Orchestrator
**Role:** The brain that coordinates everything

What it does:
- Receives "start interview" request from React Native app
- Calls Twilio API to **initiate the outbound phone call** to the candidate
- When Twilio connects the call, a WebSocket opens
- **Spins up the Pipecat pipeline** for that specific interview session
- Injects context: job description + candidate resume + interview rubric into LLM
- Stores the transcript to the database after each turn
- Saves final evaluation/score to database when interview ends
- Sends results back to React Native app

**Connects to:** React Native (REST), Twilio (REST + WebSocket), Pipecat pipeline, Database

---

### 🔵 Twilio — Phone Call Infrastructure
**Role:** PSTN telephone layer — calls the candidate and streams audio

What it does:
1. Backend calls Twilio API: `"Call +91-9812345678"`
2. Twilio dials the candidate's mobile/landline
3. Candidate picks up → Twilio establishes the call
4. Twilio **streams candidate's audio** via WebSocket to your Pipecat server
5. Your Pipecat server streams **AI's audio response** back via WebSocket
6. Twilio plays that audio to the candidate through the phone call
7. When interview ends, Twilio disconnects the call

**Audio format:** 8kHz, µ-law (G.711) — standard phone audio codec  
**Cost:** $0.013–0.025/min for India outbound calls  
**No WebRTC involved** — this is pure PSTN telephony

---

### 🔵 TwilioFrameSerializer (Pipecat component)
**Role:** Audio format translator between Twilio and Pipecat

What it does:
- Twilio sends audio as **8kHz µ-law** (phone codec) 
- Pipecat works with **16kHz PCM** (standard AI audio)
- `TwilioFrameSerializer` converts: Twilio µ-law → PCM for Pipecat input
- And back: PCM → µ-law for Twilio output
- Also handles call hangup via Twilio REST API

This component is invisible to you — Pipecat handles it internally.

---

### 🔵 VAD: Silero — Voice Activity Detection
**Role:** Knows WHEN the candidate is speaking vs. silent

What it does:
- Continuously monitors the audio stream
- Detects: "Candidate started talking" → triggers STT
- Detects: "Candidate stopped talking" → sends audio to Sarvam for transcription
- **Prevents the AI from interrupting** mid-sentence
- Handles pauses, fillers ("umm", "aah"), and natural turn-taking
- Critical for natural conversation flow on a phone call

Without VAD: AI might start responding while candidate is still talking — very bad UX.

**Model:** Silero VAD (open-source, runs locally in Pipecat)  
**Latency added:** ~20–50ms

---

### 🔵 Sarvam Saaras v3 — STT (Speech to Text)
**Role:** Converts the candidate's voice into text

What it does:
- Receives audio from VAD (when candidate finishes speaking)
- Transcribes speech to text
- **Key feature: `codemix` mode** — handles Hindi, Marathi, and English all mixed in one sentence
  - Example: `"मेरे पास 5 years का experience है in Python and React"`
  - Transcribes correctly even when candidate switches languages mid-sentence
- Sends the text transcript to GPT-4o

**Why Sarvam and not Deepgram/Whisper:**
- Only model purpose-built for Indian code-switching
- ~19% Word Error Rate on IndicVoices (beats Gemini, GPT-4o)
- 952ms latency from India (vs Deepgram's 1,830ms — US only)
- ₹15/10K characters for Saaras v3

**Output:** Clean text like `"I have 5 years of experience in Python"`

---

### 🔵 GPT-4o (128K context) — LLM, The "Brain"
**Role:** The actual interviewer intelligence — reads transcripts, thinks, asks questions

What it does:
- Receives the full conversation history (every question asked + every answer given)
- Has the system prompt injected at start:
  - Job description
  - Candidate's resume
  - Interview rubric / question bank
  - Instructions: "You are an AI interviewer for [Company]. Ask about..."
- Generates the **next interview question or follow-up**
- Maintains the interview flow: introduction → technical questions → behavioral → closing
- Evaluates each answer against the rubric internally
- At the end of interview: generates structured evaluation JSON

**Why GPT-4o:**
- 128K context window fits an entire 45-minute interview (~20–35K tokens)
- Strong reasoning for follow-up questions
- Reliable structured output (for evaluation JSON)
- Fast (200–500ms first-token latency)

**Context management:**
- Single 45-min session: full context fits easily, no extra tools needed
- Multi-round interviews (Round 1 → Round 2): Mem0 retrieves prior round memories

---

### 🔵 Sarvam Bulbul v2 — TTS (Text to Speech)
**Role:** Converts GPT-4o's text response into a natural voice to play to the candidate

What it does:
- Receives GPT-4o's text: `"Can you tell me about a challenging project you worked on?"`
- Converts to natural-sounding speech in Hindi or English
- Streams audio back through Twilio to the candidate's phone
- Can switch language based on context (Hindi response → Hindi voice, English → English voice)

**Why Sarvam Bulbul:**
- Most natural-sounding Indian accent
- Supports Hindi + English natively
- Fast: ~300–500ms first audio byte
- ₹15/10,000 characters (v2), affordable for screening

**Output:** Audio file (WAV/PCM) → handed to TwilioFrameSerializer → sent to candidate

---

### 🔵 Pipecat Framework — The Pipeline Orchestrator
**Role:** The glue that connects all components into a flowing pipeline

What it does:
- Defines the pipeline: `Transport → VAD → STT → LLM → TTS → Transport`
- Manages the frame-based data flow between components
- Handles async processing (audio input while processing previous turn)
- Manages the LLM context (`messages[]` array — the full conversation history)
- Built-in transcript collection (saves every STT output)
- Error handling and recovery
- VAD integration for natural turn detection

```python
# What the Pipecat pipeline looks like (simplified)
pipeline = Pipeline([
    transport.input(),          # Audio from Twilio
    silero_vad,                 # Detect speech
    sarvam_stt,                 # Speech → Text
    llm_context_aggregator,     # Build conversation history
    gpt4o_llm,                  # Generate response
    sarvam_tts,                 # Text → Speech
    transport.output(),         # Audio back to Twilio
])
```

---

### 🔵 Memory Layer — Interview Context Management

#### Within a single interview (built-in, no extra tools):
- GPT-4o's 128K context window holds the entire conversation
- Pipecat manages the `messages[]` array automatically
- No extra tools needed for a 45-minute session

#### Across multiple interview rounds (Mem0):
- After Round 1 ends: Mem0 extracts and stores key facts
  - "Candidate mentioned 5 years Python experience"
  - "Mentioned working at Infosys from 2019–2023"
  - "Struggled with system design question"
- Before Round 2 starts: Mem0 retrieves those facts
- GPT-4o in Round 2 is aware of Round 1 context

---

## 3. End-to-End Flow: One Interview Session

```
1. RECRUITER ACTION
   Recruiter opens React Native app
   → Selects candidate "Ravi Sharma" for Python Developer role
   → Taps "Start Interview" button

2. API TRIGGER
   RN App → POST /api/interviews/start
   Body: { candidate_id, job_role, round_number }
   
3. TWILIO CALL INITIATION
   FastAPI Backend:
   → Fetches candidate phone: +91-9812345678
   → Calls Twilio API: "Dial +91-9812345678"
   → Sets webhook URL for audio stream

4. CANDIDATE ANSWERS
   Twilio dials Ravi's phone
   Ravi picks up: "Hello?"

5. PIPECAT PIPELINE STARTS
   Twilio WebSocket opens → Pipecat pipeline activates
   
   Backend injects system prompt to GPT-4o:
   "You are an AI interviewer at TechCorp. 
    Job: Python Developer. 
    Candidate: Ravi Sharma (5 yrs exp, Infosys).
    Rubric: Focus on Django, REST APIs, system design.
    Start with a greeting in Hindi."

6. AI INTRODUCES ITSELF
   GPT-4o → "नमस्ते Ravi जी! मैं TechCorp का AI interviewer हूँ।
               आज हम Python Developer role के बारे में बात करेंगे।
               क्या आप अपने बारे में कुछ बताएंगे?"
   Bulbul TTS → Hindi audio
   Twilio → Ravi hears the greeting on his phone

7. CANDIDATE SPEAKS
   Ravi: "नमस्ते! मेरा नाम Ravi है, मैंने 5 साल Infosys में
          Python और Django पर काम किया है..."
   
   Silero VAD → detects speech end
   Sarvam Saaras v3 (codemix) → transcribes correctly
   Text: "My name is Ravi, I worked for 5 years at Infosys 
          on Python and Django..."
   
   GPT-4o receives transcript → adds to conversation history
   GPT-4o thinks → generates follow-up question

8. INTERVIEW CONTINUES (45 minutes)
   This loop repeats:
   Candidate speaks → VAD → STT → LLM → TTS → Candidate hears
   
   GPT-4o manages interview flow:
   - Intro questions (5 min)
   - Technical depth (25 min)
   - Behavioral/situational (10 min)
   - Candidate's questions (5 min)

9. INTERVIEW ENDS
   GPT-4o: "Thank you Ravi! That completes the interview.
             We will get back to you within 3 business days."
   Bulbul TTS → audio played
   
   Pipeline shutdown:
   → GPT-4o generates structured evaluation JSON:
     {
       "overall_score": 7.5/10,
       "technical_score": 8/10,
       "communication_score": 7/10,
       "strengths": ["Django expertise", "API design"],
       "concerns": ["System design depth"],
       "recommendation": "Proceed to Round 2"
     }
   → Full transcript saved to database
   → Mem0 stores key facts for Round 2 (if applicable)
   → Twilio disconnects call

10. RECRUITER SEES RESULTS
    React Native app → GET /api/interviews/{id}/results
    Recruiter sees: transcript, score, recommendation
```

---

## 4. Technology Stack Summary

| Layer | Tool | Why This Choice |
|---|---|---|
| **Recruiter App** | React Native | Cross-platform iOS + Android for HR team |
| **Backend** | FastAPI (Python) | Async, compatible with Pipecat's Python ecosystem |
| **Phone Call** | Twilio | PSTN outbound calling, India coverage, WebSocket streaming |
| **AI Pipeline** | Pipecat | Open-source orchestrator for STT+LLM+TTS with Twilio integration |
| **Audio Format Bridge** | TwilioFrameSerializer | Converts 8kHz µ-law ↔ 16kHz PCM automatically |
| **Turn Detection** | Silero VAD | Knows when candidate starts/stops talking |
| **Speech to Text** | Sarvam Saaras v3 | Only model for Hindi+Marathi+English code-switching |
| **AI Brain** | GPT-4o (128K) | Conducts interview, evaluates answers, generates reports |
| **Text to Speech** | Sarvam Bulbul v2 | Natural Indian accent, Hindi + English |
| **Single-session Memory** | GPT-4o 128K context | 45-min interview fits entirely in context window |
| **Cross-round Memory** | Mem0 | Remembers Round 1 facts for Round 2/3 interview |
| **Database** | PostgreSQL | Stores candidates, transcripts, evaluations |

---

## 5. Latency Budget: What the Candidate Experiences

```
Candidate finishes speaking
          │
    ~20ms │ VAD detects speech end
          │
   ~952ms │ Sarvam Saaras v3 transcribes (from India)
          │
   ~300ms │ GPT-4o generates response (first token)
          │
   ~400ms │ Sarvam Bulbul v2 generates audio
          │
   ~150ms │ Twilio delivers audio to phone
          │
          ▼
Total: ~1.8–2.3 seconds (candidate hears AI response)
```

This is within the acceptable range for a phone interview (humans have 1–3s pauses naturally).

---

## 6. Data Flow & Storage

```
Interview Data:
├── Transcript (text) → PostgreSQL → Recruiter dashboard
├── Audio recording (optional) → S3 bucket → Compliance archive
├── Evaluation JSON → PostgreSQL → Recruiter scores view
└── Cross-round memories → Mem0 → Used in next round

Candidate Data:
├── Phone number → Used by Twilio (not stored in AI pipeline)
├── Resume text → Injected into GPT-4o system prompt
└── Job match score → Generated by GPT-4o, stored in PostgreSQL
```

---

## 7. What Still Needs Research

| Topic | Status | Priority |
|---|---|---|
| Speech pipeline architecture (S2S vs STT+LLM+TTS) | ✅ Done | — |
| Multilingual support (Hindi/Marathi/English) | ✅ Done | — |
| Sarvam AI deep dive (accuracy, pricing, Pipecat) | ✅ Done | — |
| Memory & context management (45-min sessions) | ✅ Done | — |
| Transport layer (Twilio + React Native) | ✅ Done | — |
| **LLM selection** (GPT-4o vs Claude vs Gemini) | 🔲 Pending | High |
| **VAD & turn-taking** (phone call specifics) | 🔲 Pending | High |
| **Prompt engineering** (interview system prompt design) | 🔲 Pending | High |
| **Scoring & evaluation** (rubric design, structured output) | 🔲 Pending | Medium |
| **Deployment** (Docker, cloud, scaling) | 🔲 Pending | Medium |
