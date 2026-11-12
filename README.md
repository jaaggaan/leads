# Magnus Copo – Lead Generation Engine

An n8n-based B2B lead discovery, website-audit, AI-qualification, and sales-pipeline automation for Magnus Copo.

> **Source of truth:** This README was prepared by inspecting the supplied n8n workflow export (`Magnus Copo - Lead Generation Engine`). It describes the workflow as exported. Credentials, Data Tables, API access, and the schedule configuration must be checked in the target n8n instance before production use.

**Open the live n8n workflow:** [Magnus Copo – Lead Generation Engine](https://jgnnn.app.n8n.cloud/workflow/J5uN5FiPCHuitqIO)

## 1. What the workflow does

The engine searches for businesses in India (with its built-in fresh-search rotation focused on Karnataka), collects publicly available business details, checks whether a website exists, audits websites when available, asks an OpenAI model to qualify each business, calculates a deterministic lead score, and stores qualified records.

It also:

- Runs a queue of searches with priority ordering and retry handling.
- Prevents overlapping runs with a Data Table lock.
- Checks an estimated daily API budget before processing.
- Avoids re-processing leads already present in the master table.
- Adds high- and medium-priority leads to a sales pipeline table.
- Writes execution metrics and estimated API costs.
- Builds and emails a daily summary at 7 PM.

**Important:** This is a lead-research and qualification workflow. It does not send sales outreach to prospects, schedule meetings, or send proposals. The Gmail node is used only for the internal daily report.

## 2. Technology and integrations

| Component | Purpose |
|---|---|
| n8n | Workflow orchestration, scheduling, branching, loops, and persistence |
| n8n Data Tables | Configuration, search queue, lead records, audit records, pipeline, and execution logs |
| Brave Search n8n node | Discovers business websites and location results |
| Firecrawl n8n node | Scrapes business websites and returns page content, HTML, metadata, and links |
| Google PageSpeed Insights API | Runs mobile Performance and SEO Lighthouse checks |
| OpenAI n8n node | Classifies and qualifies each business using supplied evidence |
| Gmail | Sends the daily internal summary |

The AI node is configured to use **GPT-5.4-mini** and a strict JSON schema. The workflow export references managed/gateway credentials for Brave Search, Firecrawl, and OpenAI, plus a Google PageSpeed credential and a Gmail OAuth2 credential. Re-select or recreate these credentials in the destination n8n instance; exported credential IDs are not portable guarantees.

## 3. High-level architecture

```text
Every 5 Minutes (schedule node label)
        |
Read config + today's execution logs
        |
Check engine enabled, lock, and estimated daily budget
        |
     Can Run?
        |
Acquire lock
        |
Read search queue -> pick next eligible search
        |
   Search available?
        |
Mark search RUNNING
        |
Brave Search (up to 20 results)
        |
Normalise, filter, and deduplicate search results
        |
   Route result
   /    |      \
error  empty   businesses
 |      |          |
retry   finish   split into individual businesses
                    |
              Find existing lead
                    |
              Already in database?
                /         \
              yes          no
               |            |
       Refresh last       Has website?
       verified data       /       \
       and mark duplicate yes       no
                         |           |
                    Firecrawl    Prepare no-website
                         |         audit profile
                    PageSpeed          |
                         |             |
                    Audit website      |
                         \             /
                          AI qualification
                                  |
                          Score & finalise
                                  |
                          Save master lead
                                  |
                          Save website audit
                                  |
                     High or medium priority?
                           /            \
                         yes             no
                          |               |
                   Sales pipeline     Mark new lead
                           \              /
                            Continue loop
                                  |
                           Summarise run
                                  |
                      Update search + execution log
                                  |
                             Release lock

Separate schedule: Daily 7 PM
        |
Read leads + last 30 days of logs + queue
        |
Build HTML daily report
        |
Email via Gmail
```

The workflow also has a queue replenishment branch: it reads past searches, generates a fresh industry/location combination, and inserts it into `MC_Searches` as `PENDING`.

## 4. Workflow node reference

The export contains 48 nodes. The following groups explain their responsibilities.

### A. Scheduler, guardrails, and queue selection

| Node | Responsibility |
|---|---|
| `Every 5 Minutes` | Starts the lead-engine branch on a minutes-based schedule. See the schedule verification note below. |
| `Read Engine Config` | Reads all rows from `MC_Config`. |
| `Read Today's Spend` | Reads today's rows from `MC_Execution_Log`. |
| `Check Lock & Budget` | Applies engine-enabled, lock-timeout, record-limit, retry, and estimated-cost settings. |
| `Can Run?` | Stops the main branch when the guard check fails. |
| `Acquire Lock` | Stores the current execution ID and lock timestamp in `MC_Config`. |
| `Read Search Queue` | Reads up to 250 queue rows ordered by priority. |
| `Pick Next Search` | Selects the next eligible `PENDING`, due `RETRY`, or stale `RUNNING` search. |
| `Search Available?` | Routes to processing only when an eligible search exists. |
| `Mark Search Running` | Marks the selected search as `RUNNING` and updates `last_run`. |

### B. Business discovery and search error handling

| Node | Responsibility |
|---|---|
| `Discover Businesses` | Calls Brave Search for up to 20 results, requesting web and location results for India in English. |
| `Normalise Businesses` | Filters directories, listicles, aggregators, documents, and public-sector results; extracts business identity, website, contact clues, address, evidence, and dedupe keys. |
| `Route Search Result` | Separates search errors, empty results, and usable businesses. |
| `Plan Retry With Backoff` | Increments retry count and calculates exponential retry delay. |
| `Mark Search For Retry` | Updates search status and retry metadata. |
| `Log Search Failure` | Writes failed/retry execution information to `MC_Execution_Log`. |
| `Release Lock` | Clears the `engine_lock` value. |
| `Read Past Searches` | Supplies past search combinations to the fresh-search generator. |
| `Generate Fresh Search` | Selects an industry and location combination not found in the supplied past-search set. |
| `Queue Fresh Search` | Inserts the generated search into `MC_Searches` with `PENDING` status. |

The result normaliser has a long explicit exclusion list for directories, marketplaces, social platforms, job sites, aggregators, public-sector sources, and other non-prospect sources. It has a narrow exception for certain individual Justdial business-detail pages, which are treated as low-confidence no-website candidates.

### C. Per-business processing

| Node | Responsibility |
|---|---|
| `Split Businesses` | Splits the normalised business array into individual items. |
| `Loop Over Businesses` | Processes businesses one at a time. |
| `Current Business` | Holds the current item for downstream node references. |
| `Find Existing Lead` | Looks for an existing record in `MC_Master_Leads` using lead ID and dedupe keys. |
| `Already In Database?` | Routes existing records away from new-lead processing. |
| `Refresh Last Verified` | Refreshes verification time and fills available phone, email, or address fields on an existing lead. |
| `Mark Duplicate` | Marks the item as a duplicate for run statistics. |
| `Has Website?` | Chooses the website-audit path or no-website path. |
| `Scrape Website` | Uses Firecrawl to retrieve website content, HTML, and links. |
| `PageSpeed Check` | Calls PageSpeed Insights for mobile Performance and SEO categories. |
| `Audit Website` | Derives technical, SEO, contact, CTA, booking, social, and content observations from scrape and PageSpeed data. |
| `Prepare No-Website Profile` | Creates a structured `NO_WEBSITE_FOUND` audit profile without attempting a scrape. |
| `AI Lead Qualification` | Uses GPT-5.4-mini to classify one business based only on supplied evidence and return structured JSON. |
| `Score & Finalise Lead` | Calculates the final score and priority, sets follow-up dates, and builds the master-lead record. |
| `Save Lead` | Inserts the final lead into `MC_Master_Leads`. |
| `Save Website Audit` | Inserts the audit snapshot into `MC_Website_Audit`. |
| `High Or Medium Priority?` | Routes high/medium leads to the sales pipeline. |
| `Add To Sales Pipeline` | Creates a sales-pipeline row for high/medium leads. |
| `Mark New Lead` | Marks the processed item as new and supplies counters for run summarisation. |

### D. Run summary and daily email

| Node | Responsibility |
|---|---|
| `Summarise Run` | Aggregates new leads, duplicates, errors, scrape/AI counts, and estimated API cost. |
| `Mark Search Done` | Marks the selected search `DONE` and stores result counts. |
| `Write Execution Log` | Writes the run summary to `MC_Execution_Log`. |
| `Daily 7 PM Summary` | Starts the separate daily reporting branch at 19:00. |
| `Read All Leads` | Reads master leads for dashboard-style totals. |
| `Read Last 30 Days Of Logs` | Reads execution history used in the report. |
| `Read Queue Status` | Reads the search queue for status counts. |
| `Build Daily Report` | Builds an HTML report with daily, weekly, monthly, all-time, cost, industry, area, and queue metrics. |
| `Email Daily Summary` | Emails the report to the recipient configured in the node. |

## 5. n8n Data Tables

The workflow references six Data Tables. Create them in n8n before activating the workflow, or map the nodes to equivalent existing tables. The IDs below are from the supplied export and may differ in another n8n workspace.

| Table | Exported table ID | Purpose |
|---|---|---|
| `MC_Config` | `1hHjRbLNzJ4ZLoR1` | Engine settings and execution lock |
| `MC_Searches` | `dHo10PiVJBO0G7JA` | Search queue, search metadata, retry state, and completion stats |
| `MC_Master_Leads` | `x3kvrnCuysR8olpu` | Canonical lead database and deduplication |
| `MC_Website_Audit` | `k1qHYwTX0gT6HKRy` | Website audit snapshots |
| `MC_Sales_Pipeline` | `WMq4gtVRiKM2hCbb` | Sales follow-up pipeline for high/medium leads |
| `MC_Execution_Log` | `tglDlqaaEhKIp5N7` | Execution outcomes, counts, errors, and estimated costs |

### 5.1 `MC_Config`

The code reads rows with `config_key` and `config_value`. The lock row also uses `locked_at`.

| `config_key` | Meaning | Default used by code |
|---|---|---:|
| `engine_enabled` | Master on/off switch | `true` |
| `max_records_per_run` | Maximum businesses passed onward from one search | `5` (clamped to 1–20) |
| `daily_api_budget_usd` | Estimated daily spend ceiling | `3` |
| `max_retries` | Maximum search attempts | `3` |
| `retry_base_minutes` | Base retry delay | `10` |
| `lock_timeout_minutes` | Lock/stale-run timeout | `20` |
| `cost_search_usd` | Estimated cost per search | `0.005` |
| `cost_scrape_usd` | Estimated cost per website scrape | `0.002` |
| `cost_ai_usd` | Estimated cost per AI call | `0.003` |
| `engine_lock` | Current execution ID; blank means unlocked | Must exist as a row |

Add one row per key. Values are read from `config_value`; numeric values should be stored as numeric strings. Create the `engine_lock` row even when its value is blank. The workflow updates `locked_at` when acquiring the lock and clears `config_value` when releasing it.

### 5.2 `MC_Searches`

The workflow uses these fields:

- `search_id`
- `industry`
- `search_keyword`
- `location`
- `city`
- `region`
- `target_customer`
- `priority`
- `status`
- `retry_count`
- `next_retry_at`
- `last_run`
- `last_error`
- `businesses_found`
- `leads_added`

Statuses used in code: `PENDING`, `RUNNING`, `RETRY`, `DONE`, and `FAILED`.

Seed this table with search rows if you want to control the initial search topics. The workflow also generates fresh searches from its built-in rotation.

### 5.3 `MC_Master_Leads`

The lead-save node maps these fields:

- Identity and location: `lead_id`, `date_added`, `business_name`, `industry`, `sub_category`, `business_size`, `city`, `area`, `address`
- Contact and web: `phone`, `public_email`, `website_url`, `website_status`, `website_quality`, `mobile_friendly`, `ssl_status`, `page_speed`, `seo_status`
- Social: `social_media_url`, `linkedin_url`, `instagram_url`, `facebook_url`
- Qualification: `digital_gaps`, `recommended_service`, `opportunity_type`, `ai_analysis`, `sales_pitch`
- Scoring: `lead_score`, `lead_priority`, `confidence`
- Research traceability: `evidence_url`, `data_source`, `search_id`, `remarks`, `last_verified`
- Sales tracking: `contact_status`, `follow_up_date`, `assigned_to`, `lead_status`
- Deduplication: `domain_key`, `phone_key`, `name_city_key`
- Audit fields: `audit_final_url`, `audit_http_status`, `audit_seo_score`, `audit_title`, `audit_meta`, `audit_broken_links`, `audit_cta`, `audit_form`, `audit_ux`, `audit_issues`, `audit_improvements`

The duplicate lookup uses `lead_id`, `domain_key`, `phone_key`, and `name_city_key`. The workflow's query node is configured with all four conditions; validate how your n8n Data Table node combines these conditions and consider enforcing uniqueness or implementing explicit OR-based matching if any matching key should be sufficient.

### 5.4 `MC_Website_Audit`

The audit-save node maps:

- `lead_id`, `business_name`, `website_url`, `final_url`, `audit_date`
- `http_status`, `ssl`, `mobile_responsive`, `page_speed_score`, `seo_score`
- `seo_title`, `meta_description`, `broken_links`
- `cta_status`, `contact_form_status`, `ux_observations`
- `issues_found`, `recommended_improvements`, `confidence`

`broken_links` is deliberately recorded as `NOT TESTED (link checking not enabled)` for audited websites. Do not interpret this field as a successful broken-link check.

### 5.5 `MC_Sales_Pipeline`

The pipeline node creates records with fields including:

- `lead_id`, `business_name`, `industry`, `city`, `area`, `phone`, `public_email`, `website_url`
- `lead_score`, `lead_priority`, `recommended_service`, `sales_pitch`
- `contact_status`, `meeting_status`, `proposal_status`
- `next_follow_up`, `sales_owner`, `deal_status`, `remarks`

New pipeline rows start with `NOT CONTACTED`, `NOT SCHEDULED`, `NOT SENT`, `Unassigned`, and `OPEN` values as configured in the node.

### 5.6 `MC_Execution_Log`

The workflow writes:

- `execution_id`, `execution_date`, `search_id`, `source`
- `records_collected`, `new_leads`, `duplicates`, `errors`
- `execution_status`, `api_cost`, `remarks`

The daily budget check sums `api_cost` for rows returned by the date filter. The report uses execution logs to estimate spend and display run outcomes.

## 6. Search strategy and lead discovery

### Search query

The queue processor builds a query in this form:

```text
<search_keyword> in <area>, <city>
```

The Brave Search node requests up to 20 results, uses country `IN`, English search, India English UI language, moderate SafeSearch, and requests both web and location results plus extra snippets.

### Built-in fresh-search rotation

The fresh-search generator rotates across 15 industry categories and 16 Karnataka locations:

**Industries:** Web Design & Digital Agencies, Manufacturing, Real Estate, Healthcare Clinics, Education & Coaching, Engineering Services, Professional Services, Restaurants & Cafes, Salons & Wellness, Logistics & Transport, Interior Design, Legal Services, Event & Wedding Services, Auto Services, IT & Software Services.

**Locations:** Whitefield, Koramangala, Indiranagar, Jayanagar, HSR Layout, Electronic City, Peenya, Yelahanka, Marathahalli, JP Nagar, Rajajinagar, Hebbal, Mysuru, Hubballi, Mangaluru, and Belagavi.

The generator avoids combinations present in the rows supplied by `Read Past Searches`. It selects a random available combination and queues one search at priority `1`.

## 7. Website audit logic

For businesses with a website, the workflow:

1. Scrapes the homepage through Firecrawl.
2. Requests mobile PageSpeed Insights Performance and SEO categories.
3. Extracts page title, meta description, H1, viewport metadata, page text, contact details, social links, and selected conversion signals.
4. Checks for visible CTA language, contact forms, booking options, WhatsApp references, and homepage content volume.
5. Detects selected outdated indicators such as jQuery 1.x, legacy HTML tags, Flash references, and an old copyright year.
6. Produces an issue list and suggested improvements.

Website statuses include:

- `WEBSITE_AVAILABLE`
- `WEBSITE_INACCESSIBLE`
- `UNDER_CONSTRUCTION`
- `REDIRECTS_ELSEWHERE`
- `NO_WEBSITE_FOUND`

The no-website path creates an audit profile and marks the business as a potential website-development opportunity. Directory-only candidates are explicitly marked low confidence and not independently verified.

**Audit limitations:** This is a homepage-level heuristic audit, not a complete crawl or formal technical SEO audit. It does not test broken links, inspect every page, validate form submission, verify business ownership, or independently confirm that contact details are current. The mobile-friendly result is based on detecting a responsive viewport meta tag, not a full device-layout test.

## 8. AI qualification

The OpenAI node receives a JSON evidence package containing the business identity, industry, location, search evidence, contact availability, website audit fields, and homepage excerpt.

The system prompt instructs the model to use only the supplied evidence, avoid fabricated facts, and return `Unknown` when evidence is insufficient.

The strict output schema includes:

- `is_real_business`
- `business_name`
- `business_category`
- `sub_category`
- `business_size_estimate`
- `website_quality`
- `main_digital_problems`
- `recommended_services`
- `opportunity_types`
- `business_fit_score`
- `growth_potential_score`
- `sales_pitch`
- `qualification_reason`

The model is asked to classify business size as Startup, Small, SME, Medium, Large, or Unknown; recommend services from the workflow's defined service list; and classify opportunities using the `A_NO_WEBSITE` through `F_HIGH_VALUE_B2B` categories.

The AI result is an assistive qualification, not independently verified truth. Review evidence and contact details before outreach.

## 9. Lead scoring and priority

The `Score & Finalise Lead` code calculates a score out of 100 using the following components:

| Component | Maximum |
|---|---:|
| Website opportunity | 30 |
| Digital-presence gaps | 20 |
| Contact availability | 10 |
| Evidence confidence | 10 |
| AI business fit | 20 |
| AI growth potential | 10 |
| **Total** | **100** |

### Priority thresholds

| Score / condition | Priority | Default follow-up interval |
|---|---|---:|
| 80–100 | `HIGH PRIORITY` | 1 day |
| 60–79 | `MEDIUM PRIORITY` | 3 days |
| 40–59 | `LOW PRIORITY` | 7 days |
| Below 40 | `REJECT` | 30 days |
| Low confidence or AI error (when below the higher thresholds) | `RESEARCH REQUIRED` | 30 days |

If AI flags a result as not being a real business, the score is capped at 30. The score breakdown is stored in `ai_analysis`.

Only `HIGH PRIORITY` and `MEDIUM PRIORITY` leads are routed into `MC_Sales_Pipeline`. Other leads are still saved to the master lead table and marked with their corresponding lead status.

## 10. Retry, locking, and budget controls

### Search retry

Search failures increment `retry_count`. The delay is exponential:

```text
retry delay = retry_base_minutes × 2^(attempt_number - 1)
```

With the default base of 10 minutes, the intended sequence is 10, 20, then 40 minutes. After the configured maximum retry count, the search is marked `FAILED`.

The Brave node itself is also configured for up to three attempts with a five-second wait between attempts.

### Execution lock

The engine checks the `engine_lock` row before running. A lock is considered active while its `locked_at` timestamp is within `lock_timeout_minutes`. A stale lock can be superseded after the timeout.

### Estimated API budget

The budget check compares today's logged `api_cost` total with `daily_api_budget_usd`. The run summary estimates costs using the configured unit costs for search, scraping, and AI calls.

These are **configured estimates**, not provider-billed usage. Update unit costs to match your actual plan and usage. The budget check is performed before the run; it is not a hard real-time provider-side spending cap.

## 11. Installation and setup

### Prerequisites

- An n8n instance with permission to create Data Tables and workflows.
- Brave Search API access and the compatible Brave Search community node.
- Firecrawl API access and the compatible Firecrawl community node.
- OpenAI API access.
- Google PageSpeed Insights API access and a valid API key.
- Gmail OAuth2 access for the account that will send the report.

### Setup steps

1. **Import the workflow**
   - In n8n, import the supplied workflow JSON.
   - Review all nodes and connections after import.

2. **Install required community nodes**
   - Install the Brave Search and Firecrawl n8n nodes if they are not already available.
   - Confirm the installed node versions are compatible with the exported workflow.

3. **Create the six Data Tables**
   - Create `MC_Config`, `MC_Searches`, `MC_Master_Leads`, `MC_Website_Audit`, `MC_Sales_Pipeline`, and `MC_Execution_Log`.
   - Add the columns referenced in Section 5.
   - Update each Data Table node to point to the newly created table if the imported IDs do not resolve.

4. **Seed engine configuration**
   - Add the configuration rows listed in Section 5.1.
   - Ensure an `engine_lock` row exists with a blank `config_value`.
   - Set `engine_enabled` to `false` during initial setup.

5. **Configure credentials**
   - Select Brave Search credentials on `Discover Businesses`.
   - Select Firecrawl credentials on `Scrape Website`.
   - Select OpenAI credentials on `AI Lead Qualification`.
   - Configure the PageSpeed API key credential on `PageSpeed Check`.
   - Select Gmail OAuth2 credentials on `Email Daily Summary`.
   - Never paste API keys into Code nodes or commit them to source control.

6. **Set the daily report recipient**
   - The exported Gmail node is configured to send to `jaaggaan777@gmail.com`.
   - Change the recipient if the operational owner or reporting mailbox is different.
   - Confirm Gmail OAuth permissions and test delivery.

7. **Verify schedule settings**
   - Inspect `Every 5 Minutes` and confirm the actual interval is five minutes.
   - Inspect `Daily 7 PM Summary` and confirm the desired timezone and 19:00 schedule.
   - See the important schedule note below.

8. **Test with a small queue**
   - Set `max_records_per_run` to `1`.
   - Set a conservative daily budget.
   - Add one known test search to `MC_Searches`.
   - Execute manually and inspect each node's output and all six Data Tables.

9. **Validate the full lifecycle**
   - Test a business with a working website.
   - Test a no-website candidate.
   - Test a duplicate candidate.
   - Test a search failure and retry.
   - Confirm high/medium leads enter the sales pipeline.
   - Confirm the daily email is generated and delivered.

10. **Enable production scheduling**
    - Set `engine_enabled` to `true` only after all checks pass.
    - Activate the workflow and monitor its first scheduled runs.

## 12. Important verification findings before production

The following are observations from the exported JSON, not assumptions about your live n8n environment.

1. **The “Every 5 Minutes” node needs verification.** Its exported schedule rule specifies the `minutes` field but does not visibly include a numeric interval value. The node's name says five minutes, but the exported parameter alone does not prove a five-minute cadence. Open the node and explicitly set/confirm every five minutes.

2. **Confirm the daily report timezone.** The report node is configured for hour `19`, but the export does not explicitly document the intended timezone. Confirm the workflow/instance timezone is the one you expect.

3. **Confirm Data Table filter semantics.** `Read Search Queue` has conditions for `PENDING`, `RETRY`, and `RUNNING`; `Find Existing Lead` has conditions for four identity keys. Verify how the installed n8n Data Table version combines these conditions. Duplicate detection should match any valid identity key, not require every key to match simultaneously.

4. **Review lock release on every branch.** The workflow includes a lock acquisition and release node. Test normal completion, no eligible search, search error, and unexpected node failure to ensure a failed execution cannot leave a lock until timeout. Consider an error-workflow/finally-style cleanup path.

5. **The daily report recipient is hard-coded in the Gmail node.** Change it to the intended recipient and verify authorization.

6. **Fresh searches are generated from a fixed list.** The generator is limited to the 15 industries and 16 locations in Section 6. Add or edit combinations in the `Generate Fresh Search` Code node to expand coverage.

7. **Cost values are estimates.** They are configuration constants and should be reconciled with actual provider billing and plan limits.

8. **The lead score is a custom heuristic.** Review the scoring weights and thresholds against actual sales outcomes before treating the priority as a reliable prediction.

9. **Contact extraction is evidence-based but not verification.** Publicly surfaced phone numbers and emails can be stale, shared, or incorrectly associated. Validate before contacting a prospect and comply with applicable privacy, anti-spam, and outreach rules.

10. **No automated prospect outreach is configured.** The sales pipeline is populated, but there is no prospect email, WhatsApp, calling, proposal, or meeting automation in this workflow export.

## 13. Troubleshooting

| Symptom | Checks |
|---|---|
| Workflow does not start | Confirm workflow activation, schedule configuration, instance timezone, and n8n execution status. |
| Engine refuses to run | Inspect `engine_enabled`, the `engine_lock` row, `locked_at`, lock timeout, and today's logged spend. |
| Search queue stays idle | Check for `PENDING` rows, due `RETRY` rows, stale `RUNNING` rows, and priority values. |
| Brave Search fails | Reconnect credentials, verify API access/credits, confirm community node installation, and inspect the node error output. |
| No businesses are returned | Review query relevance, country/language parameters, and the exclusion/normalisation rules. |
| Firecrawl fails | Check the API credential, target site accessibility, rate limits, and scrape response structure. |
| PageSpeed fields are `UNKNOWN` | Check the Google API key, PageSpeed API enablement, quota, and HTTP response. |
| AI qualification is missing | Check OpenAI credentials, model access, response shape, token limits, and strict JSON output. |
| Leads are incorrectly marked duplicate | Inspect the four dedupe keys and Data Table condition semantics. |
| Sales pipeline is empty | Confirm lead scores are at least 60 and inspect the `High Or Medium Priority?` branch. |
| Daily report is not emailed | Check the 19:00 schedule, Gmail OAuth2 credential, recipient, and Gmail node execution. |
| Lock remains set | Inspect `MC_Config.engine_lock` and the last execution; clear only after confirming no run is still active. |

## 14. Operational checklist

### Before enabling
- [ ] All six Data Tables exist and have the expected columns.
- [ ] `MC_Config` contains all required keys and an `engine_lock` row.
- [ ] Brave Search, Firecrawl, OpenAI, PageSpeed, and Gmail credentials are connected.
- [ ] Community nodes are installed and compatible.
- [ ] Five-minute schedule and 7 PM report schedule are verified.
- [ ] Duplicate matching semantics are tested.
- [ ] Error paths and lock cleanup are tested.
- [ ] Test lead and daily report have been checked.

### Ongoing
- [ ] Review failed/retry searches in `MC_Searches`.
- [ ] Review daily cost estimates against actual provider usage.
- [ ] Review lead evidence and contact details before outreach.
- [ ] Review high/medium lead quality and adjust scoring weights as needed.
- [ ] Monitor n8n execution failures and stale locks.
- [ ] Keep API credentials private and rotate them when required.

## 15. Source workflow

- Workflow name: `Magnus Copo - Lead Generation Engine`
- Live workflow: https://jgnnn.app.n8n.cloud/workflow/J5uN5FiPCHuitqIO
- Export format: n8n workflow JSON
- Exported nodes: 48
- Referenced Data Tables: 6

This README documents the supplied workflow export. If the workflow is changed, update this file to keep node behaviour, table mappings, credentials, scoring rules, and operational instructions aligned.
