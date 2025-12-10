# S4RA — Known Issues & Pitfalls

Questo documento raccoglie i problemi noti del progetto e le possibili soluzioni.

**Ultimo aggiornamento:** 10 Dicembre 2025

---

# 🟩 PROBLEMI RISOLTI

## 1. D/4 — Race condition session.update
**Stato:** ✅ RISOLTO

**Problema originale:**
- l'AI continuava in inglese anche quando doveva parlare in italiano
- onboarding non sincronizzato
- transcript delta mancanti

**Soluzione implementata:**
- Architettura semplificata con `S4RAClient.ts` unico
- Un solo `session.update` dopo `datachannel.open`
- Eliminato il Sequencer (logica spostata nel prompt)

---

## 2. Errori "conversation_already_has_active_response"
**Stato:** ✅ RISOLTO

**Soluzione:** Un solo `response.create` per messaggio.

---

## 3. Doppio onboarding
**Stato:** ✅ RISOLTO

**Soluzione:** Logica onboarding nel prompt, non più nel codice.

---

## 4. "Solo inglese" all'avvio
**Stato:** ✅ RISOLTO

**Soluzione:** `session.update` è il primo evento dopo `datachannel.open`.

---

## 5. Pronuncia "S4RA" come "S-4-R-A"
**Stato:** ✅ RISOLTO

**Soluzione:** Nel prompt: `pronounced "Sara", not "S-four-R-A"`

---

## 6. Flusso onboarding non strutturato
**Stato:** ✅ RISOLTO

**Soluzione:** Prompt diviso in fasi:
1. PHASE 1: Onboarding (saluto italiano + 3-4 domande inglese)
2. PHASE 2: Level Assessment (valutazione in italiano)
3. PHASE 3: Scenario Practice (roleplay in inglese)

---

## 7. S4RA si ferma dopo "Inizio io..."
**Stato:** ✅ RISOLTO

**Problema:** Dopo aver spiegato lo scenario in italiano, S4RA diceva "Inizio io..." ma non continuava.

**Soluzione:** Prompt aggiornato con istruzione esplicita: "Then IMMEDIATELY say your first line IN ENGLISH. Do NOT wait for the student to speak first. YOU start the roleplay."

---

## 8. UI transcript non sempre coerente
**Stato:** ✅ RISOLTO

**Soluzione:** Buffer separati per user e assistant transcript. Gestione corretta di entrambi gli eventi:
- `response.audio_transcript.delta`
- `response.output_audio_transcript.delta`

---

## 9. idle_timeout_ms causava malfunzionamenti
**Stato:** ✅ RISOLTO (5 Dic 2025)

**Problema:** Aggiungere `idle_timeout_ms: 30000` al turn_detection causava comportamenti imprevedibili.

**Causa:** `idle_timeout_ms` non è un parametro standard dell'API Realtime di OpenAI.

**Soluzione:** Rimosso il parametro.

---

## 10. File deprecati nel progetto
**Stato:** ✅ RISOLTO (5 Dic 2025)

**File rimossi:**
- `app/daily-session/` (duplicato di `/session`)
- `lib/debug/` (file di debug non necessari)
- `components/VoiceChat/VoiceChatContainer.tsx`
- `components/VoiceChat/Controls.tsx`
- `components/VoiceChat/MessageBubble.tsx`
- `components/VoiceChat/Transcript.tsx`

---

## 11. session.update con formato sbagliato
**Stato:** ✅ RISOLTO (7 Dic 2025)

**Problema:** L'API Realtime GA richiede un formato specifico per `session.update`. Molti parametri (voice, input_audio_format, turn_detection, etc.) non sono più accettati.

**Soluzione:** Formato minimo funzionante:
```javascript
{
  type: "session.update",
  session: {
    type: "realtime",
    instructions: S4RA_SYSTEM_PROMPT
  }
}
```

---

## 12. S4RA non aspettava "Sei pronto?"
**Stato:** ✅ RISOLTO (7 Dic 2025)

**Problema:** S4RA iniziava le domande senza aspettare la conferma dell'utente.

**Soluzione:** Prompt aggiornato con "AND WAIT for their response" e "WAIT for their response before proceeding".

---

## 13. Feedback finale in inglese invece che italiano
**Stato:** ✅ RISOLTO (7 Dic 2025)

**Problema:** Alla fine dello scenario, S4RA dava feedback in inglese.

**Soluzione:** Aggiunta sezione "END OF SCENARIO" nel prompt con regola esplicita di dare feedback in ITALIANO.

---

## 14. Balbettio iniziale
**Stato:** ✅ RISOLTO (7 Dic 2025)

**Problema:** Il mic si attivava troppo presto mentre S4RA stava ancora parlando.

**Soluzione:** Mic si attiva automaticamente dopo il primo `output_audio_buffer.stopped`.

---

## 15. S4RA si ferma durante scenario se utente non risponde
**Stato:** ✅ RISOLTO (7 Dic 2025)

