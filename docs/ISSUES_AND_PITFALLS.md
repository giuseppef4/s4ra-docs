# S4RA — Known Issues & Pitfalls

Questo documento raccoglie i problemi noti del progetto e le soluzioni implementate.

**Ultimo aggiornamento:** 13 Dicembre 2025

---

# 🟩 PROBLEMI RISOLTI

## 1-18. Problemi Legacy WebRTC (Dic 2025)

Tutti i problemi relativi all'architettura WebRTC sono stati risolti con la migrazione a **WebSocket Proxy**.

Vedi sezione "Architettura Legacy" per riferimento storico.

---

## 19. API GA vs Beta — Limitazioni turn_detection

**Stato:** ✅ RISOLTO (12-13 Dic 2025)

**Problema:** L'API GA (ephemeral key) ignora `turn_detection: null`. Il VAD resta sempre attivo, impossibile controllare i turni.

**Scoperta:**
- Ephemeral key da `openai.realtime.clientSecrets.create()` → API GA → limitazioni
- API Beta richiede: header `OpenAI-Beta: realtime=v1` + API key diretta

**Soluzione:** Architettura proxy server-side:
```
Browser ←WebSocket→ ProxyServer ←WebSocket→ OpenAI Beta API
```

Il server usa l'API key direttamente con header Beta, permettendo `turn_detection: null`.

---

## 20. Modello parla senza controllo

**Stato:** ✅ RISOLTO (12-13 Dic 2025)

**Problema:** Con VAD attivo, il modello risponde automaticamente a qualsiasi input audio.

**Soluzione:** Hard-gated control:
- `turn_detection: null` disabilita VAD
- Il modello parla SOLO quando il server chiama `response.create`
- Ogni risposta è tracciabile a uno specifico stato

---

## 21. Audio buffer sempre vuoto (0 chunks)

**Stato:** ✅ RISOLTO (13 Dic 2025)

**Problema:** `[Mic] Committing 0 chunks` — il mic era ARMED ma non passava mai a RECORDING.

**Causa:** `startRecording()` veniva chiamato con `setTimeout` ma i frame audio arrivavano prima.

**Soluzione:** Auto-transition da MIC_ARMED a MIC_RECORDING al primo frame audio:
```typescript
if (this.micState === "MIC_ARMED") {
  this.setMicState("MIC_RECORDING");
}
```

---

## 22. AudioContext suspended (autoplay policy)

**Stato:** ✅ RISOLTO (13 Dic 2025)

**Problema:** I probe audio non comparivano nei log. L'AudioContext restava in stato "suspended".

**Soluzione:** Resume esplicito dopo creazione:
```typescript
if (this.audioContext.state === "suspended") {
  await this.audioContext.resume();
}
```

---

## 23. AudioWorklet non invia frame

**Stato:** ✅ RISOLTO (13 Dic 2025)

**Problema:** Worklet "alive" ma nessun frame audio.

**Causa:** Il `process()` del worklet non riceveva input data.

**Soluzione:** Verificato wiring corretto:
```typescript
const source = this.audioContext.createMediaStreamSource(this.mediaStream);
source.connect(this.workletNode);
```

Aggiunto logging di debug nel worklet per tracciare il flusso.

---

## 24. response.create dipendeva dal transcript

**Stato:** ✅ RISOLTO (13 Dic 2025)

**Problema:** Dopo "End Turn", il server aspettava `conversation.item.input_audio_transcription.completed` prima di chiamare `response.create`.

**Correzione concettuale:**
> End Turn = fine turno = `response.create` IMMEDIATO.
> Il transcript è un dato, non un trigger.

**Soluzione:** `handleTurnComplete()` chiamato subito dopo `input_audio_buffer.commit`:
```typescript
case "commit":
  this.sendToOpenAI({ type: "input_audio_buffer.commit" });
  this.handleTurnComplete(); // ← IMMEDIATO
```

---

# 🟧 PROBLEMI APERTI

## 25. Qualità audio playback

**Stato:** ⚠️ DA RIFINIRE (non bloccante)

**Problema:** Occasionali artefatti audio durante il playback.

**Causa probabile:** Chunk scheduling non perfetto o mismatch sample rate.

**Mitigazioni:**
- Seamless scheduling con `source.start(scheduledTime)`
- Reset playback su nuovi stati

---

## 26. Silence detection non implementata

**Stato:** 🟡 DA FARE

**Problema:** L'utente deve cliccare "End Turn (POC only)" per chiudere il turno.

**Soluzione futura:**
- Client-side silence detection (N secondi di silenzio → commit automatico)
- Oppure client-side VAD

---

# 📐 ARCHITETTURA ATTUALE (Dic 2025)

## WebSocket Proxy

```
Browser                    ProxyServer                 OpenAI
   │                           │                          │
   ├──{type:"audio"}──────────►│                          │
   │                           ├──input_audio_buffer.append──►
   │                           │                          │
   ├──{type:"commit"}─────────►│                          │
   │                           ├──input_audio_buffer.commit──►
   │                           ├──response.create────────►│
   │                           │                          │
   │◄──{type:"audio"}──────────┤◄──response.audio.delta───┤
   │◄──{type:"state"}──────────┤                          │
   │◄──{type:"transcript"}─────┤◄──transcription.completed─┤
```

## State Machine

```
IDLE → INTRO → READY → ASSESS_Q1 → ASSESS_Q2 → ASSESS_Q3 → LEVEL → DONE
```

## Mic Lifecycle

```
State allows mic? ──► createMic() ──► MIC_ARMED
                                          │
                                    First audio frame
                                          │
                                          ▼
                                    MIC_RECORDING
                                          │
                                       commit()
                                          │
                                          ▼
                                    MIC_COMMITTED ──► destroyMic() ──► MIC_OFF
```

---

# 📁 FILE ATTIVI

```
server/
├── S4RAProxyServer.ts        ✅ Proxy → OpenAI Beta
└── start-proxy.ts            ✅ Entry point

lib/realtime/proxy/
├── S4RAProxyClient.ts        ✅ Client browser
├── MicrophoneManager.ts      ✅ State-driven mic
└── useS4RAProxy.ts           ✅ React hook

app/poc-proxy/page.tsx        ✅ UI test
```

---

# 🧊 ARCHITETTURA LEGACY (WebRTC)

I seguenti file sono **FROZEN** e mantenuti solo per riferimento:

```
lib/realtime/client/
├── S4RAClient.ts             🧊 FROZEN
├── WebRTCClient.ts           🧊 FROZEN
├── AudioTranscriber.ts       🧊 FROZEN
└── useS4RA.ts                🧊 FROZEN

app/session/page.tsx          🧊 FROZEN
```

### Problemi risolti nell'era WebRTC (1-18)

1. Race condition session.update
2. Errori "conversation_already_has_active_response"
3. Doppio onboarding
4. "Solo inglese" all'avvio
5. Pronuncia "S4RA" come "S-4-R-A"
6. Flusso onboarding non strutturato
7. S4RA si ferma dopo "Inizio io..."
8. UI transcript non coerente
9. idle_timeout_ms malfunzionamenti
10. File deprecati
11. session.update formato sbagliato
12. S4RA non aspettava "Sei pronto?"
13. Feedback finale in inglese
14. Balbettio iniziale
15. S4RA si ferma se utente non risponde
16. VAD troppo sensibile / Echo feedback
17. Mic si accendeva troppo presto
18. Nessun indicatore visivo
19. Transcript utente non disponibile (WebRTC limitation)
20. Whisper hallucinations su silenzio
