# Capitolo 9 — Stack Tecnologico Frontend

## Introduzione

Il frontend di Gestore Spese è una **Single Page Application (SPA)** costruita con tre tecnologie principali: React per la logica dell'interfaccia, Vite come strumento di build, e Tailwind CSS per lo stile. A queste si aggiunge JavaScript asincrono — con il pattern `async/await` — come meccanismo fondamentale per comunicare con il backend ASP.NET senza bloccare l'interfaccia.

Le versioni usate nel progetto sono particolarmente moderne:

| Tecnologia | Versione | Note |
|---|---|---|
| React | 19.2.4 | Ultima major release |
| Vite | 8.0.0 | Build tool di nuova generazione |
| Tailwind CSS | 4.2.1 | Nuovo engine CSS-first |
| React Router DOM | 7.13.1 | Routing dichiarativo |
| Node.js / npm | LTS | Gestore dei pacchetti |

---

## 9.1 React — Componenti, JSX e Stato

### Cos'è React

React è una **libreria JavaScript** (non un framework completo) sviluppata da Meta per costruire interfacce utente. Il suo modello mentale centrale è semplice: **l'interfaccia è una funzione dello stato**.

```
UI = f(state)
```

Quando lo stato cambia, React ricalcola automaticamente quali parti dell'interfaccia devono aggiornarsi e aggiorna solo quelle, senza ricaricare la pagina. Questo è il cuore della SPA.

### Componenti

In React, ogni elemento dell'interfaccia è un **componente**: una funzione JavaScript che riceve dati in input (`props`) e restituisce JSX (la descrizione di cosa mostrare).

```jsx
// Componente semplice
function NomeGruppo({ nome, membri }) {
    return (
        <div className="card">
            <h2>{nome}</h2>
            <p>{membri.length} membri</p>
        </div>
    );
}
```

I componenti sono **componibili**: si inserisce un componente dentro un altro come fossero tag HTML custom. Tutta l'app è un albero di componenti con `App.jsx` come radice.

### JSX — JavaScript + XML

JSX è una sintassi che permette di scrivere markup simile all'HTML direttamente nel codice JavaScript. Non è HTML puro: viene trasformato da Vite (tramite Babel/SWC) in chiamate `React.createElement(...)` prima di essere eseguito nel browser.

```jsx
// JSX (quello che scrivi)
const elemento = <h1 className="titolo">{nome}</h1>;

// JavaScript equivalente (quello che gira nel browser)
const elemento = React.createElement('h1', { className: 'titolo' }, nome);
```

Differenze chiave tra JSX e HTML:
- `class` diventa `className` (perché `class` è una parola riservata in JS)
- Gli eventi usano camelCase: `onClick`, `onChange`, `onSubmit`
- Le espressioni JS vanno dentro `{ }`: `{utente.nome}`, `{lista.map(...)}`
- I tag devono essere sempre chiusi: `<input />` non `<input>`

### State e Hook — useState

Lo **stato** è la memoria interna di un componente. In React si gestisce con l'hook `useState`:

```jsx
const [utente, setUtente] = useState(null);
const [gruppi, setGruppi] = useState([]);
const [loading, setLoading] = useState(false);
```

`useState` restituisce una coppia: il valore corrente e una funzione per aggiornarlo. Quando si chiama `setUtente(nuovoValore)`, React ri-renderizza automaticamente il componente con il nuovo stato.

### useEffect — Effetti Collaterali

`useEffect` gestisce tutto ciò che avviene "fuori" dal rendering: chiamate API, timer, sottoscrizioni a eventi. Nel progetto lo trovi ovunque per caricare dati dal backend:

```jsx
useEffect(() => {
    const caricaGruppi = async () => {
        const res = await getGruppiUtente(utente.id);
        const data = await res.json();
        setGruppi(data);
    };
    if (utente) caricaGruppi();
}, [utente]); // si riesegue quando 'utente' cambia
```

