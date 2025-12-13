# S4RA Project — Master Document

**Ultimo aggiornamento:** 13 Dicembre 2025

---

# PARTE 1: VISIONE E IDENTITÀ

## 1.1 Cos'è S4RA

S4RA English Tutor è un tutor AI vocale serio e professionale per italiani che vogliono migliorare il proprio inglese parlato. Non è un chatbot, non è un gioco: è un tutor personale disponibile 24/7.

**Obiettivo:** "Parlo con una vera insegnante privata d'inglese"

## 1.2 Target

Persone che vogliono:
- Migliorare l'inglese parlato
- Correzioni utili e contestualizzate
- Percorso strutturato con progressione
- Feedback continui e adattamento al livello (A1 → C1)

## 1.3 Identità di S4RA

### Ruolo
Insegnante privata di inglese, calma, paziente, incoraggiante.

### Personalità
- Tono caldo, rassicurante, non giudicante
- Parla lentamente e chiaramente
- Mai dire "Come assistente AI..."

### Regola Bilingue

S4RA parla **in inglese**, tranne:
- Saluto iniziale (italiano)
- Valutazione livello (italiano)
- Spiegazione scenario (italiano)
- Feedback finale (italiano)
- Quando l'utente mostra difficoltà

### Comportamento Didattico
- Fa parlare lo studente
- Corregge solo errori utili
- Evita spiegazioni lunghe
- Aspetta "Sei pronto?" prima di iniziare

---

# PARTE 2: ARCHITETTURA TECNICA

## 2.1 Stack

- **Frontend:** Next.js 14 + React 18 + Tailwind CSS
- **Backend:** Next.js API Routes + WebSocket Proxy Server
- **Realtime:** OpenAI Realtime API **Beta** via WebSocket Proxy
- **Linguaggio:** TypeScript

## 2.2 Architettura WebSocket Proxy

```
Browser ←WebSocket→ S4RAProxyServer (localhost:8080) ←WebSocket→ OpenAI Beta API
                           │
                           ├── turn_detection: null (funziona)
                           ├── response.create esplicito
                           └── Hard-gated state machine
```

### Perché Proxy invece di WebRTC?

| Aspetto | WebRTC (GA) | WebSocket Proxy (Beta) |
|---------|-------------|------------------------|
| `turn_detection: null` | ❌ Ignorato | ✅ Funziona |
| Controllo turni | ❌ VAD sempre attivo | ✅ Esplicito via `response.create` |
| API Key | Ephemeral (client-side) | Direct (server-side, sicura) |
| Trascrizione | ❌ Non supportata | ✅ `input_audio_transcription` |

## 2.3 File Principali

```
server/
├── S4RAProxyServer.ts        # Proxy → OpenAI Beta, State Machine, Prompts
└── start-proxy.ts            # Entry point

lib/realtime/proxy/
├── S4RAProxyClient.ts        # Client browser (dumb pipe)
├── MicrophoneManager.ts      # State-driven mic lifecycle
└── useS4RAProxy.ts           # React hook

app/poc-proxy/page.tsx        # UI test POC
```

## 2.4 Configurazione OpenAI Beta

```javascript
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
    input_audio_transcription: { model: "whisper-1" },
    turn_detection: null  // CRITICAL: Disabilita VAD
  }
}
```

## 2.5 Protocollo Client ↔ Server

### Client → Server
```typescript
{ type: "audio", audio: string }   // base64 PCM16 chunk
{ type: "commit" }                  // Fine turno utente
{ type: "stop" }                    // Disconnessione
```

### Server → Client
```typescript
{ type: "ready" }                              // Sessione pronta
{ type: "state", state: S4RAState }            // Cambio stato
{ type: "audio", audio: string }               // Audio S4RA
{ type: "transcript", text: string, role: "user" | "assistant" }
{ type: "error", error: string }
{ type: "debug", debug: string }
```

---

# PARTE 3: STATE MACHINE — LESSON ENGINE v0

## 3.1 Stati

```
IDLE → INTRO → READY → ASSESS_Q1 → ASSESS_Q2 → ASSESS_Q3 → LEVEL → DONE
                │
                └→ (not ready) → DONE
```

## 3.2 Hard-Gated Control — Principi

1. **Il modello parla SOLO via `response.create` esplicito**
2. **Un solo `response.create` per stato**
3. **Silenzio è corretto se `response.create` non viene chiamato**
4. **Lo stato è deciso PRIMA di selezionare il prompt**
5. **Un prompt per stato, nessuna composizione dinamica**

## 3.3 Mapping Stato → Comportamento

