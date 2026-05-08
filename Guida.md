# Guida Completa al Progetto Gestore Spese

Questa guida è stata creata per fornire una comprensione approfondita del progetto **“Gestore Spese”**, una web application sviluppata con un backend **ASP.NET Core** e un frontend **React/Vite**. L’obiettivo è permettere a tutti i membri del gruppo di esporre il progetto in modo chiaro e dettagliato, coprendo l’architettura generale, la funzione di ogni componente e le sezioni di codice più rilevanti.

## 1. Architettura Generale

Il progetto “Gestore Spese” è una **Single-Page Application (SPA)** composta da due macro-aree principali che comunicano tramite **API RESTful**:

*   **Backend ASP.NET Core**: Residente nella cartella `gestione-spese`, è responsabile della logica di business, della persistenza dei dati e dell’esposizione degli endpoint API. Utilizza **Entity Framework Core** con **SQLite** per la gestione del database.
*   **Frontend React/Vite**: Residente nella cartella `frontend-gestione-spese`, offre l’interfaccia utente interattiva. Consuma le API esposte dal backend per visualizzare e manipolare i dati.

### 1.1 Flusso Logico di Sviluppo

Il processo di sviluppo e il flusso logico dell’applicazione seguono i seguenti passaggi:

1.  **Definizione dei Modelli di Dominio**: Le entità principali (Utente, Gruppo, Spesa, DivisioneSpesa, Riepilogo) sono definite nella cartella `Models` del backend.
2.  **Configurazione del Contesto EF Core**: La classe `ApplicationDbContext` (nella cartella `Data`) configura le relazioni tra le entità e il database.
3.  **Creazione del Database**: Tramite migrazioni basate sui modelli e `EnsureCreated()`, viene generato il database SQLite (`gestionespese.db`).
4.  **Implementazione dei Controller API**: I controller nella cartella `Controllers` espongono gli endpoint per autenticazione, gestione gruppi, spese e riepiloghi.
5.  **Sviluppo del Frontend React**: Il frontend gestisce il routing, le chiamate API e la visualizzazione dei dati tramite componenti riutilizzabili.

## 2. Backend ASP.NET Core: Analisi Dettagliata

Il backend è il cuore logico dell’applicazione. Di seguito, un’analisi dei suoi componenti chiave.

### 2.1 Program.cs: Configurazione e Avvio

Il file `Program.cs` è il punto di ingresso dell’applicazione ASP.NET Core. Configura i servizi, il middleware e avvia il server. Le configurazioni principali includono:

*   **Servizi**: Registrazione di `ControllersWithViews`, `HttpClient`, `ApplicationDbContext` (con SQLite che punta a `C:\home\gestionespese.db` per persistenza su Azure), `SwaggerGen` per la documentazione API.
*   **CORS**: Definizione di una policy `AllowAll` per permettere al frontend di comunicare con il backend da domini diversi.
*   **Database Initialization**: Un blocco `try/catch` assicura la creazione del database e delle tabelle tramite `context.Database.EnsureCreated()` all’avvio, con logging degli errori su `C:\home\startup-log.txt`.
*   **Pipeline HTTP**: Configurazione di `UseDeveloperExceptionPage`, `UseSwagger`, `UseSwaggerUI`, `UseRouting`, `UseCors`, `UseAuthorization` e `MapControllers`.

**Parte di codice rilevante (Program.cs):**

```csharp
// ✅ Percorso persistente su Azure App Service (C:\home)
var dbPath = Path.Combine("C:\\home", "gestionespese.db");
builder.Services.AddDbContext<gestione_spese.Data.ApplicationDbContext>
(options =>
options.UseSqlite($"Data Source={dbPath}"));
// ...
try
{
    using (var scope = app.Services.CreateScope())
    {
        var context = scope.ServiceProvider
            .GetRequiredService<gestione_spese.Data.ApplicationDbContext>();
        context.Database.EnsureCreated();
    }
}
catch (Exception ex)
{
    File.AppendAllText("C:\\home\\startup-log.txt",
        $"[{DateTime.Now}] DB ERROR: {ex.Message}\n");
}
```

