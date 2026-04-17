# Job Monitor — Design Specification

**Data**: 2026-04-17
**Autore**: Stefano Russello + Claude
**Stato**: Approvato

## Obiettivo

Monitorare automaticamente i principali siti europei di lavoro per posizioni di sviluppo mobile (iOS, Android, Flutter, React Native) e notificare via Telegram gli annunci rilevanti, con l'obiettivo di rispondere per primi agli annunci più in linea con il profilo.

## Architettura

### Approccio: Claude Code Routine puro

Una singola Claude Code Routine eseguita nel cloud Anthropic su cron orario. La Routine legge la configurazione e lo stato da un repository GitHub, esegue il fetching degli annunci, li filtra e li valuta, notifica via Telegram e aggiorna lo stato nel repo.

```
┌─────────────────────────────────────────────────┐
│           Claude Code Routine (oraria)           │
│              Cloud Anthropic - Max               │
│                                                   │
│  1. Legge config dal repo GitHub                  │
│     └── sites.json (siti + query di ricerca)      │
│     └── profile.json (profilo/criteri)            │
│     └── seen_jobs.json (annunci già notificati)   │
│                                                   │
│  2. Per ogni sito:                                │
│     ├── API/RSS → fetch diretto                   │
│     └── HTML → fetch + parsing intelligente       │
│                                                   │
│  3. Filtra eliminatori:                           │
│     ├── Non full remote? → SCARTA                 │
│     └── Solo assunzione? → SCARTA                 │
│                                                   │
│  4. Scoring (0-100) sugli annunci rimanenti       │
│                                                   │
│  5. Per ogni annuncio con score ≥ 50%:            │
│     └── Notifica Telegram (singolo annuncio)      │
│                                                   │
│  6. Alle 22:00 → riepilogo giornaliero Telegram   │
│                                                   │
│  7. Aggiorna seen_jobs.json → commit + push       │
└─────────────────────────────────────────────────┘
```

### Repository GitHub

```
Spettacolo83/job-monitor
├── config/
│   ├── sites.json            # Siti da monitorare + parametri ricerca
│   ├── profile.json          # Profilo, criteri, pesi scoring
│   └── scoring_rules.json    # Regole scoring dettagliate
├── data/
│   └── seen_jobs.json        # ID annunci già processati (dedup)
├── routine/
│   └── prompt.md             # Prompt della Routine
└── README.md
```

### GitHub Secrets

| Secret | Uso |
|---|---|
| `TELEGRAM_BOT_TOKEN` | Invio notifiche Telegram |
| `TELEGRAM_CHAT_ID` | Chat ID destinatario (181648234) |

## Siti monitorati

### Siti con API/RSS (affidabili)

| Sito | Metodo | Endpoint |
|---|---|---|
| Indeed EU | RSS Feed | `https://{country}.indeed.com/rss?q={keyword}` (varianti: it, co.uk, de, es, fr) |
| RemoteOK | JSON API | `https://remoteok.com/api?tag=mobile` |
| Remotive | JSON API | `https://remotive.com/api/remote-jobs?category=software-dev` |
| Welcome to the Jungle | API pubblica | `https://www.welcometothejungle.com/api/v1/jobs?query={keyword}` |
| Jobicy | RSS Feed | `https://jobicy.com/feed/newjob?search={keyword}` |

### Siti con scraping HTML

| Sito | Metodo | Note |
|---|---|---|
| InfoJobs | HTML scraping | IT/ES |
| StepStone | HTML scraping | DE/NL/BE |
| Glassdoor | HTML scraping | Anti-bot aggressivo, può fallire |
| Monster | HTML scraping | EU |
| Honeypot | HTML scraping | Tech-focused EU |
| Landing.Jobs | HTML scraping | EU tech |

## Strategia keyword

### Principio

Keyword ampie per massima copertura → filtro intelligente di Claude per scartare i non rilevanti.

### Tier di keyword

**Tier 1 — Tecnologie (massima copertura):**
```
iOS, Swift, SwiftUI, Android, Kotlin,
Jetpack Compose, Flutter, React Native, Mobile
```

**Tier 2 — Ruoli (se il sito richiede combinazione):**
```
developer, engineer, sviluppatore, desarrollador
```

**Tier 3 — Combinazioni (solo se il sito richiede query più specifiche):**
```
iOS Swift, Android Kotlin, Flutter developer,
Mobile engineer, app developer
```

