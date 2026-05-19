# Infrastructure Update Spec: Data-Driven Decision Hub (Meta + Google + Clarity + CRM + Daily Claude)

## 1) Why this update is needed
Your PRD direction is strong, but there are several gaps that will cause unstable reporting, poor attribution, and brittle migration to BigQuery if not fixed now:

1. **No canonical grain definition per table** (event-level vs daily aggregate is mixed).
2. **No idempotency strategy** for daily/hourly pulls (risk of duplicate spend/leads).
3. **No source-of-truth hierarchy** when metrics conflict (e.g., Meta clicks vs landing sessions).
4. **No explicit timezone policy for source ingestion windows** (UTC storage alone is not enough).
5. **No late-arriving data policy** (ad/CRM conversions backfill for days/weeks).
6. **No unified identity bridge** across `gclid`, `fbclid`, `utm_*`, and CRM lead/deal lifecycle.
7. **No data quality SLAs/tests** (null keys, malformed UTMs, orphan facts).
8. **No schema versioning/change management** for API field evolution.
9. **No governance for AI outputs** (Claude summaries not structured for analytics reuse).

This spec closes those gaps and upgrades your VPS stack into a stable, precise, and migration-ready decision hub.

---

## 2) Target architecture (VPS now, BigQuery-ready later)

### 2.1 Data flow layers
1. **Raw ingestion layer** (`raw_*`): exact API payload snapshots + ingest metadata.
2. **Standardized staging layer** (`stg_*`): cleaned, typed, UTC-normalized records.
3. **Core warehouse layer** (`dim_*`, `fact_*`, bridge tables): canonical analytics model.
4. **Serving marts** (`mart_*`): dashboard-ready denormalized views.
5. **AI insights layer** (`fact_ai_insights`): structured daily narrative + scores.

### 2.2 Recommended stack
- **Database**: PostgreSQL 15+ on VPS (partitioned facts by date).
- **Orchestration**: cron + Python runners (or n8n if already standardized).
- **Transform**: SQL scripts and versioned migrations.
- **Storage discipline**: append raw, upsert standardized/core.
- **Future portability**: avoid DB-specific features that BigQuery cannot map.

---

## 3) Required API integrations

## 3.1 Meta Ads API
- Pull hierarchy: account → campaign → ad_set → ad.
- Pull metrics by `date`, `campaign_id`, `adset_id`, `ad_id`, `country` (if geo split needed).
- Persist attribution settings used for pull (click/view windows) as columns.

## 3.2 Google Ads API
- Pull `campaign`, `ad_group`, `ad_group_ad` performance and conversions.
- Persist `segments.date`, network, device where needed.
- Keep original Google micros units and standardized decimal amount.

## 3.3 Microsoft Clarity API
- Pull behavior metrics by landing page and campaign parameter mapping.
- Add derived quality metrics: rage/dead click rates, scroll depth bands.

## 3.4 Bitrix24 CRM (+ Zapier/n8n/webhooks)
- Enforce a strict input contract for lead create/update and deal conversion events.
- Persist both source payload and normalized event rows.
- Ensure lead→deal lineage cannot break (bridge table + constraints).

## 3.5 Claude API daily analysis
- Run once daily after data freshness cutoff.
- Input must be a structured JSON bundle (not free text only).
- Store response in both:
  - raw text (traceability),
  - parsed JSON fields (queryable insights, risk flags, recommendations).

---

## 4) Canonical data model (standardized)

## 4.1 Naming and typing standards
- `snake_case` for all columns/tables.
- UTC timestamps as `timestamp with time zone`.
- Money: `numeric(18,6)` + original source unit fields.
- IDs as `text` (cross-platform safety).
- Every table includes: `ingested_at_utc`, `updated_at_utc`, `source_system`, `run_id`.

## 4.2 Core dimensions

### `dim_platform`
- `platform_key` (pk)
- `platform_name` (`meta_ads`, `google_ads`, `clarity`, `bitrix24`)

### `dim_date`
- standard calendar attributes (for fast rollups and future BigQuery parity)

### `dim_campaign`
- `campaign_key` (surrogate pk)
- `platform_name`
- `platform_campaign_id`
- `campaign_name`
- `objective`
- `status`
- `effective_start_utc`
- `effective_end_utc`

### `dim_ad`
- `ad_key` (pk)
- `platform_name`
- `platform_ad_id`
- `platform_adset_or_adgroup_id`
- `campaign_key` (fk)

