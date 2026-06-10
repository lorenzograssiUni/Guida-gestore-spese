# Capitolo 13 — CI/CD e YAML

In questo capitolo spieghiamo cos'è YAML, cosa significa CI/CD, e analizziamo riga per riga il file `ci.yml` del progetto che automatizza il build di backend e frontend ad ogni push su `main`.

---

## 13.1 Cos'è YAML — Sintassi e Use Case

### Definizione

**YAML** (YAML Ain't Markup Language) è un formato di serializzazione dati leggibile dall'uomo, usato per file di configurazione. A differenza di JSON (che usa `{}` e `[]`) o XML (che usa tag), YAML usa l'**indentazione** per definire la struttura.

### Sintassi Base

```yaml
# Chiave: valore (stringa)
nome: Lorenzo
versione: '1.0'

# Numero
eta: 22

# Booleano
attivo: true

# Lista (array)
lingue:
  - italiano
  - inglese
  - francese

# Oggetto annidato
indirizzo:
  citta: Camerino
  cap: 62032

# Lista di oggetti
utenti:
  - nome: Marco
    ruolo: admin
  - nome: Sara
    ruolo: user
```

### Regole Fondamentali

| Regola | Dettaglio |
|--------|----------|
| **Indentazione** | Solo spazi (mai TAB). Di solito 2 spazi per livello |
| **Commenti** | Con `#`, ignorati dal parser |
| **Stringhe** | Di solito senza virgolette; usare `'...'` o `"..."` se contengono caratteri speciali |
| **Booleani** | `true`/`false` (minuscolo) |
| **Null** | `null` o `~` |
| **Multiline** | `|` per blocco letterale (preserva newline), `>` per blocco folded (unisce le righe) |

### Differenza YAML vs JSON

```yaml
# YAML
persona:
  nome: Marco
  eta: 25
  hobby:
    - calcio
    - musica
```

```json
// JSON equivalente
{
  "persona": {
    "nome": "Marco",
    "eta": 25,
    "hobby": ["calcio", "musica"]
  }
}
```

YAML è più leggibile per configurazioni complesse; JSON è preferito per scambio dati tra API.

### Use Case Comuni di YAML

- **GitHub Actions** — file `.yml` nella cartella `.github/workflows/`
- **Docker Compose** — `docker-compose.yml` per definire servizi container
- **Kubernetes** — manifesti per pod, service, deployment
- **Ansible** — playbook di automazione
- **Azure Pipelines / GitLab CI** — pipeline CI/CD

---

## 13.2 Cos'è CI/CD — Continuous Integration / Deployment

### Il Problema che Risolve

Senza CI/CD, il ciclo di sviluppo è:
```
Sviluppatore scrive codice
    ↓
Pusha su Git
    ↓
Nessuno sa se il codice compila  ← PROBLEMA
    ↓
Settimane dopo: "perché non funziona?"
```

Con CI/CD:
```
Sviluppatore pusha codice
    ↓
Pipeline automatica si avvia entro secondi
    ↓
Build + Test automatici
    ↓
✅ Tutto OK → (opzionale) Deploy automatico
❌ Errore   → Notifica immediata allo sviluppatore
```

### Continuous Integration (CI)

**CI** è la pratica di integrare le modifiche di ogni sviluppatore nel ramo principale frequentemente (anche più volte al giorno), verificando automaticamente che il codice:
- **Compili** senza errori
- **Superi i test** automatizzati
- **Rispetti gli standard** di qualità (linting)

L'obiettivo è trovare i bug **subito**, quando il contesto è ancora fresco nella mente dello sviluppatore.

### Continuous Deployment (CD)

**CD** estende la CI: se tutti i check passano, il codice viene **deployato automaticamente** in produzione (o in staging). Il ciclo diventa:

```
Commit → CI (build + test) → CD (deploy) → In produzione
```

Nel nostro progetto il **frontend** ha CD implicito tramite **Vercel**, che fa auto-deploy ad ogni push su `main` indipendentemente dalla pipeline GitHub Actions.

### I 3 Livelli di Automazione

| Livello | Nome | Cosa fa | Il nostro progetto |
|---------|------|---------|--------------------|
| 1 | **CI** | Compila e testa | ✅ Backend + Frontend build |
| 2 | **Continuous Delivery** | Produce artefatto pronto al deploy | ⚠️ Parziale (solo build) |
| 3 | **Continuous Deployment** | Deploy automatico in produzione | ✅ Vercel (frontend) |

---

## 13.3 Il Nostro Workflow ci.yml — Analisi Riga per Riga

Ecco il file completo con spiegazione di ogni sezione:

```yaml
name: CI - Build & Test
```

Il **nome** della pipeline, mostrato nell'interfaccia GitHub Actions e nel badge del README.

---

### Sezione `on` — Trigger della Pipeline

```yaml
on:
  push:
    branches: [ main ]
  pull_request:
    branches: [ main ]
```

Definiamo **quando** la pipeline si avvia:

| Evento | Quando scatta |
|--------|---------------|
| `push: branches: [main]` | Ad ogni commit pushato direttamente su `main` |
| `pull_request: branches: [main]` | Quando viene aperta o aggiornata una PR verso `main` |

Questo significa che la pipeline protegge `main` su entrambi i fronti: i push diretti e le PR in entrata.

---

### Sezione `jobs` — I Due Job Paralleli

```yaml
jobs:
  backend:
    ...
  frontend:
    ...
```

Il workflow ha **due job indipendenti** che girano **in parallelo** su macchine virtuali separate. Il job `backend` e il job `frontend` non si aspettano a vicenda (nessuna direttiva `needs:`), quindi entrambi partono contemporaneamente appena la pipeline si avvia.

```
Pipeline avviata
      │
   ┌──┴──┐
   │       │
job:     job:
backend  frontend
   │       │
   │       │   (girano in parallelo)
   ↓       ↓
✅/❌    ✅/❌
```

---

### Job Backend — Build .NET

```yaml
  backend:
    name: Build Backend (.NET)
    runs-on: ubuntu-latest
```

- `name` — etichetta visibile in GitHub Actions
- `runs-on: ubuntu-latest` — il job gira su una macchina virtuale Ubuntu fornita da GitHub (2 core, 7GB RAM, gratuita per repo pubblici)

#### Step 1 — Checkout del codice

```yaml
    steps:
      - uses: actions/checkout@v4
```

`actions/checkout@v4` è un'**Action predefinita** di GitHub che clona il repository nella macchina virtuale. Senza questo step, il runner non avrebbe accesso al codice sorgente.

#### Step 2 — Installazione .NET

```yaml
      - name: Setup .NET
        uses: actions/setup-dotnet@v4
        with:
          dotnet-version: '10.0.x'
```

Installa il .NET SDK versione 10 (la `x` indica "ultima patch disponibile" nella serie 10.0). La macchina Ubuntu di GitHub non ha .NET preinstallato nella versione specifica richiesta, quindi questo step lo scarica e configura.

#### Step 3 — Restore delle dipendenze

```yaml
      - name: Restore dependencies
        run: dotnet restore gestione-spese/gestione-spese.csproj
```

`dotnet restore` scarica tutti i pacchetti NuGet elencati nel file `.csproj` (BCrypt.Net, Entity Framework Core, Swagger, ecc.). Equivale a `npm install` per il mondo .NET.

#### Step 4 — Build in modalità Release

```yaml
      - name: Build
        run: dotnet build gestione-spese/gestione-spese.csproj --no-restore --configuration Release
```

- `--no-restore` — evita di ri-scaricare i pacchetti appena installati
- `--configuration Release` — compila in modalità ottimizzata (senza simboli di debug, con ottimizzazioni del compilatore)

Se il codice C# ha errori di sintassi o tipi incompatibili, questo step fallisce e la pipeline si ferma.

---

### Job Frontend — Build React/Vite

```yaml
  frontend:
    name: Build Frontend (React/Vite)
    runs-on: ubuntu-latest
```

Anche il frontend gira su Ubuntu, ma in un **runner separato e indipendente** dal backend.

#### Step 1 — Checkout

```yaml
    steps:
      - uses: actions/checkout@v4
```

Identico al backend: clona il repo.

#### Step 2 — Installazione Node.js con cache

```yaml
      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
          cache-dependency-path: frontend-gestione-spese/package-lock.json
```

- `node-version: '20'` — installa Node.js 20 LTS
- `cache: 'npm'` — **ottimizzazione importante**: GitHub memorizza nella cache la cartella `node_modules` basandosi sull'hash del `package-lock.json`. Se le dipendenze non cambiano tra un push e l'altro, `npm ci` salta il download e usa la cache, riducendo il tempo da ~60 secondi a ~5 secondi
- `cache-dependency-path` — indica quale file usare per calcolare l'hash della cache

#### Step 3 — Installazione dipendenze

```yaml
      - name: Install dependencies
        run: cd frontend-gestione-spese && npm ci
```

`npm ci` ("clean install") è diverso da `npm install`:

| Comando | Comportamento |
|---------|---------------|
| `npm install` | Aggiorna `package-lock.json` se necessario | 
| `npm ci` | Usa **esattamente** le versioni nel `package-lock.json`, non lo modifica mai |

`npm ci` è obbligatorio in CI perché garantisce **build riproducibili**: la stessa versione di ogni pacchetto, ogni volta.

#### Step 4 — Build con variabile d'ambiente

```yaml
      - name: Build
        run: cd frontend-gestione-spese && npm run build
        env:
          VITE_API_URL: https://gestione-spese-hbhga0crf6hsagdn.swedencentral-01.azurewebsites.net/api
```

Questo step ha una particolarità importante: fornisce la variabile d'ambiente `VITE_API_URL` direttamente nel workflow.

**Perché è necessario?** Vite inietta `import.meta.env.VITE_API_URL` a **compile time**: se la variabile non è disponibile durante il build, il valore sarebbe `undefined` nell'app compilata. Il file `.env.local` (usato in locale) non viene committato su Git per sicurezza, quindi la pipeline deve fornire il valore esplicitamente.

`npm run build` esegue `vite build`, che:
1. Compila tutto il JSX/JS in JavaScript standard
2. Effettua il **tree-shaking** (rimuove il codice non usato)
3. Crea il bundle ottimizzato nella cartella `dist/`

---

### Schema Completo del Workflow

```
Push su main
     │
     │  GitHub Actions si avvia
     │
  ┌──┴──────────────────────────────────────────┐
  │                                          │
  │  JOB: backend (ubuntu-latest)            │  JOB: frontend (ubuntu-latest)
  │  ├─ checkout@v4                          │  ├─ checkout@v4
  │  ├─ setup-dotnet@v4 (v10.0.x)           │  ├─ setup-node@v4 (v20 + cache)
  │  ├─ dotnet restore                       │  ├─ npm ci
  │  └─ dotnet build --configuration Release │  └─ npm run build (VITE_API_URL)
  │                                          │
  └──────────────────────────────────────────┘
```

---

## 13.4 Badge di Stato nel README

Il README del progetto mostra un badge dinamico che riflette lo stato dell'ultima esecuzione della pipeline:

```markdown
![CI](https://github.com/lorenzograssiUni/gestore-spese/actions/workflows/ci.yml/badge.svg)
```

### Come Funziona il Badge

GitHub genera automaticamente un'immagine SVG per ogni workflow. L'URL segue il pattern:
```
https://github.com/{owner}/{repo}/actions/workflows/{file}.yml/badge.svg
```

Il badge cambia aspetto in base allo stato:

| Stato pipeline | Badge mostrato | Colore |
|----------------|----------------|--------|
| Ultima esecuzione OK | `CI passing` | Verde 🟢 |
| Ultima esecuzione fallita | `CI failing` | Rosso 🔴 |
| In esecuzione | `CI running` | Giallo 🟡 |
| Nessuna esecuzione | `CI no status` | Grigio ⚪ |

Il badge viene incluso nel README con la sintassi Markdown delle immagini `![testo-alternativo](url)`. Essendo un URL esterno, GitHub lo aggiorna in tempo reale ad ogni visita alla pagina.

### Perché i Badge Sono Importanti

1. **Visibilità immediata** — chiunque visiti il repository vede subito se il codice è "sano"
2. **Accountability** — spinge i developer a non pushare codice rotto su `main`
3. **Professionalità** — segnale standard nei progetti open source e accademici

---

## Differenza tra CI Pipeline e Deploy su Azure/Vercel

Un punto di confusione frequente: la nostra pipeline CI **non fa il deploy**. Fa solo il **build**.

Il deploy avviene separatamente:

| Servizio | Trigger del deploy | Cosa deploya |
|----------|--------------------|--------------|
| **Vercel** | Push su `main` (webhook automatico) | Frontend React |
| **Azure App Service** | Deploy manuale o Azure DevOps | Backend .NET |
| **GitHub Actions CI** | Push su `main` o PR | Solo verifica che compili |

Quindi il flusso completo è:

```
Developer pusha su main
         │
    ┌───┴────────────────────────────────┐
    │                                   │
GitHub Actions                       Vercel webhook
(build + verifica)                   (build + deploy)
    │                                   │
    │ badge verde/rosso               Frontend in produzione
    │ in ~2 minuti                     in ~1 minuto
```