L'array di dipendenze `[utente]` controlla quando l'effetto si riesegue:
- `[]` vuoto → si esegue solo al primo rendering (come `componentDidMount`)
- `[utente]` → si riesegue ogni volta che `utente` cambia
- omesso → si riesegue ad ogni rendering (da evitare di solito)

### React 19 — Novità Principali

Il progetto usa React 19, l'ultima major release. Le novità principali rispetto a React 18 sono:
- **React Compiler** (sperimentale): ottimizzazione automatica del re-rendering senza `useMemo`/`useCallback` manuali
- **Server Components** (architettura): componenti che girano lato server (non usati nel progetto, ma parte del modello)
- **`use()` hook**: per leggere Promise e Context direttamente nel rendering
- **Actions e `useActionState`**: per gestire form e mutazioni in modo dichiarativo

Nel nostro progetto si usano le funzionalità classiche (`useState`, `useEffect`, componenti funzionali), compatibili con tutte le versioni di React dalla 16 in poi.

---

## 9.2 Vite — Build Tool Moderno

### Cos'è Vite

Vite (pronuncia: `/viːt/`, dal francese "veloce") è un **build tool** creato da Evan You (creatore di Vue.js) che risolve uno dei maggiori problemi dello sviluppo frontend moderno: la lentezza del server di sviluppo con i bundler tradizionali.

### Il Problema con Webpack (e perché Vite è diverso)

Webpack (il bundler storico usato da Create React App) funziona così:

```
Avvio del dev server:
  legge TUTTI i file JS → analizza le dipendenze → compila TUTTO → avvia il server
  → con un progetto grande: 20-60 secondi di attesa

Modifica di un file:
  ricompila TUTTO il bundle → ricarica il browser
  → 1-5 secondi di attesa per ogni salvataggio
```

Vite invece sfrutta una caratteristica moderna dei browser: il supporto nativo agli **ES Modules** (`import`/`export`):

```
Avvio del dev server:
  serve i file direttamente SENZA bundling
  → avvio istantaneo (<300ms indipendentemente dalla dimensione del progetto)

Modifica di un file:
  invia al browser SOLO il modulo modificato (HMR — Hot Module Replacement)
  → aggiornamento in <50ms, senza perdere lo stato dell'app
```

| Aspetto | Webpack (CRA) | Vite |
|---|---|---|
| Avvio dev server | 20-60s | < 300ms |
| Aggiornamento a caldo | 1-5s | < 50ms |
| Bundler in produzione | Webpack | Rollup |
| Config di base | `webpack.config.js` complessa | `vite.config.js` minimale |
| Trasformazione JSX | Babel | SWC (Rust, 20x più veloce) |

### Il Nostro vite.config.js

```js
import { defineConfig } from 'vite'
import react from '@vitejs/plugin-react'
import tailwindcss from '@tailwindcss/vite'

export default defineConfig({
    plugins: [
        react(),        // abilita JSX transform e React Fast Refresh
        tailwindcss(),  // integra Tailwind CSS v4 direttamente come plugin Vite
    ],
    resolve: {
        dedupe: ['react', 'react-dom', 'react-router-dom'],
    },
})
```

**`react()`** attiva due cose:
1. La trasformazione JSX (`.jsx` → JS puro) tramite SWC
2. **React Fast Refresh**: quando modifichi un componente, Vite aggiorna solo quel componente nel browser mantenendo lo stato degli altri. È la magia che fa sì che puoi modificare un form senza perdere i dati già inseriti.

**`tailwindcss()`** integra Tailwind CSS v4 direttamente nella pipeline di Vite, senza bisogno di un `postcss.config.js` separato. È un'integrazione più profonda e veloce rispetto alla versione precedente.

**`dedupe`** risolve un problema specifico: se React Router DOM include la propria copia di React come dipendenza, si potrebbero avere due istanze di React attive contemporaneamente — causando l'errore "Invalid hook call". `dedupe` forza Vite a usare sempre una sola istanza.

### Vite e il File index.html

```html
<!doctype html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>frontend-gestione-spese</title>
  </head>
  <body>
    <div id="root"></div>
    <script type="module" src="/src/main.jsx"></script>
  </body>
</html>
```

