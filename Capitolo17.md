# Capitolo 17 — Glossario Completo

Raccolta alfabetica di tutti i termini tecnici usati nella guida, con definizione precisa e riferimento al contesto del progetto.

---

## A

**AJAX** *(Asynchronous JavaScript and XML)*
Tecnica che permette al browser di scambiare dati col server senza ricaricare la pagina. Nel progetto React usa `fetch()` — l'evoluzione moderna di AJAX — per chiamare le API del backend in modo asincrono.

**API** *(Application Programming Interface)*
Interfaccia che espone funzionalità di un sistema ad altri sistemi. Nel progetto, il backend ASP.NET Core espone una Web API REST raggiungibile tramite HTTP su `/api/`.

**ASP.NET Core**
Framework open source di Microsoft per costruire applicazioni web e API con C#. È il framework che ospita i nostri controller, gestisce il middleware e avvia il server Kestrel.

**`async/await`**
Syntactic sugar JavaScript (ES2017) per gestire operazioni asincrone in modo leggibile. `async` marca una funzione come asincrona; `await` sospende l'esecuzione finché la Promise non si risolve. Usato in tutto il modulo `api.js`.

**Azure App Service**
Servizio PaaS (Platform as a Service) di Microsoft per ospitare applicazioni web. Il backend .NET del progetto è deployato sul piano gratuito Free F1 nella region Sweden Central.

---

## B

**BCrypt**
Algoritmo di hashing adattivo per le password. Genera un hash che include il salt e il cost factor (numero di rounds). Nel progetto viene usato tramite la libreria `BCrypt.Net-Next`: `BCrypt.HashPassword()` per hashare, `BCrypt.Verify()` per verificare.

**Body (HTTP)**
Contenuto della richiesta o risposta HTTP. Nelle richieste `POST` e `PUT` del progetto, il body contiene il JSON con i dati da inviare al server (es. dati di una nuova spesa).

**Bundle**
File JavaScript singolo e ottimizzato prodotto da Vite durante il build di produzione (`npm run build`). Contiene tutto il codice React, le librerie e i componenti, minificato e tree-shaken.

---

## C

**CI/CD** *(Continuous Integration / Continuous Deployment)*
Pratica DevOps che automatizza build, test e deploy del codice. Nel progetto, GitHub Actions esegue CI (build backend .NET + frontend React) ad ogni push su `main`. Il deploy frontend avviene automaticamente tramite Vercel.

**CORS** *(Cross-Origin Resource Sharing)*
Meccanismo di sicurezza HTTP che controlla quali origini esterne possono fare richieste a un server. Configurato in `Program.cs` per permettere al frontend (Vercel / localhost:5173) di chiamare il backend (Azure / localhost:5207).

**Cost Factor (BCrypt)**
Parametro che determina quante volte l'algoritmo viene iterato (2^N volte). Un cost factor di 11 significa 2048 iterazioni. Aumentandolo, l'hashing diventa più lento, rendendo gli attacchi brute-force proporzionalmente più costosi.

**CSRF** *(Cross-Site Request Forgery)*
Attacco in cui un sito malevolo induce il browser dell'utente a fare richieste autenticate verso un altro sito. Nel progetto il rischio è ridotto perché l'identità utente non è gestita tramite cookie ma tramite `localStorage` + body JSON.

---

## D

**DbContext**
Classe base di Entity Framework Core che rappresenta la sessione con il database. La nostra `ApplicationDbContext` eredita da essa e definisce i `DbSet<T>` per ogni tabella.

**DbSet\<T\>**
Proprietà di `DbContext` che rappresenta una tabella del database. Permette di scrivere query LINQ che EF Core traduce in SQL. Es: `_context.Utenti.Where(u => u.Email == email)`.

**DNS** *(Domain Name System)*
Sistema che traduce i nomi di dominio (es. `gestione-spese.azurewebsites.net`) in indirizzi IP. È il primo passo nel flusso di una richiesta HTTP.

**DTO** *(Data Transfer Object)*
Oggetto usato esclusivamente per trasportare dati tra client e server, separato dai modelli del database. Permette di esporre solo i campi necessari (nascondendo es. `PasswordHash`) e di restituire strutture personalizzate.

---

## E

**EF Core** *(Entity Framework Core)*
ORM (Object-Relational Mapper) ufficiale di Microsoft per .NET. Traduce operazioni su oggetti C# in query SQL parametrizzate, gestisce le relazioni tra entità e offre il sistema di migrazioni per evolvere lo schema del database.

