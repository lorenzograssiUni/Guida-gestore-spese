# Capitolo 8 — Configurazione e Deploy Backend

## Introduzione

In ASP.NET Core, tutta la configurazione dell'applicazione è centralizzata in un unico file: `Program.cs`. A partire da .NET 6, il modello di hosting è stato semplificato eliminando la classe `Startup.cs` separata: builder e pipeline convivono in un unico file lineare. Capire questo file è essenziale perché **l'ordine delle istruzioni non è arbitrario** — un middleware registrato fuori sequenza può causare errori silenziosi difficili da diagnosticare.

---

## 8.1 Program.cs — Struttura Generale

Il file `Program.cs` del progetto si divide in due fasi distinte:

```
fase 1: builder  ──►  registrazione dei servizi nel container DI
fase 2: app      ──►  configurazione della pipeline middleware HTTP
```

Questa separazione riflette il pattern **Builder** di .NET: prima si configura tutto ciò di cui l'app ha bisogno (`builder.Services.*`), poi si costruisce l'app (`var app = builder.Build()`) e infine si definisce come le richieste HTTP la attraversano (`app.Use*`, `app.Map*`).

```csharp
var builder = WebApplication.CreateBuilder(args);

// FASE 1 — Registrazione servizi
builder.Services.AddControllersWithViews()...;
builder.Services.AddHttpClient();
builder.Services.AddDbContext<ApplicationDbContext>(...);
builder.Services.AddEndpointsApiExplorer();
builder.Services.AddSwaggerGen(...);
builder.Services.AddCors(...);

var app = builder.Build();

// FASE 2 — Pipeline middleware
app.UseDeveloperExceptionPage();
app.UseSwagger();
app.UseSwaggerUI(...);
app.UseRouting();
app.UseCors("AllowAll");
app.UseAuthorization();
app.MapControllers();

app.Run();
```

---

## 8.2 Configurazione dei Servizi (builder.Services)

### Controller e Opzioni JSON

```csharp
builder.Services.AddControllersWithViews()
    .AddJsonOptions(options =>
    {
        options.JsonSerializerOptions.ReferenceHandler =
            System.Text.Json.Serialization.ReferenceHandler.IgnoreCycles;
    });
```

`AddControllersWithViews()` registra il supporto sia per le API REST (controller che restituiscono JSON) sia per le View MVC tradizionali. Nel nostro progetto si usano solo le API, ma il metodo va bene lo stesso.

L'opzione critica è `ReferenceHandler.IgnoreCycles`. Senza di essa, il serializzatore JSON entrerebbe in un **loop infinito** quando tenta di serializzare le entità con relazioni circolari. Ad esempio:

```
Gruppo → ha una lista di Utenti
Utente → ha una lista di Gruppi  ← torna a Gruppo → loop!
```

Con `IgnoreCycles`, quando il serializzatore incontra un oggetto già visitato nel grafo corrente, lo sostituisce con `null` anziché ripetere la serializzazione, spezzando il ciclo.

### HttpClient

```csharp
builder.Services.AddHttpClient();
```

Registra `IHttpClientFactory` nel container DI. Permette ai controller di effettuare chiamate HTTP verso servizi esterni in modo gestito (con pooling delle connessioni TCP e gestione dei timeout). Nel progetto attuale non è usato attivamente ma è registrato per possibili estensioni future.

### DbContext — SQLite con Percorso Azure

```csharp
// Percorso persistente su Azure App Service
var dbPath = Path.Combine("C:\\home", "gestionespese.db");

builder.Services.AddDbContext<ApplicationDbContext>(options =>
    options.UseSqlite($"Data Source={dbPath}"));
```

Questo è uno dei punti più importanti per il deploy su Azure. Il percorso `C:\home` è la **directory persistente** di Azure App Service su Windows: è l'unica cartella garantita a sopravvivere ai riavvii e ai re-deploy dell'applicazione. Tutto il resto del filesystem è effimero e può essere cancellato da Azure in qualsiasi momento.

Se avessimo usato il percorso relativo di default (solo `gestionespese.db`), il file SQLite verrebbe creato nella directory di lavoro del processo — che su Azure corrisponde a `D:\home\site\wwwroot` o simili, non garantita. Al prossimo deploy, i dati andrebbero persi.

