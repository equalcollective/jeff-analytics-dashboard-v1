# Jeff Analytics — Claude Code context

## Dashboards in this folder

This folder serves two dashboards:

1. **Analytics dashboard** (`dashboard.html`) — sales funnel, goal pacing, week-over-week, series performance. Powered by the Smartlead MCP. **This is the default.** Specs live in this file (Section: "Dashboard").
2. **Deliverability dashboard** (`deliverability.html`) — bounce-pattern tracking (A / B / C) for EQU-328 RCA. Powered by a direct Postgres query. Specs live in [`deliverability.md`](deliverability.md).
3. **Monthly Analytics dashboard** (`monthly.html`) — month-over-month funnel, monthly goal pacing, MoM change chips, funnel diagnostic. Powered by the Smartlead MCP. Low-frequency companion to dashboard #1. Specs live in [`monthly.md`](monthly.md).

Routing rules:
- **"update dashboard"** (default) → update the analytics dashboard using the specs below.
- **"update deliverability dashboard"** (or any phrase containing "deliverability") → follow [`deliverability.md`](deliverability.md) instead. Do not touch `dashboard.html`.
- **"update the Monthly Analytics dashboard"** (or any phrase containing "monthly") → follow [`monthly.md`](monthly.md) instead. Do not touch `dashboard.html` or `deliverability.html`.

`deliverability.md`, `queries/`, and `.env*` are gitignored — they contain a DB credential and internal client IDs. Never `git add` them.

---

You are helping the sales lead run outbound analytics for two businesses on the Jeff platform.

## Business context

Two businesses, one platform:

- **Jeff** — an outbound sales platform that runs cold email campaigns via Smartlead for Amazon marketing agencies. Multiple clients.
- **MerchantBots** — our own Amazon marketing agency. Commission-only, 4–6% of gross sales, targets early-stage sellers under $30,000/month revenue. Jeff is the outbound engine powering MerchantBots sales.

Strategic direction: phasing out external Jeff clients over time, scaling MerchantBots. Eventually Jeff becomes MerchantBots' internal sales tool.

## Client list

### MerchantBots clients — current (primary focus)

| Client | ID | Role | MB since |
|---|---|---|---|
| Equal Collective | 108917 | Parent company — IS MerchantBots | always |
| DTC Retail | 82914 | Originated the MerchantBots model | always |
| Nexus | 51431 | Infrastructure partner — sends volume, no meetings | 2026-03-30 |
| Mr. Prime | 33748 | Infrastructure partner — sends volume, no meetings. Being phased out. | 2026-04-15 |

**Query filter for MerchantBots (current state):** `client_id=108917,82914,51431,33748`

Infrastructure partners (Nexus, Mr. Prime) contribute to volume metrics (prospects_reached, emails_sent, replies) but naturally return 0 for outcome metrics (meetings, positive replies). No need for separate queries.

### Historical client transitions

For analyses that span the transition dates, attribute volume per the client's status at that time. CRM outcomes (positive replies, meetings) are agency-attributed in the source of truth and do not need rewriting.

| Client | ID | Was | Became MB | Notes |
|---|---|---|---|---|
| Nexus | 51431 | Jeff client | 2026-03-30 (Mon) | Clean week-start cutover. For weekly windows, exclude Nexus from MB total in any week ending on or before 2026-03-29; include from week of 2026-03-30 onward. |
| Mr. Prime | 33748 | Jeff client | 2026-04-15 (Wed) | Mid-week cutover. The Mon-Sun week of 2026-04-13 to 2026-04-19 straddles the transition: Apr 13–14 was Jeff, Apr 15–19 was MB. |

**MerchantBots series:**

- **Active:** 6, 10, 11, 13
- **Paused (historical, still in data):** 15 — launched as MB, later paused. Include for retrospective analyses, exclude from "current MB series" filters.
- **Series 13** — outreach to leads already existing in a CRM.