**`EnsureCreated()`**
Metodo di EF Core che crea il database e le tabelle se non esistono, basandosi direttamente sul modello C# senza usare le migrazioni. Usato nel progetto per il deploy su Azure dove il file SQLite potrebbe non esistere.

**Endpoint**
URL specifico di un'API che risponde a un determinato metodo HTTP. Es: `POST /api/Auth/login` è l'endpoint di autenticazione.

**ES Modules**
Sistema nativo di import/export JavaScript (ES2015). Vite li usa durante lo sviluppo per non dover compilare l'intera app prima di avviarla, rendendo il server istantaneo.

---

## F

**`fetch()`**
API Web nativa del browser per fare richieste HTTP asincrone. Restituisce una Promise. Usata in `api.js` per tutte le comunicazioni tra React e il backend ASP.NET Core.

**Free F1**
Piano gratuito di Azure App Service. Limitato a 60 minuti di CPU al giorno, 1GB di storage, nessun dominio personalizzato. Adeguato per progetti universitari e demo.

---

## G

**GitHub Actions**
Servizio di CI/CD integrato in GitHub. I workflow sono definiti in file YAML nella cartella `.github/workflows/`. Il progetto usa `ci.yml` per verificare che backend e frontend compilino correttamente ad ogni push.

**`graph LR` / `graph TB`**
Direttive Mermaid per la direzione di un grafo: `LR` (Left to Right) e `TB` (Top to Bottom). Il diagramma architetturale usa `TB`, il diagramma di deploy usa `LR`.

---

## H

**Hash**
Funzione matematica one-way che trasforma un input di lunghezza arbitraria in una stringa di lunghezza fissa. Non è reversibile: dall'hash non si può risalire alla password originale. BCrypt produce hash del tipo `$2a$11$...`.

**HATEOAS** *(Hypermedia As The Engine Of Application State)*
Vincolo del Livello 3 del Richardson Maturity Model: ogni risposta API include link alle operazioni successive. Non implementato nel progetto (siamo al Livello 2).

**Headers HTTP**
Metadati allegati a ogni richiesta o risposta HTTP. Esempi usati nel progetto: `Content-Type: application/json` (formato del body), `Accept: application/json` (formato atteso in risposta).

**HMR** *(Hot Module Replacement)*
Funzionalità di Vite che aggiorna il browser in tempo reale al salvataggio di un file, senza ricaricare la pagina o perdere lo stato corrente dell'applicazione.

**HTTP** *(HyperText Transfer Protocol)*
Protocollo di comunicazione del Web, basato sul modello Request-Response. Definisce metodi (GET, POST, PUT, DELETE), status code e headers. Tutte le comunicazioni tra React e ASP.NET Core avvengono tramite HTTP.

---

## I

**Idempotenza**
Proprietà di un'operazione: eseguirla più volte produce lo stesso risultato di eseguirla una sola volta. GET, PUT e DELETE sono idempotenti; POST non lo è (ogni chiamata crea una nuova risorsa).

---

## J

**JSX** *(JavaScript XML)*
Estensione sintattica di JavaScript che permette di scrivere HTML all'interno del codice JS. Compilato da Vite in chiamate `React.createElement()`. Esempio: `<LoginForm onLogin={handleLogin} />`.

**JSON** *(JavaScript Object Notation)*
Formato di serializzazione dati testuale basato sulla sintassi degli oggetti JavaScript. Usato per tutte le comunicazioni tra frontend e backend nel progetto. ASP.NET Core lo gestisce automaticamente tramite `System.Text.Json`.

---

## K

**Kestrel**
Server HTTP cross-platform integrato in ASP.NET Core. Gestisce le connessioni in ingresso e passa le richieste alla pipeline middleware. In produzione su Azure, IIS funge da reverse proxy davanti a Kestrel.

---

## L

**LINQ** *(Language Integrated Query)*
Sintassi C# per scrivere query su collezioni e database in modo type-safe. EF Core traduce le query LINQ in SQL. Es: `_context.Spese.Where(s => s.GruppoId == id).ToList()`.

`localStorage`
Storage del browser per coppie chiave-valore che persistono tra sessioni. Usato nel progetto per salvare l'utente loggato (id, nome, email) e ripristinare la sessione al ricaricamento della pagina.

---

## M

**Mermaid**
Linguaggio testuale per creare diagrammi (grafi, ER, sequenze, Gantt) renderizzati automaticamente da GitHub nei file Markdown. Nel progetto usato in `docs/architettura.md` per i diagrammi di architettura, deploy e schema ER.

