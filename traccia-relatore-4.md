# 🎤 Traccia Relatore 4 — Protocolli HTTP, CI/CD & Demo

> **Capitoli di riferimento:** Cap. 1 (HTTP, CORS, Status Code), Cap. 5 (Deploy, GitHub Actions, PWA)
> **Durata stimata:** ~4 minuti

---

## Apertura — Passaggio di consegne

> *"Concludo parlando dei protocolli su cui si basa tutta la comunicazione di Split Mate, del sistema di integrazione continua che abbiamo configurato, e vi mostro la demo live."*

---

## 1. HTTP — Il protocollo fondamentale

Tutta la comunicazione tra React e ASP.NET Core avviene tramite **HTTP (HyperText Transfer Protocol)**.

HTTP è un protocollo **stateless** (senza stato): ogni richiesta è completamente indipendente dalla precedente — il server non ricorda chi sei tra una chiamata e l'altra. Per questo usiamo il `localStorage` per persistere i dati dell'utente loggato lato client.

Ogni interazione dell'utente genera una coppia **Request-Response**:
- Il client (React) invia una **richiesta** con metodo, URL, header e opzionalmente un body JSON
- Il server (ASP.NET Core) elabora e risponde con uno **status code** e i dati

---

## 2. Verbi HTTP e Status Code

L'API di Split Mate usa i 4 verbi HTTP in modo semantico:

| Verbo | Azione | Esempio endpoint |
|-------|--------|------------------|
| `GET` | Leggi una risorsa | `GET /api/Spesa/gruppo/5` |
| `POST` | Crea una nuova risorsa | `POST /api/Spesa` |
| `PUT` | Aggiorna una risorsa esistente | `PUT /api/Spesa/3` |
| `DELETE` | Elimina una risorsa | `DELETE /api/Gruppo/2` |

I principali **status code** restituiti dal nostro backend:

| Codice | Significato | Quando |
|--------|-------------|--------|
| `200 OK` | Successo | Login riuscito, GET dati |
| `201 Created` | Risorsa creata | Nuovo gruppo o spesa |
| `204 No Content` | Operazione ok senza risposta | DELETE riuscita |
| `400 Bad Request` | Dati errati | Email vuota, importo negativo |
| `401 Unauthorized` | Non autenticato | Password sbagliata |
| `404 Not Found` | Risorsa non trovata | Gruppo inesistente |

---

## 3. Il percorso completo di una richiesta

Quando l'utente clicca "Aggiungi Spesa", ecco cosa succede passo per passo:

1. **DNS Lookup**: il browser risolve `gestione-spese-api.azurewebsites.net` in un indirizzo IP (dalla cache locale o interrogando un resolver DNS)
2. **TCP Three-Way Handshake**: client e server si accordano sulla connessione (SYN → SYN-ACK → ACK)
3. **TLS Handshake**: negoziazione HTTPS per cifrare la comunicazione
4. **Invio richiesta**: React chiama `fetch()` in `api.js` con `POST /api/Spesa` e body JSON
5. **Elaborazione backend**: `SpesaController` deserializza il DTO, valida, calcola divisioni, salva su SQLite → risponde `201 Created`
6. **Aggiornamento UI**: React riceve la risposta → `useState` aggiorna lo stato → lista spese ri-renderizzata **senza ricaricare la pagina** (vantaggio SPA)

---

## 4. CORS — Cross-Origin Resource Sharing

Il frontend è su `gestore-spese-xi.vercel.app` e il backend su `gestione-spese-api.azurewebsites.net` — **domini diversi**. Il browser impone la **Same-Origin Policy** e blocca le richieste cross-origin per default.

Il browser invia prima una **Preflight Request** (`OPTIONS`) per chiedere al server se permette la richiesta dall'origine del frontend. Il server risponde con gli header `Access-Control-Allow-Origin`, e solo se confermato il browser invia la richiesta reale.

In `Program.cs` abbiamo configurato la policy CORS:

```csharp
policy.AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod();
```

> **Nota di sicurezza**: `AllowAnyOrigin` (wildcard `*`) è permissivo — in produzione si dovrebbe usare `.WithOrigins("https://gestore-spese-xi.vercel.app")` per accettare solo il nostro frontend, applicando il principio del privilegio minimo.

---

## 5. HTTPS e TLS

Entrambi i servizi usano **HTTPS** — obbligatorio per proteggere i JWT in transito. Senza TLS, chiunque sulla rete potrebbe intercettare il token e impersonare l'utente (attacco Man-in-the-Middle).

**TLS** garantisce:
- **Autenticazione** del server tramite certificato firmato da una CA
- **Integrità**: i messaggi non possono essere alterati in transito
- **Confidenzialità**: i dati sono crittografati

