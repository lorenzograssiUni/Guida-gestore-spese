# Capitolo 5 — Deploy: Azure, Vercel e Architettura Cloud

> Questo capitolo spiega come Split Mate passa dallo sviluppo locale alla produzione: il backend ASP.NET Core deployato su **Azure App Service**, il frontend React deployato su **Vercel**, le scelte architetturali legate al cloud, e i concetti chiave del corso (CDN, HTTPS/TLS, PWA) collegati al progetto reale.

---

## 5.1 Architettura di Produzione

In produzione, Split Mate è diviso in due servizi completamente separati e indipendenti — esattamente come prevede il principio REST di **Client-Server Architecture** spiegato dal prof. Bonura: i due layer possono essere sviluppati, deployati e scalati indipendentemente, fintanto che l'interfaccia API non cambia.

```
┌─────────────────────────────────────────────────────────────┐
│                        INTERNET                             │
└──────────────┬───────────────────────────┬──────────────────┘
               │                           │
               ▼                           ▼
┌──────────────────────┐     ┌──────────────────────────────┐
│       VERCEL         │     │      AZURE APP SERVICE       │
│  (Edge Network CDN)  │     │   (backend ASP.NET Core)     │
│                      │     │                              │
│  split-mate.vercel.  │     │  split-mate-api.azurewebsi-  │
│  app                 │     │  tes.net                     │
│                      │     │                              │
│  HTML + JS + CSS     │     │  API REST + EF Core + SQLite │
│  (file statici)      │     │  + JWT + Google OAuth        │
└──────────────────────┘     └──────────────────────────────┘
         HTTPS                           HTTPS
    TLS certificato                 TLS certificato
    automatico Vercel               automatico Azure
```

---

## 5.2 HTTPS e TLS — Perché è Obbligatorio

Il prof. Bonura sottolinea che **HTTPS è un requisito obbligatorio** per qualsiasi applicazione moderna: è necessario per le PWA, per proteggere i JWT in transito, e per prevenire attacchi Man-in-the-Middle (MitM).

**TLS (Transport Layer Security)** è il protocollo crittografico che sta sotto HTTPS — è il successore di SSL, opera a livello di presentazione e garantisce:

- **Autenticazione** del server tramite certificato digitale firmato da una Certificate Authority (CA)
- **Integrità** dei dati: i messaggi non possono essere alterati in transito
- **Confidenzialità**: i dati sono crittografati end-to-end

```
Senza TLS (HTTP)                    Con TLS (HTTPS)
─────────────────────               ─────────────────────────
Browser → JWT in chiaro → Server   Browser → JWT cifrato → Server
  ↑ Chiunque sulla rete              ↑ Illeggibile agli
    può leggere il token               intercettatori
```

### Certificati in Split Mate

Sia Vercel che Azure gestiscono i certificati TLS **automaticamente** senza costo aggiuntivo, tramite servizi come **Let's Encrypt** — l'ente certificatore gratuito citato dal prof. Bonura. Non è necessario acquistare o rinnovare certificati manualmente.

