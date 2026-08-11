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

- Self-host OpenHands per `SELF_HOSTING.md` on a cloud VM.
- Candidate free/cheap hosting:
  - Oracle Cloud Free Tier (always-free ARM VM) — run backend + Docker sandbox.
  - E2B (free hobby credits) as an alternative/supplemental sandbox provider if not self-hosting Docker directly.
- Configure LiteLLM model routing to Gemini and/or GPT APIs (API keys supplied by us).
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
