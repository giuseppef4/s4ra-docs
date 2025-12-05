# S4RA Project Brain — Master Document

**Ultimo aggiornamento:** 5 Dicembre 2025

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

È ideale per:

- principianti assoluti
- utenti intermedi
- utenti avanzati che cercano fluency

### Value Proposition

S4RA combina:

- voce naturale e conversazione reale
- lezioni strutturate e coerenti
- adattività intelligente
- feedback di qualità
- progressione misurabile
- esperienza semplice, stabile e fluida

### Rischi da evitare

Per mantenere alta la qualità del tutor, è fondamentale prevenire:

- instabilità audio
- feedback incoerenti
- scarsa personalizzazione
- complessità o confusione nell'esperienza utente

S4RA English Tutor è un AI Agent vocale che aiuta italiani a migliorare **speaking**, **listening** e **pronunciation** tramite conversazioni naturali Realtime WebRTC.

Obiettivo finale: *"parlo con una vera insegnante privata"*.

### Moduli principali

1. Conversazione live intelligente
2. Simulazioni reali (ristorante, aeroporto, colloquio…)
3. Piano giornaliero/settimanale
4. Analisi vocale avanzata
5. Progress tracking

### Strumenti

- OpenAI Realtime API
- Next.js web app
- Futuri: Lesson Engine, Scoring Engine, Analysis Engine

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

### Regola Bilingue

S4RA parla sempre **in inglese**, tranne quando l'utente mostra difficoltà ("non ho capito", "non riesco", "me lo spieghi?") o risponde in Italiano. In quel caso:

1. risponde prima **in italiano** per aiutare
2. poi torna **in inglese**

---

## 3. Architettura Realtime (attuale)

### WebRTC

Endpoint: `POST https://api.openai.com/v1/realtime/calls?model=gpt-realtime`

### Audio

- input PCM 24000
- output PCM 24000
- tracce microfono aggiunte prima della offer

### Turn-taking (VAD Settings Funzionanti)

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

⚠️ **NON usare `idle_timeout_ms`** — causa malfunzionamenti.

### Session Update (unico)

Inviato su `dataChannel.onopen`.
Contiene:

- type: realtime
- istruzioni S4RA
- input audio + transcription
- output audio
- turn detection

---

## 4. UI Attuale

- Un bottone "Start Session" / "End Session"
- Animazione MicPulse quando mic attivo
- Transcript collassabile
- Nessun debug log visibile

### Visione UI Finale

- Un bottone "Start"
- Animazione tipo Siri
- S4RA parla per prima
- Nessun testo visibile
- Full voice experience

---

## 5. Lesson Engine (futuro)

- rileva livello
- genera scenario
- adatta difficoltà
- segue obiettivi
- monitora progressi

---

## 6. Scoring Engine (futuro)

- accuratezza
- fluidità
- pronuncia
- lessico
- trend

---

## 7. Analysis Engine (futuro)

- analisi errori ricorrenti
- suggerimenti personalizzati
- esercizi post-sessione

---

## 8. Principi Invarianti

- mai rompere schema Realtime GA
- un solo session.update
- nessun testo nella UI finale
- S4RA sempre nel ruolo
- italiano solo per aiuto → ritorno guidato all'inglese
- semplicità prima di tutto
- mai usare parametri non documentati dell'API OpenAI

---

## 9. Roadmap

### Fase 1 ✅ COMPLETATA

- UI minimale definitiva
- S4RA parla per prima
- turn-taking ottimizzato
- file deprecati rimossi

### Fase 2

- AudioWorklet
- streaming PCM manuale

### Fase 3

- integrazione Lesson, Scoring, Analysis

### Fase 4

- UX finale completa
- animazioni
- schermate progressi

---

## 10. Stato Reale Oggi (5 Dic 2025)

### Funziona ✅

- WebRTC stabile
- Audio bidirezionale
- Personalità S4RA
- Fallback bilingue
- Correzioni pronuncia
- Turn-taking naturale
- Onboarding completo (3 fasi)
- Roleplay automatico dopo assessment

### Da fare

- UI finale
- AudioWorklet
- Lesson Engine
- Scoring Engine
- Report sessione
- Evaluation, Automated Evaluation
- Guardrail avanzati

---

## 11. Posizionamento Strategico di S4RA nel Mercato

S4RA English Tutor non è un chatbot generico né un'app di esercizi gamificati. Il posizionamento del progetto è:

**"Tutor AI vocale serio, professionale e accessibile, con progressione reale e personalizzazione profonda."**

Concorre con soluzioni come Fluently, Duolingo, Elsa Speak, Speak, ChatGPT Realtime, Gemini Live e altri tutor AI, ma si differenzia grazie alla combinazione unica di conversazione naturale, struttura educativa e continuità didattica.

---

## 12. Target Utente di S4RA

S4RA è progettata per utenti che desiderano:

- un tutor vocale disponibile 24/7
- correzioni mirate e intelligenti
- progressione reale
- scenari realistici
- adattamento al proprio livello
- conversazioni pratiche e professionalizzanti

Il target comprende principianti (A1), utenti intermedi (A2–B1) e avanzati (B2–C1).

---

## 13. Value Proposition di S4RA

1. Tutor vocale naturale e credibile
2. Lezioni strutturate e coerenti
3. Adattività intelligente
4. Feedback qualitativo
5. Progressione misurabile
6. Disponibilità continua (H24)

---

## 14. Modello di Prezzo (da definire)

Sulla base del mercato (piani annuali 70–150 \$/anno), si suggerisce un modello composto da:

- trial gratuito
- abbonamento mensile
- piano annuale con sconto
- pacchetti premium con feedback avanzati e analisi

---

## 15. Implicazioni su Lesson / Scoring / Analysis Engine

### Lesson Engine

- lezioni basate su scenario
- difficoltà adattiva
- sequenze didattiche coerenti
- obiettivi chiari e personalizzati

### Scoring Engine

- valutazione pronuncia
- misurazione fluidità
- rilevazione errori grammaticali
- trend e punteggi

### Analysis Engine

- analisi progressi e regressi
- report di sessione
- esercizi mirati
- guardrail contro deviazioni di scenario

---

## 16. Principi Commerciali Fondamentali

1. Il valore percepito viene prima dell'estetica
2. L'utente paga per risultati, non per la conversazione in sé
3. Tracking del progresso come elemento centrale
4. Personalizzazione profonda del percorso
5. Esperienza coerente, stabile e affidabile

---

## 17. Obiettivo Finale

Un tutor d'inglese vocale naturale, empatico, disponibile H24. Un insegnante reale, ma digitale.

---

## 18. File Struttura Attuale

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
