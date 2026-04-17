# Job Monitor Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Create a Claude Code Routine that monitors 11 EU job sites hourly and sends matching job listings to Telegram.

**Architecture:** A single Claude Code Routine runs on a cron schedule against a GitHub repo. The repo contains configuration files (sites, profile, scoring rules), state (seen jobs), and the routine prompt. Claude fetches job listings via curl (APIs/RSS/HTML), applies elimination filters and scoring, notifies via Telegram Bot API, and commits updated state back to the repo.

**Tech Stack:** Claude Code Routines (cloud), GitHub repo for config/state, Telegram Bot API, curl for HTTP requests, JSON for data, Markdown for routine prompt.

---

### Task 1: Create GitHub repository and base structure

**Files:**
- Create: `README.md`
- Create: `.gitignore`

- [ ] **Step 1: Create remote repo on GitHub**

```bash
gh repo create Spettacolo83/job-monitor --public --description "Automated EU job monitor for mobile developer positions — powered by Claude Code Routines" --clone=false
```

- [ ] **Step 2: Set the remote origin on the local repo**

```bash
git remote add origin git@github.com:Spettacolo83/job-monitor.git
```

- [ ] **Step 3: Rename branch to main**

```bash
git branch -M main
```

- [ ] **Step 4: Create .gitignore**

```
# .gitignore
.DS_Store
*.swp
*.swo
.env
```

- [ ] **Step 5: Create README.md**

```markdown
# Job Monitor

Automated monitoring of EU job sites for mobile developer positions (iOS, Android, Flutter, React Native).

Powered by [Claude Code Routines](https://claude.ai/code).

## How it works

A Claude Code Routine runs on a scheduled cron and:
1. Fetches job listings from 11 EU job sites (APIs, RSS feeds, and HTML pages)
2. Applies elimination filters (must be full remote + freelance/B2B)
3. Scores remaining jobs (0-100) based on tech match, seniority, language, freshness
4. Sends matching jobs (score ≥ 50) to Telegram
5. Sends a daily summary at 22:00 CET
6. Commits updated state to this repo

## Configuration

- `config/sites.json` — Sites to monitor and search parameters
- `config/profile.json` — Candidate profile and preferences
- `config/scoring_rules.json` — Scoring weights and thresholds

## State

- `data/seen_jobs.json` — Previously seen job IDs (auto-managed, cleaned after 30 days)
```

- [ ] **Step 6: Commit**

```bash
git add .gitignore README.md
git commit -m "chore: add README and .gitignore"
```

---

### Task 2: Create sites configuration

**Files:**
- Create: `config/sites.json`

- [ ] **Step 1: Create config directory**

```bash
mkdir -p config
```

- [ ] **Step 2: Create config/sites.json**

