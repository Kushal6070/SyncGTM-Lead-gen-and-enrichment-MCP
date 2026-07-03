---
name: daily-meeting-prep
description: Fully automated pre-meeting recon for every external meeting on today's Google Calendar — finds each attendee's and their company's LinkedIn, pulls 5 recent posts from each, identifies sales/revenue decision makers, checks the company's tech stack for CRM and outbound/sales-engagement tools, checks sales-team headcount and its recent growth plus open sales job postings and their growth, drafts deeper discovery questions, and writes a brief formatted meeting-notes page to the "Sales Calls" Notion database. Trigger this whenever the user asks to "prep today's meetings," "run recon on my calendar," "get me ready for today's calls," "pull intel on today's meetings," or similar — even if they only mention one part of the workflow (e.g. "check LinkedIn for today's meetings") since the full pipeline is the point of this skill. Also trigger for "daily meeting prep," "sales demo prep for today," or "who am I meeting with today and what should I know."
---

# Meeting Recon

Turns "what's on my calendar today" into a short, useful Notion brief for every external meeting, with zero manual lookups. This is the automation-first sibling of `call-prep` — it doesn't ask the user many questions, it goes and gets the answers itself using Google Calendar, SyncGTM, web search, and Notion.

Notes are deliberately **brief** — this is a quick scan before you walk into the room, not a research dossier. Short sections, no filler.

## Required connectors

| Connector | Used for |
|---|---|
| **Google Calendar** | Pulling today's meetings and attendees |
| **SyncGTM** | LinkedIn discovery, LinkedIn post scraping, decision-maker search, tech stack, sales headcount/growth/job data |
| **Notion** | Writing the final meeting notes page |
| **Web search** | Filling gaps SyncGTM doesn't cover (e.g. "what they do" if not obvious from enrichment) |

If any of these aren't connected, tell the user which piece will be skipped and continue with the rest — don't block the whole run on one missing connector.

## Notion destination (fixed)

Meeting notes always go to the **"Sales Calls"** database:

- Database URL: `https://app.notion.com/p/zekyaa/215db8b6409a80fea6c3f4bfc73bf94e`
- Data source ID: `215db8b6-409a-8091-87ec-000b312362fc` — pass this as `parent: {"type": "data_source_id", "data_source_id": "215db8b6-409a-8091-87ec-000b312362fc"}` in `notion-create-pages`.
- Schema: `Name` (title), `Date` (date), `Company` (text). Map: `Name` = meeting title (e.g. "Demo — Acme Corp"), `Company` = company name, `date:Date:start` = the meeting's start date, `date:Date:is_datetime` = 1 if the event has a specific time.
- There are page templates on this database (GTM/Sales Engineer, Agencies, Sales teams) — these are discovery-question templates, not note formats. Don't apply them by default; only use one if the user explicitly asks for it.
- If this database is ever deleted or the user asks to file notes elsewhere, ask where and update this section of the skill for next time — don't silently default to standalone pages.

---

## Execution Flow

### Step 0 — Establish "today"

