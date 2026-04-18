# Application Agent — Design Specification

**Data**: 2026-04-18
**Autore**: Stefano Russello + Claude
**Stato**: Approvato

## Obiettivo

Sistema automatico di candidatura che, partendo dagli annunci trovati dal Job Monitor, compila e invia application su qualsiasi sito di lavoro usando un browser controllato da Claude, con apprendimento progressivo delle risposte.

## Architettura

```
Tu premi "📝 Applica" su Telegram
  ↓
Webhook su scraper-proxy (VPS) riceve il tap
  ↓
Scraper-proxy:
  1. Aggiunge job a application_queue.json nel repo (commit + push)
  2. Triggera la Routine "Application Agent" via API endpoint
  ↓
Routine "Application Agent" (Claude Code, cloud Anthropic):
  1. Legge la coda, profilo, knowledge base
  2. Pilota il browser su scraper-proxy via API di sessione
  3. Loop adattivo: analizza pagina → decide azione → esegue → verifica
  4. Notifica risultato su Telegram
  ↓
Scraper-proxy (Playwright):
  Endpoint di sessione interattiva:
  - /session/create → apre browser
  - /session/:id/goto → naviga a URL
  - /session/:id/content → restituisce HTML/testo pagina
  - /session/:id/fill → compila un campo
  - /session/:id/click → clicca un elemento
  - /session/:id/select → seleziona opzione da dropdown
  - /session/:id/upload → carica file
  - /session/:id/screenshot → screenshot della pagina
  - /session/:id/close → chiude sessione
```

### Due Routines

| Routine | Trigger | Frequenza | Scopo |
|---|---|---|---|
| Job Monitor | Cron orario | 15/giorno (Max plan) | Monitoraggio annunci + notifiche |
| Application Agent | API (triggerata da webhook) | On-demand | Candidature automatiche |

## Componenti

### 1. Webhook Telegram (scraper-proxy)

Endpoint `/telegram/webhook` sullo scraper-proxy che riceve i callback dei bottoni inline.

**Setup**: registrare il webhook con Telegram Bot API:
```
POST https://api.telegram.org/bot{TOKEN}/setWebhook
  url: https://scraper.followtheflowai.com/telegram/webhook
```

**Flusso bottone "Applica":**
1. Utente preme "📝 Applica" nella notifica
2. Telegram invia callback_query al webhook
3. Webhook estrae `job_id` dal callback data
4. Aggiunge job a `application_queue.json` nel repo (via GitHub API)
5. Triggera la Routine Application Agent (via API endpoint Routines)
6. Risponde al callback: "📝 Candidatura in coda!"

**Flusso bottone "❌ Ignora":**
1. Utente preme "❌ Ignora"
2. Webhook aggiorna il messaggio: "❌ Ignorato"

**Flusso risposta a domanda insolita:**
1. Utente risponde al messaggio della domanda insolita
2. Webhook riceve il messaggio
3. Salva la risposta in `knowledge_base.json` (via GitHub API)
4. Triggera la Routine per riprendere l'application in corso

### 2. Endpoint di sessione interattiva (scraper-proxy)

Nuovi endpoint REST per il controllo interattivo del browser da parte di Claude:

#### POST /session/create
Crea una nuova sessione browser.
```json
Request: { "use_residential": true }
Response: { "session_id": "sess_abc123", "created": true }
```

#### POST /session/:id/goto
```json
Request: { "url": "https://indeed.com/job/12345" }
Response: { "success": true, "title": "Page Title", "url": "final_url" }
```

#### GET /session/:id/content
```json
Request: query param ?format=text|html
Response: { "success": true, "content": "...", "url": "current_url" }
```

#### POST /session/:id/fill
```json
Request: { "selector": "#email", "value": "stefano.russello+jobs@gmail.com" }
Response: { "success": true }
```
Se `selector` non funziona, supporta anche ricerca per label:
```json
Request: { "label": "Email", "value": "stefano.russello+jobs@gmail.com" }
```

