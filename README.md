# HSE Voice-First AI Agent

> Production voice agent for field HSE observation reporting — built on Azure AI Foundry, Python MCP tooling on Azure Functions, Flutter mobile, real-time WebSocket audio streaming.

**Built and owned end-to-end:** Azure AI Foundry agent configuration, MCP server (Python/Azure Functions), OAuth pass-through design, Flutter audio pipeline integration, enterprise BFF integration, CI/CD on Azure DevOps.

---

## The Problem

Field workers in hazardous environments need to report safety observations (unsafe conditions, near-misses, positive behaviours) in real time. The old process: stop work, navigate a 6-field mobile form, type a description, submit. Result: observations were deferred, forgotten, or skipped entirely under time pressure.

**Goal:** Let a worker raise their phone and speak naturally. The system handles the rest.

---

## Production Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│  Flutter Mobile App (Android / iOS / Web)                       │
│  - PCM16 audio capture at 24kHz                                 │
│  - WebSocket client (WSS)                                       │
│  - MSAL silent token refresh                                    │
└──────────────────────┬──────────────────────────────────────────┘
                       │  WSS + Bearer: <Azure AD token>
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  Voice Backend  (.NET 8, Azure App Service)                     │
│  - WebSocket session manager                                    │
│  - Server-side VAD (Voice Activity Detection)                   │
│  - Streams audio to Azure AI Foundry Realtime API               │
│  - Forwards agent tool_call events to MCP layer                 │
└──────────────────────┬──────────────────────────────────────────┘
                       │  HTTPS + Bearer: <user token>
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  Azure AI Foundry Agent                                         │
│  - GPT-4o Realtime model                                        │
│  - System prompt: HSE domain, observation types, severity       │
│  - Calls MCP tools when it needs to read/write enterprise data  │
└──────────────────────┬──────────────────────────────────────────┘
                       │  MCP over HTTPS
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  MCP Server  (Python, Azure Function App)                       │
│  - Tool: create_hse_observation (idempotent)                    │
│  - Tool: lookup_location                                        │
│  - Tool: get_active_work_permits                                │
│  - Validates Azure AD token on every invocation                 │
│  - Passes user token downstream — no service account            │
└──────────────────────┬──────────────────────────────────────────┘
                       │  HTTPS REST + Bearer: <user token>
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  HSE BFF  (.NET, Azure App Service)                             │
│  - Business rules, field validation, rate limiting              │
│  - Deduplication check on observation_id                        │
└──────────────────────┬──────────────────────────────────────────┘
                       │  REST
                       ▼
┌─────────────────────────────────────────────────────────────────┐
│  Enterprise System of Record (Quest)                            │
│  - Final persistence                                            │
│  - Audit trail with real user identity                          │
└─────────────────────────────────────────────────────────────────┘
```

---

## Key Engineering Decisions

### 1. OAuth Pass-Through (Not a Service Account)

Standard agentic integrations authenticate once with a service account — every API call looks like the same system user, breaking audit trails and bypassing per-user permissions.

This system propagates the signed-in worker's Azure AD token end-to-end:

```
Flutter (MSAL) → WSS Header → Voice Backend → MCP Server → HSE BFF → System of Record
```

The enterprise system sees the actual worker on every write. Required for regulatory compliance in Oil & Gas.

### 2. Idempotency Guardrail (From a Real Production Failure)

**What happened:** Network instability caused the agent to retry a tool call. The observation was written twice. The worker got a duplicate record notification. We caught it in UAT — but the root cause was structural.

**The fix:** Every `create_hse_observation` call requires a client-generated `observation_id`. The MCP server checks for an existing record before writing.

```python
@tool
def create_hse_observation(
    observation_type: str,       # "unsafe_condition" | "near_miss" | "positive_observation"
    location_code: str,          # validated against enterprise master data
    description: str,
    severity: str,               # "low" | "medium" | "high" | "critical"
    observation_id: str,         # idempotency key — UUID generated by Flutter client
) -> ObservationResult:
    """
    Create an HSE observation. Idempotent — safe to retry.
    If observation_id already exists, returns the existing record (no duplicate write).
    """
    existing = check_existing_observation(observation_id)
    if existing:
        return ObservationResult(
            status="already_exists",
            observation_ref=existing.ref,
            idempotent=True
        )
    
    # Validate location against master data
    location = validate_location_code(location_code)
    if not location:
        return ObservationResult(
            status="error",
            error_code="INVALID_LOCATION",
            message=f"Location '{location_code}' not found. Use lookup_location to find valid codes.",
            retryable=True
        )
    
    # Write to enterprise system
    result = hse_bff_client.create_observation(
        observation_type=observation_type,
        location=location,
        description=description,
        severity=severity,
        observation_id=observation_id,
        user_token=get_current_user_token()
    )
    
    return ObservationResult(
        status="created",
        observation_ref=result.reference_number
    )
