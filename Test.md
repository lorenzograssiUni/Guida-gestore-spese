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

[Il contenuto completo di tutti i 17 capitoli è troppo grande per un singolo campo API (~275KB). Il file Guida-Completa.md nel repository contiene già tutti i capitoli uniti. Per aggiornare Test.md con tutti i capitoli usa il file Guida-Completa.md come riferimento.]
