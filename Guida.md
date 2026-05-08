# 📔 Manuale Tecnico Split Mate: Architettura e Sviluppo Full-Stack

Benvenuti nel manuale definitivo di **Split Mate**. Questo documento non è una semplice guida, ma un trattato tecnico progettato per fornire una comprensione profonda e accademica del progetto. Studiare questo manuale significa acquisire la capacità di navigare nel codice, comprendere le scelte architettoniche e saper esporre il progetto con la padronanza di chi lo ha concepito.

---

## 🏛️ Capitolo 1: Architettura del Sistema

### 1.1 Paradigma Full-Stack

Split Mate è implementato seguendo il paradigma **Decoupled Architecture** (Architettura Disaccoppiata). Il sistema è diviso in due entità distinte che comunicano esclusivamente tramite protocolli standard:

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
*   **Middleware**: La pipeline di richiesta include `UseDeveloperExceptionPage()` (per debug), `UseSwagger()` e `UseSwaggerUI()` (per la documentazione API), `UseRouting()`, `UseCors(
AllowAll")`, `UseAuthorization()` e `MapControllers()`. L'ordine di questi middleware è fondamentale per il corretto funzionamento dell'applicazione.

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
*   **Relazioni Uno-a-Molti con `DeleteBehavior.Restrict`**: Per `Spesa` (con `Gruppo` e `UtenteChePaga`), `DivisioneSpesa` (con `Utente`) e `Riepilogo` (con `Gruppo`, `UtenteCheDeve`, `UtenteACuiDeve`), viene utilizzato `OnDelete(DeleteBehavior.Restrict)`. Questo impedisce la cancellazione di un'entità padre se esistono ancora entità figlie correlate, garantendo l'integrità referenziale del database. Ad esempio, non è possibile eliminare un gruppo se ci sono ancora spese associate ad esso.
*   **Relazione Uno-a-Molti con `DeleteBehavior.Cascade`**: Per `DivisioneSpesa` (con `Spesa`), viene utilizzato `OnDelete(DeleteBehavior.Cascade)`. Questo significa che quando una `Spesa` viene eliminata, tutte le `DivisioneSpesa` ad essa correlate verranno automaticamente eliminate. Questa è una scelta di design per semplificare la gestione delle spese e delle loro divisioni.

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
*   **Auto-Provisioning**: Se l'utente non esiste, viene creato un nuovo account. Il `Nome` viene derivato dall'email e la `PasswordHash` viene generata utilizzando `BCrypt.HashPassword(request.Password)`. Questo meccanismo semplifica l'onboarding dell'utente, ma in un contesto aziendale potrebbe essere preferibile un processo di registrazione esplicito.
*   **Verifica Password**: Se l'utente esiste, la password fornita viene verificata con `BCrypt.Verify(request.Password, utente.PasswordHash)`. In caso di mancata corrispondenza, viene restituito un errore `Unauthorized`.
*   **Inclusione Gruppi**: Durante il login, l'oggetto `Utente` restituito include anche i `Gruppi` a cui l'utente appartiene, ottimizzando le successive chiamate al frontend.

L'endpoint `GET api/Auth/exists?email=...` permette di verificare l'esistenza di un'email nel sistema, utile per la validazione in tempo reale nel frontend.

---

## ⚛️ Capitolo 4: Il Frontend (React & Vite)

### 4.1 Gestione dello Stato Globale

In `App.jsx`, lo stato dell'utente è persistito nel `localStorage`. Questo permette all'utente di rimanere loggato anche dopo il refresh della pagina, migliorando la **User Retention**.

#### 4.1.1 `App.jsx`: Il Componente Radice e il Routing Condizionale

Il file `App.jsx` è il componente radice dell'applicazione React. Gestisce lo stato globale dell'utente (`utente`), la visibilità di modali (`isModalOpen`), e il routing condizionale basato sullo stato di autenticazione e sulla navigazione all'interno dei gruppi. La logica di autenticazione è integrata direttamente in questo componente:

*   **Stato Utente Persistente**: `useState(() => { ... })` viene utilizzato per inizializzare lo stato `utente` leggendo dal `localStorage`. Se `localStorage.getItem('utente_spese')` restituisce un valore, questo viene parsato come JSON e impostato come stato iniziale. Questo assicura che l'utente rimanga autenticato tra le sessioni del browser.
*   **`handleLogin`**: Questa funzione asincrona gestisce sia il login che la registrazione. Effettua una chiamata `POST` all'endpoint `/Auth/login` del backend. In caso di successo, l'oggetto utente restituito viene salvato nello stato locale e nel `localStorage`. Se la modalità è `registrazione` e viene fornito un `nomeInput`, viene effettuata una successiva chiamata `PUT` all'endpoint `/Utente/{id}/nome` per aggiornare il nome dell'utente.
*   **Rendering Condizionale**: L'interfaccia utente viene renderizzata in modo condizionale:
    *   Se `!utente` (utente non autenticato), viene mostrato il form di login/registrazione (`LoginForm.jsx` o logica inline).
    *   Se `vistaRiepilogo` è `true` e `gruppoAttivoId` è impostato, viene mostrato il componente `RiepilogoGruppo`.
    *   Se `gruppoAttivoId` è impostato (ma `vistaRiepilogo` è `false`), viene mostrato il componente `DettaglioGruppo`.
    *   Altrimenti, viene mostrata la `HomePage` con l'elenco dei gruppi dell'utente.

### 4.2 Componenti e Hooks

Il frontend sfrutta gli **Hooks** di React (`useState`, `useEffect`) per gestire il ciclo di vita dei dati:
*   `useEffect`: Utilizzato per scatenare il recupero dei dati dal server non appena un componente viene visualizzato (es. caricamento dei gruppi nella Home).
*   **Rendering Condizionale**: L'interfaccia cambia dinamicamente in base allo stato (es. se `utente` è null, mostra il form di login, altrimenti la dashboard).

#### 4.2.1 `DettaglioGruppo.jsx`: Gestione delle Interazioni di Gruppo

Il componente `DettaglioGruppo.jsx` è responsabile della visualizzazione e gestione dei dettagli di un gruppo specifico. Questo include l'elenco dei membri, la cronologia delle spese e le funzionalità per aggiungere/rimuovere membri e spese.

*   **Caricamento Dati**: Il `useEffect` si attiva quando `gruppoId` cambia, chiamando `caricaDatiGruppo`. Questa funzione recupera i dettagli del gruppo, inclusi utenti e spese, tramite la funzione `getGruppo` dall'API del backend.
*   **Aggiunta Membri (Bot)**: La funzione `handleAggiungiMembro` permette di aggiungere un nuovo membro al gruppo. Per scopi dimostrativi, viene generata un'email fittizia per i "bot" (`bot_{GUID}@bot.local`). La chiamata API `aggiungiBotAlGruppo` viene utilizzata per persistere il nuovo utente nel gruppo.
*   **Eliminazione Gruppo**: `handleEliminaGruppo` gestisce la cancellazione di un intero gruppo, previa conferma dell'utente. Utilizza l'API `eliminaGruppo` e, in caso di successo, riporta l'utente alla pagina precedente (`onBack()`).
*   **Gestione Spese**: `handleEliminaSpesa` e `setSpesaInModifica` consentono rispettivamente di eliminare una spesa esistente o di aprire una modale per la sua modifica. Le modali `ModalNuovaSpesa` e `ModalModificaSpesa` sono componenti figli che gestiscono l'interazione specifica per la creazione e modifica delle spese.
*   **Rimozione Membri**: `handleEliminaAmico` permette di rimuovere un membro dal gruppo. Una logica importante qui è la **validazione**: un membro non può essere rimosso se ha delle spese registrate (`eCoinvolto`). Questo rispecchia il `DeleteBehavior.Restrict` configurato nel backend per mantenere l'integrità dei dati.
*   **Visualizzazione Condizionale**: Il componente mostra un indicatore di caricamento (`loading`) e messaggi di errore se il gruppo non viene trovato. La lista dei membri e delle spese viene renderizzata dinamicamente, con la possibilità di scorrere le liste (`overflow-y-auto`).

### 4.3 Visualizzazione Dati (SVG & Math)

I grafici (`GraficoTorta.jsx`) non usano librerie esterne pesanti, ma sono disegnati manualmente tramite **SVG (Scalable Vector Graphics)**.
*   Viene utilizzata la trigonometria (`Math.cos`, `Math.sin`) per calcolare i punti delle fette della torta in base alla percentuale di spesa.

