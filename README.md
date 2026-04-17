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