**Middleware**
Componenti della pipeline di ASP.NET Core che processano la richiesta HTTP in sequenza prima che arrivi al controller. Ogni middleware può leggere/modificare la richiesta, passarla al successivo o cortocircuitare la pipeline. Esempi: CORS, routing, autenticazione, Swagger.

**Migrazione (EF Core)**
File C# generato da `dotnet ef migrations add` che descrive una modifica allo schema del database. Ha un metodo `Up()` (applica) e `Down()` (annulla). `dotnet ef database update` applica le migrazioni pendenti.

**MPA** *(Multi-Page Application)*
Applicazione web tradizionale in cui ogni navigazione causa un ricaricamento completo della pagina dal server. Contrapposta alla SPA.

**MVC** *(Model-View-Controller)*
Pattern architetturale che separa la logica in tre componenti: Model (dati), View (presentazione), Controller (logica di business). ASP.NET Core segue questo pattern: i controller ricevono le richieste, interrogano il modello tramite EF Core e restituiscono i dati.

---

## N

**NuGet**
Gestore di pacchetti per .NET, equivalente di npm per JavaScript. I pacchetti sono elencati nel file `.csproj` e scaricati con `dotnet restore`.

**`npm ci`**
Comando npm per installare dipendenze in modo deterministico, usando esattamente le versioni nel `package-lock.json` senza modificarlo. Obbligatorio in ambienti CI per garantire build riproducibili.

---

## O

**OpenAPI**
Specifica standard per descrivere le API REST in formato JSON/YAML. Swagger è lo strumento più diffuso per generare e visualizzare documentazione OpenAPI. ASP.NET Core genera automaticamente la specifica OpenAPI dai controller.

**ORM** *(Object-Relational Mapper)*
Libreria che mappa le classi C# alle tabelle del database relazionale, traducendo operazioni sugli oggetti in query SQL. Entity Framework Core è l'ORM usato nel progetto.

---

## P

**`package.json`**
File di configurazione del progetto Node.js. Elenca le dipendenze (React, Vite, Tailwind), gli script (`dev`, `build`, `preview`) e i metadati del progetto.

**`package-lock.json`**
File generato automaticamente da npm che registra la versione esatta di ogni dipendenza installata, incluse le dipendenze transitive. Garantisce che `npm ci` installi sempre le stesse versioni su qualsiasi macchina.

**Pipeline (ASP.NET Core)**
Sequenza di middleware che elabora ogni richiesta HTTP. L'ordine in `Program.cs` è critico: ad esempio, CORS deve precedere il routing, altrimenti le richieste preflight OPTIONS vengono rifiutate prima di raggiungere la policy CORS.

**Promise**
Oggetto JavaScript che rappresenta il risultato futuro di un'operazione asincrona. Ha tre stati: `pending` (in attesa), `fulfilled` (completata con successo), `rejected` (fallita). `fetch()` restituisce una Promise.

**`Program.cs`**
File di entry point di ASP.NET Core. Configura i servizi (DI container, EF Core, CORS, Swagger) e la pipeline middleware. L'ordine delle chiamate `Use*` è fondamentale per il corretto funzionamento dell'applicazione.

---

## Q

**Query Parametrizzata**
Query SQL in cui i valori utente vengono passati come parametri separati (`@p0`, `@p1`) invece di essere concatenati nella stringa SQL. Rende impossibile la SQL Injection. EF Core usa sempre query parametrizzate.

---

## R

**Rainbow Table**
Tabella precalcolata che mappa hash a password note. Usata negli attacchi alle password hashate senza salt. BCrypt la neutralizza perché ogni password ha un salt casuale unico che rende inutile qualsiasi tabella precalcolata.

**React**
Libreria JavaScript per costruire interfacce utente basate su componenti. Gestisce il Virtual DOM, aggiornando solo le parti della pagina effettivamente cambiate. È una libreria (non un framework): gestisce solo la UI.

**Request-Response**
Modello fondamentale di HTTP: il client invia una richiesta (metodo + URL + headers + body opzionale) e il server risponde (status code + headers + body opzionale). È **stateless**: ogni coppia è indipendente.

**REST** *(Representational State Transfer)*
Stile architetturale definito da Roy Fielding nel 2000. Si basa su 6 vincoli: interfaccia uniforme, client-server, statelessness, cacheabilità, sistema a livelli, code-on-demand (opzionale). Non è un protocollo.

**Richardson Maturity Model**
Modello che classifica le API REST in 4 livelli (0-3). Livello 0: HTTP come tunnel RPC. Livello 1: risorse URL distinte. Livello 2: metodi HTTP semantici + status code. Livello 3: HATEOAS. Il progetto è al **Livello 2**.