Questa sezione è cruciale per spiegare come il database viene configurato e inizializzato, e come l’applicazione gestisce la persistenza dei dati in un ambiente come Azure.

### 2.2 ApplicationDbContext.cs: Modello Dati e Relazioni

La classe `ApplicationDbContext` estende `DbContext` e definisce i `DbSet` per le entità del dominio, rappresentando le tabelle del database. Il metodo `OnModelCreating` configura le relazioni tra queste entità:

*   **Utente-Gruppo (Molti-a-Molti)**: Creata una tabella di join `UtenteGruppo`.
*   **Spesa-Gruppo (Uno-a-Molti)**: `DeleteBehavior.Restrict` per evitare cancellazioni a cascata delle spese se un gruppo viene eliminato.
*   **Spesa-UtenteChePaga (Uno-a-Molti)**: `DeleteBehavior.Restrict` per proteggere l’integrità dei dati dell’utente pagatore.
*   **DivisioneSpesa-Spesa (Uno-a-Molti)**: `DeleteBehavior.Cascade` per eliminare automaticamente le divisioni quando una spesa viene cancellata.
*   **DivisioneSpesa-Utente (Uno-a-Molti)**: `DeleteBehavior.Restrict` per evitare la cancellazione di un utente con divisioni attive.
*   **Riepilogo-Gruppo e Riepilogo-Utenti (Uno-a-Molti)**: `DeleteBehavior.Restrict` per proteggere i dati dei riepiloghi.