```json
{
  "version": 1,
  "sites": [
    {
      "id": "indeed_uk",
      "name": "Indeed UK",
      "type": "rss",
      "base_url": "https://co.uk.indeed.com/rss",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS", "Swift", "Android", "Kotlin", "Flutter", "React Native", "Mobile"],
      "enabled": true
    },
    {
      "id": "indeed_it",
      "name": "Indeed Italia",
      "type": "rss",
      "base_url": "https://it.indeed.com/rss",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS", "Swift", "Android", "Kotlin", "Flutter", "React Native", "Mobile", "sviluppatore mobile"],
      "enabled": true
    },
    {
      "id": "indeed_de",
      "name": "Indeed Deutschland",
      "type": "rss",
      "base_url": "https://de.indeed.com/rss",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS", "Swift", "Android", "Kotlin", "Flutter", "React Native", "Mobile"],
      "enabled": true
    },
    {
      "id": "indeed_es",
      "name": "Indeed España",
      "type": "rss",
      "base_url": "https://es.indeed.com/rss",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS", "Swift", "Android", "Kotlin", "Flutter", "React Native", "Mobile", "desarrollador móvil"],
      "enabled": true
    },
    {
      "id": "indeed_fr",
      "name": "Indeed France",
      "type": "rss",
      "base_url": "https://fr.indeed.com/rss",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS", "Swift", "Android", "Kotlin", "Flutter", "React Native", "Mobile"],
      "enabled": true
    },
    {
      "id": "remoteok",
      "name": "RemoteOK",
      "type": "json_api",
      "base_url": "https://remoteok.com/api",
      "params": {},
      "keywords": ["ios", "android", "mobile", "swift", "kotlin", "flutter", "react-native"],
      "notes": "API returns all jobs, filter client-side by tags",
      "enabled": true
    },
    {
      "id": "remotive",
      "name": "Remotive",
      "type": "json_api",
      "base_url": "https://remotive.com/api/remote-jobs",
      "params": {
        "category": "software-dev"
      },
      "keywords": ["ios", "android", "mobile", "swift", "kotlin", "flutter", "react native"],
      "notes": "Filter by category, then match keywords in title/description",
      "enabled": true
    },
    {
      "id": "welcometothejungle",
      "name": "Welcome to the Jungle",
      "type": "html",
      "base_url": "https://www.welcometothejungle.com/en/jobs",
      "params": {
        "query": "{keyword}",
        "refinementList[contract_type][]": "freelance"
      },
      "keywords": ["iOS", "Android", "Mobile", "Flutter", "React Native", "Swift", "Kotlin"],
      "enabled": true
    },
    {
      "id": "jobicy",
      "name": "Jobicy",
      "type": "rss",
      "base_url": "https://jobicy.com/feed/newjob",
      "params": {
        "search": "{keyword}"
      },
      "keywords": ["mobile developer", "iOS", "Android", "Swift", "Kotlin", "Flutter", "React Native"],
      "enabled": true
    },
    {
      "id": "infojobs_it",
      "name": "InfoJobs Italia",
      "type": "html",
      "base_url": "https://www.infojobs.it/lavoro",
      "params": {
        "keyword": "{keyword}"
      },
      "keywords": ["iOS", "Swift", "Android", "Kotlin", "Flutter", "React Native", "Mobile", "sviluppatore mobile"],
      "enabled": true
    },
    {
      "id": "infojobs_es",
      "name": "InfoJobs España",
      "type": "html",
      "base_url": "https://www.infojobs.net/ofertas-trabajo",
      "params": {
        "keyword": "{keyword}"
      },
      "keywords": ["iOS", "Swift", "Android", "Kotlin", "Flutter", "React Native", "Mobile", "desarrollador móvil"],
      "enabled": true
    },
    {
      "id": "stepstone",
      "name": "StepStone",
      "type": "html",
      "base_url": "https://www.stepstone.de/jobs",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS", "Swift", "Android", "Kotlin", "Flutter", "React Native", "Mobile"],
      "enabled": true
    },
    {
      "id": "glassdoor",
      "name": "Glassdoor",
      "type": "html",
      "base_url": "https://www.glassdoor.com/Job/jobs.htm",
      "params": {
        "sc.keyword": "{keyword}"
      },
      "keywords": ["iOS developer", "Android developer", "Mobile developer", "Flutter developer", "React Native developer"],
      "notes": "Aggressive anti-bot — may fail frequently",
      "enabled": true
    },
    {
      "id": "monster",
      "name": "Monster",
      "type": "html",
      "base_url": "https://www.monster.com/jobs/search",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS developer", "Android developer", "Mobile developer", "Flutter", "React Native"],
      "enabled": true
    },
    {
      "id": "honeypot",
      "name": "Honeypot",
      "type": "html",
      "base_url": "https://www.honeypot.io/pages/jobs",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS", "Android", "Mobile", "Swift", "Kotlin", "Flutter", "React Native"],
      "enabled": true
    },
    {
      "id": "landingjobs",
      "name": "Landing.Jobs",
      "type": "html",
      "base_url": "https://landing.jobs/jobs",
      "params": {
        "q": "{keyword}"
      },
      "keywords": ["iOS", "Android", "Mobile", "Swift", "Kotlin", "Flutter", "React Native"],
      "enabled": true
    }
  ]
}
```

- [ ] **Step 3: Validate JSON is well-formed**

```bash
python3 -c "import json; json.load(open('config/sites.json')); print('OK')"
```

Expected: `OK`

- [ ] **Step 4: Commit**

```bash
git add config/sites.json
git commit -m "feat: add sites configuration for 11 EU job sites"
```

---

### Task 3: Create candidate profile configuration

**Files:**
- Create: `config/profile.json`

- [ ] **Step 1: Create config/profile.json**

