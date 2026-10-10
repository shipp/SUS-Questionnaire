# SUS Questionnaire

Online questionnaire for the System Usability Scale (SUS), using the all-positive version (Sauro & Lewis, 2011). Participants answer anonymously in German or English, without an account. The admin view starts and stops runs, shows incoming responses live and exports the answers as CSV.

Questionnaire-questions are in [docs/sus-questions-and-scale.md](docs/sus-questions-and-scale.md)

## Status

Planning. The specification is in [sus-questionnaire-spec-v2.md](sus-questionnaire-spec-v2.md).

## Planned setup

- **Frontend:** a single static `index.html`, hosted on GitHub Pages.
  - `/` – participant questionnaire
  - `/#admin` – admin view (login required)
- **Backend:** Supabase (EU region): a Postgres database with Row Level Security, and Auth for the admin login.

## Planned files

| File | Purpose |
|---|---|
| `index.html` | Participant and admin view |
| `schema.sql` | Database tables, access rules, start/stop functions, default questions |
| `docs/SETUP.md` | Step-by-step setup (Supabase, GitHub Pages) |
| `docs/spec-v2.md` | Specification |