When attributing infrastructure-partner sends to MB pre-transition (or as a sanity check), filter to series IN (6, 10, 11, 15) AND respect the transition date — Series 6 was also used by Nexus/Mr. Prime for Jeff customers before they migrated, so series alone is not sufficient.

When a series transitions: append to the lists above with the date. A new series launched as a replacement (e.g. Series 21 replacing Series 10) → mark predecessor as paused rather than removing it from the data list.

DTC Retail (82914) and Equal Collective (108917) have always been MerchantBots — no transition handling needed. All other clients (AMZ Ads, Riverguide, Accelo Brand) have always been Jeff and have not transitioned.

### Jeff clients (secondary tracking)

| Client | ID | Goal |
|---|---|---|
| AMZ Ads | 25946 | 3 meetings/week |
| Riverguide | 25948 | 3 meetings/week |
| Accelo Brand | 223329 | 3 meetings/week |

**Query filter for Jeff clients:** `client_id=25946,25948,223329`

## Goals

- **MerchantBots: 30 meetings/week and 100 positive replies/week.** These are the primary goals. Goal pacing banner always shows both.
- **Jeff clients: 3 meetings/week per client** (9 total across 3 clients). Secondary tracking in the Jeff Clients tab.

## MCP tools available

The **Jeff Analytics MCP** exposes the outbound funnel:
- `smartlead_metric_definitions` — full catalog of metrics, dimensions, filters. Call this FIRST in any session where you're unsure what's queryable.
- `smartlead_query_funnel` — the workhorse. CSV output. Supports grouping and filtering.
- `smartlead_list_clients`, `smartlead_list_campaigns`, `smartlead_list_domains` — for resolving IDs to names.

## The funnel

```
prospects_reached
   ↓  (deliverability)
emails_delivered
   ↓  (engagement — genuine_reply_rate is the proxy)
crm_positive_replies  (CRM is source of truth for outcomes)
   ↓  (conversion)
crm_meetings_booked   (CRM is source of truth for outcomes)
```

### Source of truth rules
- **CRM metrics are always the primary metrics** for positive replies and meetings booked. Use `crm_positive_replies` and `crm_meetings_booked` in all dashboards and analyses.
- **Genuine reply rate** (`genuine_reply_rate` = genuine_replies / prospects_reached) is the engagement indicator at the emails_delivered step — human replies only. The dashboard uses `genuine_replies` / `genuine_reply_rate`. `total_replies` / `reply_rate` count any reply including automated auto-responders, so they are not the engagement signal.

### Key rates
- `crm_booking_rate` = crm_meetings_booked / prospects_reached. End-to-end conversion. **This is the north star.** Computed client-side from counts.
- `crm_positive_reply_rate` = crm_positive_replies / prospects_reached. Signal quality. Computed client-side from counts.
- `genuine_reply_rate` = genuine_replies / prospects_reached. Engagement rate — human replies only.
- `email_1_reply_rate` = genuine_replies_email_1 / email_1_sent. Genuine reply rate on the opening (email 1) message only — computed client-side, there is no prebuilt MCP rate for it. Always rendered directly next to the genuine reply rate (daily, weekly, and series tables) so the opener's pull can be read against the all-steps genuine reply rate. Exploratory metric — looking for inferences between opener engagement and the overall funnel.


## Dimensions that matter

- `series` / `sub_series` — different campaign archetypes. **Series 13 is currently the breakout** (~10x booking rate vs. others as of week of Apr 6). Always segment by series when analyzing.
- `client_id` — the Smartlead client running the campaign.
- `domain` — sending domain. Matters for deliverability debugging.
- `sequence_number` — which step of the email sequence. Useful for deciding if follow-ups are earning their keep.
- `reply_class` — splits reply metrics into genuine / auto_reply / bounce. Use to break `total_replies` into `genuine_replies` vs `auto_replies`.
- `bounce_bucket` — breaks `smtp_bounces` into its 8 reasons (recipient_dead, domain_dead, mailbox_full, sender_block, content_block, transient, other_permanent, unknown). Slice by `domain` to separate sender-reputation problems from list-quality problems.
- Recipient/inbox slices — `seller_revenue_bucket`, `seller_address_country_segment`, `email_address_type`, `email_domain_type`, `age_at_send_band` (inbox age at send). Slice the funnel and `crm_*` by who was contacted and how mature the sending inbox was. With `crm_*`, `age_at_send_band` requires `date_mode=cohort` and a non-`all` granularity.