**Parte di codice rilevante (ApplicationDbContext.cs):**

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    base.OnModelCreating(modelBuilder);
    modelBuilder.Entity<Utente>()
        .HasMany(u => u.Gruppi)
        .WithMany(g => g.Utenti)
        .UsingEntity(j => j.ToTable("UtenteGruppo"));

    modelBuilder.Entity<Spesa>()
        .HasOne(s => s.Gruppo)
        .WithMany(g => g.Spese)
        .HasForeignKey(s => s.Gruppo_ID)
        .OnDelete(DeleteBehavior.Restrict);

    modelBuilder.Entity<DivisioneSpesa>()
        .HasOne(d => d.Spesa)
        .WithMany(s => s.Divisioni)
        .HasForeignKey(d => d.Spesa_ID)
        .OnDelete(DeleteBehavior.Cascade);

    // ... altre relazioni con DeleteBehavior.Restrict
}
```

### 2.3 Modelli Principali (Models)

La cartella `Models` contiene le definizioni delle entità che mappano le tabelle del database. Ogni modello include proprietà e collezioni di navigazione per le relazioni:

*   **Utente.cs**: `Id`, `Nome`, `Email`, `PasswordHash`, `Gruppi`, `SpesePagate`, `DivisioniSpesa`, `Riepiloghetti`.
*   **Gruppo.cs**: `Id`, `Nome`, `Descrizione`, `CodiceInvito`, `Attivo`, `Utenti`, `Spese`, `Riepiloghetti`.
*   **Spesa.cs**: `Id`, `Importo`, `Descrizione`, `DataSpesa`, `Gruppo_ID`, `ChiPaga_ID`, `Divisioni`.
*   **DivisioneSpesa.cs**: `Id`, `Spesa_ID`, `Utente_ID`, `Importo`.
*   **Riepilogo.cs**: `Id`, `Gruppo_ID`, `ChiDeve_ID`, `AChiDeve_ID`, `Importo`, `Pagato`, `DataCalcolo`.
*   **SpesaDTO.cs / NuovaSpesaDTO**: per il trasferimento dati delle spese, include `Gruppo_ID`, `ChiPaga_ID`, `Importo`, `Descrizione`, `UtentiCoinvoltiIds`.

### 2.4 Controller API (Controllers)

I controller gestiscono la logica applicativa e espongono gli endpoint REST. Tutti utilizzano `ApplicationDbContext` tramite Dependency Injection.

*   **AuthController.cs**: Gestisce la registrazione e il login degli utenti. L’endpoint `POST login` verifica le credenziali, usa `BCrypt.Verify` per le password e, se l’utente non esiste, lo registra automaticamente. Restituisce i dati dell’utente o un errore di autenticazione.

**Parte di codice rilevante (AuthController.cs - login):**

```csharp
[HttpPost("login")]
public async Task<ActionResult<Utente>> Login([FromBody] LoginRequest request)
{
    var utente = await _context.Utenti
        .Include(u => u.Gruppi)
        .FirstOrDefaultAsync(u => u.Email == request.Email);

    if (utente == null)
    {
        // Registrazione automatica se l'utente non esiste
        utente = new Utente
        {
            Email = request.Email,
            PasswordHash = BCrypt.Net.BCrypt.HashPassword(request.Password),
            Nome = request.Email.Split('@')[0] // Nome predefinito
        };
        _context.Utenti.Add(utente);
        await _context.SaveChangesAsync();
        return Ok(utente);
    }

    if (!BCrypt.Net.BCrypt.Verify(request.Password, utente.PasswordHash))
    {
        return Unauthorized("Password errata.");
    }

    return Ok(utente);
}
```

*   **UtenteController.cs**: Gestisce operazioni CRUD sugli utenti e un endpoint per l’aggiornamento del nome utente (`PUT {id}/nome`).
*   **GruppoController.cs**: Gestisce la creazione, l’aggiornamento e l’eliminazione dei gruppi. Include logiche per l’aggiunta/rimozione di membri, la generazione di codici invito e, in particolare, l’algoritmo per il calcolo dei bilanci del gruppo.

**Parte di codice rilevante (GruppoController.cs - GetBilanciGruppo):**

```csharp
[HttpGet("{id}/Bilanci")]
public async Task<ActionResult<IEnumerable<object>>> GetBilanciGruppo(int id)
{
    // ... (omissis per brevità)
    var saldi = new Dictionary<int, decimal>();
    // Inizializzazione saldi per ogni utente
    // Calcolo saldi basato su spese e divisioni
    foreach (var spesa in spese)
    {
        if (saldi.ContainsKey(spesa.ChiPaga_ID))
        {
            saldi[spesa.ChiPaga_ID] += spesa.Importo; // Chi paga ha un credito
        }
        foreach (var div in spesa.Divisioni)
        {
            if (saldi.ContainsKey(div.Utente_ID))
            {
                saldi[div.Utente_ID] -= div.Importo; // Chi deve ha un debito
            }
        }
    }
    // Algoritmo di pareggio debiti/crediti (Greedy Algorithm)
    var debitori = saldi.Where(s => s.Value < -0.01m).OrderBy(s => s.Value).ToList();
    var creditori = saldi.Where(s => s.Value > 0.01m).OrderByDescending(s => s.Value).ToList();
    // ... (logica per creare/aggiornare Riepilogo e restituire i bilanci)
}
```

*   **SpesaController.cs**: Implementa la logica CRUD per le spese. Il metodo `PostSpesa` e `PutSpesa` sono particolarmente importanti per la gestione della divisione automatica delle spese.

**Parte di codice rilevante (SpesaController.cs - PostSpesa):**

```csharp
[HttpPost]
public async Task<ActionResult<Spesa>> PostSpesa([FromBody] NuovaSpesaDTO dto)
{
    // ... (validazioni e creazione nuovaSpesa)
    List<Utente> utentiDaAddebitare;
    if (dto.UtentiCoinvoltiIds == null || !dto.UtentiCoinvoltiIds.Any())
    {
        utentiDaAddebitare = gruppo.Utenti.ToList(); // Tutti i membri del gruppo
    }
    else
    {
        utentiDaAddebitare = gruppo.Utenti.Where(u => dto.UtentiCoinvoltiIds.Contains(u.Id)).ToList();
    }

    if (utentiDaAddebitare.Any())
    {
        decimal quota = Math.Round(nuovaSpesa.Importo / utentiDaAddebitare.Count, 2);
        foreach (var utente in utentiDaAddebitare)
        {
            var divisione = new DivisioneSpesa
            {
                Spesa_ID = nuovaSpesa.Id,
                Utente_ID = utente.Id,
                Importo = quota
            };
            _context.Divisioni.Add(divisione);
        }
        await _context.SaveChangesAsync();
    }
    return CreatedAtAction(nameof(GetSpesa), new { id = nuovaSpesa.Id }, nuovaSpesa);
}
```

*   **DivisioneSpesaController.cs**: Permette di leggere e aggiornare le singole divisioni, spesso usato indirettamente.
*   **RiepilogoController.cs**: Gestisce la persistenza e lo stato di saldatura dei record `Riepilogo`. Non contiene l’algoritmo di calcolo dei bilanci, che è invece in `GruppoController`.

## 3. Frontend React/Vite: Interfaccia Utente

Il frontend è una Single-Page Application (SPA) sviluppata con React e Vite, che interagisce con il backend tramite chiamate API.

### 3.1 App.jsx: Gestione dello Stato e Routing

`App.jsx` è il componente radice del frontend. Gestisce lo stato globale dell’applicazione (utente loggato, gruppo attivo, modalità login/registrazione) e il routing condizionale delle pagine.

*   **Autenticazione**: Utilizza `useState` per `utente` (recuperato da `localStorage`) e gestisce la logica di login/registrazione tramite `handleLogin`, che chiama l’API `Auth/login`.
*   **Routing Condizionale**: La visualizzazione delle pagine (`HomePage`, `DettaglioGruppo`, `RiepilogoGruppo`) dipende dallo stato di `utente`, `gruppoAttivoId` e `vistaRiepilogo`.
*   **Modali**: Gestisce l’apertura e la chiusura di `ModalNuovoGruppo`.

**Parte di codice rilevante (App.jsx - handleLogin e rendering condizionale):**

```javascript
// ... (stato e funzioni di login/registrazione)
const handleLogin = async (e) => {
    e.preventDefault();
    // ... (logica di validazione e chiamata API)
    try {
        const response = await fetch(`${API_BASE_URL}/Auth/login`, { /* ... */ });
        if (response.ok) {
            const user = await response.json();
            // ... (gestione registrazione e salvataggio utente in localStorage)
            setUtente(user);
            localStorage.setItem('utente_spese', JSON.stringify(user));
        } else { /* ... */ }
    } catch (error) { /* ... */ }
};

