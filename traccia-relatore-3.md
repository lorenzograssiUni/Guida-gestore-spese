# 🎤 Traccia Relatore 3 — Frontend & UX

> **Capitoli di riferimento:** Cap. 4 (React, Componenti, Stato, Hooks)
> **Durata stimata:** ~4 minuti

---

## Apertura — Passaggio di consegne

> *"Passiamo ora al frontend. L'interfaccia di Split Mate è costruita con React 18 — una libreria JavaScript per costruire UI attraverso componenti riutilizzabili. Vi spiego come è strutturata l'app, come gestisce i dati e cosa succede quando l'utente compie un'azione."*

---

## 1. React e il Virtual DOM

**React** è focalizzato sul layer di presentazione — è la "View" nel pattern MVC. Il suo punto di forza è il **Virtual DOM**.

Il DOM è la rappresentazione ad albero della pagina HTML. Manipolarlo direttamente è costoso perché ogni modifica può innescare un ricalcolo del layout e un ridisegno del browser. React risolve questo con il Virtual DOM: mantiene in memoria una **copia leggera del DOM**, e quando lo stato cambia:

1. Crea un nuovo Virtual DOM con le modifiche
2. Lo confronta con il precedente (**Diffing**)
3. Calcola il set minimo di operazioni necessarie (**Reconciliation**)
4. Aggiorna **solo** i nodi del DOM reale che devono cambiare

Quando aggiungiamo una spesa, React non ridisegna tutta la lista — aggiunge solo il nuovo `<li>` della spesa appena creata.

---

## 2. JSX — HTML dentro JavaScript

**JSX** è la sintassi che permette di scrivere HTML dentro il codice JavaScript. Non è HTML vero — Vite/Babel lo trasforma in chiamate `React.createElement()`. Le regole principali che abbiamo applicato:

- `className` invece di `class` (parola riservata in JS)
- Ogni componente restituisce **un solo elemento radice**
- Le espressioni JavaScript vanno tra `{}`
- Ogni elemento in una lista deve avere la prop `key` con un ID univoco (aiuta React nel Diffing)

---

## 3. Struttura dei componenti

L'app è organizzata in:

```
src/
├── App.jsx                 ← Root: gestisce routing condizionale e stato utente
├── api.js                  ← Tutte le chiamate HTTP al backend (centralizzate)
├── pages/
│   ├── HomePage.jsx
│   ├── DettaglioGruppo.jsx
│   └── RiepilogoGruppo.jsx
└── components/
    ├── Navbar.jsx           ← Modifica nome utente inline
    └── ModalNuovoGruppo.jsx
```

**`App.jsx`** non usa un router basato su URL (`react-router-dom`) — usa un **routing condizionale basato sullo stato**:

```jsx
if (!utente) return <Login />;
if (view === 'dettaglio') return <DettaglioGruppo />;
return <Dashboard />;
```

Se l'utente non è loggato → mostra Login. Se ha selezionato un gruppo → mostra DettaglioGruppo. Altrimenti → mostra la Dashboard con la lista gruppi.

---

## 4. Props — flusso dati tra componenti

I dati fluiscono in React in modo **unidirezionale**: dal componente padre verso i figli tramite **props** (sola lettura). Per comunicare verso l'alto, il figlio chiama una **callback** passata come prop.

Esempio concreto: `App.jsx` passa a `DettaglioGruppo` sia i dati del gruppo (prop dati) che la funzione `onEliminaSpesa` (prop callback). Quando l'utente clicca elimina in un componente figlio, questo chiama la callback — che si trova nel padre — per aggiornare lo stato globale.

---

## 5. useState — gestione dello stato locale

`useState` è il hook per lo stato locale. Ogni variabile di stato, quando viene aggiornata tramite il suo setter, causa il **re-render** del componente e di tutti i suoi figli.

In `DettaglioGruppo` abbiamo ad esempio:

```jsx
const [spese, setSpese] = useState([]);
const [loading, setLoading] = useState(true);
const [errore, setErrore] = useState(null);
```

**Regola critica**: non modificare mai lo stato direttamente (`spese.push(...)` è sbagliato). Bisogna sempre usare il setter e creare un nuovo array:

```jsx
setSpese([...spese, nuovaSpesa]); // spread: copia l'array e aggiunge il nuovo elemento
```

Per lo stato globale (chi è loggato, quale gruppo è aperto) usiamo `localStorage`, che persiste tra sessioni browser.

---

## 6. useEffect — chiamate API al mount

`useEffect` è il hook per gli **effetti collaterali**: chiamate API, timer, iscrizioni a eventi. Si esegue dopo il rendering.

Il pattern che si ripete in ogni pagina che carica dati:

```jsx
useEffect(() => {
  setLoading(true);
  getSpese(gruppoId)        // chiamata ad api.js
    .then(data => setSpese(data))
    .finally(() => setLoading(false));
}, [gruppoId]); // dipendenze: si riesegue quando gruppoId cambia
```

L'array di dipendenze `[gruppoId]` è fondamentale: con array vuoto `[]` si esegue solo al primo render (mount), senza array si esegue ad ogni render (quasi mai quello che si vuole).

---

## 7. api.js — chiamate AJAX centralizzate

Tutte le chiamate `fetch()` al backend sono centralizzate in **`api.js`**. I componenti React non sanno nulla di URL, header o JWT — chiamano funzioni come `creaSpesa(dati)` o `getSpese(gruppoId)` e ricevono i dati.

Vantaggi:
- **Un solo posto da modificare** se cambia l'URL del backend
- **Coerenza**: tutte le chiamate gestiscono errori allo stesso modo
- I componenti rimangono **puliti e leggibili**

Usando **`Promise.all`** carichiamo spese e riepilogo in **parallelo** invece che in sequenza, dimezzando i tempi di attesa:

```jsx
const [speseData, riepilogoData] = await Promise.all([
  getSpese(gruppo.id),
  getRiepilogoGruppo(gruppo.id),
]);
```

---

## 8. Funzionalità UX notevoli

- **Modifica nome utente inline dalla Navbar**: click sul nome → campo di testo → salva senza uscire dalla pagina
- **Codice invito copiabile**: ogni gruppo ha un codice univoco generato dal backend
- **Selezione parziale dei partecipanti**: la spesa può essere divisa tra tutti i membri o solo tra alcuni selezionati con checkbox
- **Render condizionale loading/errore**: ogni componente mostra uno stato di caricamento e gestisce gli errori API

---

## Possibili domande del professore

**"Cos'è il Virtual DOM e perché è utile?"**
> È una copia leggera del DOM reale mantenuta in memoria. Quando lo stato cambia, React confronta il nuovo Virtual DOM con il precedente (Diffing), calcola le modifiche minime necessarie (Reconciliation), e aggiorna solo quei nodi nel DOM reale. Questo evita ridisegni inutili e mantiene l'app veloce.

**"Differenza tra useState e useEffect?"**
> `useState` gestisce dati che cambiano nel tempo e causano re-render quando aggiornati. `useEffect` esegue operazioni con effetti collaterali (come chiamate API) dopo il rendering, e si riesegue quando cambiano le sue dipendenze.

**"Perché centralizzare le chiamate API in api.js?"**
> Per separare le responsabilità: i componenti gestiscono la UI, `api.js` gestisce la comunicazione con il server. Se cambia l'URL o il formato dei dati, modifichiamo solo `api.js` senza toccare i componenti.

**"Come funziona il routing senza react-router?"**
> Usiamo un routing condizionale basato sullo stato in `App.jsx`: a seconda del valore di alcune variabili di stato (`utente`, `view`, `gruppoSelezionato`), `App.jsx` renderizza componenti diversi. È un approccio semplice adatto alla struttura ridotta dell'app.
