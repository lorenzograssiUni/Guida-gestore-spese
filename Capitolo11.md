# Capitolo 11 — Componenti Principali

In questo capitolo analizziamo i componenti React che formano l'interfaccia utente dell'applicazione. Ogni componente è responsabile di una schermata o funzionalità specifica, riceve dati tramite `props` e comunica con il genitore tramite callback.

---

## 11.1 LoginForm — Autenticazione e Registrazione

### Ruolo e Responsabilità

`LoginForm` è il componente mostrato quando nessun utente è salvato in `localStorage`. Gestisce sia il **login** che la **registrazione** con un'unica form, sfruttando la logica di **Auto-Provisioning** implementata in `AuthController` (vedi Capitolo 7.1).

### Struttura del Componente

```jsx
// src/components/LoginForm.jsx
import { useState } from 'react';
import { login } from '../api/api';

function LoginForm({ onLoginSuccess }) {
    const [email, setEmail]       = useState('');
    const [password, setPassword] = useState('');
    const [errore, setErrore]     = useState('');

    const handleSubmit = async (e) => {
        e.preventDefault();
        setErrore('');
        const res = await login(email, password);

        if (res.ok) {
            const utente = await res.json();
            onLoginSuccess(utente);          // callback verso App.jsx
        } else if (res.status === 401) {
            setErrore('Password errata. Riprova.');
        } else {
            setErrore('Errore durante il login. Riprova.');
        }
    };
    // ... JSX
}
```

### Flusso Login / Registrazione

```
Utente inserisce email + password
          ↓
    POST /auth/login
          ↓
   ┌──────┴──────────┐
   │                 │
 200 OK           404 Not Found
(utente esiste)  (email non trovata)
   │                 │
   │          Backend crea utente
   │          e risponde 200 OK
   │                 │
   └────────┬────────┘
            ↓
      onLoginSuccess(utente)
            ↓
   App.jsx salva in localStorage
   e mostra HomePage
```

Il pulsante è un solo **"Accedi / Registrati"** — non ci sono due form separate. Se l'email non esiste, il backend la crea automaticamente (Auto-Provisioning).

### Gestione degli Errori

| Status HTTP | Significato              | Messaggio mostrato            |
|-------------|--------------------------|-------------------------------|
| `200 OK`    | Login/registrazione OK   | —  (redirect alla HomePage)   |
| `401`       | Password errata          | *"Password errata. Riprova."* |
| Altro       | Errore generico/rete     | *"Errore durante il login."*  |

### Props del Componente

| Prop             | Tipo       | Descrizione                                         |
|------------------|------------|-----------------------------------------------------|
| `onLoginSuccess` | `function` | Callback chiamata con l'oggetto utente dal backend  |

---

## 11.2 HomePage — Lista Gruppi e Codice Invito

### Ruolo e Responsabilità

`HomePage` è la schermata principale dopo il login. Mostra tutti i gruppi di cui l'utente fa parte, permette di **crearne di nuovi** (aprendo un Modal) e di **unirsi a un gruppo esistente** tramite codice invito.

### Struttura delle Props

```jsx
function HomePage({ utente, refreshKey, onOpenModal, onApriGruppo }) { ... }
```

| Prop          | Tipo       | Descrizione                                                       |
|---------------|------------|-------------------------------------------------------------------|
| `utente`      | `object`   | L'utente loggato (da `localStorage`)                              |
| `refreshKey`  | `number`   | Chiave incrementale: quando cambia, scatta il re-fetch dei gruppi |
| `onOpenModal` | `function` | Apre il Modal per creare un nuovo gruppo                          |
| `onApriGruppo`| `function` | Naviga al `DettaglioGruppo` passando l'id del gruppo              |

### Pattern "Derived State" per il Re-Fetch

React non esegue effetti collaterali durante il render, ma il componente usa un pattern speciale per rilevare quando `refreshKey` cambia:

```jsx
const [prevKey, setPrevKey] = useState(refreshKey);
if (refreshKey !== prevKey) {
    setPrevKey(refreshKey);
    setLoading(true);    // ← triggera il re-fetch al prossimo render
}
```

> ⚠️ Questo è un uso avanzato del render: modificare lo stato **durante** il render (senza `useEffect`) è consentito in React solo per aggiornamenti derivati dallo stato precedente. React gestisce questo caso rieseguendo immediatamente il render con il nuovo stato prima di dipingere il DOM.

### Fetch dei Gruppi

```jsx
useEffect(() => {
    if (!utente?.id) return;
    fetch(`${API_URL}/Gruppo/miei-gruppi/${utente.id}`)
        .then(res => res.json())
        .then(data => {
            setGruppi(data);
            setLoading(false);
        })
        .catch(err => setStatus("Errore di connessione al server."));
}, [refreshKey, utente?.id, API_URL]);
```