## Date modes — don't get these wrong

- `activity` (default): each event bucketed by its own timestamp. Use for "what happened on day X".
- `cohort`: all downstream events bucketed by the lead's email_1 send date. Use for "of leads we contacted in week X, how many eventually converted". **Cohort mode is the right call when measuring conversion rate trends** because it avoids the lag artifact where this week's positive replies are mostly from last week's sends.

## Granularity rules
- `daily` — any dates
- `weekly` — date_from must be Monday, date_to must be Sunday
- `monthly` — date_from must be 1st, date_to must be last day

## How to work with the user

- **`git pull` before any local edit.** The daily dashboard routine pushes `dashboard.html` and `dashboards/YYYY-MM-DD-HHMM.html` to GitHub on its own schedule, so the local clone is often behind. Pull first to avoid stale edits and merge conflicts. (This applies to local Claude Code sessions only — the cloud routine works in a fresh clone and can skip this.)
- **Run the query first, then talk.** Don't ask permission for read-only queries. Just pull the data.
- **Surface anomalies unprompted.** If one series is 10x others, say so even if not asked.
- **Always contrast.** Every number is meaningless without a comparison — vs. last week, vs. other series, vs. goal.
- **Be direct about broken data.** PostHog metrics are 0 — when you see that, call it out.
- **No apologies, no hedging.** "The booking rate dropped" not "it appears the booking rate may have dropped".

## Dashboard

### Hosting and update workflow
- **GitHub repo**: `equalcollective/jeff-analytics-dashboard-v1` (public)
- **Team URL**: `https://equalcollective.github.io/jeff-analytics-dashboard-v1/dashboard.html`
- **Permanent file**: `dashboard.html` at the repo root — this is what the team sees. Always overwrite this file with the latest data.
- **Archive**: `dashboards/YYYY-MM-DD-HHMM.html` — snapshot copy, named with the build's IST date and 24-hour IST time (e.g. `dashboards/2026-07-16-1545.html`). Create one each time the dashboard is updated; the routine runs several times a day, so the time in the filename keeps each run distinct.

### Update process ("update dashboard")
1. Pull fresh data from MCP for all sections per the specs below.
2. Overwrite `dashboard.html` with the new data.
3. Save an identical copy in `dashboards/` named `YYYY-MM-DD-HHMM.html` with the build's IST date and 24-hour IST time.
4. Git add, commit, and push to `origin/main`.
5. GitHub Pages auto-deploys within ~1 minute. Team refreshes to see new data.

When the user says "show analytics" or similar vague requests, run the queries described below and present the data conversationally.

### Dashboard sections

Each section below defines exactly what to query and how to display it. When building or updating the dashboard, follow these specs precisely.

#### Section 1: Goal Pacing Banner (always visible at top, MerchantBots only)
- **Query**:
  - Metrics: `crm_meetings_booked,crm_positive_replies`
  - Granularity: `daily`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: current week Monday through today
  - Date mode: `activity`
- **Display**:
  - Two large numbers side by side: **Meetings This Week** and **Positive Replies This Week** (sum across all days returned)
  - Progress bar under meetings showing progress toward 30/week goal
  - Progress bar under positive replies showing progress toward 100/week goal
  - Status badge on each: ahead (green) / on pace (amber) / behind (red) based on where the count stands relative to the day of the week

#### Section 2: Daily Trend (tab, MerchantBots)

