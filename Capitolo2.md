# Capitolo 2 — Architettura Web, SPA, MPA e Pattern MVC

> Questo capitolo copre le architetture applicative web studiate con il Prof. Bonura: SPA, MPA, SSR, il pattern MVC e come Split Mate si posiziona in questo panorama.

---

## 2.1 Le Due Famiglie di Applicazioni Web

Esistono due grandi famiglie di applicazioni web, che si distinguono per **dove avviene il rendering** dell'HTML che l'utente vede.

### 2.1.1 MPA — Multi Page Application

Una **Multi Page Application** è un'applicazione composta da più pagine HTML separate. Ogni volta che l'utente clicca su un link o compie un'azione significativa, il browser invia una nuova richiesta HTTP al server, che genera e restituisce una **pagina HTML completa**.

```
Utente clicca "Vai alla pagina profilo"
        |
        | GET /profilo HTTP/1.1
        |------------------------------> Server
                                           |
                                       Genera HTML completo
                                       (PHP, ASP, ecc.)
                                           |
        <------------------------------ HTML + CSS + JS
        |
Browser riceve tutto e ridisegna la pagina da zero
```

**Esempi classici di MPA**: WordPress, Joomla, siti e-commerce tradizionali.

**Caratteristiche distintive:**

| Caratteristica | MPA |
|----------------|-----|
| Rendering | Lato server (SSR) |
| Cambio pagina | Nuova richiesta HTTP, reload completo |
| Stato | Reinizializzato ad ogni pagina |
| JavaScript | Poco, per animazioni o piccole interazioni |
| SEO | Ottimo (HTML già pronto per i crawler) |
| Prima apertura | Veloce (HTML pre-renderizzato) |
| Navigazione successiva | Più lenta (reload completo) |

### 2.1.2 SPA — Single Page Application

Una **Single Page Application** carica **una sola pagina HTML** al primo accesso. Da quel momento in poi, **JavaScript** gestisce tutto: aggiorna il contenuto, cambia la vista, comunica con il server — senza mai ricaricare la pagina.

```
Prima richiesta:
        | GET / HTTP/1.1
        |------------------------------> Server (Vercel)
        <------------------------------ index.html + bundle JS (React)

Navigazione successiva (es. apri un gruppo):
        | Nessuna richiesta HTTP per HTML!
        | React cambia il DOM direttamente
        |
        | GET /api/Gruppo/1  (solo i dati)
        |------------------------------> ASP.NET Core
        <------------------------------ JSON { ... }
        |
React aggiorna solo la parte della pagina che cambia
```

**Esempi classici di SPA**: Gmail, Twitter/X, Facebook, Google Maps.

**Caratteristiche distintive:**

| Caratteristica | SPA |
|----------------|-----|
| Rendering | Lato client (browser) |
| Cambio "pagina" | JavaScript aggiorna il DOM, nessun reload |
| Stato | Mantenuto in memoria durante tutta la sessione |
| JavaScript | Molto — è il motore dell'intera app |
| SEO | Più difficile (HTML vuoto al primo caricamento) |
| Prima apertura | Più lenta (deve scaricare tutto il JS) |
| Navigazione successiva | Istantanea |

### 2.1.3 Split Mate è una SPA

**Split Mate è costruita come SPA.** Il frontend React viene servito da Vercel come bundle statico (`index.html` + JS compilato da Vite). Il browser scarica questo bundle **una volta sola**, poi React gestisce tutta la navigazione: da "login" a "lista gruppi" a "dettaglio gruppo" senza mai ricaricare la pagina.

La comunicazione con il server avviene **esclusivamente tramite chiamate API** (`api.js`) che restituiscono JSON — mai HTML.

---

## 2.2 SSR — Server-Side Rendering

L'SSR è una via di mezzo: il server genera l'HTML della prima pagina (come una MPA), ma poi il JavaScript "prende vita" sul client e si comporta come una SPA per le navigazioni successive. Questo processo si chiama **idratazione del DOM** (*hydration*).

**Framework che usano SSR**: Next.js (React), Nuxt.js (Vue), SvelteKit.

**Split Mate non usa SSR.** Usiamo un approccio **CSR puro** (Client-Side Rendering): il server invia un `index.html` pressoché vuoto, e React costruisce l'intera interfaccia nel browser. Per un'app di gestione spese interna (non pubblica), la SEO non è prioritaria, quindi CSR è la scelta corretta.

---

## 2.3 AJAX — Il Motore della SPA

**AJAX** (*Asynchronous JavaScript And XML*) è la tecnica che ha reso possibili le SPA. Prima di AJAX, ogni interazione con il server richiedeva il reload completo della pagina. Con AJAX, JavaScript può fare richieste HTTP in background senza bloccare l'interfaccia.

> Storicamente AJAX usava XML, ma oggi si usa quasi esclusivamente **JSON**. Il nome è rimasto per ragioni storiche.

