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

### 3.3 Ciclo di Vita di una Spesa e Logica di Divisione (`SpesaController.cs`)

Il `SpesaController.cs` è responsabile della gestione completa delle spese, dalla creazione alla modifica e cancellazione, inclusa la complessa logica di divisione. Questo controller interagisce strettamente con il `NuovaSpesaDTO` per ricevere i dati dal frontend.

*   **`PostSpesa([FromBody] NuovaSpesaDTO dto)`**: Questo metodo gestisce la creazione di una nuova spesa. Dopo aver validato l'esistenza del gruppo e del pagatore, crea un'istanza di `Spesa` e la persiste. La parte cruciale è la logica di divisione:
    *   Se `dto.UtentiCoinvoltiIds` è nullo o vuoto, la spesa viene divisa equamente tra *tutti* i membri del gruppo.
    *   Altrimenti, la spesa viene divisa equamente solo tra gli utenti specificati in `dto.UtentiCoinvoltiIds`.
    *   Per ogni utente coinvolto, viene creata una `DivisioneSpesa` con la quota calcolata (`Math.Round(nuovaSpesa.Importo / utentiDaAddebitare.Count, 2)`) e persistita nel database. L'uso di `Math.Round` con due cifre decimali è fondamentale per gestire correttamente gli importi monetari.
*   **`PutSpesa(int id, [FromBody] NuovaSpesaDTO dto)`**: Questo metodo gestisce la modifica di una spesa esistente. Recupera la spesa e le sue divisioni attuali, aggiorna i campi base (`Descrizione`, `Importo`, `ChiPaga_ID`), e poi **ricalcola completamente le divisioni**. Questo implica la rimozione delle `DivisioneSpesa` precedenti (`_context.Divisioni.RemoveRange(spesa.Divisioni)`) e la creazione di nuove basate sul `dto` aggiornato. Questo approccio garantisce che la logica di divisione sia sempre coerente con lo stato attuale della spesa.
*   **`DeleteSpesa(int id)`**: Gestisce la cancellazione di una spesa. Grazie alla configurazione `DeleteBehavior.Cascade` definita in `ApplicationDbContext.cs` per la relazione tra `Spesa` e `DivisioneSpesa`, la cancellazione di una spesa comporta automaticamente la cancellazione di tutte le sue divisioni associate, mantenendo l'integrità del database.

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

## 🎓 Capitolo 10: Simulazione Esame Orale - Domande e Risposte Tecniche

Questa sezione è progettata per prepararti a esporre il progetto "Split Mate" con la sicurezza e la profondità di un vero esperto, simulando le domande che un professore universitario di un corso di web app potrebbe porre.

### 10.1 Domande sull'Architettura e Design

**Domanda 1**: "Qual è il paradigma architetturale adottato per Split Mate e quali sono i vantaggi di questa scelta?"

**Risposta**: "Split Mate adotta un'**Architettura Disaccoppiata (Decoupled Architecture)**, suddividendo il sistema in un backend ASP.NET Core e un frontend React. Il vantaggio principale è la **separazione delle preoccupazioni (separation of concerns)**: il backend si concentra sulla logica di business e sulla persistenza dei dati, esponendo API RESTful, mentre il frontend gestisce l'interfaccia utente e l'esperienza utente. Questo permette lo sviluppo indipendente dei due strati, facilitando la scalabilità, la manutenibilità e la possibilità di sostituire uno strato senza impattare l'altro. Ad esempio, potremmo facilmente sviluppare un'applicazione mobile nativa che consumi le stesse API del backend esistente."

**Domanda 2**: "Perché avete scelto Entity Framework Core e SQLite per la persistenza dei dati? Quali alternative avreste considerato e in quali contesti?"

**Risposta**: "Abbiamo optato per **Entity Framework Core (EF Core)** come ORM e **SQLite** come database per la sua **portabilità e semplicità di setup**, ideale per un progetto dimostrativo o con requisiti di deployment leggeri, come su Azure App Service con un file system persistente. EF Core offre un'astrazione potente per interagire con il database tramite oggetti C#, riducendo la necessità di scrivere SQL manuale e supportando migrazioni. In contesti di produzione con carichi elevati o requisiti di scalabilità orizzontale, avremmo considerato database relazionali più robusti come PostgreSQL o SQL Server, o soluzioni NoSQL come MongoDB per scenari con schemi flessibili, sempre accoppiati a EF Core o a driver specifici per il database scelto."

