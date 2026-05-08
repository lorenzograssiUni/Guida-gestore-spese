# 💲 Guida Definitiva a Split Mate: Gestore Spese di Gruppo

Benvenuti nella guida ufficiale per comprendere, studiare ed esporre il progetto **Split Mate**. Questa guida è pensata per essere letta da chiunque: dal programmatore esperto al compagno di studi che non ha mai visto una riga di codice.

L'obiettivo è trasformare concetti tecnici complessi in concetti semplici e quotidiani, permettendo a tutti di saper spiegare il progetto "alla perfezione".

---

## 📖 1. Il Progetto in Breve
**Split Mate** è una Web Application (un sito web interattivo) che risolve un problema comune: **chi deve soldi a chi?**
Immagina una vacanza con amici o una cena: qualcuno paga la pizza, qualcuno le bibite, qualcuno il taxi. Alla fine, fare i conti è un incubo. Split Mate lo fa in automatico, minimizzando il numero di scambi di denaro necessari.

---

## 💡 2. Glossario: I termini tecnici spiegati "facile"
Prima di entrare nel codice, capiamo le parole che useremo. Se devi esporre il progetto, usa queste definizioni:

| Termine | Spiegazione Semplice | Esempio Reale |
| :--- | :--- | :--- |
| **Frontend** | Tutto ciò che l'utente vede e clicca. La "vetrina" del negozio. | I pulsanti, i grafici e i moduli dell'app. |
| **Backend** | Il "cervello" invisibile che fa i calcoli e salva i dati. La "cucina" del ristorante. | La logica che decide chi deve dare soldi a chi. |
| **API** | Il "cameriere" che porta le ordinazioni dal tavolo (Frontend) alla cucina (Backend). | Quando clicchi "Salva spesa", l'API porta i dati al server. |
| **Database** | L'archivio digitale dove le informazioni restano salvate per sempre. | La lista degli utenti e delle spese salvata nel file `gestionespese.db`. |
| **REST** | Un modo standard e ordinato di organizzare le API. | Usare indirizzi chiari come `/api/Spesa` per gestire le spese. |
| **Swagger** | Una pagina web di servizio che elenca tutte le API disponibili per provarle. | Una sorta di "manuale d'uso" interattivo per gli sviluppatori. |
| **SPA (Single Page Application)** | Un sito che non si ricarica mai completamente quando clicchi, rendendolo fluidissimo. | Come l'app di Instagram o Gmail. |
| **ORM (Entity Framework)** | Un traduttore che permette al codice (C#) di parlare con il Database senza scrivere linguaggi complessi (SQL). | Trasforma una "classe" Spesa in una "tabella" nel database. |
| **CORS** | Una guardia di sicurezza che decide chi può parlare con il server. | Permette al sito (Vercel) di comunicare con il server (Azure). |

---

## 🛠 3. Lo Stack Tecnologico (I Software utilizzati)
Il progetto è costruito con le tecnologie più moderne del settore:

### ⚙️ Backend (Il Motore)
*   **ASP.NET Core 10.0**: Il framework di Microsoft per creare server veloci e sicuri.
*   **Entity Framework Core**: Per gestire il database in modo intelligente.
*   **SQLite**: Un database leggero che salva tutto in un unico file, perfetto per questo progetto.
*   **BCrypt**: Un sistema di sicurezza per "criptare" le password (non vengono salvate in chiaro, così nessuno può rubarle).

### 🖥️ Frontend (L'Interfaccia)
*   **React**: La libreria più usata al mondo per creare interfacce interattive.
*   **Vite**: Lo strumento ultra-veloce che prepara il codice per essere letto dal browser.
*   **Tailwind CSS**: Un sistema per dare uno stile moderno e pulito all'app (colori, bordi arrotondati, ombre).
*   **Lucide React**: La libreria che fornisce le icone eleganti che vedi nell'app.

---

## 🏗 4. Architettura: Come "viaggiano" i dati
Il flusso è circolare e semplicissimo:
1.  **L'Utente** compie un'azione sul **Frontend** (es. inserisce una spesa).
2.  Il **Frontend** invia una richiesta tramite **API** al **Backend**.
3.  Il **Backend** riceve i dati, fa i calcoli e li salva nel **Database**.
4.  Il **Backend** risponde al **Frontend** dicendo "Fatto!".
5.  Il **Frontend** si aggiorna e mostra i nuovi dati all'utente (es. il grafico a torta cambia).

---

## 🔍 5. Analisi del Codice: I file che contano
Se il professore ti chiede "Dove succede questa cosa?", ecco la risposta:

### 📍 Backend (Cartella `gestione-spese`)
*   **`Program.cs`**: È il punto di partenza. Qui viene configurato Swagger, il Database e chi può accedere (CORS).
*   **`ApplicationDbContext.cs`**: È la "mappa" del database. Qui diciamo che un Gruppo può avere tanti Utenti e tante Spese.
*   **`GruppoController.cs`**: Contiene la logica dei gruppi e, soprattutto, l'**algoritmo di bilancio** (il calcolo dei debiti).
*   **`SpesaController.cs`**: Gestisce l'inserimento delle spese e decide come dividerle tra i membri selezionati.
*   **`AuthController.cs`**: Gestisce l'accesso. Una particolarità: se inserisci una mail nuova, l'app ti registra automaticamente!

### 📍 Frontend (Cartella `frontend-gestione-spese`)
*   **`App.jsx`**: È il cuore del sito. Decide se mostrarti la pagina di Login o la Home.
*   **`HomePage.jsx`**: Mostra i tuoi gruppi e ti permette di crearne di nuovi o unirti tramite codice.
*   **`DettaglioGruppo.jsx`**: La pagina dove vedi la lista delle spese e i membri del gruppo.
*   **`RiepilogoGruppo.jsx`**: Qui avvengono le magie grafiche (i grafici a torta e istogrammi) per vedere i debiti.

---

## 🧠 6. L'Algoritmo di Pareggio (Il "Cervello")
Questa è la parte più importante da saper spiegare. Come fa l'app a dire "Marco deve dare 10€ a Luca"?

1.  **Calcolo del Saldo**: Per ogni persona, l'app calcola: `Quanto ha pagato - Quanto doveva pagare`.
    *   Se il risultato è **positivo**, la persona è un **Creditore** (deve ricevere soldi).
    *   Se il risultato è **negativo**, la persona è un **Debitore** (deve dare soldi).
2.  **Pareggio Ottimizzato**: L'app prende il debitori più grande e il creditore più grande e li fa "incontrare".
    *   *Esempio*: Se io devo 50€ e tu devi riceverne 30€, io ti do 30€. Ora io devo ancora 20€ a qualcun altro e tu sei a posto.
3.  **Risultato**: Questo continua finché tutti i debiti sono estinti con il **minor numero di passaggi possibili**.

---

## 🎨 7. Visualizzazione Dati: I Grafici
Per rendere tutto più comprensibile, abbiamo usato dei grafici:
*   **Grafico a Torta**: Mostra visivamente chi ha speso di più nel gruppo. Se una fetta è enorme, quella persona è il "finanziatore" del gruppo!
*   **Grafico a Istogramma**: Mostra le spese nel tempo o i saldi individuali.

---

## 🎤 8. Consigli per l'Esposizione (Come fare colpo)
1.  **Inizia dal Problema**: "Abbiamo creato Split Mate perché dividere le spese tra amici è sempre un caos".
2.  **Mostra il Flusso**: Spiega come un utente si registra, crea un gruppo e invita gli amici col codice.
3.  **Parla della Sicurezza**: Menziona che le password sono criptate con **BCrypt**.
4.  **Enfatizza l'Algoritmo**: Spiega che non facciamo solo somme, ma ottimizziamo i rimborsi per fare meno transazioni.
5.  **Tecnologie Moderne**: Di' con orgoglio che avete usato **.NET 10** e **React con Vite**, gli standard più recenti dell'industria.

---

## 🏁 Conclusione
Split Mate non è solo un esercizio di stile, ma un'applicazione completa, sicura e pronta all'uso. Ogni pezzo del puzzle, dal database SQLite all'interfaccia React, è stato scelto per essere efficace e facile da mantenere.

---
*Guida aggiornata basata sull'analisi del codice sorgente della repository ufficiale.*
