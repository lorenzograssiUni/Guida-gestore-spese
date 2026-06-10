# Capitolo 15 — Guida al Build Locale

In questo capitolo spieghiamo come avviare l'intero stack dell'applicazione in locale sul proprio computer, partendo da zero. La procedura è la stessa usata durante lo sviluppo e serve anche per dimostrare il funzionamento del progetto all'esame.

---

## 15.1 Prerequisiti

Prima di avviare il progetto occorre installare tre strumenti sul proprio sistema:

### .NET 10 SDK

**Cos'è:** Il Software Development Kit di Microsoft che include il compilatore C#, il runtime .NET e gli strumenti da riga di comando (`dotnet`).

**Download:** [dotnet.microsoft.com/download](https://dotnet.microsoft.com/download) → scegliere **.NET 10.0 SDK**

**Verifica installazione:**
```bash
dotnet --version
# Output atteso: 10.0.x
```

### Node.js 20+ (con npm)

**Cos'è:** Il runtime JavaScript server-side. Necessario per eseguire Vite, installare i pacchetti npm e compilare il frontend React.

**Download:** [nodejs.org](https://nodejs.org) → scegliere la versione **LTS (20.x o superiore)**

**Verifica installazione:**
```bash
node --version
# Output atteso: v20.x.x o superiore

npm --version
# Output atteso: 10.x.x o superiore
```

### Git

**Cos'è:** Il sistema di controllo versione per clonare il repository.

**Download:** [git-scm.com](https://git-scm.com)

**Verifica installazione:**
```bash
git --version
# Output atteso: git version 2.x.x
```

### Riepilogo Prerequisiti

| Strumento | Versione minima | Comando verifica |
|-----------|----------------|------------------|
| .NET SDK  | 10.0           | `dotnet --version` |
| Node.js   | 20.0           | `node --version` |
| npm       | 10.0           | `npm --version` |
| Git       | 2.x            | `git --version` |

---

## 15.2 Avvio Backend

### Passo 1 — Clona il Repository

```bash
git clone https://github.com/lorenzograssiUni/gestore-spese.git
cd gestore-spese
```

Questo scarica l'intero repository (backend + frontend) nella cartella `gestore-spese/`.

### Passo 2 — Entra nella cartella del backend

```bash
cd gestione-spese
```

La struttura a questo punto è:
```
gestore-spese/
├── gestione-spese/        ← sei qui (backend C#)
│   ├── Controllers/
│   ├── Models/
│   ├── Data/
│   ├── Migrations/
│   ├── Program.cs
│   └── gestione-spese.csproj
└── frontend-gestione-spese/ (frontend React)
```

### Passo 3 — Restore delle dipendenze NuGet

```bash
dotnet restore
```

**Cosa fa:** Legge il file `gestione-spese.csproj` e scarica tutti i pacchetti NuGet necessari:
- `Microsoft.EntityFrameworkCore.Sqlite`
- `Microsoft.EntityFrameworkCore.Tools`
- `BCrypt.Net-Next`
- `Swashbuckle.AspNetCore` (Swagger)

I pacchetti vengono salvati nella cache globale NuGet (`~/.nuget/packages/`), non nella cartella del progetto.

**Output atteso:**
```
Determining projects to restore...
Restored gestione-spese/gestione-spese.csproj (in X.XXs)
```

### Passo 4 — Migrazioni EF Core

```bash
dotnet ef database update
```

**Cosa fa:** Legge i file nella cartella `Migrations/` e li applica al database SQLite, creando il file `gestione-spese.db` se non esiste ancora.

> ⚠️ **Prerequisito nascosto:** Per usare `dotnet ef` come comando globale, serve installare gli strumenti EF Core CLI:
> ```bash
> dotnet tool install --global dotnet-ef
> ```
> Se il comando `dotnet ef` non viene trovato, eseguire questo prima.

**Output atteso:**
```
Build started...
Build succeeded.
Applying migration '20250101000000_InitialCreate'.
Done.
```

Dopo questo comando, nella cartella `gestione-spese/` troverai il file `gestione-spese.db` — il database SQLite con tutte le tabelle create.

**Alternativa — EnsureCreated:**
Se le migrazioni non sono disponibili (come nel caso del deploy su Azure), EF Core usa `EnsureCreated()` in `Program.cs` che crea il database direttamente dallo schema senza passare per i file di migrazione. Vedi Capitolo 8.

### Passo 5 — Avvia il backend

```bash
dotnet run
```

**Cosa fa:** Compila il progetto (se non già compilato) e avvia il server Kestrel di ASP.NET Core.

**Output atteso:**
```
Building...
info: Microsoft.Hosting.Lifetime
      Now listening on: http://localhost:5207
info: Microsoft.Hosting.Lifetime
      Application started. Press Ctrl+C to shut down.
```

Il backend è ora disponibile su:
- **API:** `http://localhost:5207/api/`
- **Swagger UI:** `http://localhost:5207/swagger`

### Riepilogo Comandi Backend

```bash
# Da eseguire una volta sola (primo avvio)
git clone https://github.com/lorenzograssiUni/gestore-spese.git
cd gestore-spese/gestione-spese
dotnet restore
dotnet tool install --global dotnet-ef   # solo se non già installato
dotnet ef database update

# Da eseguire ogni volta
dotnet run
```

---

## 15.3 Avvio Frontend

Aprire un **nuovo terminale** (il backend deve continuare a girare nel primo).

### Passo 1 — Entra nella cartella del frontend

```bash
# Dalla root del repository clonato
cd gestore-spese/frontend-gestione-spese
```

La struttura:
```
frontend-gestione-spese/
├── src/
│   ├── pages/
│   ├── components/
│   ├── api/
│   └── App.jsx
├── package.json
├── package-lock.json
├── vite.config.js
├── .env.local          ← da creare manualmente (non è su Git)
└── index.html
```

### Passo 2 — Crea il file `.env.local`

Questo è il passo **più importante e più spesso dimenticato**.

Crea il file `.env.local` nella cartella `frontend-gestione-spese/` con il seguente contenuto:

```env
VITE_API_URL=http://localhost:5207/api
```

**Perché è necessario:**
- Vite inietta questa variabile a **compile time** in `import.meta.env.VITE_API_URL`
- Senza di essa, tutte le chiamate API avrebbero `url = undefined` e fallirebbero silenziosamente
- Il file `.env.local` **non viene committato su Git** (sta nel `.gitignore`) per sicurezza: ogni sviluppatore lo crea localmente

**Gerarchia dei file `.env` di Vite:**

| File | Quando viene letto | Committato su Git? |
|------|--------------------|--------------------|
| `.env` | Sempre | ✅ Sì |
| `.env.local` | Sempre (locale) | ❌ No |
| `.env.production` | Solo `npm run build` | ✅ Sì |
| `.env.development` | Solo `npm run dev` | ✅ Sì |

`.env.local` ha priorità più alta di `.env` e sovrascrive i valori di quest'ultimo.

### Passo 3 — Installa le dipendenze npm

```bash
npm install
```

**Cosa fa:** Legge `package.json` e scarica tutti i pacchetti nella cartella `node_modules/`.

> ⚠️ La cartella `node_modules/` non è su Git (sta nel `.gitignore`) e può pesare centinaia di MB. Va ricreata localmente ad ogni clone.

**Output atteso:**
```
added 312 packages in 15s
```

### Passo 4 — Avvia il dev server

```bash
npm run dev
```

**Cosa fa:** Avvia Vite in modalità sviluppo con:
- **Hot Module Replacement (HMR):** ogni salvataggio aggiorna il browser istantaneamente senza ricaricare la pagina
- **Source maps:** gli errori nel browser mostrano il file JSX originale, non il bundle compilato
- **Nessuna ottimizzazione:** il codice non è minimizzato, il build è più veloce

**Output atteso:**
```
  VITE v5.x.x  ready in XXX ms

  ➜  Local:   http://localhost:5173/
  ➜  Network: use --host to expose
```

Il frontend è ora disponibile su `http://localhost:5173/`.

### Riepilogo Comandi Frontend

```bash
# Da eseguire una volta sola (primo avvio)
cd gestore-spese/frontend-gestione-spese
echo "VITE_API_URL=http://localhost:5207/api" > .env.local
npm install

# Da eseguire ogni volta
npm run dev
```

---

## Stack Completo in Esecuzione

Con entrambi i terminali attivi, il setup completo è:

```
Terminale 1 (Backend)          Terminale 2 (Frontend)
┌────────────────────┐     ┌────────────────────┐
│ dotnet run                │     │ npm run dev              │
│                           │     │                          │
│ ASP.NET Core Kestrel      │     │ Vite Dev Server          │
│ http://localhost:5207     │     │ http://localhost:5173    │
│ /api   → REST endpoints   │     │                          │
│ /swagger → Swagger UI     │     │ React SPA con HMR        │
└────────────────────┘     └────────────────────┘
         ↑                              ↑
         └────── Browser apre ──────┘
                http://localhost:5173
                         │
              React chiama il backend
              tramite VITE_API_URL
              (http://localhost:5207/api)
```

---

## Errori Comuni e Soluzioni

| Errore | Causa | Soluzione |
|--------|-------|-----------|
| `dotnet: command not found` | .NET SDK non installato | Installare .NET 10 SDK |
| `dotnet ef: command not found` | EF tools non installati | `dotnet tool install --global dotnet-ef` |
| `npm: command not found` | Node.js non installato | Installare Node.js 20 LTS |
| API risponde `undefined` o errore CORS | `.env.local` mancante o errato | Creare `.env.local` con `VITE_API_URL=http://localhost:5207/api` |
| `Port 5207 already in use` | Un altro processo occupa la porta | `kill $(lsof -t -i:5207)` (Mac/Linux) o chiudere il processo |
| `ENOENT: no such file or directory, node_modules` | `npm install` non eseguito | Eseguire `npm install` |
| Build fallisce con errori di migrazione | Database non aggiornato | Eseguire `dotnet ef database update` |
| `Failed to fetch` nel browser | Backend non avviato | Avviare `dotnet run` nel primo terminale |