Nel nostro progetto **ogni funzione in `api.js` è AJAX**: `fetch()` invia la richiesta in background, `async/await` aspetta la risposta senza bloccare React, e `useState` aggiorna solo la parte di UI che deve cambiare.

```javascript
// api.js — esempio di chiamata AJAX in Split Mate
export async function getRiepilogoGruppo(gruppoId) {
  const res = await fetch(`${API_URL}/Riepilogo/gruppo/${gruppoId}`);
  // L'interfaccia non si blocca mentre aspettiamo la risposta
  return res.json();
}
```

---

## 2.4 Il Pattern MVC — Model-View-Controller

**MVC** è un pattern architetturale che divide l'applicazione in tre componenti con responsabilità distinte, seguendo il principio SRP (Single Responsibility Principle).

```
         Utente
           |
           | interagisce
           v
        CONTROLLER
        (dirige il traffico)
           |           |
           |           |
           v           v
         MODEL       VIEW
       (i dati)   (l'interfaccia)
```

### 2.4.1 Le Tre Responsabilità

| Componente | Responsabilità | In Split Mate (Backend) | In Split Mate (Frontend) |
|------------|----------------|------------------------|--------------------------|
| **Model** | Gestire e rappresentare i dati | Classi `Spesa.cs`, `Utente.cs`, `Gruppo.cs` + EF Core | Stato React (`useState`) |
| **View** | Mostrare i dati all'utente | (delegato al frontend React) | Componenti `.jsx` (JSX) |
| **Controller** | Ricevere input, coordinare Model e View | `SpesaController.cs`, `AuthController.cs` | `App.jsx` (routing condizionale) |

### 2.4.2 Il Flusso MVC in una Richiesta Reale

Esempio: l'utente aggiunge una spesa nel gruppo "Vacanza".

```
1. UTENTE compila il form e clicca "Aggiungi"
        |
2. CONTROLLER (React - App.jsx / ModalNuovaSpesa.jsx)
   riceve l'evento onClick, chiama api.js
        |
        | POST /api/Spesa  (JSON body)
        v
3. CONTROLLER (ASP.NET - SpesaController.cs)
   riceve la richiesta, valida i dati
        |
4. MODEL (Entity Framework Core + SQLite)
   salva la nuova Spesa e le DivisioneSpesa
        |
5. CONTROLLER restituisce 201 Created + JSON spesa
        |
6. VIEW (React component)
   riceve il JSON, aggiorna useState, ri-renderizza la lista
```

### 2.4.3 Perché il Pattern MVC è Importante