```json
{
  "version": 1,
  "candidate": {
    "name": "Stefano Russello",
    "location": "Petra, Mallorca, Spain",
    "email": "stefano.russello@gmail.com",
    "phone": "+34 633666065",
    "website": "https://www.stefanorussello.it",
    "linkedin": "https://www.linkedin.com/in/stefano-russello-579a4471/",
    "github": "https://github.com/Spettacolo83/"
  },
  "languages": ["english", "italian", "spanish"],
  "experience_years": 10,
  "core_technologies": {
    "native_ios": ["Objective-C", "Swift", "SwiftUI", "UIKit", "SPM", "Combine", "Alamofire", "Core Data", "XCTest", "CocoaPods", "MapKit", "APNs"],
    "native_android": ["Java", "Kotlin", "Jetpack Compose", "Retrofit", "Coroutines", "Gradle", "Flow", "Material Design", "Firebase", "JUnit"],
    "cross_platform": ["Flutter", "React Native"],
    "backend": ["PHP", "MySQL", "Firebase", "REST APIs", "AWS", "GCP", "Azure", "JWT"],
    "tools": ["Git", "GitHub", "Fastlane", "CI/CD", "Jira", "Figma"]
  },
  "cross_platform_note": "Flutter and React Native are being learned — not production-level experience yet",
  "methodologies": ["Agile", "MVVM", "Clean Architecture", "SOLID"],
  "past_industries": ["streaming", "media", "healthcare", "pharma"],
  "past_employers": [
    {
      "company": "Roche",
      "role": "Android & iOS Developer",
      "period": "2025",
      "type": "remote",
      "country": "Switzerland"
    },
    {
      "company": "StreamAMG",
      "role": "Android & iOS Developer",
      "period": "2018-2025",
      "type": "remote",
      "country": "Malta"
    }
  ],
  "published_apps": ["The FA Player", "FlyToDiscover", "Cartoonito", "PDF Watch", "Barcode Organizer", "Focus Me"],
  "preferences": {
    "contract_type": "freelance_only",
    "contract_types_accepted": ["freelance", "B2B", "partita IVA", "contractor", "contract", "self-employed", "autónomo"],
    "contract_types_rejected": ["permanent", "full-time employee", "CDI", "Festanstellung", "contrato indefinido", "tempo indeterminato", "assunzione diretta", "nómina", "W-2", "convenio", "contrat salarié", "unbefristeter Vertrag"],
    "work_mode": "remote_only",
    "target_seniority": ["junior", "mid"],
    "seniority_preference": "junior preferred over mid to avoid complex technical interviews",
    "locations_accepted": ["any EU country", "worldwide if remote"],
    "notification_language": "italian"
  }
}
```

- [ ] **Step 2: Validate JSON is well-formed**

```bash
python3 -c "import json; json.load(open('config/profile.json')); print('OK')"
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add config/profile.json
git commit -m "feat: add candidate profile configuration"
```

---

### Task 4: Create scoring rules configuration

**Files:**
- Create: `config/scoring_rules.json`

- [ ] **Step 1: Create config/scoring_rules.json**