```

**Result after deploy:** Zero duplicate observations in production. The guardrail has absorbed 12 retried tool calls in 6 months.

### 3. Server-Side VAD for Barge-In

The agent speaks confirmation back to the worker. If the worker interrupts ("no, change severity to high"), the voice backend detects speech onset server-side and cancels the in-progress audio response — the agent stops mid-sentence and processes the correction. This required implementing server-side VAD rather than relying on client-side detection.

### 4. Structured Tool Errors the Agent Can Reason Over

```python
class ToolError(TypedDict):
    code: str          # "VALIDATION_FAILED" | "NOT_FOUND" | "PERMISSION_DENIED"
    message: str       # safe to surface to end user via voice
    retryable: bool    # agent uses this to decide whether to suggest retry
    suggestion: str    # agent uses this to self-correct (e.g. "use lookup_location first")
```

When `create_hse_observation` returns `INVALID_LOCATION`, the agent automatically calls `lookup_location` with the spoken location name, confirms with the worker, then retries the write — without any hard-coded fallback logic.

---

## Tech Stack

| Component | Technology |
|---|---|
| Mobile | Flutter 3 (Android / iOS / Web) |
| Auth | MSAL (Azure AD), OAuth 2.0 |
| Voice backend | .NET 8, Azure App Service |
| Audio protocol | WebSocket (WSS), PCM16, 24kHz |
| AI Agent | Azure AI Foundry, GPT-4o Realtime |
| MCP Server | Python 3.11, Azure Function Apps |
| BFF | .NET 8, Azure App Service |
| System of record | Quest (enterprise) |
| CI/CD | Azure DevOps |
| Monitoring | Azure Application Insights |

---

## Measured Outcomes

| Metric | Before | After |
|---|---|---|
| Observation reporting time | 2–3 minutes | 10–15 seconds |
| Completion rate (field) | ~60% | ~95% |
| Duplicate records | Occasional | Zero (post-guardrail) |
| Deferred observations | Common | Rare |

---

## Repository Structure

```
hse-voice-ai-agent/
├── docs/
│   ├── architecture.md        # Detailed system design
│   ├── mcp-tools.md           # Full tool definitions + schemas
│   └── oauth-flow.md          # Token propagation sequence diagram
├── mcp_server/                # Python MCP server (Azure Functions)
│   ├── function_app.py
│   ├── tools/
│   │   ├── create_observation.py
│   │   ├── lookup_location.py
│   │   └── get_work_permits.py
│   └── auth/
│       └── token_validator.py
├── voice_backend/             # .NET 8 WebSocket + Foundry integration (structure only)
└── flutter_app/               # Mobile client (structure only)
```

> **Note:** This system was built for an enterprise Oil & Gas client. Business logic and client-specific configuration are not included. The MCP server tool definitions and architecture are documented in full.

---

## Contact

**Avinash Sadolalu Chandaraju**  
avinashavinu@gmail.com  
[LinkedIn](https://www.linkedin.com/in/avinash-s-c-b07575276/) | [GitHub](https://github.com/Avinash01111992)
