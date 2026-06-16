# SSL & Domain Expiry Watchdog (n8n)

Runs once a day and watches a list of domains for two separate failure modes: TLS certificate expiry and domain registration (WHOIS) expiry. It scores each domain by severity, opens an incident ticket for anything critical, sends an urgent Slack alert, and finishes by posting a single deduplicated daily digest to email and Slack. Repeat alerts for the same issue on the same day are suppressed using workflow static data, so on-call is not paged every 24 hours for a slow-burning problem.

## How it works

1. A Schedule Trigger fires daily at 08:00 (cron `0 8 * * *`).
2. `Load Domains` produces the watchlist. In the demo it is a static array; in production it reads from a database, Google Sheet, or a `domains.json` object in storage.
3. `Loop Over Domains` iterates the list one domain at a time.
4. For each domain, `Check TLS Cert` calls a certificate-info API and `Check WHOIS` calls a WHOIS API. Both retry on failure. If WHOIS fails, the run continues down an error branch (`WHOIS Fallback`) so the domain is still scored on cert data alone.
5. `Merge Cert + WHOIS` joins the two responses, and `Compute Severity` parses days-until-expiry for both the certificate and the registration, then assigns the worse of the two bands.
6. After the loop completes, `Dedup vs Static Data` removes findings already alerted today using `$getWorkflowStaticData('global')`.
7. `Switch Severity` routes each finding to the critical, warning, or ok path.
8. Critical findings open an incident ticket and fire an urgent Slack message. Warning findings are collected into the digest. OK findings end at a no-op.
9. `Build Digest` produces an HTML table and a plain-text summary sorted by severity, which are sent via `Send Daily Digest Email` (SendGrid) and `Post Digest to Slack`.

## Node highlights

- **Compute Severity (Code):** parses heterogeneous cert/WHOIS response shapes defensively and escalates a domain to the worse of its certificate or registration band (critical `< 7d`, warning `< 21d`, otherwise ok).
- **Dedup vs Static Data (Code):** persists a per-UTC-day ledger of `domain:severity` pairs in workflow static data. The ledger resets at the day boundary, giving idempotent alerting without an external store.
- **Check WHOIS (HTTP):** uses `continueErrorOutput` so a single flaky WHOIS provider never aborts the whole run.
- **Switch Severity:** three-way fan-out into independent critical, warning, and ok pipelines.
- **Merge nodes:** `Merge Cert + WHOIS` joins the two lookups per domain; `Merge Alerts` recombines critical and warning findings before the digest.

## Required credentials / environment

Create these credentials in n8n and map them to the placeholder ids (`REPLACE_ME_*`) on import:

| Credential (type) | Used by |
| --- | --- |
| Cert API key (HTTP Header Auth) | Check TLS Cert |
| WHOIS API key (HTTP Header Auth) | Check WHOIS |
| Tracker API (HTTP Header Auth, Jira or Linear) | Create Incident Ticket |
| SendGrid API (HTTP Header Auth) | Send Daily Digest Email |

Environment variables:

| Variable | Purpose |
| --- | --- |
| `CERT_API_URL` | Base URL of the certificate-info API (domain is appended as a path segment). |
| `WHOIS_API_URL` | WHOIS API endpoint (domain sent as a query parameter). |
| `TRACKER_API_URL` | Issue tracker create-issue endpoint. |
| `TRACKER_PROJECT_ID` | Project or team id the incident ticket is filed under. |
| `SLACK_WEBHOOK_URL` | Incoming webhook for urgent alerts and the digest. |
| `DIGEST_FROM_EMAIL` | Verified SendGrid sender address. |
| `DIGEST_TO_EMAIL` | Digest recipient. |

No secrets are stored in the workflow JSON. All tokens are referenced through credentials or environment expressions.

## Import & setup

1. In n8n choose **Import from File** and select `workflow.json`.
2. Open each HTTP node with a credential warning and select or create the matching credential listed above.
3. Set the seven environment variables on the n8n host (or in your container env), then restart n8n so they are picked up.
4. Edit `Load Domains` to point at your real domain source, or replace the static array.
5. Run the workflow manually once to confirm credentials and endpoints respond, then set it **Active**.

## Production notes

- **Retries:** every external HTTP node retries three times with a 2-second backoff (`retryOnFail`, `maxTries: 3`, `waitBetweenTries: 2000`).
- **Dedup via static data:** `Dedup vs Static Data` keeps a `domain:severity` ledger keyed by UTC day in workflow static data. The same critical domain alerts once per day, not once per run. State survives restarts but is per-workflow and is not copied when the workflow is duplicated.
- **Rate limiting:** domains are processed one at a time through `Loop Over Domains`, which keeps request rates to the cert and WHOIS providers low and predictable. Increase the batch size only if your providers allow it.
- **Severity thresholds:** bands are defined in `Compute Severity` (critical `< 7` days, warning `< 21` days). Adjust both the day thresholds and the `0 8 * * *` schedule to match your renewal lead times and on-call timezone.
- **Partial failures:** a failed WHOIS lookup degrades gracefully to cert-only scoring with a note in the digest rather than failing the run.