### Logica per sito

- Siti con buon motore di ricerca → query Tier 1 singole
- Siti con ricerca basica (RSS, API semplici) → Tier 1 + Tier 2
- Siti con filtri per categoria → categoria "mobile/software-dev" + keyword Tier 1

### Cosa NON includiamo nelle query

- remote / remoto → verificato da Claude nel testo
- freelance / contract → verificato da Claude nel testo
- junior / mid → verificato da Claude nel testo

## Sistema di scoring

### Fase 1 — Filtri eliminatori

| Filtro | Logica |
|---|---|
| Non full remote | Qualsiasi obbligo in sede/ibrido → SCARTA |
| Solo assunzione | Contratto dipendente senza opzione freelance → SCARTA |

Claude riconosce indicatori multilingua:
- IT: "tempo indeterminato", "assunzione diretta", "RAL"
- ES: "contrato indefinido", "nómina", "convenio"
- EN: "permanent position", "full-time employee", "W-2"
- DE: "Festanstellung", "unbefristeter Vertrag"
- FR: "CDI", "contrat salarié"

**Caso ambiguo**: se il tipo contratto non è specificato → NON scartare, score penalizzato ma notificato.

### Fase 2 — Scoring (0–100)

| Criterio | Peso | Punteggio |
|---|---|---|
| Match tecnologie | 35% | Swift/Kotlin/ObjC/Java = 100%, Flutter/RN = 80%, "mobile generico" = 60% |
| Seniority | 25% | Junior = 100%, Mid = 70%, non specificato = 85%, Senior = 30% |
| Lingua annuncio | 20% | EN/IT/ES = 100%, DE/FR con requisito EN = 70%, solo DE/FR = 30% |
| Freschezza | 20% | < 6h = 100%, < 24h = 80%, < 48h = 60%, < 7gg = 30% |

### Bonus/malus dal profilo

- **+10%** se menziona tech con esperienza diretta (SDK, SPM, Jitpack, CI/CD, Fastlane)
- **+5%** se settore simile a esperienze passate (streaming/media, healthcare/pharma)
- **-10%** se richiede tech non conosciute (es. "5 anni Xamarin obbligatori")

### Soglia notifica: ≥ 50/100

### Ordinamento

Annunci notificati in ordine decrescente di score. Se >10 annunci in una run → top 10 + messaggio "e altri X trovati".

## Formato notifiche Telegram

### Notifica singolo annuncio

```
🟢 Score: 85/100 | iOS Developer
🏢 Acme Corp — Germania
💼 Freelance/B2B
🔧 Swift, SwiftUI, REST APIs, CI/CD
📅 3h fa — via Indeed

✅ Perché matcha:
• Tech stack core (Swift/SwiftUI) +35
• Junior level +25
• Annuncio in inglese +20
• Pubblicato < 6h fa +20
• Bonus: menziona CI/CD e SPM +10

🔗 https://indeed.com/job/12345
```

### Notifica annuncio ambiguo

```
🟡 Score: 62/100 | Android Kotlin — Progetto 6 mesi
🏢 StartupXYZ — Spagna
💼 ⚠️ Tipo contratto non specificato
🔧 Kotlin, Jetpack Compose
📅 1h fa — via Welcome to the Jungle

✅ Match: tech Kotlin +35, junior +25
⚠️ Nota: non specifica se freelance o assunzione.
   Potrebbe valere la pena verificare.

🔗 https://welcometothejungle.com/job/xyz
```

### Riepilogo serale (22:00)

```
📊 Riepilogo giornaliero — 17 Apr 2026

Oggi trovati: 8 annunci rilevanti
• 🟢 Score alto (≥75): 3
• 🟡 Score medio (50-74): 4
• 🟠 Ambigui (contratto non chiaro): 1

Top 3 del giorno:
1. 🟢 92 — Senior Flutter Dev → RemoteOK
2. 🟢 85 — iOS Developer → Indeed
3. 🟢 78 — Mobile Engineer → Remotive

Siti monitorati: 11/11 ✅
Prossimo controllo: domani 08:00

Totale questa settimana: 23 annunci notificati
```

### Notifica errore sito

```
⚠️ Attenzione: InfoJobs non risponde da 3 ore
Ultimo successo: 17 Apr 14:00
Errore: timeout / 403 Forbidden
Gli altri 10 siti funzionano regolarmente.
```

## Gestione stato e deduplicazione

