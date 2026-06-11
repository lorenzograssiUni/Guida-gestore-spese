# Capitolo 1 — Protocolli e Comunicazione HTTP

> Questo capitolo copre tutta la teoria sui protocolli di rete spiegata dal Prof. Bonura, applicata concretamente al funzionamento di Split Mate.

---

## 1.1 Il Modello Request-Response

HTTP (*HyperText Transfer Protocol*) è il protocollo applicativo su cui si basa l'intera comunicazione di Split Mate. È un protocollo **stateless** (senza stato): ogni richiesta è indipendente dalle precedenti, il server non ricorda chi sei tra una chiamata e l'altra.

Il modello fondamentale è **Request → Response**:

```
Browser/React (Client)           ASP.NET Core (Server)
       |                                  |
       |  POST /api/Auth/login            |
       |  Content-Type: application/json  |
       |  Body: { email, password }       |
       | -------------------------------->|
       |                                  |  elabora la richiesta
       |                                  |  verifica credenziali
       |                                  |  interroga SQLite
       |  200 OK                          |
       |  Body: { id, nome, gruppi }      |
       |<---------------------------------|
```

Nel nostro progetto **ogni interazione dell'utente** (login, aggiunta spesa, visualizzazione gruppo) genera una o più coppie Request-Response tra React (client) e ASP.NET Core (server).

---

## 1.2 Struttura di una Richiesta HTTP

Una richiesta HTTP è composta da tre parti principali: metodo, URL, headers e (opzionalmente) body.

### 1.2.1 Metodo (Verbo)

Il metodo indica l'**azione** da compiere sulla risorsa. REST usa i verbi HTTP in modo semantico:

| Metodo | Significato | Uso in Split Mate | Esempio endpoint |
|--------|-------------|-------------------|------------------|
| `GET` | Leggi una risorsa | Carica gruppi, spese, riepilogo | `GET /api/Gruppo/utente/5` |
| `POST` | Crea una nuova risorsa | Crea gruppo, aggiungi spesa, login | `POST /api/Spesa` |
| `PUT` | Aggiorna una risorsa esistente | Modifica spesa, salda debito | `PUT /api/Spesa/3` |
| `DELETE` | Elimina una risorsa | Elimina spesa, rimuovi membro | `DELETE /api/Spesa/3` |