**Domanda 3**: "Il vostro `Program.cs` mostra una configurazione CORS molto permissiva (`AllowAnyOrigin`). Quali sono le implicazioni di sicurezza di questa scelta e come la modifichereste in produzione?"

**Risposta**: "La configurazione `AllowAnyOrigin()` è stata adottata per facilitare lo sviluppo e il testing, permettendo al frontend di comunicare con il backend da qualsiasi origine. Tuttavia, in un ambiente di produzione, questa è una **grave vulnerabilità di sicurezza**. Un attaccante potrebbe effettuare richieste cross-origin malevole. In produzione, restringeremmo la policy CORS specificando esplicitamente solo i domini autorizzati del frontend, ad esempio `WithOrigins("https://tuofrontend.com")`, e limitando i metodi HTTP e gli header consentiti, per aderire al **principio del privilegio minimo**."

### 10.2 Domande sulla Logica di Business e Backend

**Domanda 4**: "Descrivete l'algoritmo di bilanciamento delle spese implementato nel `GruppoController`. Qual è la sua complessità computazionale e perché è stata scelta questa strategia?"

**Risposta**: "L'algoritmo di bilanciamento mira a minimizzare il numero di transazioni necessarie per saldare i debiti all'interno di un gruppo. Si articola in tre fasi: **Aggregazione**, dove si calcolano i saldi netti di ogni utente; **Classificazione**, dove gli utenti vengono divisi in debitori e creditori e ordinati per entità del saldo; e **Matching (Greedy Algorithm)**, dove il maggior debitore viene accoppiato con il maggior creditore, risolvendo il debito parzialmente o totalmente e aggiornando i saldi. Questo processo si ripete finché tutti i saldi non sono azzerati. La complessità è dominata dalle operazioni di ordinamento e dal ciclo di matching, che in un caso peggiore potrebbe essere `O(N log N)` o `O(N^2)` a seconda dell'implementazione precisa del matching, dove N è il numero di utenti. È stata scelta una strategia greedy per la sua relativa semplicità implementativa e per la sua efficacia nel ridurre il numero di transazioni, che è un obiettivo primario per la chiarezza finanziaria."

**Domanda 5**: "Il sistema implementa un 'auto-provisioning' degli utenti. Spiegate come funziona e quali sono i pro e i contro di questo approccio per la gestione degli account."

**Risposta**: "L'auto-provisioning, gestito nel `AuthController`, crea automaticamente un nuovo account utente se un'email non registrata tenta il login. Il nome utente viene derivato dall'email e la password viene hashata con BCrypt. Il principale **pro** è una **riduzione dell'attrito per l'utente (user friction)**, semplificando l'onboarding e incoraggiando l'adozione. Il **contro** è una potenziale **mancanza di controllo** sulla creazione degli account e una minore sicurezza percepita, poiché non c'è un processo di registrazione esplicito con conferma email. In un'applicazione con requisiti di sicurezza più stringenti, si preferirebbe un flusso di registrazione tradizionale con verifica dell'email per prevenire abusi e garantire l'identità dell'utente."

**Domanda 6**: "Descrivete il ciclo di vita di una spesa, dalla sua creazione nel frontend alla persistenza e alla gestione delle divisioni nel backend."

**Risposta**: "Il ciclo inizia nel frontend, tipicamente tramite il componente `ModalNuovaSpesa.jsx`, dove l'utente inserisce i dettagli (descrizione, importo, pagatore, utenti coinvolti). Questi dati vengono incapsulati in un oggetto e inviati al backend tramite la funzione `creaSpesa` in `api.js`. Nel backend, il `SpesaController.cs` riceve questi dati come `NuovaSpesaDTO`. Il controller valida il DTO, crea un'istanza del modello `Spesa`, la persiste nel database e, crucialmente, calcola le `DivisioneSpesa`. Se non specificato, la spesa viene divisa equamente tra tutti i membri del gruppo; altrimenti, tra gli utenti selezionati. Per ogni utente coinvolto, viene creata una `DivisioneSpesa` con la quota calcolata (`Math.Round(importo / count, 2)`), e queste divisioni vengono anch'esse persistite. Le relazioni tra `Spesa` e `DivisioneSpesa` sono gestite da Entity Framework Core, con `DeleteBehavior.Cascade` per le divisioni, assicurando che vengano eliminate automaticamente con la spesa principale."

### 10.3 Domande sul Frontend e User Experience

