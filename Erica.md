# 🎤 Traccia Relatore 1 — Introduzione & Architettura

> **Capitoli di riferimento:** Cap. 2 (Architettura Web, SPA, MVC, REST), Cap. 5 (Deploy, Azure, Vercel)  
> **Durata stimata:** ~4 minuti

---

## Apertura — Presentare il progetto

> *"Buongiorno, vi presentiamo Split Mate: una web application full-stack per la gestione e la divisione delle spese condivise. È pensata per coinquilini, viaggi, cene tra amici — situazioni in cui più persone pagano e bisogna tenere traccia di chi deve quanto a chi."*

---

## 1. Cos'è Split Mate — le funzionalità principali

Le funzionalità principali dell'applicazione sono:

- **Login e registrazione** con form separati — il login accede solo ad utenti esistenti
- **Gestione gruppi**: creazione, visualizzazione ed eliminazione; ogni gruppo genera un **codice invito univoco** per aggiungere altri utenti
- **Gestione spese**: chi ha pagato, l'importo, e tra chi viene divisa (tutti o solo alcuni membri selezionati)
- **Calcolo saldi in tempo reale**: un algoritmo calcola chi deve rimborsare chi, minimizzando il numero di transazioni necessarie
- **Eliminazione a cascata**: cancellare un gruppo rimuove in modo sicuro anche le spese e i dati correlati

---

## 2. Tipo di applicazione — SPA

Split Mate è una **SPA (Single Page Application)**.

In una SPA il browser scarica **una sola pagina HTML** al primo accesso — poi JavaScript gestisce tutto: aggiorna il contenuto, cambia la vista, comunica con il server, **senza mai ricaricare la pagina**. Questo è diverso da una **MPA (Multi Page Application)** tradizionale, dove ogni click genera una nuova richiesta HTTP e un reload completo.

Il vantaggio della SPA è la navigazione **istantanea** dopo il primo caricamento. Lo svantaggio è che il bundle JavaScript iniziale è più pesante, e la SEO è più difficile senza SSR.

Split Mate non necessita di SEO (è uno strumento privato), quindi la scelta CSR puro è corretta per questo tipo di applicazione.

---

## 3. Stack tecnologico

Lo stack è diviso in due parti ben separate:

**Frontend — deployato su Vercel:**
- React 18 come libreria UI
- Vite come build tool (transpila JSX, minimizza, ottimizza)
- Tailwind CSS per lo stile

**Backend — deployato su Azure:**
- ASP.NET Core Web API con .NET 10
- Entity Framework Core come ORM
- SQLite come database (file `gestionespese.db`)

La comunicazione tra i due avviene **esclusivamente via HTTP/JSON** — il frontend non sa nulla di come il backend è implementato, e viceversa. Questa è un'**architettura disaccoppiata**.

---

## 4. Architettura — Pattern MVC

L'applicazione segue il **pattern MVC (Model-View-Controller)**:

| Componente | Responsabilità | In Split Mate |
|------------|----------------|---------------|
| **Model** | Gestisce i dati | Classi C# (`Spesa`, `Utente`, `Gruppo`) + EF Core |
| **View** | Mostra i dati | Componenti React (`.jsx`) |
| **Controller** | Coordina Model e View | Controller ASP.NET + `App.jsx` (routing condizionale) |

Il vantaggio del pattern MVC è la **separazione delle responsabilità**: se cambia il database, modifichi solo il Model senza toccare la View. Se cambia l'interfaccia, non tocchi il backend.

---

## 5. REST — Come comunica il sistema

Le API seguono il modello **REST a livello 2** (Richardson Maturity Model):
- Gli URL identificano le **risorse** (`/api/Spesa`, `/api/Gruppo`)
- I **verbi HTTP** definiscono l'azione (`GET` = leggi, `POST` = crea, `PUT` = aggiorna, `DELETE` = elimina)
- Ogni richiesta è **stateless** — il server non ricorda nulla tra una chiamata e l'altra

---

## 6. Deploy — Architettura cloud

In produzione i due servizi sono completamente indipendenti:

- **Vercel**: serve il frontend React come bundle statico su una **CDN globale** (Content Delivery Network). I file vengono serviti dal nodo più vicino all'utente — riduce il tempo di caricamento iniziale
- **Azure App Service (PaaS)**: ospita il backend ASP.NET Core. Microsoft gestisce sistema operativo, aggiornamenti e certificati TLS automaticamente

Entrambi i servizi espongono il loro endpoint su **HTTPS** con certificati TLS gestiti automaticamente (Vercel tramite Let's Encrypt, Azure tramite Microsoft).

> *"Potete vedere la demo live su gestore-spese-xi.vercel.app e le API documentate via Swagger all'endpoint Azure — vi mostreremo entrambi nella demo finale."*

---

## Possibili domande del professore

**"Perché SPA e non MPA?"**
> Per un'app interattiva con molti aggiornamenti di dati in tempo reale, la SPA è la scelta giusta. Una MPA ricaricare la pagina ad ogni operazione — pessima esperienza utente per uno strumento usato continuamente. La SEO non è un requisito per la nostra app.

**"Cos'è il disaccoppiamento frontend-backend?"**
> Frontend e backend sono due applicazioni separate che comunicano solo tramite API REST. Possiamo aggiornare il backend senza toccare React, e viceversa. Possiamo anche sostituire il backend con Node.js senza cambiare una riga di React.

**"Cos'è la CDN di Vercel?"**
> Una rete di server distribuiti geograficamente. Il bundle React viene servito dal nodo fisicamente più vicino all'utente, riducendo la latenza di download.
