# Local Business Lead Scorer — n8n Workflow

Triggered by a **web form** (category, city, state, min reviews). Searches Google Maps,
filters by review count, and for businesses **with** a website, scores it on:
- Page speed (Google PageSpeed Insights, mobile)
- Agentic-search readiness (robots.txt AI-crawler access, llms.txt, schema.org structured data)
- Design/copy quality (Claude Haiku judges the scraped homepage text)

Businesses with **no** website are tagged as immediate leads with no scoring needed.
Everything lands in a Google Sheet.

## 1. Import
In n8n: **Workflows → Import from File** → select `lead-scoring-workflow.json`.

## 2. Set up credentials (3 total)

API keys are **not** stored in the workflow file — they live in n8n's encrypted Credentials
store and get attached to the relevant nodes (used by both forms, since both branches call the
same three external APIs). After import, open each node below; n8n will show "Credential not
found" until you create/select one.

| Nodes that use it | Credential type to create | Credential Name | Inner Name/Value pair |
|---|---|---|---|
| **2. Search Places** / **B4. Find Matching Google Listing** | Generic → Header Auth | `Google Places API Key (header: X-Goog-Api-Key)` | Name: `X-Goog-Api-Key`, Value: your Google API key |
| **8. PageSpeed Insights** / **B6. PageSpeed Insights** | Generic → Query Auth | `Google PageSpeed API Key (query param: key)` | Name: `key`, Value: same Google API key |
| **14. AI Score Site** / **B12. AI Score Site** | Generic → Header Auth | `Anthropic Claude API Key (header: x-api-key)` | Name: `x-api-key`, Value: your Anthropic API key |

Note: the "Inner Name" in the last column (`key`, `x-api-key`, `X-Goog-Api-Key`) is a fixed
string each API requires — not something you get to choose, and not editable without breaking
the request. Since n8n's generic credential dialog has no separate description field, that
literal required name is baked into the Credential Name itself (the parenthetical) purely so
it's identifiable at a glance in your credentials list — it has no functional effect.

To create one: open the node → **Credential** dropdown → **Create New Credential** → pick the
generic type shown above → fill in the Name/Value pair exactly as listed → Save. Reuse the same
credential across both forms and any future workflows once created.

## 3. Getting the Google API key
1. console.cloud.google.com → create/select a project.
2. **APIs & Services → Library** → enable **Places API (New)** and **PageSpeed Insights API**.
3. **APIs & Services → Credentials → Create Credentials → API key.**
4. Click the key → **Restrict Key** → limit it to those two APIs.
5. Attach a billing account to the project — PageSpeed is free, but Places Text Search is metered past its free tier. Set a budget alert.

## 4. Getting the Anthropic API key
1. console.anthropic.com → **Get API key**. This is a separate account/billing context from
   your claude.ai subscription — Pro/Max plans do not include API usage, and API usage is billed
   separately by token, not deducted from your subscription.
2. The workflow defaults to `claude-haiku-4-5-20251001` — cheap and sufficient for this task.
   Swap the `model` string inside Node 14's JSON body if you want Sonnet instead.

## 5. Set up the Google Sheet destination
Open **Node 17 "Append to Google Sheet"**:
- Attach your Google Sheets OAuth2 credential (n8n prompts you to connect your Google account).
- Set `documentId` to your target Google Sheet's ID (from its URL).
- Set `sheetName` to a tab named `Leads`, with this exact header row (verified against what
  each branch actually outputs — see the note below if you set this up before):