> **In locale** il percorso `C:\home` potrebbe non esistere e causare un errore all'avvio. Per sviluppo locale si può creare manualmente la cartella oppure usare la connection string da `appsettings.json` (che punta al percorso relativo senza `C:\home`).

### Swagger / OpenAPI

```csharp
builder.Services.AddEndpointsApiExplorer();

builder.Services.AddSwaggerGen(c =>
{
    c.CustomSchemaIds(type => type.FullName);
    c.ResolveConflictingActions(apiDescriptions => apiDescriptions.First());
});
```

`AddEndpointsApiExplorer()` espone i metadati degli endpoint (necessario per Swagger con Minimal API e controller). `AddSwaggerGen()` genera il documento OpenAPI in formato JSON.

Le due opzioni custom risolvono due problemi concreti del progetto:

- **`CustomSchemaIds(type => type.FullName)`**: nel progetto ci sono classi con lo stesso nome in namespace diversi (es. `LoginRequest` esiste in `AuthController` e in `UtenteController`). Swagger di default genera uno schema con il nome breve causando conflitti. Usando il `FullName` (es. `gestione_spese.Controllers.Api.AuthController+LoginRequest`) si garantisce l'unicità.

- **`ResolveConflictingActions(...)`**: se due endpoint hanno la stessa signature HTTP (stesso metodo + stessa route), Swagger non sa quale documentare. Questa opzione lo forza a prendere il primo, evitando un'eccezione al build della documentazione.

### CORS — Cross-Origin Resource Sharing

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowAll", policy =>
    {
        policy.AllowAnyOrigin()
              .AllowAnyHeader()
              .AllowAnyMethod();
    });
});
```

Come spiegato nel Capitolo 4, CORS è un meccanismo di sicurezza del browser che blocca le richieste cross-origin per default. Il backend React (su Vercel, dominio `*.vercel.app`) e il backend ASP.NET (su Azure, dominio `*.azurewebsites.net`) sono **origini diverse**: senza questa policy, il browser rifiuterebbe ogni chiamata API.

La policy `"AllowAll"` è permissiva: accetta qualsiasi origine, header e metodo. È appropriata per un progetto universitario. In produzione reale si specificherebbe l'origine esatta:

```csharp
// Versione production-ready (esempio)
policy.WithOrigins("https://mia-app.vercel.app")
      .AllowAnyHeader()
      .AllowAnyMethod();