##### Sub-section A: Daily Summary Table (toggle: activity / cohort mode)
- **Query (activity mode)**:
  - Metrics: `prospects_reached,total_emails_sent,genuine_replies,email_1_sent,genuine_replies_email_1,crm_positive_replies,crm_meetings_booked,genuine_reply_rate`
  - Granularity: `daily`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: last 16 days through yesterday
  - Date mode: `activity`
- **Query (cohort mode)**: same parameters but `date_mode=cohort` — buckets positive replies and meetings by each lead's email_1 send date (conversion-by-send-cohort). Days with no email_1 sends (e.g. weekends) return no row; render them as zero-value rows so the 16-day axis stays continuous.
- **Display**:
  - Toggle switch at top: Activity / Cohort. Swaps the entire table data. Label clearly which mode is active.
  - 16 rows, **sorted most recent date at top**
  - Columns: Date, Day, Prospects Reached, Emails Sent, Genuine Replies, Positive Replies (with inline bar chart), Meetings, Genuine Reply Rate, Email 1 Reply Rate, Positive Reply Rate
  - Positive Reply Rate = `crm_positive_reply_rate` (crm_positive_replies / prospects_reached), computed client-side from counts
  - Email 1 Reply Rate = genuine_replies_email_1 / email_1_sent (computed client-side), placed directly next to Genuine Reply Rate
  - No booking rate column
  - Inline bar chart on the Positive Replies column (scaled to max day)
  - Weekends muted
  - Highlight yesterday's row (most recent) so it reads as the "what happened yesterday" anchor — this absorbs the role of the deleted Yesterday's Snapshot sub-section
  - Highlight any day with positive replies >2x the median

##### Sub-section B: Sequence Step Breakdown
- **Query**:
  - Metrics: `email_1_sent,email_2_sent,email_3_sent,crm_positive_replies`
  - Granularity: `daily`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: last 16 days through yesterday
  - Date mode: `activity`
- **Display**:
  - Table with one row per day, sorted most recent at top
  - Columns: Date, Email 1 Sent (absolute), Email 2 Sent (absolute), Email 3 Sent (absolute), Stacked % Bar (showing percentage split of email 1/2/3 with labels like "58% / 28% / 14%"), Positive Replies
  - The stacked bar shows the mix visually; the absolute numbers show the volume
  - Positive Replies column placed next to the stacked bar so the user can eyeball correlation between email 1 volume/mix and positive reply count
- **Business question answered**: "Are we sending enough email 1s? On days with higher email 1 mix, do we get more positive replies?"

#### Section 3: Weekly Review (tab, MerchantBots)

##### Sub-section A: Week-over-Week Table (toggle: activity / cohort mode)
- **Query (activity mode)**:
  - Metrics: `prospects_reached,unique_prospects_reached,total_emails_sent,genuine_replies,smtp_bounces,email_1_sent,genuine_replies_email_1,crm_positive_replies,crm_meetings_booked,genuine_reply_rate,smtp_bounce_rate,reply_to_crm_positive_rate`
  - Granularity: `weekly`
  - Client IDs: `108917,82914,51431,33748`
  - Dates: last 10 complete weeks (Mon–Sun) + current partial week
    - **Important — apply transition-date adjustments before displaying totals.** The client_id filter naively includes Nexus (51431) and Mr. Prime (33748) across the whole range, but they only became MerchantBots on 2026-03-30 (Nexus, clean Monday cutover) and 2026-04-15 (Mr. Prime, mid-week cutover). Three cases per client:
        - **Nexus (51431):**
          - Week ending ≤ 2026-03-29 → **exclude Nexus entirely** (pre-transition Jeff volume).
          - Week of 2026-03-30 onward → **include Nexus** (clean Monday cutover, no partial-week math needed).
        - **Mr. Prime (33748):**
          - Week ending ≤ 2026-04-12 → **exclude Mr. Prime entirely** (pre-transition Jeff volume). This was the gap that caused the May 2026 dashboard build to over-count Mar 30–Apr 5 and Apr 6–12 prospects by ~12k–13k each — query Mr. Prime separately for the week and subtract from the 4-client total.
          - Week of 2026-04-13 to 2026-04-19 (straddles transition) → **subtract only Mr. Prime's Apr 13–14 volume** (those two days were still Jeff; Apr 15–19 was MB). Pull a daily query for Mr. Prime alone over the week and subtract Apr 13–14.
          - Week of 2026-04-20 onward → **include Mr. Prime** (fully MB).
        - **Apply adjustments to volume metrics only**: prospects_reached, total_emails_sent, genuine_replies, smtp_bounces, email_1_sent, genuine_replies_email_1. CRM outcomes (crm_positive_replies, crm_meetings_booked) are agency-attributed in the source of truth and do not need rewriting.
        - **Label adjusted weeks** in the rendered table with a small parenthetical: e.g. "(3 clients, no MrP)", "(MrP adj.)", "(Nexus joined Mar 30)", "(maturing)". The label must accurately reflect the math — do NOT label a row "no MrP" if the displayed number is the 4-client total.
        - See the "Historical client transitions" table at the top of this doc for canonical dates.
  - Date mode: `activity`