Call `user_time_v0` to get the current date and timezone. Use this to build the `startTime`/`endTime` window for the calendar query (midnight to midnight, user's local timezone) — do not assume UTC or guess the date.

Also note the user's own email domain (from their primary calendar ID, usually their email address). You'll use it in Step 2 to tell internal attendees apart from external ones.

### Step 1 — Pull today's meetings

Call `Google Calendar:list_events` with `startTime`/`endTime` set to today's window (from Step 0) and `orderBy: startTime`. Pull the full attendee list, title, and description for each event.

**Filter to events worth researching:** skip events with no external attendees (1:1s with yourself, internal team syncs, personal blocks like "lunch" or "focus time"). An event is "external" if at least one attendee's email domain differs from the user's own domain.

If there are zero external meetings today, tell the user that and stop — don't fabricate a report.

### Step 2 — Identify the company + attendee per meeting

For each external meeting:
1. Take the external attendee(s)' email(s). Group by company domain if a meeting has multiple people from the same company.
2. Derive the company domain from the email (part after `@`).
3. Call `SyncGTM:enrich_organization` with that domain to get the company name, LinkedIn URL, industry, size, and a short description (this covers "what they do"). If it doesn't return a LinkedIn URL, fall back to `SyncGTM:enrich_linkedin_page` using the company name.
4. If `enrich_organization` doesn't give a clear one-liner on what the company does, do one `web_search` for `"[Company name]"` and pull it from their site/description — don't skip "what they do" for lack of a single tool call.

If a meeting has multiple external attendees from different companies, treat it as one meeting but research each company/attendee separately within the same notes page.

### Step 3 — Find the prospect's LinkedIn + background

For each external attendee:
- Call `SyncGTM:find_linkedin_from_work_email` with their email to get their profile URL.
- If that misses, try `SyncGTM:enrich_person` with the email as a fallback.
- Once you have the URL, call `SyncGTM:linkedin_profile_enrich` to get their title, **school/education**, and **current city/location** — both needed for the "About the prospect" section.

Skip gracefully (note "LinkedIn not found") if a lookup misses rather than guessing a profile or school.

### Step 4 — Scrape LinkedIn posts (5 each)

For each attendee with a resolved profile:
- `SyncGTM:linkedin_profile_posts` with `profile_url` and `max_posts: 5`.

For each company:
- `SyncGTM:linkedin_page_posts` with `company_url` and `max_posts: 5`.

You only need the **prospect's** posts for the notes ("what they posted"). The company posts are optional context — use them only if they sharpen the overview or a post is directly relevant; don't pad the notes with company post summaries. Summarize themes, don't quote at length.

### Step 5 — Find sales & revenue decision makers

Using the company domain from Step 2, call `SyncGTM:find_people` with:
- `current_company_domains: [domain]`
- `current_functions: ["Sales"]`
- `seniority_levels: ["c_suite", "vp", "director"]`
- `limit: 8`

This targets people who actually own budget/buying decisions — not every individual rep. If it comes back empty (small company, sparse data), fall back to `SyncGTM:find_people_within_company` with `domain` and `job_title` set to common revenue/sales leadership titles, e.g. `["VP Sales", "Chief Revenue Officer", "CRO", "Head of Sales", "Director of Sales", "VP Revenue", "Chief Sales Officer", "SVP Sales"]`, `max_profiles: 8`.

Pull just **name + title** for each — this is a scan list for the notes, not a full enrichment pass. Don't run `linkedin_profile_enrich` on each of them (that's reserved for the actual meeting attendee in Step 3).

If the meeting's own attendee is already one of these decision makers, still list them here too — useful to see them in context with peers rather than deduped out.

### Step 6 — Tech stack: CRM & outbound tools

Call `SyncGTM:find_company_techstack` with the company domain. The tool returns the company's broader tech stack (analytics, dev tools, marketing, etc.) — filter it down to just what's relevant for a sales conversation:

- **CRM tools to look for:** Salesforce, HubSpot, Pipedrive, Zoho CRM, Close, Copper, Microsoft Dynamics, Freshsales, Attio.
- **Outbound / sales-engagement tools to look for:** Outreach, Salesloft, Apollo.io, Instantly, Smartlead, Lemlist, Reply.io, Mailshake, Woodpecker, Clay, ZoomInfo, Groove.

Report only the tools actually found in the returned stack — don't list the categories as unfilled placeholders, and don't guess a tool isn't there just because it's not a common name; scan the full returned list, not just the examples above. If nothing in either category shows up, say "no CRM/outbound tools detected" rather than omitting the section silently — that's itself a useful signal (e.g. they may be using something homegrown or nothing at all).

This is a strong signal for the conversation: knowing their current CRM/outbound stack tells you what you're replacing or integrating with, and is exactly the kind of thing worth asking about directly if it's unclear (see Step 9).

### Step 7 — Sales headcount + growth

Use the company's LinkedIn page identifier from Step 2.

1. `SyncGTM:headcount_by_department` with `department: "Sales"` → current sales headcount.
2. `SyncGTM:premium_linkedin_page_insights` → check for department-level hiring/growth trends; if it includes a sales/function breakdown, use that for "recent growth." If it doesn't break out by department, fall back to `SyncGTM:head_count_growth_rate` (company-wide) and label it clearly as company-wide, not sales-specific.

### Step 8 — Sales job openings + growth

