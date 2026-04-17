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
