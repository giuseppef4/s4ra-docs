# S4RA Project Brain — Master Document

**Ultimo aggiornamento:** 10 Dicembre 2025

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

## 3. Architettura Realtime (attuale)

### WebRTC + Hybrid Mode

- **WebRTC:** Comunicazione vocale bidirezionale con OpenAI Realtime API
- **Whisper API (Hybrid Mode):** Trascrizione utente in parallelo (perché WebRTC non supporta `input_audio_transcription`)

Endpoint: `POST https://api.openai.com/v1/realtime/calls?model=gpt-realtime`

### Session Update (API GA)

```javascript
{
  type: "session.update",
  session: {
    type: "realtime",
    instructions: S4RA_SYSTEM_PROMPT
  }
}
```

⚠️ L'API GA non accetta parametri aggiuntivi come voice, turn_detection, input_audio_transcription, etc.

---

## 4. UI Attuale

- Un bottone "Start Session" / "End Session"
- Animazione MicPulse quando mic attivo
- Transcript collassabile con bottone "Copy Transcript"
- Debug log panel con bottone "Copy All"
- **Conversazione naturale:** mic sempre attivo, utente può interrompere

---

## 5. Flusso Sessione Completo

1. **Saluto italiano** - S4RA si presenta, spiega cosa farà, chiede "Sei pronto?"
2. **Aspetta conferma** - L'utente dice "sì", "pronto", etc.
3. **3-4 domande inglese** - Domande semplici, nessuna correzione
4. **Valutazione italiano** - Comunica il livello (A1-C1)
5. **Spiegazione scenario italiano** - Descrive la situazione
6. **Roleplay inglese** - S4RA inizia subito, gestisce silenzio
7. **Feedback italiano** - Alla fine dello scenario

---

## 6. Stato Reale Oggi (10 Dic 2025)

### Funziona ✅

- WebRTC stabile
- Audio bidirezionale
- Personalità S4RA completa
- Fallback bilingue
- Turn-taking naturale
- Onboarding completo
- Aspetta "Sei pronto?"
- Roleplay automatico dopo assessment
- Gestione silenzio utente
- Feedback finale in italiano
- **Conversazione naturale** (mic sempre attivo, niente mute/unmute meccanico)
- **Utente può interrompere** S4RA
- **Hybrid Mode:** Trascrizione utente via Whisper API
- **Gestione rumore/VAD** via System Prompt (sezione 9)

### Da fare

#### - Testing multi-device
#### - UI finale (senza debug)
#### - Lesson Engine
    rileva livello
    genera scenario
    adatta difficoltà
    segue obiettivi
    monitora progressi
    lezioni basate su scenario
    difficoltà adattiva
    sequenze didattiche coerenti
    obiettivi chiari e personalizzati
#### - Scoring Engine
    accuratezza
    fluidità
    pronuncia
    lessico
    trend
    valutazione pronuncia
    misurazione fluidità
    rilevazione errori grammaticali
    trend e punteggi
#### - Report sessione
    analisi errori ricorrenti
    suggerimenti personalizzati
    esercizi post-sessione
    analisi progressi e regressi
    report di sessione
    esercizi mirati
    guardrail contro deviazioni di scenario
#### - Evaluation
    Automated Evaluation
    Test su ogni rilascio
    Coerenza del role
    Rispetto guardrail
    Evaluation continua
    Logging anomalie
    Health score
    Evaluation manuale
    Golden conversations
#### - Guardrail
    Livello Prompt (Nessuna politica o salute, Non uscire dal ruolo, 
    Non criticare l'accento), Livello Backend (Filtri input/output, Rate limiting)
    Livello Lesson Engine (Impedire deviazioni di scenario)

---

## 7. Principi Invarianti

- mai rompere schema Realtime GA
- un solo session.update
- italiano solo per: saluto, livello, scenario, feedback, aiuto
- S4RA sempre nel ruolo
- semplicità prima di tutto
- mai usare parametri API non documentati
- **conversazione naturale** (no mute/unmute meccanico)

---

## 8. File Struttura Attuale

```
app/
├── session/page.tsx          # Pagina principale sessione
└── api/
    ├── realtime/key/         # Ephemeral key endpoint
    └── transcribe/           # Whisper API proxy (Hybrid Mode)

components/VoiceChat/
├── MicPulse.tsx              # Animazione microfono
└── S4RAVoiceChat.tsx         # UI principale

lib/realtime/client/
├── S4RAClient.ts             # Client + System Prompt + AudioTranscriber
├── WebRTCClient.ts           # Layer WebRTC
├── AudioTranscriber.ts       # Cattura audio utente per Whisper
└── useS4RA.ts                # React hook
```
