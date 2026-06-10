# 📔 Manuale Tecnico Split Mate: Architettura e Sviluppo Full-Stack

Benvenuti nel manuale definitivo di **Split Mate**. Questo documento non è una semplice guida, ma un trattato tecnico progettato per fornire una comprensione profonda e accademica del progetto. Studiare questo manuale significa acquisire la capacità di navigare nel codice, comprendere le scelte architettoniche e saper esporre il progetto con la padronanza di chi lo ha concepito.

---

## 🏛️ Capitolo 1: Architettura del Sistema

### 1.1 Paradigma Full-Stack

Split Mate adotta un **paradigma architetturale disaccoppiato (Decoupled Architecture)**, suddividendo il sistema in due entità distinte che comunicano esclusivamente tramite protocolli standard:

1.  **Server-Side (Backend)**: Un'applicazione ASP.NET Core che funge da *Provider di Risorse*. Espone endpoint RESTful e gestisce la persistenza dei dati.
2.  **Client-Side (Frontend)**: Una Single-Page Application (SPA) sviluppata in React. È il *Consumatore di Risorse* che gestisce lo stato dell'interfaccia e l'esperienza utente.

#### 1.1.1 Configurazione del Backend (Program.cs)

Il file `Program.cs` è il punto di ingresso e configurazione dell'applicazione backend ASP.NET Core. Qui vengono definiti i servizi e la pipeline di richiesta HTTP. Elementi chiave includono:

*   **`AddControllersWithViews().AddJsonOptions(...)`**: Configura i controller MVC e gestisce la serializzazione JSON. L'opzione `ReferenceHandler.IgnoreCycles` è cruciale per prevenire errori di serializzazione dovuti a riferimenti circolari tra oggetti del modello (ad esempio, un `Gruppo` che contiene `Utenti` e `Utenti` che contengono `Gruppi`).
*   **`AddHttpClient()`**: Registra il servizio `HttpClient` per effettuare richieste HTTP verso risorse esterne, se necessario.
*   **`AddDbContext<ApplicationDbContext>(...)`**: Configura il contesto del database utilizzando Entity Framework Core. Viene specificato l'uso di SQLite come database, con il percorso del file `gestionespese.db` impostato in `C:\home`. Questo indica una strategia di persistenza pensata per ambienti come Azure App Service, dove `C:\home` è una directory persistente.
*   **`AddSwaggerGen()`**: Abilita la generazione della documentazione API interattiva tramite Swagger/OpenAPI. `CustomSchemaIds` e `ResolveConflictingActions` sono configurazioni per gestire la generazione di ID univoci per gli schemi e risolvere potenziali conflitti tra endpoint con lo stesso percorso ma metodi HTTP diversi.
*   **`AddCors(...)`**: Configura la politica CORS (Cross-Origin Resource Sharing) denominata "AllowAll". Questa politica è molto permissiva (`AllowAnyOrigin().AllowAnyHeader().AllowAnyMethod()`) e consente al frontend, che risiede su un dominio diverso, di effettuare richieste al backend. In un ambiente di produzione, questa politica dovrebbe essere più restrittiva per motivi di sicurezza.
*   **`context.Database.EnsureCreated()`**: Questo blocco di codice, eseguito all'avvio dell'applicazione, garantisce che il database e le tabelle siano creati se non esistono già. Eventuali errori durante questo processo vengono loggati in `C:\home\startup-log.txt`.
*   **Middleware**: La pipeline di richiesta include `UseDeveloperExceptionPage()` (per debug), `UseSwagger()` e `UseSwaggerUI()` (per la documentazione API), `UseRouting()`, `UseCors("AllowAll")`, `UseAuthorization()` e `MapControllers()`. L'ordine di questi middleware è fondamentale per il corretto funzionamento dell'applicazione.

### 1.2 Flusso dei Dati (Data Flow)

La comunicazione avviene tramite il protocollo **HTTP** utilizzando il formato **JSON** (JavaScript Object Notation) per lo scambio di dati.
*   **Request**: Il frontend invia verbi HTTP (`GET`, `POST`, `PUT`, `DELETE`) verso URL specifici.
*   **Response**: Il backend elabora la richiesta, interagisce con il database SQLite e restituisce un codice di stato (es. `200 OK`, `404 Not Found`) e i dati richiesti.

---

## 🗄️ Capitolo 2: Modellazione dei Dati (Il Database)

