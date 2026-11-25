# RelayDesk — Enterprise Customer Support Voice Platform

A production-oriented real-time voice support platform that handles inbound customer calls, verifies CRM accounts, retrieves answers from a company knowledge base, creates support tickets, schedules specialist appointments, escalates complex cases to human agents with full conversation context, dispatches signed webhooks to external systems, and produces automated call quality evaluations and analytics.

## Capabilities

- **Real-Time Bidirectional Voice**: Streams 16 kHz 16-bit PCM microphone audio over WebSockets (`/ws/support`) and schedules 24 kHz PCM audio playback gaplessly through Web Audio API buffers.
- **Interruption / Barge-In Handling**: Detects caller speech mid-turn (or manual interrupt), immediately stops queued `AudioBufferSourceNode` instances, resets the playback cursor, and records interrupted turns in the transcript and call metrics.
- **Hybrid Company Knowledge Retrieval**: Queries company documentation via vector embeddings combined with lexical term scoring (`search_knowledge_base`), returning ranked excerpts and citations that ground spoken responses.
- **Real-Time Multi-Tool Execution with Persistent State**: Executes five tools during active calls while maintaining a unified `CallStateSnapshot` (verified account, retrieved articles, created ticket IDs, booked appointments, and handoff reference) across every turn:
  - `lookup_account`
  - `search_knowledge_base`
  - `create_support_ticket`
  - `schedule_appointment`
  - `transfer_to_human_agent`
- **Human Handoff with Full Context**: Packages caller identity, verified CRM account snapshot, consulted knowledge articles, created ticket/appointment IDs, escalation urgency, and full timestamped transcript into a `HumanHandoffPacket` routed to the appropriate department queue.
- **External System Integrations**: Dispatches HMAC-SHA256 signed JSON webhooks (`X-RelayDesk-Event`, `X-RelayDesk-Signature`) on ticket creation, appointment scheduling, and human escalation, with SSRF URL validation and a built-in loopback verification sink (`/api/integrations/webhook-sink`).
- **Monitoring, Analytics & Quality Evaluation**: Tracks time-to-first-audio (TTFA), per-tool execution latency in milliseconds, barge-in counts, escalation rates, webhook delivery logs, and structured post-call QA evaluations scored across five contact-center dimensions.

## Architecture

```
Browser (Web Audio API + Operator UI)
  ├── 16kHz PCM Mic Stream ──► WebSocket (/ws/support) ──► Live Voice Bridge ──► Tool Router
  ├── 24kHz PCM Playback   ◄── WebSocket (/ws/support) ◄── Audio + State Diffs       │
  └── REST Console (/api/*) ──► Express Server ◄─────────────────────────────────────┘
                                     │
                 ┌───────────────────┼────────────────────┬────────────────────┐
                 ▼                   ▼                    ▼                    ▼
        Hybrid KB Retrieval    CRM & Ticket Store   Signed Webhooks      QA Audit Engine
       (Embeddings + Lexical)  (Persistent JSON)   (CRM/Helpdesk/Ops)   (Structured JSON)
```

## Project Deliverables

- `enterprise_support_voice_platform.ts`: Standalone single-file TypeScript server and embedded web console containing the WebSocket voice pipeline, tool router, hybrid retrieval engine, persistent store, signed webhook dispatcher, and QA evaluator.
- `README.md`: Architecture, setup, API contracts, and operational guide.

## Setup & Execution

### Prerequisites

- Node.js 20+
- `GEMINI_API_KEY` environment variable

### Install Dependencies

```bash
npm install express ws @google/genai dotenv
npm install -D tsx typescript @types/express @types/ws @types/node
```

### Environment Variables

Create a `.env` file:

```env
GEMINI_API_KEY="your_api_key_here"
PORT=3001
CRM_WEBHOOK_URL=""
TICKET_WEBHOOK_URL=""
ESCALATION_WEBHOOK_URL=""
WEBHOOK_SIGNING_SECRET="replace-with-random-secret"
```

When external webhook URLs are left empty, signed events deliver to the built-in verification endpoint (`http://127.0.0.1:3001/api/integrations/webhook-sink`) so integration payloads and delivery logs can be verified immediately.

### Run the Single-File Platform

```bash
npx tsx enterprise_support_voice_platform.ts
```

Open `http://localhost:3001` in your browser, allow microphone access, select or enter a caller (for example `Maya Lin` / `ACC-4091`), and click **Start Support Call**.

## API & WebSocket Reference

### REST Endpoints

| Method | Path | Description |
| :--- | :--- | :--- |
| `GET` | `/api/bootstrap` | Return accounts, knowledge base, tickets, appointments, handoffs, calls, and webhook logs |
| `POST` | `/api/calls` | Initialize a new inbound customer support call |
| `POST` | `/api/calls/:id/evaluate` | Run structured 5-dimension QA evaluation on a recorded call |
| `POST` | `/api/webhooks/test` | Send a signed test webhook event and log delivery latency |
| `POST` | `/api/integrations/webhook-sink` | Built-in local webhook receiver for delivery verification |

### WebSocket Protocol (`/ws/support`)

**Client to Server**
- `{ "type": "start", "callId": "CALL-..." }`
- `{ "type": "audio", "audio": "<base64-pcm16-16khz>" }`
- `{ "type": "text", "text": "<optional-keyboard-input>" }`
- `{ "type": "manual_interrupt" }`
- `{ "type": "end", "durationSeconds": 240 }`

**Server to Client**
- `{ "type": "ready" }`
- `{ "type": "audio", "audio": "<base64-pcm16-24khz>" }`
- `{ "type": "transcript_committed", "entry": { ... } }`
- `{ "type": "tool_executed", "toolLog": { ... }, "state": { ... } }`
- `{ "type": "interrupted" }`
- `{ "type": "error", "message": "..." }`

## Quality Evaluation Output Schema

Each call evaluated via `POST /api/calls/:id/evaluate` persists a structured audit object:

```json
{
  "generatedAt": "2026-11-25T09:15:00.000Z",
  "overallScore": 91,
  "sentiment": "Positive",
  "firstCallResolution": true,
  "summary": "Verified Northstar Logistics account, grounded rate-limit policy from KB-101, filed High-priority ticket TKT-4821, and confirmed a Solutions Engineering callback.",
  "dimensions": [
    { "name": "Resolution & Accuracy", "score": 94, "rationale": "..." },
    { "name": "Knowledge Base Grounding", "score": 92, "rationale": "..." },
    { "name": "Tool Execution Precision", "score": 95, "rationale": "..." },
    { "name": "Empathy & Tone", "score": 88, "rationale": "..." },
    { "name": "Escalation & Handoff Hygiene", "score": 86, "rationale": "..." }
  ],
  "policyAdherenceNotes": ["..."],
  "coachingRecommendations": ["..."]
}
```
