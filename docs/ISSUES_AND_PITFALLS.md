# S4RA — Known Issues & Pitfalls

Questo documento raccoglie i problemi noti del progetto e le possibili soluzioni.

**Ultimo aggiornamento:** 5 Dicembre 2025

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

**Soluzione:** Prompt diviso in 3 fasi:
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

**Problema:** Aggiungere `idle_timeout_ms: 30000` al turn_detection causava comportamenti imprevedibili (S4RA non seguiva il piano, si interrompeva).

**Causa:** `idle_timeout_ms` non è un parametro standard dell'API Realtime di OpenAI.

**Soluzione:** Rimosso il parametro. VAD settings funzionanti:
```javascript
turn_detection: {
  type: "server_vad",
  threshold: 0.45,
  prefix_padding_ms: 600,
  silence_duration_ms: 1600,
  create_response: true,
  interrupt_response: true,
}
```

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

# 🟧 PROBLEMI APERTI

*Nessun problema aperto al momento.*

---

# 🟦 RISCHI FUTURI

- Conflitti tra Cursor e Claude Desktop
- Modifiche non documentate in `/docs`
- Aggiunta di parametri non standard all'API OpenAI

**Mitigazione:** 
- Aggiornare documenti nella cartella /docs del progetto dopo ogni sessione di sviluppo
- Verificare sempre la documentazione ufficiale OpenAI prima di aggiungere parametri

---

# ⚠️ PARAMETRI VAD DA NON USARE

I seguenti parametri NON sono supportati dall'API Realtime di OpenAI e causano problemi:

- `idle_timeout_ms` — causa comportamenti imprevedibili

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