- **Manutenibilità**: se cambia il database, modifichi solo il Model senza toccare la View
- **Testabilità**: puoi testare Controller e Model indipendentemente dalla UI
- **Separazione delle competenze**: i designer lavorano sulla View, i backend developer sul Model
- **Riutilizzo**: lo stesso Model può servire più View (es. la stessa API serve sia il frontend web che un'eventuale app mobile)

---

## 2.5 REST — REpresentational State Transfer

**REST** è un insieme di principi (non uno standard rigido) per progettare API web. È il modello che Split Mate adotta per la comunicazione frontend-backend.

### 2.5.1 I Principi REST

| Principio | Descrizione | Come lo rispettiamo |
|-----------|-------------|---------------------|
| **Stateless** | Ogni richiesta è autosufficiente, il server non mantiene stato | Nessuna sessione server-side, i dati utente viaggiano in ogni richiesta |
| **Interfaccia uniforme** | URL identificano risorse, verbi HTTP definiscono azioni | `/api/Spesa/3` con `GET`/`PUT`/`DELETE` |
| **Client-Server** | Frontend e backend separati e indipendenti | React su Vercel, ASP.NET su Azure |
| **Cacheability** | Le risposte devono dichiarare se sono cacheable | Header standard HTTP |
| **Layered System** | Il client non sa se parla direttamente col server finale | CDN, proxy trasparenti |

### 2.5.2 Richardson REST Maturity Model

Martin Fowler ha definito 4 livelli di maturità REST:

| Livello | Nome | Caratteristica | Esempio |
|---------|------|----------------|---------|
| **0** | The Swamp of POX | Un solo endpoint per tutto | `POST /api` con azione nel body |
| **1** | Resources | URL separati per ogni risorsa | `POST /api/Spesa`, `GET /api/Gruppo` |
| **2** | HTTP Verbs | Usa i verbi HTTP correttamente | `GET /api/Spesa/3`, `DELETE /api/Spesa/3` |
| **3** | Hypermedia (HATEOAS) | Le risposte contengono link alle azioni possibili | Livello più avanzato, raro in pratica |

> **Split Mate si trova al Livello 2** — la maggior parte delle API RESTful reali si ferma qui. Usiamo URL semantici per le risorse e verbi HTTP per le azioni.

### 2.5.3 Design degli Endpoint in Split Mate

Gli endpoint seguono la convenzione REST: la risorsa è nel path, l'azione è il verbo HTTP.

```
# Autenticazione
POST   /api/Auth/login               → login / auto-provisioning
GET    /api/Auth/exists?email=...    → verifica esistenza email

# Gruppi
GET    /api/Gruppo/utente/{id}       → gruppi di un utente
POST   /api/Gruppo                   → crea gruppo
DELETE /api/Gruppo/{id}              → elimina gruppo
POST   /api/Gruppo/{id}/membri       → aggiungi membro
DELETE /api/Gruppo/{id}/membri/{uid} → rimuovi membro

# Spese
GET    /api/Spesa/gruppo/{id}        → spese di un gruppo
POST   /api/Spesa                    → crea spesa
PUT    /api/Spesa/{id}               → modifica spesa
DELETE /api/Spesa/{id}              → elimina spesa

# Riepilogo
GET    /api/Riepilogo/gruppo/{id}    → debiti di un gruppo
PUT    /api/Riepilogo/{id}/Saldato   → segna debito come saldato
```

---

## 2.6 Architettura Disaccoppiata (Decoupled Architecture)

Split Mate adotta un'**architettura disaccoppiata**: frontend e backend sono due applicazioni **completamente separate** che comunicano solo tramite API.

```
+-------------------------+       HTTP/JSON        +-------------------------+
|   FRONTEND              | ---------------------->|   BACKEND               |
|   React + Vite          |                        |   ASP.NET Core          |
|   Deployato su Vercel   | <----------------------|   Deployato su Azure    |
+-------------------------+                        +------------+------------+
                                                                |
                                                        Entity Framework Core
                                                                |
                                                   +------------+------------+
                                                   |   SQLite Database       |
                                                   |   gestionespese.db      |
                                                   +-------------------------+
```

**Vantaggi di questa scelta:**

- **Deploy indipendente**: puoi aggiornare il frontend senza toccare il backend e viceversa
- **Scalabilità**: frontend e backend scalano indipendentemente
- **Tecnologia libera**: il backend potrebbe essere riscritto in Node.js senza cambiare una riga di React
- **Team separati**: frontend e backend developer lavorano in parallelo

**Svantaggi:**

- **CORS**: comunicazione cross-origin richiede configurazione esplicita
- **Complessità**: due deploy, due ambienti, due set di configurazioni
- **SEO**: senza SSR, i motori di ricerca faticano a indicizzare il contenuto

---

## 2.7 Confronto SPA vs MPA — Quale Scegliere?

| Criterio | SPA (Split Mate) | MPA |
|----------|-----------------|-----|
| **Tipo di app** | Dashboard, tool interattivi, social | Blog, e-commerce, siti di contenuto |
| **Interattività** | Alta — molti aggiornamenti in tempo reale | Bassa — principalmente lettura |
| **SEO** | Difficile senza SSR | Ottimale |
| **Performance iniziale** | Più lenta (bundle JS) | Più veloce |
| **Navigazione** | Istantanea dopo il caricamento | Un reload per ogni pagina |
| **Stato utente** | Mantenuto in memoria | Perso ad ogni navigazione |
| **Complessità frontend** | Alta (gestione stato, routing client-side) | Bassa |

> **Regola pratica**: usa una SPA per applicazioni con **molta interattività** e feedback in tempo reale (come Split Mate). Usa una MPA per contenuti **statici o quasi statici** (blog, vetrina aziendale).

---

## 2.8 Web Vitals — Misurare la Performance

Google misura la qualità dell'esperienza utente con i **Web Vitals**, metriche standard che influenzano anche il ranking SEO:

| Metrica | Significato | Obiettivo | Impatto su Split Mate |
|---------|-------------|-----------|----------------------|
| **LCP** — Largest Contentful Paint | Tempo per renderizzare il contenuto principale | < 2.5s | Il bundle React deve essere piccolo e ottimizzato |
| **FID** — First Input Delay | Reattività al primo click dell'utente | < 100ms | Il JS non deve bloccare il thread principale |
| **CLS** — Cumulative Layout Shift | Stabilità visiva (elementi che saltano) | < 0.1 | Evitare layout shifts durante il caricamento dati |
| **TTFB** — Time to First Byte | Tempo di risposta del server | < 800ms | Il cold start di Azure influenza negativamente questo valore |

---

## Riepilogo Capitolo 2

| Concetto | Come si manifesta in Split Mate |
|----------|---------------------------------|
| SPA | React su Vercel, nessun reload di pagina, routing gestito da JavaScript |
| MPA | Non usata — confronto utile per l'esame |
| SSR | Non usata (CSR puro) — possibile evoluzione futura con Next.js |
| AJAX | Ogni `fetch()` in `api.js` è una chiamata AJAX asincrona |
| Pattern MVC | Controller C# + Model EF Core + View React |
| REST Livello 2 | URL semantici per le risorse + verbi HTTP corretti per le azioni |
| Architettura Disaccoppiata | Frontend Vercel + Backend Azure, comunicazione solo tramite JSON |
