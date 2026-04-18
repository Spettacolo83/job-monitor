# Scraper Proxy Service — Design Specification

**Data**: 2026-04-18
**Autore**: Stefano Russello + Claude
**Stato**: Approvato

## Obiettivo

Servizio proxy di scraping su VPS che usa un headless browser (Playwright + Chromium) per renderizzare pagine web, superare protezioni Cloudflare e restituire HTML pulito. Utilizzabile dal Job Monitor e da qualsiasi altro progetto futuro.

## Architettura

```
Client (Routine / curl / qualsiasi app)
  → GET https://scraper.followtheflowai.com/fetch?url=<encoded_url>
    Header: X-API-Key: <key>
      ↓
  EasyPanel (Docker container)
  Node.js + Express + Playwright
      ↓
  Chromium headless renderizza la pagina
  Attende caricamento JS, supera Cloudflare
      ↓
  Restituisce HTML renderizzato (JSON con campo html)

Dashboard admin:
  → https://scraper.followtheflowai.com/admin
    Login con username + password
      ↓
  Pagina HTML statica per gestire:
  - API keys
  - Rate limits
  - Whitelist domini (opzionale)
  - Log ultime richieste
```

## Infrastruttura

- **VPS**: Contabo, IP 161.97.125.230
- **Pannello**: EasyPanel v2.28.0 su https://panel.followtheflowai.com
- **RAM disponibile**: ~5.8 GB liberi (8.3 totali, 2.5 usati)
- **Deploy**: progetto EasyPanel "scraper", app Docker da Git repo
- **Dominio**: scraper.followtheflowai.com (configurato via EasyPanel)
- **Repository**: Spettacolo83/scraper-proxy

## API Endpoints

### Scraping (richiedono API key)

| Endpoint | Metodo | Descrizione |
|---|---|---|
| `GET /fetch?url=<url>` | GET | Renderizza la URL con Playwright, restituisce HTML |
| `GET /fetch?url=<url>&format=text` | GET | Restituisce solo il testo visibile (senza tag HTML) |
| `GET /health` | GET | Health check (no auth) — restituisce `{"status": "ok"}` |

**Headers richiesti per /fetch**: `X-API-Key: <key>`

**Risposta successo**:
```json
{
  "success": true,
  "url": "https://indeed.com/...",
  "html": "<html>...</html>",
  "status_code": 200,
  "elapsed_ms": 3200
}
```

**Risposta con format=text**:
```json
{
  "success": true,
  "url": "https://indeed.com/...",
  "text": "Job Title: iOS Developer\nCompany: Acme Corp\n...",
  "status_code": 200,
  "elapsed_ms": 3200
}
```

**Risposta errore**:
```json
{
  "success": false,
  "error": "Timeout after 30s",
  "url": "https://..."
}
```

### Admin Dashboard (richiede login)

| Endpoint | Metodo | Descrizione |
|---|---|---|
| `GET /admin` | GET | Pagina HTML dashboard |
| `POST /admin/login` | POST | Login con username + password, restituisce cookie sessione |
| `POST /admin/logout` | POST | Logout, cancella sessione |
| `GET /admin/api/config` | GET | Legge la config corrente |
| `PUT /admin/api/config` | PUT | Aggiorna la config |
| `GET /admin/api/logs` | GET | Ultime 100 richieste |

## Configurazione

File `config.json` persistito nel volume Docker:

```json
{
  "admin": {
    "username": "admin",
    "password": "da-cambiare-al-primo-accesso"
  },
  "api_keys": [
    {
      "key": "job-monitor-xxxxx",
      "name": "Job Monitor Routine",
      "created": "2026-04-18"
    }
  ],
  "rate_limit": {
    "max_requests_per_minute": 30,
    "max_concurrent": 3
  },
  "whitelist": [],
  "browser": {
    "timeout_ms": 30000,
    "wait_after_load_ms": 2000,
    "user_agent": "Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36"
  }
}
```

### Regole configurazione

- `whitelist` vuota (`[]`) = tutti i domini ammessi
- `whitelist` con valori (es. `["indeed.com", "infojobs.it"]`) = solo quei domini ammessi
- `api_keys` è un array — supporta multiple chiavi per progetti diversi
- `rate_limit.max_concurrent` limita il numero di browser aperti contemporaneamente (risparmio RAM)
- `browser.wait_after_load_ms` è il tempo extra di attesa dopo il caricamento (per Cloudflare challenge che ha animazioni JS)

## Dashboard Admin

Pagina HTML statica singola servita su `/admin`. Funzionalità:

### Sezione API Keys
- Lista delle API keys attive con nome e data creazione
- Bottone per aggiungere nuova key (generata automaticamente)
- Bottone per eliminare key esistente

### Sezione Rate Limiting
- Campo per modificare max richieste/minuto
- Campo per modificare max browser concorrenti

