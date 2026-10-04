# 🎙️ AI Voice Agent

**Un agente vocale basato sull'intelligenza artificiale che risponde al telefono come una persona:
capisce, ragiona, agisce sui sistemi del locale e risponde a voce, in tempo reale e in italiano.**

Il primo caso d'uso reale è una **pizzeria/ristorante**: l'agente risponde alle telefonate, prende
prenotazioni dei tavoli e ordini d'asporto, risponde su menu, allergeni e orari, e tutto finisce in
una **dashboard web** usata dal personale in sala, dal tablet o dal telefono.

> ℹ️ Questa repository **non contiene il codice sorgente**. Descrive lo stack tecnologico, le
> funzionalità, le difficoltà incontrate (e come sono state affrontate) e la visione futura del
> progetto: da agente telefonico per attività commerciali a **assistente personale stile Jarvis**.

### ⚡ In breve
- **In produzione**, con telefonate vere di clienti veri: non è un prototipo da demo.
- **Agente AI con strumenti** (16 funzioni: disponibilità, prenotazioni, ordini, modifiche,
  disdette, trasferimento, richiamate…) che agisce su un database reale.
- **Human-in-the-loop**: nessuna azione viene salvata senza che il cliente l'abbia detta e
  confermata; un livello di verifica blocca i dati inventati dal modello.
- **Tempo reale**: audio bidirezionale, interruzioni gestite, strumenti asincroni che non bloccano la voce.
- **Prodotto completo**: backend, telefonia, dashboard web responsive, deploy, sicurezza, log e test.
- Progettato, sviluppato e messo in produzione **da solo**, end-to-end.

> 🇬🇧 **TL;DR** — A production real-time **AI voice agent** that answers a restaurant's phone line,
> books tables and takes orders through **tool calling** with a **human-confirmation and grounding
> layer** (no action is committed unless the caller said and confirmed it), plus a staff dashboard
> sharing the same data. Stack: Gemini Live (speech-to-speech), Telnyx, Python/FastAPI, Firestore,
> Cloud Run. Roadmap: on-device AI, autonomous agents that propose actions (including payments) and
> wait for user approval, up to a Jarvis-like assistant for computer, phone, smartwatch and smart home.

---

## 📑 Indice