```

---

## 8.3 Pipeline Middleware — Ordine Critico

Dopo `builder.Build()`, si configura la **pipeline middleware**: la sequenza di funzioni che ogni richiesta HTTP attraversa in ordine, dall'ingresso all'uscita.

```csharp
app.UseDeveloperExceptionPage();   // 1. Mostra errori dettagliati
app.UseSwagger();                  // 2. Genera il documento JSON OpenAPI
app.UseSwaggerUI(...);             // 3. Serve l'interfaccia grafica Swagger
app.UseRouting();                  // 4. Analizza l'URL e trova il controller
app.UseCors("AllowAll");           // 5. Applica gli header CORS
app.UseAuthorization();            // 6. Controlla i permessi
app.MapControllers();              // 7. Collega le route ai controller
```

### Perché l'ordine conta

In ASP.NET Core, ogni `app.Use*` aggiunge un componente alla pipeline. Ogni componente può :
- Elaborare la richiesta e passarla al successivo (`next()`)
- Cortocircuitare la pipeline restituendo una risposta immediata

Se l'ordine è sbagliato, si rompono le funzionalità:

| Errore tipico | Causa |
|---|---|
| CORS bloccato nonostante la policy | `UseCors` messo **dopo** `UseRouting` in versioni precedenti, o dopo `MapControllers` |
| Swagger non funziona | `UseSwagger` messo dopo `UseAuthorization` che blocca le richieste non autenticate |
| 401 su tutte le rotte | `UseAuthorization` prima di `UseRouting` (non riesce a valutare i metadata della route) |

L'ordine corretto canonico in ASP.NET Core è:
```
Exception → StaticFiles → Routing → Cors → Auth → Endpoints
```

### EnsureCreated — Creazione automatica del database

Prima di avviare la pipeline, il codice garantisce che le tabelle esistano:

```csharp
try
{
    using (var scope = app.Services.CreateScope())
    {
        var context = scope.ServiceProvider
            .GetRequiredService<ApplicationDbContext>();
        context.Database.EnsureCreated();
    }
}
catch (Exception ex)
{
    File.AppendAllText("C:\\home\\startup-log.txt",
        $"[{DateTime.Now}] DB ERROR: {ex.Message}\n");
}
```

`EnsureCreated()` è un metodo di EF Core che:
- Se il database **non esiste** → lo crea con tutte le tabelle (basandosi sul modello)
- Se il database **esiste già** → non fa nulla

Il blocco `try/catch` è fondamentale su Azure: se la creazione del database fallisce (es. percorso `C:\home` non accessibile al primo avvio), l'errore viene scritto nel file `startup-log.txt` invece di far crashare l'intera applicazione silenziosamente. Questo rende il debug molto più agevole.

> **EnsureCreated vs Migrate**: `EnsureCreated()` crea il database da zero ma **non supporta le migrazioni** incrementali. Se si modifica il modello, bisogna cancellare il file `.db` e ricrearlo. `Database.Migrate()` invece applica le migrazioni in sequenza ed è la scelta per produzione. Nel progetto si usa `EnsureCreated` per semplicità (ambiente universitario).

---

## 8.4 appsettings.json

```json
{
  "ConnectionStrings": {
    "DefaultConnection": "Data Source=gestionespese.db"
  },
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedHosts": "*"
}
```

Il file `appsettings.json` è il file di configurazione standard di ASP.NET Core. In esso si definiscono valori configurabili senza toccare il codice.

Nel progetto la connection string `DefaultConnection` punta al percorso relativo `gestionespese.db` — usata in locale. Su Azure invece la connection string viene **sovrascritta** direttamente nel codice da `Program.cs` con il percorso `C:\home\gestionespese.db`, ignorando questo file.

`AllowedHosts: "*"` permette richieste da qualsiasi host (utile per lo sviluppo). In produzione si specificherebbe il dominio Azure (`*.azurewebsites.net`).

---

## 8.5 Il File .csproj — Pacchetti NuGet e Target Framework

```xml
<Project Sdk="Microsoft.NET.Sdk.Web">
  <PropertyGroup>
    <TargetFramework>net10.0</TargetFramework>
    <Nullable>enable</Nullable>
    <ImplicitUsings>enable</ImplicitUsings>
    <RootNamespace>gestione_spese</RootNamespace>
  </PropertyGroup>

  <ItemGroup>
    <PackageReference Include="BCrypt.Net-Next" Version="4.1.0" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Design" Version="10.0.5" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.InMemory" Version="10.0.3" />
    <PackageReference Include="Microsoft.EntityFrameworkCore.Sqlite" Version="10.0.5" />
    <PackageReference Include="Swashbuckle.AspNetCore" Version="10.1.5" />
  </ItemGroup>
</Project>
```

Il progetto usa **.NET 10.0** (ultima versione LTS al momento dello sviluppo). I pacchetti NuGet inclusi sono:

| Pacchetto | Versione | Scopo |
|---|---|---|
| `BCrypt.Net-Next` | 4.1.0 | Hashing adattivo delle password con salt (installato ma non attualmente usato nella logica) |
| `EF Core Design` | 10.0.5 | Strumenti per generare le migrazioni da riga di comando (`dotnet ef`) |
| `EF Core InMemory` | 10.0.3 | Provider database in-memory per i test |
| `EF Core Sqlite` | 10.0.5 | Provider SQLite per il database reale |
| `Swashbuckle.AspNetCore` | 10.1.5 | Generazione documentazione Swagger/OpenAPI |

L'opzione `<Nullable>enable</Nullable>` abilita i **nullable reference types** di C#: il compilatore avverte quando una variabile che potrebbe essere `null` viene usata senza controllo. `<ImplicitUsings>enable</ImplicitUsings>` aggiunge automaticamente gli `using` più comuni (come `System`, `System.Linq`, ecc.) senza doverli scrivere in ogni file.

---

## 8.6 Deploy su Azure App Service

### Architettura del Deploy

```
GitHub (main branch)
       │
       │  GitHub Actions (ci.yml)
       ▼
Azure App Service (Windows)
  ├── Runtime: .NET 10.0
  ├── OS: Windows (necessario per il percorso C:\home)
  └── Filesystem persistente: C:\home\gestionespese.db