**Domanda 7**: "Come viene gestita la persistenza dello stato dell'utente nel frontend e quali sono le implicazioni per l'esperienza utente?"

**Risposta**: "Lo stato dell'utente (`utente`) viene persistito nel `localStorage` del browser, come visibile in `App.jsx`. Questo significa che, anche dopo aver chiuso e riaperto il browser o aver ricaricato la pagina, l'utente rimane autenticato. L'implicazione principale è un'**esperienza utente migliorata (User Retention)**, poiché l'utente non deve effettuare il login ad ogni visita. Tuttavia, è fondamentale che il `localStorage` non contenga informazioni sensibili non hashate o token di autenticazione a lungo termine senza adeguate misure di sicurezza, poiché è vulnerabile ad attacchi XSS. Per token JWT, ad esempio, si preferirebbe `HttpOnly cookies`."

**Domanda 8**: "Perché avete scelto di implementare i grafici (torta e istogramma) manualmente con SVG anziché utilizzare una libreria di charting esistente? Quali sono i pro e i contro di questa decisione?"

**Risposta**: "La decisione di implementare i grafici `GraficoTorta.jsx` e `GraficoIstogramma.jsx` manualmente con **SVG (Scalable Vector Graphics)** è stata dettata dalla volontà di avere il **controllo granulare** sull'aspetto e il comportamento dei grafici, oltre a **ridurre le dipendenze esterne** e il peso del bundle JavaScript. I **pro** includono: **leggerezza** dell'applicazione, **ottimizzazione delle performance** (nessun overhead di librerie complesse), **personalizzazione completa** del design e una dimostrazione di **competenze tecniche avanzate** nella manipolazione di SVG e trigonometria. I **contro** sono un **tempo di sviluppo maggiore** per funzionalità complesse, la necessità di una profonda conoscenza di SVG e matematica, e una potenziale **minore robustezza** rispetto a librerie mature che gestiscono molti edge case e interattività avanzate. Per questo progetto, i benefici hanno superato i costi."

**Domanda 9**: "Nel `DettaglioGruppo.jsx`, un membro non può essere rimosso se ha delle spese registrate. Come si riflette questa logica di business a livello di database e quali sono i vantaggi?"

**Risposta**: "Questa logica di business, implementata nel frontend tramite la verifica `eCoinvolto` e nel backend tramite `DeleteBehavior.Restrict` nelle relazioni di Entity Framework Core, garantisce l'**integrità referenziale** del database. Non è possibile eliminare un utente (o un gruppo) se esistono record correlati (spese o divisioni) che fanno riferimento ad esso. Il vantaggio principale è la **prevenzione della corruzione dei dati**: si evita di avere 'record orfani' o dati inconsistenti. Questo assicura che il database rifletta sempre uno stato valido e coerente delle informazioni finanziarie, fondamentale per un'applicazione di gestione spese."

### 10.4 Domande su Best Practices e Pattern di Sviluppo

**Domanda 10**: "Spiegate il concetto di Dependency Injection (DI) e come viene applicato nel backend di Split Mate. Quali benefici apporta?"

**Risposta**: "La **Dependency Injection (DI)** è un pattern di design che permette di invertire il controllo sulla creazione e gestione delle dipendenze. Invece di creare direttamente le dipendenze all'interno di una classe, queste vengono 'iniettate' dall'esterno. Nel backend ASP.NET Core di Split Mate, la DI è ampiamente utilizzata. Ad esempio, i controller come `GruppoController` o `AuthController` ricevono un'istanza di `ApplicationDbContext` tramite il loro costruttore. Il framework si occupa di creare e fornire questa istanza. I benefici sono molteplici: **testabilità** (è facile iniettare mock o stub per i test unitari), **modularità** (le classi sono meno accoppiate e più riutilizzabili), **manutenibilità** (le modifiche a una dipendenza non richiedono modifiche a tutte le classi che la usano) e **scalabilità** (facilita la gestione del ciclo di vita degli oggetti e l'uso di servizi a livello di applicazione)."

**Domanda 11**: "Qual è il ruolo dei DTO (Data Transfer Objects) nel vostro progetto e perché sono preferibili all'esposizione diretta dei modelli di dominio nelle API?"

