# Capitolo 7 — Controller e Logica di Business

## Introduzione

In ASP.NET Core, i **Controller** sono le classi che ricevono le richieste HTTP, eseguono la logica di business e restituiscono una risposta al client. Ogni controller del progetto corrisponde a una risorsa del dominio (Utenti, Gruppi, Spese, Riepiloghi) e segue il pattern **MVC** già descritto nel Capitolo 2: la richiesta arriva dal frontend React, il controller la elabora, interroga il database tramite Entity Framework Core e restituisce un JSON.

Tutti i controller del progetto si trovano nella cartella `gestione-spese/Controllers/` ed ereditano da `ControllerBase` (non da `Controller`, perché non servono le View MVC — è una Web API pura). Ogni classe è decorata con:

```csharp
[Route("api/[controller]")]
[ApiController]
```

L'attributo `[ApiController]` abilita automaticamente la validazione del model binding, la gestione degli errori 400 e l'inferenza della sorgente dei parametri (`[FromBody]`, `[FromQuery]`, ecc.).

---

## 7.1 AuthController — Registrazione e Login

### Responsabilità

`AuthController` gestisce l'accesso al sistema. Espone due endpoint: `/api/Auth/login` per accedere con un account esistente e `/api/Auth/register` per crearne uno nuovo.

### Scelta di Design: Auto-Provisioning

> ⚠️ **Nota importante per la presentazione**: nel progetto l'autenticazione è **semplificata**. Non viene usata una password con hashing BCrypt (nonostante sia previsto dall'indice della guida): l'utente accede tramite email soltanto. Questo è un compromesso consapevole per mantenere il progetto semplice a livello universitario. Se il professore chiede, è corretto spiegarlo come "prototipo accademico" dove la sicurezza è stata deliberatamente semplificata.

Il sistema implementa di fatto un pattern di **auto-provisioning**: `UtenteController` espone anche un endpoint `/api/Utente/login` che, se l'email non esiste ancora nel database, crea automaticamente l'utente:

```csharp
// POST: api/Utente/login  (auto-provisioning)
var utente = await _context.Utenti
    .Include(u => u.Gruppi)
    .FirstOrDefaultAsync(u => u.Email.ToLower() == request.Email.ToLower());

if (utente == null)
{
    utente = new Utente
    {
        Email = request.Email,
        Nome = request.Email.Split('@')[0]  // nome derivato dall'email
    };
    _context.Utenti.Add(utente);
    await _context.SaveChangesAsync();
}

return Ok(utente);
```

Questo approccio unifica login e registrazione in un unico gesto: se l'email non esiste, l'utente viene creato al volo. Il nome predefinito viene estratto dalla parte locale dell'email (prima della `@`).

### Endpoint di AuthController

`AuthController` offre invece i due endpoint separati:

**Login** — `POST /api/Auth/login`

```csharp
[HttpPost("login")]
public async Task<ActionResult<Utente>> Login([FromBody] LoginRequest request)
{
    if (string.IsNullOrEmpty(request.Email))
        return BadRequest("L'email è obbligatoria.");

    var utente = await _context.Utenti
        .Include(u => u.Gruppi)
        .FirstOrDefaultAsync(u => u.Email == request.Email);

    if (utente == null)
        return Unauthorized("Utente non trovato. Registrati prima di accedere.");

    return Ok(utente);
}
```

**Registrazione** — `POST /api/Auth/register`

```csharp
[HttpPost("register")]
public async Task<ActionResult<Utente>> Register([FromBody] RegisterRequest request)
{
    var esistente = await _context.Utenti
        .FirstOrDefaultAsync(u => u.Email == request.Email);

    if (esistente != null)
        return Conflict("Un utente con questa email esiste già.");

    var utente = new Utente
    {
        Email = request.Email,
        Nome = string.IsNullOrEmpty(request.Nome)
            ? request.Email.Split('@')[0]
            : request.Nome
    };

    _context.Utenti.Add(utente);
    await _context.SaveChangesAsync();

    return Ok(utente);
}
```

Il metodo `Register` restituisce `409 Conflict` se l'email è già presente, proteggendo dall'inserimento di duplicati.

### DTO di Input

I due record di richiesta sono classi interne al file:

```csharp
public class LoginRequest
{
    public string Email { get; set; }
    public string Password { get; set; }
}

public class RegisterRequest
{
    public string Email { get; set; }
    public string Password { get; set; }
    public string Nome { get; set; }
}
```

