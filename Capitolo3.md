# Capitolo 3 — Backend, Database e Sicurezza

> Questo capitolo copre la parte server-side di Split Mate: come funziona ASP.NET Core, come EF Core gestisce il database, e come proteggiamo i dati degli utenti da SQL Injection, hashing delle password e attacchi CORS/CSRF.

---

## 3.1 Cos'è il Backend e Cosa Fa

Il **backend** è la parte invisibile di un'applicazione web: gestisce i dati, la logica applicativa e risponde alle richieste HTTP del frontend. L'utente non lo vede mai direttamente, ma ogni azione che compie nell'interfaccia passa attraverso di esso.

Il backend di Split Mate (ASP.NET Core) si occupa di:

- **Rispondere alle richieste HTTP** dal frontend React (GET, POST, PUT, DELETE)
- **Autenticare l'utente** — verifica che chi chiede i dati sia chi dice di essere
- **Autorizzare le azioni** — verifica che quell'utente abbia il permesso di fare quella cosa
- **Comunicare con il database** — leggere e scrivere dati tramite Entity Framework Core
- **Restituire JSON** — mai HTML, solo dati strutturati che React trasforma in interfaccia

### Il Backend Non Usa Socket Direttamente

Come spiega il prof. Bonura, in teoria un backend risponde alle richieste HTTP tramite **socket** — connessioni TCP raw che ascoltano su una porta. Scrivere la gestione dei socket a mano è complesso e inefficiente:

```csharp
// Gestione socket "a mano" — quello che ASP.NET Core fa per noi sotto al cofano
var listener = new TcpListener(IPAddress.Any, 5000);
listener.Start();
while (true)
{
    var client = await listener.AcceptTcpClientAsync(); // bloccante fino a nuova connessione
    // leggi la richiesta HTTP, interpreta header/body, costruisci la risposta...
    // aprire e chiudere un socket ad ogni richiesta è oneroso anche per la CPU
}
```

**ASP.NET Core** (come Express.js per Node) gestisce tutta questa complessità per noi: socket, keep-alive, parsing HTTP, routing, middleware — tutto automatico.

---

## 3.2 Routing in ASP.NET Core

Il **routing** mappa gli URL in arrivo ai metodi dei controller. È l'equivalente di `app.get('/path', handler)` in Express.js.

In ASP.NET Core usiamo il routing basato su **attributi**:

```csharp
[ApiController]
[Route("api/[controller]")]  // → /api/Spesa
public class SpesaController : ControllerBase
{
    // GET /api/Spesa/gruppo/3
    [HttpGet("gruppo/{gruppoId}")]
    public async Task<IActionResult> GetSpese(int gruppoId) { ... }

    // POST /api/Spesa
    [HttpPost]
    public async Task<IActionResult> CreaSpesa([FromBody] NuovaSpesaDTO dto) { ... }

    // PUT /api/Spesa/7
    [HttpPut("{id}")]
    public async Task<IActionResult> ModificaSpesa(int id, [FromBody] ModificaSpesaDTO dto) { ... }

    // DELETE /api/Spesa/7
    [HttpDelete("{id}")]
    public async Task<IActionResult> EliminaSpesa(int id) { ... }
}
```

Il decoratore `[Route("api/[controller]")]` usa il nome del controller (`Spesa`) per costruire automaticamente il prefisso URL. `{gruppoId}` e `{id}` sono **parametri di route** che ASP.NET Core estrae dall'URL e passa al metodo.

---

## 3.3 Dependency Injection (DI)

La **Dependency Injection** è un pattern in cui le dipendenze di una classe (es. il contesto del database) vengono fornite dall'esterno invece di essere create dalla classe stessa.

In ASP.NET Core è integrata nel framework. Nel file `Program.cs` si **registrano** i servizi:

```csharp
// Program.cs — registrazione dei servizi nel container DI
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=gestionespese.db"));

builder.Services.AddScoped<IRiepilogoService, RiepilogoService>();
```

Poi nei controller si **iniettano** tramite costruttore:

```csharp
public class SpesaController : ControllerBase
{
    private readonly AppDbContext _context;

    // ASP.NET Core injetta automaticamente AppDbContext qui
    public SpesaController(AppDbContext context)
    {
        _context = context;
    }
}
```

**Perché è utile?**
- **Testabilità**: puoi sostituire il database reale con uno mock nei test unitari
- **Disaccoppiamento**: il controller non sa "come" si connette al DB, sa solo che può usarlo
- **Ciclo di vita gestito**: `AddScoped` crea una nuova istanza per ogni richiesta HTTP, poi la distrugge

---

## 3.4 Async/Await nel Backend

In un server che gestisce centinaia di richieste contemporanee, le operazioni di I/O (query al DB, chiamate a API esterne) sono le più lente. Se il thread del server si bloccasse ad aspettare ogni query, non potrebbe gestire altre richieste nel frattempo.

**`async/await`** risolve questo problema: il thread viene **rilasciato** mentre aspetta la risposta del database, e può servire altre richieste. Quando il risultato arriva, il thread riprende l'esecuzione.

```csharp
// ❌ Versione sincrona — blocca il thread per tutta la durata della query
public IActionResult GetSpese(int gruppoId)
{
    var spese = _context.Spese.Where(s => s.GruppoId == gruppoId).ToList();
    return Ok(spese);
}

// ✅ Versione asincrona — rilascia il thread durante la query
public async Task<IActionResult> GetSpese(int gruppoId)
{
    var spese = await _context.Spese
        .Where(s => s.GruppoId == gruppoId)
        .ToListAsync();
    return Ok(spese);
}
```

In Split Mate **tutti i metodi dei controller sono `async`** e usano le versioni `Async` dei metodi EF Core (`ToListAsync`, `FindAsync`, `SaveChangesAsync`).

---

## 3.5 DTO — Data Transfer Object

Un **DTO** è un oggetto pensato esclusivamente per trasportare dati tra frontend e backend. Non corrisponde 1:1 al modello del database — è una "vista" ridotta o specializzata.

**Perché non usare direttamente il modello EF Core?**

```csharp
// ❌ Modello EF Core — NON esporre questo direttamente nelle API
public class Spesa
{
    public int Id { get; set; }
    public string Descrizione { get; set; }
    public decimal Importo { get; set; }
    public int PagatoreId { get; set; }
    public int GruppoId { get; set; }
    public Utente Pagatore { get; set; }         // relazione → cicli JSON infiniti!
    public List<DivisioneSpesa> Divisioni { get; set; } // dati non necessari al frontend
}

// ✅ DTO — solo quello che serve al frontend
public class NuovaSpesaDTO
{
    public string Descrizione { get; set; }
    public decimal Importo { get; set; }
    public int PagatoreId { get; set; }
    public int GruppoId { get; set; }
    public List<int> PartecipiIds { get; set; }
}
```

**Vantaggi dei DTO in Split Mate:**
- Evitano i **cicli di serializzazione** JSON (Spesa → Utente → Spese → loop infinito)
- Espongono **solo i campi necessari** (sicurezza: non esponi mai password o token)
- Permettono di **validare** i dati in ingresso con `[Required]`, `[Range]`, ecc.

---

## 3.6 ORM e Entity Framework Core

### Cos'è un ORM?

Un **ORM** (Object-Relational Mapping) è uno strato software che traduce le operazioni sugli oggetti C# in query SQL. Invece di scrivere SQL a mano, scrivi codice C# e l'ORM genera e ottimizza le query.

**Senza ORM (SQL a mano):**
```sql
SELECT s.Id, s.Descrizione, s.Importo
FROM Spese s
WHERE s.GruppoId = 3
ORDER BY s.DataCreazione DESC;
```

**Con Entity Framework Core (LINQ):**
```csharp
var spese = await _context.Spese
    .Where(s => s.GruppoId == 3)
    .OrderByDescending(s => s.DataCreazione)
    .ToListAsync();
```