### `dim_landing_page`
- `landing_page_key` (pk)
- `url_normalized`
- `host`
- `path`

### `dim_tracking`
- `tracking_key` (pk surrogate)
- `tracking_id` (canonical generated id)
- `gclid`
- `fbclid`
- `utm_source`
- `utm_medium`
- `utm_campaign`
- `utm_term`
- `utm_content`
- `first_seen_utc`
- unique constraints to reduce collisions

### `dim_contact` (optional but recommended)
- anonymized hash keys for privacy-safe person-level stitching.

## 4.3 Core facts

### `fact_ad_performance_daily`
**Grain:** 1 row per (`date_utc`, `platform_name`, `campaign_key`, `ad_key` nullable, `geo` nullable, `device` nullable)
- `impressions`, `clicks`, `spend`, `conversions`, `conversion_value`
- source-attribution-window fields
- `is_backfill` flag

### `fact_website_behavior_daily`
**Grain:** 1 row per (`date_utc`, `landing_page_key`, `utm_campaign`, `platform_name='clarity'`)
- `sessions`, `avg_time_on_page_sec`, `avg_scroll_depth_pct`
- `rage_clicks`, `dead_clicks`, `quick_backs`

### `fact_crm_lead_events`
**Grain:** event-level (lead_created, lead_updated, deal_created, stage_changed)
- `crm_event_id` (pk)
- `event_type`
- `event_at_utc`
- `bitrix_lead_id`
- `bitrix_deal_id` nullable
- `tracking_key` fk
- `lead_status`, `deal_stage`, `revenue_value`

### `fact_crm_lead_snapshot_daily`
**Grain:** 1 row per (`date_utc`, `bitrix_lead_id`) latest state snapshot for fast BI joins.

### `fact_ai_insights`
**Grain:** 1 row per (`analysis_date_utc`, `analysis_type='daily_marketing_brief'`)
- `input_window_start_utc`, `input_window_end_utc`
- `model_provider` (`anthropic_claude`)
- `model_name`
- `summary_text`
- `insight_json` (JSON)
- `risk_score` (0-100)
- `anomaly_flags` (JSON array)
- `recommended_actions` (JSON array)

## 4.4 Bridge tables

### `bridge_lead_deal`
- `bitrix_lead_id`
- `bitrix_deal_id`
- `linked_at_utc`
- unique pair constraint

### `bridge_tracking_touchpoints`
- maps one `tracking_key` to multiple touches with sequence/order and timestamps.

---

## 5) Data contracts and validation rules

## 5.1 UTM contract (strict)
- Required for paid traffic leads: `utm_source`, `utm_medium`, `utm_campaign`.
- Optional: `utm_term`, `utm_content`.
- Allowed value dictionaries for `utm_source` and `utm_medium` (controlled taxonomy).
- Invalid UTMs go to quarantine table `err_invalid_utm` with reason codes.

## 5.2 Click ID capture
- Capture and persist both raw and normalized forms of `gclid`, `fbclid`.
- Validation regex checks + max lengths.
- If missing for paid lead, set `tracking_quality_status='missing_click_id'`.

## 5.3 Event idempotency keys
- API pulls: unique key = (`source_system`, `source_primary_id`, `date_utc`, `run_granularity`).
- Webhooks: unique key = (`source_system`, `external_event_id`) or payload hash.

## 5.4 Freshness and completeness SLAs
- Ad data available by **06:00 UTC daily** for D-1.
- CRM events lag < 15 min.
- Clarity daily aggregation ready by 07:00 UTC.
- Fail pipeline if thresholds are violated.

---

## 6) Standard metric definitions (single source of truth)
- **Spend**: platform-reported cost at selected attribution settings.
- **Leads**: unique `bitrix_lead_id` created in time window.
- **Qualified leads**: leads entering configured stage set.
- **Revenue**: won deal value in CRM, net of duplicates/cancellations policy.
- **CAC**: spend / qualified leads (or won deals, explicitly versioned).
- **ROAS**: attributed revenue / spend with declared attribution model.

Store these definitions in `dim_metric_definition` with versioning.

---

## 7) Pipeline scheduling and backfill policy

### 7.1 Daily schedule (UTC)
1. 01:30–03:00: pull ad platform D-1 + rolling D-7 backfill.
2. 03:00–04:00: pull Clarity D-1.
3. Near-real-time: ingest CRM webhooks continuously.
4. 05:00: run standardization + integrity tests.
5. 05:30: build marts.
6. 06:00: run Claude daily brief.

