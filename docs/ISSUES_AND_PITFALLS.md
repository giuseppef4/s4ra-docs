# S4RA — Known Issues & Pitfalls

Questo documento raccoglie i problemi noti del progetto e le possibili soluzioni.

**Ultimo aggiornamento:** 7 Dicembre 2025

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

**Soluzione:** Delay mic aumentato da 3s a 5s.

---

## 15. S4RA si ferma durante scenario se utente non risponde
**Stato:** ✅ RISOLTO (7 Dic 2025)

**Problema:** Se l'utente non rispondeva, S4RA restava in silenzio.

**Soluzione:** Aggiunta sezione "SILENCE HANDLING" nel prompt.

---

# 🟧 PROBLEMI APERTI

## 16. VAD troppo sensibile (falsi positivi)
**Stato:** 🟧 APERTO (8 Dic 2025)

**Problema:** Il Voice Activity Detection di OpenAI rileva rumore ambientale come "speech" e triggera risposte di S4RA anche quando l'utente non ha parlato.

**Sintomi:**
- `input_audio_buffer.speech_started` appare nei log senza che l'utente parli
- S4RA risponde a "fantasmi"
- Audio di S4RA troncato a volte

**Causa:** Il VAD è gestito lato OpenAI. L'API GA NON accetta il parametro `turn_detection` per modificare threshold/sensibilità.

**Tentativi falliti:**
- Aggiungere `turn_detection` al session.update → errore "unknown_parameter"

**Workaround attuali:**
- Usare ambiente silenzioso
- Cuffie con microfono (meno rumore captato)
- Accettare occasionali falsi positivi

**Possibili soluzioni future:**
- Attendere che OpenAI aggiunga supporto turn_detection all'API GA
- Implementare VAD client-side (complesso)
- Usare WebSocket invece di WebRTC (più controllo, ma più latenza)

---

# 🟦 RISCHI FUTURI

- Conflitti tra Cursor e Claude Desktop
- Modifiche non documentate in `/docs`
- Cambiamenti API OpenAI Realtime

**Mitigazione:** 
- Aggiornare documenti nella cartella /docs del progetto dopo ogni sessione di sviluppo
- Verificare sempre la documentazione ufficiale OpenAI prima di aggiungere parametri

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
├── MicPulse.tsx          ✅ Attivo
└── S4RAVoiceChat.tsx     ✅ Attivo

lib/realtime/client/
├── S4RAClient.ts         ✅ Attivo
├── WebRTCClient.ts       ✅ Attivo
└── useS4RA.ts            ✅ Attivo
```