Il cuore della persistenza è **Entity Framework Core**, un ORM (Object-Relational Mapper) che permette di mappare classi C# in tabelle relazionali.

### 2.1 Schema Relazionale

Il database è composto da cinque entità principali interconnesse:

1.  **Utente**: L'entità centrale. Gestisce identità (`Email`, `Nome`) e sicurezza (`PasswordHash`).
2.  **Gruppo**: Un'entità collettiva che aggrega utenti. Possiede un `CodiceInvito` univoco generato tramite `Guid`.
3.  **Spesa**: Registra un evento economico. È legata a un "Pagatore" e a un "Gruppo".
4.  **DivisioneSpesa**: Una tabella di associazione "Granulare". Specifica quanto ogni singolo utente deve per una specifica spesa.
5.  **Riepilogo**: Memorizza lo stato finale dei debiti calcolati, permettendo di segnarli come "Saldati".

### 2.2 Codice: La Definizione del Modello (Esempio `Spesa.cs`)

```csharp
public class Spesa {
    [Key] // Chiave primaria autoincrementale
    public int Id { get; set; }

    [ForeignKey("Gruppo")] // Vincolo di integrità referenziale
    public int Gruppo_ID { get; set; }

    [Required] // Vincolo di obbligatorietà a livello DB
    [Range(0.01, double.MaxValue)]
    public decimal Importo { get; set; }

    public virtual ICollection<DivisioneSpesa> Divisioni { get; set; } // Proprietà di navigazione
}
```

#### 2.2.1 `ApplicationDbContext.cs`: Configurazione delle Relazioni

Il file `ApplicationDbContext.cs` estende `DbContext` di Entity Framework Core e definisce le entità del database (`DbSet`). La configurazione delle relazioni tra le entità avviene nel metodo `OnModelCreating`:

*   **Relazione Molti-a-Molti tra Utente e Gruppo**: `modelBuilder.Entity<Utente>().HasMany(u => u.Gruppi).WithMany(g => g.Utenti).UsingEntity(j => j.ToTable("UtenteGruppo"));` crea una tabella di join implicita `UtenteGruppo` per gestire la relazione molti-a-molti, dove un utente può far parte di più gruppi e un gruppo può avere più utenti.
*   **Relazioni Uno-a-Molti con `DeleteBehavior.Restrict`**: Per `Spesa` (con `Gruppo` e `UtenteChePaga`), `DivisioneSpesa` (con `Utente`) e `Riepilogo` (con `Gruppo`, `UtenteCheDeve`, `UtenteACuiDeve`), viene utilizzato `OnDelete(DeleteBehavior.Restrict)`. Questo impedisce la cancellazione di un'entità padre se esistono ancora entità figlie correlate, garantendo l'integrità referenziale del database.
*   **Relazione Uno-a-Molti con `DeleteBehavior.Cascade`**: Per `DivisioneSpesa` (con `Spesa`), viene utilizzato `OnDelete(DeleteBehavior.Cascade)`. Questo significa che quando una `Spesa` viene eliminata, tutte le `DivisioneSpesa` ad essa correlate verranno automaticamente eliminate.

---

## ⚙️ Capitolo 3: Logica di Business e Controller

I Controller sono i direttori d'orchestra del backend. Utilizzano la **Dependency Injection** per accedere al contesto del database.

### 3.1 L'Algoritmo di Bilanciamento (`GruppoController.cs`)

Questa è la funzione tecnicamente più complessa. L'obiettivo è minimizzare le transazioni.
1.  **Fase di Aggregazione**: Viene creato un dizionario `saldi` dove per ogni utente si somma quanto ha pagato e si sottrae quanto deve (estratto da `DivisioneSpesa`).
2.  **Fase di Classificazione**: Gli utenti vengono divisi in due liste: `debitori` (saldo negativo) e `creditori` (saldo positivo), ordinate per entità del saldo.
3.  **Fase di Matching (Greedy Algorithm)**: Il sistema accoppia il maggior debitore con il maggior creditore, risolvendo il debito e aggiornando i saldi residui finché la lista non è vuota.

### 3.2 Gestione Autenticazione (`AuthController.cs`)

Split Mate utilizza un approccio di **Auto-Provisioning**:
*   Se un utente tenta il login con una mail non esistente, il sistema crea automaticamente l'account.
*   Le password sono gestite tramite **BCrypt**, un algoritmo di hashing adattivo che include il "salt" per prevenire attacchi di tipo Rainbow Table.

