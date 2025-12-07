# S4RA Project Brain — Master Document

**Ultimo aggiornamento:** 7 Dicembre 2025

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

### WebRTC

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

⚠️ L'API GA non accetta parametri aggiuntivi come voice, turn_detection, etc.

---

## 4. UI Attuale

- Un bottone "Start Session" / "End Session"
- Animazione MicPulse quando mic attivo
- Transcript collassabile con bottone "Copy Transcript"
- Debug log panel con bottone "Copy All"
- Delay mic: 5 secondi dopo connessione

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

## 6. Stato Reale Oggi (7 Dic 2025)

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
- No balbettio iniziale

### Da fare

- Testing multi-device
- UI finale (senza debug)
- Lesson Engine
- Scoring Engine
- Report sessione

---

## 7. Principi Invarianti

- mai rompere schema Realtime GA
- un solo session.update
- italiano solo per: saluto, livello, scenario, feedback, aiuto
- S4RA sempre nel ruolo
- semplicità prima di tutto
- mai usare parametri API non documentati

---

## 8. File Struttura Attuale

```
app/
├── session/page.tsx          # Pagina principale sessione
└── api/realtime/key/         # Ephemeral key endpoint

components/VoiceChat/
├── MicPulse.tsx              # Animazione microfono
└── S4RAVoiceChat.tsx         # UI principale

lib/realtime/client/
├── S4RAClient.ts             # Client + System Prompt
├── WebRTCClient.ts           # Layer WebRTC
└── useS4RA.ts                # React hook
```
