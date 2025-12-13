# S4RA Technical Architecture

**Ultimo aggiornamento:** 13 Dicembre 2025

---

## Overview

S4RA usa:
- **WebSocket Proxy** per connessione a OpenAI Realtime API Beta
- **Hard-gated control**: modello parla SOLO via `response.create`
- **State-driven architecture**: stato decide prompt, mic, e flusso
- **Prompt-per-state**: un prompt fisso per ogni stato della state machine

---

# 1. Architettura dei Livelli

## Architettura Attuale (WebSocket Proxy)

```
Browser (UI)
    ↓
app/poc-proxy/page.tsx
    ↓
lib/realtime/proxy/S4RAProxyClient.ts
    ↓
    ├── MicrophoneManager.ts (state-driven mic)
    │
    └──WebSocket──► server/S4RAProxyServer.ts
                              │
                              ├── State Machine
                              ├── Prompt per stato
                              │
                              └──WebSocket──► OpenAI Realtime API Beta
                                              (turn_detection: null)
```

## Architettura Legacy (WebRTC) — FROZEN

```
UI (React)
    ↓
useS4RA.ts (hook)
    ↓
S4RAClient.ts (client + prompt)
    ↓
WebRTCClient.ts
    ↓
OpenAI Realtime API GA
```

---

# 2. File Attivi

| File | Ruolo |
|------|-------|
| `server/S4RAProxyServer.ts` | Proxy WebSocket → OpenAI Beta, State Machine, Prompts |
| `server/start-proxy.ts` | Entry point proxy server |
| `lib/realtime/proxy/S4RAProxyClient.ts` | Client browser (dumb pipe audio) |
| `lib/realtime/proxy/MicrophoneManager.ts` | State-driven mic lifecycle |
| `lib/realtime/proxy/useS4RAProxy.ts` | React hook |
| `app/poc-proxy/page.tsx` | UI di test POC |

## File Frozen (Legacy WebRTC)

| File | Stato |
|------|-------|
| `lib/realtime/client/S4RAClient.ts` | 🧊 FROZEN |
| `lib/realtime/client/WebRTCClient.ts` | 🧊 FROZEN |
| `lib/realtime/client/useS4RA.ts` | 🧊 FROZEN |
| `app/session/page.tsx` | 🧊 FROZEN |

---

# 3. State Machine — Lesson Engine v0

## Stati

```
IDLE → INTRO → READY → ASSESS_Q1 → ASSESS_Q2 → ASSESS_Q3 → LEVEL → DONE
                │
                └→ (not ready) → DONE
```

## Mapping Stato → Comportamento

| Stato | `response.create`? | Mic | Prompt |
|-------|-------------------|-----|--------|
| IDLE | ❌ | OFF | - |
| INTRO | ✅ | OFF | Saluto italiano |
| READY | ❌ | ARMED/RECORDING | Attende input |
| ASSESS_Q1 | ✅ | ARMED/RECORDING | Domanda 1 |
| ASSESS_Q2 | ✅ | ARMED/RECORDING | Domanda 2 |
| ASSESS_Q3 | ✅ | ARMED/RECORDING | Domanda 3 |
| LEVEL | ✅ | OFF | Valutazione livello |
| DONE | ✅ | OFF | Saluto finale |

## Transizioni

```typescript
// In handleTurnComplete() - chiamato SUBITO dopo commit
READY     → ASSESS_Q1
ASSESS_Q1 → ASSESS_Q2
ASSESS_Q2 → ASSESS_Q3
ASSESS_Q3 → LEVEL

// In handleResponseComplete() - chiamato dopo response.done
INTRO → READY
LEVEL → DONE
DONE  → cleanup()
```

---

# 4. Flusso Sessione

```
1. User clicca "Start"
2. S4RAProxyClient.connect() → WebSocket a localhost:8080
3. S4RAProxyServer riceve connessione
4. Server apre WebSocket a OpenAI Beta con header "OpenAI-Beta: realtime=v1"
5. Server invia session.update con turn_detection: null
6. session.updated → transitionTo("INTRO") → response.create
7. S4RA saluta in italiano
8. response.done → transitionTo("READY")
9. Mic si arma (MIC_ARMED)
10. User parla → primo frame → MIC_RECORDING → buffer audio
11. User clicca "End Turn" → commit → handleTurnComplete() → ASSESS_Q1
12. ... ciclo per Q2, Q3 ...
13. ASSESS_Q3 commit → LEVEL → valutazione
14. LEVEL response.done → DONE → saluto finale → cleanup
```