```json
{
  "version": 1,
  "elimination_filters": [
    {
      "id": "remote_only",
      "description": "Must be 100% remote — any on-site or hybrid requirement eliminates the job",
      "reject_indicators": {
        "en": ["on-site", "in-office", "office-based", "hybrid", "days in office", "relocate to", "must be based in"],
        "it": ["in sede", "in ufficio", "ibrido", "presenza richiesta", "sede di lavoro"],
        "es": ["presencial", "en oficina", "híbrido", "presencia requerida", "sede de trabajo"],
        "de": ["vor Ort", "im Büro", "hybrid", "Büropflicht"],
        "fr": ["sur site", "au bureau", "hybride", "présentiel"]
      },
      "accept_indicators": {
        "en": ["fully remote", "100% remote", "remote-first", "work from anywhere", "remote only", "distributed team"],
        "it": ["full remote", "completamente remoto", "lavoro da remoto", "100% remoto"],
        "es": ["100% remoto", "totalmente remoto", "trabajo remoto", "teletrabajo"],
        "de": ["100% remote", "vollständig remote", "Remote-Arbeit"],
        "fr": ["100% télétravail", "full remote", "entièrement à distance"]
      },
      "when_ambiguous": "do_not_reject"
    },
    {
      "id": "freelance_only",
      "description": "Must accept freelance/B2B/contractor — reject if only permanent employment",
      "reject_indicators": {
        "en": ["permanent position", "full-time employee", "W-2", "employment contract", "permanent contract", "staff position"],
        "it": ["tempo indeterminato", "assunzione diretta", "contratto dipendente", "RAL", "CCNL", "assunzione"],
        "es": ["contrato indefinido", "nómina", "convenio", "contrato laboral", "puesto fijo"],
        "de": ["Festanstellung", "unbefristeter Vertrag", "Arbeitsvertrag"],
        "fr": ["CDI", "contrat salarié", "poste permanent", "contrat de travail"]
      },
      "accept_indicators": {
        "en": ["freelance", "contractor", "B2B", "contract", "self-employed", "independent", "consulting"],
        "it": ["partita IVA", "freelance", "collaborazione", "consulenza", "contratto a progetto", "P.IVA"],
        "es": ["autónomo", "freelance", "B2B", "colaboración", "consultoría", "por proyecto"],
        "de": ["Freelancer", "Freiberufler", "Selbstständig", "Projektbasis"],
        "fr": ["freelance", "indépendant", "prestataire", "mission"]
      },
      "when_ambiguous": "do_not_reject"
    }
  ],
  "scoring_criteria": [
    {
      "id": "tech_match",
      "weight": 35,
      "description": "How well the required technologies match the candidate's skills",
      "scoring": {
        "core_native_match": 100,
        "cross_platform_match": 80,
        "generic_mobile": 60,
        "partial_match": 40,
        "no_match": 0
      },
      "notes": "Core = Swift/Kotlin/ObjC/Java. Cross-platform = Flutter/RN (learning). Generic = 'mobile developer' without specific tech"
    },
    {
      "id": "seniority",
      "weight": 25,
      "description": "Seniority level preference — junior preferred to avoid complex technical interviews",
      "scoring": {
        "junior": 100,
        "not_specified": 85,
        "mid": 70,
        "senior": 30,
        "lead_staff_principal": 10
      }
    },
    {
      "id": "language",
      "weight": 20,
      "description": "Language of the job posting and required communication language",
      "scoring": {
        "english_italian_spanish": 100,
        "other_language_but_english_required": 70,
        "other_language_only": 30
      }
    },
    {
      "id": "freshness",
      "weight": 20,
      "description": "How recently the job was posted",
      "scoring": {
        "less_than_6h": 100,
        "less_than_24h": 80,
        "less_than_48h": 60,
        "less_than_7d": 30,
        "older": 10
      }
    }
  ],
  "bonus_malus": [
    {
      "id": "specific_tech_bonus",
      "value": 10,
      "description": "+10 if mentions specific tools candidate knows well: SPM, Jitpack, Fastlane, CI/CD, MVVM, Clean Architecture"
    },
    {
      "id": "industry_bonus",
      "value": 5,
      "description": "+5 if in streaming/media or healthcare/pharma (past experience)"
    },
    {
      "id": "unknown_tech_malus",
      "value": -10,
      "description": "-10 if requires years of experience in tech candidate doesn't know (e.g., Xamarin, .NET MAUI, Ionic)"
    }
  ],
  "notification_threshold": 50,
  "max_notifications_per_run": 10,
  "daily_summary_hour_cet": 22
}
```

- [ ] **Step 2: Validate JSON is well-formed**

```bash
python3 -c "import json; json.load(open('config/scoring_rules.json')); print('OK')"
```

Expected: `OK`

- [ ] **Step 3: Commit**

```bash
git add config/scoring_rules.json
git commit -m "feat: add scoring rules with elimination filters and weighted criteria"
```

---

### Task 5: Create initial state file

**Files:**
- Create: `data/seen_jobs.json`

- [ ] **Step 1: Create data directory**

```bash
mkdir -p data
```

- [ ] **Step 2: Create data/seen_jobs.json**

```json
{
  "last_updated": null,
  "jobs": {},
  "daily_stats": {},
  "site_errors": {}
}
```

- [ ] **Step 3: Validate JSON**

```bash
python3 -c "import json; json.load(open('data/seen_jobs.json')); print('OK')"
```

Expected: `OK`

- [ ] **Step 4: Commit**

