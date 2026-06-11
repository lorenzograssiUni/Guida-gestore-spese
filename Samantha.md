# 🎤 Traccia Relatore 2 — Backend & Database

> **Capitoli di riferimento:** Cap. 3 (Backend, Database, Sicurezza), porzione Cap. 2 (REST, Dependency Injection)
> **Durata stimata:** ~4 minuti

---

## Apertura — Passaggio di consegne

> *"Ora entriamo nel dettaglio del backend. Il server di Split Mate è costruito con ASP.NET Core e si occupa di tutta la logica applicativa: autenticazione, gestione dei dati, calcolo dei saldi e comunicazione con il database."*

---

## 1. ASP.NET Core — Struttura dei Controller

Il backend espone un'**API RESTful** organizzata in 5 controller, ognuno responsabile di una risorsa del dominio:

| Controller | Responsabilità |
|------------|----------------|
| `AuthController` | Login e registrazione utenti |
| `GruppoController` | CRUD gruppi, codice invito, gestione membri |
| `SpesaController` | Aggiunta, lettura, modifica ed eliminazione spese |
| `RiepilogoController` | **Algoritmo di calcolo saldi** — chi deve quanto a chi |
| `UtenteController` | Modifica profilo, cambio nome |

Ogni metodo dei controller usa il routing per attributi:

```csharp
[HttpGet("gruppo/{gruppoId}")] // → GET /api/Spesa/gruppo/3
public async Task<IActionResult> GetSpese(int gruppoId) { ... }
```

Tutti i metodi sono **`async/await`**: il thread viene rilasciato mentre aspetta la risposta del database, e può servire altre richieste nel frattempo. In un server con molte connessioni simultanee questo fa una differenza enorme.

---

## 2. DTO — Perché non esponiamo direttamente i modelli

Non passiamo gli oggetti EF Core direttamente tra frontend e backend. Usiamo i **DTO (Data Transfer Object)**: classi pensate solo per trasportare dati, con esattamente i campi che servono.

**Motivi:**
- Evitano i **cicli di serializzazione** JSON: un oggetto `Spesa` ha un riferimento a `Utente`, che ha una lista di `Spese` → loop infinito
- **Sicurezza**: non esponiamo mai campi interni come `PasswordHash` o chiavi interne
- **Validazione**: sui DTO si mettono attributi come `[Required]` e `[Range]` per validare i dati prima di processarli

---

## 3. Dependency Injection

In ASP.NET Core la **Dependency Injection** è integrata nel framework. In `Program.cs` si registrano i servizi:

```csharp
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite("Data Source=gestionespese.db"));
```

Poi nei controller il `AppDbContext` viene **iniettato automaticamente** nel costruttore. Il vantaggio è la **testabilità**: nei test unitari possiamo sostituire il database reale con uno in-memory senza modificare i controller.

---

## 4. Entity Framework Core — ORM e Schema del Database

**EF Core** è l'ORM (Object-Relational Mapping) che traduce il codice C# in query SQL. Invece di scrivere SQL a mano, scriviamo LINQ:

```csharp
// EF Core genera automaticamente la query SQL corretta
var spese = await _context.Spese
    .Where(s => s.GruppoId == gruppoId)
    .ToListAsync();
```

Il database ha **4 entità principali** collegate tramite relazioni:

| Entità | Relazione |
|--------|-----------|
| `Utente` ↔ `Gruppo` | Many-to-Many: un utente è in più gruppi, un gruppo ha più utenti |
| `Gruppo` → `Spesa` | One-to-Many: un gruppo ha molte spese |
| `Spesa` → `DivisioneSpesa` | One-to-Many: una spesa è divisa tra più utenti |
| `Spesa` → `Utente` (pagatore) | Many-to-One: ogni spesa ha un unico pagatore |

Le **Migrations** versionano lo schema: ogni modifica al modello genera un file C# che descrive la trasformazione e che EF Core applica in sequenza con `dotnet ef database update`.

---

## 5. SQLite — Scelta del database

Split Mate usa **SQLite**: un singolo file (`gestionespese.db`) che vive nello stesso processo dell'app, senza server separato. È la scelta giusta per un prototipo: zero configurazione, zero deployment aggiuntivo, portabilità totale.

Il limite è la scalabilità: SQLite non gestisce bene le scritture concorrenti. Se il progetto dovesse crescere, sostituiamo `UseSqlite()` con `UseSqlServer()` in `Program.cs` — il resto del codice rimane identico grazie ad EF Core.

---

## 6. L'algoritmo di pareggio debiti

Questa è la parte più interessante del backend. Il `RiepilogoController` calcola i saldi con un **algoritmo greedy**:

1. Per ogni membro del gruppo, calcola il **saldo netto** = totale pagato − quota dovuta
2. Separa chi ha saldo positivo (**creditore**) da chi ha saldo negativo (**debitore**)
3. Abbina iterativamente il debitore con il debito maggiore al creditore con il credito maggiore
4. Genera la **lista minima di transazioni** necessarie per saldare tutti i conti

**Esempio**: Alice paga €60 per un gruppo di 3 (quota equa €20 cad.).
- Beatrice → Alice: €20
- Carlo → Alice: €20

Risultato: 2 transazioni — il minimo possibile. Senza questo algoritmo, in un gruppo di 10 persone potreste avere decine di pagamenti incrociati.

---

## 7. Sicurezza — SQL Injection e BCrypt

**SQL Injection**: EF Core usa automaticamente **query parametrizzate** — i valori dell'utente non vengono mai concatenati alla stringa SQL. L'attaccante non può iniettare codice malevolo.

**Hashing delle password con BCrypt**: il prof. Bonura ha spiegato l'evoluzione da password in chiaro → MD5 → hash + salt per utente. BCrypt implementa quest'ultimo livello: genera automaticamente un salt univoco per ogni utente, e lo incorpora nell'hash. Il **workFactor** (tipicamente 12 = 2^12 iterazioni) rende il brute-force computazionalmente impraticabile.

---

## 8. Middleware Pipeline

In `Program.cs` l'ordine del middleware è fondamentale:

```
HTTPS Redirection → CORS → Authentication → Authorization → Controllers
```

CORS deve stare **prima** di Authorization, altrimenti le preflight request (`OPTIONS`) vengono bloccate prima di raggiungere il layer CORS.

---

## Possibili domande del professore

**"Perché usare i DTO?"**
> Per evitare cicli di serializzazione JSON, per non esporre campi interni del database, e per validare i dati in ingresso con attributi `[Required]` e `[Range]`.

**"Come funzionano le migrations?"**
> Ogni modifica al modello C# genera un file di migration che descrive la trasformazione dello schema. EF Core li applica in sequenza al database con `dotnet ef database update`. È il versionamento del database.

**"Perché SQLite e non SQL Server?"**
> Per semplicità di deployment su Azure Free Tier: SQLite è un file, non richiede un servizio di database separato. Se il progetto dovesse scalare, basta cambiare una riga in `Program.cs` — il codice rimane identico.

**"Spiegami l'algoritmo dei saldi"**
> Calcola il saldo netto di ogni membro (pagato meno dovuto), separa creditori e debitori, e abbina greedily il maggior debitore col maggior creditore finché tutti i saldi non sono zero. Questo produce il numero minimo di transazioni.