- **Query (cohort mode)**: same parameters but `date_mode=cohort`
- **Display**:
  - Toggle switch at top: Activity / Cohort. Swaps the entire table data.
  - Table with most recent week at top
  - Columns: Week, Prospects, Unique Prospects, Emails, Bounce Rate, Genuine Replies, Genuine Reply Rate, Email 1 Reply Rate, Positive Replies, Pos. Reply Rate, Pos. Reply Rate (unique), `reply_to_crm_positive_rate`, Meetings, Booking Rate, Pos. Reply to Booking Rate
  - Bounce Rate = smtp_bounces / prospects_reached (from API as `smtp_bounce_rate`, or compute client-side from counts). System-detected SMTP bounce NDRs (reply_class='bounce'), not the rep-tagged category.
  - Pos. Reply Rate = `crm_positive_reply_rate` (crm_positive_replies / prospects_reached), computed client-side from counts
  - Unique Prospects = `unique_prospects_reached` (distinct leads deduped on lead_id — non-additive, do not sum across weeks or group-bys)
  - Pos. Reply Rate (unique) = crm_positive_replies / unique_prospects_reached, computed client-side from counts
  - Booking Rate = `crm_booking_rate` (crm_meetings_booked / prospects_reached), computed client-side from counts
  - `reply_to_crm_positive_rate` = crm_positive_replies / genuine_replies (from API as `reply_to_crm_positive_rate`, or compute client-side from counts). Caveat: numerator is agency-attributed while denominator is campaign-attributed, so the ratio is a directional signal of reply quality, not a precise per-client conversion.
  - Pos. Reply to Booking Rate = crm_meetings_booked / crm_positive_replies (calculated in HTML, not from API)
  - Label clearly which mode is active
- **Saturday rule**: when building on a Saturday, treat the current Mon–Fri as the most recent week (it is complete for sending purposes)

