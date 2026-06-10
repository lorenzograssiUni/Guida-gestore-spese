# Capitolo 16 — Domande e Risposte Tecniche

Questo capitolo raccoglie le domande più probabili dell'esame orale, organizzate per tema, con risposte pronte e riferimenti diretti al codice del progetto.

---

## 16.1 Domande sull'Architettura e sul Protocollo HTTP

---

**D: Cos'è il modello Request-Response e come lo usa il vostro progetto?**

R: Il modello Request-Response è il fondamento di HTTP: il client invia una richiesta e il server risponde. Nel nostro progetto, React (client) invia richieste HTTP tramite il modulo `api.js` usando `fetch()`, e ASP.NET Core (server) risponde con JSON e uno status code. Il ciclo è **stateless**: ogni richiesta è indipendente, il server non ricorda le precedenti.

---

**D: Quali metodi HTTP usate e perché?**

R: Usiamo i quattro metodi principali rispettando la semantica REST:
- `GET` — per leggere dati (es. lista spese di un gruppo)
- `POST` — per creare nuove risorse (es. nuova spesa, login)
- `PUT` — per aggiornare una risorsa esistente (es. modifica nome utente)
- `DELETE` — per eliminare una risorsa (es. rimozione membro da gruppo)

La scelta del metodo non è arbitraria: un `GET` non deve mai modificare dati sul server (principio di idempotenza e sicurezza).

---

**D: Cosa sono gli status code HTTP? Quali restituite nelle vostre API?**

R: Gli status code comunicano l'esito della richiesta. Nel progetto restituiamo:
- `200 OK` — operazione completata con successo (es. login riuscito, lista caricata)
- `201 Created` — risorsa creata con successo (es. nuovo gruppo)
- `400 Bad Request` — dati inviati non validi (es. campo mancante)
- `401 Unauthorized` — credenziali errate (es. password sbagliata)
- `404 Not Found` — risorsa non trovata (es. gruppo inesistente)
- `500 Internal Server Error` — errore imprevisto lato server

---

**D: Cos'è CORS e perché lo avete configurato?**

R: CORS (Cross-Origin Resource Sharing) è un meccanismo di sicurezza del browser che blocca le richieste HTTP verso un dominio diverso da quello della pagina corrente. Il nostro frontend gira su `localhost:5173` (o Vercel) e chiama il backend su `localhost:5207` (o Azure): sono origini diverse, quindi senza CORS il browser rifiuterebbe le chiamate. In `Program.cs` configuriamo una policy che autorizza esplicitamente le origini del frontend.

---

**D: Cosa succede esattamente quando il browser fa una richiesta alla vostra API?**

R: Il flusso completo è:
1. L'utente esegue un'azione in React (es. clicca "Aggiungi spesa")
2. JavaScript chiama `fetch()` con metodo, URL e body JSON
3. Il browser risolve il DNS dell'host del backend
4. Si stabilisce una connessione TCP (3-way handshake: SYN → SYN-ACK → ACK)
5. Il browser invia la richiesta HTTP con headers (`Content-Type: application/json`, `Accept`)
6. Il server ASP.NET Core riceve la richiesta, il middleware la instrada al controller corretto
7. Il controller esegue la logica, interroga EF Core, che genera SQL per SQLite
8. Il server restituisce la risposta JSON con lo status code appropriato
9. React aggiorna lo stato e ri-renderizza il componente

---

**D: Qual è la differenza tra HTTP/1.1 e HTTP/2?**

R: HTTP/1.1 apre una connessione TCP per ogni richiesta (o usa keep-alive per riutilizzarla in sequenza), creando code di attesa. HTTP/2 introduce il **multiplexing**: più richieste viaggiano in parallelo sulla stessa connessione TCP, riducendo la latenza. Il nostro server Kestrel (ASP.NET Core) supporta HTTP/2; in produzione su Azure App Service viene usato HTTP/2 quando il client lo supporta.

---

## 16.2 Domande su REST e le Nostre API

---

**D: Cos'è REST? Chi lo ha definito?**

R: REST (Representational State Transfer) è uno stile architetturale definito da Roy Fielding nella sua dissertazione dottorale del 2000. Non è un protocollo né uno standard, ma un insieme di vincoli che, se rispettati, rendono un sistema scalabile, semplice e interoperabile. I 6 vincoli sono: interfaccia uniforme, architettura client-server, statelessness, cacheabilità, sistema a livelli e code-on-demand (opzionale).

---

**D: Il vostro sistema è veramente RESTful?**