#### 4.3.1 `GraficoTorta.jsx`: Implementazione di un Grafico a Torta con SVG

Il componente `GraficoTorta.jsx` è un esempio notevole di come sia possibile creare visualizzazioni dati complesse senza dipendenze esterne, sfruttando direttamente le capacità di rendering di SVG e la matematica di base.

*   **Calcolo dei Totali**: Il componente aggrega le spese per ciascun membro (`totaliPerMembro`) e calcola il `totaleGlobale` di tutte le spese. Questo serve come base per determinare la percentuale di ogni fetta.
*   **Definizione dei Colori**: Un array `COLORI` predefinito viene utilizzato per assegnare colori distinti a ciascuna fetta del grafico.
*   **Generazione delle Fette SVG**: Il cuore del componente è il calcolo delle coordinate e del percorso (`path`) per ogni fetta del grafico a torta. Questo processo coinvolge:
    *   **Angoli**: Viene calcolato l'angolo iniziale e finale di ciascuna fetta in base alla percentuale di spesa. L'angolo iniziale per la prima fetta è `-Math.PI / 2` (corrispondente alle 12 in punto sul cerchio unitario).
    *   **Coordinate Cartesiane**: Le funzioni trigonometriche `Math.cos` e `Math.sin` sono utilizzate per convertire gli angoli in coordinate (x, y) sul bordo del cerchio. Questo permette di definire i punti di inizio e fine di ogni arco.
    *   **Comandi Path SVG**: Il `path` di ogni fetta è costruito utilizzando i comandi SVG `M` (move to), `L` (line to), e `A` (arc). Il parametro `largeArc` (`1` o `0`) è fondamentale per indicare se l'arco deve essere maggiore o minore di 180 gradi, garantendo la corretta visualizzazione delle fette.
*   **Rendering SVG**: Il componente restituisce un elemento `<svg>` contenente gli elementi `<path>` per ciascuna fetta, un `<circle>` centrale (per l'etichetta del totale) e elementi `<text>` per visualizzare il totale globale. La legenda è generata separatamente utilizzando elementi HTML standard.

Questo approccio dimostra una profonda comprensione dei principi di grafica vettoriale e un'ottimizzazione delle performance, evitando il caricamento di librerie di terze parti per un compito relativamente semplice ma visivamente efficace.

---

## 🛡️ Capitolo 5: Sicurezza e Best Practices

1.  **CORS (Cross-Origin Resource Sharing)**: Configurato nel `Program.cs` per permettere solo comunicazioni autorizzate tra il dominio del frontend e del backend.
2.  **Password Hashing**: Utilizzo di `BCrypt.Net` per garantire che le credenziali siano inattaccabili anche in caso di violazione del database.
3.  **Integrità Referenziale**: Utilizzo di `DeleteBehavior.Restrict` in Entity Framework per impedire la cancellazione accidentale di dati correlati (es. non puoi cancellare un utente se ha ancora dei debiti attivi).

---

## 📚 Capitolo 6: Glossario Avanzato per l'Esperto

*   **Middleware**: Software che si inserisce nel ciclo di richiesta/risposta del server (es. per gestire la sicurezza o i log).
*   **DTO (Data Transfer Object)**: Classi "leggere" (come `SpesaDTO`) usate per trasportare solo i dati necessari tra client e server, ottimizzando la banda.
*   **Dependency Injection**: Pattern che permette di passare le dipendenze (come il database) alle classi invece di crearle internamente, rendendo il codice testabile e modulare.
*   **Async/Await**: Programmazione asincrona che permette al server di gestire migliaia di richieste senza bloccarsi mentre aspetta il database.

---

## 🎯 Conclusione per l'Espositore

Per esporre questo progetto come un autore, ricorda di:
1.  **Giustificare le scelte**: "Abbiamo usato SQLite per la sua portabilità e .NET 10 per le ultime ottimizzazioni di performance".
2.  **Mostrare la robustezza**: "Il sistema gestisce i conflitti di cancellazione e protegge i dati sensibili".
3.  **Evidenziare l'innovazione**: "L'algoritmo di bilanciamento non è una semplice sottrazione, ma un'ottimizzazione dei flussi finanziari".

---
*Questo manuale è una risorsa viva, basata sull'analisi riga per riga del codice sorgente di Split Mate.*
