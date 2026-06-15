# SEO Blog Content Engine (n8n)

An automated, schedule-driven content pipeline that turns a queued keyword into a
review-ready WordPress draft. On a weekday schedule it pulls the next priority
keyword from an editorial backlog, researches the live SERP, generates an
SEO-optimised outline, writes a full long-form draft, runs automated SEO quality
checks, and publishes only passing drafts to WordPress as **draft** status for a
human editor. Failing drafts are flagged in Slack with the exact issues found.

Nothing is ever published live automatically. A human always reviews and presses
publish in WordPress.

## Summary

- 28 nodes, branching (not linear), with a quality-gate loop-back.
- Two LLM stages (outline, then draft) using OpenAI `gpt-4o` with
  `response_format: json_object` for structured output.
- A self-correcting outline step that regenerates once if validation fails.
- A deterministic SEO scoring step in code (word count, keyword density,
  meta length, heading structure, internal links, readability proxy).
- Every external HTTP call retries 3 times with a 2 second backoff.
- All secrets live in n8n credentials and all endpoints come from environment
  variables. No keys are stored in the workflow file.

## How it works

1. **Schedule Trigger** fires weekdays at 09:00 (`0 9 * * 1-5`).
2. **Fetch Content Queue** GETs pending keyword rows from the editorial backlog
   (`CONTENT_QUEUE_URL`, header-authenticated). In real use this is the
   content calendar held in Airtable, a Google Sheet, or a CMS.
3. **Pick Next Topic** normalises the different possible payload shapes, filters
   to `pending` rows, and selects the lowest `priority` number. If nothing is
   queued it sets `has_topic = false`.
4. **Has topic?** routes to research when there is work, otherwise to the
   **Nothing queued** no-op so the run ends cleanly.
5. **SERP Research** queries a SERP API (`SERP_API_URL`, SerpApi style) for the
   top organic results and People Also Ask. It uses `continueErrorOutput` so a
   SERP failure does not kill the run; both outputs feed the brief.
6. **Build Research Brief** extracts competitor headings, PAA questions, common
   subtopics, and a target word count derived from competitor length signals. It
   degrades gracefully when the SERP is empty.
7. **Generate Outline (LLM)** asks OpenAI for structured JSON: `title`, `slug`,
   `meta_description`, `target_keyword`, `secondary_keywords[]`,
   `h2_sections[{h2, h3[]}]`, and `internal_link_suggestions[]`.
8. **Validate Outline** parses and checks the outline: required fields, meta
   length 120 to 158 characters, and at least four H2 sections. An
   `outline_attempt` counter lives on the item.
9. **Outline OK?** (Switch) has three outputs: `valid` continues to drafting,
   `retry` loops back to **Generate Outline (LLM)** once, and the fallback
   `giveup` path forces a fail after the second failed attempt.
10. **Write Draft (LLM)** writes the full article as semantic HTML following the
    outline, placing the keyword in the title, first paragraph, and at least one
    H2. Returns `{ html_body, word_count, internal_links_rendered }`.
11. **SEO Quality Checks** computes metrics and produces `seo_score`,
    `seo_issues[]`, and a `seo_pass` boolean (see node highlights below).
12. **Quality passed?** branches:
    - PASS: **Create WordPress Draft** (REST `POST /wp-json/wp/v2/posts`,
      `status=draft`), then **Notify Reviewer (Slack)** with the edit link and
      score, then **Mark Queue: drafted**.
    - FAIL: **Flag for Human (Slack)** with the issue list, then
      **Mark Queue: needs_review**.
13. Both branches converge at **Merge Outcomes**, then **Build Run Report**
    assembles one log row and **Append Run Log** persists it.

## Node highlights

- **Pick Next Topic** (Code): tolerates Airtable (`records`), generic
  (`items`/`data`/`rows`) and flat shapes, then prioritises and seeds the run
  state (`outline_attempt`, `run_started_at`).
- **Build Research Brief** (Code): turns raw SERP output into a usable brief and
  sets a sensible target word count even with no competitor data.
- **Validate Outline** + **Outline OK?**: the quality-gate loop-back. One free
  regeneration, then a hard stop that still reports the problem to a human.
- **SEO Quality Checks** (Code): the deterministic gate. It measures
  - word count against the brief target,
  - primary keyword density, flagged below 0.5% or above 2.5% (the latter is a
    critical, automatic fail),
  - meta description length 120 to 158 characters,
  - presence of H2 and H3 headings (missing H2 is critical),
  - internal link count,
  - a readability proxy (average sentence length under ~25 words),
  - keyword presence in the title and intro.
  It starts at 100, deducts per issue, and passes only when
  `score >= 80` with no critical issue.
- **Create WordPress Draft** (HTTP): always `status=draft`. The workflow never
  sets `publish`.

## Required credentials / environment

Configure these as n8n credentials (the workflow ships with placeholder
credential references, not secrets):

| Credential | Type | Used by |
| --- | --- | --- |
| Content Queue API | `httpHeaderAuth` | Fetch Content Queue, Mark Queue nodes, Append Run Log |
| SerpApi | `httpQueryAuth` | SERP Research |
| OpenAI API | `httpHeaderAuth` (`Authorization: Bearer <key>`) | Generate Outline, Write Draft |
| WordPress REST | `wordpressApi` (Application Password) | Create WordPress Draft |

Slack notifications post to an incoming webhook URL supplied via environment, so
no Slack credential is required for the webhook approach.

Environment variables:

| Variable | Purpose |
| --- | --- |
| `CONTENT_QUEUE_URL` | Base URL of the editorial backlog API |
| `SERP_API_URL` | SERP provider search endpoint |
| `WORDPRESS_BASE_URL` | WordPress site root (used for the REST endpoint and the admin edit link) |
| `SLACK_WEBHOOK_URL` | Slack incoming webhook for reviewer and failure alerts |
| `RUN_LOG_URL` | Endpoint that stores the per-run report row |

## Import and setup

1. In n8n choose **Import from File** and select `workflow.json`.
2. Open each HTTP node that shows a credential placeholder and select or create
   the matching credential from the table above.
3. Set the five environment variables in your n8n instance (`.env`, Docker env,
   or host environment) and restart n8n so they are picked up.
4. Confirm your queue API returns rows with at least `target_keyword`,
   `priority`, and `status`, and accepts a `PATCH` to update a row by id.
5. Adjust the schedule in **Schedule Trigger** if 09:00 on weekdays does not suit
   your publishing cadence.
6. Run once manually with a single test keyword in the queue and confirm a draft
   appears in WordPress under Posts, in draft status.

## Production notes

- **No live publishing.** WordPress posts are created as `draft`. A human edits
  and publishes. This is intentional and should not be changed without an
  explicit editorial sign-off process.
- **Retries.** Every external HTTP node uses `retryOnFail` with `maxTries: 3` and
  `waitBetweenTries: 2000` to absorb transient API errors.
- **SERP resilience.** SERP Research uses `continueErrorOutput`; the brief step
  handles an empty result set so the pipeline still produces an outline.
- **Self-correction.** The outline is regenerated once on validation failure
  before the run gives up, which catches most malformed LLM responses without a
  human in the loop.
- **Observability.** Every run, pass or fail, writes a report row
  (score, outcome, issues, word count, density) to `RUN_LOG_URL` for trend
  tracking and content audits.
- **Cost control.** The draft LLM call only runs after the outline passes
  validation, so failed outlines do not incur the expensive long-form generation.
- **Tuning.** Density thresholds, the minimum passing score, and the target word
  count logic live in the Code nodes and can be tuned per site or per niche.