```

### Il percorso C:\home su Azure

Azure App Service su Windows mappa `C:\home` a un **Azure Storage condiviso** (Azure Files) montato come drive di rete. Questo storage:
- Persiste tra i riavvii del container
- Persiste tra i re-deploy del codice
- È accessibile anche tramite **Kudu** (pannello di debug di Azure, raggiungibile da `https://<app>.scm.azurewebsites.net`)

Senza questo percorso, SQLite creerebbe il file `.db` nella cartella `wwwroot` o nella directory temporanea del processo, e al prossimo deploy il file andrebbe perso (reset completo dei dati).

### startup-log.txt — Debug su Azure

Il file di log creato alla startup serve per diagnosticare problemi che avvengono prima che il web server sia attivo (e quindi prima che Swagger o qualsiasi altra interfaccia sia raggiungibile).

```csharp
File.AppendAllText("C:\\home\\startup-log.txt",
    $"[{DateTime.Now}] DB ERROR: {ex.Message}\n");
```

Per leggerlo su Azure si può usare:
1. **Kudu Console** (`https://<app>.scm.azurewebsites.net/DebugConsole`) → navigare in `C:\home`
2. **Azure Portal** → App Service → Advanced Tools → Bash/CMD

### Differenze Locale vs Azure

| Aspetto | Locale (sviluppo) | Azure App Service |
|---|---|---|
| Percorso database | Relativo (cartella progetto) | `C:\home\gestionespese.db` |
| Creazione tabelle | `EnsureCreated()` al primo avvio | `EnsureCreated()` al primo deploy |
| HTTPS | Opzionale (HTTP in dev) | Obbligatorio (Azure forza redirect) |
| Swagger | Sempre visibile | Sempre visibile (nessun filtro per environment) |
| Log errori | Console di Visual Studio | `C:\home\startup-log.txt` + Azure Log Stream |

### UseDeveloperExceptionPage

```csharp
app.UseDeveloperExceptionPage();
```

Normalmente questo middleware si usa **solo in sviluppo** (dentro `if (app.Environment.IsDevelopment())`), perché mostra lo stack trace completo dell'errore nel browser — un rischio di sicurezza in produzione. Nel progetto è lasciato attivo anche su Azure perché agevola il debug durante la fase di sviluppo universitario. In un'app di produzione reale si sostituirebbe con `app.UseExceptionHandler("/Error")`.

---

## Schema Riassuntivo — Ciclo di Vita di una Richiesta HTTP

```
Client (React su Vercel)
         │
         │  GET https://gestione-spese.azurewebsites.net/api/Gruppo/1
         ▼
[1] DeveloperExceptionPage  ──►  (wrap per catturare eccezioni)
[2] Swagger middleware       ──►  (non è /swagger, passa oltre)
[3] UseRouting               ──►  identifica: GruppoController, action GetGruppo, id=1
[4] UseCors                  ──►  aggiunge header Access-Control-Allow-Origin: *
[5] UseAuthorization         ──►  nessun [Authorize] sulla route, passa
[6] MapControllers           ──►  invoca GruppoController.GetGruppo(1)
         │
         │  ApplicationDbContext.Gruppi.Include(...).FirstOrDefaultAsync()
         ▼
    SQLite (C:\home\gestionespese.db)
         │
         │  ritorna oggetto Gruppo
         ▼
    Serializzazione JSON (IgnoreCycles attivo)
         │
         │  200 OK  { "id": 1, "nome": "Vacanza", "utenti": [...] }
         ▼
Client React ──► aggiorna lo stato ──► re-render del componente
```

---

## Punti Chiave per la Presentazione

1. **Ordine middleware**: `UseRouting` prima di `UseCors` e `UseAuthorization` è obbligatorio. Se qualcosa non funziona, il primo posto dove guardare è l'ordine in `Program.cs`.

2. **ReferenceHandler.IgnoreCycles**: senza questa opzione il progetto non funzionerebbe — tutte le API che restituiscono entità con relazioni circolari darebbero errore 500.

3. **C:\home su Azure**: la scelta del percorso persistente è una decisione infrastrutturale consapevole, non casuale. Dimostra comprensione dell'ambiente di deploy.

4. **EnsureCreated vs Migrate**: `EnsureCreated` è semplice ma non supporta aggiornamenti incrementali. La scelta è consapevole per un prototipo universitario.

5. **CORS AllowAll**: accettabile per un progetto universitario. In produzione si restringe all'origine del frontend.
