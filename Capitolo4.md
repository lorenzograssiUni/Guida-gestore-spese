# Capitolo 4 — Frontend: React, Componenti, Stato e Hooks

> Questo capitolo copre l'intera architettura frontend di Split Mate: come funziona React, il Virtual DOM, JSX, la gestione dello stato con hooks, il flusso dati tra componenti, e come l'app interagisce con il backend tramite `api.js`.

---

## 4.1 Cos'è React e Perché lo Usiamo

**React** è una libreria JavaScript per costruire interfacce utente attraverso **componenti** riutilizzabili. Non è un framework completo (come Angular) — è focalizzato esclusivamente sul layer di presentazione (la "View" nel pattern MVC).

Split Mate usa React per tre motivi principali:

- **Componenti dichiarativi**: descrivi come appare l'UI in ogni stato, React si occupa di aggiornarla
- **Virtual DOM**: aggiornamenti efficienti — React modifica solo le parti del DOM che cambiano
- **Ecosistema**: Vite per il build, `fetch` API per le chiamate REST, ampia disponibilità di librerie

---

## 4.2 Il DOM e il Virtual DOM

### Il DOM (Document Object Model)

Il **DOM** è la rappresentazione ad albero che il browser crea da un documento HTML. Ogni elemento (`<div>`, `<p>`, `<button>`) diventa un nodo dell'albero. JavaScript può modificare questo albero per aggiornare l'interfaccia.

```
HTML                          DOM (albero)
───────────────────────       ──────────────────────
<div id="app">                   #app (div)
  <h1>Split Mate</h1>              ├── h1: "Split Mate"
  <ul>                             └── ul
    <li>Vacanza</li>                    ├── li: "Vacanza"
    <li>Casa</li>                       └── li: "Casa"
  </ul>
</div>
```

**Il problema**: manipolare il DOM direttamente è costoso. Ogni modifica può innescare un *reflow* (ricalcolo layout) e un *repaint* (ridisegno) del browser. Con molti aggiornamenti frequenti, l'app diventa lenta.

### Il Virtual DOM — La Soluzione di React

Il **Virtual DOM** è una copia leggera del DOM reale, mantenuta in memoria da React come oggetti JavaScript. Quando lo stato cambia, React:

1. Crea un nuovo Virtual DOM con le modifiche
2. Lo confronta con il precedente tramite un algoritmo chiamato **Diffing**
3. Calcola il set minimo di operazioni necessarie (**Reconciliation**)
4. Applica **solo quelle modifiche** al DOM reale

```
Stato cambia (es. spesa aggiunta)
        |
React crea nuovo Virtual DOM
        |
Diffing: confronta vecchio vs nuovo V-DOM
        |
        ├── <h1> invariato → nessuna operazione
        ├── <ul> invariato → nessuna operazione
        └── <li> nuova spesa → inserisci solo questo nodo nel DOM reale
        |
DOM reale aggiornato con chirurgicità
```

**Vantaggio**: invece di ridisegnare tutta la pagina, React aggiorna solo il `<li>` della nuova spesa. L'utente percepisce una risposta istantanea anche con liste lunghe.

---

## 4.3 JSX — JavaScript + XML

**JSX** è una sintassi che permette di scrivere HTML all'interno del codice JavaScript. Non è HTML vero — è zucchero sintattico che Vite/Babel trasforma in chiamate `React.createElement()`.

```jsx
// JSX — quello che scrivi
function ListaSpese({ spese }) {
  return (
    <div className="lista-spese">
      <h2>Spese del gruppo</h2>
      <ul>
        {spese.map(spesa => (
          <li key={spesa.id}>
            {spesa.descrizione}: €{spesa.importo}
          </li>
        ))}
      </ul>
    </div>
  );
}

// Quello che Babel genera (semplificato)
function ListaSpese({ spese }) {
  return React.createElement('div', { className: 'lista-spese' },
    React.createElement('h2', null, 'Spese del gruppo'),
    React.createElement('ul', null,
      spese.map(spesa =>
        React.createElement('li', { key: spesa.id },
          `${spesa.descrizione}: €${spesa.importo}`
        )
      )
    )
  );
}
```

**Regole fondamentali di JSX in Split Mate:**

- `className` invece di `class` (parola riservata in JS)
- Ogni componente deve restituire **un solo elemento radice** (o un Fragment `<>...</>`)
- Le espressioni JavaScript vanno tra `{}`
- I componenti personalizzati iniziano sempre con **lettera maiuscola** (`<ListaSpese>`)
- Ogni elemento in una lista deve avere una prop `key` univoca

---