#### POST /session/:id/click
```json
Request: { "selector": "#submit-btn" }
Response: { "success": true, "navigation": true, "new_url": "..." }
```
Supporta anche ricerca per testo:
```json
Request: { "text": "Apply Now" }
```

#### POST /session/:id/select
```json
Request: { "selector": "#country", "value": "Spain" }
Response: { "success": true }
```

#### POST /session/:id/upload
```json
Request: { "selector": "#cv-upload", "file_url": "https://raw.githubusercontent.com/.../CV.pdf" }
Response: { "success": true }
```

#### GET /session/:id/screenshot
```json
Response: { "success": true, "image_base64": "..." }
```

#### DELETE /session/:id
Chiude la sessione e il browser context.

**Autenticazione**: stessa API key dello scraper (`X-API-Key` header).

**Timeout sessione**: auto-chiusura dopo 5 minuti di inattività.

**Routing residenziale**: se `use_residential: true` e il dominio è nella lista residential_proxy, la sessione viene creata sul Mac via Tailscale.

### 3. Application Profile (job-monitor repo)

File `config/application_profile.json`:

```json
{
  "personal": {
    "full_name": "Stefano Russello",
    "first_name": "Stefano",
    "last_name": "Russello",
    "email": "stefano.russello+jobs@gmail.com",
    "email_primary": "stefano.russello@gmail.com",
    "phone": "+34 633666065",
    "location": "Petra, Mallorca, Spain",
    "country": "Spain",
    "nationality": "Italian",
    "right_to_work_eu": true,
    "linkedin": "https://www.linkedin.com/in/stefano-russello-579a4471/",
    "github": "https://github.com/Spettacolo83/",
    "website": "https://www.stefanorussello.it",
    "cv_url": "https://raw.githubusercontent.com/Spettacolo83/job-monitor/main/assets/CV_Stefano_Russello_EN.pdf"
  },
  "work_preferences": {
    "work_mode": "Full remote only",
    "contract_type": "Freelance / B2B / Contractor",
    "availability": "Immediately available",
    "notice_period": "None",
    "why_leaving": "My last project has concluded and I'm actively looking for new opportunities to grow as a developer",
    "relocation": false
  },
  "salary": {
    "strategy": "high_end_of_range",
    "default_hourly_eur": "25-30",
    "default_daily_eur": "200-240",
    "default_annual_eur": "35000-42000",
    "notes": "If job posting specifies a range, stay within 80-100% of the high end. Never exceed the max. Adapt currency to the job's country."
  },
  "languages_spoken": [
    { "language": "Italian", "level": "Native" },
    { "language": "Spanish", "level": "Fluent" },
    { "language": "English", "level": "Fluent" }
  ],
  "response_language": "Match the language of the job posting and form. Use IT/ES/EN only.",
  "cover_letter_template": {
    "max_words": 200,
    "paragraphs": 3,
    "structure": [
      "Who I am + why this specific role interests me",
      "Relevant skills for THIS specific job (pick from profile)",
      "Why I'm a good fit: remote experience, EU timezone, immediate availability"
    ],
    "tone": "Professional but personal, not generic. Never mention AI.",
    "language": "Same as job posting"
  }
}
```

### 4. Knowledge Base (job-monitor repo)

File `config/knowledge_base.json`:

```json
{
  "version": 1,
  "learned_answers": {
    "linkedin_url": {
      "value": "https://www.linkedin.com/in/stefano-russello-579a4471/",
      "source": "profile",
      "date": "2026-04-18"
    },
    "github_url": {
      "value": "https://github.com/Spettacolo83/",
      "source": "profile",
      "date": "2026-04-18"
    },
    "portfolio_url": {
      "value": "https://www.stefanorussello.it",
      "source": "profile",
      "date": "2026-04-18"
    },
    "right_to_work_eu": {
      "value": "Yes — Italian citizen, EU national",
      "source": "profile",
      "date": "2026-04-18"
    },
    "availability": {
      "value": "Immediately available",
      "source": "profile",
      "date": "2026-04-18"
    }
  },
  "field_mappings": {
    "linkedin": "linkedin_url",
    "linkedin url": "linkedin_url",
    "linkedin profile": "linkedin_url",
    "perfil linkedin": "linkedin_url",
    "github": "github_url",
    "github url": "github_url",
    "github profile": "github_url",
    "portfolio": "portfolio_url",
    "portfolio url": "portfolio_url",
    "personal website": "portfolio_url",
    "sito web": "portfolio_url",
    "sitio web": "portfolio_url",
    "right to work": "right_to_work_eu",
    "work authorization": "right_to_work_eu",
    "permesso di lavoro": "right_to_work_eu",
    "permiso de trabajo": "right_to_work_eu",
    "when can you start": "availability",
    "disponibilità": "availability",
    "disponibilidad": "availability",
    "start date": "availability",
    "fecha de inicio": "availability",
    "data di inizio": "availability"
  },
  "standard_questions": {
    "motivation": "Respond based on the specific job description and company. Mention relevant experience from profile.",
    "describe_project": "Choose from: FA Player (streaming app), Roche chat app (healthcare), StreamAMG SDK (media SDK). Pick the most relevant to the job.",
    "experience_with_X": "If X is in profile → detailed answer. If not → 'Eager to learn, experienced with similar technology Y'",
    "why_leaving": "My last project has concluded and I'm actively looking for new opportunities to grow as a developer and take on new challenges.",
    "strengths": "10+ years mobile experience, strong remote work skills, adaptable to new technologies, clean code advocate",
    "weaknesses": "Frame positively: 'I can be overly detail-oriented, which I manage by setting clear priorities and deadlines'"
  }
}
```

### 5. Application Queue (job-monitor repo)

File `data/application_queue.json`:

```json
{
  "queue": [],
  "processing": null
}
```

Ogni entry nella coda:
```json
{
  "job_id": "indeed_abc123",
  "url": "https://indeed.com/job/12345",
  "title": "Junior iOS Developer",
  "company": "Acme Corp",
  "source": "indeed_it",
  "score": 85,
  "requested_at": "2026-04-18T14:30:00Z",
  "requested_by": "manual",
  "status": "pending"
}
```

### 6. Application Tracking (job-monitor repo)

File `data/applications.json`:

```json
{
  "applications": [],
  "stats": {
    "total_applied": 0,
    "total_success": 0,
    "total_failed": 0,
    "total_pending": 0
  }
}
```

Ogni application tracciata:
```json
{
  "id": "apply_indeed_abc123",
  "job_id": "indeed_abc123",
  "url": "https://indeed.com/job/12345",
  "title": "Junior iOS Developer",
  "company": "Acme Corp",
  "source": "indeed_it",
  "score": 85,
  "status": "success|failed|pending_question|in_progress",
  "requested_at": "2026-04-18T14:30:00Z",
  "requested_by": "manual|auto",
  "started_at": "2026-04-18T15:00:00Z",
  "completed_at": "2026-04-18T15:02:30Z",
  "failure_reason": null,
  "steps_completed": ["navigate", "login", "fill_name", "fill_email", "upload_cv", "submit"],
  "steps_remaining": [],
  "fields_filled": {},
  "fields_remaining": [],
  "answers_given": {},
  "cover_letter_generated": false,
  "new_knowledge_learned": [],
  "screenshots": {}
}
```

### 7. Application Settings (job-monitor repo)

Aggiunto a `config/scoring_rules.json` o file separato `config/application_settings.json`:

```json
{
  "application_mode": "manual",
  "auto_apply_threshold": 80,
  "max_applications_per_day": 20,
  "application_routine_api_url": "https://api.claude.ai/routines/xxx/trigger",
  "application_routine_api_token": "from_env:APPLICATION_ROUTINE_TOKEN"
}
```

## Flusso dell'Application Agent (Routine)

### Prompt della Routine Application Agent

La Routine legge `routine/application_prompt.md` ed esegue:

```
1. LOAD
   - Legge application_queue.json → prende il primo job "pending"
   - Legge application_profile.json → dati personali
   - Legge knowledge_base.json → risposte apprese
   - Legge site_credentials dalla dashboard admin (via API)
   - Segna il job come "in_progress"

2. CREATE SESSION
   - POST /session/create su scraper-proxy
   - use_residential: true se il dominio è nella lista

3. NAVIGATE
   - POST /session/:id/goto → URL dell'annuncio
   - GET /session/:id/content → analizza la pagina
   - Cerca bottone "Apply" / "Candidati" / equivalente

4. AUTH CHECK
   - La pagina chiede login?
   - SÌ + ho credenziali → login automatico
   - SÌ + no credenziali → registrazione con email alias + password generata
   - Salva nuove credenziali via dashboard API

5. FORM LOOP
   Per ogni campo nella pagina:
   a. GET /session/:id/content → leggi la pagina
   b. Identifica il prossimo campo da compilare
   c. Cerca in knowledge_base.field_mappings → ho la risposta?
      - SÌ → POST /session/:id/fill con il valore
      - NO, campo standard (nome/email/tel) → compila dal profilo
      - NO, domanda standard → Claude genera risposta nella lingua del form
      - NO, upload CV → POST /session/:id/upload
      - NO, cover letter → Claude genera e compila
      - NO, salary → applica strategia salariale
      - NO, domanda insolita → screenshot + notifica Telegram + attendi
   d. Verifica: GET /session/:id/content → errori di validazione?
      - SÌ → correggi e riprova (max 2 tentativi)
   e. Prossimo campo o prossima pagina

6. PRE-SUBMIT
   - GET /session/:id/screenshot → salva come pre_submit
   - Verifica tutti i campi compilati

7. SUBMIT
   - POST /session/:id/click → bottone submit
   - GET /session/:id/content → verifica pagina di conferma
   - GET /session/:id/screenshot → salva come post_submit

8. REPORT
   - Aggiorna applications.json con risultato
   - Aggiorna knowledge_base.json con eventuali nuove risposte apprese
   - Notifica Telegram con risultato (successo/fallimento/in attesa)
   - Commit + push su main

9. NEXT
   - Ci sono altri job in coda? → torna a step 1
   - Coda vuota → fine
```

### Gestione errori

| Errore | Azione |
|---|---|
| Pagina non caricata | Retry 1 volta. Se fallisce → report "sito non raggiungibile" |
| Campo non riconosciuto | Prova label, placeholder, aria-label. Se impossibile → chiedi su Telegram |
| Validazione fallita | Leggi errore, correggi, riprova (max 2) |
| CAPTCHA | Prova a procedere. Se bloccato → screenshot + report + link manuale |
| Timeout sessione | Riprendi da ultimo step completato |
| Redirect inaspettato | Analizza nuova pagina, adatta |
| Form multi-pagina | Traccia progresso per pagina |
| Errore server sito | Report + link manuale |
| Domanda insolita | Screenshot + notifica Telegram + salva in KB dopo risposta |
| Browser crash | Riapri sessione + riprendi da ultimo step |

### Notifiche Telegram aggiornate

**Notifica Job Monitor (con bottoni inline):**
```
🟢 Score: 85/100 | Junior iOS Developer
🏢 Acme Corp — Germania
💼 Freelance/B2B
🔧 Swift, SwiftUI
📅 2h fa — via Indeed

✅ Perché matcha: ...

🔗 https://indeed.com/job/12345

[📝 Applica]  [❌ Ignora]
```