```
place_id, name, website, rating, review_count, phone, address, lead_type, technical_score, pagespeed_score, lcp, cls, ai_crawlers_blocked, crawler_access_ok, has_schema_markup, has_llms_txt, design_score, copy_score, verdict, pitch, batch_size, batch_llms_txt_count, batch_crawler_blocked_count, batch_avg_pagespeed, google_listing_matched, name_source
```
  `batch_*` columns only populate on Flow 1 rows (the category/location search — there's no
  batch to compare against for a single graded site). `google_listing_matched` and
  `name_source` only populate on Flow 2 rows (grading a single website — Flow 1's businesses
  come straight from Maps, so there's no name-matching step to report on). Both are expected
  to show blank on the other flow's rows, not a bug.

  **If you set your header row up before this note was added:** two fields
  (`google_listing_matched`, `name_source`) were missing from earlier guidance, which meant
  that data was being silently dropped rather than written to your sheet. Add those two
  columns now to start capturing them going forward.

## 6. The triggers: Webhook nodes + a local control panel

Both entry points are **Webhook** nodes (not Form Trigger, not Manual Trigger). A separate file,
`lead-scorer-panel.html`, gives you an actual page with two forms that POST straight to them.

**Flow 1 — New Lead Search:** `Webhook: New Lead Search`, path `new-lead-search`, expects
JSON body `{ category, location, min_reviews }`.

**Flow 2 — Grade an Existing Website:** `Webhook: Grade Website`, path `grade-website`,
expects JSON body `{ website, name_override, city, state }`.

### Activate the workflow
Webhooks only listen while the workflow is **Active** (top-right toggle in the n8n editor) —
unlike Manual Trigger, there's no "Test workflow" button involved in normal use. Once active,
the real endpoints are:
```
http://<your-unraid-host>:5678/webhook/new-lead-search
http://<your-unraid-host>:5678/webhook/grade-website
```
Both respond immediately ("Search started…") and let the workflow keep running in the
background — so the page doesn't hang waiting for every business to finish scoring. Check the
Google Sheet a few minutes later for results.

### Using the control panel
Open `lead-scorer-panel.html` directly in a browser (double-click it, or host it any way you
like on your LAN). The n8n base URL field defaults to `http://192.168.10.10:5678` and is also
remembered per-browser via localStorage once you change it — so you shouldn't need to type it
at all unless it changes. Fill in a tab, hit the button, or hit **Clear** to reset that tab's
fields.

**This now waits for the real result, not just "request received."** The webhooks were
originally set to respond the instant they received a request — before any actual scraping or
scoring happened — which meant the page could never tell you if something failed downstream.
That's fixed: both webhooks now hold the connection open until the workflow actually finishes
(Node 17 append → a summary is built → sent back as the HTTP response), so the panel's log
shows the real outcome — verdict/score for a single graded site, or a lead count for a search —
or a genuine error if something broke mid-run. Trade-off: the button now sits on "Working…" for
the full duration (a single site grade is ~10–20s; a full category search can take a couple of
minutes since businesses are processed one at a time).

If a request fails from the panel, it's almost always one of:
- **Workflow not active** — the toggle in the n8n editor, not a "listening" test mode.
- **Wrong base URL** — must be reachable from whatever device is displaying the page (a LAN IP if you're on your phone, `localhost` only works if the page is opened on the same machine n8n runs on).
- **CORS** — the webhook nodes have `Allowed Origins` set to `*` already so any page can call them; if you tighten that later, make sure to include whatever origin the panel is served from.

Both flows write to the same Google Sheet (Node 17), with `lead_type` distinguishing rows:
`no_website`, `has_website` (Flow 1's scoring branch), and `direct_website_check` (Flow 2).

How Flow 2 infers the business name and finds the Google listing:
1. Fetches the homepage and looks for a name in this order: your `name_override` field →
   schema.org `Organization`/`LocalBusiness` JSON-LD `name` → `og:site_name` meta tag →
   `<title>` tag (text before the first `|` or `-`) → falls back to a title-cased version of
   the domain itself if nothing else is found.
2. Searches Google Places by that inferred name (plus `city`/`state` if filled in — this is the
   single biggest accuracy lever; a bare name like "Riverside Dental" can match the wrong
   listing without a location hint).
3. If multiple results come back, it prefers whichever one's own website hostname matches the
   URL you submitted; otherwise it takes the top result.
4. If no listing is found at all, `google_listing_matched` is `false` and review count/rating
   are left blank — the technical/AI scoring still runs regardless, since that doesn't depend
   on Maps data.

**Caveat worth knowing:** name inference is a best-effort heuristic, not guaranteed. Generic
titles ("Home", "Welcome"), multi-location businesses, or sites with no metadata at all can
produce a wrong or unhelpful inferred name, which then produces a wrong or empty Google Places
match. Check `name_source` and `google_listing_matched` in the output before trusting the
review count on any given row — or just fill in the Business Name field in the panel when you
already know it.


## 7. Competitor claims in the AI pitch — now grounded in real data

Earlier versions of the prompt let Claude Haiku write things like "competitors are already using
this" without any actual competitor data behind it — that's a fabricated claim, not a verified
one, and a real risk if a prospect ever asks "which competitors, specifically?"

This is now fixed for **Flow 1** (the category/location search): since it processes up to 20
real local businesses in one batch, a new node (**13b. Compute Real Competitor Stats**) counts,
across that same batch, how many actually have an `llms.txt` file and what their average
PageSpeed score is — then the AI prompt is instructed to only make a comparative claim when the
numbers genuinely support it, and to cite the real count if it does. Those numbers are also now
their own sheet columns (`batch_size`, `batch_llms_txt_count`, `batch_crawler_blocked_count`,
`batch_avg_pagespeed`) so you can audit any pitch line against real data before sending it,
rather than taking the AI's word for it.

**Flow 2** (grading a single website) has no batch to compare against at all — there's no other
business in that run — so its prompt now explicitly forbids any competitor or "industry" framing
entirely. The pitch there is grounded only in that one site's own numbers.

General takeaway worth keeping in mind: any time you extend this workflow's AI prompts further,
watch for the model asserting things it wasn't actually given data for — plausible-sounding
claims about competitors, industry trends, or "what's common" are the easiest kind of AI
hallucination to miss, precisely because they sound reasonable.
- **Places API response shape**: Google occasionally tweaks field names on the (New) Places API. If Node 3 ("Split Into Items") comes back empty, open Node 2's output and confirm the `places[]` array structure still matches (`displayName.text`, `websiteUri`, `userRatingCount`, etc.).
- **continueOnFail is enabled** on all the "might legitimately 404" HTTP calls (homepage fetch, robots.txt, llms.txt) so one missing file doesn't kill the whole run — those nodes just pass through with defaults.
- **Rate limiting**: PageSpeed Insights defaults to ~100 requests/100 seconds per key. If you're running large batches, add a `Wait` node (1-2 sec) before Node 8 to avoid 429s.
- **technical_score weighting** (in Node 13) is a starting point: 50% page speed, 20% AI-crawler access, 20% structured data, 10% llms.txt. Adjust the multipliers to match what you actually want to prioritize in outreach.
- **Costs**: Places Text Search + Places Details are the main paid external cost; PageSpeed Insights and the robots.txt/llms.txt fetches are free; Claude Haiku costs roughly $0.005–0.01 per business scored at current pricing (Aug 2026), billed separately from any claude.ai subscription.
- **Storage**: lead results go to your Google Sheet (Node 17). n8n's own execution history (every run's inputs/outputs, useful for debugging) is stored locally in n8n's database on your Unraid box — worth setting a data-pruning policy in n8n's settings so it doesn't grow unbounded from all the scraped HTML.