Sia Vercel che Azure gestiscono i certificati TLS **automaticamente** tramite Let's Encrypt — nessun costo o rinnovo manuale.

---

## 6. Cookie vs localStorage — Gestione della sessione

HTTP è stateless — bisogna persistere l'identità dell'utente lato client. Le due opzioni principali:

| Meccanismo | Pro | Contro |
|------------|-----|--------|
| **Cookie HttpOnly** | Non accessibile da JS → protezione XSS | Più complesso, CSRF da gestire |
| **localStorage** ← scelta nostra | Semplice, ideale per SPA | Accessibile da JS (attenzione XSS) |

Abbiamo scelto `localStorage` per la semplicità: non abbiamo JWT o OAuth complessi, è uno strumento interno, e la semplicità di gestione supera la differenza di sicurezza per questo caso d'uso.

---

## 7. CI/CD con GitHub Actions

Ad ogni push su `main`, la pipeline **`ci.yml`** si avvia automaticamente:

```
push su main
    ↓
[Job 1] dotnet restore && dotnet build   ← build backend
[Job 2] npm install && npm run build     ← build frontend
    ↓
badge di stato aggiornato nel README
```

Questo garantisce che il codice sul branch principale sia **sempre in uno stato buildabile** — nessun commit rotto arriva in produzione senza essere rilevato.

I segreti di deploy (credenziali Azure, API keys) sono salvati come **GitHub Secrets** e non compaiono mai nel codice sorgente.

---

## 8. Manuale — struttura dei 17 capitoli

Il manuale `guida-gestore-spese` collega ogni concetto teorico del corso all'implementazione concreta in Split Mate:

| Capitoli | Argomenti |
|----------|-----------|
| 1–3 | HTTP, architettura SPA/MVC/REST, backend e sicurezza |
| 4–6 | Frontend React, frontend avanzato, deployment cloud |
| 7–9 | WebSocket, autenticazione, testing |
| 10–13 | Performance, PWA, accessibilità, internazionalizzazione |
| 14–17 | Microservizi, Docker, CI/CD avanzato, pattern architetturali |

Ogni sezione del manuale risponde anche alle domande tipiche del professore, con esempi di codice reale tratti direttamente dal repository.

---

## 9. Demo live

> *"Vi mostro ora l'applicazione in produzione."*

**Credenziali di test:**
- Email: `test@splitmate.it`
- Password: `test1234`

L'account di test ha già gruppi e spese di esempio per mostrare tutte le funzionalità. Possiamo anche aprire lo **Swagger UI** del backend per vedere tutti gli endpoint documentati automaticamente.

**Cosa mostrare nella demo:**
1. Login con le credenziali di test
2. Lista gruppi nella homepage
3. Entrare in un gruppo → mostrare la lista spese
4. Aggiungere una spesa → mostrare l'aggiornamento in tempo reale
5. Aprire il riepilogo → mostrare l'algoritmo che calcola i saldi
6. Aprire Swagger UI → mostrare gli endpoint REST documentati

---

## Possibili domande del professore

**"Cos'è CORS e perché serve?"**
> Il browser blocca per default le richieste verso un dominio diverso da quello della pagina (Same-Origin Policy). CORS permette al server di dichiarare esplicitamente quali origini sono autorizzate a fare richieste. Nel nostro caso, il backend Azure deve autorizzare esplicitamente il frontend Vercel.

**"Differenza tra HTTP/1.1, HTTP/2 e HTTP/3?"**
> HTTP/1.1 apre una connessione TCP per ogni richiesta. HTTP/2 introduce il multiplexing: più richieste viaggiano sulla stessa connessione TCP come frame interlacciati, eliminando il head-of-line blocking. HTTP/3 abbandona TCP e usa QUIC su UDP, con handshake più veloce (1-RTT) e resilienza migliore alla perdita di pacchetti. Azure App Service supporta HTTP/2 automaticamente.

**"Perché HTTPS è obbligatorio?"**
> Senza TLS, i JWT viaggiano in chiaro e chiunque sulla rete può intercettarli (Man-in-the-Middle). Con HTTPS la comunicazione è crittografata end-to-end. È anche un requisito tecnico per le PWA e per Google OAuth.

**"Cos'è il CI/CD?"**
> Continuous Integration / Continuous Deployment: ogni push sul repository avvia automaticamente build e test. Se tutto passa, il codice viene deployato in produzione senza intervento manuale. GitHub Actions è il sistema che esegue questa pipeline — il file di configurazione è `.github/workflows/ci.yml`.