1. [Cos'è](#-cosè)
2. [Stack tecnologico](#-stack-tecnologico)
3. [Architettura](#-architettura)
4. [Funzionalità](#-funzionalità)
5. [Difficoltà del progetto](#-difficoltà-del-progetto)
6. [Roadmap e sviluppi futuri](#-roadmap-e-sviluppi-futuri)
7. [Principi di progettazione](#-principi-di-progettazione)
8. [Competenze dimostrate](#-competenze-dimostrate)

---

## 💡 Cos'è

Un **agente vocale voce-a-voce** (speech-to-speech): il cliente chiama un normale numero di
telefono, l'agente risponde, conversa in modo naturale, può essere interrotto mentre parla
(*barge-in*) e intanto **usa degli strumenti** (function calling) per leggere il menu, controllare
i tavoli liberi, registrare una prenotazione, passare la chiamata a un collega o chiudere la
telefonata.

A differenza delle classiche pipeline *STT → LLM → TTS* (tre servizi separati, tre latenze che si
sommano), il cuore del progetto è un **modello multimodale nativo audio** che ascolta e parla
direttamente: latenza molto più bassa, voce più naturale, interruzioni gestite in modo nativo.

---

## 🧰 Stack tecnologico

### Intelligenza artificiale
| Componente | Tecnologia | Ruolo |
|---|---|---|
| Modello vocale | **Google Gemini Live API** (modello *native audio*) | Ascolta, ragiona e risponde direttamente in audio |
| Backend del modello | **Vertex AI** (service account) oppure **Google AI Studio** (API key) | Selezionabile da configurazione, stesso codice |
| SDK | `google-genai` | Sessione Live su WebSocket, function calling, trascrizioni |
| Function calling | Strumenti dichiarati con schema JSON, anche **asincroni** (`NON_BLOCKING`) | L'agente agisce sui dati del locale senza bloccare l'audio |

### Telefonia
| Componente | Tecnologia | Ruolo |
|---|---|---|
| Operatore VoIP | **Telnyx Voice API** (Call Control + Media Streaming) | Numero di telefono, webhook di chiamata, stream audio bidirezionale |
| Audio | μ-law 8 kHz ⇄ PCM 16/24 kHz | Conversione tra la qualità telefonica e quella del modello |
| Sicurezza | Firma **Ed25519** dei webhook, **HMAC** sui dati dello stream | Niente chiamate finte o numeri contraffatti |

### Backend
| Componente | Tecnologia |
|---|---|
| Linguaggio | **Python 3.14** (`asyncio` ovunque) |
| Server | **FastAPI** + **Uvicorn** (HTTP per i webhook, WebSocket per l'audio) |
| Client HTTP | `httpx` (comandi REST verso Telnyx: answer, transfer, speak, hangup) |
| Database | **SQLite** in locale, **Google Firestore** in produzione (stessa interfaccia) |
| Agenda | Motore interno su database, compatibile con **Google Calendar API** (attivabile) |
| Registrazioni | **Google Cloud Storage** (opzionale) |

### Infrastruttura e DevOps
| Componente | Tecnologia |
|---|---|
| Container | **Docker** (immagine `python-slim`, solo il server) |
| Hosting | **Google Cloud Run** (scala a zero, region europea vicina ai clienti) |
| Build | **Cloud Build** (nessun Docker richiesto in locale) |
| Segreti | **Secret Manager** (API key, password del pannello, token) |
| Job pianificati | **Cloud Scheduler** (sincronizzazione notturna di menu e orari) |
| Log | **Cloud Logging** con una sigla per telefonata + script di scarico in locale |
| Test | `unittest` su motore, pannello, server e log, con **telefonate vere rigiocate** come casi di test |

### Dashboard
| Componente | Tecnologia |
|---|---|
| Frontend | HTML + CSS + JavaScript vanilla, responsive (desktop, tablet, telefono) |
| API | REST su FastAPI, stesso database dell'agente |
| Accesso | Account per operatore, ruoli *sala* / *titolare*, password cifrate |

---

## 🏗️ Architettura

```
                    ┌──────────────────────────── Google Cloud Run ─────────────────────────────┐
                    │                                                                           │
 📞 Cliente ──► Telnyx ──webhook──► Server FastAPI ──► Core dell'agente ◄──WebSocket──► Gemini Live API
   (telefono)       ◄──audio μ-law──►  (telefonia)      │   (sessione,                     (Vertex AI)
                    │                                   │    audio, tool)
                    │                                   ▼
                    │                    Strumenti: menu · tavoli · prenotazioni
                    │                    asporto · richiami · trasferimento
                    │                                   │
                    │                                   ▼
                    │              Agenda + Registro (SQLite / Firestore) ◄─── Dashboard web
                    │                                                         (sala, titolare)
                    └───────────────────────────────────────────────────────────────────────────┘

 🖥️ Modalità sviluppo:  Microfono ──► Core dell'agente ──► Gemini Live API ──► Altoparlanti
```

Lo **stesso codice** gira in tre ambienti, senza rami condizionali sparsi:

| Ambiente | Audio | Credenziali |
|---|---|---|
| Test locale | microfono e cuffie del PC | file del service account |
| Telefonia in locale | telefono vero, tunnel verso il PC | file del service account |
| Produzione | telefono vero, Cloud Run | identità del servizio (nessuna chiave nell'immagine) |

---

## ✨ Funzionalità

### 📞 Al telefono (agente vocale)
- **Conversazione naturale in italiano**, con interruzioni gestite (il cliente può parlare sopra l'agente).
- **Prenotazione tavoli**: verifica della disponibilità per giorno e orario, numero di persone,
  assegnazione automatica del tavolo (anche tavoli uniti), nome e telefono del cliente.
- **Modifica, disdetta e ripristino** di una prenotazione esistente, ritrovata per nome o numero.
- **Ordini d'asporto**: pizze, antipasti, panini, dolci e bevande dal menu vero, con varianti,
  aggiunte, impasti speciali e supplementi; stima dei tempi di attesa e orario di ritiro.
- **Domande su menu, ingredienti e allergeni** (con le cautele di legge: l'agente dice cosa riporta
  il menu, non inventa).
- **Informazioni sul locale**: orari, giorni di chiusura, parcheggio, seggiolone, coperto…
- **Riconoscimento del numero del chiamante**: non serve dettarlo, ma l'agente gestisce anche numeri
  nascosti e chiamate inoltrate dal numero del locale.
- **Passaggio a un operatore umano** quando il cliente lo chiede o qualcosa non va, con messaggio
  di cortesia se nessuno risponde.
- **Richiamate**: se l'agente non può aiutare (fuori orario, gruppo troppo numeroso, problema
  tecnico) registra la richiesta e il personale la ritrova nella dashboard.
- **Orari dell'agente** configurabili: risponde sempre, solo in certe fasce, in pausa, oppure con
  regole speciali per festività e date particolari.
- **Chiusura della chiamata** pulita: saluto, conferma che l'audio è finito, poi riaggancio.
- **Costo di ogni chiamata** stimato e registrato a fine telefonata (token audio e testo).
- **Registrazione delle chiamate** (opzionale) ascoltabile dalla dashboard.

### 🖥️ Dashboard del locale
- **Oggi**: coperti del turno, prenotazioni, arrivi, tavoli occupati, stato dell'agente con pausa rapida.
- **Prenotazioni**: agenda giornaliera, ricerca per nome o telefono, storico di ogni modifica
  (chi l'ha fatta: l'agente o un operatore), creazione manuale con i posti liberi calcolati con le
  stesse regole dell'agente.
- **Asporto**: ordini per stato (da preparare, in preparazione, pronto, ritirato), modifica dei piatti.
- **Da richiamare**: lista dei clienti che aspettano una risposta.
- **Impostazioni** (titolare): orari dell'agente, date speciali, tavoli e posti, operatori e ruoli.
- **Sincronizzazione in tempo reale**: una prenotazione presa al telefono compare in sala in pochi
  secondi, e un tavolo segnato dalla sala lo vede subito l'agente.

### ⚙️ Dietro le quinte
- **Sincronizzazione automatica del menu** dal sito del locale (prezzi, ingredienti, allergeni,
  supplementi) ogni notte, con copia di riserva se il database non risponde.
- **Anagrafica clienti**: il cliente abituale viene riconosciuto.
- **Log per telefonata**: ogni riga è marcata con la sigla della chiamata, così più telefonate in
  contemporanea non si mescolano.

### 🧩 Cosa può fare in generale un agente vocale di questo tipo
Lo stesso motore si adatta a molti settori cambiando solo strumenti e istruzioni:

| Settore | Esempi di funzioni |
|---|---|
| Ristorazione | prenotazioni, asporto, delivery, menu, eventi privati |
| Sanità / ambulatori | appuntamenti, promemoria, smistamento delle urgenze, informazioni su referti |
| Saloni, centri estetici, palestre | agenda, disdette, liste d'attesa, abbonamenti |
| Hotel e B&B | disponibilità camere, check-in, servizi, richieste degli ospiti |
| Officine e assistenza tecnica | apertura ticket, stato della riparazione, preventivi |
| Negozi ed e-commerce | stato dell'ordine, resi, disponibilità prodotti |
| Studi professionali | filtro delle chiamate, presa messaggi, fissare appuntamenti |
| Customer care | FAQ di primo livello, raccolta dati, escalation a un umano |
| Chiamate in uscita | conferma appuntamenti, sondaggi di soddisfazione, recupero clienti |

Capacità trasversali: multilingua, riconoscimento del cliente, integrazione con CRM / calendari /
gestionali, SMS o WhatsApp di conferma dopo la chiamata, riepiloghi e analisi delle conversazioni,
trasferimento intelligente a reparti diversi.

---

## 🧗 Difficoltà del progetto

Costruire un agente vocale che funzioni **davvero**, al telefono, con clienti veri, è molto più
difficile di una demo. Queste sono le sfide principali affrontate.

### 1. Latenza e naturalezza
- Al telefono una pausa di più di un secondo sembra un guasto. Le pipeline a tre stadi
  (riconoscimento, LLM, sintesi) accumulano ritardi: da qui la scelta di un modello **speech-to-speech**.
- Il **barge-in** (il cliente che parla sopra l'agente) deve fermare subito l'audio in riproduzione
  e scartare quello già in coda verso il telefono.
- Il **cold start** del container dopo un periodo di inattività rallenta la prima chiamata: è un
  compromesso tra costo (scalare a zero) e prontezza.

### 2. Audio telefonico
- Il telefono viaggia in **μ-law a 8 kHz**, il modello vuole PCM 16 kHz in ingresso e produce
  24 kHz in uscita: serve conversione e ricampionamento in tempo reale, in entrambe le direzioni.
- La libreria standard di Python per questa conversione è stata **rimossa da Python 3.13**: va
  sostituita con un pacchetto esterno.
- Eco, rumore di fondo, linee disturbate, persone che parlano in auto o in un locale affollato.
- Nei test in locale col PC servono le **cuffie**, altrimenti l'agente sente se stesso.

### 3. Function calling in tempo reale
- Mentre uno strumento lavora (es. controllo dei tavoli sul database) l'audio **non deve fermarsi**:
  gli strumenti lenti girano in task separati e il ciclo di ricezione audio non si blocca mai.
- Il comportamento reale del modello è diverso da quello documentato e **va misurato**: un risultato
  consegnato mentre il modello sta parlando può andare perso, quindi va trattenuto e consegnato a
  fine frase.
- Alcuni parametri sono accettati da un backend ma fanno chiudere la connessione all'altro: serve
  uno strato che adatti le richieste al backend attivo.
- Se lo strumento tarda, l'agente deve dire "sto controllando, un attimo" senza inventare il risultato.

### 4. Allucinazioni e affidabilità ("grounding")
Un LLM tende a **riempire i vuoti**: un nome mai detto, un orario dedotto, una pizza che non esiste.
Al telefono un errore diventa una prenotazione sbagliata. Per questo esiste un livello di
**verifica** tra il modello e il database:
- ogni dato di una prenotazione (data, ora, persone, nome, telefono, piatti) deve essere stato
  **davvero detto dal cliente** nella trascrizione, altrimenti lo strumento rifiuta e l'agente chiede;
- il cliente deve aver **confermato il riepilogo** prima del salvataggio;
- un piatto che non è nel menu non viene accettato, anche se il modello lo propone;
- una disdetta richiede una conferma esplicita.

### 5. Nomi, numeri e date a voce
- I **nomi italiani** (e stranieri) vengono trascritti in modi diversi: si usano elenchi di nomi e
  cognomi reali, distanza di edit e una chiave fonetica per riconoscere il nome giusto.
- **Numeri di telefono** dettati a gruppi, con ripetizioni e correzioni ("no scusi, 3-4-7").
- **Date relative**: "sabato prossimo", "dopodomani", "stasera alle nove" → data e ora esatte,
  tenendo conto del fuso orario e dei giorni di chiusura.
- Trascrizioni che a volte arrivano **in un altro alfabeto**: vanno riconosciute e scartate.

### 6. Logica di business reale
- Assegnazione dei tavoli: capienza, tavoli uniti, turni, ultimo orario prenotabile, gruppi numerosi.
- Ordini d'asporto: varianti, aggiunte con supplementi variabili, impasti speciali, tempi della cucina.
- **Allergeni**: tema con responsabilità legali; l'agente riporta solo quello che dice il menu e lo dichiara.
- Fuori orario, festività, chiusure straordinarie, pause decise dal titolare all'ultimo minuto.

### 7. Integrazione telefonica
- Con Call Control **chiudere la connessione audio non chiude la telefonata**: il riaggancio va
  comandato esplicitamente, e solo dopo che il saluto è stato davvero riprodotto.
- **Trasferimento a un umano** con timeout e messaggio di riserva se nessuno risponde.
- Chiamate **inoltrate** che mostrano il numero del locale invece di quello del cliente.

### 8. Sicurezza
- L'endpoint dei webhook è pubblico per forza (l'operatore telefonico non si autentica con IAM):
  ogni webhook va verificato con la **firma Ed25519** e un controllo sul timestamp.
- I dati che passano allo stream audio sono firmati con **HMAC**, così non si può falsificare il
  numero del chiamante.
- Nessuna chiave privata nell'immagine Docker; segreti in Secret Manager; password del pannello cifrate.
- Rischio di **prompt injection a voce** e di uso improprio della linea: limiti chiari su cosa
  l'agente può fare.

### 9. Infrastruttura e costi
- Una WebSocket conta come **una sola richiesta HTTP**: con il timeout di default la telefonata
  cadrebbe dopo pochi minuti.
- Il modello Live fa pagare **a ogni turno tutto il contesto** (istruzioni + conversazione): il
  costo al minuto cresce con la durata della chiamata, quindi le istruzioni vanno tenute compatte.
- La Live API è disponibile **solo in alcune region**: va scelta vicina ai clienti.
- Codifica **UTF-8** ovunque: senza, accenti e caratteri speciali arrivano corrotti nei log.
- Token OAuth che scadono (app in modalità test: 7 giorni) → passaggio a un service account.

### 10. Osservabilità e test
- Più telefonate in contemporanea producono log intrecciati: serve una **sigla per chiamata** su
  ogni riga e uno strumento che ricostruisca ogni conversazione.
- Testare un sistema non deterministico: le **telefonate reali problematiche** diventano casi di
  test che vengono rigiocati a ogni modifica.
- Il prompt di sistema è codice: ogni modifica può rompere comportamenti che prima funzionavano.

### 11. Esperienza del personale
- La dashboard deve essere usabile **in sala, di fretta, dal telefono**.
- Agente e personale scrivono sugli **stessi dati** nello stesso momento: servono storico,
  tracciabilità di chi ha fatto cosa e possibilità di annullare.

---

## 🚀 Roadmap e sviluppi futuri

L'agente telefonico per attività commerciali è solo la **prima fase**. Il motore (voce in tempo
reale + strumenti + verifica dei dati + memoria) è lo stesso che serve per un vero assistente
personale. L'obiettivo finale è un assistente in stile **J.A.R.V.I.S.**

```
 Fase 1              Fase 2               Fase 3                 Fase 4               Fase 5
 ───────────         ───────────          ───────────            ───────────          ───────────
 Agente vocale  ──►  Piattaforma     ──►  Assistente del    ──►  Assistente su   ──►  J.A.R.V.I.S.
 telefonico          multi-attività       computer               smartphone e         casa domotica
 (pizzeria) ✅       (SaaS)                                      smartwatch           + studio
```

### ✅ Fase 1 — Agente vocale telefonico per attività (attuale)
Pizzeria/ristorante in produzione, con dashboard, database, sincronizzazione del menu, log e test.

### 🔄 Fase 2 — Piattaforma multi-attività
- Configurazione di un nuovo locale **senza toccare il codice** (menu, tavoli, orari, voce, tono).
- Modelli pronti per settori diversi (ambulatori, saloni, hotel, officine).
- Multi-tenant, fatturazione a consumo, statistiche delle chiamate.
- Conferme via **SMS / WhatsApp**, chiamate in uscita per promemoria.
- Analisi delle conversazioni: motivi delle chiamate, chiamate perse, domande frequenti.

### 🖥️ Fase 3 — Assistente personale del computer
Un assistente sempre disponibile sul PC, attivabile con una **parola chiave** o una scorciatoia.
- Aprire programmi, file e cartelle, gestire finestre, cercare nel computer.
- Leggere e scrivere **email**, gestire **calendario** e promemoria, prendere appunti a voce.
- Riassumere documenti, pagine web e riunioni; dettatura avanzata.
- **Controllo del computer** (vedere lo schermo e usare mouse e tastiera) per compiti a più passi.
- Assistente per **programmazione**: lanciare comandi, leggere errori, spiegare codice.
- Integrazione con strumenti tramite **MCP** (Model Context Protocol) per collegare servizi e app.
- **Memoria personale**: preferenze, persone, progetti in corso, abitudini.
- Esecuzione locale dove possibile (riconoscimento della parola chiave, modelli piccoli) per
  privacy e velocità.

**Difficoltà specifiche**: permessi e sicurezza (un assistente che controlla il PC può fare danni),
conferma delle azioni irreversibili, consumo di risorse in background, attivazione vocale senza
falsi positivi, privacy dei dati locali.

### 📱⌚ Fase 4 — Assistente vocale per smartphone e smartwatch
L'assistente esce dal computer e accompagna la **vita di tutti i giorni**.
- App per **Android e iOS** con attivazione vocale e widget.
- Chiamate e messaggi, navigazione, promemoria basati su **luogo** e **orario**.
- Agenda della giornata, briefing del mattino, lista della spesa, spese e budget.
- **Smartwatch**: comandi rapidi senza tirare fuori il telefono, notifiche intelligenti, salute e
  allenamento (battito, sonno, passi), timer e promemoria al polso.
- Funzionamento **in auto** (mani libere) e con auricolari.
- Continuità tra dispositivi: una conversazione iniziata sul PC continua sul telefono o sull'orologio.
- **Agente che telefona per te**: prenotare un ristorante o un appuntamento chiamando l'attività.

**Difficoltà specifiche**: batteria e consumo dati, limiti dei sistemi operativi mobili per
l'ascolto in background, connettività instabile, latenza su rete cellulare, sincronizzazione della
memoria tra dispositivi, privacy dei dati sanitari e della posizione.

### 🔐 Direzione trasversale — AI on-device e agenti economici autonomi
Un filo che attraversa le fasi 2–5: portare l'intelligenza **sul dispositivo** e permettere
all'agente di **gestire valore**, sempre con l'utente che approva.
- **Modelli locali** (riconoscimento vocale, LLM piccoli, sintesi) che girano su PC e telefono:
  niente audio né dati personali inviati al cloud, funzionamento anche offline.
- **Pagamenti nella conversazione**: l'agente prepara un pagamento (caparra di una prenotazione,
  ordine d'asporto, fattura, rimborso) e lo **propone**; parte solo dopo la conferma esplicita
  dell'utente. È lo stesso schema già in produzione per le prenotazioni: *il modello propone, il
  codice verifica, l'umano approva*.
- **Wallet self-custodial**: le chiavi restano sul dispositivo dell'utente, nessun intermediario
  che custodisce i fondi; l'assistente può leggere saldi e movimenti in locale.
- **Intelligenza finanziaria privata**: budget, spese ricorrenti, avvisi su movimenti sospetti,
  elaborati dal modello locale senza condividere i dati.
- **Agente che paga per te**, con limiti di spesa, liste di destinatari fidati e conferma vocale
  o biometrica per ogni operazione fuori dalle regole.

**Difficoltà specifiche**: modelli locali abbastanza piccoli da girare su un telefono ma abbastanza
affidabili da non sbagliare un importo; impedire che un comando vocale contraffatto o una prompt
injection muovano fondi; UX della conferma che sia sicura ma non fastidiosa.

### 🏠 Fase 5 — J.A.R.V.I.S.: casa domotica e studio
L'obiettivo finale: un assistente che **vive nell'ambiente**, conosce chi ci abita e controlla tutto.

**Casa domotica**
- Integrazione con **Home Assistant**, Matter, Zigbee, Wi-Fi (luci, tapparelle, clima, prese, elettrodomestici).
- Microfoni e altoparlanti in **più stanze**: l'assistente risponde dove ti trovi.
- **Riconoscimento della voce** di chi parla (permessi diversi per adulti, bambini, ospiti).
- Scenari e routine: "buonanotte", "esco", "arrivo tra 20 minuti", sveglia graduale.
- Sicurezza: telecamere, sensori, allarme, avvisi su porte e finestre, citofono intelligente.
- Gestione dell'energia: consumi, fotovoltaico, fasce orarie convenienti.
- Comportamento **proattivo**: avvisare prima che qualcosa succeda, non solo rispondere ai comandi.

**Studio / laboratorio**
- Collegamento ai **propri strumenti di lavoro**: PC, stampanti 3D, strumenti di misura,
  attrezzature audio/video, server e NAS.
- Avvio di build, deploy e test **a voce**, stato dei progetti, lettura dei log.
- Gestione di progetti, documenti e calendario di lavoro.
- **Visione**: tramite telecamera l'assistente vede cosa c'è sul banco di lavoro e aiuta passo passo.
- Dashboard olografica / su schermo con lo stato di casa, studio e progetti.

**Difficoltà specifiche**: affidabilità 24/7 (una casa non può dipendere da internet → serve un
**server locale** con modelli in esecuzione locale e il cloud solo come supporto), sicurezza
informatica di dispositivi fisici, standard domotici frammentati, privacy di microfoni e telecamere
sempre attivi, riconoscimento del parlante, gestione di più persone che parlano insieme.

---

## 🧭 Principi di progettazione

1. **Il modello parla, il codice decide.** L'LLM gestisce la conversazione; regole, disponibilità e
   dati vengono sempre dal codice e dal database, mai dalla "memoria" del modello.
2. **Niente dati inventati.** Ogni informazione salvata deve essere stata detta e confermata.
3. **L'audio non si ferma mai.** Tutto ciò che è lento gira in parallelo alla conversazione.
4. **Un umano è sempre raggiungibile.** Quando l'agente non può aiutare, passa la mano o fa richiamare.
5. **Misurare, non supporre.** Il comportamento reale di modelli e API si verifica con test e log.
6. **Privacy e sicurezza dal primo giorno.** Segreti protetti, endpoint verificati, dati minimi.

---

## 🎯 Competenze dimostrate

| Area | Cosa ho fatto in questo progetto |
|---|---|
| **AI agentica** | Agente con function calling su dati reali, strumenti asincroni, prompt di sistema come codice, gestione dei limiti del modello misurati sul campo |
| **Affidabilità degli LLM** | Livello di *grounding* che verifica ogni argomento degli strumenti contro la trascrizione, conferme esplicite prima di ogni azione, rifiuto dei dati inventati |
| **Sistemi in tempo reale** | Audio bidirezionale su WebSocket, `asyncio`, conversione di formati audio, barge-in, concorrenza tra più chiamate |
| **Sicurezza** | Verifica di firme **Ed25519**, **HMAC** sui dati dello stream, gestione dei segreti, password cifrate, ruoli e permessi |
| **Backend e dati** | FastAPI, API REST, SQLite / Firestore dietro un'unica interfaccia, storico e annullamento delle modifiche |
| **Prodotto e UX** | Dashboard responsive pensata per chi lavora in sala, flussi vocali progettati per clienti reali |
| **Cloud e DevOps** | Docker, Cloud Run, Cloud Build, Secret Manager, Cloud Scheduler, log strutturati per chiamata |
| **Qualità** | Test automatici su motore, server e dashboard; le telefonate reali problematiche diventano casi di test |
| **Esecuzione** | Dall'idea alla produzione in poche settimane, iterando sulle chiamate vere e sul feedback del titolare |

---

## 📬 Contatti

Progetto sviluppato da [@Antoninova](https://github.com/Antoninova).
Il codice sorgente non è pubblico: per demo, collaborazioni o per portare un agente vocale nella
tua attività, apri una *issue* o contattami tramite GitHub.
