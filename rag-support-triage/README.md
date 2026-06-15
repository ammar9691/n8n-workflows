# RAG Support Triage (n8n)

An importable n8n workflow that turns inbound support tickets into grounded, auto-handled responses. A ticket arrives by webhook, an LLM classifies it, a self-hosted retrieval-augmented service supplies knowledge-base context, an LLM drafts a grounded reply, and a confidence model decides whether to auto-send or route to a human. High-urgency tickets are paged to on-call regardless of confidence. Every run produces an audit record.

## How it works

1. **Inbound Ticket** webhook receives `POST /support-triage` with `{ ticket_id, from, subject, body, channel }`.
2. **Normalize Ticket** trims and standardizes the fields and stamps `received_at`.
3. **Classify (LLM)** calls OpenAI `gpt-4o` and returns strict JSON: `intent`, `category`, `urgency` (`low|normal|high|critical`), `sentiment`, `language`, and a self-reported `certainty`.
4. **Parse Classification** validates the model output, clamps `certainty` to `0..1`, and falls back to safe defaults if parsing fails.
5. **Urgency Switch** routes `critical`/`high` down an escalation branch and `normal`/`low` (plus any fallback) down the standard branch. Both branches reconverge through **Merge Paths**, and **Flag Escalation** records whether on-call paging will be required.
6. **Retrieve KB (RAG)** calls the self-hosted retrieval service at `POST {{RAG_SERVICE_URL}}/ask` with `{ question, top_k }`. It uses an error output so an unreachable service degrades to human review instead of failing the run.
7. **Normalize RAG** reads `{ answer, sources, confidence }`, applies a retrieval floor (`>= 0.35`), and sets a `kb_hit` flag.
8. **KB hit?** branches: a hit drafts a reply, a miss uses a neutral holding response and zero self-confidence.
9. **Draft Reply (LLM)** calls `gpt-4o` constrained to the retrieved context and returns `{ reply, used_sources, self_confidence }`.
10. **Score Confidence** blends retrieval confidence, model self-confidence, and classification certainty into a single `0..1` score and selects an action.
11. **Auto-send threshold** sends and resolves the ticket when the score clears the bar and urgency is not critical; otherwise it posts a human review task with the draft attached.
12. **Needs paging?** pages on-call for any escalated ticket in addition to the review task.
13. **Build Audit Log** assembles a structured record and **Append Audit** ships it to a logging sink.
14. **Respond** returns `{ status, ticket_id, action_taken, confidence, escalated }` to the caller.

## Node highlights

- **Confidence model (Score Confidence):** weighted blend of `retrieval * 0.45 + self_confidence * 0.35 + certainty * 0.20`. Auto-send requires a KB hit, a blended score `>= 0.78`, and non-critical urgency. Anything else becomes a human review task.
- **Graceful RAG degradation:** the retrieval node uses `continueErrorOutput`, and the normalizer treats a missing answer or a sub-floor score as a KB miss, so retrieval problems never silently auto-send an ungrounded reply.
- **Branch reconvergence:** the urgency switch fans out to escalation and standard paths, then merges so the core retrieval and drafting logic is defined once rather than duplicated per branch.
- **Strict JSON parsing in Code nodes:** both LLM calls request `response_format: json_object`, and the parser Code nodes still defend against malformed output with try/catch and clamping.
- **Idempotency:** the helpdesk send and tag calls carry an `Idempotency-Key` derived from `ticket_id`, and the pager call uses a `dedup_key`, so retries (built-in `maxTries: 3`) do not double-send or double-page.

## Required credentials / environment

Create these as n8n credentials (HTTP Header Auth) and map them onto the matching nodes. Placeholder credential ids in the export read `REPLACE_ME`.

| Credential name | Used by | Header |
| --- | --- | --- |
| `OpenAI key` | Classify (LLM), Draft Reply (LLM) | `Authorization: Bearer <key>` |
| `Helpdesk API` | Send Reply, Tag Ticket Resolved | provider token header |
| `Audit sink` | Append Audit (optional) | provider token header |

Set these environment variables on the n8n host:

| Variable | Purpose |
| --- | --- |
| `RAG_SERVICE_URL` | Base URL of the self-hosted RAG service (the `/ask` path is appended) |
| `HELPDESK_API_URL` | Base URL of the helpdesk/email API used to send and tag |
| `SLACK_WEBHOOK_URL` | Incoming webhook for human review tasks |
| `PAGER_WEBHOOK_URL` | On-call paging endpoint (PagerDuty Events v2 or similar) |
| `AUDIT_SINK_URL` | Logging endpoint or sheet ingest URL for audit records |

This workflow pairs with a companion RAG service, **github.com/ammar9691/rag-qa**, a FastAPI app that exposes exactly the `POST /ask` endpoint consumed here and returns `{ answer, sources, confidence }` over an embedded vector store. Point `RAG_SERVICE_URL` at that service to run the full retrieval-augmented loop.

## Import and setup

1. In n8n, choose **Import from File** and select `workflow.json`.
2. Open each HTTP Request node that shows a credential warning and bind the credentials listed above.
3. Set the five environment variables on the n8n host and restart the instance so they are picked up.
4. Stand up the companion RAG service and confirm `POST $RAG_SERVICE_URL/ask` returns `{ answer, sources, confidence }`.
5. Activate the workflow and send a test ticket:

   ```bash
   curl -X POST https://<your-n8n-host>/webhook/support-triage \
     -H "Content-Type: application/json" \
     -d '{"ticket_id":"T-1001","from":"user@example.com","subject":"Cannot reset password","body":"The reset link in my email is expired.","channel":"email"}'
   ```

## Production notes

- **No secrets in the export.** Every key is referenced through n8n credentials and every service URL through `$env`. The file is safe to publish.
- **Retries.** Every external HTTP call sets `retryOnFail` with `maxTries: 3` and a 2 second backoff. Combined with idempotency keys, transient failures recover without duplicate side effects.
- **Failure isolation.** The RAG node continues on error to the normalizer, and the audit node continues on error so logging problems never block the webhook response.
- **Auditability.** Each run emits a single structured record with the classification, retrieval confidence, blended score, action taken, and escalation status, which supports later quality review and threshold tuning.
- **Tuning.** The blend weights and the `0.78` auto-send threshold live in the **Score Confidence** node. Start conservative, review the audit log, and raise the auto-send rate only as grounded accuracy is confirmed.
- **Human in the loop by default.** Anything that is not confidently grounded goes to a person, and critical tickets always reach on-call, so automation never silences an urgent customer.