#### 3.2.1 Dettagli di Implementazione di `AuthController.cs`

Il controller `AuthController` gestisce le operazioni di autenticazione e registrazione. L'endpoint `POST api/Auth/login` è il fulcro di questa logica:

*   **Validazione Input**: Verifica che `Email` e `Password` non siano vuote.
*   **Ricerca Utente**: Cerca un utente esistente nel database tramite l'email (case-insensitive).
*   **Auto-Provisioning**: Se l'utente non esiste, viene creato un nuovo account. Il `Nome` viene derivato dall'email e la `PasswordHash` viene generata utilizzando `BCrypt.HashPassword(request.Password)`.
*   **Verifica Password**: Se l'utente esiste, la password fornita viene verificata con `BCrypt.Verify(request.Password, utente.PasswordHash)`. In caso di mancata corrispondenza, viene restituito un errore `Unauthorized`.
*   **Inclusione Gruppi**: Durante il login, l'oggetto `Utente` restituito include anche i `Gruppi` a cui l'utente appartiene, ottimizzando le successive chiamate al frontend.

L'endpoint `GET api/Auth/exists?email=...` permette di verificare l'esistenza di un'email nel sistema, utile per la validazione in tempo reale nel frontend.

### 3.3 Ciclo di Vita di una Spesa e Logica di Divisione (`SpesaController.cs`)

Il `SpesaController.cs` è responsabile della gestione completa delle spese, dalla creazione alla modifica e cancellazione, inclusa la complessa logica di divisione.

*   **`PostSpesa([FromBody] NuovaSpesaDTO dto)`**: Crea una nuova spesa. Se `dto.UtentiCoinvoltiIds` è nullo o vuoto, la spesa viene divisa equamente tra tutti i membri del gruppo; altrimenti solo tra gli utenti specificati.
*   **`PutSpesa(int id, [FromBody] NuovaSpesaDTO dto)`**: Modifica una spesa esistente. Rimuove le divisioni precedenti e le ricalcola completamente.
*   **`DeleteSpesa(int id)`**: Cancella una spesa. Grazie a `DeleteBehavior.Cascade`, le divisioni vengono eliminate automaticamente.

---

## ⚛️ Capitolo 4: Il Frontend (React & Vite)

### 4.1 Gestione dello Stato Globale

In `App.jsx`, lo stato dell'utente è persistito nel `localStorage`. Questo permette all'utente di rimanere loggato anche dopo il refresh della pagina, migliorando la **User Retention**.

#### 4.1.1 `App.jsx`: Il Componente Radice e il Routing Condizionale

*   **Stato Utente Persistente**: `useState(() => { ... })` inizializza lo stato `utente` leggendo dal `localStorage`.
*   **`handleLogin`**: Gestisce sia il login che la registrazione tramite chiamata `POST` a `/Auth/login`.
*   **Rendering Condizionale**: L'interfaccia cambia in base allo stato di autenticazione e alla navigazione tra gruppi.

### 4.2 Componenti e Hooks

Il frontend sfrutta gli **Hooks** di React (`useState`, `useEffect`) per gestire il ciclo di vita dei dati:
*   `useEffect`: Utilizzato per scatenare il recupero dei dati dal server non appena un componente viene visualizzato.
*   **Rendering Condizionale**: L'interfaccia cambia dinamicamente in base allo stato.

#### 4.2.1 `DettaglioGruppo.jsx`: Gestione delle Interazioni di Gruppo

*   **Caricamento Dati**: Il `useEffect` si attiva quando `gruppoId` cambia, chiamando `caricaDatiGruppo`.
*   **Aggiunta Membri (Bot)**: Genera email fittizie per i bot (`bot_{GUID}@bot.local`).
*   **Eliminazione Gruppo**: Previa conferma, elimina l'intero gruppo.
*   **Rimozione Membri**: Un membro non può essere rimosso se ha spese registrate (`eCoinvolto`), rispecchiando il `DeleteBehavior.Restrict` del backend.

### 4.3 Visualizzazione Dati (SVG & Math)

I grafici (`GraficoTorta.jsx`, `GraficoIstogramma.jsx`) sono disegnati manualmente tramite **SVG (Scalable Vector Graphics)**, sfruttando trigonometria (`Math.cos`, `Math.sin`) per calcolare i punti delle fette.

---