1. `SyncGTM:company_job_openings` with the company's `company_linkedin_url` (or `company_domain`) and `job_title` filtered to sales roles, e.g. `["Sales", "Account Executive", "SDR", "BDR", "Sales Development Representative", "Account Manager", "Sales Manager", "VP Sales"]` → count of currently open sales roles.
2. `SyncGTM:job_openings_growth_rate` → job-openings growth trend. This is company-wide (the tool doesn't filter by department) — label it as such, but it's still a useful directional signal alongside the sales-specific open-role count from step 1.

### Step 9 — Synthesize + draft discovery questions

Combine everything from Steps 1–8 into the Output Format below, one section per meeting. Keep every bullet to one line where possible — this is a brief, not a report. Skip a section entirely (rather than writing "not found" filler) if nothing relevant turned up, except the two growth metrics and the decision-makers list, which should always show data or an explicit "no data available."

Then draft **3–5 discovery questions** that go deeper than small talk, each grounded in something you actually found — not generic sales-call questions. Good sources for these:
- A gap or ambiguity in the tech stack (e.g. no CRM detected, or an outbound tool that's commonly paired with a pain point your product solves)
- A growth signal (e.g. fast sales headcount growth → ask how they're onboarding/ramping new reps; lots of open sales roles → ask what's driving the hiring push)
- Something the prospect posted about recently
- The presence (or absence) of other decision makers found in Step 5 — e.g. asking who else is typically involved in evaluating a new tool

Avoid generic questions that don't use any researched detail (e.g. "What are your goals for this quarter?") — every question here should read as if you clearly did your homework.

### Step 10 — Write to Notion

Call `Notion:notion-create-pages` with the parent set to the Sales Calls data source (see **Notion destination** above).

- **One page per meeting** (not one page for the whole day) — `Name` = `[Meeting title]`, `Company` = company name, `Date` = the meeting's date.
- Use the Output Format below as the page content, in Notion-flavored markdown. Read the `notion://docs/enhanced-markdown-spec` resource first if you haven't already in this session.

After creating the pages, give the user a short summary in chat (not the full content again) — meeting count, companies, and a one-line link list — since the notes now live in Notion.

---

## Output Format (per meeting — used as the Notion page content)

Keep this brief — short sections, no extra headers, no talking-point fluff or general news unless the user asks for it back.

```markdown
## [Meeting Title] — [Time]

**Attendee:** [Name, Title] · **Company:** [Name]

### Overview
[1–2 sentences: industry, size, HQ]

### What they do
[1–2 sentences on their product/business]

### About the prospect
- **[Name]** — [Title]
- School: [from LinkedIn, or omit if not listed]
- City: [from LinkedIn, or omit if not listed]
- What they've been posting: [1–2 line synthesis of themes across their last 5 posts]

### Decision makers (Sales & Revenue)
- [Name] — [Title]
- [Name] — [Title]
- [...]

### Tech stack (CRM & outbound)
- CRM: [tool(s) found, or "none detected"]
- Outbound/sales engagement: [tool(s) found, or "none detected"]

### Sales headcount & growth
- Current sales headcount: [#]
- Recent growth: [X% / trend — note if this is company-wide vs. sales-specific]

### Sales job openings & growth
- Open sales roles: [#]
- Job-openings growth: [X% over Y months — company-wide]

### Questions to go deeper
1. [Question grounded in a specific finding]
2. [Question grounded in a specific finding]
3. [Question grounded in a specific finding]
```

---

## Edge Cases

- **No external meetings today:** say so plainly, don't run the rest of the pipeline.
- **Attendee email is personal (gmail.com, etc.):** skip company enrichment for that person, note it, still try LinkedIn-from-email for the individual.
- **Recurring/all-day/declined events:** skip declined events; for all-day or "block" style events with no real attendees, skip them.
- **LinkedIn lookups miss:** never fabricate a profile URL, school, city, or post content — omit the line and move on.
- **No decision makers found:** say "no decision makers found for this domain" rather than fabricating names/titles.
- **No CRM/outbound tools detected:** say so explicitly — it's a useful signal, not a gap to hide.
- **No sales-department data available:** some companies' LinkedIn pages don't break out department headcount. Say "no department data available" rather than substituting total company headcount without a label.
- **Low SyncGTM credits:** these calls have real credit costs (roughly: 2 credits per person LinkedIn lookup, 3 credits per `find_people` decision-maker search, 1.5 credits for the `find_people_within_company` fallback, 1 credit per techstack lookup, 0.5–3 credits per company call, 0.3 credits per posts pull, 5 credits per job-openings pull). If a run covers many meetings/attendees, mention the rough credit spend before running.

---

## Tips

- This is meant to run first thing in the morning, once, for the day. Re-running later the same day should skip meetings already researched (check the Sales Calls database for an existing page with a matching Name/Date before re-creating).
- If the user only wants *one* meeting prepped (e.g. "just prep the Acme call"), skip Step 1's filtering and just process that single event — no need to touch the rest of the day.
- If the user asks to bring back recent company news, that's a one-line addition to Step 9 / Output Format — don't reinvent the whole flow, just extend it.

---

## Related Skills

- **call-prep** — lighter-weight, question-driven version of this for a single ad-hoc call when you don't want the full automated pipeline
- **account-research** — deeper standalone company dive, useful before first contact rather than day-of
- **call-summary** — run after the meeting to process notes/transcript and draft follow-up