L'ORM genera lo stesso SQL ottimizzato, ma il codice è C# tipizzato — il compilatore cattura gli errori prima del runtime.

### DbContext — Il Cuore di EF Core

`AppDbContext` è la classe che rappresenta il database in C#. Ogni proprietà `DbSet<T>` corrisponde a una tabella:

```csharp
public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Utente> Utenti { get; set; }
    public DbSet<Gruppo> Gruppi { get; set; }
    public DbSet<Spesa> Spese { get; set; }
    public DbSet<DivisioneSpesa> DivisioniSpese { get; set; }
    public DbSet<Riepilogo> Riepiloghi { get; set; }

    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        modelBuilder.Entity<DivisioneSpesa>()
            .HasOne(d => d.Spesa)
            .WithMany(s => s.Divisioni)
            .HasForeignKey(d => d.SpesaId);
    }
}
```

### Le Relazioni nel Database di Split Mate

| Relazione | Tipo | Descrizione |
|-----------|------|-------------|
| Utente ↔ Gruppi | Many-to-Many | Un utente può essere in più gruppi, un gruppo ha più utenti |
| Gruppo → Spese | One-to-Many | Un gruppo ha molte spese |
| Spesa → DivisioneSpesa | One-to-Many | Una spesa è divisa tra più utenti |
| Spesa → Utente (pagatore) | Many-to-One | Ogni spesa ha un unico pagatore |
| Riepilogo | Derivato | Calcolato da spese e divisioni per mostrare i saldi |

### Migrations — Versionamento dello Schema

Le **migrations** sono la storia del database: ogni modifica allo schema genera un file C# che descrive la trasformazione. EF Core le applica in sequenza.

```bash
# Creare una migration dopo aver modificato il modello
dotnet ef migrations add AggiuntaColonnaNote

# Applicare la migration al database
dotnet ef database update
```

```csharp
// Esempio di migration generata automaticamente da EF Core
public partial class AggiuntaColonnaNote : Migration
{
    protected override void Up(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.AddColumn<string>(name: "Note", table: "Spese", nullable: true);
    }

    protected override void Down(MigrationBuilder migrationBuilder)
    {
        migrationBuilder.DropColumn(name: "Note", table: "Spese");
    }
}
```

---

## 3.7 SQLite — Il Database di Split Mate

Split Mate usa **SQLite** come database: un singolo file (`gestionespese.db`) che vive nello stesso processo dell'applicazione, senza server separato.

| Caratteristica | SQLite | SQL Server / PostgreSQL |
|----------------|--------|------------------------|
| Installazione | Nessuna — è un file | Server separato |
| Deployment | Il file va con l'app | Servizio DB indipendente |
| Scalabilità | Bassa (file singolo) | Alta (connessioni concorrenti) |
| Scritture concorrenti | Limitate (lock globale) | Ottimizzate |
| Uso ideale | Prototipo, app piccole | Produzione, alta concorrenza |

Se il progetto dovesse scalare, si sostituisce `UseSqlite()` con `UseSqlServer()` in `Program.cs` — il resto del codice rimane identico grazie a EF Core.

---

## 3.8 Sicurezza delle Password — BCrypt e il Salt

### Evoluzione delle Tecniche di Protezione

Come spiega il prof. Bonura, salvare le password in chiaro è la peggior pratica possibile. L'evoluzione storica:

| Livello | Tecnica | Vulnerabilità residua |
|---------|---------|----------------------|
| **1** | Password in chiaro | Chiunque legga il DB vede tutto |
| **2** | Hash MD5/SHA senza sale | Rainbow tables — hash precompilati di password comuni |
| **3** | Hash + Secret globale | Se trovano il Secret nel codice, rifanno tutta la tabella |
| **4** | Hash + **Salt per utente** | L'attaccante deve attaccare ogni utente separatamente |

### Come Funziona BCrypt

**BCrypt** implementa il Livello 4: genera automaticamente un **salt univoco** per ogni password e lo incorpora nell'hash risultante.