> **Domanda da esame**: *Qual è la differenza tra POST e PUT?*
> POST crea una **nuova** risorsa (l'ID lo decide il server). PUT **sostituisce** una risorsa esistente identificata dall'ID nell'URL.

### 1.2.2 URL (Uniform Resource Locator)

La struttura di un URL nel nostro progetto:

```
https://splitmate-api.azurewebsites.net/api/Spesa/3
  |               |                     |    |    |
  schema          host (Azure)       prefisso risorsa  ID
```

Nel file `api.js` tutti gli URL vengono costruiti concatenando `VITE_API_URL` con il percorso della risorsa. La variabile d'ambiente evita di scrivere l'URL hardcoded nel codice.

### 1.2.3 Headers

Gli header sono **metadati** allegati alla richiesta o risposta. I più importanti nel nostro progetto:

**Header di richiesta (inviati da React):**

| Header | Valore | Scopo |
|--------|--------|-------|
| `Content-Type` | `application/json` | Dice al server che il body è JSON |
| `Accept` | `application/json` | Dice al server che vogliamo JSON in risposta |

**Header di risposta (inviati da ASP.NET):**

| Header | Esempio | Scopo |
|--------|---------|-------|
| `Content-Type` | `application/json; charset=utf-8` | Formato della risposta |
| `Access-Control-Allow-Origin` | `*` | Permesso CORS |

### 1.2.4 Body

Presente solo nelle richieste `POST` e `PUT`. Contiene i dati serializzati in JSON. Esempio di body per la creazione di una spesa:

```json
{
  "Gruppo_ID": 1,
  "ChiPaga_ID": 3,
  "Importo": 45.00,
  "Descrizione": "Cena al ristorante",
  "UtentiCoinvoltiIds": [3, 5, 7]
}
```

Questo JSON viene deserializzato da ASP.NET Core nel `NuovaSpesaDTO` grazie alla configurazione `AddControllersWithViews().AddJsonOptions(...)` in `Program.cs`.

---

## 1.3 Status Code — I Codici di Risposta

Ogni risposta HTTP include un codice numerico a 3 cifre che indica l'esito. Il primo digit indica la categoria:

| Famiglia | Range | Significato | Esempi nel progetto |
|----------|-------|-------------|---------------------|
| **2xx** | 200-299 | Successo | `200 OK`, `201 Created`, `204 No Content` |
| **3xx** | 300-399 | Reindirizzamento | `301 Moved Permanently` |
| **4xx** | 400-499 | Errore del client | `400 Bad Request`, `401 Unauthorized`, `404 Not Found` |
| **5xx** | 500-599 | Errore del server | `500 Internal Server Error` |

**Status code che il nostro backend restituisce:**

```csharp
// AuthController.cs
return Ok(utente);           // 200 OK - login riuscito
return Unauthorized();       // 401 - password errata
return BadRequest("...");    // 400 - email vuota

// GruppoController.cs
return NotFound();           // 404 - gruppo non trovato
return CreatedAtAction(...); // 201 - gruppo creato
return NoContent();          // 204 - DELETE riuscita, niente da restituire
```

> **Domanda da esame**: *Cosa restituisce il tuo backend se la password è sbagliata?*
> `401 Unauthorized` - il client sa che deve richiedere le credenziali all'utente.

---

## 1.4 Cookie e Gestione della Sessione

I cookie sono piccoli file di dati che il server invia al client, che li rimanda in ogni richiesta successiva.

| Tipo | Caratteristica |
|------|---------------|
| **Session Cookie** | Scade alla chiusura del browser |
| **Persistent Cookie** | Ha una data di scadenza esplicita |
| **Secure Cookie** | Trasmesso solo su HTTPS |
| **HttpOnly Cookie** | Non accessibile da JavaScript (protezione XSS) |

**Nel nostro progetto non usiamo cookie.** Abbiamo scelto il `localStorage` per persistere i dati dell'utente autenticato. Questo è un trade-off:
- `localStorage` è accessibile da JavaScript (teoricamente meno sicuro contro XSS)
- Ma è molto più semplice da gestire in una SPA senza logica server-side di sessione
- Non abbiamo JWT o OAuth, quindi non c'è un token vero e proprio da proteggere

---

## 1.5 Il Percorso Completo di una Richiesta

Quando un utente clicca "Aggiungi Spesa" in Split Mate, ecco cosa succede esattamente, passo per passo:

### Step 1 - Risoluzione DNS

Il browser deve conoscere l'IP di `splitmate-api.azurewebsites.net`:
1. Controlla la **cache DNS locale** (sistema operativo + browser)
2. Se non c'è, chiede al **resolver DNS** (es. Google DNS `8.8.8.8`)
3. Il resolver interroga i nameserver autoritativi in modo ricorsivo
4. Ottiene l'IP e lo memorizza per il valore del **TTL** (*Time to Live*)

> Il TTL è un valore in secondi che indica per quanto tempo un record DNS può essere cachato. Senza cache DNS, ogni richiesta aprirebbe un socket verso il DNS, consumando più batteria e tempo.

### Step 2 - TCP Three-Way Handshake

Stabilisce la connessione tra client e server:

```
Client          Server
  |--- SYN ----->|   "voglio connettermi"
  |<-- SYN-ACK --|   "ok, confermo"
  |--- ACK ----->|   "connessione stabilita"
```

### Step 3 - TLS Handshake (solo HTTPS)

Poiché usiamo HTTPS (porta 443), dopo il TCP handshake avviene la negoziazione TLS per cifrare la connessione. Aggiunge latenza ma garantisce sicurezza.

> HTTPS = HTTP + TLS. TLS è la connessione sicura su TCP. SSL è il predecessore deprecato di TLS.

### Step 4 - Invio della Richiesta HTTP

React invia la richiesta tramite `fetch()`:

```javascript
// api.js - esempio reale del progetto
export async function creaSpesa(dati) {
  const res = await fetch(`${API_URL}/Spesa`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(dati)
  });
  return res.json();
}
```

### Step 5 - Elaborazione Server

ASP.NET Core riceve la richiesta, il middleware la instrada al `SpesaController`, che:
1. Deserializza il JSON nel `NuovaSpesaDTO`
2. Valida i dati (gruppo esiste? pagatore esiste?)
3. Calcola le divisioni per ogni membro coinvolto
4. Salva su SQLite tramite Entity Framework Core
5. Restituisce `201 Created` con il corpo della spesa creata

### Step 6 - Ricezione e Aggiornamento UI

React riceve la risposta, aggiorna lo stato con `useState` e ri-renderizza il componente mostrando la nuova spesa nella lista, senza ricaricare la pagina (questo è il vantaggio della SPA).

---

## 1.6 HTTP/2 e HTTP/3 - Evoluzione del Protocollo

### HTTP/1.1 - I Limiti

HTTP/1.1 ha alcune limitazioni che impattano le performance:
- Ogni richiesta richiedeva una nuova connessione TCP (overhead)
- Le richieste sono **sequenziali**: una richiesta lenta blocca le successive (*head-of-line blocking*)
- Chrome limita a **6 connessioni TCP simultanee per dominio**
- Gli header vengono inviati per intero ad ogni richiesta (ridondanti)

### HTTP/2 - Multiplexing

Standardizzato nel 2015, HTTP/2 introduce il **multiplexing**: più richieste e risposte viaggiano sulla **stessa connessione TCP** come frame interlacciati. Risolve il head-of-line blocking a livello HTTP. Introduce anche la compressione degli header (HPACK) e il Server Push.

### HTTP/3 - QUIC

HTTP/3 abbandona TCP e si basa su **QUIC** (*Quick UDP Internet Connections*, sviluppato da Google su UDP):

| Caratteristica | HTTP/2 (TCP) | HTTP/3 (QUIC/UDP) |
|---|---|---|
| Protocollo di trasporto | TCP | UDP |
| Handshake | Multiple round-trip | 1-RTT (molto più veloce) |
| Head-of-line blocking | Persiste a livello TCP | Risolto per ogni stream |
| Crittografia | TLS separato | TLS 1.3 integrato in QUIC |
| Resilienza perdita pacchetti | Un pacchetto blocca tutto | Blocca solo lo stream interessato |

> Azure App Service (dove è deployato Split Mate) supporta HTTP/2. La comunicazione tra React su Vercel e il backend su Azure avviene già in HTTP/2.

---

## 1.7 WebSocket - Comunicazione Bidirezionale (Cenno)

HTTP è **monodirezionale**: il client chiede, il server risponde. Per aggiornamenti in tempo reale (es. "un membro ha appena aggiunto una spesa") HTTP classico non basta.

**WebSocket** risolve questo:
1. Il client apre una connessione HTTP normale con header `Upgrade: websocket`
2. Il server accetta il cambio di protocollo
3. La connessione TCP rimane aperta e **entrambi possono inviare messaggi in qualsiasi momento**

**Nel nostro progetto non usiamo WebSocket.** Le spese vengono aggiornate solo al reload del componente. Come evoluzione futura si potrebbe integrare **SignalR** (la libreria .NET per WebSocket) per notifiche in tempo reale.

---

## 1.8 CORS - Cross-Origin Resource Sharing

### Il Problema

Il browser impone la **Same-Origin Policy**: una pagina può fare richieste solo al suo stesso dominio (stesso schema + host + porta). Il nostro frontend è su `https://split-mate.vercel.app` e il backend su `https://splitmate-api.azurewebsites.net` - **domini diversi**. Il browser blocca tutte le richieste di default.

### Come Funziona CORS

1. Il browser invia una **Preflight Request** (`OPTIONS`) per chiedere il permesso
2. Il server risponde con gli header che dichiarano cosa è permesso
3. Il browser decide se permettere la richiesta reale

### La Nostra Configurazione in Program.cs

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()    // qualsiasi dominio
              .AllowAnyHeader()    // qualsiasi header
              .AllowAnyMethod();   // GET, POST, PUT, DELETE...
    });
});