## 4.4 Componenti — I Mattoni di React

Un **componente** React è una funzione JavaScript che accetta props in ingresso e restituisce JSX. È l'unità base di cui è composta tutta l'interfaccia di Split Mate.

### Struttura di un Componente

```jsx
// src/components/CardSpesa.jsx

// Props: dati che arrivano dal componente padre (sola lettura)
function CardSpesa({ spesa, onElimina, onModifica }) {
  return (
    <div className="card-spesa">
      <span className="descrizione">{spesa.descrizione}</span>
      <span className="importo">€{spesa.importo.toFixed(2)}</span>
      <span className="pagatore">Pagato da: {spesa.pagatoreNome}</span>
      <div className="azioni">
        <button onClick={() => onModifica(spesa)}>✏️</button>
        <button onClick={() => onElimina(spesa.id)}>🗑️</button>
      </div>
    </div>
  );
}

export default CardSpesa;
```

### Props — Comunicazione Padre → Figlio

Le **props** (abbreviazione di *properties*) sono il meccanismo con cui un componente padre passa dati a un figlio. Come spiega il prof. Bonura, sono **sola lettura**: il figlio non può modificarle direttamente.

```jsx
// Componente padre
function ListaSpese({ gruppoId }) {
  const [spese, setSpese] = useState([]);

  return (
    <div>
      {spese.map(spesa => (
        // Passo le props a CardSpesa
        <CardSpesa
          key={spesa.id}
          spesa={spesa}                              // prop: oggetto spesa
          onElimina={(id) => eliminaSpesa(id)}       // prop: callback function
          onModifica={(s) => setSpesaDaModificare(s)} // prop: callback function
        />
      ))}
    </div>
  );
}
```

**Flusso dati in React è sempre unidirezionale**: i dati scendono dall'alto (padre) verso il basso (figlio) tramite props. Per comunicare verso l'alto, il figlio chiama una **callback** passata dal padre come prop.

---

## 4.5 useState — La Gestione dello Stato Locale

Lo **stato** è qualsiasi dato che, quando cambia, deve causare un aggiornamento dell'interfaccia. `useState` è il hook fondamentale per gestire lo stato all'interno di un componente.

```jsx
import { useState } from 'react';

function ModalNuovaSpesa({ gruppoId, onChiudi, onSpesaCreata }) {
  // [valoreDelloStato, funzioneCheLoAggiorna] = useState(valoreIniziale)
  const [descrizione, setDescrizione] = useState('');
  const [importo, setImporto] = useState('');
  const [loading, setLoading] = useState(false);
  const [errore, setErrore] = useState(null);

  const handleSubmit = async (e) => {
    e.preventDefault();
    setLoading(true);
    setErrore(null);

    try {
      const nuovaSpesa = await creaSpesa({ descrizione, importo: parseFloat(importo), gruppoId });
      onSpesaCreata(nuovaSpesa); // comunica al padre che la spesa è stata creata
      onChiudi();
    } catch (err) {
      setErrore('Errore durante la creazione della spesa');
    } finally {
      setLoading(false);
    }
  };

  return (
    <form onSubmit={handleSubmit}>
      <input
        value={descrizione}
        onChange={(e) => setDescrizione(e.target.value)}
        placeholder="Descrizione"
      />
      <input
        value={importo}
        onChange={(e) => setImporto(e.target.value)}
        type="number"
        placeholder="Importo €"
      />
      {errore && <p className="errore">{errore}</p>}
      <button type="submit" disabled={loading}>
        {loading ? 'Salvataggio...' : 'Aggiungi Spesa'}
      </button>
    </form>
  );
}
```

### Regole Critiche di useState

```jsx
// ❌ NON modificare lo stato direttamente — React non rileva il cambiamento
spese.push(nuovaSpesa);   // sbagliato!

// ✅ Usa sempre la funzione setter — crea un nuovo array
setSpese([...spese, nuovaSpesa]);   // corretto: spread + nuovo elemento

// ❌ NON aggiornare lo stato in base al valore attuale senza callback
setContatore(contatore + 1);   // può causare race conditions

// ✅ Usa la forma funzionale per aggiornamenti basati sul valore precedente
setContatore(prev => prev + 1);   // sicuro, sempre aggiornato
```

---

## 4.6 useEffect — Effetti Collaterali e Ciclo di Vita

`useEffect` è il hook per gestire gli **effetti collaterali**: operazioni che avvengono *dopo* il rendering (chiamate API, sottoscrizioni, manipolazione diretta del DOM, timer).

