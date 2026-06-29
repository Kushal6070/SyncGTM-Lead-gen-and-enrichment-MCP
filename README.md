# SyncGTM MCP

Reference guide for the [SyncGTM MCP server](https://docs.syncgtm.com/mcp_server/tools) — covers all available tools, credit costs, and real outbound use cases.

---

> **Tip:** Install the [sync-mcp agent](./agents/sync.md) for better formatting and experience.

## Prerequisites

- **SyncGTM account** — logged in at [syncgtm.com](https://syncgtm.com)
- **Credits** in your SyncGTM account — [buy credits here](https://app.syncgtm.com/settings/credits/buy)

---

## Getting Started

For installation guides across different AI clients (Claude Code, Claude Desktop, Cursor, Windsurf, and more), see the [SyncGTM MCP documentation](https://docs.syncgtm.com/mcp_server).

---

## Use Case Examples

**1. Enrich a prospect from LinkedIn URL**
Find work email, job title, company info, and recent posts for a prospect before outreach.
→ `find_work_email` + `linkedin_profile_enrich` + `linkedin_profile_posts`

**2. Build a hiring signal list**
Identify companies actively hiring for RevOps or Sales roles — personalize outreach around growth.
→ `company_job_listings` + `enrich_organization`

**3. Score inbound leads by company size**
Check headcount and headcount growth rate for inbound signups to prioritize enterprise accounts.
→ `head_count_growth_rate` + `enrich_organization`

**4. Verify an email list before sending a cold campaign**
Bulk-validate emails to protect sender reputation before launching a sequence.
→ `verify_email`

**5. Find decision-makers at a target account**
Search for VPs and Directors at a company, then enrich with work emails.
→ `find_people_within_company` + `find_work_email`

**6. Research a company's tech stack before outreach**
Know what tools a prospect uses to tailor your pitch (e.g., "we integrate with HubSpot").
→ `find_company_techstack` + `enrich_organization`

**7. Reverse-lookup LinkedIn from a personal email**
Convert a personal email (from a form submission) into a LinkedIn profile for deeper enrichment.
→ `find_linkedin_from_personal_email` + `linkedin_profile_enrich`

**8. Time outreach around job changes and promotions**
Trigger a personalized sequence when a prospect moves to a new role or gets promoted.
→ `check_job_change` + `check_promotions`

**9. Build warm lead lists from social engagement**
Collect people who commented on a competitor's post, then enrich their work emails.
→ `x_post_commenters` + `find_work_email`

**10. Monitor competitor ads and traffic**
See what ads competitors are running and how much traffic they're getting.
→ `b2b_ads_search` + `semrush_scraper` + `similarweb_scraper`

---

## Available Tools

Server prefix: `mcp__syncgtm__`  
Most tools use **waterfall enrichment** — multiple providers tried in sequence until a result is found.

### Account

| Tool | Cost | Description |
|------|------|-------------|
| `check_credits` | Free | Check your remaining credit balance. Call before expensive tools. |

---

### Person Enrichment

| Tool | Cost | Description | Optional |
|------|------|-------------|----------|
| `enrich_person` | 2 credits/result | Enrich by name, email, LinkedIn URL, or org. Returns contact details, title, company info. | `first_name`, `last_name`, `email`, `linkedin_url`, `organization_name` |
| `find_work_email` | 1 credit/result | Find work email from LinkedIn URL or name. | `linkedin_url`, `first_name`, `last_name`, `organization_name` |
| `find_personal_email` | 3 credits/result | Find personal email from LinkedIn URL or name. | `linkedin_url`, `first_name`, `last_name`, `organization_name` |
| `find_work_phone` | 15 credits/result | Find direct work phone number. | `linkedin_url`, `first_name`, `last_name`, `organization_name` |
| `find_mobile_number` | 15 credits/result | Find mobile phone number. | `linkedin_url`, `first_name`, `last_name`, `organization_name` |
| `verify_email` | 0.3 credits/run | Verify email is valid and deliverable. | `email` |
| `validate_whatsapp` | 0.5 credits/run | Check if a phone number is registered on WhatsApp. | `mobile` |

---

### Reverse Lookup (Email → LinkedIn)

| Tool | Cost | Description | Optional |
|------|------|-------------|----------|
| `find_linkedin_from_work_email` | 2 credits/result | Find LinkedIn profile from work email. | `email` |
| `find_linkedin_from_personal_email` | 2 credits/result | Find LinkedIn profile from personal email. | `email`, `work_email` |
| `linkedin_profile_enrich` | 1 credit/run | Enrich a LinkedIn profile — work experience, education, skills. | `profile_url` |
| `linkedin_profile_posts` | 1 credit/run | Get 10 most recent posts and engagement from a LinkedIn profile. | `profile_url` |

---

### Company Enrichment

| Tool | Cost | Description | Optional |
|------|------|-------------|----------|
| `enrich_organization` | 2 credits/run | Enrich company by domain — industry, size, funding, technologies. | `domain` |
| `find_company_techstack` | 1 credit/run | Find the technology stack used by a company. | `domain` |
| `find_company_website_traffic` | 1 credit/run | Find website traffic data for a company. | `domain` |
| `find_people_within_company` | 0.5 credits/result | Search for people at a company across multiple providers (merged + deduped). | `domain`, `title`, `company_name`, `first_name`, `last_name`, `country`, `max_results` (default 5) |
| `company_job_listings` | 0.5 credits/result | Find open roles at a company — filter by title, location, department. | `company_domain`, `company_name`, `company_linkedin_url`, `workplace_type`, `job_titles_include`, `job_titles_exclude`, `posted_after`, `posted_before`, `keywords`, `remote`, `max_results` (default 5) |
| `company_job_openings` | 5 credits/run | Retrieve job openings posted by a specific company. Useful for account research and hiring signal detection. | `company_linkedin_url`, `company_domain`, `job_title` (array), `location`, `max_results` (default 10) |

---

### LinkedIn Company / Page

| Tool | Cost | Description | Optional |
|------|------|-------------|----------|
| `enrich_linkedin_page` | 0.5 credits/run | Enrich a LinkedIn company page — description, specialties, employee count. | `identifier` (LinkedIn id, URL, or name) |
| `premium_linkedin_page_insights` | 2 credits/run | Deep company insights — growth trends, hiring patterns, employee distribution. | `identifier` (LinkedIn id, URL, or name) |
| `linkedin_page_posts` | 0.3 credits/result | Get recent posts from a LinkedIn company page. | `company_url` (LinkedIn URL or name), `max_posts` (default 10, max 100) |
| `job_openings_growth_rate` | 2 credits/run | Track growth rate of a company's open job postings over time. | `linkedin_page_url` |
| `head_count_growth_rate` | 2 credits/run | Track growth rate of a company's employee headcount over time. | `linkedin_page_url` |

---

### Search / Discovery

| Tool | Cost | Description | Optional |
|------|------|-------------|----------|
| `search_people` | 0.3 credits/result | Search for people by job title, seniority, function, location, industry, and email/mobile availability. Target one company or discover across an account list with technographic and headcount filters. | `titles`, `roles`, `query`, `company_domain`, `company_domains`, `contact_job_level`, `contact_job_function`, `contact_country`, `contact_country_code`, `contact_linkedin_industry`, `has_email`, `has_mobile_phone`, `required_email`, `required_mobile`, `company_employee_ranges`, `company_country_codes`, `company_crm_tech`, `include_company`, `include_domain_intel`, `min_followers`, `max_followers`, `min_seniority`, `limit` (default 25), `offset` |
| `search_companies` | 0.3 credits/result | Search for companies by industry, headcount, country, technographics, funding stage, revenue range, or founded-year window. Set `preview:true` to get match counts before spending credits. | `company_domain`, `company_name`, `company_domains`, `industries`, `headcount_ranges`, `country_codes`, `crm_tech`, `analytics_tech`, `marketing_tech`, `funding_stages`, `revenue_ranges`, `founded_year_min`, `founded_year_max`, `preview` (bool), `limit` (default 25), `offset` |
| `google_maps_listings` | 0.5 credits/result | Search Google Maps for local business listings — name, address, phone, website, rating, category. | `query` (required), `location`, `max_results` (default 20) |

---

### Signals

| Tool | Cost | Description | Optional |
|------|------|-------------|----------|
| `check_job_change` | 2 credits/run | Detect if a person has recently changed jobs. Use as a trigger for timely outreach when a prospect moves to a new role. | `profile_url` (LinkedIn URL or username) |
| `check_promotions` | 2 credits/run | Detect if a person has recently been promoted. Use as a trigger for timely outreach when a prospect moves up. | `profile_url` (LinkedIn URL or username) |
| `company_product_launch` | 1 credit/run | Detect recent product launch news for a company. Use as a trigger for outreach when a prospect has just shipped something new. | `domain`, `company_name` |

---

### Social & Ads

| Tool | Cost | Description | Optional |
|------|------|-------------|----------|
| `meta_ads_analysis` | 0.5 credits/run | Get active ads running from a Facebook/Meta page. | `page_id`, `ad_type`, `max_results` |
| `instagram_profile_posts` | 0.5 credits/run | Get recent posts from an Instagram profile. | `username`, `profile_url`, `max_posts` (default 10) |
| `tiktok_profile_video` | 0.5 credits/run | Get recent videos from a TikTok profile. | `username`, `profile_url`, `max_posts` (default 10) |
| `x_profile_posts` | 0.4 credits/result | Get recent posts from an X (Twitter) profile — content, engagement, timestamps. | `username`, `profile_url`, `max_posts` (default 10) |
| `x_post_commenters` | 0.4 credits/result | Get people who commented on an X (Twitter) post. Build warm lead lists from engaged audiences. | `post_url`, `max_results` |
| `tiktok_comments_from_post` | 0.4 credits/result | Collect comments and commenter profiles from a TikTok video. Build warm lead lists from engaged audiences. | `post_url`, `max_results` |
| `tiktok_search_query` | 0.4 credits/result | Collect videos and related data from TikTok search results. Monitor trends, competitors, or topic coverage. | `query`, `max_results` |
| `b2b_ads_search` | 0.5 credits/result | Search B2B ads for a company via LeadMagic — creative, targeting, and performance data alongside resolved company details. Domain lookups are more reliable than name lookups. | `company_domain`, `company_name` |

---

### Competitor Intel

| Tool | Cost | Description | Optional |
|------|------|-------------|----------|
| `semrush_scraper` | 1 credit/run | Get traffic, top keywords, and backlink data for a domain via SemRush. | `domain` |
| `similarweb_scraper` | 2 credits/run | Get website traffic stats and audience insights from SimilarWeb — total visits, engagement rates, top traffic sources, competitor performance. | `domains` (array, e.g. `["stripe.com"]`), `manually_set_trigger_column` (bool), `only_run_if` (condition string) |

---

### Coming Soon

| Tool | Est. Cost | Description |
|------|-----------|-------------|
| LinkedIn Job Listings | 0.3 credits/result | Search LinkedIn jobs by keyword, location, and filters |
| LinkedIn Post Commenters | 0.3 credits/commenter | People who commented on a LinkedIn post |
| LinkedIn Post Engagers | 0.3 credits/engager | People who reacted to a LinkedIn post |
| Find Companies by Techstack | 1 credit/company | Find companies using a specific technology (via BuiltWith) |
| Headcount by Department | 2 credits | Company headcount filtered by department |
| Revenue Estimate | 1 credit | AI-estimated annual revenue for a company |
| Funding Data | 1 credit | AI-estimated or scraped funding and last round data |
| Education | 1 credit | Person's education history from LinkedIn |
| Previous Job Titles | 1 credit | Person's previous titles and companies from LinkedIn |

---

## Credit Tips

- Always call `check_credits` before running a batch enrichment.
- Phone lookups (`find_work_phone`, `find_mobile_number`) cost 15 credits — use only when needed.
- `find_people_within_company`, `company_job_listings`, `search_people`, and `search_companies` are billed per result — set `max_results` or `limit` to control spend.
- Verify emails first (`verify_email` at 0.3 credits) before paying for phone or personal email lookups.
- `search_companies` supports `preview:true` — get match counts before spending credits on results.
- `company_job_openings` costs 5 credits per run regardless of results; prefer `company_job_listings` (0.5/result) for targeted searches.
