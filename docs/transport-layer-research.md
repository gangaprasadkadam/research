# Transport Layer Research: AI Voice Interview Agent

## 📋 What This Document Is For
This document answers: **how does audio actually travel between the candidate and the Pipecat AI agent?** It covers every Pipecat transport option — from WebRTC (browser/app) to PSTN phone calls (Twilio, Plivo, Exotel) — and picks the right one for our use case: outbound phone calls to Indian candidates triggered by a REST API.

> Read this to understand the full transport landscape. The final decision is in `final-recommendations.md § Transport`.

## 📑 What It Covers
- **Section 1** — What the transport layer is and why it matters
- **Section 2** — SmallWebRTC (Pipecat's built-in peer-to-peer WebRTC)
- **Section 3** — Daily.co Transport (managed WebRTC rooms)
- **Section 4** — LiveKit Transport (self-hostable WebRTC rooms)
- **Section 5** — Twilio (PSTN phone calls via WebSocket — `FastAPIWebsocketTransport` + `TwilioFrameSerializer`)
- **Section 6** — Alternative Indian telephony providers (Plivo, Exotel)
- **Section 7** — Transport architecture by use case (decision matrix)
- **Section 8** — Full comparison table (all transports side by side)
- **Section 9** — Cost comparison for 1,000 interviews/month
- **Section 10** — Recommended architecture for this project
- **Section 11** — What about Twilio + LiveKit combined? (why it's NOT recommended)
- **Section 12** — React Native (recruiter mobile app) specifics
- **Section 13** — Summary and final recommendation

---

**Research Date:** 2025 · **Framework:** Pipecat · **Market:** India

---

## 1. Overview: What Is the Transport Layer?

The **transport layer** is the plumbing that carries audio between:
- The **candidate** (browser, mobile app, or phone)
- The **Pipecat AI agent** (your backend server)

```
Candidate (browser/phone) ←── Transport Layer ───→ Pipecat Agent
                                                        │
                                                   STT → LLM → TTS
```

It handles: WebRTC signaling, audio encoding/decoding, NAT traversal, packet loss recovery, and connection lifecycle.

### Transport Options in Pipecat (2025)

| Transport | Type | Best For |
|---|---|---|
| `SmallWebRTCTransport` | P2P WebRTC | Development / prototyping |
| `DailyTransport` | Managed SFU (Daily.co) | Production browser calls |
| `LiveKitTransport` | Managed/Self-hosted SFU | Production, phone + browser, India self-host |
| `FastAPIWebsocketTransport` + serializer | WebSocket telephony | PSTN phone calls via Twilio/Plivo/Exotel |

---

## 2. SmallWebRTC (Pipecat's Built-in P2P)

### What It Is
A lightweight, dependency-free P2P WebRTC transport built into Pipecat — no media server required.

### Architecture
```
Browser ←── Direct P2P WebRTC ───→ Pipecat Agent Server
```

### Setup
```python
from pipecat.transports.network.smallwebrtc import SmallWebRTCTransport

transport = SmallWebRTCTransport(webrtc_connection=connection)

# That's it — no API keys, no infrastructure
```

### Pros & Cons

| Pros | Cons |
|---|---|
| ✅ Zero infrastructure needed | ❌ Not production-grade (NAT traversal issues) |
| ✅ Zero cost | ❌ Fails behind corporate firewalls |
| ✅ Lowest latency (direct P2P) — 30–50ms India | ❌ 1 connection at a time only |
| ✅ 30-minute setup | ❌ No recording, no observability |

### Verdict: When to Use
- **Local development** and demos only
- **Internal testing** behind a known network
- **NOT for production** (NAT traversal failures make it unreliable for external candidates)

---

## 3. Daily.co Transport (DailyTransport)

### Background
Daily.co is the **parent company that created Pipecat**. This means `DailyTransport` has the deepest, most battle-tested integration of all Pipecat transports.

### Architecture
```
Candidate (Browser/Mobile) ──→ Daily.co Global SFU ──→ Pipecat Agent
                                  (75+ global PoPs)
```

### Pipecat Integration

**Server-side (Python):**
```python
from pipecat.transports.services.daily import DailyTransport, DailyParams

transport = DailyTransport(
    room_url=DAILY_ROOM_URL,
    token=DAILY_BOT_TOKEN,
    bot_name="AI Interviewer",
    params=DailyParams(
        audio_in_enabled=True,
        audio_out_enabled=True,
        camera_in_enabled=False,   # voice-only interview
        transcription_enabled=False,  # using Sarvam STT instead
        vad_enabled=True,
        vad_analyzer=SileroVADAnalyzer()
    )
)
```

**Client-side (Browser, TypeScript):**
```typescript
import { DailyTransport } from "@pipecat-ai/daily-transport";
import { PipecatClient } from "@pipecat-ai/client-js";

const client = new PipecatClient({
  transport: new DailyTransport(),
  enableMic: true,
  enableCam: false,
});

// Connect to AI interviewer
await client.connect({ endpoint: "/api/start-interview" });
```

**Mobile (iOS - Swift):**
```swift
// CocoaPods
pod 'PipecatClientIOSDaily'
```

**Mobile (Android - Kotlin):**
```kotlin
implementation 'ai.pipecat:daily-transport'
```

### Pricing

| Tier | Rate | Notes |
|---|---|---|
| **Free** | 10,000 participant-min/month | ~500 sessions (user + agent = 2 participants) |
| **Paid** | $0.0045/participant-min | Per each participant in the room |
| **10-min session** | $0.09 | 10 min × 2 participants × $0.0045 |
| **45-min session** | $0.405 | 45 min × 2 participants × $0.0045 |

**Monthly estimates:**
| Sessions/month | Cost |
|---|---|
| 100 (45-min each) | ~$40.50 |
| 500 | ~$202 |
| 1,000 | ~$405 |

### Infrastructure

| Feature | Detail |
|---|---|
| **Type** | Fully managed SFU (no self-hosting possible) |
| **PoPs** | 75+ globally |
| **Asia-Pacific** | ✅ PoPs exist |
| **India-specific DC** | ⚠️ Not documented (contact sales to confirm) |
| **Encryption** | DTLS/SRTP by default |
| **NAT traversal** | ✅ Built-in STUN/TURN |
| **Recording** | ✅ Supported (disabled by default) |

### SDKs

| Platform | SDK | Status |
|---|---|---|
| Web (JS/TS) | `@pipecat-ai/daily-transport` | ✅ Official |
| React | `@pipecat-ai/daily-transport` | ✅ Official |
| React Native | `@daily-co/react-native-daily-js` | ✅ Official |
| iOS (Swift) | `PipecatClientIOSDaily` | ✅ Official |
| Android (Kotlin) | `ai.pipecat:daily-transport` | ✅ Official |

### Connection Types

| Type | Support |
|---|---|
| Browser WebRTC | ✅ Primary use case |
| iOS/Android App | ✅ Official SDKs |
| SIP/PSTN phone calls | ❌ Not supported at Daily layer |

> ⚠️ **No PSTN support**: If candidates need to call via phone number, add Twilio/Plivo separately (different transport altogether).

### Latency Profile

| Component | Estimate |
|---|---|
| Candidate (India) → Daily AP PoP | ~100–150ms |
| Daily AP PoP → Agent Server (same region) | ~10–50ms |
| Round-trip (transport only) | ~200–350ms |
| + STT (Sarvam Saaras v3) | +952ms first response |
| + LLM (GPT-4o) | +200–500ms |
| + TTS (Sarvam Bulbul) | +300–500ms |
| **Total TTFA (Time to First Audio)** | **~1.5–2.2s** |

### Verdict
✅ **Best choice if you want zero infrastructure ops, official mobile SDKs, and a managed global service.**  
⚠️ Caution: No India-specific DC confirmed. Slightly more expensive than LiveKit at scale. No PSTN support.

---

## 4. LiveKit Transport (LiveKitTransport)

### Background
LiveKit is an open-source, Apache 2.0-licensed SFU platform. It can be run as:
1. **LiveKit Cloud** — managed service (no India DC currently)
2. **Self-hosted** — deploy on AWS Mumbai for India-local, low-latency, data-sovereign operation

### Architecture
```
Candidate (Browser/Mobile/Phone) ──→ LiveKit SFU ──→ Pipecat Agent
                                   (cloud or self-hosted)
        ↑ also supports ↑
        SIP/PSTN via LiveKit SIP service
```

### Pipecat Integration

**Server-side (Python):**
```python
from pipecat.transports.services.livekit import LiveKitTransport, LiveKitParams

transport = LiveKitTransport(
    url=os.getenv("LIVEKIT_URL"),
    token=os.getenv("LIVEKIT_TOKEN"),
    room_name=os.getenv("LIVEKIT_ROOM_NAME"),
    params=LiveKitParams(
        audio_in_enabled=True,
        audio_out_enabled=True,
    )
)
```

**Token generation:**
```python
from livekit.api import AccessToken, VideoGrants

def generate_token(room_name: str, participant_name: str) -> str:
    token = AccessToken(
        api_key=LIVEKIT_API_KEY,
        api_secret=LIVEKIT_API_SECRET
    ).with_grants(
        VideoGrants(room_join=True, room=room_name)
    ).with_identity(participant_name)
    return token.to_jwt()
```

**Client-side (JavaScript):**
```typescript
import { LivekitTransport } from "@pipecat-ai/livekit-transport";
import { PipecatClient } from "@pipecat-ai/client-js";

const client = new PipecatClient({
  transport: new LivekitTransport(),
  enableMic: true,
  enableCam: false,
});

await client.connect({ endpoint: "/api/start-interview" });
```

### Pricing

#### LiveKit Cloud

| Tier | Rate | Notes |
|---|---|---|
| Free | 5,000 min/month | Dev/testing |
| Cloud Standard | ~$0.003–0.005/min | Per session-minute |
| **45-min session** | ~$0.135–0.225 | Slightly cheaper than Daily |

**No India region on LiveKit Cloud** — nearest is Singapore/Tokyo.

#### Self-Hosted on AWS Mumbai (ap-south-1)

| Component | Monthly Cost |
|---|---|
| EC2 t3.large (2 vCPU, 8GB) | ~$45/month |
| RDS PostgreSQL (state) | ~$20/month |
| ElastiCache Redis | ~$15/month |
| Data transfer (audio egress) | ~$5–10/month |
| **Total (infrastructure)** | **~$85–90/month** |

> 💡 Break-even vs LiveKit Cloud at ~500 sessions/month.

### Infrastructure: Cloud vs Self-Hosted

| Factor | LiveKit Cloud | Self-Hosted AWS Mumbai |
|---|---|---|
| **India latency** | 150–200ms (Singapore) | **30–80ms** ✅ |
| **Data sovereignty** | Data leaves India | ✅ Stays in India |
| **Ops overhead** | Zero | Medium (Docker, Redis, TURN) |
| **Setup time** | 1 day | 3–5 days |
| **SIP/PSTN support** | ✅ Via livekit/sip service | ✅ Full control |
| **Recording** | ✅ Egress service | ✅ Egress service |
| **Scaling** | Auto | Manual (add nodes) |
| **GDPR / data compliance** | Check ToS | ✅ Full control |

### SIP / PSTN Support

LiveKit has a **dedicated SIP service** (`livekit/sip`) that bridges phone calls into LiveKit rooms:

```
Candidate calls +91-XXXXXX
        ↓
SIP trunk (Plivo/Twilio/VoBiz)
        ↓
LiveKit SIP Service (livekit/sip)
        ↓
LiveKit Room
        ↓
Pipecat AI Agent
```

**Setup:**
```bash
# Create SIP inbound trunk
livekit-cli create-sip-inbound-trunk \
  --name "India-Interview-Line" \
  --numbers "+911800XXXXXX"

# Create dispatch rule
livekit-cli create-sip-dispatch-rule \
  --type "individual-dispatch" \
  --room-prefix "interview-"
```

### Recording with LiveKit Egress

```python
from livekit.api import EgressClient

egress = EgressClient(LIVEKIT_URL, LIVEKIT_API_KEY, LIVEKIT_API_SECRET)

# Record full interview session
await egress.start_room_composite_egress(
    room_name=room_name,
    output=EncodedFileOutput(
        file_path=f"s3://interview-bucket/{candidate_id}-{session_id}.mp4"
    )
)
```

**Output formats:** MP4, WebM, HLS, RTMP, OGG (audio-only), real-time WebSocket stream

### SDKs

| Platform | SDK | Status |
|---|---|---|
| Web (JS/TS) | `@livekit/components-react` | ✅ Official |
| Python | `livekit` | ✅ Official (used by Pipecat agent) |
| iOS | `livekit-swift-sdk` | ✅ Official |
| Android | `livekit-android-sdk` | ✅ Official |
| React Native | Community wrapper | ⚠️ Community |

### Latency Profile (Self-Hosted AWS Mumbai)

| Component | Estimate |
|---|---|
| Candidate (India) → LiveKit SFU (Mumbai) | **30–80ms** |
| LiveKit SFU → Agent Server (same Mumbai) | ~5–10ms |
| Round-trip (transport only) | **70–170ms** |
| Total TTFA with AI pipeline | **~1.4–2.0s** |

> ~150ms faster than LiveKit Cloud (Singapore routing) for Indian candidates.

### Verdict
✅ **Best choice for India-focused deployment**: self-host on AWS Mumbai for lowest latency + data sovereignty + PSTN support.  
✅ **Best for phone call support** — full SIP/PSTN via livekit/sip.  
⚠️ Requires more DevOps than Daily.co.

---

## 5. Twilio (WebSocket Telephony — PSTN Only)

### Important Clarification
**Twilio is NOT a WebRTC transport in Pipecat.** It is a telephony provider that streams audio via WebSocket. The integration uses `TwilioFrameSerializer` — not a transport class.

### Architecture (PSTN Path)
```
Candidate's Phone
      ↓ (PSTN call to your Twilio number)
Twilio Voice
      ↓ (WebSocket media stream, µ-law 8kHz)
Your FastAPI Server
      ↓ (Pipecat pipeline)
TwilioFrameSerializer → STT → LLM → TTS → back to Twilio
```

### Pipecat Integration

```python
from pipecat.transports.network.fastapi_websocket import (
    FastAPIWebsocketTransport,
    FastAPIWebsocketParams
)
from pipecat.serializers.twilio import TwilioFrameSerializer

@app.websocket("/media-stream")
async def handle_media_stream(websocket: WebSocket):
    serializer = TwilioFrameSerializer(
        stream_sid=stream_sid,
        call_sid=call_sid,
        account_sid=TWILIO_ACCOUNT_SID,
        auth_token=TWILIO_AUTH_TOKEN,
        params=TwilioFrameSerializer.InputParams(auto_hang_up=True)
    )

    transport = FastAPIWebsocketTransport(
        websocket=websocket,
        params=FastAPIWebsocketParams(
            audio_out_enabled=True,
            add_wav_header=False,
            vad_enabled=True,
            serializer=serializer,
        )
    )
    # ... plug into Pipecat pipeline
```

**TwiML webhook:**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<Response>
  <Connect>
    <Stream url="wss://your-server.com/media-stream"/>
  </Connect>
</Response>
```

### India Pricing (2025)

| Call Type | Rate/Min | Notes |
|---|---|---|
| Inbound (India DID) | $0.013–0.025 | Varies by carrier |
| Outbound to India mobile | $0.0135–0.091 | Carrier-dependent |
| Outbound to India landline | $0.0135 | Lower rate |
| Browser/App (WebRTC) | $0.004 | WebRTC via Twilio SDK |
| India phone number | $2.00/month | DID rental |
| India toll-free (1800) | $5–10/month + call rates | TRAI approval required |

> ⚠️ **Twilio's India outbound rates are significantly more expensive than alternatives.** Consider Plivo or Exotel for high-volume India calling.

### India Compliance (TRAI Requirements)

| Requirement | Details |
|---|---|
| **Caller ID** | Must show real registered number |
| **Recording disclosure** | Mandatory voice prompt before/during call |
| **DNC compliance** | Monthly scrub against TRAI DNC registry |
| **AI disclosure (2024)** | Must announce "This is an AI interviewer" |
| **Toll-free setup** | TRAI application: 8–10 week process |
| **KYC verification** | Required for any India number |

### Latency (Phone Path)

| Component | Estimate |
|---|---|
| PSTN call (India) → Twilio Mumbai edge | ~100–150ms |
| Twilio → Your Server (WebSocket) | ~20–50ms |
| Phone round-trip baseline | ~200–350ms |
| + Full AI pipeline | +1,200–1,800ms |
| **Total TTFA (phone)** | **~1.7–2.2s** |

### Verdict
✅ Use for PSTN phone call support.  
⚠️ Expensive for India. Consider Plivo/Exotel for cost savings.  
❌ NOT a WebRTC transport — don't use for browser-based interviews.

---

## 6. Alternative Indian Telephony Providers

Better than Twilio for India-specific PSTN calls:

### Plivo

```python
from pipecat.serializers.plivo import PlivoFrameSerializer

serializer = PlivoFrameSerializer(
    stream_id=stream_id,
    call_id=call_id,
    auth_id=PLIVO_AUTH_ID,
    auth_token=PLIVO_AUTH_TOKEN,
)
```

| Feature | Detail |
|---|---|
| **India inbound rate** | $0.004–0.012/min |
| **India outbound rate** | $0.005–0.015/min |
| **vs Twilio** | ~60–70% cheaper |
| **India DCs** | ✅ Bangalore DC |
| **TRAI compliance** | ✅ Handled natively |
| **Toll-free (India)** | ✅ Available |
| **Setup time** | 3–5 days |

### Exotel (India-native)

```python
from pipecat.serializers.exotel import ExotelFrameSerializer

serializer = ExotelFrameSerializer(
    stream_sid=stream_sid,
    call_sid=call_sid
)
```

| Feature | Detail |
|---|---|
| **India inbound rate** | $0.005–0.010/min |
| **India outbound rate** | $0.006–0.012/min |
| **India DCs** | ✅ Mumbai + Bangalore |
| **TRAI compliance** | ✅ Built-in simplified |
| **Toll-free (India)** | ✅ Fast-track available |
| **Setup time** | 2–3 days |
| **Key advantage** | Built for India — fastest compliance process |

### Provider Comparison for India PSTN

| Provider | Rate/Min | India DC | TRAI Process | Best For |
|---|---|---|---|---|
| **Twilio** | $0.013–0.025 | Mumbai (in1) | Manual, 2–4 weeks | Global deployments |
| **Plivo** | $0.004–0.012 | Bangalore | Simplified, 3–5 days | Cost-sensitive |
| **Exotel** | $0.005–0.010 | Mumbai + Blr | Native, 2–3 days | India-only deployments |
| **VoBiz** | $0.003–0.008 | India | Local, fast | Budget option |

---

## 7. Transport Architecture by Use Case

### Use Case A: Browser-Only Interviews (Recommended for MVP)

```
Candidate opens browser link → Daily.co SFU → Pipecat Agent
                                               (AWS ap-south-1)
```

**Stack:**
```python
transport = DailyTransport(
    room_url=daily_room_url,
    token=bot_token,
    params=DailyParams(audio_in_enabled=True, audio_out_enabled=True, vad_enabled=True)
)
```

| Factor | Value |
|---|---|
| Setup time | 1 day |
| Cost | $0.0045/participant-min (~$0.405/45-min session) |
| Infrastructure | None (fully managed) |
| Mobile support | ✅ iOS + Android official SDKs |
| PSTN | ❌ Not available |

---

### Use Case B: Browser + Phone (Full Coverage)

```
Browser candidate → LiveKit SFU (AWS Mumbai) → Pipecat Agent
Phone candidate → SIP trunk (Plivo) → LiveKit SIP → LiveKit SFU → Pipecat Agent
```

**Stack:**
```python
# Route based on connection type
if browser_connection:
    transport = LiveKitTransport(url=LIVEKIT_URL, token=token, room_name=room)
elif phone_connection:
    # Plivo webhook → FastAPI → PlivoFrameSerializer
    serializer = PlivoFrameSerializer(stream_id=stream_id, call_id=call_id, ...)
```

| Factor | Value |
|---|---|
| Setup time | 1–2 weeks |
| Infrastructure cost | ~$85–90/month (AWS Mumbai) |
| Telephony cost | $0.004–0.012/min (Plivo India) |
| India latency | **30–80ms** (browser) |
| Data sovereignty | ✅ Stays in India |
| PSTN | ✅ Full support |

---

### Use Case C: India Compliance-First

```
Self-hosted LiveKit on AWS Mumbai (ap-south-1)
+ Exotel for toll-free 1800/1860 number
+ All data stored in India (S3 ap-south-1)
```

**For regulated sectors (banking, healthcare, government):**
- Exotel handles TRAI compliance natively
- AWS Mumbai for data residency
- Full audit trail and recording stored in India

---

## 8. Full Comparison Table

| Feature | SmallWebRTC | Daily.co | LiveKit Cloud | LiveKit Self-hosted | Twilio (PSTN) |
|---|---|---|---|---|---|
| **Best for** | Dev/testing | Production browser | Mid-scale browser | Scale + India + Phone | Phone calls only |
| **Setup time** | 30 min | 1 day | 1 day | 3–5 days | 2–4 weeks (TRAI) |
| **Cost/45-min session** | Free | $0.405 | ~$0.135–0.225 | Infra cost only | $0.45–1.13 |
| **India latency** | 30–50ms | 100–150ms | 150–200ms | **30–80ms** ✅ | 100–150ms |
| **India DC** | Local only | Unconfirmed | ❌ (Singapore) | ✅ AWS Mumbai | ✅ Mumbai edge |
| **Mobile SDKs** | ❌ | ✅ iOS + Android | ✅ iOS + Android | ✅ iOS + Android | N/A (phone) |
| **PSTN/Phone** | ❌ | ❌ | ✅ via SIP | ✅ Full SIP | ✅ Native |
| **Recording** | ❌ | ✅ | ✅ Egress | ✅ Egress | Via Twilio API |
| **Self-hostable** | N/A (Pipecat only) | ❌ | ❌ | ✅ | ❌ |
| **Data sovereignty** | N/A | ❌ | ❌ | ✅ | Partial |
| **Pipecat integration** | ✅ Native | ✅ Native (parent co) | ✅ Native | ✅ Native | ✅ via Serializer |
| **Open source** | N/A | Pipecat yes, Daily no | LiveKit yes | ✅ Apache 2.0 | ❌ |

---

## 9. Cost Comparison: 1,000 45-Minute Interviews/Month

| Stack | Transport Cost | Notes |
|---|---|---|
| **SmallWebRTC** | $0 | Not production-viable |
| **Daily.co** | ~$405/month | 1000 × 45min × 2 participants × $0.0045 |
| **LiveKit Cloud** | ~$135–225/month | Better rate, no India DC |
| **LiveKit Self-hosted (Mumbai)** | ~$90/month | Fixed infra, scales to ~500 concurrent |
| **Twilio PSTN (India)** | ~$585–2,295/month | High inbound rates |
| **Plivo PSTN (India)** | ~$180–540/month | Much cheaper for India |

**Add-on costs for all:**
- Sarvam Saaras v3 STT: ~₹9,000–15,000/month (45 min × 1000 sessions)
- GPT-4o LLM: ~$100–200/month
- Sarvam Bulbul TTS: ~₹7,500–12,000/month

---

## 10. Recommended Architecture for AI Interview Agent

### Phase 1 (MVP): Browser-Only, Quick Launch

```
Candidate → Browser Link → Daily.co DailyTransport → Pipecat
                                                         ↓
                                           Sarvam STT → GPT-4o → Sarvam TTS
```

**Justification:**
- Fastest to market (1–2 days setup)
- Official mobile SDKs work out of the box
- No infrastructure management
- 10,000 free minutes for testing
- ~$405/month at 1,000 interviews

---

### Phase 2 (Production): Browser + Phone, India-Optimized

```
Browser → LiveKit SFU ──→ Pipecat Agent (AWS Mumbai ap-south-1)
Phone → Plivo/Exotel ──→ LiveKit SIP ──→ LiveKit SFU ──→ Pipecat Agent
```

**Justification:**
- Self-hosted AWS Mumbai: lowest latency (30–80ms) for Indian candidates
- Data stays in India — compliance-friendly
- Plivo adds PSTN option at India-competitive rates
- Egress service for automatic interview recording
- ~$90/month infra + $4–12/min Plivo for phone sessions

---

### Architecture Decision Tree

```
Do you need phone (PSTN) call support?
├── No → Use DailyTransport (browser-only)
└── Yes → Use LiveKit + Plivo/Exotel (SIP bridge)
              │
              └── Do you need India data residency?
                  ├── No → LiveKit Cloud ($0.003-0.005/min)
                  └── Yes → Self-host LiveKit on AWS Mumbai (~$90/month)
```

---

## 11. What About "Twilio with LiveKit"?

This is a common search term. Here's the full explanation:

**What it is:** Using Twilio as a SIP trunk provider to route phone calls INTO a LiveKit room.

```
Phone call → Twilio number → Twilio SIP trunk → LiveKit SIP service → LiveKit room → AI agent
```

**When to use:**
- You already have existing Twilio infrastructure/numbers
- You need both phone AND browser participants in the same "room" (e.g., HR manager on browser watching candidate on phone)

**When NOT to use (most cases):**
- Simple 1:1 browser interviews — Daily.co or LiveKit alone is simpler
- Simple PSTN calls — Twilio alone is simpler
- India-focused — Plivo/Exotel directly connected to LiveKit SIP is cheaper

**Operational reality:** Adding Twilio as a SIP trunk to LiveKit adds:
- ~50–100ms extra latency (SIP handoff)
- Additional cost (Twilio rates + LiveKit rates)
- Two systems to manage

> 💡 **Verdict**: Only use Twilio + LiveKit combined if you specifically need Twilio's features (existing numbers, SMS, advanced call routing) alongside LiveKit's room model. For new projects, prefer Plivo/Exotel → LiveKit SIP directly.

---

## 12. React Native (Mobile App) Specifics

Since the interview app is a **React Native application**, transport choice matters differently than browser or native.

### Official React Native Packages

Both transports are maintained in: `github.com/pipecat-ai/pipecat-client-react-native-transports`

#### Option A: Daily.co for React Native

```bash
npm install @pipecat-ai/client-react-native @pipecat-ai/react-native-daily-transport
```

```typescript
import { RNDailyTransport } from "@pipecat-ai/react-native-daily-transport";
import { PipecatClient } from "@pipecat-ai/client-react-native";

const client = new PipecatClient({
  transport: new RNDailyTransport(),
  enableMic: true,
  enableCam: false,
});

// Start interview session
await client.startBotAndConnect({
  endpoint: `${BACKEND_URL}/start-interview`,
});
```

**Requirements:**
- iOS: >= 15
- Android: minSdkVersion >= 24 (Android 7.0+)

#### Option B: LiveKit for React Native

```bash
npm install @pipecat-ai/client-react-native @pipecat-ai/react-native-livekit-transport
# Also requires native WebRTC modules:
npm install @livekit/react-native @livekit/react-native-webrtc
```

```typescript
import { LiveKitTransport } from "@pipecat-ai/react-native-livekit-transport";
import { PipecatClient } from "@pipecat-ai/client-react-native";

const client = new PipecatClient({
  transport: new LiveKitTransport({ url: LIVEKIT_URL }),
  enableMic: true,
  enableCam: false,
});

await client.startBotAndConnect({
  endpoint: `${BACKEND_URL}/start-interview`,
});
```

**Extra setup for LiveKit RN:**
- Add Kotlin/Swift native modules (more complex Gradle/Podfile config)
- Requires `npx pod-install` for iOS
- Android: Requires ProGuard rules

---

### React Native Comparison

| Factor | Daily (`RNDailyTransport`) | LiveKit (`LiveKitTransport`) |
|---|---|---|
| **RN setup complexity** | ✅ Simple (2 packages) | ⚠️ Complex (native modules) |
| **Pipecat RN docs** | ✅ Well-documented | ⚠️ Less RN-specific docs |
| **iOS min version** | iOS 15 | iOS 13 |
| **Android min version** | SDK 24 (7.0) | SDK 21 (5.0) |
| **Expo support** | ⚠️ Needs bare workflow | ⚠️ Needs bare workflow |
| **India latency** | 100–150ms (no India DC) | **30–80ms** (self-hosted Mumbai) ✅ |
| **PSTN phone support** | ❌ | ✅ via LiveKit SIP |
| **Backend ops** | Zero (managed) | Medium (self-hosted) |
| **Audio quality** | ✅ Excellent | ✅ Excellent |

> ⚠️ **Note on Expo**: Neither Daily nor LiveKit supports Expo Managed Workflow. You need **Expo Bare Workflow** or pure React Native CLI. Both require native modules.

---

### React Native Recommendation

**For the interview app:**

**Start with Daily.co (`RNDailyTransport`):**
- Easiest setup (2 npm packages, no native config beyond Expo bare)
- API pattern is identical to web version (easy full-stack parity)
- Production-ready immediately
- No backend infrastructure to manage

**Migrate to LiveKit when you need:**
- Indian candidates with <100ms latency (self-host AWS Mumbai)
- PSTN phone call support alongside the app
- Data residency in India

**If starting from scratch today, the practical path:**
```
Phase 1 (Week 1–2):
  npm install @pipecat-ai/react-native-daily-transport
  → Working interview app immediately

Phase 2 (Month 2+):
  Migrate to LiveKit self-hosted on AWS Mumbai
  → India-optimized, PSTN-capable, data-sovereign
```

---

## 13. Summary: Final Recommendations

| Scenario | Recommended Transport | Cost/Interview |
|---|---|---|
| **MVP / Quick launch** | Daily.co `DailyTransport` | ~$0.40 |
| **Browser-only production** | Daily.co or LiveKit Cloud | $0.14–0.40 |
| **India-optimized, data-sovereign** | LiveKit self-hosted (AWS Mumbai) | ~$0.09 |
| **PSTN phone calls (India)** | Plivo/Exotel serializer | $0.05–0.15 |
| **PSTN phone calls (global)** | Twilio serializer | $0.14–0.50 |
| **Full coverage (browser + phone)** | LiveKit self-hosted + Plivo | $0.09 + phone |
| **Regulated/compliance-first** | LiveKit Mumbai + Exotel | ~$0.09 + $0.08 |

### The Short Answer

For an **AI interview agent targeting Indian candidates**:

1. **Start with**: `DailyTransport` (browser link, zero ops, works in hours)
2. **Scale to**: `LiveKitTransport` on AWS Mumbai (India latency, data residency, phone support)
3. **Add phone**: Plivo or Exotel via `PlivoFrameSerializer` / `ExotelFrameSerializer`
4. **Skip**: Twilio for India (expensive), LiveKit Cloud (no India DC), SmallWebRTC (not production-ready)