| Stato | `response.create`? | Mic | Prompt |
|-------|-------------------|-----|--------|
| IDLE | ❌ | OFF | - |
| INTRO | ✅ | OFF | Saluto italiano, "Sei pronto?" |
| READY | ❌ | ARMED/RECORDING | Attende input |
| ASSESS_Q1 | ✅ | ARMED/RECORDING | "What do you like to do on the weekend?" |
| ASSESS_Q2 | ✅ | ARMED/RECORDING | "Tell me about your home. What does it look like?" |
| ASSESS_Q3 | ✅ | ARMED/RECORDING | "Why is learning English important to you?" |
| LEVEL | ✅ | OFF | Valutazione A1/A2/B1/B2 in italiano |
| DONE | ✅ | OFF | Saluto finale italiano |

## 3.4 Flusso Commit → Response

```
End Turn → commit → handleTurnComplete() → transitionTo(nextState) → response.create
```

⚠️ `response.create` è **immediato** dopo commit. Il transcript è un dato, NON un trigger.

## 3.5 Transizioni

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

# PARTE 4: VOICE LIFECYCLE

## 4.1 Principio

> Il microfono esiste SOLO come conseguenza dello stato del Lesson Engine.
> La UI NON apre né chiude il mic.

## 4.2 Stati Mic

- **MIC_OFF:** Nessun getUserMedia, nessun AudioContext, nessun buffer
- **MIC_ARMED:** Pronto, attende primo frame audio
- **MIC_RECORDING:** Registra (auto-transition da ARMED al primo frame)
- **MIC_COMMITTED:** Buffer congelato, mic distrutto, inviato al server

## 4.3 Mapping Stato Lesson → Mic

| Stato Lesson | Mic Permesso? |
|--------------|---------------|
| IDLE, INTRO, LEVEL, DONE | ❌ MIC_OFF |
| READY, ASSESS_* | ✅ MIC_ARMED → MIC_RECORDING |

## 4.4 Lifecycle

```
State allows mic? → createMic() → MIC_ARMED
                                      │
                                First audio frame
                                      │
                                      ▼
                                MIC_RECORDING
                                      │
                                   commit()
                                      │
                                      ▼
                                MIC_COMMITTED → destroyMic() → MIC_OFF
```

## 4.5 Formato Audio

- Mono
- PCM16 little-endian
- 16 kHz (downsampled da browser rate via AudioWorklet)
- Base64 encoded
- Un commit per turno

---

# PARTE 5: REGOLE INVARIANTI

## 5.1 Regole Tecniche

- Il modello parla SOLO quando il server chiama `response.create`
- Silenzio è corretto se `response.create` non viene chiamato
- Un solo `response.create` per stato
- Il mic è una risorsa di stato, non UI-driven
- Mic distrutto dopo ogni commit

## 5.2 Regole di Identità

- Mai modificare identità S4RA senza permesso esplicito
- Mai dire "Come assistente AI..."
- Italiano solo per: saluto, livello, scenario, feedback, aiuto
- Aspettare "Sei pronto?" prima di procedere

## 5.3 Regole di Codice

- Mai codice monolitico — moduli piccoli e testabili
- Mai usare parametri API non documentati
- Aggiornare docs dopo modifiche importanti

---

# PARTE 6: COME AVVIARE

```bash
# Terminale 1 - Proxy
npm run proxy

# Terminale 2 - Next.js
npm run dev

# Browser
http://localhost:3000/poc-proxy
```

---

# PARTE 7: STATO ATTUALE (13 Dicembre 2025)

## Funziona ✅

- Proxy WebSocket → OpenAI Beta API
- Hard-gated control
- State machine completa (INTRO → DONE)
- State-driven mic lifecycle
- Audio pipeline (getUserMedia → AudioWorklet → PCM16 → base64)
- Commit/buffering
- Trascrizione utente
- Lesson Engine v0 (3 domande + livello)
- Debug logging tracciabile

## Da Rifinire ⚠️

- Qualità audio playback (non bloccante)
- Silence detection (per rimuovere bottone POC)

## Da Fare

- Roleplay dopo assessment
- Feedback dettagliato
- Scoring engine
- UI finale
- Multi-device testing

---

# PARTE 8: STRUTTURA CARTELLE COMPLETA

```
s4ra-tutor/
├── server/
│   ├── S4RAProxyServer.ts        # Proxy + State Machine + Prompts
│   └── start-proxy.ts
│
├── lib/realtime/
│   ├── proxy/                    # ← ARCHITETTURA ATTUALE
│   │   ├── S4RAProxyClient.ts
│   │   ├── MicrophoneManager.ts
│   │   └── useS4RAProxy.ts
│   │
│   └── client/                   # ← FROZEN (WebRTC legacy)
│       └── ...
│
├── app/
│   ├── poc-proxy/page.tsx        # UI test proxy
│   └── api/...
│
├── docs/
│   ├── PROJECT.md                # Questo file (master)
│   └── CHANGELOG.md              # Issues risolti + roadmap
│
└── CLAUDE.md                     # Quick reference per Claude
```