```bash
git add data/seen_jobs.json
git commit -m "feat: add initial empty state file for job deduplication"
```

---

### Task 6: Create the routine prompt

This is the core of the project — the prompt that Claude Code Routine executes each run.

**Files:**
- Create: `routine/prompt.md`

- [ ] **Step 1: Create routine directory**

```bash
mkdir -p routine
```

- [ ] **Step 2: Create routine/prompt.md**

````markdown
# Job Monitor Routine

You are a job monitoring agent. Your task is to find new mobile developer job listings across EU job sites and notify Stefano Russello via Telegram.

## Step 1: Load Configuration

Read these files from the repository:
- `config/sites.json` — sites to monitor
- `config/profile.json` — candidate profile and preferences
- `config/scoring_rules.json` — scoring weights and filters
- `data/seen_jobs.json` — previously seen jobs (for deduplication)

## Step 2: Fetch Job Listings

For each **enabled** site in `sites.json`, fetch job listings using the appropriate method:

### RSS sites (type: "rss")
For each keyword in the site's `keywords` array:
```bash
curl -s -L --max-time 30 "{base_url}?{param}={keyword}" -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
```
Parse the XML response. Extract from each `<item>`: `<title>`, `<link>`, `<description>`, `<pubDate>`.

### JSON API sites (type: "json_api")
```bash
curl -s -L --max-time 30 "{base_url}?{params}" -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
```
Parse the JSON response. For RemoteOK: filter entries where `tags` contain any of the keywords. For Remotive: filter by keyword match in `title` or `description`.

### HTML sites (type: "html")
For each keyword in the site's `keywords` array:
```bash
curl -s -L --max-time 30 "{base_url}?{param}={keyword}" -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)" -H "Accept: text/html"
```
Use your understanding of the HTML to extract job listings. Look for repeated patterns containing: job title, company name, location, and a link to the full job posting. Do not rely on specific CSS selectors — use semantic understanding.

### Error handling per site
- If a site returns an error, times out, or returns no parseable content: log it and continue with the next site.
- Track consecutive failures in `site_errors` in `seen_jobs.json`:
  ```json
  "site_errors": {
    "glassdoor": { "consecutive_failures": 3, "last_error": "403 Forbidden", "last_success": "2026-04-17T14:00:00Z" }
  }
  ```
- If consecutive failures reach 3, include a warning in the Telegram notification.
- Reset counter to 0 on successful fetch.

## Step 3: Deduplicate

For each job found, generate an ID: `{site_id}_{md5_hash_of_url}` or `{site_id}_{native_id}` if the site provides one.

Check against `seen_jobs.json`. If the ID already exists, skip it entirely.

## Step 4: Apply Elimination Filters

For each new (unseen) job, read the full job description. If needed, fetch the detail page:
```bash
curl -s -L --max-time 30 "{job_url}" -H "User-Agent: Mozilla/5.0 (Macintosh; Intel Mac OS X 10_15_7)"
```

Apply elimination filters from `scoring_rules.json`:

### Filter: remote_only
- Check for `reject_indicators` in the job text. If found → **ELIMINATE**.
- Check for `accept_indicators`. If found → **PASS**.
- If ambiguous (neither found) → **PASS** (do not eliminate, but flag as ambiguous).

### Filter: freelance_only
- Check for `reject_indicators` in the job text. If found → **ELIMINATE**.
- Check for `accept_indicators`. If found → **PASS**.
- If ambiguous → **PASS** (do not eliminate, but flag as ambiguous).

Add eliminated jobs to `seen_jobs.json` with `"eliminated": true` to avoid re-processing.

## Step 5: Score Remaining Jobs

For each job that passes elimination filters, calculate a score (0–100):

1. **tech_match (35%)**: Compare required technologies in the job with `core_technologies` in profile.json.
   - Mentions Swift/Kotlin/ObjC/Java → 100
   - Mentions Flutter/React Native → 80
   - Generic "mobile developer" → 60
   - Partial match → 40

2. **seniority (25%)**: Determine seniority level from job title and description.
   - Junior → 100
   - Not specified → 85
   - Mid → 70
   - Senior → 30
   - Lead/Staff/Principal → 10

3. **language (20%)**: What language is the posting in? What language is required for work?
   - English, Italian, or Spanish → 100
   - Other language but English required for work → 70
   - Other language only → 30