```jsx
import { useState, useEffect } from 'react';

function DettaglioGruppo({ gruppoId }) {
  const [spese, setSpese] = useState([]);
  const [loading, setLoading] = useState(true);

  // useEffect(funzione, [dipendenze])
  // Si esegue dopo ogni render in cui le dipendenze cambiano
  useEffect(() => {
    setLoading(true);

    // Chiamata API al backend
    getSpese(gruppoId)
      .then(data => setSpese(data))
      .catch(err => console.error(err))
      .finally(() => setLoading(false));

  }, [gruppoId]); // ← array di dipendenze: si riesegue ogni volta che gruppoId cambia

  if (loading) return <p>Caricamento...</p>;

  return (
    <ul>
      {spese.map(s => <li key={s.id}>{s.descrizione}: €{s.importo}</li>)}
    </ul>
  );
}
```

### Varianti di useEffect per Array di Dipendenze

| Dipendenze | Quando si esegue | Uso tipico |
|------------|-----------------|------------|
| `[]` (array vuoto) | Solo al **primo render** (mount) | Caricamento dati iniziali, subscriptions |
| `[gruppoId]` | Al mount + ogni volta che `gruppoId` cambia | Ricaricare dati al cambio di parametro |
| (omesso) | Ad **ogni** render | Raro, quasi mai necessario |

### Cleanup Function — Evitare Memory Leaks

Se un componente viene smontato prima che una chiamata asincrona termini, aggiornare lo stato di un componente smontato causa errori. La **cleanup function** di useEffect previene questo:

```jsx
useEffect(() => {
  let isMounted = true; // flag per sapere se il componente è ancora montato

  getSpese(gruppoId).then(data => {
    if (isMounted) setSpese(data); // aggiorna solo se ancora montato
  });

  return () => {
    isMounted = false; // cleanup: eseguito quando il componente si smonta
  };
}, [gruppoId]);
```

---

## 4.7 Struttura dei File in Split Mate

```
src/
├── App.jsx               ← Componente radice, gestisce il routing condizionale
├── main.jsx              ← Entry point, monta React nel <div id="root">
├── api.js                ← Tutte le chiamate HTTP al backend (AJAX centralizzati)
├── components/
│   ├── Login.jsx         ← Schermata di login con Google OAuth
│   ├── Dashboard.jsx     ← Vista principale con lista gruppi
│   ├── DettaglioGruppo.jsx ← Vista singolo gruppo con spese e saldi
│   ├── ModalNuovaSpesa.jsx ← Form modale per aggiungere/modificare spese
│   ├── ModalNuovoGruppo.jsx ← Form modale per creare gruppi
│   └── ListaDebiti.jsx   ← Riepilogo saldi e pulsante "Segna come saldato"
└── index.css             ← Stili globali
```

### App.jsx — Il Router Condizionale

Split Mate non usa un router basato su URL (`react-router-dom`) — usa un **routing condizionale basato sullo stato**. Il componente radice `App.jsx` decide cosa mostrare in base a quale gruppo è selezionato:

```jsx
// App.jsx — routing condizionale con stato
function App() {
  const [utente, setUtente] = useState(null);
  const [gruppoSelezionato, setGruppoSelezionato] = useState(null);
  const [view, setView] = useState('dashboard'); // 'dashboard' | 'dettaglio'

  // Se non loggato → mostra Login
  if (!utente) {
    return <Login onLogin={setUtente} />;
  }

  // Se è selezionato un gruppo → mostra il dettaglio
  if (view === 'dettaglio' && gruppoSelezionato) {
    return (
      <DettaglioGruppo
        gruppo={gruppoSelezionato}
        onTornaIndietro={() => setView('dashboard')}
      />
    );
  }

  // Altrimenti → mostra la dashboard con la lista gruppi
  return (
    <Dashboard
      utente={utente}
      onApriGruppo={(gruppo) => {
        setGruppoSelezionato(gruppo);
        setView('dettaglio');
      }}
    />
  );
}
```

---

## 4.8 api.js — Le Chiamate AJAX Centralizzate

Tutte le comunicazioni con il backend ASP.NET Core sono centralizzate in un unico file `api.js`. Questo segue il principio di separazione delle responsabilità: i componenti React non sanno nulla degli endpoint HTTP — sanno solo che possono chiamare funzioni e ricevere dati.