Le dipendenze dell'`useEffect` includono `refreshKey`: ogni volta che App.jsx incrementa questa chiave (dopo la creazione di un gruppo), la `useEffect` si riesegue e carica i gruppi aggiornati.

### Funzione Join con Codice Invito

```jsx
const handleJoin = async (e) => {
    e.preventDefault();
    const response = await fetch(
        `${API_URL}/Gruppo/join?codice=${codice}&utenteId=${utente.id}`,
        { method: 'POST' }
    );
    if (response.ok) {
        setCodice('');
        window.location.reload();  // ← ricarica la pagina per aggiornare la lista
    } else {
        alert("Codice non valido o sei già presente in questo gruppo.");
    }
};
```

> 📌 **Nota**: dopo il join viene usato `window.location.reload()` invece di incrementare il `refreshKey`. Questo è un approccio più semplice ma meno elegante: ricarica l'intera applicazione React invece di fare solo il re-fetch.

### Stati della UI

| Condizione                      | UI mostrata                                    |
|---------------------------------|------------------------------------------------|
| `loading === true`              | Spinner animato con `animate-spin`             |
| `gruppi.length === 0`           | Empty state con messaggio e bordo tratteggiato |
| `gruppi.length > 0`             | Grid responsiva di `GruppoCard`                |
| `!utente`                       | Messaggio "Effettua il login per continuare"   |

---

## 11.3 DettaglioGruppo — Membri, Spese e Validazioni

### Ruolo e Responsabilità

`DettaglioGruppo` è il componente più complesso del frontend. Mostra i dettagli di un gruppo specifico: lista dei membri, cronologia delle spese, e fornisce le azioni per aggiungere/rimuovere membri, aggiungere/modificare/eliminare spese ed eliminare il gruppo.

### Props del Componente

```jsx
function DettaglioGruppo({ gruppoId, onBack, onApriRiepilogo }) { ... }
```

| Prop             | Tipo       | Descrizione                                    |
|------------------|------------|------------------------------------------------|
| `gruppoId`       | `number`   | ID del gruppo da visualizzare                  |
| `onBack`         | `function` | Torna alla `HomePage`                          |
| `onApriRiepilogo`| `function` | Naviga al `RiepilogoGruppo`                    |

### Funzione `caricaDatiGruppo`

La funzione di caricamento dati è estratta dall'`useEffect` e resa riutilizzabile, perché deve essere chiamata dopo ogni operazione di modifica (aggiungi membro, elimina spesa, ecc.):

```jsx
const caricaDatiGruppo = async () => {
    try {
        const resGruppo = await getGruppo(gruppoId);
        if (!resGruppo.ok) throw new Error("Gruppo non trovato");
        const dataGruppo = await resGruppo.json();
        setGruppo(dataGruppo);
    } catch (error) {
        console.error("Errore nel caricamento dati:", error);
    } finally {
        setLoading(false);
    }
};

useEffect(() => {
    if (gruppoId) caricaDatiGruppo();
}, [gruppoId]);
```

### Aggiunta Membro "Bot"

Il sistema supporta l'aggiunta di persone **non registrate** come utenti "bot". La funzione genera un'email fittizia per soddisfare il vincolo di unicità del database:

```jsx
const handleAggiungiMembro = async (e) => {
    e.preventDefault();
    const nuovoUtente = {
        nome: nuovoMembroNome,
        email: `${nuovoMembroNome.toLowerCase().replace(/\s+/g, '')}_${Math.floor(Math.random() * 10000)}@bot.it`
    };
    const res = await aggiungiBotAlGruppo(gruppoId, nuovoUtente);
    if (res.ok) caricaDatiGruppo();
};
```

> 💡 **Perché i bot?** L'app è pensata per dividere le spese anche con persone che non hanno un account. I bot sono utenti "fantoccio" con email casuale che non possono fare login, ma possono essere associati alle spese.

### Validazione Rimozione Membro

Prima di rimuovere un membro, il componente verifica lato **frontend** se quel membro ha spese registrate:

```jsx
const handleEliminaAmico = async (amicoId, nomeAmico) => {
    const eCoinvolto = gruppo?.spese?.some(s => s.chiPagaID === amicoId);
    if (eCoinvolto) {
        alert(`Non puoi eliminare ${nomeAmico} perché ha delle spese registrate.`);
        return;  // ← blocca la chiamata API prima ancora di farla
    }
    // ... conferma e chiamata API
};
```

> ⚠️ Questa è una **validazione lato client**: utile per UX rapida, ma non è una garanzia di sicurezza. Il backend ha i propri vincoli (`DeleteBehavior.Restrict`) che impediscono comunque l'eliminazione a livello di database.