## 🛡️ Capitolo 5: Sicurezza e Best Practices

1.  **CORS**: Configurato nel `Program.cs` per permettere comunicazioni autorizzate tra frontend e backend.
2.  **Password Hashing**: Utilizzo di `BCrypt.Net` per garantire che le credenziali siano inattaccabili anche in caso di violazione del database.
3.  **Integrità Referenziale**: Utilizzo di `DeleteBehavior.Restrict` in Entity Framework per impedire la cancellazione accidentale di dati correlati.

---

## 📚 Capitolo 6: Glossario Avanzato per l'Esperto

*   **Middleware**: Software che si inserisce nel ciclo di richiesta/risposta del server.
*   **DTO (Data Transfer Object)**: Classi "leggere" usate per trasportare solo i dati necessari tra client e server.
*   **Dependency Injection**: Pattern che permette di passare le dipendenze alle classi invece di crearle internamente.
*   **Async/Await**: Programmazione asincrona che permette al server di gestire migliaia di richieste senza bloccarsi.
*   **YAML**: Formato testuale per file di configurazione, usa l'indentazione per definire strutture gerarchiche.
*   **CI/CD**: Continuous Integration / Continuous Deployment — automazione del build, test e deploy del codice.
*   **Mermaid**: Linguaggio per creare diagrammi tramite testo, supportato nativamente da GitHub.

---

## 🌐 Capitolo 7: Layer di Comunicazione Frontend e DTO Backend

### 7.1 `api.js`: Il Modulo di Interazione con il Backend

Il file `api.js` centralizza tutte le chiamate HTTP verso il backend. Ogni funzione corrisponde a un'operazione specifica sull'API RESTful:

*   **`login(email, password)`**: `POST` a `/Auth/login`.
*   **`getGruppiUtente(utenteId)`**: `GET` a `/Gruppo/utente/${utenteId}`.
*   **`creaSpesa(dati)`**: `POST` a `/Spesa` con body `NuovaSpesaDTO`.
*   **`saldaDebito(riepilogoId)`**: `PUT` a `/Riepilogo/${riepilogoId}/Saldato`.

### 7.2 `NuovaSpesaDTO.cs`: Data Transfer Object per la Creazione di Spese

```csharp
public class NuovaSpesaDTO
{
    public int Gruppo_ID { get; set; }
    public int ChiPaga_ID { get; set; }
    [Required]
    [Range(0.01, double.MaxValue)]
    public decimal Importo { get; set; }
    public string? Descrizione { get; set; }
    public List<int> UtentiCoinvoltiIds { get; set; } = new List<int>();
}
```

---

## 📊 Capitolo 8: Visualizzazioni Dati Avanzate nel Frontend

### 8.1 `GraficoIstogramma.jsx`: Implementazione con SVG

Visualizza l'ammontare totale delle spese per membro tramite barre SVG. Calcola `barH` in proporzione al `maxValore`, usa `<rect>` con angoli arrotondati e genera dinamicamente linee della griglia e etichette.

---

## 📚 Capitolo 9: Gestione dello Stato e Interazione Utente

### 9.1 `ModalNuovaSpesa.jsx` e `ModalModificaSpesa.jsx`

Componenti modali riutilizzabili che gestiscono form controllati per la creazione e modifica delle spese. Costruiscono l'array `UtentiCoinvoltiIds` per la divisione selettiva e chiamano le funzioni `creaSpesa`/`modificaSpesa` da `api.js`.

---

## 🎓 Capitolo 10: Simulazione Esame Orale - Domande e Risposte Tecniche

### 10.1 Domande sull'Architettura

**D: Qual è il paradigma architetturale adottato?**
R: Architettura Disaccoppiata (Decoupled Architecture) con backend ASP.NET Core e frontend React separati, che comunicano tramite API RESTful.

**D: Perché SQLite e non SQL Server?**
R: SQLite è ideale per progetti dimostrativi per la sua portabilità (file singolo `.db`). In produzione con carichi elevati si userebbe PostgreSQL o SQL Server.

**D: Le implicazioni di CORS AllowAnyOrigin?**
R: In sviluppo è comodo, ma in produzione va ristretto a `WithOrigins("https://tuofrontend.com")` per il principio del privilegio minimo.

### 10.2 Domande sulla Logica di Business

**D: Complessità dell'algoritmo di bilanciamento?**
R: O(N log N) per l'ordinamento + O(N) per il matching greedy, dove N è il numero di utenti.