**Candidatura in coda:**
```
📝 Candidatura in coda!
Junior iOS Developer @ Acme Corp
Verrà processata a breve...
```

**Candidatura riuscita:**
```
✅ Candidatura inviata!

Junior iOS Developer @ Acme Corp
🔧 Swift, SwiftUI | Score: 85/100
⏱ Completata in 2m 30s
📝 Cover letter generata (EN)
📎 CV caricato

🔗 https://indeed.com/job/12345
```

**Candidatura fallita:**
```
❌ Candidatura non completata

Junior Android Developer @ StartupXYZ
⚠️ Motivo: CAPTCHA non risolvibile

✅ Già compilato:
• Nome, Email, Telefono
• CV caricato

❌ Da completare:
• Cover letter
• Domanda motivazione

📸 [Vedi screenshot]
👉 Completa manualmente: https://glassdoor.com/job/xyz789
```

**Domanda insolita:**
```
🟡 Domanda durante candidatura

Junior Developer @ TechCorp (InfoJobs)
Il form chiede: "Indica il tuo NIE/NIF"

Rispondi qui con il valore.
Verrà salvato per le prossime candidature.

📸 [Vedi screenshot]
```

## Modifiche ai componenti esistenti

### Scraper-proxy (VPS)
- Aggiungere endpoint `/telegram/webhook` per bottoni inline
- Aggiungere endpoint `/session/*` per controllo browser interattivo
- Aggiungere sezione "Account" nella dashboard admin per credenziali siti
- Registrare webhook con Telegram Bot API

### Job Monitor (Routine)
- Aggiornare formato notifiche: aggiungere bottoni inline (reply_markup)
- In modalità semi_auto/full_auto: accodare automaticamente i job

### Job Monitor (repo)
- Aggiungere `config/application_profile.json`
- Aggiungere `config/knowledge_base.json`
- Aggiungere `config/application_settings.json`
- Aggiungere `data/application_queue.json`
- Aggiungere `data/applications.json`
- Aggiungere `assets/CV_Stefano_Russello_EN.pdf`
- Aggiungere `routine/application_prompt.md`

## Sessioni interattive e routing residenziale

### Creazione sessione sul Mac

Quando `use_residential: true` e il dominio è nella lista residential_proxy:

1. Il scraper-proxy VPS invia la richiesta di sessione al Mac via Tailscale
2. Il Mac crea la sessione Playwright locale
3. Tutti i comandi della sessione vengono proxati VPS → Mac
4. Il browser gira sul Mac con IP residenziale

Il residential-proxy sul Mac deve essere esteso per supportare le sessioni interattive (stessi endpoint `/session/*`).

### Fallback

Se il Mac non è raggiungibile:
1. Prova la sessione sul VPS (locale)
2. Se il sito blocca → report con motivo "Mac offline, sito richiede IP residenziale"

## Sicurezza

- Le credenziali dei siti sono nel config.json dello scraper-proxy (volume Docker, non nel repo)
- L'API key dello scraper protegge tutti gli endpoint
- Le sessioni hanno timeout di 5 minuti
- Il webhook Telegram verifica la firma del messaggio
- Le password generate per auto-registrazione vengono salvate nella dashboard

## Modalità applicazione

### Manual (default)
- Job Monitor notifica con bottoni [Applica] [Ignora]
- Utente preme "Applica" → parte la candidatura
- Ogni candidatura è una scelta esplicita

### Semi-auto
- Score ≥ threshold (default 80) → applica automaticamente
- Score < threshold → notifica con bottoni come manual
- Notifica post-candidatura in entrambi i casi

### Full-auto
- Tutto ciò che passa i filtri viene applicato
- Solo notifiche post-candidatura (successo/fallimento)

### Limiti
- Max 20 application/giorno (configurabile)
- Almeno 2 minuti tra un'application e l'altra (anti-ban)