### Gestione dei Modal

Il componente gestisce due modal sovrapposti tramite stato locale:

```jsx
const [isModalSpesaOpen, setIsModalSpesaOpen] = useState(false);
const [spesaInModifica, setSpesaInModifica]   = useState(null);
```

- `isModalSpesaOpen = true` → mostra `ModalNuovaSpesa`
- `spesaInModifica !== null` → mostra `ModalModificaSpesa` con i dati della spesa da modificare

La lista delle spese mostra le più recenti in cima usando `.slice().reverse()` — che crea una copia prima di invertire, per non mutare l'array originale nello stato React.

---

## 11.4 ModalNuovaSpesa / ModalModificaSpesa — Form Controllati

### Concetto: Form Controllati in React

Un **form controllato** è un form in cui ogni campo `<input>` ha il proprio valore gestito dallo stato React, non dal DOM. Ogni carattere digitato aggiorna lo stato, e lo stato aggiorna il valore del campo:

```
Utente digita → onChange → setState → re-render → value={state}
```

Questo crea un **unico source of truth**: React ha sempre il controllo del valore del campo, e può validarlo o trasformarlo ad ogni keystroke.

### Stato del ModalNuovaSpesa

```jsx
const [descrizione, setDescrizione]         = useState('');
const [importo, setImporto]                 = useState('');
const [chiPagaId, setChiPagaId]             = useState(membri[0]?.id ?? '');
const [dividiConTutti, setDividiConTutti]   = useState(true);
const [utentiCoinvolti, setUtentiCoinvolti] = useState([]);  // array di ID
const [isSubmitting, setIsSubmitting]       = useState(false);
const [error, setError]                     = useState(null);
```

### Gestione Checkbox Multipli

La selezione dei partecipanti usa un array di ID. La funzione `handleCheckboxChange` usa un pattern **toggle**: aggiunge l'ID se non c'è, lo rimuove se c'è già:

```jsx
const handleCheckboxChange = (membroId) => {
    setUtentiCoinvolti(prev => {
        if (prev.includes(membroId)) {
            return prev.filter(id => id !== membroId);  // rimuove
        } else {
            return [...prev, membroId];                  // aggiunge
        }
    });
};
```

### Costruzione del DTO e Invio

```jsx
const spesaData = {
    gruppo_ID:           gruppoId,
    chiPaga_ID:          Number(chiPagaId),
    importo:             parseFloat(importo),
    descrizione:         descrizione,
    utentiCoinvoltiIds:  dividiConTutti ? [] : utentiCoinvolti
    //                   ↑ array vuoto = backend divide con tutti
};

const response = await fetch(`${API_BASE_URL}/spesa`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify(spesaData),
});
```

> 📌 La scelta di passare `utentiCoinvoltiIds: []` quando si divide con tutti è una **convenzione backend**: `SpesaController` interpreta un array vuoto come "includi tutti i membri del gruppo" (vedi Capitolo 7.3).

### Validazione Pre-Invio

```jsx
if (!dividiConTutti && utentiCoinvolti.length === 0) {
    setError("Seleziona almeno un membro con cui dividere la spesa.");
    setIsSubmitting(false);
    return;
}
```

La validazione avviene **prima** della chiamata API per evitare richieste inutili al backend.

### Differenza ModalNuovaSpesa vs ModalModificaSpesa

| Aspetto           | `ModalNuovaSpesa`           | `ModalModificaSpesa`              |
|-------------------|-----------------------------|-----------------------------------|
| Stato iniziale    | Campi vuoti                 | Pre-popolati con i dati della spesa |
| Metodo HTTP       | `POST /spesa`               | `PUT /spesa/{id}`                 |
| Prop ricevuta     | `gruppoId`, `membri`        | `spesa` (oggetto), `membri`       |
| Callback successo | `onSpesaAggiunta()`         | `onSpesaModificata()`             |

### Pattern "Portal" per i Modal

I modal usano `fixed inset-0` con z-index alto per sovrapporsi all'intera pagina:

```jsx
<div className="fixed inset-0 bg-black/60 backdrop-blur-sm flex justify-center items-center p-4 z-50">
    <div className="bg-white rounded-3xl w-full max-w-md shadow-2xl ...">
        {/* contenuto del modal */}
    </div>
</div>
```

`bg-black/60` crea uno sfondo semitrasparente, `backdrop-blur-sm` sfoca il contenuto dietro. Questo è possibile grazie a Tailwind CSS e alle classi di opacità.

---

## 11.5 RiepilogoGruppo — Debiti Calcolati e Salda Debito

### Ruolo e Responsabilità