### Sezione Whitelist
- Lista dei domini ammessi
- Campo per aggiungere dominio
- Bottone per eliminare dominio
- Indicatore se whitelist è attiva o disattivata (vuota = disattivata)

### Sezione Log
- Tabella con ultime 100 richieste
- Colonne: timestamp, URL, API key (nome), status, durata (ms)
- Auto-refresh ogni 10 secondi

### Sezione Credenziali Admin
- Form per cambiare username e password

### Stile
- Dark mode (coerente con EasyPanel)
- Layout responsivo, font monospace per i dati tecnici
- Nessun framework CSS — CSS custom minimale

## Docker e Deploy

### Dockerfile

Basato su `mcr.microsoft.com/playwright:v1.52.0-noble`:
- Include Chromium pre-installato
- Node.js 20
- Dimensione immagine: ~1.2 GB
- RAM a runtime: ~300-500 MB (dipende da pagine concorrenti)

### Struttura repository

```
Spettacolo83/scraper-proxy
├── Dockerfile
├── package.json
├── src/
│   ├── server.js          # Express server + routes
│   ├── scraper.js          # Playwright browser management
│   ├── config.js           # Config read/write con persistenza su file
│   ├── auth.js             # API key + admin auth middleware (cookie session)
│   ├── rate-limiter.js     # Rate limiting in-memory
│   └── logger.js           # Request logging in-memory (ultime 100)
├── public/
│   └── admin.html          # Dashboard HTML statica
├── config.default.json     # Config di default (copiato in config.json al primo avvio)
├── .dockerignore
└── README.md
```

### Deploy su EasyPanel

1. Crea progetto "scraper" su EasyPanel
2. Aggiungi app di tipo "App" da GitHub repo `Spettacolo83/scraper-proxy`
3. Configura build: Dockerfile
4. Dominio: `scraper.followtheflowai.com` con HTTPS automatico
5. Volume persistente: `/app/data` → per `config.json` e log
6. Porta esposta: 3000
7. Variabili d'ambiente: nessuna (tutto in config.json)

## Gestione Browser

### Pool di browser

- Playwright lancia **un browser Chromium** all'avvio del container
- Le richieste aprono **nuove pagine** (tab) nel browser esistente (più veloce che lanciare browser nuovi)
- `max_concurrent` limita il numero di pagine aperte simultaneamente
- Se tutte le slot sono occupate, le richieste vengono messe in coda
- Se la coda supera 10 richieste, restituisce 429 Too Many Requests

### Ciclo di vita di una richiesta /fetch

1. Valida API key
2. Controlla rate limit
3. Controlla whitelist (se attiva)
4. Apre nuova pagina nel browser
5. Naviga alla URL con timeout configurato
6. Attende `wait_after_load_ms` dopo il caricamento (per Cloudflare)
7. Estrae HTML o testo
8. Chiude la pagina
9. Restituisce il risultato
10. Logga la richiesta

### Gestione errori

- Timeout → `{"success": false, "error": "Timeout after 30000ms"}`
- Pagina non trovata → `{"success": false, "error": "HTTP 404"}`
- Browser crash → restart automatico del browser, retry della richiesta
- URL non valida → `{"success": false, "error": "Invalid URL"}`

## Sicurezza

- **API key obbligatoria** per `/fetch` — senza key = 401 Unauthorized
- **Login username+password** per `/admin` — sessione via cookie httpOnly
- **HTTPS** gestito da EasyPanel/Traefik automaticamente
- **Rate limiting** per prevenire abusi
- **Whitelist opzionale** per restringere i domini
- **No esecuzione di JavaScript arbitrario** — il client invia solo una URL, non può iniettare codice

## Integrazione con Job Monitor

Dopo il deploy del proxy, aggiornare nel repo `Spettacolo83/job-monitor`:

### config/sites.json
Aggiungere `"use_proxy": true` ai siti che necessitano del proxy e riabilitarli:

```json
{
  "id": "indeed_uk",
  "name": "Indeed UK",
  "enabled": true,
  "use_proxy": true,
  ...
}
```

### routine/prompt.md
Aggiornare Step 2 per usare il proxy quando `use_proxy` è true:

```bash
# Senza proxy (API/RSS che funzionano con curl)
curl -s -L --max-time 30 "{url}" -H "User-Agent: ..."

# Con proxy (siti Cloudflare/JS-rendered)
curl -s "https://scraper.followtheflowai.com/fetch?url={encoded_url}" \
  -H "X-API-Key: job-monitor-xxxxx"
```

### GitHub Secrets
Aggiungere `SCRAPER_API_KEY` ai secrets del repo job-monitor.

## Performance attese

- Tempo per richiesta: 3-8 secondi (include rendering JS + Cloudflare wait)
- Max concorrenza: 3 pagine simultanee
- RAM: ~300-500 MB con 3 pagine aperte
- CPU: picchi durante rendering, poi idle
- Il VPS ha risorse sufficienti (5.8 GB RAM liberi, CPU al 1.3%)