// IMPORTANTE: va chiamato prima di UseRouting e MapControllers
app.UseCors("AllowAll");
```

> **Domanda da esame**: *La tua politica CORS è sicura per la produzione?*
> No, `AllowAnyOrigin()` è troppo permissiva. In produzione si dovrebbe usare `.WithOrigins("https://split-mate.vercel.app")` per accettare solo il dominio del nostro frontend, applicando il principio del privilegio minimo.

---

## 1.9 Timing HTTP - Performance

Il timing misura quanto impiega ogni fase di una richiesta HTTP:

| Fase | Descrizione | Ottimizzazione possibile |
|------|-------------|-------------------------|
| **DNS Lookup** | Risoluzione dominio in IP | Cache DNS, TTL appropriato |
| **TCP Handshake** | Apertura connessione TCP | HTTP/2 (connessione riutilizzata) |
| **TLS Handshake** | Negoziazione HTTPS | HTTP/3 QUIC (1-RTT) |
| **TTFB** | Time to First Byte - attesa risposta | Query DB ottimizzate, caching |
| **Content Download** | Download del contenuto | Compressione Gzip, CDN |

Nel nostro progetto il bottleneck principale è il **TTFB**: Azure App Service sul piano gratuito ha un **"cold start"** - se il server è inattivo da un po', la prima richiesta impiega diversi secondi per risvegliarlo.

---

## Riepilogo Capitolo 1

| Concetto | Dove si vede in Split Mate |
|----------|---------------------------|
| HTTP Request-Response | Ogni chiamata in `api.js` verso ASP.NET Core |
| Metodi GET/POST/PUT/DELETE | Attributi `[HttpGet]`, `[HttpPost]` nei controller |
| Status Code 200/201/400/401/404 | Return dei controller (`Ok`, `NotFound`, `Unauthorized`) |
| Headers Content-Type | `application/json` in tutte le chiamate |
| CORS | Policy `AllowAll` configurata in `Program.cs` |
| HTTPS | Comunicazione cifrata tra Vercel e Azure |
| Stateless | Nessuna sessione server-side, stato salvato in `localStorage` |
| HTTP/2 | Usato automaticamente da Azure App Service |

---

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

---

# Capitolo 3 — REST e API Design

> Questo capitolo approfondisce i principi REST, il Richardson Maturity Model, e documenta in dettaglio tutti gli endpoint del progetto Split Mate con la loro logica di business.

---

## 3.1 Cos'è REST — Roy Fielding e i 6 Vincoli

### La Storia

**REST** (REpresentational State Transfer) è uno stile architetturale definito da **Roy Fielding** nella sua dissertazione dottorale all'Università della California nel **2000**. Fielding era uno degli autori principali delle specifiche HTTP/1.0 e HTTP/1.1, quindi REST nasce come una riflessione su come usare al meglio le caratteristiche già presenti nel protocollo HTTP.

> REST non è un protocollo, non è uno standard, non è una libreria. È un **insieme di vincoli architetturali**. Un sistema che rispetta tutti i vincoli si dice **RESTful**.

### I 6 Vincoli di REST

#### 1. Client-Server

Il sistema è diviso in due ruoli distinti con una separazione netta delle responsabilità:
- Il **client** gestisce l'interfaccia utente e l'esperienza
- Il **server** gestisce i dati e la logica di business

Questi due ruoli evolvono indipendentemente: puoi riscrivere il frontend senza toccare il backend, e viceversa.

**In Split Mate**: React (client) su Vercel e ASP.NET Core (server) su Azure sono completamente separati. Comunicano solo tramite HTTP/JSON.

#### 2. Stateless (Senza Stato)

Ogni richiesta al server deve contenere **tutte le informazioni necessarie** per essere elaborata. Il server non mantiene memoria delle richieste precedenti.

Questo significa:
- Nessuna sessione server-side (no session ID in memoria)
- Ogni richiesta è autosufficiente
- Il server può scalare orizzontalmente (qualsiasi istanza può gestire qualsiasi richiesta)

**In Split Mate**: non usiamo sessioni server-side. Quando React chiama `GET /api/Gruppo/utente/5`, il numero `5` è l'ID utente che deve essere passato esplicitamente — il server non lo ricorda da una chiamata precedente.

> **Attenzione**: stateless non significa che i dati non vengono salvati. I dati persistono nel database. È lo **stato della sessione** che non viene mantenuto.

#### 3. Cacheable

Le risposte devono dichiarare esplicitamente se possono essere memorizzate nella cache. Una risposta cacheable può essere riutilizzata da client e proxy intermedi, riducendo il carico sul server.

**In Split Mate**: non gestiamo esplicitamente la cache (le risposte non hanno header `Cache-Control` configurati manualmente), ma il browser applica comunque le policy di default.

#### 4. Uniform Interface (Interfaccia Uniforme)

Questo è il vincolo più caratteristico di REST. Si compone di 4 sotto-vincoli:

| Sotto-vincolo | Significato | In Split Mate |
|---------------|-------------|---------------|
| **Identificazione delle risorse** | Ogni risorsa ha un URI univoco | `/api/Spesa/3` identifica la spesa con ID 3 |
| **Manipolazione tramite rappresentazioni** | Il client modifica le risorse inviando una rappresentazione (JSON) | Il body JSON contiene i campi da aggiornare |
| **Messaggi auto-descrittivi** | Ogni messaggio contiene abbastanza info per essere elaborato | Header `Content-Type: application/json` |
| **HATEOAS** | Le risposte contengono link alle azioni possibili | **Non implementato** (siamo al Livello 2) |

#### 5. Layered System (Sistema a Livelli)

Il client non sa se sta comunicando direttamente con il server finale o con un intermediario (proxy, load balancer, CDN). Ogni livello vede solo il livello adiacente.

**In Split Mate**: il browser non sa che la richiesta al frontend passa per la CDN di Vercel, né che il backend è dietro il reverse proxy di Azure App Service.

#### 6. Code on Demand (Opzionale)

Il server può inviare codice eseguibile al client (es. JavaScript). È l'unico vincolo **opzionale** di REST.

**In Split Mate**: tecnicamente Vercel invia il bundle JavaScript di React al browser, ma questo è un pattern standard delle SPA, non un uso particolare di REST.

---

## 3.2 Richardson REST Maturity Model

Leonard Richardson (2008) ha proposto un modello per misurare quanto un'API è "matura" in termini di aderenza ai principi REST, organizzato in **4 livelli** (0-3).

### Livello 0 — The Swamp of POX

Un solo endpoint che accetta tutto tramite POST. L'azione è descritta nel body della richiesta.

```http
# Esempio Livello 0
POST /api
Body: { "azione": "getSpesa", "id": 3 }