##### Sub-section B: Trends (toggle: activity / cohort mode)
- **Query (activity mode)**: reuse sub-section A's activity-mode data (10 complete weeks + current partial week) — no new query
- **Query (cohort mode)**: reuse sub-section A's cohort-mode data (same window)
- **Display**:
  - Toggle switch at top: Activity / Cohort. Swaps the data feeding both charts. Label clearly which mode is active.
  - Two charts stacked, both plotting the same weekly series sub-section A already pulls; weeks on the x-axis (oldest left, most recent right).
  - **Chart 1 — Line chart (rate/metric trends, amplitude-normalized):**
    - Series: Prospects Reached, Genuine Reply Rate, SMTP Bounce Rate, Pos. Reply Rate, Booking Rate.
    - Each series is min–max normalized independently (amplitude scaling): rescaled to fill the chart height so a small-magnitude rate shows the same visual swing as prospects. This makes each line's trend-over-time legible. It is NOT a shared axis — do not read one line sitting higher than another as meaningful; only each line's own shape over time is.
    - No numeric y-axis (the normalized values are synthetic) — label the axis "relative — hover for true value".
    - Hover any point shows the TRUE underlying value in real units: counts with commas (Prospects); percentages for the four rates (Genuine Reply / SMTP Bounce to 2 dp, Pos. Reply / Booking to 3 dp).
    - Interactive legend: each series is a chip that toggles its line on/off. Each chip also shows the series' real min–max range over the window for magnitude context (e.g. "Booking Rate 0.031%–0.149%").
  - **Chart 2 — Bar chart (absolute counts):**
    - Series: Prospects Reached, Genuine Replies, Bounces (smtp_bounces), CRM Positive Replies, Meetings Booked.
    - Grouped bars per week on a log-scale y-axis (counts span ~15,000 down to ~15, so a linear axis buries the small series; log keeps all visible at once). Do NOT stack — these are funnel subsets (positives ⊂ replies ⊂ prospects), not additive parts, so a stack would imply a false sum.
    - Interactive legend toggles each series on/off; hover shows the true count.
  - Both charts follow the Activity / Cohort toggle above (same data source as sub-section A, rendered differently).
- **Business question answered**: "How are the funnel rates and volumes trending week over week — which are climbing or sliding?"

##### Sub-section C: Funnel Diagnostic (toggle: activity / cohort mode)
- **Query (activity mode)**: reuse sub-section A's activity-mode data (most recent week + 6-week averages)
- **Query (cohort mode)**: reuse sub-section A's cohort-mode data (most recent week + 6-week averages)
- **Display**:
  - Toggle switch at top: Activity / Cohort. Swaps the entire funnel data. Label clearly which mode is active.
  - Vertical funnel visualization (styled like the ICAP funnel image — colored boxes with arrows between them)
  - 4 steps top to bottom:
    1. **Prospects Reached** — this week's count vs 6-week avg count
    2. **Emails Delivered** — genuine reply rate this week vs 6-week avg genuine reply rate
    3. **Positive Replies** — positive reply rate this week vs 6-week avg
    4. **Meetings Booked** — booking rate this week vs 6-week avg
  - Each box shows: this week's rate, 6-week avg rate, and a status color:
    - Green: this week >= 6-week avg
    - Amber: this week is 10-30% below 6-week avg
    - Red: this week is >30% below 6-week avg
  - Also show Pos. Reply to Booking Rate as a label between steps 3 and 4
- **Business question answered**: "At a glance, where in the funnel did we win or lose this week?"

##### Sub-section D: Series Performance Table (activity mode)
- **Query**:
  - Metrics: `prospects_reached,total_emails_sent,genuine_replies,smtp_bounces,email_1_sent,genuine_replies_email_1,crm_positive_replies,crm_meetings_booked,genuine_reply_rate,smtp_bounce_rate,reply_to_crm_positive_rate`
  - Granularity: `weekly`
  - Client IDs: `108917,82914,51431,33748`
  - Group by: `series,sub_series` — one row per series + sub-series combination (e.g. 6C, 6E, 11B, 11C). Grouping by `sub_series` alone is wrong: it merges the same letter across different series (6C and 11C collapse into one "C"). The combined grain keeps series identity while splitting each series into its active sub-series.
  - Dates: current partial week + last 4 complete weeks (pull the current partial week and each complete week individually to support the toggle buttons)
  - Date mode: `activity`