```csharp
// REGISTRAZIONE — hash della password
string hash = BCrypt.Net.BCrypt.HashPassword("miaPassword123", workFactor: 12);
// hash = "$2a$12$K.Do85pTHVmZg7..." ← il salt è già incorporato

// Salviamo SOLO l'hash nel database, mai la password originale
utente.PasswordHash = hash;

// LOGIN — verifica della password
bool isValid = BCrypt.Net.BCrypt.Verify("miaPassword123", utente.PasswordHash);
// BCrypt estrae il salt dall'hash, ri-hasha, confronta → true/false
```

Il **workFactor** (rounds) controlla la lentezza dell'hashing. Un valore di 12 significa `2^12 = 4096` iterazioni: hashare un milione di password richiede ore invece di secondi, rendendo il brute force impraticabile. Il prof. consiglia un valore tra 8 e 12.

> **Split Mate usa Google OAuth**: non gestiamo password dirette. Il principio BCrypt rimane fondamentale da conoscere per l'esame e per qualsiasi sistema con credenziali proprie.

---

## 3.9 SQL Injection — e Come EF Core ci Protegge

### Cos'è SQL Injection

La **SQL Injection** è una vulnerabilità che permette di iniettare codice SQL malevolo attraverso i campi input, manipolando le query del database.

**Esempio — login vulnerabile:**
```sql
-- Input attaccante come email: admin'--
SELECT * FROM utenti WHERE email = 'admin'--' AND password = 'qualsiasi'
-- Il -- commenta tutto il resto → accesso come admin senza password!

-- Input come password: password' OR '1'='1
SELECT * FROM utenti WHERE email = 'x' AND password = 'password' OR '1'='1'
-- '1'='1' è sempre vera → restituisce tutti gli utenti!
```

### Come EF Core Previene la SQL Injection

**EF Core usa automaticamente Parameterized Queries**: i valori utente vengono passati come **parametri separati**, mai concatenati alla stringa SQL.

```csharp
// ✅ EF Core — completamente protetto da SQL Injection
var utente = await _context.Utenti
    .FirstOrDefaultAsync(u => u.Email == emailDallUtente);
// EF Core genera: SELECT * FROM Utenti WHERE Email = @p0
// con @p0 = valore dell'email — mai concatenato nella stringa SQL

// ❌ PERICOLO — SQL grezzo con interpolazione di stringa
var risultato = await _context.Database
    .ExecuteSqlRawAsync($"SELECT * FROM Spese WHERE Note = '{noteDallUtente}'");

// ✅ SICURO — SQL grezzo con parametri
var risultato = await _context.Database
    .ExecuteSqlRawAsync("SELECT * FROM Spese WHERE Note = {0}", noteDallUtente);
```

---

## 3.10 CORS in ASP.NET Core

La configurazione CORS in Split Mate si trova in `Program.cs`:

```csharp
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowFrontend", policy =>
    {
        policy
            .WithOrigins("https://split-mate.vercel.app", "http://localhost:5173")
            .AllowAnyHeader()
            .AllowAnyMethod()
            .AllowCredentials();
    });
});

app.UseCors("AllowFrontend"); // prima di app.UseAuthorization()
```

**Perché `WithOrigins` e non `AllowAnyOrigin`?** Con `AllowAnyOrigin` (wildcard `*`), qualsiasi sito potrebbe fare richieste alla nostra API — inclusi siti malevoli che sfruttano la sessione attiva dell'utente (**attacco CSRF**). Specificando solo il dominio del nostro frontend, il browser blocca automaticamente tutte le altre origini.

---

## 3.11 Autenticazione con JWT

**JWT** (JSON Web Token) mantiene l'utente autenticato dopo il login. È un token firmato salvato nel `localStorage` e inviato in ogni richiesta all'API.

### Struttura di un JWT

| Parte | Contenuto | Modificabile? |
|-------|-----------|---------------|
| **Header** | Algoritmo di firma (`HS256`) | No — altera la firma |
| **Payload** | Dati utente (userId, email, exp) | No — altera la firma |
| **Signature** | HMAC(header + payload, secret) | Non senza il secret |

