# AI Lead Enrichment & Scoring SDR

Turns an inbound lead into a scored, enriched, and routed sales action in a single webhook call. A lead arrives by HTTP, the workflow dedupes it, enriches the person and company from external data providers, uses an LLM to score fit and write a personalized opening line, then routes hot, warm, and cold leads to different destinations and writes an audit record.

This is built the way you would run it for a real revenue team: idempotent against duplicate submissions, resilient to provider outages, retry-aware on every external call, and observable through an audit log.

## How it works

1. **New Lead (Webhook)** accepts `POST /new-lead` with a JSON body of `{ email, name, company, domain, source, message }`. It responds through a response node so the caller gets the final result synchronously.
2. **Normalize Lead** trims and lowercases the input, and derives the company domain from the email when `domain` is not supplied.
3. **Dedup by Email** uses `$getWorkflowStaticData('global')` to keep a timestamped map of recently seen emails. If the same email arrived inside the dedup TTL window (default 24 hours) the lead is flagged as a duplicate. Expired entries are pruned on each run so the map does not grow without bound.
4. **Is Duplicate?** branches. Duplicates short-circuit to a no-op and a `duplicate` response, so no paid enrichment or LLM calls are spent. New leads continue.
5. **Enrich Company** and **Enrich Person** call the enrichment provider in parallel (company by domain, person by email). Both use retries and `continueErrorOutput` so a provider outage degrades the run instead of killing it.
6. **Merge Enrichment** combines the two enrichment responses by position.
7. **Build Feature Set** assembles a clean firmographic object (industry, employee count, estimated revenue, tech stack, country, person title, seniority). Every field is read defensively with fallbacks so missing enrichment never throws.
8. **Score & Personalize (LLM)** calls the OpenAI Chat Completions API (`gpt-4o` by default) with `response_format: json_object`, returning `{ fit_score, reasons, persona, personalized_opener, recommended_sequence }`.
9. **Parse & Tier** parses the model output, clamps `fit_score` to 0-100, and assigns a tier deterministically: hot (>= 80), warm (50-79), cold (< 50). Malformed model output falls back to a score of 0 and a cold tier rather than failing.
10. **Route by Tier (Switch)** sends each tier down its own path:
    - **HOT**: create a CRM deal, notify sales in Slack with the personalized opener, and draft an outreach email.
    - **WARM**: add the contact to a nurture sequence.
    - **COLD**: log the contact in the CRM as low priority.
11. **Merge Branches** rejoins the paths, then **Build Result + Audit** assembles the caller response and a richer audit record.
12. **Append Audit Log** writes the audit record to an external sink (set to continue on error so logging never blocks the response).
13. **Respond Result** returns `{ lead_id, tier, fit_score, action, status }` to the original caller.

## Node highlights

- **Dedup by Email (Code)**: the cost-control core. Workflow-global static data plus a TTL window makes the workflow idempotent against retried form posts and replayed CRM webhooks, so the same lead is never enriched or scored twice within the window.
- **Enrich Company / Enrich Person (HTTP)**: `retryOnFail` with `maxTries: 3` and `waitBetweenTries: 2000`, plus `onError: continueErrorOutput`. The feature builder then tolerates partial or absent enrichment.
- **Build Feature Set (Code)**: normalizes two differently shaped provider payloads into one stable object with safe path lookups and fallbacks.
- **Score & Personalize (LLM)**: strict JSON mode against `gpt-4o`, low temperature for stable scoring, system prompt pinned to the exact output schema.
- **Parse & Tier (Code)**: never trusts the model blindly. It clamps the score, validates types, and derives the tier in code so routing is deterministic.
- **Route by Tier (Switch)**: three real branches with distinct downstream actions, not a cosmetic split.

## Required credentials / environment

Create these n8n credentials (Credentials menu) and reference them as the workflow expects:

| Credential name | Type | Used by |
| --- | --- | --- |
| `enrichApi` | HTTP Header Auth | Enrich Company, Enrich Person |
| `openAiApi` | OpenAI API | Score & Personalize (LLM) |
| `crmApi` | HTTP Header Auth | Create CRM Deal, Log Cold Lead |
| `slackApi` | HTTP Header Auth | Notify Sales (Slack) |
| `emailToolApi` | HTTP Header Auth | Draft Outreach Email, Add to Nurture Sequence |
| `auditApi` | HTTP Header Auth | Append Audit Log |

Set these environment variables on the n8n host:

| Variable | Purpose | Example |
| --- | --- | --- |
| `ENRICH_API_URL` | Base URL of the enrichment provider | `https://company.clearbit.com` |
| `OPENAI_MODEL` | Chat model id (optional, defaults to `gpt-4o`) | `gpt-4o` |
| `CRM_API_URL` | CRM API base URL | `https://api.hubapi.com/crm/v3` |
| `SLACK_API_URL` | Slack API base URL | `https://slack.com/api` |
| `EMAIL_TOOL_URL` | Sequence / nurture tool base URL | `https://api.youremailtool.com` |
| `AUDIT_API_URL` | Audit sink base URL | `https://logs.internal.example.com` |
| `DEDUP_TTL_MS` | Dedup window in milliseconds (optional, defaults to 86400000) | `86400000` |

No secrets are stored in `workflow.json`. API keys live only in n8n credentials, and all base URLs are read from environment variables.

## Import & setup

1. In n8n, open the workflows list and choose **Import from File**, then select `workflow.json`.
2. Open each HTTP node and bind the matching credential from the table above (the credential names are pre-set; you only need to point them at real credentials).
3. Set the environment variables listed above on the n8n host and restart n8n so they are picked up.
4. Open the **New Lead** node and copy the production webhook URL.
5. Send a test request:
   ```bash
   curl -X POST "https://YOUR-N8N-HOST/webhook/new-lead" \
     -H "Content-Type: application/json" \
     -d '{"email":"jane@acme.io","name":"Jane Doe","company":"Acme","domain":"acme.io","source":"contact-form","message":"Looking for a solution"}'
   ```
6. Confirm the response returns `lead_id`, `tier`, `fit_score`, and `action`. Resend the same payload to confirm it returns `status: duplicate`.
7. Activate the workflow once the destinations (CRM, Slack, email tool, audit sink) are wired to real endpoints.

## Production notes

- **Idempotency and cost control**: dedup runs before any paid call. The static-data map is pruned every execution and keyed on the normalized email.
- **Resilience**: every external HTTP node retries three times with a 2 second backoff. Enrichment nodes continue on error so a provider outage does not block scoring. The audit and cold-log nodes continue on error so logging or low-priority writes never block the caller response.
- **Deterministic routing**: tiering happens in code after the LLM call. Score clamping and a fallback cold tier mean malformed model output cannot misroute or crash the run.
- **Observability**: the audit log captures the score, reasons, tier, action, enrichment status, and timestamps for every processed lead.
- **Scaling the dedup store**: workflow static data is fine for a single instance and moderate volume. For high volume or multi-instance n8n, move the dedup check to an external store such as Redis or your database and keep the same short-circuit logic.
- **Provider swaps**: the enrichment, CRM, Slack, and email nodes are plain HTTP requests, so swapping Clearbit for Apollo, or HubSpot for Pipedrive, is a matter of changing the URL path and the credential, not the workflow shape.