### Struttura `seen_jobs.json`

```json
{
  "last_updated": "2026-04-17T15:00:00Z",
  "jobs": {
    "indeed_abc123": {
      "title": "iOS Developer",
      "url": "https://indeed.com/job/abc123",
      "score": 85,
      "first_seen": "2026-04-17T14:00:00Z",
      "source": "indeed"
    }
  },
  "daily_stats": {
    "2026-04-17": {
      "total_found": 8,
      "notified": 5,
      "filtered_out": 12
    }
  }
}
```

### Generazione ID

Ogni annuncio: `{sito}_{hash(url)}` oppure `{sito}_{id_nativo}` se disponibile.

### Ciclo di vita annuncio

```
Nuovo annuncio trovato
  → È in seen_jobs.json?
     ├── SÌ → ignora
     └── NO → filtra eliminatori
              → calcola score
              → score ≥ 50?
                 ├── SÌ → notifica Telegram + aggiungi a seen_jobs
                 └── NO → aggiungi a seen_jobs (per non riprocessarlo)
```

### Pulizia automatica

- Annunci > 30 giorni rimossi da `seen_jobs.json`
- `daily_stats` mantenute per 90 giorni

### Persistenza

Dopo ogni run: aggiorna `seen_jobs.json` → commit + push con messaggio `chore: update seen_jobs [timestamp]`.

## Schedule

### Configurazione Routine

- **Trigger**: cron orario (modificabile da claude.ai/code senza toccare il repo)
- **Orario consigliato**: 08:00–22:00 CET = 15 esecuzioni/giorno (limite Max plan)
- **Run 22:00**: include riepilogo giornaliero

### Piano Max

15 routines/giorno. 15 ore di copertura oraria. Modificabile dall'utente in qualsiasi momento.

## Gestione errori

| Errore | Strategia |
|---|---|
| Sito non raggiungibile (timeout/5xx) | Log, skip, continua. Dopo 3 consecutivi → notifica Telegram |
| Anti-bot / 403 | Log, skip. Notifica dopo 3 consecutivi |
| Struttura HTML cambiata | Claude tenta parsing semantico. Se fallisce → log + notifica |
| Telegram API fallisce | Retry 1 volta. Se fallisce → annunci NON aggiunti a seen_jobs, ri-notificati alla run successiva |
| GitHub push fallisce | Retry 1 volta. Run successiva riprocessa (dedup cattura duplicati) |
| Nessun annuncio trovato | Silenzio. Solo nel riepilogo serale: "0 annunci" |

### Resilienza del parsing

Claude capisce il testo semanticamente → se un sito cambia layout HTML, il parsing continua a funzionare perché Claude riconosce titolo, azienda e descrizione dal contesto, non da selettori CSS.

## Profilo candidato

### Dati personali
- **Nome**: Stefano Russello
- **Residenza**: Petra, Mallorca, Spagna
- **Lingue**: Italiano (nativo), Spagnolo (fluente), Inglese (fluente)
- **Contatto**: stefano.russello@gmail.com / +34 633666065
- **Web**: https://www.stefanorussello.it
- **LinkedIn**: https://www.linkedin.com/in/stefano-russello-579a4471/
- **GitHub**: https://github.com/Spettacolo83/

### Esperienza
- 10+ anni sviluppo mobile nativo iOS + Android
- StreamAMG (Malta/Remoto) 2018–2025: app native, SDK interni, CI/CD
- Roche (Svizzera/Remoto) 2025: app chat nativa, REST APIs

### Tech stack
- **Android**: Java, Kotlin, Jetpack Compose, Retrofit, Coroutines, Flow, Firebase, JUnit
- **iOS**: Objective-C, Swift, SwiftUI, SPM, Combine, UIKit, Alamofire, Core Data, XCTest
- **Cross-platform**: Flutter, React Native (in apprendimento)
- **Backend**: PHP, MySQL, Firebase, REST APIs, AWS, GCP, Azure, JWT
- **Tools**: Git, GitHub, Fastlane, CI/CD, Agile, MVVM, Clean Architecture, SOLID, Jira, Figma

### Preferenze
- Solo full remote
- Solo freelance/B2B/partita IVA
- Target seniority: Junior (priorità), Mid
- Disponibile in tutta l'EU

## Fase 2 (futura, design separato)

Sistema automatico di invio candidature. Verrà progettato come progetto indipendente dopo che la Fase 1 è operativa e validata.
