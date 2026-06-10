# Capitolo 10 — Architettura Frontend

## Introduzione

L'architettura del frontend di Gestore Spese è volutamente **semplice e piatta**: nessun sistema di state management globale (Redux, Zustand), nessun router dichiarativo con URL che cambiano, nessun layer di cache per le API. Tutto lo stato applicativo vive in `App.jsx` e viene passato ai figli tramite props. Questa scelta è appropriata per la dimensione del progetto e dimostra una comprensione consapevole di quando la complessità aggiuntiva non è giustificata.

---

## 10.1 App.jsx — Componente Radice e Stato Globale

`App.jsx` è il **cervello dell'applicazione**: gestisce l'autenticazione, lo stato di navigazione, e decide quale vista mostrare. È l'unico componente che sa "dove si trova" l'utente nell'app.

### Lo Stato Globale

```jsx
// Tutto lo stato dell'app è qui
const [utente, setUtente] = useState(() => { ... });      // utente autenticato (o null)
const [gruppoAttivoId, setGruppoAttivoId] = useState(null); // quale gruppo sta vedendo
const [vistaRiepilogo, setVistaRiepilogo] = useState(false); // è nella vista riepilogo?
const [refreshKey, setRefreshKey] = useState(0);            // forza il re-fetch della lista gruppi
const [isModalOpen, setIsModalOpen] = useState(false);      // modal "nuovo gruppo" aperto?

// Stato del form di login (potrebbe stare in un componente separato)
const [emailInput, setEmailInput] = useState('');
const [passwordInput, setPasswordInput] = useState('');
const [erroreLogin, setErroreLogin] = useState('');
const [modalita, setModalita] = useState('login');           // 'login' | 'registrazione'
const [nomeInput, setNomeInput] = useState('');
const [confermaPasswordInput, setConfermaPasswordInput] = useState('');
const [messaggioSuccesso, setMessaggioSuccesso] = useState('');
```

Ogni variabile di stato ha un ruolo preciso:

| Stato | Tipo | Scopo |
|---|---|---|
| `utente` | `Object \| null` | Se `null` → mostra login. Se valorizzato → mostra l'app. |
| `gruppoAttivoId` | `Number \| null` | `null` → HomePage. Valorizzato → DettaglioGruppo. |
| `vistaRiepilogo` | `boolean` | `true` + `gruppoAttivoId` → RiepilogoGruppo. |
| `refreshKey` | `Number` | Incrementato dopo la creazione di un gruppo: forza HomePage a ricaricare la lista. |
| `isModalOpen` | `boolean` | Mostra/nasconde il modal di creazione gruppo. |

### Inizializzazione con localStorage — Lazy Initializer

```jsx
const [utente, setUtente] = useState(() => {
    const savedUtente = localStorage.getItem('utente_spese');
    return savedUtente ? JSON.parse(savedUtente) : null;
});
```

Questo è un pattern React chiamato **lazy initializer**: invece di passare il valore iniziale direttamente a `useState`, si passa una **funzione** che viene eseguita una volta sola al primo rendering.

Perché la funzione anziché il valore diretto?

```jsx
// SBAGLIATO (ma funzionante) — legge localStorage ad ogni rendering
const [utente, setUtente] = useState(
    JSON.parse(localStorage.getItem('utente_spese')) // eseguito sempre
);

// CORRETTO — legge localStorage solo al primo rendering
const [utente, setUtente] = useState(() => {
    const saved = localStorage.getItem('utente_spese'); // eseguito UNA volta
    return saved ? JSON.parse(saved) : null;
});
```

`localStorage.getItem` e `JSON.parse` sono operazioni sincrone che bloccano brevemente il thread. Con il lazy initializer vengono chiamate una sola volta invece di ad ogni re-render del componente.

L'effetto pratico per l'utente: se l'utente ha già fatto login in precedenza e ricarica la pagina, l'app legge i dati dal localStorage e mostra direttamente l'interfaccia principale **senza passare per la schermata di login**.

### Persistenza con localStorage

Il dato utente viene sincronizzato con localStorage in tre punti:

```jsx
// 1. Al login — salva
localStorage.setItem('utente_spese', JSON.stringify(user));
setUtente(user);

// 2. Al logout — cancella
localStorage.removeItem('utente_spese');
setUtente(null);

// 3. All'aggiornamento del profilo — aggiorna
const handleAggiornaUtente = (nuoviDati) => {
    const utenteAggiornato = { ...utente, ...nuoviDati }; // spread: merge dei dati
    setUtente(utenteAggiornato);
    localStorage.setItem('utente_spese', JSON.stringify(utenteAggiornato));
};
```

