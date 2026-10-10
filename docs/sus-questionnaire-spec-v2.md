# SUS Questionnaire – Spec v2 (draft 2026-10-10)

## Architecture
- **Frontend:** one static HTML file (inline CSS/JS) on **GitHub Pages**, e.g. `https://<user>.github.io/sus/`.
  - `/` → participant questionnaire
  - `/#admin` → admin login, then the admin view
- **Backend:** **Supabase**, project in an EU region (Frankfurt), free tier.
  - Postgres stores the data. Row Level Security (RLS) controls who may do what.
  - Supabase Auth handles the admin login (email + password). Public sign-ups are disabled.
  - Realtime pushes the live response count to the admin view; polling every ~10 s as a fallback.
- The page talks to Supabase with the public "anon" key, which is safe to publish. All protection comes from the RLS rules. The service-role key is never put in the page.
## Data model (Postgres)
- `settings`: a single row (id = 1)
  - `active_run_id` (uuid, null when no run is active)
  - `questions` (jsonb `{de: [10 texts], en: [10 texts]}`)
- `runs`: `id` (uuid), `name`, `started_at`, `stopped_at` (null while running), `questions` (jsonb snapshot taken at start)
- `responses`:
  - `id` (uuid)
  - `run_id` (references runs)
  - `created_at` (server time, `default now()`; the client's clock is never used)
  - `lang` (`'de'` | `'en'`)
  - `answers` (smallint[10]: values 1–5, null = skipped)
  - A CHECK constraint enforces exactly 10 entries, each 1–5 or null.
- `admins`: `user_id` (references auth.users). Only listed users get admin rights.

## Access rules (RLS)
| Table | Anonymous visitor | Admin |
|---|---|---|
| settings | read | read + update |
| runs | read the active run only (to get its question snapshot) | everything |
| responses | **insert only**, and only when `run_id` = the current `active_run_id`; no reading | read (no edit/delete in the UI) |
| admins | none | read own row |

- An insert after a run has stopped is rejected by the database. The page then shows the "survey has ended" message (same as v1).
- Starting and stopping runs happens in a database function (RPC) that admins only can call. It updates `runs` and `settings` in one step.

## Participant view (as v1)
- The 10 all-positive SUS items (Sauro & Lewis 2011), wording provided by the owner. DE/EN toggle; the language is stored with each response.
- 5-point scale with the endpoint labels "Stimme überhaupt nicht zu (1) – Stimme voll zu (5)" / "Strongly Disagree (1) – Strongly Agree (5)".
- Items are optional (click a selected value again to clear it). Anonymous, unlimited submissions.
- After Send: thank-you screen + "New response" button.
- No active run: "No survey active" message (DE/EN). The page updates when a run starts.
- No privacy notice for now (decided 2026-10-10; to add before real public use).

Questionnaire-questions are in [docs/sus-questions-and-scale.md](docs/sus-questions-and-scale.md)
## Spam protection (light)
- Answers are accepted only during an active run; the database enforces this.
- A hidden honeypot field: if it's filled in, the submission is silently dropped.
- Minimum fill time of about 10 s (client-side check).
- No captcha, no cookies, no IP storage by the page.

## Admin view
- Email + password login (Supabase Auth) and a logout button. The session is kept in the browser.
- **Run:**
  - Start a named run (one active at a time; the question snapshot is saved with it).
  - While it runs: live counter, list of submission timestamps (newest first), QR code linking to the public page URL, "Show full screen" and "Copy link".
  - Stop the run, with an in-page confirmation.
- **Results:** past runs, a raw answers table per run, CSV download.
- **Questions:** edit the DE + EN wording of the 10 items; locked while a run is active. Item count, order and scale stay fixed.

## CSV
- One row per response with the columns `run_name;response_id;timestamp;language;q1…q10`. Timestamp in ISO format with the Europe/Berlin offset; skipped items are empty.
- Semicolon separator and a UTF-8 BOM so German Excel opens it correctly. Downloaded directly by the browser.

## Setup steps (owner)
1. Create a Supabase project in an EU region (Frankfurt).
2. In the SQL editor, run the provided `schema.sql` (tables, constraints, RLS, RPC functions, default questions).
3. Under Auth: disable public sign-ups and create the admin user (email + password). Add that user's id to `admins`.
4. Copy the project URL and the anon key into the config block at the top of `index.html`.
5. Create a GitHub repo, add `index.html` and enable GitHub Pages.
6. Test: open `/#admin` → log in → start a test run → submit from a phone via the QR code → stop the run → download the CSV.

## Deliverables (Claude)
- `index.html`: participant and admin views.
- `schema.sql`: tables, constraints, RLS policies, start/stop RPCs, seed questions.
- `SETUP.md`: step-by-step guide with screenshots-free instructions.

## Risks / notes
- Free Supabase projects may be paused after a period of inactivity. Check the current free-tier terms, and open the project before each study session.
- Public link: anyone with the URL can submit while a run is active. Keep runs short and stop them after the session.
- A privacy notice is still missing; legally required for real public studies in Germany.

## Out of scope for v2
- SUS score / evaluation (later). The owner will specify the missing-data rule ("more complex").
- Export of all runs at once; deleting runs or responses in the UI (possible directly in Supabase).
- Multiple admins via the UI (possible by adding rows to `admins`).
- Custom domain.
