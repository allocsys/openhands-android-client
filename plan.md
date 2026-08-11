# OpenHands Android Client — Project Plan

## 1. Goal

Build a native Android app that acts as a remote client for [OpenHands](https://github.com/OpenHands/OpenHands), letting a user drive an AI coding agent (bash execution, file edits, etc.) running in a cloud sandbox — from their phone. No existing Android client for OpenHands has been found (as of Aug 2026), so this would be a first.

Model backend is flexible — OpenHands routes LLM calls through LiteLLM, so we can point it at Gemini, GPT, Claude, or others via API key/config, independent of the client.

## 2. High-Level Architecture

```
[Android App] <--REST/WebSocket--> [OpenHands Agent Server] <--spawns--> [Docker Sandbox]
                                           |
                                           v
                                    [LLM API: Gemini / GPT / etc via LiteLLM]
```

- **Android App**: Thin client. Auth, conversation UI, streaming event display, bash/file view.
- **OpenHands Backend**: Self-hosted (not written by us) — FastAPI + WebSocket server, manages agent loop and sandbox lifecycle.
- **Sandbox**: Docker container (or remote VM) spawned per task, where the agent's bash/file operations actually execute.
- **LLM**: Swappable via LiteLLM config — Gemini/GPT/Claude/etc.

## 3. Backend Deployment (reused, not built)

- Self-host OpenHands per `SELF_HOSTING.md`.
- Configure LiteLLM model routing to Gemini and/or GPT APIs (API keys supplied by us).

### 3.1 Hosting evaluation (updated Aug 2026)

Constraint: no ID/credit-card verification (ruled out anything requiring card-based identity checks).

| Option | Findings | Verdict |
|---|---|---|
| **Fly.io** | Free tier removed for new accounts in 2024. New signups get a 2-hour or 7-day trial only, then pay-as-you-go (~$2-5/mo minimum for an always-on shared VM). Requires a card. | Ruled out — no real free tier, plus a card requirement. |
| **Render** | Genuine permanent free tier, no card required. 750 free instance-hours/month, Docker supported. BUT: free web services are capped at 512MB RAM / 0.1 CPU and sleep after 15 min inactivity (30-60s cold start on wake). Not enough headroom to run the OpenHands controller *and* spawn Docker-in-Docker sandbox containers per conversation. | Ruled out as primary backend host. Could still serve a lightweight secondary role later (e.g. webhook/status endpoint). |
| **Oracle Cloud Free Tier** | Genuinely generous always-free ARM VM (4 OCPU / 24GB RAM possible) — would easily fit backend + sandbox. BUT requires a valid credit/debit card for identity verification (no prepaid/virtual/PIN debit cards accepted), plus periodic card-validity holds. Ampere A1 capacity can also be regionally unavailable, and idle instances risk reclaim. | Ruled out — card/ID verification is a hard blocker for us. |
| **Google Cloud Run free tier** | Free tier exists but generally requires billing/card setup to activate compute, similar issue to Oracle. Not fully verified. | Likely blocked too — deprioritized, revisit only if card constraint changes. |
| **E2B (sandbox-only, not full backend)** | Free hobby tier gives a one-time $100 credit, sessions capped at 1 hour. Useful only as a supplemental sandbox execution layer, not a place to run the OpenHands controller itself. | Possible supplemental piece, not primary hosting. |
| **Self-host on owned hardware (spare laptop / Raspberry Pi / home server) + Cloudflare Tunnel** | No card, no ID verification, full control over RAM/CPU for real Docker-in-Docker sandboxing. Cloudflare Tunnel is free and gives a public HTTPS endpoint without exposing a home IP or needing port forwarding. Tradeoffs: uptime/reliability depends on our own hardware and home internet; no vendor SLA. | **Current recommendation** — sidesteps the ID/card blocker entirely and is genuinely free. |

**Decision:** Primary hosting plan is self-hosted OpenHands (backend + Docker sandbox) on owned hardware, exposed via a free Cloudflare Tunnel. Revisit managed cloud options later if the card/ID constraint changes or if home-hardware reliability becomes a problem.
- Note: OpenHands is transitioning from a V0 (FastAPI + Socket.IO, deprecated ~Apr 2026) to a V1 app_server architecture. Plan should target the current V1 API surface — verify against live docs before implementation, not from training knowledge.
- Confirm current REST/WebSocket endpoint shapes: conversations, bash, files, events (per OpenHands Agent Server docs).
- API key based auth for client <-> server.

## 4. Android Client Scope

### 4.1 Core features (v1)
- Connect to a self-hosted OpenHands server (user enters server URL + API key).
- Start/resume a conversation with the agent.
- Send natural-language task messages to the agent.
- Stream agent events in real time via WebSocket (agent thoughts, bash commands run, file diffs, errors).
- Render bash command output and file diffs in a readable mobile-friendly format.
- Basic conversation history / session list.

### 4.2 Stretch features (later)
- Push notifications when a long-running agent task completes (since sandbox can run detached from the app).
- Multiple server profiles (switch between personal VM / team server).
- File browser view of the sandbox workspace.
- Voice input for task description.
- Support switching the backing LLM model from within the app (if server exposes this).

### 4.3 Explicitly out of scope for v1
- Running bash/sandbox locally on-device (not feasible/secure on Android).
- Building our own agent loop or sandbox orchestration — we rely entirely on OpenHands' server.
- Multi-tenant / team management features.

## 5. Tech Stack (proposed)

- **Language/UI**: Kotlin + Jetpack Compose.
- **Networking**: OkHttp (WebSocket) + Retrofit or Ktor client (REST).
- **Async**: Kotlin Coroutines / Flow for streaming events into the UI.
- **Local storage**: DataStore for server profiles/API keys (encrypted where possible).
- **Auth**: API key stored securely (Android Keystore-backed encrypted storage).

## 6. Open Questions / Risks

- **API stability**: V0→V1 migration means endpoint/event schemas may shift under us. Need to lock onto current docs at implementation time, not rely on cached knowledge.
- **Long-running tasks vs mobile lifecycle**: Android kills backgrounded apps/sockets aggressively. Need a strategy (foreground service, push notifications, or polling fallback) so agent tasks don't silently disconnect.
- **Hosting cost/reliability**: Free-tier VMs (Oracle) are viable but have their own reliability caveats; Docker-in-VM sandbox needs enough RAM/CPU headroom.
- **Security**: API keys for LLM providers live on the backend (good — not on device), but the OpenHands server API key + server URL live on the phone. Need secure storage and probably TLS-only connections.
- **Branding/name**: not yet finalized (evaluated Handoff, Palm, Cuff, Codewalk, Remote, Pocket Agent, Sift, Tether, Waypoint, Relay, Anvil, Loop, Satchel, Outpost, Ferry — most collide with existing trademarks/products; revisit later, non-blocking for engineering work).

## 7. Milestones

1. **Backend up**: OpenHands self-hosted on a VM, reachable over HTTPS, LiteLLM wired to Gemini/GPT, verified working via existing web UI.
2. **API mapping**: Document exact current REST endpoints + WebSocket event schema we'll integrate against.
3. **Android skeleton**: Project scaffold, server connection screen, auth/API key storage.
4. **Chat MVP**: Send a message, see streamed agent events render as they arrive (text only).
5. **Bash/file rendering**: Format bash output and file diffs properly in the UI.
6. **Session persistence**: Reconnect to an in-progress or past conversation.
7. **Polish/stretch**: notifications, multiple profiles, file browser.

## 8. Status

Planning stage. No code written yet. Backend not yet deployed. Branding not finalized (non-blocking).