Lo **spread operator** `{ ...utente, ...nuoviDati }` crea un nuovo oggetto copiando tutte le proprietà di `utente` e sovrascrivendo quelle presenti anche in `nuoviDati`. È il modo idiomatico di aggiornare un oggetto in React (mai modificare lo stato direttamente).

---

## 10.2 Routing Condizionale — Senza React Router

Uno degli aspetti architetturali più interessanti del progetto è il sistema di navigazione. **Non viene usato React Router** per gestire la navigazione tra le pagine principali: invece, si usa **rendering condizionale** basato sullo stato.

### Come Funziona

```jsx
// Nel return di App.jsx, dopo il check !utente
<main>
    {vistaRiepilogo && gruppoAttivoId ? (
        <RiepilogoGruppo
            gruppoId={gruppoAttivoId}
            onBack={() => setVistaRiepilogo(false)}
        />
    ) : gruppoAttivoId ? (
        <DettaglioGruppo
            gruppoId={gruppoAttivoId}
            onBack={() => setGruppoAttivoId(null)}
            onApriRiepilogo={() => setVistaRiepilogo(true)}
        />
    ) : (
        <HomePage
            utente={utente}
            refreshKey={refreshKey}
            onOpenModal={() => setIsModalOpen(true)}
            onApriGruppo={(id) => setGruppoAttivoId(id)}
        />
    )}
</main>
```

La logica è un albero decisionale a cascata:

```
è loggato?
├─ NO  → mostra form login/registrazione
└─ SÌ  → è in vistaRiepilogo E ha un gruppo attivo?
           ├─ SÌ  → <RiepilogoGruppo gruppoId={gruppoAttivoId} />
           └─ NO  → ha un gruppo attivo?
                      ├─ SÌ  → <DettaglioGruppo gruppoId={gruppoAttivoId} />
                      └─ NO  → <HomePage utente={utente} />
```

### Confronto: Routing Condizionale vs React Router

