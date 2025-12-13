# S4RA Changelog

**Ultimo aggiornamento:** 13 Dicembre 2025

---

# ROADMAP

## ✅ PHASE 0 — POC Proxy & Hard-Gated Control (Completata)

- [x] Migrazione da WebRTC (GA) a WebSocket Proxy (Beta)
- [x] `turn_detection: null` funzionante
- [x] Hard-gated control: modello parla SOLO via `response.create`
- [x] State machine: INTRO → READY → ASSESS_Q1..Q3 → LEVEL → DONE
- [x] State-driven mic lifecycle
- [x] Audio pipeline funzionante
- [x] Lesson Engine v0 (3 domande + livello)

## 🔄 PHASE 1 — Rifinitura POC (In Corso)

- [ ] Silence detection client-side
- [ ] Miglioramento qualità audio
- [ ] UI finale (rimuovere debug)
- [ ] Testing multi-device

## 📋 PHASE 2 — Lesson Engine Completo

- [ ] Roleplay dopo assessment
- [ ] Scenari multipli per livello
- [ ] Feedback dettagliato
- [ ] Correzioni soft durante roleplay

## 📋 PHASE 3 — Scoring & Analytics

- [ ] Scoring engine
- [ ] Salvataggio su database
- [ ] Storico sessioni
- [ ] Report

## 📋 PHASE 4+ — Futuro

- [ ] Modalità dialogo libero
- [ ] Monetizzazione
- [ ] App mobile

---

# PROBLEMI RISOLTI

## Dicembre 2025 — Architettura Proxy

### #24 — response.create dipendeva dal transcript
**Problema:** Server aspettava transcript prima di chiamare `response.create`.

**Soluzione:** `response.create` immediato dopo commit. Il transcript è un dato, non un trigger.

---

### #23 — AudioWorklet non invia frame
**Problema:** Worklet "alive" ma nessun frame audio.

**Soluzione:** Verificato wiring `source.connect(workletNode)`, aggiunto debug logging.

---

### #22 — AudioContext suspended
**Problema:** AudioContext in stato "suspended" per autoplay policy.

**Soluzione:** `await this.audioContext.resume()` dopo creazione.

---

### #21 — Audio buffer sempre vuoto (0 chunks)
**Problema:** Mic ARMED ma mai RECORDING.

**Soluzione:** Auto-transition MIC_ARMED → MIC_RECORDING al primo frame audio.

---

### #20 — Modello parla senza controllo
**Problema:** Con VAD attivo, modello risponde automaticamente.

**Soluzione:** Hard-gated control con `turn_detection: null` + `response.create` esplicito.

---

### #19 — API GA ignora turn_detection
**Problema:** Ephemeral key → API GA → `turn_detection: null` ignorato.

**Soluzione:** Architettura proxy server-side con API Beta e key diretta.

---

## Pre-Dicembre 2025 — Architettura WebRTC (Legacy)

### #1-18 — Vari problemi WebRTC

Tutti risolti o resi obsoleti dalla migrazione a proxy:

1. Race condition session.update
2. "conversation_already_has_active_response"
3. Doppio onboarding
4. "Solo inglese" all'avvio
5. Pronuncia "S4RA" errata
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
16. VAD troppo sensibile
17. Mic si accendeva troppo presto
18. Nessun indicatore visivo
19. Transcript utente non disponibile (WebRTC)
20. Whisper hallucinations

---

# PROBLEMI APERTI

## ⚠️ Qualità audio playback
**Stato:** Non bloccante

**Problema:** Occasionali artefatti audio.

**Mitigazioni:** Seamless scheduling, reset playback su nuovi stati.

---

## ⚠️ Silence detection
**Stato:** Da implementare

**Problema:** Bottone "End Turn (POC only)" richiesto per chiudere turno.

**Soluzione futura:** Client-side silence detection o VAD.

---

# NOTE TECNICHE

## Architettura Attuale

```
Browser ←WebSocket→ ProxyServer ←WebSocket→ OpenAI Beta
```

## File Attivi

```
server/S4RAProxyServer.ts
lib/realtime/proxy/S4RAProxyClient.ts
lib/realtime/proxy/MicrophoneManager.ts
app/poc-proxy/page.tsx
```

## File Frozen (Legacy)

```
lib/realtime/client/*
app/session/page.tsx
```
