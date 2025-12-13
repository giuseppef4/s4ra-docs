# S4RA Project Brain — Master Document

**Ultimo aggiornamento:** 13 Dicembre 2025

## 1. Visione del Progetto

S4RA English Tutor è un tutor AI vocale serio e professionale, progettato per offrire un'esperienza educativa strutturata, naturale e accessibile. Non è un chatbot, non è un gioco, non è un assistente generico: è un vero tutor personale disponibile 24/7.

### Posizionamento

S4RA si posiziona come un tutor AI affidabile, intelligente e adattivo, capace di seguire l'utente in un percorso reale di crescita linguistica.

### Target

S4RA è pensata per persone che vogliono:

- migliorare il proprio inglese parlato
- ottenere correzioni utili e contestualizzate
- seguire un percorso strutturato con progressione
- ricevere feedback continui
- adattamento al proprio livello (A1 → C1)
- una guida vocale sempre disponibile

### Value Proposition

S4RA combina:

- voce naturale e conversazione reale
- lezioni strutturate e coerenti
- adattività intelligente
- feedback di qualità
- progressione misurabile
- esperienza semplice, stabile e fluida

---

## 2. Identità di S4RA

### Ruolo

Insegnante privata di inglese, calma, paziente, incoraggiante.

### Personalità

- tono caldo
- rassicurante
- non giudicante
- parla lentamente e chiaramente

### Comportamento Didattico

- fa parlare lo studente
- corregge solo gli errori utili
- evita spiegazioni lunghe
- guida con gentilezza
- aspetta "Sei pronto?" prima di iniziare
- feedback finale sempre in italiano

### Regola Bilingue

S4RA parla sempre **in inglese**, tranne:
- Saluto iniziale (italiano)
- Valutazione livello (italiano)
- Spiegazione scenario (italiano)
- Feedback finale (italiano)
- Quando l'utente mostra difficoltà

---

## 3. Architettura Realtime

### ⚠️ ARCHITETTURA ATTUALE: WebSocket Proxy (Dicembre 2025)

A seguito dei limiti dell'API GA WebRTC (nessun controllo su `turn_detection`), abbiamo implementato un'architettura **proxy server-side**:

```
Browser ←WebSocket→ S4RAProxyServer (localhost:8080) ←WebSocket→ OpenAI Beta API
                           │
                           └── turn_detection: null (FUNZIONA)
                           └── response.create esplicito
                           └── Hard-gated state machine
```

### Perché Proxy invece di WebRTC

| Aspetto | WebRTC (GA) | WebSocket Proxy (Beta) |
|---------|-------------|------------------------|
| `turn_detection: null` | ❌ Ignorato | ✅ Funziona |
| Controllo turni | ❌ VAD sempre attivo | ✅ Esplicito via `response.create` |
| API Key | Ephemeral (client-side) | Direct (server-side, sicura) |
| Trascrizione | ❌ Non supportata | ✅ `input_audio_transcription` |

### File Architettura Proxy

```
server/
├── S4RAProxyServer.ts     # Server WebSocket proxy → OpenAI Beta
└── start-proxy.ts         # Entry point

lib/realtime/proxy/
├── S4RAProxyClient.ts     # Client browser (dumb pipe)
├── MicrophoneManager.ts   # Mic state-driven lifecycle
└── useS4RAProxy.ts        # React hook

app/poc-proxy/page.tsx     # UI di test POC
```

### Architettura Legacy (WebRTC) — FROZEN

I file in `lib/realtime/client/` sono **congelati** con il prompt V1 per riferimento:

```
lib/realtime/client/
├── S4RAClient.ts          # FROZEN - WebRTC legacy
├── WebRTCClient.ts        # FROZEN - WebRTC legacy
└── ...
```

---

## 4. Lesson Engine v0 — State Machine

### Stati

```
IDLE → INTRO → READY → ASSESS_Q1 → ASSESS_Q2 → ASSESS_Q3 → LEVEL → DONE
                │
                └→ (not ready) → DONE
```

### Principi Hard-Gated Control

1. **Il modello parla SOLO via `response.create` esplicito**
2. **Un solo `response.create` per stato**
3. **Silenzio è corretto se `response.create` non viene chiamato**
4. **Lo stato è deciso PRIMA di selezionare il prompt**
5. **Un prompt per stato, nessuna composizione dinamica**

### Mapping Stato → Output

| Stato | `response.create`? | Prompt |
|-------|-------------------|--------|
| IDLE | ❌ No | - |
| INTRO | ✅ Sì | Saluto italiano, "Sei pronto?" |
| READY | ❌ No | Attende input utente |
| ASSESS_Q1 | ✅ Sì | "What do you like to do on the weekend?" |
| ASSESS_Q2 | ✅ Sì | "Tell me about your home. What does it look like?" |
| ASSESS_Q3 | ✅ Sì | "Why is learning English important to you?" |
| LEVEL | ✅ Sì | Valutazione A1/A2/B1/B2 in italiano |
| DONE | ✅ Sì | Saluto finale italiano |

### Flusso Commit → Response

```
End Turn → commit → handleTurnComplete() → transitionTo(nextState) → response.create
```

