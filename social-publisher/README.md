# Multi-Platform Social Publisher (n8n)

A production-grade n8n workflow that turns a single "blog published" event into approved, platform-tailored posts on Facebook, Instagram, X, and LinkedIn. It uses an LLM to write the copy, enforces each platform's hard limits in code, pauses for one human approval before anything goes live, fans out to each API independently so one failure cannot block the rest, then records every post ID and reports the outcome.

## Summary

When your CMS publishes a post, it calls this workflow's webhook with the title, URL, excerpt, image, and tags. The workflow drafts four sets of copy, formats them per platform, and sends a single Slack approval card. A reviewer clicks Approve or Reject. On approval the workflow publishes to all four platforms, collects the returned post IDs, appends a row to a tracking sheet, and posts a success/failure summary back to Slack. On rejection it notifies Slack and aborts without posting anything.

The whole run is one execution: it pauses on the approval gate and resumes on the reviewer's click, so no state is lost between draft and publish.

## How it works

1. **Blog Published** webhook receives `POST /blog-published` with `{ title, url, excerpt, image_url, tags[] }`.
2. **Normalize Source** maps the request body into clean top-level fields used throughout the run.
3. **Generate Variants (LLM)** calls OpenAI Chat Completions with `response_format: json_object` and returns structured copy: a Facebook post, an Instagram caption, an X thread (array of tweets), a LinkedIn post, and a hashtag list.
4. **Enforce Platform Rules** (Code) parses the LLM JSON, enforces per-platform character limits, appends UTM parameters to the link per platform, attaches hashtags appropriately, and splits the X thread into ordered, clamped tweets. It outputs one structured payload object.
5. **Post Approval Request** sends a Slack Block Kit message showing all four variants with Approve and Reject buttons. Both buttons point at `{{ $execution.resumeUrl }}` with a `decision` query parameter.
6. **Await Approval** is a Wait node set to resume on a webhook. The run pauses here until the reviewer clicks a button, which POSTs back to the resume URL.
7. **Approved?** (IF) reads `decision`. The false branch runs **Notify Rejected** and **Respond Rejected**, ending the run. The true branch continues.
8. **Fan Out Platforms** dispatches to all four publishing branches in parallel:
   - **Facebook**: single `POST /{PAGE_ID}/feed` with message and link.
   - **Instagram**: two steps, **IG Create Container** then **IG Publish** using the returned `creation_id`.
   - **X**: **Split X Tweets** expands the thread, **Loop Over Tweets** posts each tweet, chaining `reply.in_reply_to_tweet_id` to the previous tweet ID, and **Collect X Result** gathers the IDs.
   - **LinkedIn**: single `POST /v2/ugcPosts` article share.
9. **Merge Results** joins the four branch outputs and **Collect Post IDs & Status** (Code) normalizes each into `{ platform, ok, post_id, error }` and computes succeeded/failed lists.
10. **Append to Tracking Sheet** posts the IDs, URLs, and timestamps to your sheet webhook (Google Sheets/Apps Script, Airtable, or similar).
11. **Post Summary to Slack** reports which platforms succeeded and which failed.
12. **Respond Published** returns `{ status, results }` to the original caller.

## Node highlights

- **Human approval gate** (`Await Approval`): a Wait node with `resume: webhook`. Nothing is published before a person approves. The approval card carries `$execution.resumeUrl`, so the reviewer's click wakes the exact paused execution.
- **Per-platform rule engine** (`Enforce Platform Rules`): a single Code node that owns all formatting logic. Limits enforced: X 280 per tweet, Instagram 2200, LinkedIn 3000, Facebook 63206 (trimmed sensibly). It also appends UTM tags, normalizes hashtags, and orders the thread.
- **Thread chaining** (`Loop Over Tweets` + `Post Tweet`): each tweet replies to the previous one's ID to build a real thread rather than four detached posts.
- **Error isolation**: every external API node uses `onError: continueErrorOutput` so a single platform outage routes to the merge as a failure instead of killing the run. The summary still reports the platforms that did succeed.
- **Idempotent-friendly tracking** (`Append to Tracking Sheet`): every published ID and URL is logged with a timestamp so re-runs can be detected and reconciled.

## Required credentials / environment

Configure these in the n8n Credential store and bind them on import:

| Credential type | Used by | Purpose |
| --- | --- | --- |
| `openAiApi` | Generate Variants (LLM) | OpenAI API key for copy generation |
| `facebookGraphApi` | Facebook Post, IG Create Container, IG Publish | Page + Instagram publishing token |
| `twitterOAuth2Api` | Post Tweet | X (Twitter) v2 OAuth2 |
| `linkedInOAuth2Api` | LinkedIn Post | LinkedIn member share |
| `httpHeaderAuth` | Post Approval Request, Notify Rejected, Post Summary to Slack | Slack bot token as `Authorization: Bearer xoxb-...` |

Environment variables:

| Variable | Example | Purpose |
| --- | --- | --- |
| `OPENAI_MODEL` | `gpt-4o` | Model for variant generation (defaults to gpt-4o) |
| `FB_PAGE_ID` | `123456789012345` | Facebook Page ID for `/feed` |
| `IG_USER_ID` | `178414...` | Instagram Business account ID |
| `LINKEDIN_AUTHOR_URN` | `urn:li:person:abc123` | LinkedIn author URN |
| `SLACK_CHANNEL_ID` | `C0123456789` | Channel that receives the approval card and summary |
| `TRACKING_SHEET_WEBHOOK` | `https://script.google.com/.../exec` | Endpoint that appends a tracking row |

No secrets are stored in the workflow. All tokens resolve through the Credential store and all IDs through environment variables.

## Import & setup

1. In n8n choose **Import from File** and select `workflow.json`.
2. Open each node that shows a missing-credential warning and bind it to a credential of the matching type (table above).
3. Set the six environment variables on your n8n instance (Docker `-e`, `.env`, or queue-mode env).
4. Activate the workflow. Copy the production URL of the **Blog Published** webhook and point your CMS publish hook at it.
5. Confirm the **Await Approval** node has a reachable production webhook URL. Your n8n `WEBHOOK_URL` must be publicly resolvable so Slack buttons can call back.
6. Send a test `POST` to `/blog-published` and approve the Slack card to verify an end-to-end run.

## Production notes

**Approval gate.** The Wait node holds the execution open until the resume webhook is called. Set a sensible execution timeout policy in n8n if you want stale, unapproved drafts to expire. The Approve and Reject buttons differ only by the `decision` query parameter, so a single resume URL serves both outcomes.

**Rate limits.** X v2 and the Instagram Graph API are the tightest. The X thread is posted sequentially in a loop with retry backoff, which keeps it under per-app write limits. Instagram requires the two-step container-then-publish flow; the container can take a moment to be ready, so retries cover transient `not ready` responses. All external calls retry three times with a 2 second wait.

**Per-platform character limits.** Enforced in `Enforce Platform Rules`: X 280 per tweet, Instagram 2200, LinkedIn 3000, Facebook 63206. Copy that exceeds a limit is trimmed at a word-safe boundary with an ellipsis rather than rejected, so a run never fails purely on length.

**Idempotency.** Each successful publish writes its platform post ID, the source URL, and a timestamp to the tracking sheet. Before wiring an automatic CMS trigger, decide how to dedupe: gate on source URL already present in the sheet, or have the CMS send a one-time event. The error-isolated branches mean a partial failure can be retried per platform without double-posting the ones that already succeeded.

**Failure handling.** A failing platform routes to the merge as an error record instead of aborting. The Slack summary lists succeeded and failed platforms so an operator can re-run a single branch manually if needed.
