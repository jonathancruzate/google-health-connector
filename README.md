# `google-health-connector` — Write to Google Health / Health Connect via the Google Health API

A working write path into **Google Health** (the app formerly known as Fitbit) built on the
**Google Health API v4**, using the open-source `ghealth` CLI. Writes sleep, weight, and body
fat into a Google Health account — from where the phone app exports them onward to
**Health Connect**, tagged as manually entered.

Built because the legacy route is gone: Fitbit Web API support ended **September 30, 2026**,
new developer-app registration is disabled, and the legacy API and OAuth endpoints shut off
**November 2, 2026**. This is the migration path that still works after that date.

---

## Why

Health Connect has no general-purpose write API for third parties, and most integrations
are read-only. The Google Health API is the exception: several data types are writable
(`create` / `update` / `delete`), and entries created through the API are first-class —
verified by static analysis of the current Google Health Android app, whose Health Connect
export pipeline converts API/manual entries with recording method `MANUALLY_ENTERED` and
does not filter them out.

## What it does

- **Write sleep sessions** (interval + metadata), **weight**, and **body fat** via `ghealth`
- **Read everything back** through the same lane (sleep history, identity, data types)
- Provenance discipline for estimated data: estimates are written only when no observed
  session exists, always tagged (`metadata.externalId = "muse-estimate-<date>"`,
  `manuallyEdited: true`), and deleted/reconciled the moment observed data lands

## Architecture

```
ghealth CLI ──OAuth 2.0 (PKCE)──▶ Google Health API v4 ──▶ Google Health account
                                                                    │
                       Google Health phone app (Health Connect sync) ▼
                                                              Health Connect
```

Reads flow the other way too: tracker apps that write to Health Connect (e.g. Sleep as
Android) surface in the Google Health account and become readable through this API — a
useful backup read path when direct Health Connect access is unavailable.

## Setup

1. **Google Cloud project** — create one, enable the **Google Health API**
   (`health.googleapis.com`).
2. **OAuth consent screen** — External; add yourself as a test user while in Testing.
   Select only the scopes you need (this connector uses five):
   - `googlehealth.sleep.readonly` / `googlehealth.sleep.writeonly`
   - `googlehealth.health_metrics_and_measurements.readonly` / `.writeonly`
   - `googlehealth.profile.readonly`
3. **OAuth client** — type **Desktop app**; download the client-secret JSON to
   `~/.config/ghealth/client_secret.json` (mode `0600`).
4. **Link the account first** — sign in to the Google Health phone app and link it to the
   Google Account before consenting, or the identity call returns `400 ACCOUNT_NOT_LINKED`.
5. **Authorize headlessly** (PKCE, no browser needed on the host):
   ```bash
   ghealth setup --project-id <project> --client-secret ~/.config/ghealth/client_secret.json \
     --scopes sleep.readonly,sleep.writeonly,health_metrics_and_measurements.readonly,health_metrics_and_measurements.writeonly,profile.readonly \
     --skip-enable-api --no-prompt --non-interactive-auth
   ghealth auth login --non-interactive --scopes <same list>   # prints an auth_url
   # open auth_url anywhere, consent, copy the `code` param from the failed localhost redirect
   ghealth auth login --complete <code>
   ghealth user identity   # verify: returns your healthUserId
   ```

## The Testing-mode trap (read this before you ship)

An OAuth app left in **Testing** publishing status gets refresh tokens that **expire after
7 days** — the connector silently dies a week after setup. Publish the app to production:
the Branding page requires an app name, support email, a **home page URL**, and a
**privacy policy URL** on an authorized domain before the Publish button enables. For a
personal, single-user app no verification submission is needed; the "requires verification"
banner (shown because the health scopes are Restricted) does not block your own use.
Re-run the consent once after publishing so the stored refresh token is production-issued.

## Payload notes (learned live)

Sleep create bodies wrap the fields in a type-named object, offsets are protobuf
Durations, and civil times are structured objects (omit them — UTC + offset is enough):

```bash
ghealth data sleep create --json '{
  "sleep": {
    "interval": {
      "startTime": "2026-01-01T05:45:00Z",
      "endTime":   "2026-01-01T15:00:00Z",
      "startUtcOffset": "-14400s",
      "endUtcOffset":   "-14400s"
    },
    "metadata": {"mainSleep": true, "manuallyEdited": true, "externalId": "example-1"}
  }}'
```

- Forgetting the `"sleep"` wrapper → `400 Unknown name "interval" at 'data_point'`
- Offsets like `"-04:00"` → rejected; durations must end in `s`
- Writes are asynchronous Operations; success returns `"done": true` with the new
  data point's name. Deletes: `ghealth data sleep delete --ids <id>`

## Verification

End-to-end verified 2026-10-24: identity read, a real sleep create (tagged, appeared in
list reads), and a clean delete. The same pattern covers weight and body fat
(`ghealth data weight` / `ghealth data body-fat`).

Full operating notes: [CONNECTOR.md](CONNECTOR.md). Worked payload: [examples/sleep-create.json](examples/sleep-create.json).

## Built with

[ghealth](https://github.com/Google-Health-API/google-health-cli) — the open-source,
agent-oriented CLI for the Google Health API (Apache 2.0) — plus the connector
configuration and operating procedure in this repo.