**IMPORTANTE:** `response.create` viene chiamato **immediatamente** dopo il commit, NON si aspetta il transcript. Il transcript è un dato, non un trigger.

---

## 5. Voice Lifecycle (State-Driven Mic)

### Principio

> Il microfono esiste SOLO come conseguenza dello stato del Lesson Engine.
> La UI NON apre né chiude il mic.

### Mapping Stato → Mic

| Stato | Mic State |
|-------|-----------|
| IDLE | MIC_OFF |
| INTRO | MIC_OFF |
| READY | MIC_ARMED → MIC_RECORDING |
| ASSESS_* | MIC_ARMED → MIC_RECORDING |
| LEVEL | MIC_OFF |
| DONE | MIC_OFF |

### Stati Mic

- **MIC_OFF:** Nessun getUserMedia, nessun AudioContext, nessun buffer
- **MIC_ARMED:** Pronto ma NON registra (attende primo frame audio)
- **MIC_RECORDING:** Registra (auto-transition da ARMED al primo frame)
- **MIC_COMMITTED:** Buffer congelato, mic distrutto, inviato al server

### Regole

1. Mic OFF di default (load, refresh, reconnect)
2. Mic nasce solo se lo stato lo consente
3. Mic viene SEMPRE distrutto dopo commit o cambio stato non valido
4. Nessun audio pre-sessione
5. Server riceve SOLO audio committed
6. Buffer non sopravvive al mic

### Formato Audio

- Mono
- PCM16 little-endian
- 16 kHz (downsampled da browser rate)
- Un commit per turno

---

## 6. UI POC

### Pagina di Test

`/poc-proxy` — UI minimale per testare il flusso:

- Bottone Start/Disconnect
- Bottone "End Turn (POC only)" — solo per delimitare fine turno nel POC
- Display stato Lesson Engine
- Display stato Mic
- Transcript
- Debug logs con "Copy All"

### Note sul Bottone End Turn

> Questo bottone esiste SOLO per il POC.
> In produzione sarà sostituito da silence detection o VAD client-side.
> NON contiene logica di stato o chiamate OpenAI.
> Il suo UNICO ruolo è notificare il server che il turno utente è concluso.

---

## 7. Stato Attuale (13 Dic 2025)

### Funziona ✅

- ✅ **Proxy WebSocket** → OpenAI Beta API
- ✅ **Hard-gated control** — modello parla solo via `response.create`
- ✅ **State machine** — INTRO → READY → ASSESS_Q1..Q3 → LEVEL → DONE
- ✅ **State-driven mic lifecycle** — MIC_OFF/ARMED/RECORDING/COMMITTED
- ✅ **Audio pipeline** — getUserMedia → AudioWorklet → PCM16 → base64
- ✅ **Commit/buffering** — chunks accumulati e inviati al commit
- ✅ **Trascrizione utente** — arriva via `input_audio_transcription`
- ✅ **Lesson Engine v0** — 3 domande progressive + valutazione livello
- ✅ **Debug logging** — tracciabilità completa stato → prompt → response

### Da Rifinire ⚠️

- ⚠️ **Qualità audio playback** — occasionali artefatti, non bloccante
- ⚠️ **Silence detection** — da implementare per rimuovere bottone POC

### Da Fare

- Roleplay dopo assessment
- Feedback dettagliato
- Scoring engine
- UI finale (senza debug)
- Multi-device testing

---

## 8. Principi Invarianti

- Mai modificare identità S4RA senza permesso esplicito
- Mai codice monolitico — moduli piccoli e testabili
- Un solo `response.create` per stato
- Il modello parla SOLO quando il server decide
- Silenzio è corretto se `response.create` non viene chiamato
- Mic è una risorsa di stato, non UI-driven
- Italiano solo per: saluto, livello, scenario, feedback, aiuto

---

## 9. Come Avviare

### Terminale 1 — Proxy Server

```bash
npm run proxy
```

### Terminale 2 — Next.js

```bash
npm run dev
```

### Browser

```
http://localhost:3000/poc-proxy
```

---

## 10. File Struttura Completa

```
s4ra-tutor/
├── server/
│   ├── S4RAProxyServer.ts     # Proxy WebSocket → OpenAI Beta
│   └── start-proxy.ts         # Entry point proxy
│
├── lib/realtime/
│   ├── proxy/                 # ← ARCHITETTURA ATTUALE
│   │   ├── S4RAProxyClient.ts
│   │   ├── MicrophoneManager.ts
│   │   └── useS4RAProxy.ts
│   │
│   ├── client/                # ← FROZEN (WebRTC legacy)
│   │   ├── S4RAClient.ts
│   │   ├── WebRTCClient.ts
│   │   └── ...
│   │
│   └── websocket/             # ← POC iniziale (abbandonato)
│       └── ...
│
├── app/
│   ├── poc-proxy/page.tsx     # UI test proxy
│   ├── session/page.tsx       # UI legacy WebRTC
│   └── api/...
│
├── components/VoiceChat/
│   └── ...                    # UI legacy
│
├── docs/
│   ├── s4ra_project_brain.md  # Questo file
│   └── ISSUES_AND_PITFALLS.md
│
└── CLAUDE.md
```