- **Display**:
  - 5 toggle buttons: **Current week** | **Last 1 week** | **Last 2 weeks** | **Last 3 weeks** | **Last 4 weeks**
    - **Current week** = the current partial week-to-date alone (Mon of this week through today), one row per series + sub-series active this week. This is the primary view for series testing — it surfaces a newly-launched series the moment it starts sending, even mid-week (this is exactly why series 13 was invisible under the old "last 4 complete weeks" rule). Label the button/header "Current week (partial)" while the week is incomplete.
    - **Last 1 / 2 / 3 / 4 weeks** each aggregate that many most-recent *complete* weeks (Mon–Sun) into one row per series + sub-series. They do NOT include the current partial week.
  - **Never gray, mute, or fade the current-week view** — it is the most important week. If partial, write "(partial)" in the label only; do not dim it. (See the global design rule.)
  - Table columns: Series · Sub-series (label each row by its combined identity, e.g. "11B"), Prospects, Emails, Bounce Rate, Genuine Replies, Genuine Reply Rate, Email 1 Reply Rate, Positive Replies, Pos. Reply Rate, `reply_to_crm_positive_rate`, Meetings, Booking Rate, Pos. Reply to Booking Rate
  - Bounce Rate = smtp_bounces / prospects_reached (works correctly at series + sub-series level — no CRM dependency)
  - Pos. Reply Rate = `crm_positive_reply_rate` (crm_positive_replies / prospects_reached); Booking Rate = `crm_booking_rate` (crm_meetings_booked / prospects_reached) — both computed client-side from counts
  - `reply_to_crm_positive_rate` = crm_positive_replies / genuine_replies. Same agency-vs-campaign attribution caveat as Section 3A — directional signal of reply quality, not a precise per-client conversion.
  - Pos. Reply to Booking Rate = crm_meetings_booked / crm_positive_replies (calculated in HTML)
  - Sorted by booking rate descending
  - **All rows rendered in full white** — do NOT fade out series/sub-series with 0 meetings. Every row stays equally readable so the user can see zero-meeting performance side-by-side with high performers.
  - **Use CRM metrics** (`crm_positive_replies`, `crm_meetings_booked`) — they populate correctly at series + sub-series level.
- **Business question answered**: "Which series/sub-series should I scale this week? Is the performance real or a fluke?"

#### Section 4: Jeff Clients (tab, activity mode only)

##### Sub-section B: Combined Jeff Performance (last 4 weeks)
- **Query**:
  - Metrics: `prospects_reached,total_emails_sent,genuine_replies,smtp_bounces,email_1_sent,genuine_replies_email_1,crm_positive_replies,crm_meetings_booked,genuine_reply_rate,smtp_bounce_rate,reply_to_crm_positive_rate`
  - Granularity: `weekly`
  - Client IDs: `25946,25948,223329`
  - Dates: last 4 complete weeks (Mon–Sun) + current partial week
  - Date mode: `activity`
- **Display**:
  - All Jeff clients clubbed into one aggregate row per week (not individual client rows)
  - Most recent week at top
  - Columns: Week, Prospects, Emails, Bounce Rate, Genuine Replies, Genuine Reply Rate, Email 1 Reply Rate, Positive Replies, Pos. Reply Rate, `reply_to_crm_positive_rate`, Meetings, Booking Rate, Pos. Reply to Booking Rate
  - Bounce Rate = smtp_bounces / prospects_reached (from API as `smtp_bounce_rate`, or compute client-side from counts). System-detected SMTP bounce NDRs (reply_class='bounce'), not the rep-tagged category.
  - Pos. Reply Rate = `crm_positive_reply_rate` (crm_positive_replies / prospects_reached), computed client-side from counts
  - Booking Rate = `crm_booking_rate` (crm_meetings_booked / prospects_reached), computed client-side from counts
  - `reply_to_crm_positive_rate` = crm_positive_replies / genuine_replies (from API as `reply_to_crm_positive_rate`, or compute client-side from counts). Same agency-vs-campaign attribution caveat as in Section 3A — directional signal of reply quality, not precise per-client conversion.
  - Pos. Reply to Booking Rate = crm_meetings_booked / crm_positive_replies (calculated in HTML)
- **Business question answered**: "How is the Jeff book of business performing overall week over week?"