**Risposta**: "I **DTO (Data Transfer Objects)**, come `NuovaSpesaDTO`, sono oggetti semplici utilizzati per trasferire dati tra il frontend e il backend. Il loro ruolo è quello di definire un **contratto di comunicazione** specifico per ogni operazione API, disaccoppiato dai modelli di dominio interni del database. Sono preferibili all'esposizione diretta dei modelli di dominio per diverse ragioni: **sicurezza** (si espongono solo i campi necessari, prevenendo l'over-posting e l'information leakage), **performance** (si trasferiscono solo i dati essenziali, riducendo il payload di rete), **flessibilità** (il modello di dominio può evolvere indipendentemente dal contratto API) e **separazione delle preoccupazioni** (il frontend non ha bisogno di conoscere la struttura interna completa del database)."

**Domanda 12**: "Come gestite la programmazione asincrona nel backend e quali sono i vantaggi di `async/await` in un'applicazione web?"

**Risposta**: "Nel backend ASP.NET Core, la programmazione asincrona è gestita tramite le parole chiave `async` e `await` di C#. Quasi tutte le operazioni I/O, come l'interazione con il database (`_context.SaveChangesAsync()`, `_context.Utenti.FirstOrDefaultAsync()`), sono implementate in modo asincrona. Il vantaggio principale in un'applicazione web è la **scalabilità**. Quando una richiesta I/O viene avviata, il thread del server che l'ha iniziata può essere rilasciato per servire altre richieste, invece di rimanere bloccato in attesa del completamento dell'operazione. Questo permette al server di gestire un numero molto maggiore di richieste concorrenti con un numero limitato di thread, migliorando l'utilizzo delle risorse e la reattività complessiva dell'applicazione, specialmente sotto carico elevato."

---

## 🎯 Conclusione Finale per l'Espositore

Con questa guida completa, hai ora a disposizione tutti gli strumenti concettuali e tecnici per presentare il progetto "Split Mate" a un livello accademico elevato. Ricorda di:

*   **Articolare le scelte di design**: Ogni decisione tecnica (architettura, database, framework, implementazione grafici) deve essere giustificata con pro e contro.
*   **Dimostrare comprensione del codice**: Non limitarti a descrivere cosa fa il codice, ma spiega *perché* è stato scritto in quel modo e quali pattern o principi di design applica.
*   **Enfatizzare la robustezza e la sicurezza**: Sottolinea come il progetto gestisca l'integrità dei dati, l'autenticazione e le vulnerabilità comuni (CORS, hashing password).
*   **Mostrare la visione d'insieme**: Collega le diverse parti del progetto (frontend, backend, database) in un flusso coerente, evidenziando come interagiscono per raggiungere gli obiettivi funzionali.
*   **Prepararti alle domande**: Utilizza la sezione di simulazione per anticipare le obiezioni e formulare risposte chiare e concise, dimostrando non solo conoscenza ma anche capacità critica.

Buona fortuna per la tua esposizione!

---
*Questo manuale è una risorsa viva, basata sull'analisi riga per riga del codice sorgente di Split Mate.*


## 🛠️ Capitolo 11: Gestione dello Schema, Ambiente e Principi SOLID

Per completare la comprensione architetturale, è essenziale analizzare come il progetto gestisce l'evoluzione del database, la configurazione dell'ambiente di esecuzione e l'aderenza ai principi di ingegneria del software.

### 11.1 Evoluzione del Database: Migrazioni EF Core

Il progetto utilizza le **Migrazioni di Entity Framework Core** per gestire le modifiche allo schema del database nel tempo. Questo approccio "Code-First" permette di definire il database tramite classi C# e di generare script SQL per aggiornare il database fisico.

*   **`ApplicationDbContextModelSnapshot.cs`**: Questo file, generato automaticamente, rappresenta lo stato attuale del modello del database. È fondamentale per EF Core per calcolare le differenze (diff) quando viene creata una nuova migrazione. Analizzando questo file, si può vedere la traduzione esatta delle classi C# in tabelle relazionali, inclusi i tipi di colonna (es. `decimal(10, 2)` per `Importo`), i vincoli di lunghezza (`HasMaxLength(100)` per `Nome`) e le chiavi esterne.
*   **Vantaggi**: Le migrazioni garantiscono che il database sia sempre sincronizzato con il codice, permettono il versionamento dello schema (come si fa con il codice sorgente tramite Git) e facilitano il deployment in ambienti diversi (sviluppo, test, produzione) eseguendo semplicemente le migrazioni pendenti all'avvio dell'applicazione (come visto in `Program.cs` con `context.Database.EnsureCreated()`, sebbene in produzione si preferisca `context.Database.Migrate()`).

### 11.2 Configurazione dell'Ambiente (Vite e ASP.NET Core)

