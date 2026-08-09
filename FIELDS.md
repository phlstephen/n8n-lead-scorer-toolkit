# Field Reference — Leads Sheet Columns

What each column means, where it comes from, and when to expect it blank. Blank isn't always
a bug — several columns only apply to one of the two flows (see the **Populated on** note for
each).

---

### Identity & Maps data
*Populated on: all rows*

| Column | Meaning |
|---|---|
| `place_id` | Google's internal ID for this business listing. Useful for looking it up again later or de-duplicating across multiple searches. |
| `name` | Business name. From Google Maps directly (Flow 1), or the AI's best-guess inferred name (Flow 2 — see `name_source` below). |
| `website` | The business's website URL, if it has one. Blank means no website was found on the Maps listing (Flow 1) — that's the whole point of a `no_website` lead. |
| `rating` | Google's average star rating (1–5). |
| `review_count` | Number of Google reviews. This is the number the Minimum Reviews filter checks against in Flow 1. |
| `phone` | Business phone number, from Google Maps. |
| `address` | Business address, from Google Maps. |
| `lead_type` | One of three values — see the table below. Tells you which flow produced this row and what to expect from the other columns. |

**`lead_type` values:**
| Value | Meaning |
|---|---|
| `no_website` | Flow 1 found this business but Google Maps lists no website for it. No scoring ran — nothing to score. |
| `has_website` | Flow 1 found this business, it has a website, and the full technical + AI scoring chain ran. |
| `direct_website_check` | Came from Flow 2 (grading a single URL you supplied directly), not from a category/location search. |

---

### Technical scoring
*Populated on: `has_website` and `direct_website_check` rows only. Blank on `no_website` rows — there's nothing to score.*

| Column | Meaning |
|---|---|
| `technical_score` | A single 0–100 score combining the four signals below: 50% page speed, 20% AI-crawler access, 20% structured data, 10% llms.txt. See `13. Compute Technical Score` in the workflow if you want to change these weights. |
| `pagespeed_score` | Raw Google PageSpeed Insights performance score (0–100), mobile strategy. Fluctuates run-to-run by a few points — that's normal Lighthouse variance, not a bug. |
| `lcp` | Largest Contentful Paint — how long the biggest visible element took to load, from the same PageSpeed test. |
| `cls` | Cumulative Layout Shift — how much the page visually jumps around while loading. Lower is better; 0 is perfectly stable. |
| `ai_crawlers_blocked` | List of AI crawler names (GPTBot, ClaudeBot, PerplexityBot, etc.) explicitly disallowed in the site's `robots.txt`. Empty list `[]` means none are blocked. |
| `crawler_access_ok` | `TRUE` if no AI crawlers are blocked, `FALSE` if at least one is. Shorthand version of `ai_crawlers_blocked` for quick filtering/sorting. |
| `has_schema_markup` | `TRUE` if the homepage has any schema.org JSON-LD structured data (Organization, LocalBusiness, etc.) — the markup that helps both Google and AI agents understand what the business actually is. |
| `has_llms_txt` | `TRUE` if the site has an `/llms.txt` file. Most small business sites don't yet — this is a very new, mostly-not-adopted standard, so a `FALSE` here isn't a strong signal on its own. |

---

### AI-generated verdict
*Populated on: `has_website` and `direct_website_check` rows only.*

| Column | Meaning |
|---|---|
| `design_score` | 1–10, from Claude judging the scraped homepage **text**. Important: as currently built, this is judging clarity/dated-ness of the copy and messaging — **not actual visual design**, since no screenshot is fed to the model yet. Treat the name as slightly aspirational until that's added. |
| `copy_score` | 1–10, Claude's read on the quality of the on-page writing/messaging. |
| `verdict` | One of `terrible`, `needs_work`, `decent`, `good` — Claude's overall one-word summary. |
| `pitch` | A single AI-written sentence meant as a cold-outreach opener, grounded in this business's actual numbers. **Always review before sending to a real prospect** — it's a first draft from a small model, not guaranteed-accurate copy. |

---

### Batch competitor context
*Populated on: `has_website` rows only.* Blank on `no_website` and `direct_website_check` rows —
there's no "batch" to compare against for a single graded site, and no-website leads skip
scoring entirely.

| Column | Meaning |
|---|---|
| `batch_size` | How many businesses from the *same search run* (same category + location) actually had a website and got scored. |
| `batch_llms_txt_count` | Of those, how many have an `llms.txt` file. This is what makes the AI's competitor claims in `pitch` real rather than invented — see the note below. |
| `batch_crawler_blocked_count` | Of those, how many block at least one AI crawler in `robots.txt`. |
| `batch_avg_pagespeed` | Average PageSpeed score across the batch. Useful for framing — "you're below average for businesses like yours" is a stronger, truer pitch line than a number in isolation. |

**Why this exists:** early versions of the AI prompt let Claude write things like "competitors
are already using this" with zero actual data behind it — a plausible-sounding but fabricated
claim. These four columns are the real numbers now fed into the prompt, and the prompt is
instructed to only make a comparative claim when they genuinely support it. If you ever see a
competitor claim in `pitch` that seems off, cross-check it against these columns before trusting
it.

---

### Website-grading match confidence
*Populated on: `direct_website_check` rows only* — these only apply when you've graded a single
URL directly (Flow 2), since that flow has to *find* the matching Google listing rather than
starting from one.

| Column | Meaning |
|---|---|
| `google_listing_matched` | `TRUE`/`FALSE`. Whether a Google Maps listing was successfully matched to the submitted website. If `FALSE`, treat `review_count`, `rating`, `phone`, and `address` on that row with real caution — they may be missing or, worse, wrong (matched to the wrong business). |
| `name_source` | How the business name was determined, in priority order: `user_override` (you typed it in) → `schema_org` (found in the site's own structured data) → `og_site_name` (a meta tag) → `title_tag` (the page `<title>`) → `domain_fallback` (guessed from the URL itself, the least reliable option). |

**Rule of thumb:** the further down that priority list `name_source` is, the more worth
double-checking the row is before using it — `domain_fallback` in particular is a guess, not a
verified name.