`RiepilogoGruppo` è la schermata finale del flusso principale. Mostra il totale delle spese del gruppo, i grafici SVG (torta e istogramma, vedi Capitolo 12), e soprattutto la lista dei debiti calcolati dal backend con la possibilità di **segnarli come saldati**.

### Doppio Fetch al Caricamento

```jsx
const caricaDati = async () => {
    // 1. Dati del gruppo (utenti + spese)
    const resGruppo = await getGruppo(gruppoId);
    const dataGruppo = await resGruppo.json();
    setGruppo(dataGruppo);

    // 2. Bilanci calcolati (lista debiti)
    const resBilanci = await getBilanciGruppo(gruppoId);
    const dataBilanci = await resBilanci.json();
    setRiepiloghi(dataBilanci);
};
```

Le due chiamate sono **sequenziali** (non parallele con `Promise.all`): il calcolo dei bilanci dipende dallo stesso gruppo, ma sono endpoint separati.

### Calcolo delle KPI nel Frontend

Le statistiche riassuntive sono calcolate **nel frontend** dagli array già caricati:

```jsx
const totaleSpese  = listaSpese.reduce((acc, s) => acc + s.importo, 0);
const debitiAperti = riepiloghi.filter(r => !r.pagato).length;
const debitiSaldati = riepiloghi.filter(r => r.pagato).length;
```

> 📌 Questo è un esempio di **logica derivata**: invece di chiedere questi valori al backend, vengono calcolati in tempo reale dai dati già presenti in memoria. Riduce le chiamate API e mantiene i dati consistenti.

### Funzione Salda Debito

```jsx
const handleSaldaDebito = async (riepilogoId) => {
    if (window.confirm("Confermi che il debito è stato saldato?")) {
        const res = await saldaDebito(riepilogoId);
        if (res.ok) caricaDati();  // re-fetch completo per aggiornare tutto
    }
};
```

Dopo aver saldato, viene ricaricato l'intero stato (`caricaDati()`) invece di aggiornare solo il singolo debito. Questo garantisce che tutti i dati siano sincronizzati col backend.

### Rendering dei Debiti

Ogni debito ha due stati visivi distinti:

```jsx
<div className={`p-4 rounded-2xl border transition-all ${
    r.pagato
        ? 'bg-gray-50 border-gray-100 opacity-50'     // ← saldato: sbiadito
        : 'bg-white border-gray-200 hover:border-blue-200' // ← aperto: attivo
}`}>
    <p>
        <span className="text-blue-600">{r.da}</span>   {/* debitore */}
        <span className="text-gray-400 mx-2">→</span>
        <span className="text-gray-800">{r.a}</span>    {/* creditore */}
    </p>
    {r.pagato
        ? <span>✓ Saldato</span>
        : <button onClick={() => handleSaldaDebito(r.id)}>Segna come saldato</button>
    }
</div>
```

### Schema del Flusso Dati in RiepilogoGruppo

```
         RiepilogoGruppo
              │
    ┌─────────┴──────────┐
    │                    │
getGruppo(id)    getBilanciGruppo(id)
    │                    │
 setGruppo()        setRiepiloghi()
    │                    │
    ├── listaMembri       ├── debiti aperti
    ├── listaSpese        └── debiti saldati
    │
    ├──→ GraficoTorta (props: spese, membri)
    ├──→ GraficoIstogramma (props: spese, membri)
    └──→ totaleSpese (calcolato con .reduce)
```

---

## Riepilogo del Capitolo

| Componente          | File                        | Tipo     | Gestisce                                    |
|---------------------|-----------------------------|----------|---------------------------------------------|
| `LoginForm`         | `components/LoginForm.jsx`  | Puro     | Form email/password, chiamata API login     |
| `HomePage`          | `pages/HomePage.jsx`        | Pagina   | Lista gruppi, join codice, apertura modal   |
| `DettaglioGruppo`   | `pages/DettaglioGruppo.jsx` | Pagina   | Membri, spese, CRUD, validazioni UI         |
| `ModalNuovaSpesa`   | `components/ModalNuovaSpesa.jsx` | Modal | Form spesa con checkbox partecipanti       |
| `ModalModificaSpesa`| `components/ModalModificaSpesa.jsx` | Modal | Pre-popola form da spesa esistente        |
| `RiepilogoGruppo`   | `pages/RiepilogoGruppo.jsx` | Pagina   | KPI, grafici SVG, lista debiti, salda      |

I pattern fondamentali usati in questi componenti:
- **Form controllati** — stato React = source of truth del form
- **Callback props** — i figli comunicano al genitore tramite funzioni ricevute come prop
- **Re-fetch esplicito** — dopo ogni mutazione si ricaricano i dati invece di aggiornare solo lo stato locale
- **Validazione lato client** — check pre-API per migliorare UX (non sostituisce il backend)