R: Siamo al **Livello 2 del Richardson REST Maturity Model** su 3. Usiamo correttamente le risorse come URL (`/api/Gruppo`, `/api/Spesa`), i metodi HTTP con la loro semantica corretta (GET/POST/PUT/DELETE) e gli status code appropriati. Non implementiamo il **Livello 3** (HATEOAS — Hypermedia As The Engine Of Application State), dove ogni risposta include link alle operazioni successive. Per un progetto universitario, il Livello 2 è lo standard de facto.

---

**D: Cosa sono i DTO e perché li usate?**

R: DTO (Data Transfer Object) è un oggetto usato esclusivamente per trasportare dati tra il client e il server, separato dai modelli del database. Nel progetto usiamo DTO per due motivi: **sicurezza** (non esponiamo campi sensibili come l'hash della password nella risposta) e **flessibilità** (possiamo restituire una struttura diversa da quella della tabella, ad esempio aggiungendo dati calcolati). Ad esempio, `LoginDto` riceve email e password, ma la risposta non include mai il campo `PasswordHash`.

---

**D: Cos'è Swagger e perché lo avete integrato?**

R: Swagger (OpenAPI) è uno strumento che genera automaticamente una documentazione interattiva delle API leggendo le annotazioni dei controller ASP.NET Core. All'URL `/swagger` del nostro backend è possibile vedere tutti gli endpoint, i parametri attesi, i modelli di richiesta/risposta e fare chiamate di test direttamente dal browser, senza bisogno di Postman o altri client. È fondamentale per lo sviluppo e per la comunicazione tra frontend e backend.

---

**D: Come funziona l'endpoint di bilanciamento dei debiti?**

R: Il `RiepilogoController` calcola, per ogni gruppo, il saldo netto di ciascun membro: quanto ha pagato meno quanto deve. Chi ha saldo positivo è creditore, chi ha saldo negativo è debitore. L'algoritmo abbina i debitori ai creditori minimizzando il numero di transazioni: un debitore paga il creditore con il saldo più alto disponibile, e si itera finché tutti i saldi sono a zero. Il risultato è una lista di transazioni del tipo "Marco deve 15€ a Sara".

---

## 16.3 Domande sulla Sicurezza

---

**D: Come gestite le password nel progetto?**

R: Usiamo **BCrypt** con salt automatico. Quando un utente si registra, la password in chiaro viene passata a `BCrypt.HashPassword()`, che genera un hash adattivo con un salt casuale incorporato. L'hash (es. `$2a$11$...`) viene salvato nel database. Al login, `BCrypt.Verify(passwordInChiaro, hashSalvato)` confronta le due stringhe senza mai de-cifrare l'hash. In nessun momento la password in chiaro tocca il database.

---

**D: Cos'è il salt e perché è importante?**

R: Il salt è una stringa casuale aggiunta alla password prima dell'hashing. Serve a rendere inutili gli attacchi con **rainbow table**: tabelle precalcolate che mappano hash comuni a password note. Con il salt, anche due utenti con la stessa password hanno hash completamente diversi, perché ogni salt è unico. BCrypt incorpora automaticamente il salt nell'hash risultante, quindi non serve salvarlo separatamente.

---

**D: Cos'è la SQL Injection e come vi proteggete?**

R: La SQL Injection è un attacco in cui l'utente inserisce codice SQL nei campi di input per manipolare le query del database. Ad esempio, inserire `' OR '1'='1` in un campo email potrebbe restituire tutti gli utenti. Nel nostro progetto siamo protetti perché usiamo **Entity Framework Core**, che usa sempre **query parametrizzate**: i valori inseriti dall'utente non vengono mai concatenati nella stringa SQL, ma passati come parametri separati che il driver SQLite tratta come dati puri, mai come codice.

---

**D: Cos'è XSS?**

R: XSS (Cross-Site Scripting) è un attacco in cui codice JavaScript malevolo viene iniettato nella pagina e eseguito nel browser di altri utenti. React ci protegge nativamente: il JSX **escapa automaticamente** tutti i valori inseriti nelle espressioni `{}`, quindi `<div>{nomeUtente}</div>` non eseguirà mai script anche se `nomeUtente` contiene `<script>alert('xss')</script>`.

---

**D: Cos'è CSRF?**

R: CSRF (Cross-Site Request Forgery) è un attacco in cui un sito malevolo induce il browser dell'utente a fare richieste autenticate verso il nostro backend a sua insaputa. Nel nostro progetto il rischio è mitigato dal fatto che non usiamo cookie per l'autenticazione: l'identità dell'utente è salvata in `localStorage` e inviata nel body JSON, non in cookie automaticamente allegati dal browser.

---

## 16.4 Domande sul Database e EF Core

---

**D: Cos'è un ORM e perché avete scelto Entity Framework Core?**

R: Un ORM (Object-Relational Mapper) è uno strato software che traduce le operazioni sugli oggetti C# in query SQL, eliminando la necessità di scrivere SQL manualmente. Abbiamo scelto EF Core perché è il ORM ufficiale di Microsoft per .NET, è perfettamente integrato con ASP.NET Core, supporta SQLite e offre le migrazioni per gestire l'evoluzione dello schema del database in modo versionato.

---

**D: Perché avete scelto SQLite?**

R: SQLite è un database **serverless**: è un singolo file `.db` che non richiede un processo server separato. Per un progetto universitario con traffico limitato è la scelta ideale: zero configurazione, zero costi, funziona sia in locale che su Azure App Service. I limiti di SQLite (scritture concorrenti limitate, nessuna gestione multi-utente avanzata) non sono rilevanti per il nostro use case.

---

**D: Cos'è `ApplicationDbContext` e cosa fa `OnModelCreating`?**

R: `ApplicationDbContext` è la classe che rappresenta la sessione con il database. Eredita da `DbContext` di EF Core e dichiara le `DbSet<T>` — una per ogni tabella (es. `DbSet<Utente> Utenti`). Il metodo `OnModelCreating` è l'override in cui configuriamo le relazioni che EF Core non riesce a dedurre automaticamente dalle proprietà di navigazione: ad esempio, il `DeleteBehavior.Restrict` che impedisce di cancellare un utente se ha spese associate, o la configurazione della tabella ponte per la relazione N:M tra Utenti e Gruppi.

---

**D: Cosa sono le migrazioni EF Core?**

R: Le migrazioni sono file C# generati automaticamente da `dotnet ef migrations add` che descrivono le modifiche allo schema del database (aggiungere una tabella, una colonna, un indice). Ogni migrazione ha un metodo `Up()` (applicare la modifica) e `Down()` (tornare indietro). `dotnet ef database update` esegue le migrazioni pendenti nell'ordine corretto. Questo permette di evolvere il database in modo controllato e versionato insieme al codice.

---

**D: Qual è la differenza tra `DeleteBehavior.Cascade` e `DeleteBehavior.Restrict`?**

R: Con `Cascade`, quando si elimina un record padre, EF Core elimina automaticamente tutti i figli collegati. Con `Restrict`, l'eliminazione del padre fallisce se esistono figli che lo referenziano. Nel progetto usiamo `Cascade` per le spese di un gruppo (se elimini il gruppo, le spese vengono eliminate) e `Restrict` per impedire di rimuovere un membro che ha spese registrate — obbligando prima a gestire le spese.

---

## 16.5 Domande su React e il Frontend

---

**D: Cos'è React e cos'è una SPA?**

R: React è una libreria JavaScript per costruire interfacce utente basate su **componenti**. Una SPA (Single Page Application) è un'applicazione web che carica una sola pagina HTML e aggiorna dinamicamente il contenuto tramite JavaScript, senza ricaricare la pagina. Nel nostro progetto, `index.html` è l'unica pagina; React gestisce il routing condizionale mostrando o nascondendo componenti in base allo stato dell'applicazione.

---

**D: Come funziona il routing nel vostro progetto senza React Router?**

R: Invece di una libreria di routing, gestiamo la navigazione con lo **stato locale** in `App.jsx`. Una variabile di stato `paginaCorrente` determina quale componente renderizzare: se il valore è `'home'` si mostra `HomePage`, se è `'dettaglio'` si mostra `DettaglioGruppo`, e così via. Questo approccio è sufficiente per applicazioni con un numero limitato di viste e non richiede librerie aggiuntive.

---

**D: Cos'è `localStorage` e come lo usate?**

R: `localStorage` è un meccanismo del browser per salvare dati in formato chiave-valore che persistono anche dopo la chiusura del tab. Lo usiamo per salvare i dati dell'utente loggato (id, nome, email) in modo che al ricaricamento della pagina l'utente non debba ri-autenticarsi. All'avvio, `App.jsx` legge `localStorage` per ripristinare la sessione; al logout, i dati vengono cancellati con `localStorage.removeItem()`.

---

**D: Cos'è Vite e perché l'avete scelto rispetto a Create React App?**

R: Vite è un build tool moderno che usa i **ES modules nativi del browser** durante lo sviluppo, avviando il server in meno di un secondo indipendentemente dalla dimensione del progetto. Create React App (CRA) usava Webpack, che bundlava l'intera applicazione prima di avviarla, diventando lento su progetti grandi. In produzione, Vite usa Rollup per generare bundle ottimizzati. L'esperienza di sviluppo con HMR (Hot Module Replacement) è sensibilmente più reattiva.

---

**D: Cos'è Tailwind CSS e come funziona?**

R: Tailwind CSS è un framework **utility-first**: invece di scrivere classi CSS semantiche (`card`, `button-primary`), si compongono classi di utilità direttamente nell'HTML/JSX (`flex`, `p-4`, `bg-blue-500`, `rounded-lg`). In produzione, Vite analizza tutto il codice e include nel bundle CSS solo le classi effettivamente usate (**tree-shaking del CSS**), producendo file molto piccoli.

---

**D: Cosa sono le Promise e `async/await` in JavaScript?**

R: Una **Promise** è un oggetto che rappresenta il risultato futuro di un'operazione asincrona (es. una chiamata HTTP). Ha tre stati: pending, fulfilled, rejected. `async/await` è syntactic sugar sulle Promise: `await` sospende l'esecuzione della funzione asincrona finché la Promise non si risolve, rendendo il codice asincrono leggibile come se fosse sincrono. Nel modulo `api.js` ogni funzione è `async` e usa `await fetch(...)` per attendere la risposta del server.

---

## 16.6 Domande "Trabocchetto" — Cosa Rispondere

---

**D: REST e HTTP sono la stessa cosa?**

R: No. HTTP è un **protocollo** di trasporto, REST è uno **stile architetturale**. REST può essere implementato su HTTP (come facciamo noi) ma tecnicamente potrebbe usare altri protocolli. Viceversa, HTTP può essere usato senza seguire i principi REST (ad esempio, mettere tutto su POST con azioni nell'URL non è REST). Nella pratica moderna, REST implica quasi sempre l'uso di HTTP.

---

**D: I vostri endpoint sono REST o RPC?**

R: Sono prevalentemente REST. Un endpoint RPC (Remote Procedure Call) chiamerebbe un'azione (`/api/creaGruppo`, `/api/eliminaSpesa`), mentre il nostro approccio usa **risorse** (`/api/Gruppo`, `/api/Spesa`) con il metodo HTTP che indica l'azione (POST per creare, DELETE per eliminare). Alcuni endpoint di logica complessa (es. `/api/Riepilogo/SaldaDebito`) possono essere considerati ibridi, ma la struttura generale segue REST.

---

**D: Perché non usate JWT per l'autenticazione?**

R: Una scelta consapevole di semplicità. JWT (JSON Web Token) è uno standard robusto per autenticazione stateless ma aggiunge complessità: gestione della firma, refresh token, scadenza. Per il nostro progetto universitario, con un singolo utente alla volta e sessioni gestite tramite `localStorage`, la complessità di JWT non era giustificata. In un contesto produttivo con requisiti di sicurezza elevati, JWT sarebbe la scelta corretta.

---

**D: SQLite va bene per la produzione?**

R: Dipende dal contesto. SQLite è adatto per applicazioni con traffico basso, letture prevalenti e accesso da un singolo processo. Non è adatto per sistemi con molte scritture concorrenti, repliche o cluster. Per il nostro progetto universitario è una scelta appropriata. In un contesto enterprise si userebbe PostgreSQL o SQL Server con EF Core (la migrazione richiede solo di cambiare il provider nel `DbContext`).

---

**D: Cosa succederebbe se due utenti modificano la stessa spesa contemporaneamente?**

R: Nel nostro sistema non gestiamo la **concorrenza ottimistica**. Se due utenti modificassero la stessa spesa simultaneamente, l'ultima scrittura sovrascrive la precedente (last write wins). In un sistema produttivo si gestirebbe con un campo `RowVersion` o `Timestamp` in EF Core: il server rifiuta la modifica se il record è stato cambiato da quando il client l'ha letto, restituendo un `409 Conflict`.

---

**D: Perché il database SQLite si perde su Azure?**

R: Azure App Service nel piano Free F1 usa un filesystem **effimero**: al riavvio del container, i file scritti fuori dalla cartella `C:\home` vengono persi. Il file SQLite va salvato in `C:\home\data\` per persistere tra i riavvii. Nel progetto gestiamo questo con `EnsureCreated()` in `Program.cs` che ricrea il database (vuoto) se non esiste, e con il percorso corretto del file configurato per Azure.

---

**D: React è un framework o una libreria?**

R: React è tecnicamente una **libreria** — gestisce solo la UI (il "V" del pattern MVC). Non include routing, gestione stato globale, chiamate HTTP o form handling. Framework completi come Angular includono tutto questo. Noi abbiamo affiancato React con librerie specifiche: Vite (build), Tailwind (stile), e un modulo `api.js` custom per le chiamate HTTP.
