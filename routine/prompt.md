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
Apply in this order (reject has priority over accept):
1. Check for `reject_indicators` in the job text. If found → **ELIMINATE**.
2. Check for `reject_indirect_indicators`: these are employee-only benefits (vacation days, health insurance, pension, stock options, bonuses, etc.). If **2 or more** of these appear together → **ELIMINATE** (it's an employment position).
3. Check for `reject_salary_indicators`: salary expressed as annual gross (e.g., "RAL €40K", "annual salary", "Jahresgehalt"). Match these as phrases in context, NOT as isolated short strings (e.g., "RAL" alone in a word is NOT a match, but "RAL: €35.000-45.000" IS). If found → **ELIMINATE**.
4. Check for `accept_indicators` (freelance, contractor, daily rate, etc.). If found → **PASS**.
5. If ambiguous → **PASS** (do not eliminate, but flag as ambiguous).

**Important**: Reject always takes priority. If both reject AND accept indicators are found, REJECT.

Add eliminated jobs to `seen_jobs.json` with `"eliminated": true` to avoid re-processing.

## Step 5: Score Remaining Jobs

For each job that passes elimination filters, calculate a score (0–100):

1. **freshness (35%)**: How old is the posting? **This is the most important criterion** — the goal is to respond first.
   - < 6 hours → 100
   - < 24 hours → 70
   - < 48 hours → 40
   - < 7 days → 15
   - Older than 7 days → 5
   - Date unknown / not available → 30
   - **IMPORTANT**: If no publication date is found, use `date_unknown` (30). Never assume a job is fresh without a confirmed date.

2. **tech_match (30%)**: Compare required technologies in the job with `core_technologies` in profile.json.
   - Mentions Swift/Kotlin/ObjC/Java → 100
   - Mentions Flutter/React Native → 80
   - Generic "mobile developer" → 60
   - Partial match → 40

3. **seniority (20%)**: Determine seniority level from job title and description.
   - Junior → 100
   - Not specified → 85
   - Mid → 70
   - Senior → 30
   - Lead/Staff/Principal → 10

4. **language (15%)**: What language is the posting in? What language is required for work?
   - English, Italian, or Spanish → 100
   - Other language but English required for work → 70
   - Other language only → 30

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