POST /api
Body: { "azione": "deleteSpesa", "id": 3 }
```

Non si usa HTTP come strumento, solo come tunnel. Tipico dei vecchi SOAP/XML-RPC.

### Livello 1 — Resources

Si introducono URL diversi per risorse diverse, ma si usa ancora un solo verbo HTTP per tutto.

```http
# Esempio Livello 1
POST /api/Spesa        # crea
POST /api/Spesa/3      # leggi o modifica o cancella, dipende dal body
POST /api/Gruppo       # tutto con POST
```

Migliore del Livello 0, ma i verbi HTTP non vengono usati semanticamente.

### Livello 2 — HTTP Verbs

Si usano correttamente i verbi HTTP e gli status code. È il livello de facto delle API moderne.

```http
# Esempio Livello 2 (quello che facciamo in Split Mate)
GET    /api/Spesa/3     # leggi
PUT    /api/Spesa/3     # modifica
DELETE /api/Spesa/3    # cancella
POST   /api/Spesa      # crea

# Con status code semantici
200 OK          # lettura riuscita
201 Created     # creazione riuscita
204 No Content  # delete riuscita
404 Not Found   # risorsa non trovata
```

### Livello 3 — Hypermedia Controls (HATEOAS)

Le risposte includono link alle azioni successive possibili. Il client non deve conoscere a priori gli URL dell'API.

```json
// Esempio Livello 3 — risposta con HATEOAS
{
  "id": 3,
  "descrizione": "Cena",
  "importo": 45.00,
  "_links": {
    "self": { "href": "/api/Spesa/3" },
    "update": { "href": "/api/Spesa/3", "method": "PUT" },
    "delete": { "href": "/api/Spesa/3", "method": "DELETE" },
    "gruppo": { "href": "/api/Gruppo/1" }
  }
}
```

Raro in pratica: richiede molto lavoro e i client in genere hardcodano gli URL comunque.

### Dove Si Posiziona Split Mate

**Split Mate è al Livello 2**, che è il livello standard nell'industria. Usiamo:
- URL semantici per le risorse (`/api/Spesa/{id}`, `/api/Gruppo/{id}`)
- Verbi HTTP con significato corretto (GET legge, POST crea, PUT aggiorna, DELETE elimina)
- Status code appropriati (200, 201, 204, 400, 401, 404)

Non implementiamo HATEOAS (Livello 3) perché aggiunge complessità senza benefici concreti per un progetto di questa scala.

---

## 3.3 I Nostri Endpoint Documentati

### AuthController — `/api/Auth`

| Metodo | Endpoint | Body Richiesta | Risposta | Descrizione |
|--------|----------|----------------|----------|-------------|
| `POST` | `/api/Auth/login` | `{ email, password }` | `200 UtenteDto` / `401` | Login o auto-provisioning |
| `GET` | `/api/Auth/exists` | `?email=...` (query param) | `200 { exists: bool }` | Verifica se l'email è registrata |

**Logica speciale**: `POST /api/Auth/login` implementa l'**Auto-Provisioning**. Se l'email non esiste nel database, l'utente viene creato automaticamente (registrazione implicita). Se esiste, verifica la password con BCrypt.

### GruppoController — `/api/Gruppo`

| Metodo | Endpoint | Body / Params | Risposta | Descrizione |
|--------|----------|--------------|----------|-------------|
| `GET` | `/api/Gruppo/utente/{utenteId}` | — | `200 List<GruppoDto>` | Gruppi dell'utente |
| `POST` | `/api/Gruppo` | `{ nome, descrizione, creatoDA_ID }` | `201 GruppoDto` | Crea gruppo |
| `DELETE` | `/api/Gruppo/{id}` | — | `204` / `404` | Elimina gruppo |
| `GET` | `/api/Gruppo/codice/{codice}` | — | `200 GruppoDto` / `404` | Trova per codice invito |
| `POST` | `/api/Gruppo/{id}/membri` | `{ utenteId }` | `200` / `404` | Aggiungi membro |
| `DELETE` | `/api/Gruppo/{gruppoId}/membri/{utenteId}` | — | `204` / `400` | Rimuovi membro |
| `POST` | `/api/Gruppo/{id}/bot` | `{ nome }` | `201 UtenteDto` | Crea utente bot |

### SpesaController — `/api/Spesa`

| Metodo | Endpoint | Body / Params | Risposta | Descrizione |
|--------|----------|--------------|----------|-------------|
| `GET` | `/api/Spesa/gruppo/{gruppoId}` | — | `200 List<SpesaDto>` | Spese del gruppo |
| `POST` | `/api/Spesa` | `NuovaSpesaDto` | `201 SpesaDto` | Crea spesa con divisioni |
| `PUT` | `/api/Spesa/{id}` | `ModificaSpesaDto` | `200 SpesaDto` / `404` | Modifica spesa |
| `DELETE` | `/api/Spesa/{id}` | — | `204` / `404` | Elimina spesa |

### UtenteController — `/api/Utente`

| Metodo | Endpoint | Body / Params | Risposta | Descrizione |
|--------|----------|--------------|----------|-------------|
| `PUT` | `/api/Utente/{id}/nome` | `{ nuovoNome }` | `200` / `404` | Aggiorna nome utente |
| `GET` | `/api/Utente/gruppo/{gruppoId}` | — | `200 List<UtenteDto>` | Utenti di un gruppo |

### RiepilogoController — `/api/Riepilogo`

| Metodo | Endpoint | Body / Params | Risposta | Descrizione |
|--------|----------|--------------|----------|-------------|
| `GET` | `/api/Riepilogo/gruppo/{gruppoId}` | — | `200 List<RiepilogoDto>` | Calcola debiti del gruppo |
| `PUT` | `/api/Riepilogo/{id}/Saldato` | — | `200` / `404` | Marca debito come saldato |

---

## 3.4 Swagger — Documentazione Interattiva

**Swagger** (basato su **OpenAPI Specification**) è uno strumento che genera automaticamente documentazione interattiva per le API REST.

### Come Funziona in ASP.NET Core

ASP.NET Core con `Swashbuckle.AspNetCore` legge i **metadati dei controller** (attributi `[HttpGet]`, `[HttpPost]`, tipi di ritorno, parametri) e genera automaticamente:
1. Un file JSON di specifica OpenAPI (`/swagger/v1/swagger.json`)
2. Un'interfaccia web interattiva (`/swagger`)

### Configurazione in Program.cs

```csharp
// Registrazione del servizio
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen();