if (!utente) {
    return ( /* ... JSX per il form di login/registrazione ... */ );
}

return (
    <div className="min-h-screen bg-gray-50 font-sans">
        <Navbar /* ... */ />
        <main>
            {vistaRiepilogo && gruppoAttivoId ? (
                <RiepilogoGruppo /* ... */ />
            ) : gruppoAttivoId ? (
                <DettaglioGruppo /* ... */ />
            ) : (
                <HomePage /* ... */ />
            )}
        </main>
        {isModalOpen && (
            <ModalNuovoGruppo /* ... */ />
        )}
    </div>
);
```

### 3.2 Struttura src/

La cartella `src/` è organizzata in:

*   **api/api.js**: Contiene un modulo centrale per tutte le chiamate HTTP verso il backend, centralizzando l’URL base e la gestione degli errori.
*   **components/**: Componenti React riutilizzabili (es. `Navbar`, `LoginForm`, `GruppoCard`, `ModalNuovaSpesa`, `GraficoTorta`, `GraficoIstogramma`).
*   **pages/**: Rappresenta le viste ad alto livello dell’applicazione (`HomePage`, `DettaglioGruppo`, `RiepilogoGruppo`).

### 3.3 Pagine Principali (pages)

*   **HomePage.jsx**: La landing page dopo il login. Mostra i gruppi a cui l’utente partecipa e permette di unirsi a nuovi gruppi o crearne di nuovi.
*   **DettaglioGruppo.jsx**: Mostra i dettagli di un gruppo specifico, inclusa la lista delle spese, i membri e i pulsanti per aggiungere/modificare spese o aprire il riepilogo. È il centro del flusso frontend dopo la home.
*   **RiepilogoGruppo.jsx**: Interroga il `RiepilogoController` (e indirettamente `GruppoController` per i bilanci) per visualizzare i debiti/crediti tra gli utenti del gruppo, spesso con l’ausilio di grafici.

### 3.4 Componenti Riutilizzabili (components)

*   **Navbar.jsx**: Il menu superiore con il titolo dell’app e il pulsante di logout.
*   **LoginForm.jsx**: Gestisce il form di login/registrazione.
*   **GruppoCard.jsx**: Rappresenta una card per un singolo gruppo, cliccabile per accedere al dettaglio.
*   **ModalNuovoGruppo.jsx**: Modale per la creazione di un nuovo gruppo.
*   **ModalNuovaSpesa.jsx**: Modale complessa per l’inserimento di una nuova spesa, gestisce importo, descrizione, data e selezione degli utenti per la divisione.
*   **ModalModificaSpesa.jsx**: Simile a `ModalNuovaSpesa`, ma per la modifica di una spesa esistente.
*   **GraficoTorta.jsx / GraficoIstogramma.jsx**: Componenti per la visualizzazione grafica dei dati delle spese.

## 4. Consigli per l’Esposizione Orale

Per prepararsi al meglio all’esposizione, si consiglia di seguire questa strategia:

1.  **Visione Macro a Micro**: Iniziare sempre con una panoramica generale dell’architettura (backend/frontend, API REST), per poi scendere nei dettagli di specifici file o funzioni.
2.  **Funzione di Ogni File**: Per ogni file menzionato, essere in grado di descrivere chiaramente il suo ruolo nel progetto e come si integra con gli altri componenti.
3.  **Flusso Utente-Codice**: Collegare sempre le azioni dell’utente nel frontend con le chiamate API corrispondenti nel backend e la logica di business che ne deriva. Ad esempio, spiegare il flusso completo di una spesa: `Click su Aggiungi spesa` -> `ModalNuovaSpesa` -> `POST /api/Spesa` -> `creazione DivisioneSpesa` -> `ricarico lista sul frontend`.
4.  **Parti di Codice Chiave**: Concentrarsi sulle sezioni di codice evidenziate in questa guida (es. configurazione `Program.cs`, relazioni `ApplicationDbContext`, algoritmo in `GetBilanciGruppo` in `GruppoController`, logica `PostSpesa` in `SpesaController`, gestione autenticazione in `AuthController` e routing in `App.jsx`). Essere pronti a spiegare il perché di certe scelte implementative (es. `DeleteBehavior.Restrict` vs `Cascade`, uso di SQLite su Azure).
5.  **Linguaggio Semplice**: Spiegare concetti complessi in modo chiaro e conciso, come se si stesse parlando a un compagno di studi.
6.  **Preparazione alle Domande**: Anticipare possibili domande del professore, ad esempio su:
    *   Le proprietà e il ruolo di ogni modello (`Utente`, `Gruppo`, `Spesa`, `DivisioneSpesa`, `Riepilogo`).
    *   Il funzionamento di ogni controller, inclusi verbo HTTP, route, parametri e tipo di risposta.
    *   Come il frontend chiama gli endpoint API e da quali componenti.
    *   Il motivo delle scelte di `DeleteBehavior` in Entity Framework Core.
    *   La logica dietro l’uso di SQLite e la persistenza in `C:\home` su Azure.

Seguendo questa guida, il gruppo sarà in grado di presentare il progetto con sicurezza e rispondere a domande sia di alto livello che specifiche sul codice.

## Riferimenti

[1] Documento “Gestore Spese - Spiegazione del progetto” (allegato)
[2] Documento “Gestore Spese - Spiegazione del progetto (versione estesa)” (allegato)
[3] Repository GitHub: [lorenzograssiUni/gestore-spese](https://github.com/lorenzograssiUni/gestore-spese)
