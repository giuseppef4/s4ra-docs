# S4RA Roadmap

**Ultimo aggiornamento:** 13 Dicembre 2025

---

## PHASE 0 — POC Proxy & Hard-Gated Control ✅ COMPLETATA

- [x] Migrazione da WebRTC (GA) a WebSocket Proxy (Beta)
- [x] `turn_detection: null` funzionante
- [x] Hard-gated control: modello parla SOLO via `response.create`
- [x] State machine: INTRO → READY → ASSESS_Q1..Q3 → LEVEL → DONE
- [x] State-driven mic lifecycle (MIC_OFF/ARMED/RECORDING/COMMITTED)
- [x] Audio pipeline: getUserMedia → AudioWorklet → PCM16 → base64
- [x] Commit/buffering funzionante
- [x] Trascrizione utente via `input_audio_transcription`
- [x] Lesson Engine v0 (3 domande progressive + valutazione livello)
- [x] Debug logging completo e tracciabile

---

## PHASE 1 — Stabilità del Realtime System ✅ COMPLETATA (Legacy WebRTC)

*Nota: Questa fase si riferisce all'architettura WebRTC legacy, ora frozen.*

- [x] Fix definitivo D/4 (session.update dopo datachannel.open)
- [x] Rimuovere logica duplicata (Sequencer eliminato)
- [x] Architettura semplificata (S4RAClient unico)
- [x] Prompt strutturato in fasi
- [x] Pronuncia "Sara" corretta
- [x] Valutazione livello in italiano
- [x] Fix "S4RA si ferma dopo scenario"
- [x] Rimuovere file deprecati
- [x] Fix formato session.update per API GA
- [x] S4RA aspetta "Sei pronto?" prima di procedere
- [x] Feedback finale in italiano
- [x] Gestione silenzio utente
- [x] Fix balbettio iniziale

---

## PHASE 1.5 — Rifinitura POC Proxy 🔄 IN CORSO

- [ ] Silence detection client-side (rimuovere bottone "End Turn")
- [ ] Miglioramento qualità audio playback
- [ ] Testing su più device/browser
- [ ] UI finale (rimuovere debug panel)

---

## PHASE 2 — Lesson Engine Completo

- [ ] Roleplay dopo assessment
- [ ] Scenari multipli basati su livello
- [ ] Feedback dettagliato post-scenario
- [ ] Correzioni soft durante roleplay
- [ ] Gestione "non capisco" con spiegazione italiana
- [ ] Personalizzazione livello post-onboarding

---

## PHASE 3 — Scoring & Analytics

- [ ] Scoring engine (accuratezza, fluidità, lessico)
- [ ] Salvataggio transcript su database
- [ ] Storico sessioni
- [ ] Report sessione
- [ ] Trend e progressi

---

## PHASE 4 — Features Avanzate

- [ ] Modalità dialogo libero
- [ ] Modalità "lezione guidata"
- [ ] Modalità esercizi
- [ ] Cache profilo studente
- [ ] Guardrail anti-deviazione scenario

---

## PHASE 5 — Monetizzazione

- [ ] Piani Premium
- [ ] Analisi settimanale progressi
- [ ] Report PDF
- [ ] Limiti sessioni per tier
- [ ] Setup Stripe

---

## PHASE 6 — Espansione

- [ ] Voce S4RA personalizzata
- [ ] Supporto altre lingue
- [ ] App mobile (React Native / PWA)