// Abilitazione nella pipeline (solo in development)
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI();
}
```

> **Nel nostro progetto** Swagger è abilitato anche in production (Azure), così è possibile testare le API live all'indirizzo:
> `https://gestione-spese-hbhga0crf6hsagdn.swedencentral-01.azurewebsites.net/swagger`

### Vantaggi di Swagger nel Progetto

- **Durante lo sviluppo**: testare ogni endpoint senza scrivere codice frontend
- **Documentazione sempre aggiornata**: riflette automaticamente il codice reale
- **Contratto API**: il frontend sa esattamente quali endpoint esistono e quali parametri accettano
- **Demo all'esame**: mostrare le API funzionanti in tempo reale

---

## 3.5 DTO — Data Transfer Object

### Cos'è un DTO

Un **Data Transfer Object** è un oggetto usato esclusivamente per **trasportare dati** tra il client e il server (o tra layer applicativi). Non contiene logica di business, solo proprietà.

Il DTO è **diverso** dal Model del database:

```
Model (Entity)          DTO
--------------          ---
Spesa.cs                SpesaDto.cs
  Id                      Id
  Descrizione             Descrizione
  Importo                 Importo
  GruppoId (FK)           NomeGruppo  ← campo calcolato/aggregato
  ChiPagaId (FK)          NomePagatore ← join già risolto
  PasswordHash  ← MAI nel DTO! (campo sensibile)
```