| Aspetto | Routing Condizionale (nostro) | React Router |
|---|---|---|
| URL del browser | Non cambia (sempre `/`) | Cambia (`/gruppo/3`, `/riepilogo/3`) |
| Navigabilità diretta | Impossibile (no deep link) | Sì (si può condividere l'URL) |
| Pulsante back del browser | Non funziona | Funziona |
| Complessità | Minima | Richiede configurazione route |
| Adatto per | Prototipi, app semplici | App con navigazione complessa |

Il progetto usa React Router DOM (installato nel `package.json` e il `BrowserRouter` in `main.jsx`) ma lo usa solo per l'infrastruttura base. La navigazione effettiva tra le "pagine" avviene tramite cambio di stato, non tramite `<Link>` o `useNavigate`. Questa è una scelta di semplicità valida per il contesto.

### Il Pattern "Callback as Props" per la Navigazione

Poiché lo stato di navigazione sta in `App.jsx`, i componenti figli non possono cambiarlo direttamente. La soluzione è passare **funzioni di callback come props**:

```jsx
// App.jsx passa la funzione
<DettaglioGruppo
    gruppoId={gruppoAttivoId}
    onBack={() => setGruppoAttivoId(null)}       // "vai a HomePage"
    onApriRiepilogo={() => setVistaRiepilogo(true)} // "vai a Riepilogo"
/>

// DettaglioGruppo la usa
function DettaglioGruppo({ gruppoId, onBack, onApriRiepilogo }) {
    return (
        <div>
            <button onClick={onBack}>← Indietro</button>
            <button onClick={onApriRiepilogo}>Vedi Riepilogo</button>
            {/* ... */}
        </div>
    );
}
```

Questo è il pattern **"lifting state up"** di React: lo stato (e le funzioni per modificarlo) vivono nel componente comune più in alto nella gerarchia, e vengono distribuiti verso il basso tramite props.

### Il refreshKey Pattern

```jsx
const [refreshKey, setRefreshKey] = useState(0);

// In App.jsx: passato a HomePage
<HomePage refreshKey={refreshKey} ... />

// Dopo la creazione di un gruppo:
<ModalNuovoGruppo
    onGruppoCreato={() => setRefreshKey(prev => prev + 1)}
/>

// In HomePage: dentro useEffect
useEffect(() => {
    caricaGruppi();
}, [refreshKey]); // si riesegue quando refreshKey cambia
```

Quando viene creato un nuovo gruppo, il modal chiama `onGruppoCreato()`, che incrementa `refreshKey` in `App.jsx`. Questo cambio di stato fa sì che `HomePage` riceva una nuova prop, il suo `useEffect` si riesegue, e la lista gruppi viene ricaricata dal server. È un pattern semplice ma efficace per forzare il re-fetch dei dati dopo una mutazione.

---

## 10.3 api.js — Modulo Centralizzato per le Chiamate HTTP

```javascript
// src/api/api.js
const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:5097/api';

export const login = async (email, password) => {
    const res = await fetch(`${API_BASE_URL}/Auth/login`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email, password })
    });
    return res;
};
// ... tutte le altre funzioni API
```

### Perché un Modulo Centralizzato

Senza `api.js`, ogni componente scriverebbe le chiamate `fetch` inline. Questo causa diversi problemi:

- **Duplicazione**: `API_BASE_URL` ripetuto ovunque
- **Manutenzione**: se cambia un endpoint, bisogna aggiornare N file
- **Leggibilità**: il componente mescola logica UI e logica di rete

Con `api.js`, ogni funzione ha un nome descrittivo (`creaSpesa`, `eliminaGruppo`, `saldaDebito`) e il componente si limita a chiamarla:

```jsx
// Senza api.js (da evitare)
const res = await fetch(`${import.meta.env.VITE_API_URL}/Spesa/${id}`, {
    method: 'DELETE'
});

// Con api.js (nostro approccio)
const res = await eliminaSpesa(id);
```

### Il Catalogo Completo delle Funzioni API

| Funzione | Metodo HTTP | Endpoint | Scopo |
|---|---|---|---|
| `login(email, pwd)` | POST | `/Auth/login` | Autenticazione |
| `aggiornaUtente(id, dati)` | PUT | `/Utente/:id` | Aggiorna nome utente |
| `getGruppiUtente(id)` | GET | `/Gruppo/utente/:id` | Lista gruppi dell'utente |
| `creaGruppo(dati)` | POST | `/Gruppo` | Crea nuovo gruppo |
| `getGruppo(id)` | GET | `/gruppo/:id` | Dettaglio gruppo |
| `eliminaGruppo(id)` | DELETE | `/gruppo/:id` | Elimina gruppo |
| `aggiungiBotAlGruppo(id, dati)` | POST | `/Gruppo/:id/aggiungi-bot` | Aggiunge membro bot |
| `rimuoviMembroDalGruppo(gId, mId)` | DELETE | `/Gruppo/:gId/rimuovi-membro/:mId` | Rimuove membro |
| `getBilanciGruppo(id)` | GET | `/gruppo/:id/bilanci` | Saldi del gruppo |
| `creaSpesa(dati)` | POST | `/Spesa` | Crea nuova spesa |
| `eliminaSpesa(id)` | DELETE | `/spesa/:id` | Elimina spesa |
| `modificaSpesa(id, dati)` | PUT | `/Spesa/:id` | Modifica spesa |
| `saldaDebito(id)` | PUT | `/Riepilogo/:id/Saldato` | Segna debito come saldato |

### Scelta Progettuale: Restituire la Response Grezza

Tutte le funzioni ritornano la `Response` di `fetch` senza fare `.json()` automaticamente. Questa è una scelta deliberata:

```javascript
// Le funzioni di api.js restituiscono la Response, non i dati
export const creaGruppo = async (dati) => {
    const res = await fetch(`${API_BASE_URL}/Gruppo`, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(dati)
    });
    return res; // <-- Response grezza
};

// Il componente decide cosa fare
const res = await creaGruppo({ nome: 'Vacanza', creatoreId: utente.id });
if (res.ok) {
    const nuovoGruppo = await res.json();
    // ...
} else if (res.status === 409) {
    setErrore('Gruppo già esistente');
} else {
    setErrore('Errore generico');
}
```

Se `api.js` facesse il parse automatico, il componente non potrebbe distinguere un `200 OK` da un `404 Not Found` — entrambi restituirebbero un oggetto JSON (magari `null` per il 404), rendendo la gestione degli errori impossibile.

**Nota**: `App.jsx` duplica alcune chiamate direttamente inline (login e registrazione) invece di usare le funzioni di `api.js`. Questo è un'incoerenza nel codice: idealmente anche il form di login userebbe `import { login } from './api/api.js'`.

---

## 10.4 Variabili d'Ambiente — VITE_API_URL

### Come Funzionano le Env Var in Vite

Vite espone le variabili d'ambiente al codice frontend tramite `import.meta.env`. Per ragioni di sicurezza, **solo le variabili con prefisso `VITE_`** vengono esposte: le altre sono accessibili solo nel codice Node.js di Vite (es. nei plugin), non nel bundle finale.

```
Variabile senza prefisso: SECRET_KEY=abc123
→ import.meta.env.SECRET_KEY === undefined  (NON esposta al browser)

Variabile con prefisso: VITE_API_URL=http://...
→ import.meta.env.VITE_API_URL === 'http://...'  (esposta al browser)
```

### I File .env nel Progetto

Vite supporta più file `.env` con precedenza gerarchica:

| File | Quando viene usato | Contenuto nel progetto |
|---|---|---|
| `.env` | Sempre (base) | `VITE_API_URL=http://localhost:5207/api` |
| `.env.production` | Solo con `npm run build` | `VITE_API_URL=https://gestione-spese-azure.net/api` |
| `.env.local` | Sempre (override locale, non in git) | Non presente nel repo |

Durante lo sviluppo (`npm run dev`), Vite usa `.env` → le chiamate API vanno a `localhost:5207` dove gira il backend ASP.NET locale.

Durante la build di produzione (`npm run build`), Vite usa `.env.production` → le chiamate API vanno all'URL Azure. Il valore viene **inlining** nel bundle JavaScript in fase di compilazione: non c'è nessun file di config a runtime.

### Uso nel Codice

```javascript
// In api.js
const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:5097/api';

// In App.jsx (duplicazione — vedi nota precedente)
const API_BASE_URL = import.meta.env.VITE_API_URL || 'http://localhost:5207/api';
```

Il fallback `|| 'http://localhost:5097/api'` garantisce che se la variabile d'ambiente non è definita (es. in un ambiente CI/CD che non ha il file `.env`), l'app usa comunque un URL di default sensato.

> **Attenzione al numero di porta**: `api.js` usa `:5097` come fallback, `App.jsx` usa `:5207`. Questo è un bug minore: se il file `.env` non esiste, le due parti del codice potrebbero puntare a porte diverse. In presenza del file `.env` corretto, entrambi leggono lo stesso valore.

### Sicurezza delle Variabili d'Ambiente

Importante da ricordare per l'esame: le variabili `VITE_*` vengono **incluse nel bundle JavaScript** che il browser scarica. Chiunque apra i DevTools può leggerle. Non mettere mai segreti (API keys private, password, token JWT) nelle variabili `VITE_*`. In questo progetto `VITE_API_URL` è solo un URL pubblico, quindi nessun problema.

---

## Schema Architettura Frontend Completo

```
                    index.html
                       │
                    main.jsx
                 ┌───┴───┐
           BrowserRouter    React.StrictMode
                       │
                    App.jsx  ←─────────────── localStorage
              (stato globale)              ('utente_spese')
              │
    ┌───────────├────────────────────┐
    │             │                        │
  Navbar     [routing                  ModalNuovoGruppo
             condizionale]
             │
   ┌──────────┤──────────────────┐
   │           │                     │
HomePage  DettaglioGruppo        RiepilogoGruppo
   │           │                     │
   └──────────┴──────────────────┘
                  │
               api.js  ←── import.meta.env.VITE_API_URL
                  │
               fetch()
                  │
          ASP.NET Core API
          (Azure / localhost)
```

---

## Punti Chiave per la Presentazione

1. **Lazy initializer in useState**: `useState(() => ...)` esegue la funzione una sola volta — fondamentale per operazioni costose come la lettura da localStorage.

2. **Routing condizionale vs React Router**: la scelta di non usare URL che cambiano è consapevole e appropriata per questo progetto. Sapere la differenza e quando usare l'uno o l'altro.

3. **refreshKey pattern**: incrementare una chiave numerica per forzare il re-fetch è un pattern semplice ma efficace. L'alternativa sarebbe un sistema di eventi o un context globale.

4. **VITE_ prefix**: variabili senza prefisso non sono esposte al browser. È una misura di sicurezza di Vite per evitare di esporre accidentalmente segreti nel bundle.

5. **api.js restituisce Response grezza**: permette al componente di gestire i codici HTTP (200, 401, 404, 409, 500) in modo specifico anziché ricevere sempre un oggetto JSON.