```javascript
// api.js — estratto semplificato
const API_URL = import.meta.env.VITE_API_URL; // da .env: http://localhost:5000/api

// Helper per costruire gli header con il JWT
function getAuthHeaders() {
  const token = localStorage.getItem('jwt_token');
  return {
    'Content-Type': 'application/json',
    'Authorization': token ? `Bearer ${token}` : '',
  };
}

// --- AUTENTICAZIONE ---
export async function login(googleToken) {
  const res = await fetch(`${API_URL}/Auth/login`, {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ googleToken }),
  });
  if (!res.ok) throw new Error('Login fallito');
  return res.json(); // { token: "eyJ...", utente: {...} }
}

// --- SPESE ---
export async function getSpese(gruppoId) {
  const res = await fetch(`${API_URL}/Spesa/gruppo/${gruppoId}`, {
    headers: getAuthHeaders(),
  });
  if (!res.ok) throw new Error('Errore nel caricamento spese');
  return res.json();
}

export async function creaSpesa(dati) {
  const res = await fetch(`${API_URL}/Spesa`, {
    method: 'POST',
    headers: getAuthHeaders(),
    body: JSON.stringify(dati),
  });
  if (!res.ok) throw new Error('Errore nella creazione spesa');
  return res.json();
}

export async function eliminaSpesa(id) {
  const res = await fetch(`${API_URL}/Spesa/${id}`, {
    method: 'DELETE',
    headers: getAuthHeaders(),
  });
  if (!res.ok) throw new Error('Errore nella cancellazione');
}

// --- RIEPILOGO ---
export async function getRiepilogoGruppo(gruppoId) {
  const res = await fetch(`${API_URL}/Riepilogo/gruppo/${gruppoId}`, {
    headers: getAuthHeaders(),
  });
  if (!res.ok) throw new Error('Errore nel riepilogo');
  return res.json();
}
```

### Perché Centralizzare in api.js?

- **Un solo posto da aggiornare** se cambia il base URL o il formato dei token
- **Testabilità**: puoi mockare `api.js` nei test senza toccare i componenti
- **Coerenza**: tutte le chiamate gestiscono errori allo stesso modo
- **Leggibilità**: i componenti chiamano `creaSpesa(dati)` — non sanno nulla di fetch, URL o JWT

---

## 4.9 Pattern Tipico: Caricamento Dati in un Componente

Il pattern che si ripete in ogni componente che mostra dati dal backend:

```jsx
function DettaglioGruppo({ gruppo, onTornaIndietro }) {
  // 1. Stato per i dati, loading e errori
  const [spese, setSpese] = useState([]);
  const [riepilogo, setRiepilogo] = useState([]);
  const [loading, setLoading] = useState(true);
  const [errore, setErrore] = useState(null);

  // 2. Caricamento dati con useEffect
  useEffect(() => {
    const caricaDati = async () => {
      try {
        setLoading(true);
        const [speseData, riepilogoData] = await Promise.all([
          getSpese(gruppo.id),
          getRiepilogoGruppo(gruppo.id),
        ]);
        setSpese(speseData);
        setRiepilogo(riepilogoData);
      } catch (err) {
        setErrore(err.message);
      } finally {
        setLoading(false);
      }
    };

    caricaDati();
  }, [gruppo.id]);

  // 3. Render condizionale per stati di loading/errore
  if (loading) return <div className="loading">Caricamento...</div>;
  if (errore) return <div className="errore">Errore: {errore}</div>;

  // 4. Render principale
  return (
    <div>
      <button onClick={onTornaIndietro}>← Indietro</button>
      <h2>{gruppo.nome}</h2>
      {spese.map(s => <CardSpesa key={s.id} spesa={s} />)}
      <ListaDebiti debiti={riepilogo} />
    </div>
  );
}
```

**`Promise.all`** esegue le due chiamate API **in parallelo** invece che in sequenza, dimezzando i tempi di attesa quando servono più dataset contemporaneamente.

---

## 4.10 Ottimizzazione delle Prestazioni

### Il Problema del Re-rendering

Ogni volta che lo stato cambia in un componente, React ri-renderizza quel componente **e tutti i suoi figli**. Con alberi di componenti profondi, questo può diventare costoso.

```jsx
// ❌ Problema: ogni aggiornamento di spese ri-renderizza TUTTA la lista
function ListaSpese({ spese, onElimina }) {
  return spese.map(s => (
    <CardSpesa key={s.id} spesa={s} onElimina={onElimina} />
  ));
}
```

### React.memo — Evitare Re-render Non Necessari

`React.memo` fa sì che un componente si ri-renderizzi **solo se le sue props cambiano**:

```jsx
// ✅ CardSpesa si ri-renderizza solo se spesa o onElimina cambiano
const CardSpesa = React.memo(function CardSpesa({ spesa, onElimina }) {
  return (
    <div className="card-spesa">
      <span>{spesa.descrizione}</span>
      <button onClick={() => onElimina(spesa.id)}>Elimina</button>
    </div>
  );
});
```

