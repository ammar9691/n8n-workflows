# n8n Workflows



## Workflows

| Workflow | Domain | What it does |
| --- | --- | --- |
| [SSL & Domain Expiry Watchdog](./ssl-domain-watchdog) | DevOps / Infra | Daily check of TLS certificate and domain registration expiry across a fleet, with escalating severity, automatic ticket creation for critical items, and dedup so it does not re-alert the same domain twice in a day. |
| [RAG Support Triage](./rag-support-triage) | AI | Classifies inbound tickets, retrieves knowledge-base context from a RAG service, drafts a grounded reply, and auto-sends on high confidence or routes to a human otherwise. Pairs with the [rag-qa](https://github.com/ammar9691/rag-qa) service. |
| [AI Lead Enrichment & Scoring SDR](./ai-lead-sdr) | AI | Dedupes new leads, enriches person and company data, scores fit with an LLM, writes a personalized opener, and routes hot, warm and cold leads to different destinations. |
| [SEO Blog Content Engine](./seo-blog-engine) | Content / SEO | Pulls queued keywords, researches the SERP, generates an SEO outline then a full draft, runs automated quality checks, and publishes passing drafts to WordPress for human review. Never auto-publishes live. |
| [Multi-Platform Social Publisher](./social-publisher) | Social | Turns a published blog post into platform-tailored posts for Facebook, Instagram, X and LinkedIn, holds them at a human approval gate, then publishes to each platform and tracks the resulting post ids. |


## Importing a workflow

1. Open your n8n instance.
2. Go to Workflows, then Import from File.
3. Select the `workflow.json` for the workflow you want.
4. Open each credential placeholder and connect your own account.
5. Set the environment variables listed in that workflow's README.
6. Run a manual execution before activating the trigger.

Each subfolder README has the full credential and environment list, the node-by-node walkthrough, and production notes specific to that workflow.
