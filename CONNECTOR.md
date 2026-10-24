# Google Health API — connector operating notes

Reference for running the Google Health write lane via the `ghealth` CLI. Generalized
from a live deployment (verified 2026-10-24); replace `<placeholders>` with your own
values.

## Background

- The legacy Fitbit Web API is closed to new apps: registration is disabled, support ended
  Sep 30, 2026, and the legacy API + OAuth endpoints shut off Nov 2, 2026.
- Successor: the **Google Health API** (`developers.google.com/health`). The phone app is
  the same product — the Fitbit app was rebranded in place to Google Health in May 2026
  (package still `com.fitbit.FitbitMobile`).
- Muse's Health Connect connector is read + delete only. This lane exists for **writes**.

## Configuration

- GCP project with the **Google Health API** (`health.googleapis.com`) enabled.
- OAuth client type **Desktop app**; client-secret JSON at
  `~/.config/ghealth/client_secret.json` (`0600`). Tokens land in
  `~/.config/ghealth/credentials.json` (`0600`); profile in `~/.config/ghealth/config.toml`.
- Scopes (least privilege — five): `googlehealth.sleep.readonly`,
  `googlehealth.sleep.writeonly`,
  `googlehealth.health_metrics_and_measurements.readonly`,
  `googlehealth.health_metrics_and_measurements.writeonly`,
  `googlehealth.profile.readonly`.
- CLI: build `ghealth` from `github.com/Google-Health-API/google-health-cli` (Go).

## Auth lifecycle

- **Publish the consent screen to production.** Testing-mode refresh tokens expire after
  7 days. Publishing requires Branding: app name, support email, home page URL, and
  privacy policy URL on an authorized domain. The app stays **unverified** (Restricted
  scopes) — the console banner is advisory for single-user personal use; do not submit
  verification just to silence it.
- Re-run consent after publishing so the stored refresh token is production-issued.
- Headless re-auth (no local browser needed): `ghealth auth login --non-interactive
  --scopes <list>` → open the printed `auth_url` on any device → copy the `code` query
  parameter from the localhost redirect that fails to load → `ghealth auth login
  --complete <code>`. Access tokens last 1 h; the CLI refreshes automatically.
- Prerequisite: the user must have signed in to the Google Health mobile app and linked
  their Google Account first, or API calls return `400 ACCOUNT_NOT_LINKED`. Verify with
  `ghealth user identity`.

## Operating notes

- **Latency:** expect slow first calls (tens of seconds); calls warm up after the first
  success. A `context deadline exceeded` on a cold start is worth one retry before
  concluding anything is down.
- **Sleep create:** body wraps fields in a top-level `sleep` object.
  `interval.startUtcOffset` / `endUtcOffset` are protobuf Durations (`"-14400s"`).
  Civil start/end times are structured `CivilDateTime` objects — omit them.
  See `examples/sleep-create.json`.
- **Writes are async Operations**; `"done": true` + the new data point name confirms.
- **Delete:** `ghealth data sleep delete --ids <id[,id...]>`.
- **Why API entries reach Health Connect:** static analysis of the Google Health Android
  app shows its export pulls sleep/weight from its cloud API and converts every session —
  `logType manual` → Health Connect recording method `MANUALLY_ENTERED` — with no source
  filter in the fetch→convert path. Sync triggers: app launch + a WorkManager sync worker.
- **Provenance for estimated data:** tag with `metadata.externalId =
  "muse-estimate-<night_date>"` and `manuallyEdited: true`. Write an estimate only when
  no observed session exists for that night; when observed data lands, delete the API
  point and reconcile your own records. Never let an estimate and an observed session
  coexist for the same night.

## Boundaries

- This lane writes to the Google Health account; it does not replace any existing
  Health Connect read/sync pipeline — treat it as the write path plus an alternate
  read path.
- Credentials never leave `~/.config/ghealth/`; never commit tokens, client secrets,
  project IDs, or user IDs to a repo.
