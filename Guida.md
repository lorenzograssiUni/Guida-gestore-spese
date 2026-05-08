# 📔 Manuale Tecnico Split Mate: Architettura e Sviluppo Full-Stack

Benvenuti nel manuale definitivo di **Split Mate**. Questo documento non è una semplice guida, ma un trattato tecnico progettato per fornire una comprensione profonda e accademica del progetto. Studiare questo manuale significa acquisire la capacità di navigare nel codice, comprendere le scelte architettoniche e saper esporre il progetto con la padronanza di chi lo ha concepito.

---

## 🏛️ Capitolo 1: Architettura del Sistema

### 1.1 Paradigma Full-Stack
Split Mate è implementato seguendo il paradigma **Decoupled Architecture** (Architettura Disaccoppiata). Il sistema è diviso in due entità distinte che comunicano esclusivamente tramite protocolli standard:

1.  **Server-Side (Backend)**: Un'applicazione ASP.NET Core che funge da *Provider di Risorse*. Espone endpoint RESTful e gestisce la persistenza dei dati.
2.  **Client-Side (Frontend)**: Una Single-Page Application (SPA) sviluppata in React. È il *Consumatore di Risorse* che gestisce lo stato dell'interfaccia e l'esperienza utente.

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

---

## ⚛️ Capitolo 4: Il Frontend (React & Vite)

### 4.1 Gestione dello Stato Globale
In `App.jsx`, lo stato dell'utente è persistito nel `localStorage`. Questo permette all'utente di rimanere loggato anche dopo il refresh della pagina, migliorando la **User Retention**.

### 4.2 Componenti e Hooks
Il frontend sfrutta gli **Hooks** di React (`useState`, `useEffect`) per gestire il ciclo di vita dei dati:
*   `useEffect`: Utilizzato per scatenare il recupero dei dati dal server non appena un componente viene visualizzato (es. caricamento dei gruppi nella Home).
*   **Rendering Condizionale**: L'interfaccia cambia dinamicamente in base allo stato (es. se `utente` è null, mostra il form di login, altrimenti la dashboard).

### 4.3 Visualizzazione Dati (SVG & Math)
I grafici (`GraficoTorta.jsx`) non usano librerie esterne pesanti, ma sono disegnati manualmente tramite **SVG (Scalable Vector Graphics)**.
*   Viene utilizzata la trigonometria (`Math.cos`, `Math.sin`) per calcolare i punti delle fette della torta in base alla percentuale di spesa.

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