### Perché Usare i DTO

| Motivo | Spiegazione |
|--------|-------------|
| **Sicurezza** | Evitare di esporre campi sensibili (es. `PasswordHash`) |
| **Flessibilità** | Restituire strutture diverse da quelle del database |
| **Disaccoppiamento** | Il client non dipende dalla struttura interna del DB |
| **Versioning** | Puoi cambiare il Model senza rompere i client |
| **Performance** | Trasferire solo i dati necessari, non l'intera entità |

### I DTO in Split Mate

```csharp
// LoginDto — riceve le credenziali
public class LoginDto
{
    public string Email { get; set; }
    public string Password { get; set; }  // password in chiaro (solo in arrivo)
}

// UtenteDto — restituisce i dati utente (senza hash!)
public class UtenteDto
{
    public int Id { get; set; }
    public string Nome { get; set; }
    public string Email { get; set; }
    public bool IsBot { get; set; }
    // PasswordHash NON è incluso!
}

// NuovaSpesaDto — riceve i dati per creare una spesa
public class NuovaSpesaDto
{
    public string Descrizione { get; set; }
    public decimal Importo { get; set; }
    public int GruppoId { get; set; }
    public int ChiPagaId { get; set; }
    public List<int> UtentiCoinvoltiIds { get; set; }
}
```