Il campo `Password` è presente nelle classi ma non viene usato nella logica corrente (come da scelta progettuale descritta sopra).

---

## 7.2 GruppoController — CRUD, Codice Invito e Algoritmo di Bilanciamento

### Responsabilità

`GruppoController` è il controller più ricco del progetto. Gestisce:
- CRUD completo dei gruppi
- Ingresso tramite codice invito
- Rimozione di un membro dal gruppo
- Aggiunta di un utente "bot" (membro fittizio senza account)
- **L'algoritmo di bilanciamento dei debiti** (endpoint `/Bilanci`)

### CRUD Gruppi

| Metodo | Route | Descrizione |
|--------|-------|-------------|
| `GET` | `/api/Gruppo` | Tutti i gruppi con utenti e spese incluse |
| `GET` | `/api/Gruppo/{id}` | Un singolo gruppo |
| `GET` | `/api/Gruppo/miei-gruppi/{utenteId}` | Solo i gruppi di cui fa parte un utente |
| `POST` | `/api/Gruppo` | Crea un gruppo (con codice invito automatico) |
| `PUT` | `/api/Gruppo/{id}` | Aggiorna un gruppo |
| `DELETE` | `/api/Gruppo/{id}` | Elimina gruppo con tutte le spese e divisioni |

### Generazione del Codice Invito

Alla creazione di un nuovo gruppo, se non viene fornito un codice, il sistema ne genera uno automaticamente:

```csharp
if (string.IsNullOrEmpty(nuovoGruppo.CodiceInvito))
{
    nuovoGruppo.CodiceInvito = Guid.NewGuid()
        .ToString("N")
        .Substring(0, 6)
        .ToUpper();
}
```

`Guid.NewGuid()` genera un identificatore univoco universale a 128 bit. `.ToString("N")` lo serializza senza trattini (es. `a3f2c1...`). Prendiamo i primi 6 caratteri e li convertiamo in maiuscolo → un codice breve come `A3F2C1`, facile da condividere.

### Join tramite Codice Invito

`POST /api/Gruppo/join?codice=A3F2C1&utenteId=5`

```csharp
var gruppo = await _context.Gruppi
    .Include(g => g.Utenti)
    .FirstOrDefaultAsync(g => g.CodiceInvito.ToLower() == codice.ToLower());

if (gruppo.Utenti.Any(u => u.Id == utenteId))
    return BadRequest("Sei già membro di questo gruppo.");

gruppo.Utenti.Add(utente);
await _context.SaveChangesAsync();
```

La ricerca del codice è **case-insensitive** (`.ToLower()` su entrambi i lati). Prima di aggiungere il membro, si verifica che non sia già nel gruppo.

### Rimozione di un Membro

`DELETE /api/Gruppo/{id}/rimuovi-membro/{utenteId}`

Rimuove l'utente dalla collection di navigazione `Utenti` del gruppo. Grazie alla relazione many-to-many gestita da EF Core, questo aggiorna automaticamente la tabella di join intermedia senza eliminare l'utente dal sistema.

### Aggiunta di un Bot

`POST /api/Gruppo/{id}/aggiungi-bot`

Permette di aggiungere un utente "fittizio" al gruppo, utile per registrare spese che coinvolgono persone non registrate nella piattaforma. Se l'email del bot non è fornita, viene generata automaticamente con un UUID parziale:

```csharp
botUser.Email = $"bot_{Guid.NewGuid().ToString().Substring(0, 6)}@bot.local";
```

### 🧮 L'Algoritmo di Bilanciamento dei Debiti

`GET /api/Gruppo/{id}/Bilanci`

Questo è il cuore del progetto dal punto di vista algoritmico. L'obiettivo è: dato un gruppo con N spese, calcolare **il numero minimo di trasferimenti** per pareggiare tutti i conti.

#### Step 1 — Calcolo dei saldi

Per ogni utente del gruppo si calcola un saldo netto:

```
saldo(utente) = quanto ha pagato - quanto gli è stato addebitato
```

```csharp
var saldi = new Dictionary<int, decimal>();
foreach (var u in utenti) saldi[u.Id] = 0;

foreach (var spesa in spese)
{
    saldi[spesa.ChiPaga_ID] += spesa.Importo;      // chi ha pagato guadagna credito

    foreach (var div in spesa.Divisioni)
    {
        saldi[div.Utente_ID] -= div.Importo;        // chi ha ricevuto accumula debito
    }
}
```

