# Capitolo 17 — Glossario Completo

Raccolta alfabetica di tutti i termini tecnici usati nella guida, con definizione precisa e riferimento al contesto del progetto.

---

## Indice rapido

[A](#a) · [B](#b) · [C](#c) · [D](#d) · [E](#e) · [F](#f) · [G](#g) · [H](#h) · [I](#i) · [J](#j) · [K](#k) · [L](#l) · [M](#m) · [N](#n) · [O](#o) · [P](#p) · [Q](#q) · [R](#r) · [S](#s) · [T](#t) · [U](#u) · [V](#v) · [W](#w) · [X](#x) · [Y](#y) · [Z](#z)

### Tutti i termini

[AJAX](#ajax) · [API](#api) · [ASP.NET Core](#aspnet-core) · [async/await](#asyncawait) · [Azure App Service](#azure-app-service) · [BCrypt](#bcrypt) · [Body HTTP](#body-http) · [Bundle](#bundle) · [CI/CD](#cicd) · [CORS](#cors) · [Cost Factor](#cost-factor-bcrypt) · [CSRF](#csrf) · [DbContext](#dbcontext) · [DbSet\<T\>](#dbsett) · [DNS](#dns) · [DTO](#dto) · [EF Core](#ef-core) · [EnsureCreated()](#ensurecreated) · [Endpoint](#endpoint) · [ES Modules](#es-modules) · [fetch()](#fetch) · [Free F1](#free-f1) · [GitHub Actions](#github-actions) · [Hash](#hash) · [HATEOAS](#hateoas) · [Headers HTTP](#headers-http) · [HMR](#hmr) · [HTTP](#http) · [Idempotenza](#idempotenza) · [JSX](#jsx) · [JSON](#json) · [Kestrel](#kestrel) · [LINQ](#linq) · [localStorage](#localstorage) · [Mermaid](#mermaid) · [Middleware](#middleware) · [Migrazione EF Core](#migrazione-ef-core) · [MPA](#mpa) · [MVC](#mvc) · [NuGet](#nuget) · [npm ci](#npm-ci) · [OpenAPI](#openapi) · [ORM](#orm) · [package.json](#packagejson) · [package-lock.json](#package-lockjson) · [Pipeline ASP.NET Core](#pipeline-aspnet-core) · [Promise](#promise) · [Program.cs](#programcs) · [Query Parametrizzata](#query-parametrizzata) · [Rainbow Table](#rainbow-table) · [React](#react) · [Request-Response](#request-response) · [REST](#rest) · [Richardson Maturity Model](#richardson-maturity-model) · [Salt](#salt) · [SPA](#spa) · [SQLite](#sqlite) · [SQL Injection](#sql-injection) · [Status Code HTTP](#status-code-http) · [Swagger](#swagger) · [Tailwind CSS](#tailwind-css) · [TCP](#tcp) · [Tree-Shaking](#tree-shaking) · [URL](#url) · [Vercel](#vercel) · [Virtual DOM](#virtual-dom) · [Vite](#vite) · [VITE_API_URL](#vite_api_url) · [Webpack](#webpack) · [Web API](#web-api) · [XSS](#xss) · [YAML](#yaml) · [3-Way Handshake](#3-way-handshake-tcp)

---

## A

<a id="ajax"></a>
**AJAX** *(Asynchronous JavaScript and XML)*
Tecnica che permette al browser di scambiare dati col server senza ricaricare la pagina. Nel progetto [React](#react) usa [`fetch()`](#fetch) — l'evoluzione moderna di AJAX — per chiamare le [API](#api) del backend in modo asincrono.

<a id="api"></a>
**API** *(Application Programming Interface)*
Interfaccia che espone funzionalità di un sistema ad altri sistemi. Nel progetto, il backend [ASP.NET Core](#aspnet-core) espone una [Web API](#web-api) [REST](#rest) raggiungibile tramite [HTTP](#http) su `/api/`.

<a id="aspnet-core"></a>
**ASP.NET Core**
Framework open source di Microsoft per costruire applicazioni web e [API](#api) con C#. È il framework che ospita i nostri controller, gestisce il [middleware](#middleware) e avvia il server [Kestrel](#kestrel).

<a id="asyncawait"></a>
**`async/await`**
Syntactic sugar JavaScript (ES2017) per gestire operazioni asincrone in modo leggibile. `async` marca una funzione come asincrona; `await` sospende l'esecuzione finché la [Promise](#promise) non si risolve. Usato in tutto il modulo `api.js`.

<a id="azure-app-service"></a>
**Azure App Service**
Servizio PaaS (Platform as a Service) di Microsoft per ospitare applicazioni web. Il backend .NET del progetto è deployato sul piano gratuito [Free F1](#free-f1) nella region Sweden Central.

---

## B

<a id="bcrypt"></a>
**BCrypt**
Algoritmo di hashing adattivo per le password. Genera un [hash](#hash) che include il [salt](#salt) e il [cost factor](#cost-factor-bcrypt) (numero di rounds). Nel progetto viene usato tramite la libreria `BCrypt.Net-Next`: `BCrypt.HashPassword()` per hashare, `BCrypt.Verify()` per verificare.

<a id="body-http"></a>
**Body (HTTP)**
Contenuto della richiesta o risposta [HTTP](#http). Nelle richieste `POST` e `PUT` del progetto, il body contiene il [JSON](#json) con i dati da inviare al server (es. dati di una nuova spesa).

<a id="bundle"></a>
**Bundle**
File JavaScript singolo e ottimizzato prodotto da [Vite](#vite) durante il build di produzione (`npm run build`). Contiene tutto il codice [React](#react), le librerie e i componenti, minificato e [tree-shaken](#tree-shaking).

---

## C

<a id="cicd"></a>
**CI/CD** *(Continuous Integration / Continuous Deployment)*
Pratica DevOps che automatizza build, test e deploy del codice. Nel progetto, [GitHub Actions](#github-actions) esegue CI (build backend .NET + frontend [React](#react)) ad ogni push su `main`. Il deploy frontend avviene automaticamente tramite [Vercel](#vercel).

<a id="cors"></a>
**CORS** *(Cross-Origin Resource Sharing)*
Meccanismo di sicurezza [HTTP](#http) che controlla quali origini esterne possono fare richieste a un server. Configurato in [`Program.cs`](#programcs) per permettere al frontend ([Vercel](#vercel) / localhost:5173) di chiamare il backend ([Azure App Service](#azure-app-service) / localhost:5207).

<a id="cost-factor-bcrypt"></a>
**Cost Factor (BCrypt)**
Parametro che determina quante volte l'algoritmo [BCrypt](#bcrypt) viene iterato (2^N volte). Un cost factor di 11 significa 2048 iterazioni. Aumentandolo, l'hashing diventa più lento, rendendo gli attacchi brute-force proporzionalmente più costosi.

<a id="csrf"></a>
**CSRF** *(Cross-Site Request Forgery)*
Attacco in cui un sito malevolo induce il browser dell'utente a fare richieste autenticate verso un altro sito. Nel progetto il rischio è ridotto perché l'identità utente non è gestita tramite cookie ma tramite [`localStorage`](#localstorage) + [body](#body-http) [JSON](#json).

---

## D

<a id="dbcontext"></a>
**DbContext**
Classe base di [Entity Framework Core](#ef-core) che rappresenta la sessione con il database. La nostra `ApplicationDbContext` eredita da essa e definisce i [`DbSet<T>`](#dbsett) per ogni tabella.

<a id="dbsett"></a>
**DbSet\<T\>**
Proprietà di [`DbContext`](#dbcontext) che rappresenta una tabella del database. Permette di scrivere query [LINQ](#linq) che [EF Core](#ef-core) traduce in SQL. Es: `_context.Utenti.Where(u => u.Email == email)`.

<a id="dns"></a>
**DNS** *(Domain Name System)*
Sistema che traduce i nomi di dominio (es. `gestione-spese.azurewebsites.net`) in indirizzi IP. È il primo passo nel flusso di una richiesta [HTTP](#http).

<a id="dto"></a>
**DTO** *(Data Transfer Object)*
Oggetto usato esclusivamente per trasportare dati tra client e server, separato dai modelli del database. Permette di esporre solo i campi necessari (nascondendo es. `PasswordHash`) e di restituire strutture personalizzate.

---

## E

<a id="ef-core"></a>
**EF Core** *(Entity Framework Core)*
[ORM](#orm) (Object-Relational Mapper) ufficiale di Microsoft per .NET. Traduce operazioni su oggetti C# in [query SQL parametrizzate](#query-parametrizzata), gestisce le relazioni tra entità e offre il sistema di [migrazioni](#migrazione-ef-core) per evolvere lo schema del database.

<a id="ensurecreated"></a>
**`EnsureCreated()`**
Metodo di [EF Core](#ef-core) che crea il database e le tabelle se non esistono, basandosi direttamente sul modello C# senza usare le [migrazioni](#migrazione-ef-core). Usato nel progetto per il deploy su [Azure App Service](#azure-app-service) dove il file [SQLite](#sqlite) potrebbe non esistere.

<a id="endpoint"></a>
**Endpoint**
[URL](#url) specifico di un'[API](#api) che risponde a un determinato metodo [HTTP](#http). Es: `POST /api/Auth/login` è l'endpoint di autenticazione.

<a id="es-modules"></a>
**ES Modules**
Sistema nativo di import/export JavaScript (ES2015). [Vite](#vite) li usa durante lo sviluppo per non dover compilare l'intera app prima di avviarla, rendendo il server istantaneo.

---

## F

<a id="fetch"></a>
**`fetch()`**
[API](#api) Web nativa del browser per fare richieste [HTTP](#http) asincrone. Restituisce una [Promise](#promise). Usata in `api.js` per tutte le comunicazioni tra [React](#react) e il backend [ASP.NET Core](#aspnet-core).

<a id="free-f1"></a>
**Free F1**
Piano gratuito di [Azure App Service](#azure-app-service). Limitato a 60 minuti di CPU al giorno, 1GB di storage, nessun dominio personalizzato. Adeguato per progetti universitari e demo.

---

## G

<a id="github-actions"></a>
**GitHub Actions**
Servizio di [CI/CD](#cicd) integrato in GitHub. I workflow sono definiti in file [YAML](#yaml) nella cartella `.github/workflows/`. Il progetto usa `ci.yml` per verificare che backend e frontend compilino correttamente ad ogni push.

**`graph LR` / `graph TB`**
Direttive [Mermaid](#mermaid) per la direzione di un grafo: `LR` (Left to Right) e `TB` (Top to Bottom). Il diagramma architetturale usa `TB`, il diagramma di deploy usa `LR`.

---

## H

<a id="hash"></a>
**Hash**
Funzione matematica one-way che trasforma un input di lunghezza arbitraria in una stringa di lunghezza fissa. Non è reversibile: dall'hash non si può risalire alla password originale. [BCrypt](#bcrypt) produce hash del tipo `$2a$11$...`.

<a id="hateoas"></a>
**HATEOAS** *(Hypermedia As The Engine Of Application State)*
Vincolo del Livello 3 del [Richardson Maturity Model](#richardson-maturity-model): ogni risposta [API](#api) include link alle operazioni successive. Non implementato nel progetto (siamo al Livello 2).

<a id="headers-http"></a>
**Headers HTTP**
Metadati allegati a ogni richiesta o risposta [HTTP](#http). Esempi usati nel progetto: `Content-Type: application/json` (formato del [body](#body-http)), `Accept: application/json` (formato atteso in risposta).

<a id="hmr"></a>
**HMR** *(Hot Module Replacement)*
Funzionalità di [Vite](#vite) che aggiorna il browser in tempo reale al salvataggio di un file, senza ricaricare la pagina o perdere lo stato corrente dell'applicazione.

<a id="http"></a>
**HTTP** *(HyperText Transfer Protocol)*
Protocollo di comunicazione del Web, basato sul modello [Request-Response](#request-response). Definisce metodi (GET, POST, PUT, DELETE), [status code](#status-code-http) e [headers](#headers-http). Tutte le comunicazioni tra [React](#react) e [ASP.NET Core](#aspnet-core) avvengono tramite HTTP.

---

## I

<a id="idempotenza"></a>
**Idempotenza**
Proprietà di un'operazione: eseguirla più volte produce lo stesso risultato di eseguirla una sola volta. GET, PUT e DELETE sono idempotenti; POST non lo è (ogni chiamata crea una nuova risorsa). Vedi anche [REST](#rest).

---

## J

<a id="jsx"></a>
**JSX** *(JavaScript XML)*
Estensione sintattica di JavaScript che permette di scrivere HTML all'interno del codice JS. Compilato da [Vite](#vite) in chiamate `React.createElement()`. Esempio: `<LoginForm onLogin={handleLogin} />`.

<a id="json"></a>
**JSON** *(JavaScript Object Notation)*
Formato di serializzazione dati testuale basato sulla sintassi degli oggetti JavaScript. Usato per tutte le comunicazioni tra frontend e backend nel progetto. [ASP.NET Core](#aspnet-core) lo gestisce automaticamente tramite `System.Text.Json`.

---

## K

<a id="kestrel"></a>
**Kestrel**
Server [HTTP](#http) cross-platform integrato in [ASP.NET Core](#aspnet-core). Gestisce le connessioni in ingresso e passa le richieste alla [pipeline middleware](#pipeline-aspnet-core). In produzione su [Azure App Service](#azure-app-service), IIS funge da reverse proxy davanti a Kestrel.

---

## L

<a id="linq"></a>
**LINQ** *(Language Integrated Query)*
Sintassi C# per scrivere query su collezioni e database in modo type-safe. [EF Core](#ef-core) traduce le query LINQ in SQL. Es: `_context.Spese.Where(s => s.GruppoId == id).ToList()`.

<a id="localstorage"></a>
`localStorage`
Storage del browser per coppie chiave-valore che persistono tra sessioni. Usato nel progetto per salvare l'utente loggato (id, nome, email) e ripristinare la sessione al ricaricamento della pagina. Vedi anche [CSRF](#csrf).

---

## M

<a id="mermaid"></a>
**Mermaid**
Linguaggio testuale per creare diagrammi (grafi, ER, sequenze, Gantt) renderizzati automaticamente da GitHub nei file Markdown. Nel progetto usato in `docs/architettura.md` per i diagrammi di architettura, deploy e schema ER.

<a id="middleware"></a>
**Middleware**
Componenti della [pipeline](#pipeline-aspnet-core) di [ASP.NET Core](#aspnet-core) che processano la richiesta [HTTP](#http) in sequenza prima che arrivi al controller. Ogni middleware può leggere/modificare la richiesta, passarla al successivo o cortocircuitare la pipeline. Esempi: [CORS](#cors), routing, autenticazione, [Swagger](#swagger).

<a id="migrazione-ef-core"></a>
**Migrazione (EF Core)**
File C# generato da `dotnet ef migrations add` che descrive una modifica allo schema del database. Ha un metodo `Up()` (applica) e `Down()` (annulla). `dotnet ef database update` applica le migrazioni pendenti. Vedi anche [`EnsureCreated()`](#ensurecreated).

<a id="mpa"></a>
**MPA** *(Multi-Page Application)*
Applicazione web tradizionale in cui ogni navigazione causa un ricaricamento completo della pagina dal server. Contrapposta alla [SPA](#spa).

<a id="mvc"></a>
**MVC** *(Model-View-Controller)*
Pattern architetturale che separa la logica in tre componenti: Model (dati), View (presentazione), Controller (logica di business). [ASP.NET Core](#aspnet-core) segue questo pattern: i controller ricevono le richieste, interrogano il modello tramite [EF Core](#ef-core) e restituiscono i dati.

---

## N

<a id="nuget"></a>
**NuGet**
Gestore di pacchetti per .NET, equivalente di npm per JavaScript. I pacchetti sono elencati nel file `.csproj` e scaricati con `dotnet restore`.

<a id="npm-ci"></a>
**`npm ci`**
Comando npm per installare dipendenze in modo deterministico, usando esattamente le versioni nel [`package-lock.json`](#package-lockjson) senza modificarlo. Obbligatorio in ambienti [CI/CD](#cicd) per garantire build riproducibili.

---

## O

<a id="openapi"></a>
**OpenAPI**
Specifica standard per descrivere le [API](#api) [REST](#rest) in formato [JSON](#json)/[YAML](#yaml). [Swagger](#swagger) è lo strumento più diffuso per generare e visualizzare documentazione OpenAPI. [ASP.NET Core](#aspnet-core) genera automaticamente la specifica OpenAPI dai controller.

<a id="orm"></a>
**ORM** *(Object-Relational Mapper)*
Libreria che mappa le classi C# alle tabelle del database relazionale, traducendo operazioni sugli oggetti in query SQL. [Entity Framework Core](#ef-core) è l'ORM usato nel progetto.

---

## P

<a id="packagejson"></a>
**`package.json`**
File di configurazione del progetto Node.js. Elenca le dipendenze ([React](#react), [Vite](#vite), [Tailwind CSS](#tailwind-css)), gli script (`dev`, `build`, `preview`) e i metadati del progetto.

<a id="package-lockjson"></a>
**`package-lock.json`**
File generato automaticamente da npm che registra la versione esatta di ogni dipendenza installata, incluse le dipendenze transitive. Garantisce che [`npm ci`](#npm-ci) installi sempre le stesse versioni su qualsiasi macchina.

<a id="pipeline-aspnet-core"></a>
**Pipeline (ASP.NET Core)**
Sequenza di [middleware](#middleware) che elabora ogni richiesta [HTTP](#http). L'ordine in [`Program.cs`](#programcs) è critico: ad esempio, [CORS](#cors) deve precedere il routing, altrimenti le richieste preflight OPTIONS vengono rifiutate prima di raggiungere la policy CORS.

<a id="promise"></a>
**Promise**
Oggetto JavaScript che rappresenta il risultato futuro di un'operazione asincrona. Ha tre stati: `pending` (in attesa), `fulfilled` (completata con successo), `rejected` (fallita). [`fetch()`](#fetch) restituisce una Promise. Vedi anche [`async/await`](#asyncawait).

<a id="programcs"></a>
**`Program.cs`**
File di entry point di [ASP.NET Core](#aspnet-core). Configura i servizi (DI container, [EF Core](#ef-core), [CORS](#cors), [Swagger](#swagger)) e la [pipeline middleware](#pipeline-aspnet-core). L'ordine delle chiamate `Use*` è fondamentale per il corretto funzionamento dell'applicazione.

---

## Q

<a id="query-parametrizzata"></a>
**Query Parametrizzata**
Query SQL in cui i valori utente vengono passati come parametri separati (`@p0`, `@p1`) invece di essere concatenati nella stringa SQL. Rende impossibile la [SQL Injection](#sql-injection). [EF Core](#ef-core) usa sempre query parametrizzate.

---

## R

<a id="rainbow-table"></a>
**Rainbow Table**
Tabella precalcolata che mappa [hash](#hash) a password note. Usata negli attacchi alle password hashate senza [salt](#salt). [BCrypt](#bcrypt) la neutralizza perché ogni password ha un salt casuale unico che rende inutile qualsiasi tabella precalcolata.

<a id="react"></a>
**React**
Libreria JavaScript per costruire interfacce utente basate su componenti. Gestisce il [Virtual DOM](#virtual-dom), aggiornando solo le parti della pagina effettivamente cambiate. È una libreria (non un framework): gestisce solo la UI. Vedi anche [JSX](#jsx), [SPA](#spa).

<a id="request-response"></a>
**Request-Response**
Modello fondamentale di [HTTP](#http): il client invia una richiesta (metodo + [URL](#url) + [headers](#headers-http) + [body](#body-http) opzionale) e il server risponde ([status code](#status-code-http) + headers + body opzionale). È **stateless**: ogni coppia è indipendente.

<a id="rest"></a>
**REST** *(Representational State Transfer)*
Stile architetturale definito da Roy Fielding nel 2000. Si basa su 6 vincoli: interfaccia uniforme, client-server, statelessness, cacheabilità, sistema a livelli, code-on-demand (opzionale). Non è un protocollo. Vedi anche [Richardson Maturity Model](#richardson-maturity-model).

<a id="richardson-maturity-model"></a>
**Richardson Maturity Model**
Modello che classifica le [API](#api) [REST](#rest) in 4 livelli (0-3). Livello 0: [HTTP](#http) come tunnel RPC. Livello 1: risorse [URL](#url) distinte. Livello 2: metodi HTTP semantici + [status code](#status-code-http). Livello 3: [HATEOAS](#hateoas). Il progetto è al **Livello 2**.

---

## S

<a id="salt"></a>
**Salt**
Stringa casuale aggiunta alla password prima dell'hashing per rendere unico ogni [hash](#hash). [BCrypt](#bcrypt) lo genera automaticamente e lo incorpora nell'hash risultante, quindi non va salvato separatamente. Vedi anche [Rainbow Table](#rainbow-table).

<a id="spa"></a>
**SPA** *(Single Page Application)*
Applicazione web che carica una sola pagina HTML e aggiorna il contenuto dinamicamente tramite JavaScript, senza ricaricare la pagina. Il progetto usa [React](#react) come SPA. Contrapposta alla [MPA](#mpa).

<a id="sqlite"></a>
**SQLite**
Database relazionale serverless: un singolo file `.db` senza processo server separato. Usato nel progetto per semplicità di configurazione e deploy. Il file è `gestione-spese.db`. Vedi anche [`EnsureCreated()`](#ensurecreated).

<a id="sql-injection"></a>
**SQL Injection**
Attacco che inserisce codice SQL nei campi di input per manipolare le query del database. Neutralizzato da [EF Core](#ef-core) tramite [query parametrizzate](#query-parametrizzata).

<a id="status-code-http"></a>
**Status Code HTTP**
Codice numerico nella risposta [HTTP](#http) che indica l'esito dell'operazione. Gruppi: 2xx (successo), 3xx (redirect), 4xx (errore client), 5xx (errore server). Vedi anche [Request-Response](#request-response).

<a id="swagger"></a>
**Swagger**
Toolchain per la documentazione interattiva delle [API](#api) [REST](#rest) basata su [OpenAPI](#openapi). Nel progetto, `Swashbuckle.AspNetCore` genera automaticamente l'interfaccia Swagger all'[URL](#url) `/swagger`.

---

## T

<a id="tailwind-css"></a>
**Tailwind CSS**
Framework CSS utility-first: le classi di stile vengono composte direttamente nell'HTML/[JSX](#jsx) invece di scrivere CSS separato. In produzione, il [tree-shaking](#tree-shaking) include solo le classi effettivamente usate.

<a id="tcp"></a>
**TCP** *(Transmission Control Protocol)*
Protocollo di trasporto affidabile su cui si basa [HTTP](#http). Prima di ogni richiesta HTTP/1.1 viene stabilita una connessione TCP tramite il [3-way handshake](#3-way-handshake-tcp) (SYN → SYN-ACK → ACK).

<a id="tree-shaking"></a>
**Tree-Shaking**
Processo di eliminazione del codice JavaScript (o CSS) non utilizzato durante il build di produzione. [Vite](#vite)/Rollup lo applica al JavaScript; PostCSS/[Tailwind CSS](#tailwind-css) lo applica al CSS.

---

## U

<a id="url"></a>
**URL** *(Uniform Resource Locator)*
Indirizzo univoco di una risorsa sul Web. In [REST](#rest), ogni risorsa ha un URL distinto: `/api/Gruppo`, `/api/Spesa/{id}`, `/api/Auth/login`. Vedi anche [Endpoint](#endpoint).

---

## V

<a id="vercel"></a>
**Vercel**
Piattaforma di hosting per frontend statici e applicazioni Next.js. Nel progetto ospita il frontend [React](#react) con auto-deploy ad ogni push su `main` tramite webhook [GitHub Actions](#github-actions).

<a id="virtual-dom"></a>
**Virtual DOM**
Rappresentazione in memoria del DOM reale usata da [React](#react). Quando lo stato cambia, React calcola il diff tra il Virtual DOM precedente e quello nuovo (riconciliazione) e aggiorna solo i nodi DOM effettivamente cambiati, rendendo gli aggiornamenti efficienti. Vedi anche [JSX](#jsx).

<a id="vite"></a>
**Vite**
Build tool moderno per progetti frontend. Usa [ES Modules](#es-modules) nativi in sviluppo (avvio istantaneo + [HMR](#hmr)) e Rollup in produzione ([bundle](#bundle) ottimizzati). Sostituisce [Webpack](#webpack)/Create React App nel progetto.

<a id="vite_api_url"></a>
**`VITE_API_URL`**
Variabile d'ambiente che contiene l'[URL](#url) base del backend. Deve iniziare con `VITE_` per essere esposta a runtime nel browser tramite `import.meta.env`. Definita in `.env.local` in locale e nella sezione `env:` del workflow [CI/CD](#cicd).

---

## W

<a id="webpack"></a>
**Webpack**
Bundler JavaScript di prima generazione, usato da Create React App. A differenza di [Vite](#vite), compila l'intera applicazione prima di avviarla, diventando lento su progetti grandi.

<a id="web-api"></a>
**Web API**
Applicazione server che espone funzionalità tramite [endpoint](#endpoint) [HTTP](#http). Nel progetto, [ASP.NET Core](#aspnet-core) ospita una Web API [REST](#rest) che risponde con [JSON](#json).

---

## X

<a id="xss"></a>
**XSS** *(Cross-Site Scripting)*
Attacco che inietta codice JavaScript malevolo in una pagina web per eseguirlo nel browser di altri utenti. [React](#react) neutralizza XSS nativo perché il [JSX](#jsx) escapa automaticamente i valori inseriti nelle espressioni `{}`.

---

## Y

<a id="yaml"></a>
**YAML** *(YAML Ain't Markup Language)*
Formato di serializzazione dati leggibile dall'uomo, basato sull'indentazione. Usato nel progetto per il file `.github/workflows/ci.yml` che definisce la pipeline [GitHub Actions](#github-actions).

---

## Z

<a id="3-way-handshake-tcp"></a>
**3-Way Handshake (TCP)**
Procedura di apertura di una connessione [TCP](#tcp) in tre passi: il client invia `SYN`, il server risponde `SYN-ACK`, il client conferma con `ACK`. Avviene prima di ogni richiesta [HTTP](#http)/1.1 verso il backend.