**D: Pro e contro dell'auto-provisioning?**
R: Pro: riduce l'attrito dell'onboarding. Contro: meno controllo e sicurezza, nessuna verifica email.

### 10.3 Domande su Best Practices

**D: Cosa sono i DTO e perché usarli?**
R: Oggetti leggeri per il trasferimento dati, evitano l'esposizione del modello di dominio completo (sicurezza, performance, flessibilità).

**D: Vantaggi di async/await?**
R: Il thread del server non si blocca in attesa di I/O, permettendo di gestire migliaia di richieste concorrenti con risorse limitate.

---

## 🛠️ Capitolo 11: Gestione dello Schema, Ambiente e Principi SOLID

### 11.1 Migrazioni EF Core

Il file `ApplicationDbContextModelSnapshot.cs` rappresenta lo stato corrente del modello. Le migrazioni garantiscono la sincronizzazione tra codice e database in tutti gli ambienti.

### 11.2 Configurazione dell'Ambiente

*   **Frontend (`vite.config.js`)**: Usa `resolve.dedupe` per forzare una singola istanza di React, prevenendo errori da versioni duplicate.
*   **Backend (`appsettings.json`)**: Il percorso del DB viene sovrascritto in `Program.cs` con `C:\home` per Azure App Service.

### 11.3 Principi SOLID applicati

*   **SRP**: Ogni classe ha una sola responsabilità (Controller = HTTP, Model = dati, Context = persistenza).
*   **OCP**: Aggiungere un nuovo grafico non richiede modifiche a `DettaglioGruppo`.
*   **DIP**: I controller dipendono da astrazioni iniettate dal framework, non da istanze concrete.

---

## 🚀 Capitolo 12: Evoluzione e Roadmap Futura

1.  **Autenticazione JWT e OAuth**: Registrazione con verifica email e login tramite Google/GitHub.
2.  **Architettura a Microservizi**: Suddivisione del backend per scalabilità massiva.
3.  **Real-time con SignalR**: Aggiornamenti istantanei quando un membro aggiunge una spesa.
4.  **Test automatizzati**: xUnit per il backend, Cypress/Playwright per E2E sul frontend.
5.  **PWA**: Installazione su mobile, supporto offline e notifiche push.

---

## 📋 Capitolo 13: Documentazione della Consegna — Cosa Abbiamo Aggiunto

Questa sezione documenta in dettaglio tutti gli artefatti aggiunti al repository `gestore-spese` per soddisfare i requisiti formali della consegna accademica, spiegandone il significato teorico e pratico.

### 13.1 Il File YAML e la Pipeline CI/CD (`.github/workflows/ci.yml`)

#### Cos'è YAML