---

# 5. Voice Lifecycle (MicrophoneManager)

## Principio

> Il microfono esiste SOLO come conseguenza dello stato del Lesson Engine.
> La UI NON apre né chiude il mic.

## Stati Mic

```
MIC_OFF       →  createMic()  →  MIC_ARMED
                                     │
                               first audio frame
                                     │
                                     ▼
                               MIC_RECORDING
                                     │
                                  commit()
                                     │
                                     ▼
                               MIC_COMMITTED  →  destroyMic()  →  MIC_OFF
```

## Mapping Stato Lesson → Mic

| Stato Lesson | Mic Permesso? |
|--------------|---------------|
| IDLE | ❌ |
| INTRO | ❌ |
| READY | ✅ |
| ASSESS_* | ✅ |
| LEVEL | ❌ |
| DONE | ❌ |

## Formato Audio

- Mono
- PCM16 little-endian
- 16 kHz (downsampled da browser rate via AudioWorklet)
- Base64 encoded
- Un commit per turno

---

# 6. Configurazione OpenAI Beta

```javascript
// In S4RAProxyServer.ts

// WebSocket connection
const url = "wss://api.openai.com/v1/realtime?model=gpt-4o-realtime-preview-2024-12-17";
const ws = new WebSocket(url, {
  headers: {
    "Authorization": `Bearer ${apiKey}`,
    "OpenAI-Beta": "realtime=v1"  // CRITICAL
  }
});

// Session configuration
{
  type: "session.update",
  session: {
    modalities: ["audio", "text"],
    voice: "shimmer",
    input_audio_format: "pcm16",
    output_audio_format: "pcm16",
    input_audio_transcription: {
      model: "whisper-1"
    },
    turn_detection: null  // CRITICAL: Disabilita VAD
  }
}
```

---

# 7. Protocollo Client ↔ Server

## Client → Server

```typescript
{ type: "audio", audio: string }   // base64 PCM16 chunk
{ type: "commit" }                  // Fine turno utente
{ type: "stop" }                    // Disconnessione
```

## Server → Client

```typescript
{ type: "ready" }                              // Sessione pronta
{ type: "state", state: S4RAState }            // Cambio stato
{ type: "audio", audio: string }               // Audio S4RA
{ type: "transcript", text: string, role: "user" | "assistant" }
{ type: "error", error: string }
{ type: "debug", debug: string }               // Log debug
```

---

# 8. Hard-Gated Control

## Principi

1. **Il modello parla SOLO via `response.create` esplicito**
2. **Un solo `response.create` per stato**
3. **Silenzio è corretto se `response.create` non viene chiamato**
4. **Lo stato è deciso PRIMA di selezionare il prompt**
5. **Un prompt per stato, nessuna composizione dinamica**

## Verifica nei Log

```
[CONTROL] response.create #1 for state: INTRO
[OK] response.created (expected)
[OK] response.done - text received (149 chars)
```

Se appare `[WARNING] UNEXPECTED response.created` → il modello ha parlato senza richiesta (BUG).

---

# 9. Struttura Cartelle

```
s4ra-tutor/
├── server/                        # ← PROXY SERVER
│   ├── S4RAProxyServer.ts
│   └── start-proxy.ts
│
├── lib/realtime/
│   ├── proxy/                     # ← ARCHITETTURA ATTUALE
│   │   ├── S4RAProxyClient.ts
│   │   ├── MicrophoneManager.ts
│   │   └── useS4RAProxy.ts
│   │
│   ├── client/                    # ← FROZEN (WebRTC legacy)
│   │   └── ...
│   │
│   └── websocket/                 # ← POC iniziale (abbandonato)
│       └── ...
│
├── app/
│   ├── poc-proxy/page.tsx         # UI test proxy
│   ├── session/page.tsx           # UI legacy
│   └── api/...
│
├── docs/
│   ├── s4ra_project_brain.md
│   ├── S4RA_TECHNICAL_ARCHITECTURE.md  # Questo file
│   ├── S4RA_ARCHITECT_RULES.md
│   ├── ISSUES_AND_PITFALLS.md
│   └── ROADMAP.md
│
└── CLAUDE.md
```

---

# 10. Come Avviare

```bash
# Terminale 1 - Proxy
npm run proxy

# Terminale 2 - Next.js
npm run dev

# Browser
http://localhost:3000/poc-proxy
```

---

# 11. Problemi Noti

Vedi `ISSUES_AND_PITFALLS.md`.
