# QMe API (pilot v1)

The public API for QMe (QuoteMe): create quotes from your own system, keep your catalogue in step, and send quotes to customers.

* **[docs/quickstart.md](docs/quickstart.md)** — make a key and create your first quote in ten minutes.
* **[docs/openapi.yaml](docs/openapi.yaml)** — the full reference (OpenAPI 3.0.3).
* [docs/step-1-notes.md](docs/step-1-notes.md) — what the existing app already did before the API.

## Where the code lives

The API is built inside the main QMe repository (`QME`, branch `api-v1`, off `pilot-v1`), because it shares the quote maths, the database and the
deployment with the app. Nothing in the app was changed to make it work; the API only adds.

| What | Where in `QME` |
|---|---|
| The `v1` Edge Function | `supabase/functions/v1/` and `supabase/functions/_shared/api/` |
| Database (migrations 208 to 211) | `supabase/migration_208…211_*.sql` (+ rollbacks in `supabase/rollbacks/`) |
| API Keys page | `apps/web/src/pages/api-keys/` |
| Tests | `apps/web/tests/{unit,db,e2e}/api-*` |

The two documents here are copies of `docs/api/` in `QME`; `QME` is the source of truth.

Staging Supabase project: `bjodkoppwjyoyabolchf` (Qme_Staging_Pilot). Production is not touched from here.
