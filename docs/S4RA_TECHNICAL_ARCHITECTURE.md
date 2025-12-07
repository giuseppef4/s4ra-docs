# S4RA Technical Architecture

**Ultimo aggiornamento:** 7 Dicembre 2025

## Overview
S4RA usa:
- WebRTC per connessione audio real-time
- OpenAI Realtime API (GA)
- Server VAD (turn detection gestito da OpenAI)
- Prompt-first architecture (logica nel prompt, non nel codice)

---

# 1. Architettura dei livelli

```
UI (React)
    ↓
useS4RA.ts (hook)
    ↓
S4RAClient.ts (client unificato + prompt)
    ↓
WebRTCClient.ts
    ↓
OpenAI Realtime API
```

---

# 2. File attivi

| File | Ruolo |
|------|-------|
| `lib/realtime/client/WebRTCClient.ts` | Layer WebRTC base (connessione, audio, datachannel) |
| `lib/realtime/client/S4RAClient.ts` | Client unificato con State Machine e System Prompt |
| `lib/realtime/client/useS4RA.ts` | Hook React per UI |
| `lib/realtime/types.ts` | TypeScript types |
| `components/VoiceChat/S4RAVoiceChat.tsx` | UI principale |
| `components/VoiceChat/MicPulse.tsx` | Animazione microfono |
| `app/session/page.tsx` | Pagina sessione |
| `app/api/realtime/key/route.ts` | Endpoint per ephemeral key |

---

# 3. Flusso sessione

```
1. User clicca "Start Session"
2. useS4RA.connect() → fetch ephemeral key da /api/realtime/key
3. S4RAClient.connect() → WebRTC handshake con OpenAI
4. datachannel.open → session.update con prompt S4RA
5. conversation.item.create + response.create → S4RA saluta in italiano
6. Mic attivato automaticamente dopo 5 secondi
7. Conversazione gestita dal prompt (fasi multiple)
```

---

# 4. Struttura del System Prompt

Il prompt in `S4RAClient.ts` è diviso in sezioni:

### PHASE 1: ONBOARDING
- Saluto in ITALIANO
- Aspetta conferma utente ("Sei pronto?")
- 3-4 domande in INGLESE (senza correzioni)
- Solo ascolto, niente follow-up

### PHASE 2: LEVEL ASSESSMENT
- Valutazione silenziosa del livello
- Comunicazione livello in ITALIANO
- Spiegazione scenario in ITALIANO
- S4RA INIZIA SUBITO il roleplay (non aspetta lo studente)

### PHASE 3: SCENARIO PRACTICE
- Roleplay in INGLESE
- Correzioni solo per errori che bloccano comprensione
- Scenari basati sul livello (A1-C1)

### SILENCE HANDLING
- Se utente non risponde, S4RA continua naturalmente
- Prompt gentili ("Take your time", "No rush")
- Durante roleplay, continua lo scenario

### END OF SCENARIO
- Feedback SEMPRE in ITALIANO
- Proposta nuovo scenario in ITALIANO

---

# 5. Configurazione Session Update (API GA)

```javascript
{
  type: "session.update",
  session: {
    type: "realtime",
    instructions: S4RA_SYSTEM_PROMPT
  }
}
```

⚠️ **L'API GA NON accetta altri parametri come:**
- `voice`
- `input_audio_format`
- `output_audio_format`
- `input_audio_transcription`
- `turn_detection`
- `modalities`

---

# 6. State Machine (S4RAClient)

```typescript
type SessionState = 
  | "idle"        // Non connesso
  | "connecting"  // WebRTC in corso
  | "ready"       // Connesso, mic spento
  | "active"      // Connesso, mic acceso
  | "error";      // Errore
```

---

# 7. Eventi Realtime gestiti

| Evento | Azione |
|--------|--------|
| `session.created` | Log conferma |
| `session.updated` | Log conferma |
| `conversation.item.input_audio_transcription.delta` | Buffer user transcript |
| `conversation.item.input_audio_transcription.completed` | Emit user transcript |
| `response.audio_transcript.delta` | Buffer assistant transcript |
| `response.output_audio_transcript.delta` | Buffer assistant transcript (alternativo) |
| `response.audio_transcript.done` | Emit assistant transcript |
| `response.output_audio_transcript.done` | Emit assistant transcript (alternativo) |
| `response.done` | Backup per assistant transcript |
| `error` | Log errore API |

---

# 8. Struttura cartelle

```
app/
├── api/
│   └── realtime/key/route.ts    # Ephemeral key
├── session/page.tsx              # Pagina principale
└── ...

components/VoiceChat/
├── MicPulse.tsx                  # Animazione microfono
└── S4RAVoiceChat.tsx             # UI principale

lib/realtime/
├── client/
│   ├── S4RAClient.ts             # Client + prompt
│   ├── WebRTCClient.ts           # WebRTC layer
│   └── useS4RA.ts                # React hook
└── types.ts
```

---

# 9. Problemi noti

Vedi `ISSUES_AND_PITFALLS.md`.