4. **freshness (20%)**: How old is the posting?
   - < 6 hours → 100
   - < 24 hours → 80
   - < 48 hours → 60
   - < 7 days → 30
   - Older → 10

5. **Apply bonus/malus:**
   - +10 if mentions SPM, Jitpack, Fastlane, CI/CD, MVVM, Clean Architecture
   - +5 if streaming/media or healthcare/pharma industry
   - -10 if requires years in unknown tech (Xamarin, .NET MAUI, Ionic, etc.)

**Final score** = weighted sum + bonuses, capped at 0-100.

## Step 6: Notify via Telegram

Use the Telegram Bot API. The bot token is in the environment variable `TELEGRAM_BOT_TOKEN`. The chat ID is `181648234`.

### For each job with score ≥ 50 (max 10 per run, ordered by score descending):

```bash
curl -s -X POST "https://api.telegram.org/bot${TELEGRAM_BOT_TOKEN}/sendMessage" \
  -H "Content-Type: application/json" \
  -d '{
    "chat_id": "181648234",
    "parse_mode": "HTML",
    "text": "<message>"
  }'
```

**Message format for score ≥ 75:**
```
🟢 Score: {score}/100 | {title}
🏢 {company} — {country}
💼 {contract_type}
🔧 {matched_technologies}
📅 {time_ago} — via {site_name}

✅ Perché matcha:
• {reason_1}
• {reason_2}
• {reason_3}

🔗 {url}
```

**Message format for score 50–74:**
```
🟡 Score: {score}/100 | {title}
🏢 {company} — {country}
💼 {contract_type_or_warning}
🔧 {matched_technologies}
📅 {time_ago} — via {site_name}

✅ Match: {brief_reasons}
{optional_warning_if_ambiguous}

🔗 {url}
```

If there are more than 10 matching jobs, after the top 10 send:
```
📋 E altri {N} annunci trovati con score ≥ 50. I top 10 sono stati notificati sopra.
```

### Site error warnings (if any site has 3+ consecutive failures):
```
⚠️ Attenzione: {site_name} non risponde da {hours} ore
Ultimo successo: {last_success}
Errore: {last_error}
Gli altri siti funzionano regolarmente.
```

### Daily summary (only on the 22:00 CET run):

Check the current time. If it is between 21:30 and 22:30 CET, send a daily summary:

```
📊 Riepilogo giornaliero — {date}

Oggi trovati: {total} annunci rilevanti
• 🟢 Score alto (≥75): {count}
• 🟡 Score medio (50-74): {count}
• 🟠 Ambigui (contratto non chiaro): {count}

Top 3 del giorno:
1. 🟢 {score} — {title} → {site}
2. 🟢 {score} — {title} → {site}
3. 🟢 {score} — {title} → {site}

Siti monitorati: {ok_count}/{total_count} ✅ {warning_if_failures}

Totale questa settimana: {weekly_total} annunci notificati
```

If no jobs were found today, send:
```
📊 Riepilogo giornaliero — {date}

Oggi: nessun nuovo annuncio rilevante trovato.
Siti monitorati: {ok_count}/{total_count} ✅
Il monitoraggio continua domani alle 08:00.
```

## Step 7: Update State

Update `seen_jobs.json`:
- Add all new jobs (notified, eliminated, and below-threshold) with their metadata
- Update `last_updated` timestamp
- Update `daily_stats` for today
- Reset `site_errors` counters for sites that succeeded
- Remove jobs older than 30 days
- Remove daily_stats older than 90 days

## Step 8: Commit and Push

```bash
git add data/seen_jobs.json
git commit -m "chore: update seen_jobs [$(date -u +%Y-%m-%dT%H:%M)]"
git push origin main
```

## Important Notes