### Dashboard design rules
- Dark theme (bg: #0f1117, cards: #1a1d27)
- Self-contained HTML, no external dependencies
- Tabs for navigation, goal banner always visible
- Color coding: green = good/above target, red = bad/below target, amber = warning/watch
- All numbers formatted with commas. Rates as percentages with 2 decimal places.
- **No narrative text on the dashboard.** Do NOT write commentary, insight callouts, per-section summary boxes, goal-pacing prose, "last updated / Wk N…" recap blocks, or any multi-sentence text blob. The dashboard shows DATA, not analysis. Allowed text is limited to: section/tab titles, short column and axis labels, and compact status badges (e.g. "60% OF GOAL · UNDER PACE"). Everything the reader needs should come from the numbers, bars, and color coding.
  - This explicitly deletes: the goal-banner sub-text under each badge (the per-day breakdown sentences), the top-right "Last updated / Wk N…" recap paragraph (the compact "Updated … IST" timestamp badge stays — see the "Updated at" badge rule below), the green/dark insight boxes at the end of every section, and the small descriptive lines above the Funnel Diagnostic. Remove them all.
  - The Funnel Diagnostic's own boxes (this-week rate vs 6-week-avg with color status) ARE data and stay. Only the prose lines around it go.
- **"Updated at" badge.** A compact badge, top-right, showing the build timestamp as date + 12-hour time in IST: `Updated D Mon YYYY, h:mm AM/PM IST` (e.g. `Updated 16 Jul 2026, 3:45 PM IST`). The cloud routine runs in UTC — convert to Asia/Kolkata (UTC+5:30) and always label `IST`. This compact badge is a status label and is allowed — it is NOT the banned narrative "Last updated / Wk N…" recap.
- **Never gray, mute, fade, or dim the current week.** It is the single most important column/row on the dashboard. When it is partial, indicate that with a "(partial)" label only — never with reduced opacity or a muted style. (Day-level weekend muting in the daily table is unaffected — that still applies to weekend *days*.)

## Spot Checks

After every "update dashboard" run, before declaring done, execute the checks below. Each check independently re-queries MCP and compares the result to what's rendered in `dashboard.html`. If any check fails, stop and fix the build — do not push a failing dashboard.

Spot checks are intentionally narrow and load-bearing — they target the rows most likely to break under spec ambiguity or transition-date math. Add a new check whenever a real bug ships; remove a check only when the underlying logic is encoded in tested code (not narrative spec) and has stayed correct for at least 4 dashboard updates.

### Check 1: Transition-period weekly prospects (Section 3 Sub-section A)

**What it verifies:** the Week-over-Week table correctly applies the Nexus and Mr. Prime transition dates for weekly `prospects_reached`. This is the check that would have caught the May 2026 over-count of ~12k–13k prospects on the Mar 30–Apr 5 and Apr 6–12 rows.

**Procedure:** for each of the four transition-period weeks, run an independent MCP query with the per-week-correct `client_id` list (per the rules in Section 3 Sub-section A), and confirm the rendered `prospects_reached` in `dashboard.html` matches the query result. Date mode: `activity`. Granularity: `weekly`.

| Week | Correct `client_id` filter | Notes |
|---|---|---|
| 2026-03-30 to 2026-04-05 | `108917,82914,51431` | Nexus joined Mon Mar 30 (clean cutover, include). MrP not yet MB (exclude entirely). |
| 2026-04-06 to 2026-04-12 | `108917,82914,51431` | Same as above. MrP still pre-transition. |
| 2026-04-13 to 2026-04-19 | 4-client weekly total **minus** MrP's Apr 13–14 daily volume | Straddle week. Query weekly with `108917,82914,51431,33748`, then query daily for `33748` over that week and subtract the Apr 13 + Apr 14 prospects from the weekly total. |
| 2026-04-20 to 2026-04-26 | `108917,82914,51431,33748` | MrP fully MB from Apr 15. All four clients included. |

**Pass criterion:** rendered value matches the independently computed value exactly for all four weeks.

**On failure:** the build's per-week client_id filter selection is wrong (most likely cause) — fix the build logic, rebuild, re-run the spot check. Do NOT just edit the label; the label and the number must agree.
