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


## 🌐 Capitolo 7: Layer di Comunicazione Frontend e DTO Backend

### 7.1 `api.js`: Il Modulo di Interazione con il Backend

Il file `api.js` nel frontend funge da strato di astrazione per tutte le chiamate HTTP verso il backend. Questo approccio centralizza la logica di comunicazione, rendendo il codice più modulare, manutenibile e leggibile. Ogni funzione esportata in `api.js` corrisponde a un'operazione specifica sull'API RESTful del backend.

Esempi di funzioni e la loro mappatura agli endpoint del backend:

*   **`login(email, password)`**: Effettua una richiesta `POST` a `${API_BASE_URL}/Auth/login` per l'autenticazione dell'utente.
*   **`getGruppiUtente(utenteId)`**: Effettua una richiesta `GET` a `${API_BASE_URL}/Gruppo/utente/${utenteId}` per recuperare i gruppi a cui un utente appartiene.
*   **`creaSpesa(dati)`**: Effettua una richiesta `POST` a `${API_BASE_URL}/Spesa` per la creazione di una nuova spesa. Il corpo della richiesta è un oggetto JSON che rispecchia il `NuovaSpesaDTO` del backend.
*   **`saldaDebito(riepilogoId)`**: Effettua una richiesta `PUT` a `${API_BASE_URL}/Riepilogo/${riepilogoId}/Saldato` per marcare un debito come saldato.

Questo modulo incapsula i dettagli delle chiamate `fetch`, inclusi i metodi HTTP, gli header (`Content-Type: application/json`) e la serializzazione del corpo della richiesta (`JSON.stringify`). Ciò permette ai componenti React di interagire con il backend tramite funzioni semplici e chiare, senza doversi preoccupare dei dettagli implementativi delle richieste HTTP.

### 7.2 `NuovaSpesaDTO.cs`: Data Transfer Object per la Creazione di Spese

Nel contesto di un'architettura a strati, i **Data Transfer Object (DTO)** sono oggetti utilizzati per trasportare dati tra i processi. Nel progetto `gestore-spese`, `NuovaSpesaDTO.cs` è un esempio di DTO utilizzato per la creazione di nuove spese. La sua struttura è ottimizzata per ricevere i dati dal frontend in un formato specifico, disaccoppiato dal modello di dominio `Spesa` completo.

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

Caratteristiche principali:

*   **Disaccoppiamento**: Il DTO non espone tutte le proprietà del modello di dominio `Spesa` (ad esempio, `Id` della spesa, `Divisioni`). Questo riduce la superficie di attacco e previene l'over-posting di dati non desiderati.
*   **Validazione**: Le annotazioni `[Required]` e `[Range(0.01, double.MaxValue)]` applicano regole di validazione a livello di modello, garantendo che l'importo sia sempre presente e positivo. Queste validazioni vengono eseguite automaticamente dal framework ASP.NET Core prima che i dati raggiungano la logica di business del controller.
*   **Semplicità**: Il DTO fornisce un contratto chiaro e semplice per la creazione di una spesa, includendo l'ID del gruppo, l'ID del pagatore, l'importo, una descrizione opzionale e una lista di ID degli utenti coinvolti nella divisione della spesa.

L'uso di DTO migliora la sicurezza, la manutenibilità e la chiarezza del codice, separando la rappresentazione dei dati per il trasferimento dalla rappresentazione dei dati nel dominio dell'applicazione.

## 📊 Capitolo 8: Visualizzazioni Dati Avanzate nel Frontend

Oltre al `GraficoTorta.jsx` analizzato precedentemente, il progetto include `GraficoIstogramma.jsx`, un altro esempio di visualizzazione dati creata manualmente con SVG, dimostrando una profonda padronanza delle tecnologie web di base per la rappresentazione grafica.

### 8.1 `GraficoIstogramma.jsx`: Implementazione di un Istogramma con SVG

Il componente `GraficoIstogramma.jsx` visualizza l'ammontare totale delle spese per ciascun membro del gruppo sotto forma di istogramma. Anche in questo caso, la scelta di utilizzare SVG puro anziché librerie di terze parti è una decisione di design che privilegia il controllo fine, la leggerezza e l'ottimizzazione delle performance.

*   **Calcolo dei Totali e Valore Massimo**: Similmente al grafico a torta, il componente calcola il `totale` delle spese per ogni membro e determina il `maxValore` per scalare correttamente le barre dell'istogramma.
*   **Dimensioni SVG e Padding**: Vengono definite le dimensioni del canvas SVG (`svgW`, `svgH`) e i margini (`paddingLeft`, `paddingRight`, `paddingTop`, `paddingBottom`) per garantire che il grafico sia ben proporzionato e leggibile. Vengono calcolate anche le dimensioni effettive dell'area del grafico (`chartW`, `chartH`).
*   **Linee della Griglia**: Vengono generate dinamicamente delle linee orizzontali della griglia (`gridLines`) con etichette di valore per facilitare la lettura dei dati. Queste sono create utilizzando elementi `<line>` e `<text>` SVG.
*   **Barre dell'Istogramma**: Per ogni membro, viene calcolata l'altezza della barra (`barH`) in proporzione al `maxValore`. La posizione (`x`, `y`) e la larghezza (`barWidth`) di ciascuna barra sono determinate per distribuirle uniformemente all'interno dell'area del grafico. Ogni barra è un elemento `<rect>` SVG con angoli arrotondati (`rx`, `ry`) e un colore assegnato da un array predefinito.
*   **Etichette**: Sopra ogni barra viene visualizzato l'importo totale (`€{m.totale.toFixed(0)}`) e sotto la barra il nome del membro (`m.nome`), con una logica per troncare i nomi lunghi e aggiungere i puntini di sospensione (`...`).