**Problema:** Se l'utente non rispondeva, S4RA restava in silenzio.

**Soluzione:** Aggiunta sezione "SILENCE HANDLING" nel prompt.

---

## 16. VAD troppo sensibile / Echo feedback
**Stato:** ✅ RISOLTO (10 Dic 2025)

**Problema originale:** Il Voice Activity Detection di OpenAI rileva rumore ambientale come "speech" e triggera risposte di S4RA anche quando l'utente non ha parlato. Approccio mute/unmute creava esperienza "robotica".

**Sintomi:**
- `input_audio_buffer.speech_started` appare nei log senza che l'utente parli
- S4RA risponde a "fantasmi"
- Audio di S4RA troncato
- Esperienza meccanica "puoi parlare / non puoi parlare"

**Soluzione finale:**
- **Rimosso sistema mute/unmute** - mic sempre attivo dopo primo greeting
- **Conversazione naturale** - utente può interrompere S4RA (come Alexa/Siri)
- **System Prompt rafforzato (Sezione 9)** - istruzioni aggressive per ignorare rumore:
  - "Take your time" max UNA volta per sessione
  - Default: STARE IN SILENZIO se input dubbio
  - Mai reagire a suoni brevi/echo

---

## 17. Mic si accendeva troppo presto (durante saluto S4RA)
**Stato:** ✅ RISOLTO (9 Dic 2025)

**Problema:** Il mic si accendeva con un timer fisso di 5 secondi dopo la connessione, ma il saluto di S4RA dura ~14 secondi.

**Soluzione:**
- Rimosso il `setTimeout` dalla UI
- Il mic si accende automaticamente in `S4RAClient.ts` dopo il primo `output_audio_buffer.stopped`
- Flag `isFirstResponseDone` per tracciare il primo saluto

---

## 18. Nessun indicatore visivo di quando parlare
**Stato:** ✅ RISOLTO poi RIMOSSO (10 Dic 2025)

**Problema originale:** L'utente non sapeva quando poteva parlare.

**Soluzione iniziale:** Indicatore "S4RA sta parlando..." / "Puoi parlare"

**Stato attuale:** Indicatore RIMOSSO perché con approccio "conversazione naturale" non serve più - il mic è sempre attivo e l'utente può sempre parlare/interrompere.

---

## 19. Transcript utente non disponibile (WebRTC limitation)
**Stato:** ✅ RISOLTO (10 Dic 2025)

**Problema:** L'API Realtime GA via WebRTC NON supporta `input_audio_transcription`. Il parametro viene rifiutato con errore `unknown_parameter`.

**Test eseguiti:**
- `/v1/realtime/sessions` (WebSocket): ✅ Supporta `input_audio_transcription`
- `/v1/realtime/client_secrets` (WebRTC): ❌ Rifiuta con "unknown_parameter"

**Soluzione implementata: Hybrid Mode**
- WebRTC per conversazione vocale
- Whisper API in parallelo per trascrizione utente
- `AudioTranscriber.ts` cattura audio dal mic
- `/api/transcribe` proxy verso Whisper API

**File creati:**
- `lib/realtime/client/AudioTranscriber.ts`
- `app/api/transcribe/route.ts`

---

# 🟧 PROBLEMI APERTI

## 20. Whisper hallucinations su silenzio
**Stato:** 🟡 PARZIALMENTE RISOLTO

**Problema:** Quando Whisper riceve silenzio/rumore, inventa testo ("Hello.", ".", "stop it.", etc.)

**Mitigazioni implementate:**
- Filtro pattern comuni in `AudioTranscriber.ts`
- Threshold volume minimo per inviare audio
- `hadRealSound` flag

**Da migliorare:**
- Confidence score da Whisper (verbose_json)
- Threshold più aggressivi

---

# ⚠️ FORMATO SESSION.UPDATE (API GA)

L'API Realtime GA accetta SOLO questi parametri nel session.update:

```javascript
{
  type: "session.update",
  session: {
    type: "realtime",  // OBBLIGATORIO
    instructions: "..."  // Il prompt
  }
}
```

**Parametri NON supportati:**
- `voice`
- `input_audio_format`
- `output_audio_format`
- `input_audio_transcription`
- `turn_detection`
- `modalities`

---

# 📁 STRUTTURA FILE ATTUALE

```
components/VoiceChat/
├── MicPulse.tsx              ✅ Attivo
└── S4RAVoiceChat.tsx         ✅ Attivo

lib/realtime/client/
├── S4RAClient.ts             ✅ Attivo (Client + System Prompt)
├── WebRTCClient.ts           ✅ Attivo
├── AudioTranscriber.ts       ✅ Attivo (Hybrid Mode)
└── useS4RA.ts                ✅ Attivo

app/api/
├── realtime/key/route.ts     ✅ Attivo (Ephemeral key)
└── transcribe/route.ts       ✅ Attivo (Whisper proxy)
```