- **Rate limiting**: Wait 1-2 seconds between requests to different sites. Do not hammer any single site.
- **User-Agent**: Always include a realistic browser User-Agent header.
- **Timeout**: Use --max-time 30 for all curl requests. Do not wait indefinitely.
- **Language**: All Telegram messages are in Italian (notification language preference).
- **Dedup is critical**: NEVER send the same job twice. Always check seen_jobs.json first.
- **Fail gracefully**: If a site fails, log and continue. Never let one site failure stop the entire run.
- **Only notify if there are new, qualifying jobs**: If a run finds nothing new, do nothing (except daily summary if it's 22:00).
````

- [ ] **Step 3: Commit**

```bash
git add routine/prompt.md
git commit -m "feat: add main routine prompt for job monitoring agent"
```

---

### Task 7: Create CLAUDE.md for the repository

This file provides Claude Code with project-level instructions when running against this repo.

**Files:**
- Create: `CLAUDE.md`

- [ ] **Step 1: Create CLAUDE.md**

```markdown
# Job Monitor — Claude Code Instructions

This repository is used by a Claude Code Routine that monitors EU job sites for mobile developer positions.

## When running as a Routine

Follow the instructions in `routine/prompt.md` exactly. This is your playbook for each run.

## Key files

- `config/sites.json` — Sites and keywords (read-only during routine execution)
- `config/profile.json` — Candidate profile (read-only during routine execution)
- `config/scoring_rules.json` — Scoring weights and filters (read-only during routine execution)
- `data/seen_jobs.json` — State file (read + write during routine execution)
- `routine/prompt.md` — The routine instructions

## Environment variables required

- `TELEGRAM_BOT_TOKEN` — Telegram Bot API token (set as GitHub Secret)

## Rules

- Do not modify config files during routine execution
- Always commit seen_jobs.json changes after each run
- Never notify the same job twice
- If a site fails, continue with others — never abort the entire run
- All Telegram messages must be in Italian
```

- [ ] **Step 2: Commit**

```bash
git add CLAUDE.md
git commit -m "feat: add CLAUDE.md with routine instructions for Claude Code"
```

---

### Task 8: Set up GitHub Secrets

**Files:** None (GitHub API operations)

- [ ] **Step 1: Set TELEGRAM_BOT_TOKEN secret**

```bash
gh secret set TELEGRAM_BOT_TOKEN --repo Spettacolo83/job-monitor
```

When prompted, enter the bot token value.

- [ ] **Step 2: Set TELEGRAM_CHAT_ID secret**

```bash
gh secret set TELEGRAM_CHAT_ID --repo Spettacolo83/job-monitor --body "181648234"
```

- [ ] **Step 3: Verify secrets are set**

```bash
gh secret list --repo Spettacolo83/job-monitor
```

Expected output should list both `TELEGRAM_BOT_TOKEN` and `TELEGRAM_CHAT_ID`.

---

### Task 9: Push to GitHub

- [ ] **Step 1: Push all commits to remote**

```bash
git push -u origin main
```

Expected: all commits pushed successfully.

- [ ] **Step 2: Verify repo contents on GitHub**

```bash
gh repo view Spettacolo83/job-monitor --web
```

Verify the repo has: `config/`, `data/`, `routine/`, `CLAUDE.md`, `README.md`, `.gitignore`.

---

### Task 10: Configure Claude Code Routine

This task is done manually on claude.ai/code (or via CLI if available).

- [ ] **Step 1: Go to claude.ai/code and create a new Routine**

Configuration:
- **Name**: Job Monitor
- **Repository**: Spettacolo83/job-monitor
- **Schedule**: Hourly (adjust later based on notification volume)
- **Prompt**: "Read and follow the instructions in `routine/prompt.md`. Execute all steps."

- [ ] **Step 2: Set environment variables in the Routine configuration**

Add:
- `TELEGRAM_BOT_TOKEN` = (the bot token)

- [ ] **Step 3: Run the Routine manually for the first time**

Trigger a test run and verify:
- Jobs are fetched from at least some sites
- Telegram notification arrives
- `seen_jobs.json` is updated and committed

---

### Task 11: Verify end-to-end

- [ ] **Step 1: Check Telegram for notifications**

After the test run, verify at least one notification arrived in the Telegram chat.

- [ ] **Step 2: Check seen_jobs.json was updated**

```bash
git pull origin main
python3 -c "import json; data=json.load(open('data/seen_jobs.json')); print(f'Jobs tracked: {len(data[\"jobs\"])}')"
```

Expected: at least 1 job tracked.

- [ ] **Step 3: Run a second time and verify no duplicates**

Trigger the routine again. Verify that previously notified jobs are NOT sent again.

- [ ] **Step 4: Commit verification notes**

If everything works, the system is live. Adjust schedule frequency on claude.ai/code as needed based on notification volume.