**YAML** (acronimo ricorsivo di *YAML Ain't Markup Language*) è un formato di serializzazione dei dati leggibile dall'uomo, progettato per la configurazione di sistemi software. A differenza di XML o JSON, usa l'**indentazione** per definire la struttura gerarchica, senza parentesi o virgole.

Esempio di sintassi:
```yaml
# Scalari
nome: Lorenzo
eta: 22

# Lista
hobbies:
  - calcio
  - programmazione

# Oggetto annidato
indirizzo:
  citta: Camerino
  cap: 62032
```

YAML è lo standard de facto per i file di configurazione di strumenti DevOps come **GitHub Actions**, **Docker Compose**, **Kubernetes** e **Ansible**.

#### Cos'è CI/CD

**CI/CD** sta per **Continuous Integration / Continuous Deployment**:

| Sigla | Significato | Cosa fa in pratica |
|---|---|---|
| **CI** (Continuous Integration) | Integrazione Continua | Ad ogni `git push`, esegue automaticamente build e test per verificare che il codice non sia rotto |
| **CD** (Continuous Deployment) | Deploy Continuo | Se CI passa, rilascia automaticamente il codice in produzione |

Il vantaggio principale è l'**early detection**: i problemi vengono scoperti subito dopo ogni commit, non il giorno prima della consegna.

#### Il Nostro File `.github/workflows/ci.yml`

```yaml
name: CI - Build & Test

on:
  push:
    branches: [ main ]      # Si attiva ad ogni push su main
  pull_request:
    branches: [ main ]

jobs:
  backend:                  # Job 1: verifica il backend
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4         # Scarica il codice
      - uses: actions/setup-dotnet@v4     # Installa .NET 10
        with:
          dotnet-version: '10.0.x'
      - run: dotnet restore gestione-spese/gestione-spese.csproj
      - run: dotnet build ...             # Verifica che compili

  frontend:                 # Job 2: verifica il frontend
    runs-on: ubuntu-latest
    steps:
      - uses: actions/setup-node@v4       # Installa Node 20
      - run: npm ci                       # Installa dipendenze
      - run: npm run build                # Verifica che compili
```

**Come funziona**: ogni volta che viene fatto un `git push` sul branch `main`, GitHub esegue i due job **in parallelo** su macchine virtuali Ubuntu. Se uno dei due fallisce (errore di compilazione, dipendenza mancante, ecc.), GitHub invia una notifica email e mostra una ❌ sul commit. I risultati sono visibili su [github.com/lorenzograssiUni/gestore-spese/actions](https://github.com/lorenzograssiUni/gestore-spese/actions).

**Il badge nel README**: la riga `![CI](https://github.com/lorenzograssiUni/gestore-spese/actions/workflows/ci.yml/badge.svg)` mostra dinamicamente lo stato della pipeline direttamente nel README — verde se passa, rosso se fallisce.

---

### 13.2 I Diagrammi Architetturali (`docs/architettura.md`)

#### Cos'è Mermaid

**Mermaid** è un linguaggio basato su testo per creare diagrammi e grafici. Il vantaggio rispetto a immagini statiche è che i diagrammi sono **codice**: possono essere versionati con Git, modificati facilmente e vengono renderizzati automaticamente da GitHub, GitLab e molti editor.

GitHub supporta nativamente Mermaid nei file Markdown da agosto 2022: basta includere un blocco di codice con linguaggio `mermaid`.

#### Diagramma 1: Architettura Generale

Mostra il flusso completo di una richiesta utente attraverso il sistema:

```
Browser (React/Vercel)
        |
        | HTTP REST (JSON)
        ↓
ASP.NET Core Web API (Azure)
        |
        | ├── AuthController
        | ├── GruppoController
        | ├── SpesaController
        | ├── UtenteController
        | └── RiepilogoController
        |
        | Entity Framework Core
        ↓
SQLite Database
```

Tipo di diagramma usato: `graph TB` (Top to Bottom — dall'alto verso il basso).

#### Diagramma 2: Architettura di Deploy

Mostra come il codice passa dallo sviluppatore alla produzione:

```
Developer → git push → GitHub
                          |
                          ├── trigger automatico → Vercel (frontend)
                          |
                          └── (manuale) Publish VS → Azure App Service (backend)
```

Tipo di diagramma usato: `graph LR` (Left to Right — da sinistra a destra).

**Dettaglio importante**: il frontend usa il **deploy automatico** (ogni push su `main` aggiorna Vercel), mentre il backend richiede un **deploy manuale** tramite Visual Studio. Questo è un limite architetturale che potrebbe essere migliorato aggiungendo un workflow GitHub Actions per il deploy su Azure.

#### Diagramma 3: Schema ER del Database

Usa la notazione `erDiagram` di Mermaid per mostrare le entità e le relazioni:

```
UTENTI ||--o{ UTENTE_GRUPPO : "appartiene a"
GRUPPI ||--o{ UTENTE_GRUPPO : "contiene"
GRUPPI ||--o{ SPESE : "ha"
UTENTI ||--o{ SPESE : "paga"
SPESE  ||--o{ DIVISIONISPESA : "divisa in"
UTENTI ||--o{ DIVISIONISPESA : "deve"
```

La notazione `||--o{` significa: *uno e solo uno* (lato sinistro) verso *zero o molti* (lato destro), ovvero una relazione **1:N**.

**Nota tecnica**: le emoji nei label dei `subgraph` causano un parse error in GitHub Mermaid. La soluzione è usare solo testo ASCII puro nei titoli dei sottografi.

---

### 13.3 Aggiornamenti al README

Il `README.md` è il biglietto da visita del progetto: è la prima cosa che vede il professore (o qualsiasi visitatore del repository). Un README ben fatto deve permettere il **build from scratch** — ovvero a chiunque di clonare il repo ed eseguire l'app partendo da zero.

#### Cosa è stato aggiunto

**Credenziali di prova**: una tabella con email e password di un account pre-registrato permette al professore di accedere immediatamente senza dover creare un account.

**Istruzioni build locale** — divise in tre fasi:

1. *Clona il repository*: `git clone ...`
2. *Avvia il backend*:
```bash
cd gestione-spese
dotnet restore          # Scarica le dipendenze NuGet
dotnet ef database update  # Crea e aggiorna il database SQLite
dotnet run              # Avvia il server su localhost:5207
```
3. *Avvia il frontend*:
```bash
cd frontend-gestione-spese
npm install             # Scarica le dipendenze Node
npm run dev             # Avvia il dev server su localhost:5173
```

**File `.env.local`**: il frontend usa una variabile d'ambiente `VITE_API_URL` per sapere dove trovare il backend. In produzione punta ad Azure; in locale va sovrascritta con:
```
VITE_API_URL=http://localhost:5207/api
```

**Struttura del repository**: un albero visivo delle cartelle principali con descrizione di ogni directory, per orientarsi rapidamente nel codice.

**Indice documentazione**: una tabella con link diretti ai file di documentazione aggiunti (`docs/architettura.md`, `.github/workflows/ci.yml`), così il professore li trova senza dover esplorare il repository.

**Badge CI**: mostra lo stato della pipeline in tempo reale direttamente nel README.

---

### 13.4 Come Visualizzare e Modificare il Database

Il database SQLite è un **file binario** con estensione `.db`. Non è leggibile direttamente, ma esistono diversi strumenti:

#### In locale

| Strumento | Tipo | Pro |
|---|---|---|
| **DB Browser for SQLite** | App desktop gratuita | Interfaccia grafica completa, modifica diretta dei dati |
| **SQLite Viewer** (VS Code) | Estensione | Integrato nell'editor, sola lettura |
| **SQLite CLI** | Terminale | Leggero, permette query SQL dirette |

#### In produzione (Azure)

Il database in produzione si trova su Azure App Service nella directory `site/wwwroot/`. Per accedervi:

1. **Kudu** (strumento integrato in Azure): `portal.azure.com` → App Service → Strumenti avanzati → Debug console → naviga in `site/wwwroot/` → scarica il file `.db`
2. **Swagger**: senza scaricare nulla, le API esposte permettono di leggere e modificare i dati tramite browser su `/swagger`

**Differenza importante**: il database locale (`.db` nella cartella del progetto) è separato da quello in produzione su Azure. Le modifiche fatte in locale non si riflettono in produzione e viceversa.

---

### 13.5 Riepilogo Requisiti di Consegna

La tabella seguente mostra come ogni requisito formale della consegna sia stato soddisfatto:

| Requisito | File/Risorsa | Dove trovarlo |
|---|---|---|
| README chiaro e completo | `README.md` | Root del repository |
| Funzionalità e architettura | `README.md` sezione Stack e Funzionalità | Root del repository |
| Istruzioni build from scratch | `README.md` sezione "Build e Avvio in Locale" | Root del repository |
| Credenziali di prova | `README.md` sezione "Credenziali di Prova" | Root del repository |
| Diagramma architettura | `docs/architettura.md` | Cartella `docs/` |
| Diagramma deploy | `docs/architettura.md` | Cartella `docs/` |
| Schema database | `docs/architettura.md` | Cartella `docs/` |
| File YAML di configurazione | `.github/workflows/ci.yml` | Cartella `.github/workflows/` |
| Pipeline CI/CD | `.github/workflows/ci.yml` | Cartella `.github/workflows/` |
| Link al deployment live | `README.md` sezione "Demo Live" | Root del repository |

---

## 🎯 Conclusione Finale per l'Espositore

Con questa guida completa, hai ora a disposizione tutti gli strumenti concettuali e tecnici per presentare il progetto "Split Mate" a un livello accademico elevato. Ricorda di:

*   **Articolare le scelte di design**: Ogni decisione tecnica deve essere giustificata con pro e contro.
*   **Dimostrare comprensione del codice**: Non limitarti a descrivere cosa fa il codice, ma spiega *perché* è stato scritto in quel modo.
*   **Enfatizzare la robustezza e la sicurezza**: Integrità dei dati, hashing password, CORS.
*   **Mostrare la visione d'insieme**: Collega frontend, backend e database in un flusso coerente.
*   **Prepararti alle domande**: Usa la sezione di simulazione per anticipare le obiezioni.

Buona fortuna per la tua esposizione!

---
*Questo manuale è una risorsa viva, basata sull'analisi riga per riga del codice sorgente di Split Mate.*