Questo componente, insieme al `GraficoTorta.jsx`, evidenzia una competenza avanzata nella manipolazione di SVG per la visualizzazione dati, un aspetto che può essere particolarmente apprezzato in un contesto accademico per la sua efficienza e il controllo granulare sull'output grafico. La scelta di implementare queste visualizzazioni da zero, senza l'ausilio di librerie complesse, dimostra una comprensione profonda dei principi di rendering grafico e un'attenzione alla leggerezza dell'applicazione frontend.

---

## 📚 Capitolo 9: Approfondimenti sulla Gestione dello Stato e Interazione Utente

### 9.1 `DettaglioGruppo.jsx`: Interazione Complessa e Gestione dello Stato Locale

Il componente `DettaglioGruppo.jsx` è un esempio eccellente di come React gestisca stati complessi e interazioni utente. Oltre a quanto già menzionato, è importante sottolineare:

*   **Stato Locale Dettagliato**: Il componente mantiene diversi stati locali (`gruppo`, `loading`, `nuovoMembroNome`, `isAggiungendo`, `isModalSpesaOpen`, `spesaInModifica`) per gestire l'interfaccia utente e le sue interazioni. Questo approccio garantisce che le modifiche all'interfaccia siano reattive e che l'utente riceva feedback immediato.
*   **Gestione Modali**: L'apertura e chiusura delle modali (`ModalNuovaSpesa`, `ModalModificaSpesa`) sono controllate tramite stati booleani (`isModalSpesaOpen`, `spesaInModifica`). Quando una modale viene aperta, lo stato `spesaInModifica` può essere popolato con i dati della spesa da modificare, permettendo alla modale di pre-compilare i campi.
*   **Feedback Utente**: L'uso di `alert()` per le conferme di eliminazione (`handleEliminaGruppo`, `handleEliminaSpesa`, `handleEliminaAmico`) e per i messaggi di errore fornisce un feedback diretto all'utente. Sebbene per applicazioni più complesse si preferirebbero soluzioni UI più sofisticate (es. toast notifications), per questo progetto è un metodo efficace e semplice.
*   **Ottimizzazione del Rendering**: L'uso di `useEffect` con `gruppoId` come dipendenza assicura che i dati del gruppo vengano ricaricati solo quando l'ID del gruppo cambia, evitando rendering non necessari e ottimizzando le chiamate API.

### 9.2 `ModalNuovaSpesa.jsx` e `ModalModificaSpesa.jsx`: Componenti Modali Riutilizzabili

Questi componenti modali sono esempi di come si possano creare interfacce utente complesse e riutilizzabili in React. Essi incapsulano la logica per la creazione e modifica delle spese, interagendo con il backend tramite le funzioni definite in `api.js`.

*   **Form Controllo**: Entrambe le modali utilizzano stati locali per gestire i campi del form (descrizione, importo, pagatore, utenti coinvolti). Questo permette una validazione in tempo reale e un controllo preciso sull'input dell'utente.
*   **Logica di Divisione**: La `ModalNuovaSpesa` include una logica per la divisione della spesa: può essere divisa con tutti i membri del gruppo o solo con un sottoinsieme selezionato. Questo si traduce nella costruzione dell'array `UtentiCoinvoltiIds` che viene inviato al backend tramite il `NuovaSpesaDTO`.
*   **Interazione con l'API**: Le modali chiamano le funzioni `creaSpesa` o `modificaSpesa` da `api.js` per persistere i dati. Dopo un'operazione riuscita, chiamano una callback (`onSpesaAggiunta` o `onSpesaModificata`) fornita dal componente padre (`DettaglioGruppo.jsx`) per aggiornare la visualizzazione dei dati.

Questi componenti dimostrano l'efficacia dell'approccio basato su componenti di React per costruire interfacce utente interattive e complesse, mantenendo al contempo una chiara separazione delle responsabilità.

---

## 🎯 Conclusione per l'Espositore (Approfondita)

Per esporre questo progetto con una padronanza ancora maggiore, oltre ai punti precedenti, considera di enfatizzare:

1.  **Sinergia Frontend-Backend**: Sottolinea come il `api.js` del frontend e i DTO del backend lavorino in tandem per definire un contratto di comunicazione robusto e efficiente, riducendo la complessità e migliorando la sicurezza.
2.  **Decisioni di Design UI/UX**: Discuti le scelte dietro l'implementazione manuale dei grafici SVG. Spiega i vantaggi in termini di performance (nessuna dipendenza esterna pesante), controllo granulare sull'aspetto grafico e la dimostrazione di competenze tecniche avanzate. Menziona anche come la gestione dello stato locale nei componenti React contribuisca a un'esperienza utente fluida e reattiva.
3.  **Scalabilità e Manutenibilità**: Argomenta come la modularità del codice (backend suddiviso in controller e modelli, frontend in componenti e moduli API) contribuisca alla scalabilità e alla facilità di manutenzione del progetto. Ad esempio, l'aggiunta di nuove funzionalità o la modifica di quelle esistenti sarebbe facilitata dalla chiara separazione delle responsabilità.
4.  **Best Practices di Sviluppo**: Evidenzia l'applicazione di best practice come la Dependency Injection nel backend, l'uso di `localStorage` per la persistenza dello stato utente nel frontend, e le strategie di gestione degli errori e di feedback all'utente.

Presentando questi aspetti, dimostrerai non solo di aver compreso il funzionamento del progetto, ma anche di averne analizzato le scelte di design e le implicazioni tecniche a un livello universitario, come richiesto.

---
*Questo manuale è una risorsa viva, basata sull'analisi riga per riga del codice sorgente di Split Mate.*