In Vite, `index.html` è il **punto di ingresso** dell'applicazione (non `main.jsx` come si potrebbe pensare). Vite lo legge, trova il tag `<script type="module" src="/src/main.jsx">`, e da lì parte l'analisi dell'albero di dipendenze. Il `type="module"` non è un dettaglio: dice al browser di trattare il file come un ES Module, abilitando `import`/`export` nativi.

### Comandi Vite

```bash
npm run dev      # Avvia il dev server su http://localhost:5173 (default)
npm run build    # Compila per produzione nella cartella dist/
npm run preview  # Serve la build di produzione localmente per testarla
npm run lint     # ESLint su tutti i file .jsx/.js
```

In produzione (`npm run build`), Vite usa **Rollup** come bundler: analizza tutte le dipendenze, rimuove il codice non usato (tree-shaking), divide il bundle in chunk ottimizzati, e minifica tutto. L'output in `dist/` è quello che viene deployato su Vercel.

---

## 9.3 Tailwind CSS — Utility-First

### Il Paradigma Utility-First

Tailwind CSS è un framework CSS basato su **classi di utilità**: invece di scrivere CSS custom con nomi semantici, si applica direttamente sull'HTML classi predefinite che corrispondono a singole proprietà CSS.

```jsx
// Approccio tradizionale (CSS semantico)
<button className="btn-primary">Accedi</button>
// Nel CSS: .btn-primary { background: blue; color: white; padding: 8px 16px; border-radius: 4px; }

// Approccio Tailwind (utility-first)
<button className="bg-blue-600 text-white px-4 py-2 rounded hover:bg-blue-700 transition">
    Accedi
</button>
```

I vantaggi:
- **Nessun CSS da scrivere** per la stragrande maggioranza dei casi
- **Nessun problema di naming** (`.card-header-inner-wrapper-left`)
- **CSS non cresce nel tempo**: Tailwind genera solo le classi usate nel codice
- **Coerenza visiva**: si usano valori da una scala predefinita (es. spacing in multipli di 4px)

### Tailwind CSS v4 — Il Nuovo Engine

Il progetto usa **Tailwind v4**, una riscrittura completa del framework rilasciata nel 2025. Le differenze principali rispetto a v3:

| Aspetto | Tailwind v3 | Tailwind v4 |
|---|---|---|
| Configurazione | `tailwind.config.js` (obbligatorio) | Nessun file di config (tutto in CSS) |
| Integrazione con Vite | Tramite PostCSS plugin | Plugin Vite nativo (`@tailwindcss/vite`) |
| Velocità build | Veloce | 5x più veloce (engine Rust) |
| Personalizzazione | `theme.extend` in JS | Variabili CSS standard (`@theme`) |
| Direttiva principale | `@tailwind base/components/utilities` | `@import "tailwindcss"` |

In v4, la configurazione si fa direttamente nel CSS:

```css
/* index.css — tutto qui, nessun tailwind.config.js */
@import "tailwindcss";

@theme {
    --color-brand: #3b82f6;
    --spacing-18: 4.5rem;
}
```

Nel progetto il file `index.css` contiene solo `@import "tailwindcss";` — la configurazione minima necessaria per abilitare tutte le utility class.

---

## 9.4 JavaScript Asincrono — Callback, Promise, Async/Await

Questa sezione è particolarmente importante per l'esame: il professor Bonura tratta l'asincronia JS nelle slide del corso. Qui la teoria incontra il codice reale del progetto.

### Il Problema: JavaScript è Single-Thread

JavaScript esegue il codice su **un solo thread**. Se una chiamata di rete (che può durare 500ms) bloccasse il thread, l'intera interfaccia sarebbe congelata: l'utente non potrebbe cliccare nulla, scorrere, o interagire in nessun modo.

La soluzione è il modello **asincrono non-bloccante**: le operazioni lente vengono avviate, il controllo torna immediatamente al thread principale, e quando il risultato è pronto una **callback** viene messa in coda ed eseguita.