### Flusso di Autenticazione in Split Mate

```
1. FRONTEND → POST /api/Auth/login  { googleToken: "..." }
2. BACKEND verifica il token con i server Google
3. BACKEND cerca/crea l'utente nel DB
4. BACKEND genera JWT firmato  { sub: userId, email: "...", exp: now+7days }
5. BACKEND → 200 OK  { token: "eyJ..." }
6. FRONTEND salva il token in localStorage
7. Ogni richiesta: Authorization: Bearer eyJ...
8. BACKEND verifica firma → esegue o risponde 401 Unauthorized
```

### JWT vs Session Database

| Strategia | Come funziona | Pro | Contro |
|-----------|---------------|-----|--------|
| **JWT (stateless)** | Tutto nel token, nessuna query al DB | Veloce, scalabile | Non revocabile facilmente |
| **Session DB** | ID nel cookie, dati nel DB | Revocabile, aggiornabile | Query al DB ad ogni richiesta |

Split Mate usa **JWT**: per un'app dove i dati utente cambiano raramente, la velocità supera la flessibilità delle sessioni database.

---

## 3.12 Middleware Pipeline in ASP.NET Core

Il **middleware** è una catena di componenti che elaborano ogni richiesta HTTP in sequenza. L'ordine in `Program.cs` è fondamentale:

```
Richiesta in arrivo
        ↓
[HTTPS Redirection]   → forza HTTPS
        ↓
[CORS]                → aggiunge header Access-Control-*
        ↓
[Authentication]      → decodifica il JWT, popola User
        ↓
[Authorization]       → verifica i permessi [Authorize]
        ↓
[Routing → Controller] → esegue la logica di business
        ↓
Risposta al client
```

```csharp
// Program.cs — ordine corretto del middleware in Split Mate
app.UseHttpsRedirection();
app.UseCors("AllowFrontend");   // 1. CORS — prima di Authorization!
app.UseAuthentication();         // 2. Chi sei? (JWT → User)
app.UseAuthorization();          // 3. Cosa puoi fare?
app.MapControllers();            // 4. Esegui il controller
```

---

## 3.13 Swagger — Testing delle API

**Swagger** (OpenAPI) genera automaticamente una documentazione interattiva delle API, dove puoi testare ogni endpoint dal browser — come Postman, ma autogenerato dal codice.

```csharp
if (app.Environment.IsDevelopment())
{
    app.UseSwagger();
    app.UseSwaggerUI(); // → http://localhost:5000/swagger
}
```

Come sottolinea il prof.: *"Swagger è un tool di controllo delle chiamate REST in maniera grafica autocostruita, può sostituirsi a Postman."* Si aggiorna automaticamente quando aggiungi nuovi endpoint — nessuna documentazione da scrivere manualmente.

---

## Riepilogo Capitolo 3

| Concetto | Dove in Split Mate |
|----------|--------------------|
| Routing ASP.NET | Attributi `[HttpGet]`, `[HttpPost]` sui controller |
| Dependency Injection | `AppDbContext` iniettato nel costruttore dei controller |
| Async/Await | Tutti i metodi dei controller e le query EF Core |
| DTO | `NuovaSpesaDTO`, `ModificaSpesaDTO` per sicurezza e pulizia |
| EF Core + DbContext | `AppDbContext` con 5 `DbSet<T>`, LINQ per le query |
| Migrations | Storia versionata dello schema SQLite |
| SQLite | File `gestionespese.db` — semplice per il deploy su Azure |
| BCrypt | Principio hashing password con salt — auth delegata a Google |
| SQL Injection | Prevenuta automaticamente da EF Core (parameterized queries) |
| CORS | Configurato in `Program.cs` con origini specifiche |
| JWT | Token firmato per autenticazione stateless |
| Middleware Pipeline | Ordine: HTTPS → CORS → Auth → Authorization → Controllers |
| Swagger | Documentazione e test interattivo delle API in sviluppo |