Risultato: chi ha `saldo > 0` è **creditore** (gli devono dei soldi), chi ha `saldo < 0` è **debitore** (deve dei soldi ad altri).

#### Step 2 — Algoritmo Greedy di minimizzazione

```csharp
var debitori  = saldi.Where(s => s.Value < -0.01m).OrderBy(s => s.Value).ToList();
var creditori = saldi.Where(s => s.Value > 0.01m).OrderByDescending(s => s.Value).ToList();

int i = 0, j = 0;
while (i < debitori.Count && j < creditori.Count)
{
    var debito  = Math.Abs(debitori[i].Value);
    var credito = creditori[j].Value;
    var importoDaSaldare = Math.Min(debito, credito);

    // ... crea o recupera il Riepilogo nel DB ...

    debitori[i]  = new KeyValuePair<int, decimal>(debitori[i].Key,  -(debito  - importoDaSaldare));
    creditori[j] = new KeyValuePair<int, decimal>(creditori[j].Key,  credito  - importoDaSaldare);

    if (Math.Abs(debitori[i].Value)  < 0.01m) i++;
    if (creditori[j].Value           < 0.01m) j++;
}
```

L'algoritmo usa due puntatori (`i` su debitori, `j` su creditori) che avanzano a turno:

1. Prende il debitore con il debito più alto e il creditore con il credito più alto
2. Calcola l'importo minimo tra i due (`Math.Min`)
3. Registra un trasferimento di quella cifra
4. Scala i saldi di conseguenza
5. Avanza il puntatore di chi ha raggiunto saldo zero

La soglia `0.01m` evita errori di arrotondamento con i decimali.

#### Step 3 — Persistenza nel DB

Per ogni trasferimento calcolato, l'algoritmo controlla se esiste già un record `Riepilogo` nel database per quella coppia (debitore, creditore, gruppo). Se non esiste, lo crea. Questo garantisce che i debiti siano **persistenti** e possano essere segnati come pagati.

---

## 7.3 SpesaController — Creazione, Modifica e Logica di Divisione

### Responsabilità

`SpesaController` gestisce il ciclo di vita completo di una spesa: creazione con divisione automatica, modifica con ricalcolo delle quote, cancellazione.

### Il DTO `NuovaSpesaDTO`

Per la creazione e modifica delle spese, il controller non accetta direttamente l'entità `Spesa` ma un **DTO** (Data Transfer Object) specifico:

```csharp
// NuovaSpesaDTO (usato da POST e PUT)
// - Gruppo_ID
// - ChiPaga_ID
// - Importo
// - Descrizione
// - UtentiCoinvoltiIds  (lista opzionale di ID)
```

Il campo chiave è `UtentiCoinvoltiIds`: se la lista è vuota o assente, la spesa viene divisa tra **tutti i membri del gruppo**; altrimenti solo tra gli utenti indicati.

### POST — Creazione con Divisione Automatica

`POST /api/Spesa`

```csharp
var nuovaSpesa = new Spesa
{
    Gruppo_ID   = dto.Gruppo_ID,
    ChiPaga_ID  = dto.ChiPaga_ID,
    Importo     = dto.Importo,
    Descrizione = dto.Descrizione,
    DataSpesa   = System.DateTime.Now
};
_context.Spese.Add(nuovaSpesa);
await _context.SaveChangesAsync();  // ← necessario per ottenere l'ID

// Determina gli utenti da addebitare
List<Utente> utentiDaAddebitare;
if (dto.UtentiCoinvoltiIds == null || !dto.UtentiCoinvoltiIds.Any())
    utentiDaAddebitare = gruppo.Utenti.ToList();
else
    utentiDaAddebitare = gruppo.Utenti
        .Where(u => dto.UtentiCoinvoltiIds.Contains(u.Id))
        .ToList();

// Calcola e salva le quote
decimal quota = Math.Round(nuovaSpesa.Importo / utentiDaAddebitare.Count, 2);
foreach (var utente in utentiDaAddebitare)
{
    _context.Divisioni.Add(new DivisioneSpesa
    {
        Spesa_ID  = nuovaSpesa.Id,
        Utente_ID = utente.Id,
        Importo   = quota
    });
}
await _context.SaveChangesAsync();
```