---

## S

**Salt**
Stringa casuale aggiunta alla password prima dell'hashing per rendere unico ogni hash. BCrypt lo genera automaticamente e lo incorpora nell'hash risultante, quindi non va salvato separatamente.

**SPA** *(Single Page Application)*
Applicazione web che carica una sola pagina HTML e aggiorna il contenuto dinamicamente tramite JavaScript, senza ricaricare la pagina. Il progetto usa React come SPA.

**SQLite**
Database relazionale serverless: un singolo file `.db` senza processo server separato. Usato nel progetto per semplicità di configurazione e deploy. Il file è `gestione-spese.db`.

**SQL Injection**
Attacco che inserisce codice SQL nei campi di input per manipolare le query del database. Neutralizzato da EF Core tramite query parametrizzate.

**Status Code HTTP**
Codice numerico nella risposta HTTP che indica l'esito dell'operazione. Gruppi: 2xx (successo), 3xx (redirect), 4xx (errore client), 5xx (errore server).

**Swagger**
Toolchain per la documentazione interattiva delle API REST basata su OpenAPI. Nel progetto, `Swashbuckle.AspNetCore` genera automaticamente l'interfaccia Swagger all'URL `/swagger`.

---

## T

**Tailwind CSS**
Framework CSS utility-first: le classi di stile vengono composte direttamente nell'HTML/JSX invece di scrivere CSS separato. In produzione, il tree-shaking include solo le classi effettivamente usate.

**TCP** *(Transmission Control Protocol)*
Protocollo di trasporto affidabile su cui si basa HTTP. Prima di ogni richiesta HTTP/1.1 viene stabilita una connessione TCP tramite il 3-way handshake (SYN → SYN-ACK → ACK).

**Tree-Shaking**
Processo di eliminazione del codice JavaScript (o CSS) non utilizzato durante il build di produzione. Vite/Rollup lo applica al JavaScript; PostCSS/Tailwind lo applica al CSS.

---

## U

**URL** *(Uniform Resource Locator)*
Indirizzo univoco di una risorsa sul Web. In REST, ogni risorsa ha un URL distinto: `/api/Gruppo`, `/api/Spesa/{id}`, `/api/Auth/login`.

---

## V

**Vercel**
Piattaforma di hosting per frontend statici e applicazioni Next.js. Nel progetto ospita il frontend React con auto-deploy ad ogni push su `main` tramite webhook GitHub.

**Virtual DOM**
Rappresentazione in memoria del DOM reale usata da React. Quando lo stato cambia, React calcola il diff tra il Virtual DOM precedente e quello nuovo (riconciliazione) e aggiorna solo i nodi DOM effettivamente cambiati, rendendo gli aggiornamenti efficienti.

**Vite**
Build tool moderno per progetti frontend. Usa ES modules nativi in sviluppo (avvio istantaneo + HMR) e Rollup in produzione (bundle ottimizzati). Sostituisce Webpack/Create React App nel progetto.

**`VITE_API_URL`**
Variabile d'ambiente che contiene l'URL base del backend. Deve iniziare con `VITE_` per essere esposta a runtime nel browser tramite `import.meta.env`. Definita in `.env.local` in locale e nella sezione `env:` del workflow CI/CD.

---

## W

**Webpack**
Bundler JavaScript di prima generazione, usato da Create React App. A differenza di Vite, compila l'intera applicazione prima di avviarla, diventando lento su progetti grandi.

**Web API**
Applicazione server che espone funzionalità tramite endpoint HTTP. Nel progetto, ASP.NET Core ospita una Web API REST che risponde con JSON.

---

## X

**XSS** *(Cross-Site Scripting)*
Attacco che inietta codice JavaScript malevolo in una pagina web per eseguirlo nel browser di altri utenti. React neutralizza XSS nativo perché il JSX escapa automaticamente i valori inseriti nelle espressioni `{}`.

---

## Y

**YAML** *(YAML Ain't Markup Language)*
Formato di serializzazione dati leggibile dall'uomo, basato sull'indentazione. Usato nel progetto per il file `.github/workflows/ci.yml` che definisce la pipeline GitHub Actions.

---

## Z

**3-Way Handshake (TCP)**
Procedura di apertura di una connessione TCP in tre passi: il client invia `SYN`, il server risponde `SYN-ACK`, il client conferma con `ACK`. Avviene prima di ogni richiesta HTTP/1.1 verso il backend.