### Il Flusso con i DTO

```
React invia JSON
       |
       | POST /api/Spesa
       | Body: { descrizione, importo, gruppoId, chiPagaId, utentiIds }
       v
ASP.NET Core deserializza in NuovaSpesaDto
       |
       v
Controller usa NuovaSpesaDto per creare entità Spesa
       |
       v
EF Core salva l'entità Spesa nel database
       |
       v
Controller crea SpesaDto dalla Spesa salvata
       |
       v
ASP.NET Core serializza SpesaDto in JSON
       |
       | Response: 201 Created
       | Body: { id, descrizione, importo, nomePagatore, ... }
       v
React riceve il DTO (senza dati interni del DB)
```

---

## Riepilogo Capitolo 3

| Concetto | Come si manifesta in Split Mate |
|----------|---------------------------------|
| REST (Roy Fielding, 2000) | 6 vincoli: Client-Server, Stateless, Cacheable, Uniform Interface, Layered, Code on Demand |
| Richardson Livello 2 | URL semantici + verbi HTTP + status code corretti |
| Endpoint documentati | 5 controller, ~15 endpoint totali con GET/POST/PUT/DELETE |
| Swagger/OpenAPI | Interfaccia interattiva su `/swagger`, generata automaticamente |
| DTO | Oggetti separati per input (LoginDto, NuovaSpesaDto) e output (UtenteDto, SpesaDto) |