> **Nota**: il `SaveChangesAsync()` intermedio è **necessario** perché `nuovaSpesa.Id` viene popolato dal database solo dopo il primo salvataggio. Le `DivisioneSpesa` hanno bisogno di questo ID come chiave esterna.

La quota per ciascun utente si calcola con `Math.Round(..., 2)` per evitare cifre decimali infinite (es. 10 / 3 = 3.33 invece di 3.3333...).

### PUT — Modifica con Ricalcolo

`PUT /api/Spesa/{id}`

Quando si modifica una spesa, le vecchie divisioni vengono **eliminate e ricalcolate da zero**:

```csharp
// Elimina tutte le divisioni vecchie
_context.Divisioni.RemoveRange(spesa.Divisioni);

// Ricalcola con i nuovi dati
decimal quota = Math.Round(spesa.Importo / utentiDaAddebitare.Count, 2);
foreach (var utente in utentiDaAddebitare)
{
    _context.Divisioni.Add(new DivisioneSpesa { ... });
}
await _context.SaveChangesAsync();
```

Questo approccio è detto **delete-and-recreate**: è più semplice da implementare rispetto a una diff granulare e garantisce coerenza totale.

### Endpoint Spesa

| Metodo | Route | Descrizione |
|--------|-------|-------------|
| `GET` | `/api/Spesa` | Tutte le spese |
| `GET` | `/api/Spesa/{id}` | Una spesa con divisioni incluse |
| `GET` | `/api/Spesa/Gruppo/{gruppoId}` | Tutte le spese di un gruppo |
| `POST` | `/api/Spesa` | Crea spesa con divisione automatica |
| `PUT` | `/api/Spesa/{id}` | Modifica spesa e ricalcola divisioni |
| `DELETE` | `/api/Spesa/{id}` | Elimina spesa (le divisioni in cascade) |

---

## 7.4 UtenteController — Gestione Profilo e Rimozione da Gruppo

### Responsabilità

`UtenteController` fornisce il CRUD base degli utenti più due endpoint specifici: il login con auto-provisioning (già descritto in 7.1) e l'aggiornamento del nome.

### Aggiornamento del Nome

`PUT /api/Utente/{id}/nome`

```csharp
public class UpdateNomeRequest
{
    public string NuovoNome { get; set; }
}

[HttpPut("{id}/nome")]
public async Task<IActionResult> AggiornaNome(int id, [FromBody] UpdateNomeRequest request)
{
    if (string.IsNullOrWhiteSpace(request.NuovoNome))
        return BadRequest("Il nome non può essere vuoto.");

    var utente = await _context.Utenti.FindAsync(id);
    if (utente == null)
        return NotFound("Utente non trovato.");

    utente.Nome = request.NuovoNome;
    await _context.SaveChangesAsync();

    return Ok(utente);
}
```

Notare l'uso di un **DTO** dedicato (`UpdateNomeRequest`) invece di passare l'intera entità `Utente`: questo è buona pratica REST perché espone solo i dati che possono essere modificati tramite quell'endpoint, senza rischi di sovrascrivere altri campi.

### Endpoint Utente

| Metodo | Route | Descrizione |
|--------|-------|-------------|
| `GET` | `/api/Utente` | Tutti gli utenti |
| `GET` | `/api/Utente/{id}` | Un utente con gruppi e spese pagate |
| `POST` | `/api/Utente` | Crea utente (senza gruppo) |
| `POST` | `/api/Utente/login` | Login con auto-provisioning |
| `PUT` | `/api/Utente/{id}` | Aggiorna l'utente (entità completa) |
| `PUT` | `/api/Utente/{id}/nome` | Aggiorna solo il nome |
| `DELETE` | `/api/Utente/{id}` | Elimina un utente |

---

## 7.5 RiepilogoController — Debiti Persistenti e Saldo

### Responsabilità

`RiepilogoController` gestisce i record `Riepilogo` che rappresentano i debiti calcolati dall'algoritmo in `GruppoController`. Il controller non ricalcola i debiti (lo fa il Bilanci endpoint), ma permette di:
- Leggere i debiti per un gruppo
- Creare un debito manualmente (con gestione degli accumulati)
- **Segnare un debito come saldato**
- Eliminare un record

### Segnare un Debito come Saldato

`PUT /api/Riepilogo/{id}/Saldato`