### Fase 1 — Callback (il passato)

Il primo approccio storico era passare una funzione da chiamare quando l'operazione finiva:

```javascript
// Esempio teorico con XMLHttpRequest
funzione caricaGruppi(utenteId, callback) {
    const xhr = new XMLHttpRequest();
    xhr.onload = function() {
        callback(null, JSON.parse(xhr.responseText));
    };
    xhr.onerror = function() {
        callback(new Error('Errore di rete'));
    };
    xhr.open('GET', `/api/gruppo/utente/${utenteId}`);
    xhr.send();
}

// Uso
caricaGruppi(1, function(err, gruppi) {
    if (err) { gestisciErrore(err); return; }
    // Ora carica le spese del primo gruppo
    caricaSpese(gruppi[0].id, function(err2, spese) {
        // "Callback Hell": codice sempre più annidato e difficile da leggere
    });
});
```

Il problema del **Callback Hell**: quando si concatenano più operazioni asincrone, il codice si annida sempre di più diventando illeggibile e difficile da gestire (specialmente per gli errori).

### Fase 2 — Promise (ES2015)

Le **Promise** introducono un oggetto che rappresenta il risultato futuro di un'operazione asincrona. Possono trovarsi in tre stati:
- `pending` → in attesa del risultato
- `fulfilled` → completata con successo (valore disponibile)
- `rejected` → fallita (errore disponibile)

```javascript
// fetch() restituisce una Promise
fetch('/api/gruppo/utente/1')
    .then(response => response.json())        // primo .then: parse JSON
    .then(gruppi => {
        console.log(gruppi);
        return fetch(`/api/spesa/${gruppi[0].id}`); // catena di Promise
    })
    .then(res => res.json())
    .then(spese => console.log(spese))
    .catch(errore => console.error('Errore:', errore)); // un solo catch per tutta la catena
```

Meglio delle callback, ma la catena di `.then()` può diventare verbose quando le operazioni sono molte.

### Fase 3 — Async/Await (ES2017) — Quello che usiamo

`async/await` è **zucchero sintattico** sopra le Promise: non è un meccanismo diverso, ma una sintassi che rende il codice asincrono leggibile come se fosse sincrono.

```javascript
// La stessa logica di sopra con async/await
async function caricaDatiGruppo(utenteId) {
    try {
        const res = await fetch(`/api/gruppo/utente/${utenteId}`);
        const gruppi = await res.json();

        const res2 = await fetch(`/api/spesa/${gruppi[0].id}`);
        const spese = await res2.json();

        return { gruppi, spese };
    } catch (errore) {
        console.error('Errore:', errore);
    }
}
```

- `async` prima di `function` indica che la funzione è asincrona e restituisce sempre una Promise
- `await` sospende l'esecuzione della funzione (NON del thread principale!) fino a quando la Promise è risolta
- `try/catch` gestisce gli errori in modo sincrono, molto più leggibile di `.catch()`

### Async/Await nel Progetto Reale

Ecco come appare nel codice reale del progetto, prendendo `api.js` come esempio:

```javascript
// src/api/api.js — pattern usato per tutte le chiamate

const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:5097/api';

export const login = async (email, password) => {
    const res = await fetch(`${API_BASE_URL}/Auth/login`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password })
    });
    return res; // ritorna la Response grezza: il chiamante decide come gestirla
};

export const getGruppiUtente = async (utenteId) => {
    const res = await fetch(`${API_BASE_URL}/Gruppo/utente/${utenteId}`);
    return res;
};
```

Nota una scelta progettuale importante: le funzioni di `api.js` **non fanno il `.json()` autonomamente** — ritornano la `Response` grezza. Questo permette al componente chiamante di:
1. Controllare `res.ok` o `res.status` prima di fare il parse
2. Gestire errori HTTP (401, 404, 500) in modo diverso
3. Decidere se fare `.json()` o un altro tipo di parse