| Piattaforma | Certificato TLS | Rinnovo |
|-------------|-----------------|---------|
| Vercel | Automatico (Let's Encrypt) | Automatico |
| Azure App Service | Automatico (Microsoft) | Automatico |
| Sviluppo locale | Non necessario (localhost) | — |

---

## 5.3 Frontend — Deploy su Vercel

**Vercel** è la piattaforma ottimale per deployare applicazioni React costruite con Vite. È lo stesso team dietro Next.js e la loro infrastruttura è progettata specificamente per le SPA e i framework JavaScript moderni.

### Come Funziona il Deploy

```
Developer pushes to GitHub (branch main)
        │
        ▼
Vercel rileva il push automaticamente
        │
        ▼
Vercel esegue: npm install && npm run build
        │
        ├── Vite transpila JSX → JavaScript puro
        ├── Minimizza e ottimizza i bundle
        └── Genera la cartella dist/
               ├── index.html
               ├── assets/main-abc123.js (bundle React)
               └── assets/style-def456.css
        │
        ▼
Vercel distribuisce dist/ sulla sua Edge Network (CDN globale)
        │
        ▼
https://split-mate.vercel.app → disponibile worldwide
```

### La CDN di Vercel — Perché è Importante

Come spiega il prof. Bonura, una **CDN (Content Delivery Network)** è una rete di server geograficamente distribuiti che servono i contenuti dal nodo più vicino all'utente. Vercel usa la propria Edge Network come CDN globale.

**Vantaggio concreto per Split Mate**: un utente a Tokyo non scarica il bundle JavaScript dal datacenter di Vercel a New York — lo scarica dal nodo edge più vicino in Asia. Questo riduce drasticamente il tempo di caricamento iniziale (il principale svantaggio delle SPA).

```
Senza CDN:                          Con Vercel CDN:
Utente Roma → New York (120ms)      Utente Roma → Frankfurt (15ms)
Utente Tokyo → New York (180ms)     Utente Tokyo → Tokyo (5ms)
```

### Configurazione Vercel per React SPA

Un problema tipico delle SPA: se l'utente accede direttamente a un URL come `/gruppo/42`, Vercel cerca un file fisico in quella path — che non esiste (esiste solo `index.html`). La soluzione è un file `vercel.json` che reindirizza tutto verso `index.html`:

```json
// vercel.json — nella root del progetto frontend
{
  "rewrites": [
    {
      "source": "/(.*)",
      "destination": "/index.html"
    }
  ]
}
```

Questo dice a Vercel: *"qualunque URL venga richiesto, servi sempre `index.html` — sarà React a gestire il routing lato client"*.

### Variabili d'Ambiente su Vercel

Le variabili d'ambiente vengono configurate nel pannello Vercel (Settings → Environment Variables), non nel codice:

```
VITE_API_URL = https://split-mate-api.azurewebsites.net/api
VITE_GOOGLE_CLIENT_ID = 123456789-abc.apps.googleusercontent.com
```

Vercel le inietta automaticamente durante la fase di build (`npm run build`), così il bundle finale contiene già gli URL corretti di produzione.

---

## 5.4 Backend — Deploy su Azure App Service

**Azure App Service** è il servizio PaaS (Platform as a Service) di Microsoft per hostare applicazioni web. È la scelta naturale per ASP.NET Core, essendo entrambi prodotti Microsoft.

### PaaS vs IaaS vs Serverless

| Modello | Chi gestisce | Esempio | Pro/Contro |
|---------|-------------|---------|------------|
| **IaaS** (VM) | Sviluppatore | Azure VM | Controllo totale, ma devi gestire OS, patch, sicurezza |
| **PaaS** ← Split Mate | Microsoft | Azure App Service | Zero gestione infrastruttura, focus sul codice |
| **Serverless** | Microsoft | Azure Functions | Scalabilità automatica, ma cold start e limiti di durata |

Azure App Service gestisce automaticamente: sistema operativo, aggiornamenti runtime, bilanciamento del carico, scalabilità base, e certificati TLS.

### Processo di Deploy

```bash
# Opzione 1: deploy diretto da Visual Studio
# Click destro progetto → Publish → Azure App Service

# Opzione 2: Azure CLI
dotnet publish -c Release -o ./publish
az webapp deploy --resource-group rg-split-mate \
                 --name split-mate-api \
                 --src-path ./publish

# Opzione 3: GitHub Actions (CI/CD automatico)
# Vedi sezione 5.6
```

### Configurazione Application Settings

Le variabili d'ambiente di produzione (equivalente del file `.env` locale) vengono configurate nell'Azure Portal sotto **Configuration → Application Settings**:

```
ConnectionStrings__DefaultConnection = Data Source=/home/data/splitmate.db
Jwt__Secret = [stringa segreta lunga generata casualmente]
Jwt__Issuer = split-mate-api.azurewebsites.net
Google__ClientId = 123456789-abc.apps.googleusercontent.com
```

**Perché non mettere queste variabili nel codice sorgente?** Come spiega il prof. Bonura quando parla di sicurezza, esporre segreti nel repository GitHub è una vulnerabilità critica. Azure Application Settings è la soluzione sicura: i valori esistono solo sull'infrastruttura Azure, mai nel codice.

### Il Database SQLite in Produzione

Split Mate usa **SQLite** invece di SQL Server in produzione su Azure. Il file `splitmate.db` viene salvato nel **persistent storage** di Azure App Service (`/home/data/`), che sopravvive ai riavvii dell'app.

```
Azure App Service
├── /home/
│   ├── site/wwwroot/         ← codice dell'app (.dll, appsettings.json)
│   └── data/
│       └── splitmate.db      ← database SQLite (persistent)
└── Temp/                     ← file temporanei (non persistenti!)
```

**Attenzione**: il filesystem standard di Azure App Service (`/tmp`) non è persistente. È fondamentale usare `/home/` per SQLite, altrimenti il database si azzera ad ogni restart dell'app.

---

## 5.5 CORS in Produzione

Come visto nel Capitolo 3, CORS impedisce al browser di fare chiamate API verso domini diversi dall'origine. In produzione, `Program.cs` deve essere configurato con i domini Vercel esatti:

```csharp
// Program.cs — configurazione CORS per produzione
builder.Services.AddCors(options =>
{
    options.AddPolicy("AllowVercel", policy =>
    {
        policy
            .WithOrigins(
                "https://split-mate.vercel.app",           // URL produzione
                "https://split-mate-git-main.vercel.app",  // URL preview branches
                "http://localhost:5173"                     // sviluppo locale
            )
            .AllowAnyMethod()
            .AllowAnyHeader()
            .AllowCredentials();
    });
});

// IMPORTANTE: CORS deve essere registrato PRIMA di UseAuthorization
app.UseCors("AllowVercel");
app.UseAuthentication();
app.UseAuthorization();
```

**Perché non usare `AllowAnyOrigin()`?** Usare il wildcard `*` in produzione è una vulnerabilità: permette a qualsiasi sito web di fare chiamate alle tue API. `WithOrigins` specifica esattamente quali domini possono accedere.

---

## 5.6 CI/CD con GitHub Actions

**CI/CD (Continuous Integration / Continuous Deployment)** è la pratica di automatizzare build, test e deploy ogni volta che viene fatto un push sul repository. Split Mate può usare GitHub Actions per deployare automaticamente su Azure e Vercel.

### Flusso CI/CD Completo

```
Developer fa push su GitHub (branch main)
        │
        ├──── GitHub Actions si avvia automaticamente
        │
        ├── [Job 1: Build & Test Backend]
        │      dotnet restore
        │      dotnet build
        │      dotnet test
        │      dotnet publish -c Release
        │
        ├── [Job 2: Deploy Backend → Azure]
        │      Scarica artefatto da Job 1
        │      az webapp deploy → Azure App Service
        │
        └── [Job 3: Deploy Frontend → Vercel]
               npm install
               npm run build
               vercel deploy --prod
        │
        ▼
Split Mate aggiornato in produzione in ~3-5 minuti
```

### File GitHub Actions per Azure

```yaml
# .github/workflows/deploy-backend.yml
name: Deploy Backend to Azure

on:
  push:
    branches: [ main ]
    paths:
      - 'backend/**'   # esegui solo se cambiano file nel backend

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
    - uses: actions/checkout@v4

    - name: Setup .NET
      uses: actions/setup-dotnet@v4
      with:
        dotnet-version: '8.0.x'

    - name: Restore dependencies
      run: dotnet restore ./backend/SplitMate.API.sln

    - name: Build
      run: dotnet build ./backend --no-restore -c Release

    - name: Publish
      run: dotnet publish ./backend --no-build -c Release -o ./publish

    - name: Deploy to Azure App Service
      uses: azure/webapps-deploy@v3
      with:
        app-name: 'split-mate-api'
        publish-profile: ${{ secrets.AZURE_PUBLISH_PROFILE }}
        package: ./publish
```

**`${{ secrets.AZURE_PUBLISH_PROFILE }}`**: il profilo di pubblicazione Azure viene salvato come **GitHub Secret** — mai nel codice sorgente. GitHub Secrets è il meccanismo sicuro per gestire credenziali nelle pipeline CI/CD.

---

## 5.7 PWA — Progressive Web App

Il prof. Bonura dedica ampio spazio alle **PWA (Progressive Web App)**: applicazioni web che offrono un'esperienza simile alle app native — installabili, funzionanti offline, con notifiche push.

Split Mate **non è attualmente una PWA completa**, ma i requisiti tecnici sono semplici da soddisfare con Vite:

### Requisiti per Diventare PWA

| Requisito | Split Mate | Stato |
|-----------|-----------|-------|
| **HTTPS** | ✅ Vercel + Azure (TLS automatico) | Soddisfatto |
| **Service Worker** | ❌ Non implementato | Mancante |
| **Web App Manifest** | ❌ Non implementato | Mancante |
| **Installabile** | ❌ Non installabile | Mancante |

### Il Service Worker

Un **Service Worker** è codice JavaScript che gira in background, anche quando il sito non è aperto. Come spiega il prof. Bonura, gestisce:

- **Cache offline**: intercetta le richieste di rete e le serve dalla cache quando offline
- **Push Notifications**: riceve notifiche dal server anche a browser chiuso
- **Background Sync**: sincronizza dati quando la connessione viene ripristinata

```javascript
// sw.js — Service Worker base per Split Mate (esempio)
const CACHE_NAME = 'split-mate-v1';
const ASSETS_TO_CACHE = ['/', '/index.html', '/assets/main.js'];

// Evento install: precache delle risorse statiche
self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(CACHE_NAME).then(cache => cache.addAll(ASSETS_TO_CACHE))
  );
});

// Evento fetch: servi dalla cache se disponibile (offline-first)
self.addEventListener('fetch', (event) => {
  event.respondWith(
    caches.match(event.request).then(cached => cached || fetch(event.request))
  );
});
```

### Il Web App Manifest

Il **manifest.json** dice al browser come comportarsi quando l'utente "installa" l'app dalla schermata home:

```json
// public/manifest.json
{
  "name": "Split Mate",
  "short_name": "SplitMate",
  "description": "Gestisci le spese condivise con i tuoi amici",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#6366f1",
  "icons": [
    { "src": "/icons/icon-192.png", "sizes": "192x192", "type": "image/png" },
    { "src": "/icons/icon-512.png", "sizes": "512x512", "type": "image/png" }
  ]
}
```

Con `display: "standalone"`, l'app si apre senza la barra degli indirizzi del browser — esattamente come un'app nativa.

---

## 5.8 Web Vitals e Performance in Produzione

Come illustrato dal prof. Bonura, i **Web Vitals** sono le metriche Google per misurare la qualità dell'esperienza utente. Influenzano il ranking SEO e la percezione di velocità dell'app.

| Metrica | Cosa misura | Soglia "buona" | Split Mate |
|---------|------------|----------------|-----------|
| **LCP** (Largest Contentful Paint) | Tempo per mostrare il contenuto principale | < 2.5s | Critico: primo caricamento SPA |
| **FID** (First Input Delay) | Tempo prima che l'app risponda al click | < 100ms | Buono dopo idratazione |
| **CLS** (Cumulative Layout Shift) | Stabilità visiva durante il caricamento | < 0.1 | Buono (layout fisso) |
| **TTFB** (Time to First Byte) | Velocità del server a rispondere | < 800ms | Buono (Vercel CDN) |

**Il problema della SPA**: come spiega il prof. Bonura, le SPA hanno un LCP naturalmente alto perché il browser deve prima scaricare il bundle JavaScript, eseguirlo, fare le chiamate API, e poi renderizzare il contenuto. Le ottimizzazioni principali per Split Mate:

- **Code splitting con Vite**: il bundle viene diviso in chunk, caricando solo il codice necessario per la pagina corrente
- **Lazy loading delle immagini**: le immagini vengono caricate solo quando entrano nel viewport
- **CDN di Vercel**: riduce il TTFB servendo i file statici dal nodo edge più vicino
- **Caching aggressivo**: i file JS/CSS hanno hash nel nome (`main-abc123.js`) → il browser li cachea indefinitamente, e alla nuova versione cambia solo l'hash

---

## 5.9 Sicurezza in Produzione — Checklist

Ricapitolando tutti i temi di sicurezza del prof. Bonura applicati al deploy di Split Mate:

| Vulnerabilità | Come Split Mate si protegge |
|--------------|---------------------------|
| **Intercettazione JWT** | HTTPS/TLS obbligatorio su entrambi i servizi |
| **SQL Injection** | EF Core con query parametrizzate — mai SQL raw |
| **CSRF** | JWT in `Authorization` header (non cookie) — immune a CSRF |
| **XSS** | React escapa automaticamente l'output JSX |
| **Credenziali esposte** | Azure Application Settings + GitHub Secrets (mai nel codice) |
| **CORS** | `WithOrigins` con lista esplicita (non `*`) |
| **Password in chiaro** | BCrypt con salt (Livello 4 del prof. Bonura) |
| **Librerie malevole (CDN)** | Vite usa bundle locali, non CDN esterne per le dipendenze core |
| **Malicious code nelle dipendenze** | `npm audit` + Dependabot di GitHub per vulnerabilità note |

---

## 5.10 Ambienti: Locale vs Staging vs Produzione

Una best practice professionale è avere **tre ambienti separati** per evitare di testare direttamente in produzione:

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│    LOCALE       │    │    STAGING      │    │   PRODUZIONE    │
│                 │    │                 │    │                 │
│ localhost:5173  │ →  │ preview.vercel  │ →  │ split-mate.     │
│ localhost:5000  │    │ staging.azure   │    │ vercel.app      │
│                 │    │                 │    │ split-mate-api. │
│ DB: locale      │    │ DB: test        │    │ azurewebsites   │
│ JWT: test key   │    │ JWT: staging key│    │ DB: produzione  │
└─────────────────┘    └─────────────────┘    └─────────────────┘
     .env.local           .env.staging            .env.production
```

Vercel crea automaticamente un **preview deployment** per ogni Pull Request — ogni PR ha il suo URL unico dove si può testare prima del merge su main.

---

## Riepilogo Capitolo 5

| Componente | Tecnologia | URL |
|------------|-----------|-----|
| **Frontend** | React + Vite → Vercel CDN | `split-mate.vercel.app` |
| **Backend** | ASP.NET Core → Azure App Service (PaaS) | `split-mate-api.azurewebsites.net` |
| **Database** | SQLite su `/home/data/` Azure (persistent) | — |
| **HTTPS** | TLS automatico (Vercel + Azure) | Certificati Let's Encrypt/Microsoft |
| **CI/CD** | GitHub Actions → deploy automatico al push su main | — |
| **Segreti** | Azure Application Settings + GitHub Secrets | Mai nel codice sorgente |
| **Performance** | Vercel Edge CDN + Vite code splitting + caching con hash | — |