```csharp
[HttpPut("{id}/Saldato")]
public async Task<IActionResult> SegnaComeSaldato(int id)
{
    var riepilogo = await _context.Riepiloghi.FindAsync(id);
    if (riepilogo == null) return NotFound();

    riepilogo.Pagato = true;
    riepilogo.Importo = 0;

    _context.Entry(riepilogo).State = EntityState.Modified;
    await _context.SaveChangesAsync();

    return Ok(new { Messaggio = "Debito saldato con successo.", Riepilogo = riepilogo });
}
```

Quando un debito viene saldato, l'importo viene azzerato (`Importo = 0`) e il flag `Pagato` viene impostato a `true`. La risposta è un oggetto anonimo JSON con un messaggio e i dati aggiornati del riepilogo.

### Creazione con Accumulazione

`POST /api/Riepilogo`

Se esiste già un record Riepilogo tra la stessa coppia (debitore, creditore) nello stesso gruppo, il nuovo importo viene **sommato** a quello esistente invece di creare un duplicato:

```csharp
if (riepilogoEsistente != null)
{
    riepilogoEsistente.Importo += riepilogo.Importo;
    riepilogoEsistente.Pagato = false;      // reset del flag se si accumula
    _context.Entry(riepilogoEsistente).State = EntityState.Modified;
    await _context.SaveChangesAsync();
    return Ok(riepilogoEsistente);
}
```

### Validazioni a livello di dominio

Il controller include una validazione importante:

```csharp
if (riepilogo.ChiDeve_ID == riepilogo.AChiDeve_ID)
    return BadRequest("Un utente non può avere un debito verso se stesso.");
```

Questa è una regola di **business logic** (non di validazione dati): tecnicamente il JSON sarebbe valido, ma semanticamente non ha senso. Restituire `400 Bad Request` con un messaggio chiaro è la pratica corretta.

### Endpoint Riepilogo

| Metodo | Route | Descrizione |
|--------|-------|-------------|
| `GET` | `/api/Riepilogo` | Tutti i debiti |
| `GET` | `/api/Riepilogo/{id}` | Un singolo debito |
| `GET` | `/api/Riepilogo/Gruppo/{gruppoId}` | Debiti di un gruppo |
| `POST` | `/api/Riepilogo` | Crea o accumula un debito |
| `PUT` | `/api/Riepilogo/{id}/Saldato` | Segna come saldato |
| `DELETE` | `/api/Riepilogo/{id}` | Elimina un record |

---

## Schema Riassuntivo — Flusso Completo

```
Frontend React
     │
     │  POST /api/Utente/login  { email: "mario@..." }
     ▼
UtenteController ──► auto-provisioning ──► OK { id, nome, gruppi }
     │
     │  POST /api/Gruppo  { nome: "Vacanza" }  ?creatoreId=1
     ▼
GruppoController ──► genera CodiceInvito ──► CreatedAtAction 201
     │
     │  POST /api/Spesa  { gruppoId, chiPaga, importo, descrizione }
     ▼
SpesaController ──► crea Spesa ──► crea DivisioneSpesa per ogni membro
     │
     │  GET /api/Gruppo/{id}/Bilanci
     ▼
GruppoController ──► calcola saldi ──► algoritmo greedy ──► OK [ { da, a, importo } ]
     │
     │  PUT /api/Riepilogo/{id}/Saldato
     ▼
RiepilogoController ──► Pagato=true, Importo=0 ──► OK
```

---

## Punti Chiave per la Presentazione

1. **Dependency Injection**: ogni controller riceve `ApplicationDbContext` tramite il costruttore — non lo crea manualmente. È il container DI di ASP.NET Core che gestisce il ciclo di vita.

2. **async/await ovunque**: tutte le operazioni di I/O verso il database sono asincrone (`await _context.SaveChangesAsync()`). Questo libera il thread durante l'attesa, rendendo il server scalabile.

3. **Include per le relazioni**: EF Core non carica le navigation properties automaticamente (lazy loading disabilitato di default). Ogni `Include(...)` è una JOIN esplicita verso la tabella correlata.

4. **Status Code semantici**: il progetto restituisce i codici HTTP corretti — `201 Created` per le risorse create, `204 NoContent` per le cancellazioni, `409 Conflict` per i duplicati, `401 Unauthorized` per utente non trovato al login.

5. **Algoritmo Greedy**: l'algoritmo dei Bilanci minimizza il numero di transazioni necessarie, ed è dimostrabile che con N partecipanti produce al massimo N-1 trasferimenti (ottimale).
