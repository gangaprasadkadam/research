# AI Voice Agent Frameworks — Deep Research Report

> **Purpose:** Senior-level evaluation of open-source AI voice agent frameworks for production use.  
> **Frameworks Covered:** Pipecat · LiveKit Agents · TEN Framework · Dograh  
> **Focus:** Open-source, minimal cost, production-grade quality  
> **Date:** April 2026

---

## Table of Contents

1. [Understanding AI Voice Agents](#1-understanding-ai-voice-agents)
2. [Pipecat](#2-pipecat)
3. [LiveKit Agents](#3-livekit-agents)
4. [TEN Framework](#4-ten-framework)
5. [Dograh](#5-dograh)
6. [Head-to-Head Comparison](#6-head-to-head-comparison)
7. [Use Case Matrix](#7-use-case-matrix)
8. [Recommendation & Final Verdict](#8-recommendation--final-verdict)
9. [Architecture Decision Guide](#9-architecture-decision-guide)
10. [Resources & References](#10-resources--references)

---

## 1. Understanding AI Voice Agents

### What is an AI Voice Agent?

An AI voice agent is a system that can **listen**, **understand**, **reason**, and **respond** in real-time using natural spoken language. Unlike simple chatbots, voice agents handle the full audio-to-audio loop autonomously.

### The Classic Pipeline Architecture

```
User Speaks
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  VAD (Voice Activity Detection)  — detects when user speaks     │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  STT (Speech-to-Text)  — converts audio → text                  │
│  e.g., Deepgram, Whisper, AssemblyAI                            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  LLM (Large Language Model)  — reasoning + response generation  │
│  e.g., OpenAI GPT-4o, Anthropic Claude, Llama, Gemini           │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  TTS (Text-to-Speech)  — converts text → spoken audio           │
│  e.g., ElevenLabs, Deepgram Aura, Azure Neural TTS              │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Agent Responds (Audio streamed back to user)
```

### Two Core Architectures

| Architecture | How It Works | Latency | Tradeoff |
|---|---|---|---|
| **Cascaded (STT → LLM → TTS)** | Each stage processes independently | ~800ms–2s | More control, auditable |
| **Speech-to-Speech (S2S)** | End-to-end multimodal model | ~300–500ms | Less control, newer tech |

### Key Transport Protocols

- **WebRTC** — Browser/mobile real-time audio with echo cancellation, jitter buffering, NAT traversal
- **SIP/PSTN** — Traditional telephony (phone calls via Twilio, Vonage, Plivo)
- **WebSockets** — Flexible streaming, simpler but without media-layer guarantees

### Latency Budget (Industry Standard 2025)

| Component | Target Latency |
|---|---|
| VAD | < 50ms |
| STT (first token) | < 200ms |
| LLM (first token) | < 500ms |
| TTS (first audio) | < 200ms |
| **Total round-trip (P50)** | **< 800ms** |
| **Total round-trip (P95)** | **< 1200ms** |

---

## 2. Pipecat

### Overview

**Pipecat** is an open-source Python framework for building real-time voice and multimodal AI agents. Created by **Daily.co**, it provides a composable, "frame-based" pipeline architecture where audio, text, and visual data flow through a chain of processors — similar to Unix pipes for AI.

| Property | Value |
|---|---|
| **Creator** | Daily.co |
| **License** | BSD-2-Clause (permissive, commercial-friendly) |
| **Language** | Python |
| **GitHub Stars** | 11,000+ |
| **GitHub Repo** | [pipecat-ai/pipecat](https://github.com/pipecat-ai/pipecat) |
| **First Release** | 2024 |
| **Architecture** | Frame-based pipeline |

### Core Architecture

```
┌────────────────────────────────────────────────────────────────┐
│                       PIPECAT PIPELINE                         │
│                                                                │
│  Transport   →  [Frame]  →  STT  →  LLM  →  TTS  →  Transport │
│  (WebRTC /                                                      │
│   WebSocket /        Custom Processors can be inserted anywhere │
│   SIP)               between any stage                         │
└────────────────────────────────────────────────────────────────┘
```

- **Frames** are the atomic unit: audio frames, text frames, image frames, system frames
- **Processors** transform or act on frames
- Pipelines are fully composable — you insert custom logic anywhere
- Supports both cascaded and speech-to-speech architectures

### Key Features

#### ✅ Real-Time Multimodal Processing
- Handles voice, audio, video, images in one pipeline
- Sub-800ms typical latency; 200–300ms possible with S2S models
- Built-in VAD (Voice Activity Detection) and turn detection

#### ✅ Extensive Provider Integrations
- **STT:** Deepgram, AssemblyAI, Whisper, Azure, Google
- **LLM:** OpenAI (GPT-4o), Anthropic (Claude), Google Gemini, AWS Bedrock, Llama, NVIDIA NIM
- **TTS:** ElevenLabs, Deepgram Aura, Azure Neural TTS, Google TTS, CartesIA
- **Transport:** Daily (WebRTC), Twilio, WebSockets, local

#### ✅ Pipecat Flows
- State-machine-style conversation management
- Handle multi-step interactions (e.g., appointment booking, structured interviews)
- Task completion patterns built in

#### ✅ Cross-Platform Client SDKs
- JavaScript, React, React Native, Swift (iOS), Kotlin (Android), C++, ESP32

#### ✅ Developer Tooling
- **Whisker** — real-time pipeline debugger
- OpenTelemetry tracing and distributed monitoring
- Terminal dashboards and production observability

#### ✅ Enterprise & Cloud Partnerships
- Deep collaboration with AWS (Amazon Bedrock + Nova Sonic)
- NVIDIA partnership for GPU-accelerated inference

### Supported Use Cases

- 📞 Phone/telephony AI agents (customer support, IVR)
- 🛒 E-commerce voice assistants
- 🏥 Healthcare conversational agents
- 🎮 Interactive game NPCs
- 🤖 Multi-agent orchestration pipelines
- 📊 Voice analytics and data extraction
- 🌍 Multilingual agents with real-time translation

### Cost Profile

| Cost Area | Detail |
|---|---|
| Framework license | **Free** (BSD-2-Clause) |
| Self-hosting | Pay your own server costs only |
| Daily.co cloud transport | Optional; paid per minute if used |
| AI provider APIs | Deepgram ~$0.0043/min, OpenAI GPT-4o ~$0.0015/min, ElevenLabs variable |
| Infrastructure | Standard cloud VMs; ~$50–200/mo for moderate workloads |

### Pros & Cons

#### ✅ Pros
- **Maximum flexibility** — insert any logic anywhere in the pipeline
- **Vendor-agnostic** — swap STT/LLM/TTS without re-architecting
- **Battle-tested** — strong production adoption, enterprise case studies
- **Excellent documentation** and active community
- **Permissive BSD-2 license** — zero restrictions for commercial use
- **Multimodal** — voice + video + images in one framework
- **Advanced observability** built in (OTel, distributed tracing)
- Strong partnership with AWS and NVIDIA for cutting-edge integrations

#### ❌ Cons
- **Python only** on the server side — no native Node.js/Go support
- **More boilerplate** than higher-level frameworks
- Transport layer requires a separate service (Daily, Twilio, or custom)
- Larger learning curve for newcomers
- Complex pipelines can be harder to debug without tooling

---

## 3. LiveKit Agents

### Overview

**LiveKit Agents** is a production-grade, open-source framework for real-time voice, video, and physical AI agents. Built on top of LiveKit's WebRTC media infrastructure — which already powers **ChatGPT's Advanced Voice mode** — it treats AI agents as first-class participants in real-time rooms/sessions.

| Property | Value |
|---|---|
| **Creator** | LiveKit Inc. |
| **License** | Apache 2.0 |
| **Language** | Python, Node.js |
| **GitHub Stars** | ~9,900 |
| **GitHub Repo** | [livekit/agents](https://github.com/livekit/agents) |
| **First Release** | 2023 |
| **Architecture** | Session/Room-based WebRTC participants |

### Core Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        LIVEKIT ECOSYSTEM                           │
│                                                                    │
│  ┌──────────┐     ┌──────────────────┐     ┌───────────────────┐  │
│  │  Client  │────▶│  LiveKit Server  │◀───▶│    SIP Server     │  │
│  │ (Browser │     │  (Media Routing) │     │ (PSTN/Telephony)  │  │
│  │ /Mobile) │     └────────┬─────────┘     └───────────────────┘  │
│  └──────────┘              │                                        │
│                            ▼                                        │
│                   ┌────────────────┐                                │
│                   │  Agent Worker  │                                │
│                   │  (Your Code)   │                                │
│                   │  STT → LLM     │                                │
│                   │  → TTS         │                                │
│                   └────────────────┘                                │
└────────────────────────────────────────────────────────────────────┘
```

- Agents run as server-side **worker processes** that join rooms as participants
- Automatic session/room lifecycle management
- Built-in dispatcher routes calls to available workers

### Key Features

#### ✅ Complete Voice AI Pipeline (Auto-wired)
- Agent framework auto-connects STT → VAD/Turn Detection → LLM → TTS
- Transformer-based turn detection (knows when user finishes speaking)
- Barge-in support (user can interrupt agent mid-speech)

#### ✅ WebRTC-Native Infrastructure
- Sub-300ms typical round-trip latency
- End-to-end encryption (E2EE) support
- Multi-participant rooms (group conversations)
- Built-in recording and egress

#### ✅ Full SIP/Telephony Support
- Self-hostable SIP server bridges PSTN calls into LiveKit rooms
- Works with Twilio, Exotel, Vonage and other SIP providers
- Inbound and outbound call flows

#### ✅ Multi-Language SDKs
- Python and Node.js server (agent) SDKs
- Client SDKs: JavaScript, React, iOS (Swift), Android (Kotlin), Flutter, Unity

#### ✅ Model Context Protocol (MCP) Support
- Agents can use MCP tools to interact with external systems
- Function calling and RAG integration built in

#### ✅ Enterprise-Grade Compliance
- SOC 2, HIPAA, GDPR compliant (Cloud)
- Self-host for full data sovereignty

#### ✅ Scalability at Scale
- Powers ChatGPT Advanced Voice (millions of concurrent users)
- Horizontal scaling via worker pools and load balancing

### Supported Use Cases

- 📞 AI phone agents and virtual receptionists
- 🏥 Medical triage and telehealth assistants
- 🎓 Language learning and tutoring bots
- 💼 Customer support automation and IVR replacement
- 🎮 Real-time multiplayer game voice NPCs
- 📹 Video AI (visual + voice combined)
- 🤝 Multi-party conference AI facilitators

### Deployment Options

```
Option 1: LiveKit Cloud (Managed)
  - Zero ops overhead
  - Pay per minute: ~$0.01/min agent + $0.01/min telephony
  - Free tier for development

Option 2: Self-Hosted (Open Source)
  - Completely free to run
  - Requires: Redis, public IP, Docker/Kubernetes
  - SIP server fully open-source
  - Pay only for AI provider APIs + compute
```

### Cost Profile

| Cost Area | Detail |
|---|---|
| Framework license | **Free** (Apache 2.0) |
| Self-hosted media server | **Free** (open source) |
| LiveKit Cloud (optional) | $0.01/min agent session; $0.01/min telephony |
| AI provider APIs | OpenAI, Deepgram, ElevenLabs — your usage |
| Self-host infrastructure | ~$50–300/mo depending on scale |

### Pros & Cons

#### ✅ Pros
- **Fastest path to production** — convention over configuration
- **Built-in WebRTC** — no separate media server needed
- **Both Python and Node.js** — broader team adoption
- **Powers ChatGPT Voice** — proven at massive scale
- **SIP built-in** — easy telephony integration out of the box
- **Excellent documentation** with quickstarts for many use cases
- **No-code Agent Builder** for rapid prototyping
- **MCP support** for agentic tool use
- **E2EE + HIPAA/SOC2** compliance

#### ❌ Cons
- **Transport lock-in** — must use LiveKit as the WebRTC backbone
- Less flexible for non-room-based architectures
- Convention-based design limits highly customized pipelines
- Breaking changes as the platform evolves rapidly
- Self-hosting requires managing Redis + media server + SIP server
- Cloud gets expensive at very high volume

---

## 4. TEN Framework

### Overview

**TEN (Transformative Extensions Network)** is an open-source framework for building real-time, multimodal AI agents using a **graph-based node architecture**. Created by **Agora** (a major WebRTC/RTC infrastructure company), TEN is designed for production-grade AI agents with a visual drag-and-drop workflow designer (TMAN).

| Property | Value |
|---|---|
| **Creator** | Agora Inc. |
| **License** | Apache 2.0 (some components have additional restrictions) |
| **Language** | C++, Python, Go (Node.js in development) |
| **GitHub Stars** | 10,400+ |
| **GitHub Repo** | [TEN-framework/ten-framework](https://github.com/TEN-framework/ten-framework) |
| **First Release** | 2024 |
| **Architecture** | Graph-based nodes/extensions |

### Core Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                      TEN AGENT GRAPH                               │
│                                                                    │
│  ┌────────┐    ┌────────┐    ┌────────┐    ┌────────┐             │
│  │ Input  │───▶│  STT   │───▶│  LLM   │───▶│  TTS   │             │
│  │ Node   │    │  Node  │    │  Node  │    │  Node  │             │
│  └────────┘    └────────┘    └────────┘    └────────┘             │
│       │                           │              │                 │
│       ▼                           ▼              ▼                 │
│  ┌────────┐                 ┌──────────┐   ┌────────┐             │
│  │ Agora  │                 │  Tools / │   │ Output │             │
│  │  RTC   │                 │  RAG Node│   │  Node  │             │
│  └────────┘                 └──────────┘   └────────┘             │
│                                                                    │
│        Graph configured via JSON or visual TMAN designer           │
└────────────────────────────────────────────────────────────────────┘
```

- **Extensions** are reusable nodes (STT, LLM, TTS, custom tools)
- **Graphs** define how extensions connect via typed JSON messages
- **TMAN Designer** is a drag-and-drop visual tool for building graphs
- Agora SD-RTN is the primary RTC backbone

### Key Features

#### ✅ Visual Workflow Builder (TMAN Designer)
- Drag-and-drop interface to wire extensions
- Non-developers can prototype agents
- JSON graph config exportable for production deployment

#### ✅ True Multimodal
- Voice, text, images, video all as first-class citizens
- Emotional avatar support (3D digital humans)
- Real-time visual AI processing

#### ✅ Edge-Cloud Hybrid
- Deploy on cloud, local server, or edge devices (ESP32, Raspberry Pi)
- Designed for privacy-sensitive or offline deployments
- Minimal latency for edge-deployed agents

#### ✅ Broad AI Provider Support
- **LLM:** OpenAI, Llama, DeepSeek, Gemini, Qwen
- **STT:** Google, Azure, Deepgram
- **TTS:** ElevenLabs, Azure, Google, Volcengine
- **RTC:** Agora SD-RTN (primary), with extension points for others

#### ✅ Low Latency
- Sub-1 second round-trip target
- Real-time interruption support (user can cut off agent mid-speech)
- Global Agora SD-RTN for worldwide low-latency delivery

#### ✅ Multi-Language Extension Development
- Write custom extensions in C++, Python, or Go
- Node.js support in active development

### Supported Use Cases

- 🧑‍💻 3D AI avatars and digital humans
- 📖 Interactive storytelling with voice + visuals
- 📞 Real-time customer service call centers
- 🗣️ Language learning with personality-rich tutors
- 🎮 Advanced gaming NPC voice AI
- 🤖 IoT and robotics voice control
- 📊 Multi-modal business intelligence agents

### Cost Profile

| Cost Area | Detail |
|---|---|
| Framework license | **Free** (Apache 2.0) |
| Agora RTC usage | **PAID** — priced per peak concurrent user / minute |
| Agora free tier | 10,000 audio minutes/month free |
| Beyond free tier | ~$1.99 per 1,000 minutes (varies by region) |
| AI provider APIs | Standard OpenAI/Deepgram/ElevenLabs costs |
| Custom RTC (non-Agora) | Possible but requires significant engineering |

> ⚠️ **Important Cost Note:** TEN's default transport is Agora's commercial RTC network. While the framework code is free, production deployments at scale will incur ongoing Agora usage costs. This is the primary cost concern for minimal-cost deployments.

### Pros & Cons

#### ✅ Pros
- **Visual designer** lowers barrier to entry
- **Strong multimodal** capabilities (best for avatar/visual AI)
- **Backed by Agora** — enterprise-grade RTC with global SD-RTN
- **Edge deployment** support (IoT, robotics, privacy use cases)
- **Graph architecture** makes complex flows more auditable
- Active community with 10,400+ GitHub stars

#### ❌ Cons
- **Agora RTC dependency** — production cost scales with usage
- **Vendor coupling** — switching away from Agora requires effort
- **Licensing complexity** — Apache 2.0 with some additional restrictions; review carefully for commercial use
- **Smaller Python/Node.js ecosystem** compared to Pipecat/LiveKit
- Less mature community than Pipecat or LiveKit
- Documentation quality uneven (improving)
- Node.js support still in development

---

## 5. Dograh

### Overview

**Dograh** is a newer, open-source AI voice agent **platform** (not just a framework) designed as a no-code/low-code alternative to proprietary platforms like Vapi and Retell. Built on top of Pipecat, it adds a full platform layer: visual builder, telephony management, observability, and testing tools — all in a self-hostable package.

| Property | Value |
|---|---|
| **Creator** | YC Alumni / Exit Founders |
| **License** | BSD-2-Clause |
| **Language** | Python (based on Pipecat) |
| **GitHub Stars** | ~300 (early stage) |
| **GitHub Repo** | [dograh-hq/dograh](https://github.com/dograh-hq/dograh) |
| **First Release** | 2024 (new/emerging) |
| **Architecture** | Platform layer on top of Pipecat |

### Core Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                        DOGRAH PLATFORM                             │
│                                                                    │
│  ┌─────────────────────────────────────────┐                       │
│  │         Visual No-Code Builder           │                      │
│  │  (Drag & drop conversation workflows)    │                      │
│  └──────────────────┬──────────────────────┘                       │
│                     │                                               │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                       │
│  │     Dograh Platform Layer               │                       │
│  │  ┌──────────┐  ┌──────────┐  ┌───────┐ │                       │
│  │  │Telephony │  │Analytics │  │ RAG   │ │                       │
│  │  │ (Twilio/ │  │(Langfuse)│  │ / KB  │ │                       │
│  │  │ Vonage)  │  └──────────┘  └───────┘ │                       │
│  │  └──────────┘                           │                       │
│  └──────────────────┬──────────────────────┘                       │
│                     │                                               │
│                     ▼                                               │
│  ┌─────────────────────────────────────────┐                       │
│  │         Pipecat Core Engine              │                       │
│  │    (STT → LLM → TTS pipelines)          │                       │
│  └─────────────────────────────────────────┘                       │
└────────────────────────────────────────────────────────────────────┘
```

### Key Features

#### ✅ No-Code Visual Builder
- Drag-and-drop conversational workflow designer
- Real-time preview and testing within the browser
- Non-technical users can build and iterate agents

#### ✅ Built-in Telephony
- Connectors for Twilio, Vonage, Vobiz, Cloudonix
- Inbound + outbound call handling
- Voicemail detection and campaign call management

#### ✅ 30+ Language Support
- Multilingual by default
- Accent-aware STT configurations
- Global audience ready

#### ✅ Observability via Langfuse
- Full prompt management, evaluation, and metrics
- Real-time analytics dashboard
- Call performance monitoring

#### ✅ AI-to-AI Testing
- Automated stress testing — one AI calls another to validate agent
- Pre-production QA workflows built in
- Regression testing for conversation flows

#### ✅ Privacy-First Architecture
- 100% self-hostable
- No data leaves your infrastructure without your configuration
- BYOK (Bring Your Own Key) for all AI providers

### Supported Use Cases

- 📞 Customer support phone bots (inbound + outbound)
- 📣 Outbound telemarketing and lead generation
- 🏢 Enterprise voice workflow automation
- 🔒 Privacy-sensitive regulated industry agents
- 🚀 Rapid MVP voice product prototyping
- 🌐 Global multilingual call center replacement

### Cost Profile

| Cost Area | Detail |
|---|---|
| Framework license | **Free** (BSD-2-Clause) |
| Self-hosted | Pay your own server costs only |
| Telephony providers | Twilio ~$0.0085/min, Vonage ~$0.004/min |
| AI provider APIs | Standard Deepgram/OpenAI/ElevenLabs costs |
| No platform fees | Unlike Vapi ($0.05/min) or Retell ($0.07/min) |

### Pros & Cons

#### ✅ Pros
- **Lowest barrier to entry** — no-code builder out of the box
- **Platform, not just a framework** — batteries included (analytics, telephony, testing)
- **No platform SaaS fees** — direct alternative to Vapi/Retell
- **Privacy-first** — all data stays on your infrastructure
- **Built on proven Pipecat** engine
- **AI-to-AI testing** is a unique differentiator

#### ❌ Cons
- **Very early stage** — ~300 GitHub stars, small community
- **Limited documentation** compared to Pipecat or LiveKit
- **Inherits Pipecat complexity** — not a clean-room design
- **Unproven at scale** — limited production case studies
- **Smaller ecosystem** — fewer plugins and integrations
- **Active development instability** — APIs may change frequently
- Not suitable for high-risk, mission-critical deployments yet

---

## 6. Head-to-Head Comparison

### Quick Stats

| Attribute | Pipecat | LiveKit Agents | TEN Framework | Dograh |
|---|---|---|---|---|
| **License** | BSD-2-Clause | Apache 2.0 | Apache 2.0* | BSD-2-Clause |
| **GitHub Stars** | 11,000+ | ~9,900 | 10,400+ | ~300 |
| **Maturity** | Mature | Mature | Growing | Early-stage |
| **Primary Language** | Python | Python / Node.js | C++ / Python / Go | Python |
| **Architecture** | Pipeline (frames) | Room/session | Graph (nodes) | Platform (on Pipecat) |
| **WebRTC** | Via plugin | Built-in (native) | Via Agora | Via Pipecat |
| **Telephony/SIP** | 3rd party | Built-in SIP | Agora RTC | Built-in connectors |
| **Visual Builder** | ❌ | ❌ | ✅ TMAN | ✅ Drag-and-drop |
| **Self-Hostable** | ✅ | ✅ | ✅ | ✅ |
| **Transport Cost** | Optional (Daily) | Optional (LK Cloud) | Agora RTC (paid!) | None (BYO telephony) |
| **No-code support** | ❌ | Partial (Agent Builder) | ✅ | ✅ |
| **Edge Deployment** | Limited | Limited | ✅ | Limited |

*TEN has some additional license restrictions for certain components — review carefully.

### Feature Depth Comparison

| Feature | Pipecat | LiveKit | TEN | Dograh |
|---|---|---|---|---|
| Real-time voice | ✅ Excellent | ✅ Excellent | ✅ Excellent | ✅ Good |
| Voice latency | ~500-800ms | ~300-800ms | <1s | ~500-800ms |
| Interruption (barge-in) | ✅ | ✅ | ✅ | ✅ |
| Multi-provider flexibility | ✅✅ Highest | ✅ High | ✅ Medium-High | ✅ High |
| Multimodal (video/image) | ✅ | ✅ | ✅✅ Best | Limited |
| Telephony integration | ✅ (via plugins) | ✅✅ Best | ✅ (Agora) | ✅✅ Built-in |
| Group/multi-user calls | Limited | ✅✅ Best | Limited | Limited |
| Production scale proof | ✅✅ | ✅✅ (ChatGPT!) | ✅ | ❌ |
| Documentation quality | ✅✅ | ✅✅ | ✅ | ❌ Limited |
| Community size | ✅✅ Large | ✅✅ Large | ✅ Growing | ❌ Small |
| Observability / monitoring | ✅✅ (OTel) | ✅ | Limited | ✅ (Langfuse) |
| Built-in testing tools | Limited | Limited | Limited | ✅ (AI-to-AI) |
| State machine / flows | ✅ (Pipecat Flows) | Limited | Via graph | Via Pipecat |
| RAG / knowledge base | Via plugins | Via MCP/tools | Via nodes | ✅ Built-in |
| Avatar / 3D human | ❌ | Limited | ✅✅ Best | ❌ |
| Edge / IoT deployment | Limited | Limited | ✅✅ | Limited |

### Learning Curve

| Framework | Learning Curve | Best For |
|---|---|---|
| **Pipecat** | 🟡 Medium-High | Python developers wanting full control |
| **LiveKit** | 🟢 Low-Medium | Developers wanting fast production deployment |
| **TEN** | 🟡 Medium | Developers who prefer visual/graph tooling |
| **Dograh** | 🟢 Low | Non-technical teams or rapid MVP building |

### Community & Ecosystem

| Metric | Pipecat | LiveKit | TEN | Dograh |
|---|---|---|---|---|
| GitHub Stars | ⭐⭐⭐⭐⭐ 11k+ | ⭐⭐⭐⭐ 9.9k+ | ⭐⭐⭐⭐ 10.4k+ | ⭐ ~300 |
| Community Activity | 🔥 Very Active | 🔥 Very Active | 🔶 Active | 🔶 Growing |
| Third-party plugins | 🔥 Large | 🔥 Large | 🔶 Growing | 🔶 Limited |
| Production case studies | Many | Many (ChatGPT!) | Some | Very few |
| Enterprise adoption | ✅ Yes | ✅✅ Yes (major) | ✅ Yes | ❌ Early |

---

## 7. Use Case Matrix

Use this to map your requirements to the best framework:

| Your Use Case | Best Pick | Runner-up | Reason |
|---|---|---|---|
| **Phone call AI agent (customer support)** | LiveKit | Dograh | Best SIP/telephony + scale |
| **Web voice assistant (browser-based)** | LiveKit | Pipecat | Native WebRTC, fast prototyping |
| **Multimodal (voice + video)** | LiveKit | TEN | Best multi-party WebRTC |
| **Complex custom pipeline logic** | Pipecat | TEN | Maximum composability |
| **Rapid MVP / prototype** | Dograh | LiveKit | No-code + batteries included |
| **Privacy / on-premise / regulated** | Pipecat | Dograh | BSD license, fully self-hostable |
| **3D avatar / digital human** | TEN | — | Native avatar/emotional support |
| **Edge / IoT / embedded** | TEN | Pipecat | Native edge support |
| **Multi-language global** | Dograh | Pipecat | 30+ languages built-in |
| **Multi-party/group calls** | LiveKit | — | Room model designed for this |
| **Healthcare / HIPAA** | LiveKit | Pipecat | Compliance certifications |
| **Outbound calling campaigns** | Dograh | LiveKit | Campaign management built-in |
| **Maximum vendor flexibility** | Pipecat | LiveKit | Swap any provider at any time |
| **Minimal operational overhead** | LiveKit Cloud | Dograh | Managed or simple self-host |
| **Cost-optimized at scale** | Pipecat | LiveKit (self-host) | No transport platform lock-in |

---

## 8. Recommendation & Final Verdict

### 🏆 Overall Winner for Open-Source + Minimal Cost: **Pipecat + LiveKit (Hybrid)**

Before giving the single recommendation, here are the verdicts by scenario:

---

### Scenario A: You need a production voice agent fast, with telephony

> **Recommendation: LiveKit Agents (self-hosted)**

LiveKit is the safest choice for teams who:
- Need a working product quickly
- Require SIP/telephony integration
- Want proven infrastructure (it already runs ChatGPT Advanced Voice)
- Can manage a self-hosted media server

**Total cost (self-hosted):** ~$0/framework + ~$100–300/mo infrastructure + AI API costs

---

### Scenario B: You need maximum flexibility and control over the pipeline

> **Recommendation: Pipecat**

Pipecat is the best choice for teams who:
- Want to swap AI providers at any time
- Need custom analytics, compliance logging, or unique pipeline logic
- Are Python-first and comfortable building from components
- Want no dependency on any specific RTC vendor

**Total cost:** ~$0/framework + AI API costs + transport (can use free WebSockets or self-hosted WebRTC)

---

### Scenario C: Non-technical team building a voice product rapidly

> **Recommendation: Dograh (for MVP) → Migrate to Pipecat/LiveKit for production**

Dograh is the best starting point if:
- You want no-code drag-and-drop to validate your idea
- You don't want to write Python initially
- You're building an outbound calling product

⚠️ **Caution:** Dograh is early-stage with a small community. Plan to migrate to Pipecat (which Dograh is built on) once you scale.

---

### Scenario D: You need 3D avatars, edge deployment, or visual AI

> **Recommendation: TEN Framework**

TEN is the only realistic choice if you need:
- 3D digital human / avatar voice agents
- Edge/IoT deployment (ESP32, Raspberry Pi)
- Visual graph-based agent design for non-engineers

⚠️ **Important:** Budget for Agora RTC costs in production. The free tier (10k audio minutes/month) may limit scale.

---

### 🎯 Single Best Recommendation for Most Teams

If I had to choose **one framework** that offers the best balance of:
- ✅ Open source with permissive license
- ✅ Minimal cost (self-hostable, no vendor fees)
- ✅ Production-proven
- ✅ Strong community
- ✅ Maximum flexibility
- ✅ Excellent documentation

**→ The answer is: Pipecat**

Pipecat gives you the best foundation. You control every component — from the transport layer to the AI providers. It has the most permissive license (BSD-2), the largest provider ecosystem, strong AWS/NVIDIA partnerships, and proven production use. When you need WebRTC infrastructure, you can use LiveKit as the transport backend (the two integrate well together).

**If your primary use case involves real-time WebRTC rooms or you want the quickest path to production → LiveKit Agents.**

---

### Decision Summary Table

| Priority | Recommended Framework |
|---|---|
| Maximum flexibility + vendor freedom | **Pipecat** |
| Fastest time to production | **LiveKit Agents** |
| Telephony-first (phone calls) | **LiveKit Agents** |
| No-code / rapid prototyping | **Dograh** (MVP only) |
| Visual AI / avatars / edge | **TEN Framework** |
| Minimal ongoing cost | **Pipecat** (self-hosted) |
| Proven enterprise scale | **LiveKit Agents** |
| Data privacy / on-premise | **Pipecat** or **Dograh** |

---

## 9. Architecture Decision Guide

### Step 1: Define Your Transport Requirement

```
Do you need to handle phone calls (PSTN/SIP)?
    YES → LiveKit Agents or Dograh
    NO  → Continue to Step 2

Is your primary interface a browser/mobile app?
    YES → LiveKit Agents or Pipecat
    NO  → Continue to Step 2
```

### Step 2: Define Your Control Requirement

```
Do you need custom pipeline logic, analytics, or compliance hooks?
    YES, HEAVILY → Pipecat (maximum control)
    SOMEWHAT     → LiveKit Agents (tools/MCP) or TEN (graph nodes)
    NO           → LiveKit Agents (convention-based, fast)
```

### Step 3: Define Your Team Capability

```
Python developers who want to code → Pipecat or LiveKit
Node.js developers                 → LiveKit (Node.js SDK)
Non-technical / business team      → Dograh
Visual/graph-oriented developers   → TEN Framework
```

### Step 4: Define Your Scale Requirement

```
MVP / prototype         → Dograh or LiveKit Cloud
Small-medium production → Pipecat self-hosted or LiveKit self-hosted
Enterprise scale        → LiveKit Agents (ChatGPT-proven) or Pipecat
Edge / IoT              → TEN Framework
```

### Recommended Production Stack

For most teams building a voice agent product:

```
┌─────────────────────────────────────────────────────────┐
│              RECOMMENDED PRODUCTION STACK               │
│                                                         │
│  Framework:    Pipecat (core logic + pipeline)          │
│  Transport:    LiveKit (WebRTC) or Twilio (SIP/phone)   │
│  STT:          Deepgram Nova-2 (best accuracy/latency)  │
│  LLM:          OpenAI GPT-4o or Anthropic Claude        │
│  TTS:          ElevenLabs Flash or Deepgram Aura-2      │
│  Observability: OpenTelemetry + your APM of choice      │
│  Hosting:      AWS/GCP/Azure or self-managed Kubernetes │
└─────────────────────────────────────────────────────────┘
```

### Estimated Monthly Cost (Self-Hosted, 1000 calls/day)

| Component | Cost Estimate |
|---|---|
| Server (2x c5.xlarge or equivalent) | ~$150–300/mo |
| Deepgram STT (~10 min avg call) | ~$400–600/mo |
| OpenAI GPT-4o (~500 tokens/call) | ~$200–400/mo |
| ElevenLabs TTS | ~$100–300/mo |
| Telephony (Twilio SIP) | ~$85–150/mo |
| **Total** | **~$935–$1,750/mo** |

Compared to Vapi (~$0.05/min) at 1000 calls × 10 min = 600,000 min/month = **~$30,000/mo**. Self-hosted open source saves **95%+ costs at scale**.

---

## 10. Resources & References

### Official Documentation

| Framework | Docs | GitHub |
|---|---|---|
| Pipecat | https://docs.pipecat.ai | https://github.com/pipecat-ai/pipecat |
| LiveKit Agents | https://docs.livekit.io/agents | https://github.com/livekit/agents |
| TEN Framework | https://docs.agora.io/en/ten-framework | https://github.com/TEN-framework/ten-framework |
| Dograh | https://dograh.com | https://github.com/dograh-hq/dograh |

### Key Concepts to Explore Next

- **Voice Activity Detection (VAD):** Silero VAD, WebRTC VAD
- **STT Providers:** Deepgram, AssemblyAI, OpenAI Whisper, Azure
- **TTS Providers:** ElevenLabs, Deepgram Aura-2, Azure Neural TTS, Cartesia
- **LLM Providers:** OpenAI GPT-4o, Anthropic Claude 3.5, Groq (low latency), Llama (free/local)
- **Telephony:** Twilio, Vonage, Plivo, Exotel
- **Observability:** OpenTelemetry, Langfuse, Hamming AI

### Further Reading

- [LiveKit Voice Agent Architecture Guide](https://livekit.com/blog/voice-agent-architecture-stt-llm-tts-pipelines-explained)
- [Pipecat vs LiveKit: The Real Difference](https://www.cekura.ai/blogs/pipecat-vs-livekit-the-real-difference)
- [Choosing a Voice AI Agent Production Framework (WebRTC.ventures)](https://webrtc.ventures/2026/03/choosing-a-voice-ai-agent-production-framework/)
- [Best Voice Agent Stack Selection Framework (Hamming AI)](https://hamming.ai/resources/best-voice-agent-stack)
- [Building with Pipecat on AWS Bedrock](https://aws.amazon.com/blogs/machine-learning/building-intelligent-ai-voice-agents-with-pipecat-and-amazon-bedrock-part-1/)

---

*Report compiled: April 2026 | Research based on public documentation, GitHub data, and industry analysis*  
*All cost estimates are approximate and subject to change. Verify with provider pricing pages.*