```jsx
// Esempio dal componente (pattern tipico)
const handleLogin = async () => {
    setLoading(true);
    try {
        const res = await login(email, password);
        if (res.ok) {
            const utente = await res.json();
            setUtente(utente);  // aggiorna lo stato → React ri-renderizza
        } else {
            setErrore('Credenziali non valide');
        }
    } catch (e) {
        setErrore('Errore di connessione');
    } finally {
        setLoading(false);  // sempre eseguito, anche in caso di errore
    }
};
```

### Event Loop — Perché Async non Blocca il Browser

Il meccanismo sottostante che rende tutto questo possibile è l'**Event Loop**:

```
┌─────────────────────────────────────────────┐
│  Call Stack (codice sincrono in esecuzione)  │
│  es: handleLogin() → login() → fetch()       │
└────────────────────┬────────────────────────┘
                     │ fetch() avviato
                     │ (operazione I/O, esce dallo stack)
                     ▼
┌─────────────────────────────────────────────┐
│  Web APIs (browser gestisce la rete)         │
│  TCP connect → HTTP request → attende resp.  │
└────────────────────┬────────────────────────┘
                     │ risposta arrivata
                     ▼
┌─────────────────────────────────────────────┐
│  Callback Queue (in attesa di essere eseguita)│
│  la continuation dopo await fetch()          │
└────────────────────┬────────────────────────┘
                     │ Call Stack è vuoto
                     ▼
             Event Loop ri-esegue la
             continuation nel Call Stack
```

Quando il codice incontra `await fetch(...)`, la funzione viene sospesa e restituisce il controllo al Call Stack. Il browser gestisce la rete in background (tramite C++, non JS). Quando la risposta arriva, la continuazione della funzione viene messa in coda e l'Event Loop la esegue non appena il Call Stack è libero. **Il thread principale non è mai bloccato.**

---

## 9.5 Il File main.jsx — Entry Point dell'App

```jsx
import React from 'react'
import ReactDOM from 'react-dom/client'
import { BrowserRouter } from 'react-router-dom'
import App from './App'
import './index.css'

ReactDOM.createRoot(document.getElementById('root')).render(
    <React.StrictMode>
        <BrowserRouter>
            <App />
        </BrowserRouter>
    </React.StrictMode>,
)
```

Questo file è il **collante** tra il mondo HTML e il mondo React:

- **`ReactDOM.createRoot`**: crea una "radice React" sul `<div id="root">` dell'HTML. Da questo momento React controlla tutto ciò che sta dentro quel div.
- **`<BrowserRouter>`**: abilita il routing basato sull'URL del browser (usa l'History API di HTML5). Tutti i componenti dentro possono usare `useNavigate`, `useParams`, ecc.
- **`<React.StrictMode>`**: in sviluppo, esegue ogni effetto due volte e attiva warning aggiuntivi per aiutare a trovare bug. Non ha effetto in produzione.
- **`import './index.css'`**: importa il CSS di Tailwind. Vite processa questo import tramite il plugin Tailwind, generando il CSS finale con solo le classi usate nel progetto.

---

## Punti Chiave per la Presentazione

1. **React è una libreria, non un framework**: gestisce solo la UI. Per routing, chiamate API, e stato globale si aggiungono librerie separate (React Router, `fetch`, `useState`).

2. **Vite vs Webpack**: Vite è 20-100x più veloce in sviluppo perché non builda nulla all'avvio — serve i moduli ES nativi direttamente al browser. Il bundling completo avviene solo in produzione con Rollup.

3. **async/await è Promise in disguise**: `await` non blocca il thread, sospende solo la funzione corrente. L'Event Loop continua a processare eventi (click, aggiornamenti UI) durante l'attesa.

4. **Tailwind v4 senza config**: nel progetto non esiste `tailwind.config.js`. La v4 è CSS-first: tutta la configurazione è nel CSS stesso, integrata nativamente con Vite.

5. **`import.meta.env.VITE_API_URL`**: le variabili d'ambiente di Vite devono avere il prefisso `VITE_` per essere esposte al codice frontend. In sviluppo si usa il file `.env` (localhost), in produzione `.env.production` (URL Azure).