### Key — Aiutare React nel Diffing

La prop `key` è fondamentale per la performance delle liste. React usa `key` per identificare quali elementi sono stati aggiunti, rimossi o riordinati:

```jsx
// ❌ Sbagliato — usare l'indice come key
spese.map((s, index) => <CardSpesa key={index} spesa={s} />)
// Se riordino la lista, React non capisce cosa è cambiato davvero

// ✅ Corretto — usare l'ID univoco dell'elemento
spese.map(s => <CardSpesa key={s.id} spesa={s} />)
// React identifica esattamente quale elemento è cambiato
```

---

## 4.11 Vite — Il Build Tool di Split Mate

**Vite** è lo strumento che trasforma il codice JSX/JS moderno in file ottimizzati per il browser. Sostituisce webpack con un approccio più veloce basato su ES modules nativi.

| Funzione | Cosa fa |
|----------|---------|
| **Dev Server** | Avvia `npm run dev` → serve i file in locale su `http://localhost:5173` con HMR |
| **HMR** (Hot Module Replacement) | Aggiorna il componente modificato **senza ricaricare la pagina** |
| **Build** | `npm run build` → compila JSX → JS, minimizza, ottimizza → cartella `dist/` |
| **Deploy** | La cartella `dist/` viene caricata su Vercel — solo file statici (HTML + JS) |

### Variabili d'Ambiente

Vite gestisce le variabili d'ambiente tramite file `.env`:

```bash
# .env.local (sviluppo locale — non committato in git)
VITE_API_URL=http://localhost:5000/api

# .env.production (usato in build per Vercel)
VITE_API_URL=https://split-mate-api.azurewebsites.net/api
```

Nel codice si accede con `import.meta.env.VITE_*` — le variabili non prefissate con `VITE_` non vengono esposte al browser per sicurezza.

---

## 4.12 Google OAuth — Autenticazione nel Frontend

Split Mate usa **Google OAuth** come unico metodo di login. Il flusso lato frontend:

```
1. Utente clicca "Accedi con Google"
        |
2. Google OAuth: popup/redirect verso accounts.google.com
        |
3. Utente acconsente → Google restituisce un ID Token (JWT Google)
        |
4. FRONTEND invia l'ID Token al nostro backend:
   POST /api/Auth/login  { googleToken: "eyJ..." }
        |
5. BACKEND verifica il token con i server Google
   Se valido → crea/recupera l'utente nel DB
   Genera il nostro JWT interno
   Risponde: { token: "eyJ...", utente: { id, nome, email, avatar } }
        |
6. FRONTEND salva il JWT in localStorage
   Salva i dati utente in useState
   Mostra la Dashboard
```

```jsx
// Login.jsx — gestione del login Google
import { GoogleLogin } from '@react-oauth/google';

function Login({ onLogin }) {
  const handleGoogleSuccess = async (credentialResponse) => {
    try {
      const data = await login(credentialResponse.credential); // api.js
      localStorage.setItem('jwt_token', data.token);
      onLogin(data.utente); // aggiorna lo stato in App.jsx
    } catch (err) {
      console.error('Login fallito:', err);
    }
  };

  return (
    <div className="login-container">
      <h1>Split Mate</h1>
      <p>Gestisci le spese condivise con i tuoi amici</p>
      <GoogleLogin
        onSuccess={handleGoogleSuccess}
        onError={() => console.error('Google login error')}
      />
    </div>
  );
}
```

---

## Riepilogo Capitolo 4

| Concetto | Come si manifesta in Split Mate |
|----------|---------------------------------|
| Virtual DOM | React aggiorna solo i nodi cambiati (es. una nuova `<CardSpesa>`) |
| JSX | Tutti i file `.jsx` — HTML dentro JavaScript |
| Componenti | `CardSpesa`, `Dashboard`, `DettaglioGruppo`, `ModalNuovaSpesa`... |
| Props | Dati e callback passati da `App.jsx` ai componenti figli |
| useState | Stato locale: spese, gruppi, loading, errori, modal aperti |
| useEffect | Chiamate API al mount o al cambio di `gruppoId` |
| api.js | Centralizzazione di tutte le `fetch()` al backend ASP.NET Core |
| Routing condizionale | `App.jsx` mostra `Login`, `Dashboard` o `DettaglioGruppo` in base allo stato |
| Promise.all | Caricamento parallelo di spese e riepilogo |
| Vite + .env | Build tool con HMR, variabili d'ambiente per locale vs Vercel |
| Google OAuth | Login tramite ID Token Google → JWT interno del backend |