### 7.2 Backfill strategy
- Always re-pull previous 7 days (minimum) due to attribution drift.
- Weekly deep backfill for last 30/60 days configurable.
- Mark restated rows with `is_backfill=true` and `restated_at_utc`.

---

## 8) Dashboard and decision-hub outputs
Create serving views:
- `mart_daily_channel_performance`
- `mart_campaign_funnel`
- `mart_landing_page_quality`
- `mart_lead_to_revenue_attribution`
- `mart_ai_daily_brief`

Each mart includes data quality badges:
- freshness status,
- attribution coverage %,
- invalid UTM %,
- unmapped lead %.

---

## 9) Security, privacy, and governance
- Minimize PII in analytics tables; tokenize where possible.
- Keep raw payloads in restricted schema.
- Audit table for pipeline runs and data corrections.
- Store API credentials in environment secrets, not code.

---

## 10) Migration compatibility with BigQuery
- Use explicit schemas and avoid implicit casts.
- Keep JSON fields structured but bounded.
- Partition-ready keys: `date_utc` on fact tables.
- Cluster-ready keys: `platform_name`, `campaign_key`, `tracking_key`.
- Maintain dbt-style transformation logic (portable SQL where possible).

---

## 11) Implementation plan (revised)

### Phase 1 (Week 1): contracts + schema foundation
- Define UTM and click-id contracts.
- Create raw/stg/core schemas and constraints.
- Add Bitrix custom fields + webhook payload version.

### Phase 2 (Week 2): ingestion hardening
- Implement Meta/Google/Clarity ingestion with idempotency + run logs.
- Implement CRM event ingestion and lead/deal bridge.

### Phase 3 (Week 3): transformations + marts
- Build fact/dim models and dashboard marts.
- Add DQ tests (nulls, duplicates, referential integrity, freshness).

### Phase 4 (Week 4): AI + operations
- Implement daily Claude JSON briefing writeback.
- Add anomaly/risk scoring framework.
- Finalize operational runbooks and failure alerts.

### Phase 5 (Ongoing): BigQuery cutover readiness
- Export parity checks (row counts, metric parity) between VPS DB and BigQuery staging.

---

## 12) Acceptance criteria (must pass)
1. ≥ 98% of paid leads have valid `tracking_key` and required UTMs.
2. Duplicate rate in ad and CRM fact tables < 0.5% post-idempotency.
3. D-1 dashboard readiness by 07:00 UTC on 95% of days.
4. ROAS/CAC metric parity across dashboard and SQL validation queries.
5. Claude brief saved daily with parseable JSON fields and linked date window.

---

## 13) Key inconsistencies fixed vs original PRD
- Added **grain definitions** for every fact table.
- Added **raw/staging/core separation** for recoverability.
- Added **idempotency and backfill controls**.
- Added **data contracts + quarantine** for UTMs/click IDs.
- Added **lineage bridges** for lead→deal and multi-touch tracking.
- Added **AI output schema** to make Claude insight data queryable.
- Added **SLA + DQ framework** for stable results.

---

## 14) Minimal SQL DDL skeleton (illustrative)
```sql
create table if not exists dim_tracking (
  tracking_key bigserial primary key,
  tracking_id text not null,
  gclid text,
  fbclid text,
  utm_source text,
  utm_medium text,
  utm_campaign text,
  utm_term text,
  utm_content text,
  first_seen_utc timestamptz not null default now(),
  ingested_at_utc timestamptz not null default now(),
  updated_at_utc timestamptz not null default now(),
  source_system text not null,
  run_id text not null
);

create table if not exists fact_ai_insights (
  analysis_date_utc date not null,
  analysis_type text not null,
  input_window_start_utc timestamptz not null,
  input_window_end_utc timestamptz not null,
  model_provider text not null,
  model_name text not null,
  summary_text text,
  insight_json jsonb,
  risk_score int,
  anomaly_flags jsonb,
  recommended_actions jsonb,
  ingested_at_utc timestamptz not null default now(),
  updated_at_utc timestamptz not null default now(),
  source_system text not null,
  run_id text not null,
  primary key (analysis_date_utc, analysis_type)
);
```

This `update.md` should be treated as the operating spec for implementation tickets.