La configurazione dell'ambiente è cruciale per il corretto funzionamento e la build dell'applicazione.

*   **Frontend (`vite.config.js`)**: Il progetto React utilizza **Vite** come build tool, scelto per la sua estrema velocità rispetto a Webpack. Il file di configurazione include i plugin per React e Tailwind CSS. Un dettaglio interessante è la configurazione `resolve.dedupe: ['react', 'react-dom', 'react-router-dom']`. Questa istruzione forza Vite a utilizzare una singola istanza di queste librerie, prevenendo errori complessi che possono verificarsi quando dipendenze multiple includono versioni diverse di React.
*   **Backend (`appsettings.json`)**: Questo file contiene la configurazione dell'applicazione ASP.NET Core. Definisce la stringa di connessione di default (`"Data Source=gestionespese.db"`), i livelli di logging e gli host consentiti. È importante notare che in `Program.cs` il percorso del database viene sovrascritto programmaticamente (`Path.Combine("C:\\home", "gestionespese.db")`) per adattarsi all'ambiente di deployment previsto (Azure App Service), dimostrando flessibilità nella gestione della configurazione.

### 11.3 Applicazione dei Principi SOLID

Il progetto Split Mate dimostra un'aderenza ai principi SOLID, fondamentali per la creazione di software manutenibile e scalabile:

*   **Single Responsibility Principle (SRP)**: Ogni classe ha una singola responsabilità. I Controller (es. `SpesaController`) gestiscono solo le richieste HTTP e l'orchestrazione, i Modelli (es. `Spesa`) rappresentano solo i dati, e il `ApplicationDbContext` gestisce solo la persistenza. Nel frontend, i componenti React (es. `GraficoTorta`) sono focalizzati su una singola parte dell'interfaccia.
*   **Open/Closed Principle (OCP)**: Il sistema è aperto all'estensione ma chiuso alla modifica. Ad esempio, l'aggiunta di un nuovo tipo di grafico nel frontend non richiederebbe la modifica del componente `DettaglioGruppo`, ma solo l'inclusione del nuovo componente.
*   **Dependency Inversion Principle (DIP)**: Come discusso in precedenza, il backend utilizza ampiamente la Dependency Injection. I controller dipendono da astrazioni (o istanze fornite dal framework) piuttosto che creare concretamente le proprie dipendenze, disaccoppiando il codice e facilitando il testing.

---

## 🚀 Capitolo 12: Evoluzione e Roadmap Futura

Un progetto software non è mai veramente "finito". Per dimostrare una visione a lungo termine, ecco alcune direzioni in cui Split Mate potrebbe evolvere:

1.  **Autenticazione Robusta (JWT e OAuth)**: Sostituire l'auto-provisioning con un sistema di registrazione completo, verifica email e autenticazione basata su **JSON Web Tokens (JWT)**. Integrare login tramite provider esterni (Google, GitHub) tramite OAuth 2.0 per migliorare l'onboarding.
2.  **Architettura a Microservizi**: Se l'applicazione dovesse scalare massivamente, il backend monolitico potrebbe essere suddiviso in microservizi (es. Servizio Utenti, Servizio Spese, Servizio Notifiche), comunicanti tramite code di messaggi (es. RabbitMQ) per una maggiore resilienza.
3.  **Real-time Updates (SignalR / WebSockets)**: Implementare aggiornamenti in tempo reale. Quando un utente aggiunge una spesa, gli altri membri del gruppo connessi dovrebbero vedere l'aggiornamento istantaneamente senza dover ricaricare la pagina, utilizzando tecnologie come SignalR nel backend ASP.NET Core.
4.  **Test Automatizzati (Unit e E2E)**: Introdurre una suite completa di test. Test unitari per la logica di bilanciamento nel backend (usando xUnit o NUnit) e test End-to-End (E2E) per il frontend (usando Cypress o Playwright) per garantire la stabilità durante le refattorizzazioni.
5.  **PWA (Progressive Web App)**: Trasformare il frontend React in una PWA per permettere l'installazione sui dispositivi mobili, il supporto offline e le notifiche push, avvicinando l'esperienza utente a quella di un'app nativa.

Queste evoluzioni dimostrano la capacità di pensare oltre l'implementazione attuale e di progettare per la scalabilità, la sicurezza e l'esperienza utente di livello enterprise.

---
*Questo manuale è una risorsa viva, basata sull'analisi riga per riga del codice sorgente di Split Mate.*
