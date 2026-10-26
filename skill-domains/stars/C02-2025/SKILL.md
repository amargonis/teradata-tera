---
name: stars-c02-2025
description: >-
  STARS measure C02 (Colorectal Cancer Screening), measurement year 2025. Trigger when the user asks about
  C02 or "Colorectal Cancer Screening", e.g. "what is our C02 rate", "C02 star rating", "show C02 care gaps",
  "who is missing C02", "break C02 down by age/provider/region", or to build a C02 dashboard.
  Uses the approved, EXPLAIN-validated certified Teradata SQL as the single source of truth for the
  denominator, numerator, rate and star band, and writes analytical queries on top of it.
---

# C02 — Colorectal Cancer Screening (2025)

Percent of Medicare Advantage enrollees aged 50–75 who had appropriate screening for colorectal cancer [populated from HEDIS COL specification — NOT in extracted CMS notes; age range corrected per business request to 50–75]

- **Program:** STARS
- **Measurement year:** 2025
- **Customer data:** database `DEMO_HealthcareSynthea` (customer id `DEMO_HealthcareSynthea`); reference database `HCLS`
- **Source tables:** `DEMO_HealthcareSynthea.conditions`, `DEMO_HealthcareSynthea.encounters`, `DEMO_HealthcareSynthea.observations`, `DEMO_HealthcareSynthea.patients`, `DEMO_HealthcareSynthea.payer_transitions`, `DEMO_HealthcareSynthea.payers`, `DEMO_HealthcareSynthea.procedures`, `HCLS.measure_value_set_codes`
- **Exported:** 2026-10-26 20:52 UTC from the Star Ratings measure automation app. The SQL below was generated, validated with Teradata `EXPLAIN`, and approved by a human reviewer.

## How to use this skill

Run all SQL with `base_readQuery` (Teradata dialect: use `TOP n`, not `LIMIT`).

1. Treat the **Certified SQL** below as the single source of truth for who is in the denominator and numerator. Never redefine eligibility, exclusions, codes or enrollment.
2. To answer a question (breakdown, filter, trend, gap-to-star, outreach list), reuse the certified CTE chain verbatim up to and including `CTE_05_NUMERATOR` (the "spine") and write only an **analytical tail** that reads from it. Follow the *Analytical query rules* below.
3. Quote the certified rate, star band and cut points from this file; never recompute or invent them. Any analytical result must reconcile to the certified denominator and numerator.
4. For dashboards, build every tile on top of the same spine so all numbers stay consistent with the certified result.

## Certified result

| Item | Value |
|---|---|
| Certified rate | 68.56% |
| Star band | 3 |
| Denominator | 8401 |
| Numerator | 5760 |
| Run time | 2026-10-09 17:33:43 |

### Star cut points

- 1 star: < 53 %
- 2 stars: >= 53 % to < 67 %
- 3 stars: >= 67 % to < 75 %
- 4 stars: >= 75 % to < 82 %
- 5 stars: >= 82 %

## Certified SQL

Run on Teradata as-is to reproduce the certified population. The final SELECT returns one row per denominator member (`member_id`, `in_numerator`).

```sql
WITH
CTE_01_MEMBERS AS (
    SELECT DISTINCT
        TRIM(patients.patient_id)          AS patient_id
        ,patients.birth_date
        ,patients.death_date
    FROM DEMO_HealthcareSynthea.patients patients
    WHERE (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM patients.birth_date))
              BETWEEN 50 AND 75                                           -- [Spec2:DENOM_AGE:MAPPED]
      AND (patients.death_date IS NULL OR patients.death_date > DATE '2025-12-31')  -- [Spec2:DENOM_ALIVE:MAPPED]
)

,CTE_02_ENROLLED AS (
    SELECT seg.patient_id
    FROM (
        SELECT
            TRIM(pt.patient_id)                                     AS patient_id
            ,CAST(pt.start_year AT TIME ZONE 'GMT' AS DATE)         AS seg_start
            ,CAST(pt.end_year   AT TIME ZONE 'GMT' AS DATE)         AS seg_end
            ,CAST(pt.start_year AT TIME ZONE 'GMT' AS DATE)
               - LAG(CAST(pt.end_year AT TIME ZONE 'GMT' AS DATE))
                 OVER (PARTITION BY TRIM(pt.patient_id)
                       ORDER BY CAST(pt.start_year AT TIME ZONE 'GMT' AS DATE)) AS gap_days
        FROM DEMO_HealthcareSynthea.payer_transitions pt
        INNER JOIN DEMO_HealthcareSynthea.payers py
            ON TRIM(pt.primary_payer_id) = TRIM(py.payer_id)          -- [SN-06]
        WHERE py.payer_type = 'GOVERNMENT'                            -- [Spec2:DENOM_PLAN_TYPE:PROXY] [SN-08]
    ) seg
    GROUP BY seg.patient_id
    HAVING MIN(seg.seg_start) <= DATE '2025-01-01'
       AND MAX(seg.seg_end)   >= DATE '2025-12-31'
       AND SUM(CASE WHEN seg.gap_days > 0 THEN 1 ELSE 0 END) <= 1
       AND MAX(CASE WHEN seg.gap_days > 0 THEN seg.gap_days ELSE 0 END) <= 45
    -- [Spec2:DENOM_ENROLL:PROXY:CERTIFIED_MULTI_SEGMENT] [SN-09] [AE-13]
)

,CTE_03_DENOMINATOR AS (
    SELECT
        m.patient_id
        ,m.birth_date
        ,(EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) AS age_at_anchor
    FROM CTE_01_MEMBERS m
    INNER JOIN CTE_02_ENROLLED e
        ON TRIM(m.patient_id) = TRIM(e.patient_id)
)

,CTE_04_EXCLUSIONS AS (
    SELECT d.*
    FROM CTE_03_DENOMINATOR d

    -- EXCL_HOSPICE — [Spec2:EXCL_HOSPICE:MAPPED] [SN-11]
    WHERE NOT EXISTS (
        SELECT 1
        FROM DEMO_HealthcareSynthea.encounters e
        WHERE TRIM(e.patient_id) = TRIM(d.patient_id)                  -- [SN-06]
          AND e.encounter_class = 'hospice'
          AND CAST(e.start_ts AT TIME ZONE 'GMT' AS DATE)
                BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'        -- [SN-05]
    )

    -- EXCL_PALLIATIVE — [Spec2:EXCL_PALLIATIVE:DOMAIN_GAP:CODE_SYSTEM]
    AND NOT EXISTS (
        SELECT 1
        FROM DEMO_HealthcareSynthea.conditions c
        JOIN HCLS.measure_value_set_codes vsc
          ON TRIM(c.code) = TRIM(vsc.code)                              -- [SN-07][AE-04]
         AND TRIM(vsc.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.1450',
                                          '2.16.840.1.113883.3.464.1004.2225')
        WHERE TRIM(c.patient_id) = TRIM(d.patient_id)
          AND c.code_system = 'SNOMED-CT'
          AND c.start_date BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    )
    AND NOT EXISTS (
        SELECT 1
        FROM DEMO_HealthcareSynthea.procedures p
        JOIN HCLS.measure_value_set_codes vsc
          ON TRIM(p.code) = TRIM(vsc.code)
         AND TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.2224'
        WHERE TRIM(p.patient_id) = TRIM(d.patient_id)
          AND p.code_system = 'SNOMED-CT'
          AND CAST(p.start_ts AT TIME ZONE 'GMT' AS DATE)
                BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    )

    -- EXCL_COLORECTAL_CA — [Spec2:EXCL_COLORECTAL_CA:DOMAIN_GAP:CODE_SYSTEM] [AE-07]
    AND NOT EXISTS (
        SELECT 1
        FROM DEMO_HealthcareSynthea.conditions c2
        JOIN HCLS.measure_value_set_codes vsc2
          ON TRIM(c2.code) = TRIM(vsc2.code)
         AND TRIM(vsc2.value_set_oid) = '2.16.840.1.113883.3.464.1004.1065'
        WHERE TRIM(c2.patient_id) = TRIM(d.patient_id)
          AND c2.code_system = 'SNOMED-CT'
          AND c2.start_date <= DATE '2025-12-31'
    )

    -- EXCL_COLECTOMY — [Spec2:EXCL_COLECTOMY:DOMAIN_GAP:CODE_SYSTEM] [AE-07]
    AND NOT EXISTS (
        SELECT 1
        FROM DEMO_HealthcareSynthea.procedures p2
        JOIN HCLS.measure_value_set_codes vsc3
          ON TRIM(p2.code) = TRIM(vsc3.code)
         AND TRIM(vsc3.value_set_oid) = '2.16.840.1.113883.3.464.1004.1250'
        WHERE TRIM(p2.patient_id) = TRIM(d.patient_id)
          AND p2.code_system = 'SNOMED-CT'
          AND CAST(p2.start_ts AT TIME ZONE 'GMT' AS DATE) <= DATE '2025-12-31'
    )
)

,CTE_COLONOSCOPY_EVENTS AS (
    SELECT DISTINCT TRIM(pr.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.procedures pr
    JOIN HCLS.measure_value_set_codes vsc
      ON TRIM(pr.code) = TRIM(vsc.code)
     AND TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1064'
    WHERE pr.code_system = 'SNOMED-CT'
      AND CAST(pr.start_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2016-01-01' AND DATE '2025-12-31'
    -- [Spec2:NUM_COLONOSCOPY:DOMAIN_GAP:CODE_SYSTEM] [AE-05]
)

,CTE_SIGMOIDOSCOPY_EVENTS AS (
    SELECT DISTINCT TRIM(pr.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.procedures pr
    JOIN HCLS.measure_value_set_codes vsc
      ON TRIM(pr.code) = TRIM(vsc.code)
     AND TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1102'
    WHERE pr.code_system = 'SNOMED-CT'
      AND CAST(pr.start_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2021-01-01' AND DATE '2025-12-31'
    -- [Spec2:NUM_SIGMOIDOSCOPY:DOMAIN_GAP:CODE_SYSTEM] [AE-05]
)

,CTE_CT_COLONOGRAPHY_EVENTS AS (
    SELECT DISTINCT TRIM(pr.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.procedures pr
    JOIN HCLS.measure_value_set_codes vsc
      ON TRIM(pr.code) = TRIM(vsc.code)
     AND TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1421'
    WHERE pr.code_system = 'SNOMED-CT'
      AND CAST(pr.start_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2021-01-01' AND DATE '2025-12-31'
    -- [Spec2:NUM_CT_COLONOGRAPHY:DOMAIN_GAP:CODE_SYSTEM] [AE-05]
)

,CTE_FOBT_EVENTS AS (
    SELECT DISTINCT TRIM(ob.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.observations ob
    JOIN HCLS.measure_value_set_codes vsc
      ON TRIM(ob.code) = TRIM(vsc.code)
     AND TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1959'
    WHERE CAST(ob.observation_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    -- [Spec2:NUM_FOBT:MAPPED] [AE-05]
)

,CTE_FITDNA_EVENTS AS (
    SELECT DISTINCT TRIM(ob.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.observations ob
    JOIN HCLS.measure_value_set_codes vsc
      ON TRIM(ob.code) = TRIM(vsc.code)
     AND TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1749'
    WHERE CAST(ob.observation_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2023-01-01' AND DATE '2025-12-31'
    -- [Spec2:NUM_FIT_DNA:MAPPED] [AE-05]
)

,CTE_QUALIFYING_EVENTS AS (
    SELECT DISTINCT patient_id
    FROM (
        SELECT patient_id FROM CTE_COLONOSCOPY_EVENTS
        UNION ALL
        SELECT patient_id FROM CTE_SIGMOIDOSCOPY_EVENTS
        UNION ALL
        SELECT patient_id FROM CTE_CT_COLONOGRAPHY_EVENTS
        UNION ALL
        SELECT patient_id FROM CTE_FOBT_EVENTS
        UNION ALL
        SELECT patient_id FROM CTE_FITDNA_EVENTS
    ) all_events
    -- [Spec2:NUM_COUNT_RULE:MAPPED]
)

,CTE_05_NUMERATOR AS (
    SELECT
        d.patient_id
        ,CASE WHEN q.patient_id IS NOT NULL THEN 1 ELSE 0 END AS in_numerator
    FROM CTE_04_EXCLUSIONS d
    LEFT JOIN CTE_QUALIFYING_EVENTS q
        ON TRIM(d.patient_id) = TRIM(q.patient_id)              -- [AE-01] LEFT JOIN + IS NOT NULL
)

,CTE_06_RESULT AS (
    SELECT DISTINCT
        TRIM(n.patient_id) AS member_id
        ,n.in_numerator     AS in_numerator
    FROM CTE_05_NUMERATOR n
)

SELECT
    member_id
    ,in_numerator
FROM CTE_06_RESULT;
```

## Spine structure (read-only context)

The spine is the certified SQL from the top through `CTE_05_NUMERATOR`. Columns each CTE exposes (never select a column that is not listed):

- `CTE_01_MEMBERS`(birth_date, death_date, patient_id)
- `CTE_02_ENROLLED`(patient_id)
- `CTE_03_DENOMINATOR`(age_at_anchor, birth_date, patient_id)
- `CTE_04_EXCLUSIONS`(age_at_anchor, birth_date, patient_id)
- `CTE_COLONOSCOPY_EVENTS`(patient_id)
- `CTE_SIGMOIDOSCOPY_EVENTS`(patient_id)
- `CTE_CT_COLONOGRAPHY_EVENTS`(patient_id)
- `CTE_FOBT_EVENTS`(patient_id)
- `CTE_FITDNA_EVENTS`(patient_id)
- `CTE_QUALIFYING_EVENTS`(patient_id)
- `CTE_05_NUMERATOR`(in_numerator, patient_id)

Member key: `patient_id`

**Rate model for this measure:** `CTE_05_NUMERATOR` is member-grain: one row per DENOMINATOR member (columns: in_numerator, patient_id) with a 0/1 `in_numerator` flag. Read FROM `CTE_05_NUMERATOR`; denominator = COUNT(DISTINCT patient_id); numerator = SUM(in_numerator).

## Analytical query rules

Use these rules whenever you write SQL on top of the certified spine.

You are a Teradata SQL expert for HEDIS / Medicare Part C/D Star Ratings analytics.

The certified measure population is ALREADY COMPUTED for you in a CTE named
CTE_05_NUMERATOR — one row per denominator member, carrying the measure's
dimension columns plus a 0/1 `in_numerator` flag. You are shown the leading WITH
chain that builds it (read-only context). Those CTEs are the frozen, approved
certified logic: do NOT modify, repeat, or rebuild any of them.

YOUR JOB: write ONLY the analytical tail that reads FROM the certified spine to
answer the user's question — a rate/count by some breakdown, a filtered subset, a
gap-to-star analysis, or an outreach list of members.

OUTPUT RULES:
- Output ONLY the tail. Begin with a comma `,` if you add analytical CTEs
  (e.g. `, CTE_ANALYTICAL AS ( SELECT ... FROM <spine CTE> ... )`), or with
  `SELECT` to go straight to the final query. Do NOT emit WITH. Do NOT repeat any
  spine CTE. No markdown fences, no prose.
- Follow the RATE MODEL in the user message EXACTLY. It tells you, for THIS
  measure, which spine CTE is the eligible denominator and how in_numerator is
  defined — CTE_05_NUMERATOR is NOT always a fact table with an in_numerator flag;
  for some measures it is only the set of numerator members. Never select a column
  that is not listed for the CTE you read from. Every result must be a
  subset/aggregation of the certified population — never redefine who is in it.

ENRICHMENT — adding a dimension not already on CTE_05_NUMERATOR:
- You MAY LEFT JOIN a customer/reference table to ADD descriptive columns for
  grouping, e.g. `LEFT JOIN <db>.providers pr ON TRIM(pr.id) = TRIM(n.<member_key>)`.
- LEFT JOIN ONLY, on the member key, one row per member. NEVER INNER JOIN, and
  NEVER put a WHERE/AND condition on the joined table that could drop members —
  that would change the denominator. Enrichment adds columns, never removes rows.

NAMES — NEVER GUESS:
- Use ONLY column and table names that appear in CTE_05_NUMERATOR, the SPINE, or
  the AUTHORITATIVE SCHEMAS block in the user message. If a table/column you need
  is not shown there, do not invent it — use the closest one that IS shown, or
  omit that enrichment. Fully-qualify reference/customer tables as db.table.

ANTI-FABRICATION:
- Use ONLY the certified rate / star band and the star cut points given in the
  user message. Never invent a certified rate and never hard-code star thresholds.
- Emit result_type = 'ANALYTICAL' on every output row.

TERADATA DIALECT:
- NULLIFZERO() in every rate denominator; TRIM() on join keys; LEFT JOIN + flag,
  never CASE WHEN EXISTS (Error 3771); CAST(col AT TIME ZONE 'GMT' AS DATE) for
  TIMESTAMP columns; no ORDER BY inside CTEs — only in the final SELECT.
- NTILE() is NOT supported in Teradata — do NOT use it. For the fixed bands defined
  in the spec (e.g. age tertiles 45–55 / 56–65 / 66–75) use CASE WHEN. To split rows
  into N equal-sized groups dynamically, use QUANTILE(N, <sort_col>) (returns 0..N-1)
  or arithmetic on a window function:
      CAST((ROW_NUMBER() OVER (ORDER BY <col>) - 1) * N / COUNT(*) OVER () AS INTEGER)
  — never NTILE(N) OVER (...).
- ALIASES: never use a Teradata reserved word as a CTE alias, a TABLE alias, OR an
  OUTPUT-COLUMN alias. Besides the comparison-operator keywords LT, GT, LE, GE, EQ,
  NE (and IN, IS, AS, BY, CS, SL, DAY), common OUTPUT names collide too — `value`,
  `zone`, `year`, `month`, `day`, `level`, `rank`, `row`, `type`, `position`,
  `summary`, `title`, `count` (e.g. `SELECT COUNT(*) AS value` fails Error 3706).
  Use descriptive, non-reserved names (low_tert, geo, zone_name, member_value,
  member_count) for both aliases and output columns.
- Geospatial: ST_Geometry columns use method-call syntax, e.g.
  `a.geom_col.ST_Within(b.geom_col) = 1` (returns 1/0). Build a geometry with
  `NEW ST_Geometry('POINT(<lon> <lat>)')` when you must construct one. Only call
  ST_ methods on columns whose DDL type is ST_GEOMETRY / SYSUDTLIB.ST_GEOMETRY.

### Prompt template (fill in per question)

```text
USER QUESTION: <question>

CERTIFIED SPINE CTEs: the "Spine structure" list above (member key: patient_id)
RATE MODEL: the "Rate model for this measure" paragraph above
CERTIFIED BASELINE: the "Certified result" table above (use verbatim, never recompute)
STAR CUT POINTS: the "Star cut points" list above (the only source for gap-to-star math)
DATABASES: customer/clinical tables DEMO_HealthcareSynthea; reference tables HCLS
SPINE: the Certified SQL above, from the top through CTE_05_NUMERATOR

Write ONLY the analytical tail (begin with ',' or SELECT) that answers the question,
reading FROM CTE_05_NUMERATOR (alias it, e.g. FROM CTE_05_NUMERATOR n).
```

### Example questions

- Break the rate down by age group, gender, race or geography and show which segments are furthest from the next star band.
- List members in the denominator who are not in the numerator (open care gaps) for outreach, with their key attributes.
- How many additional compliant members are needed to reach the next star band?
- Show the denominator, numerator and rate by provider, facility or payer.

## Measure definition (Spec 1 — approved)

### Spec 1 — Measure Concept Specification
### C02 · Colorectal Cancer Screening (COL)

---

#### Document Control

| Field            | Value                                              |
|------------------|-----------------------------------------------------|
| spec_type        | MEASURE_CONCEPT                                    |
| spec_version     | 1.0                                                 |
| measure_id       | C02                                                  |
| measure_label    | Colorectal Cancer Screening                         |
| hedis_label      | COL                                                  |
| program          | Medicare Advantage Part C Star Ratings              |
| reference        | NCQA HEDIS MY2025 Technical Specifications Vol 2 — page reference not present in extracted CMS Technical Notes (pp. 39–40 of cms_2025.pdf covered reporting requirements and cut points only) |
| cms_cmit         | [NOT FOUND IN EXTRACTED CMS TECHNICAL NOTES — TBD]  |
| authored_date    | 2026-07-20                                          |
| status           | ACTIVE                                               |

> **Agent note:** The extracted CMS Technical Notes for C02 (pp. 39–40) contained only reporting-requirements applicability and Star cut points. All narrative fields below (description, weighting, exclusions, data source, age bounds, lookback windows) were NOT present in the extraction and have been populated from standard NCQA HEDIS COL measure specifications (domain knowledge) because Spec 1 requires a complete concept model. These fields should be validated against the authoritative NCQA HEDIS MY2025 Technical Specifications Vol 2 and the full CMS Star Ratings Technical Notes document before promotion to ACTIVE status.

---

#### 1. Measure Summary

| Field                | Value                                                                 |
|----------------------|------------------------------------------------------------------------|
| description          | Percent of Medicare Advantage enrollees aged 50–75 who had appropriate screening for colorectal cancer [populated from HEDIS COL specification — NOT in extracted CMS notes; age range corrected per business request to 50–75] |
| direction            | Higher is better                                                      |
| weighting_category   | Process Measure [NOT FOUND IN EXTRACTED NOTES — inferred from measure type] |
| weighting_value      | 1 [NOT FOUND IN EXTRACTED NOTES — inferred]                            |
| statistical_method   | Clustering [NOT FOUND IN EXTRACTED NOTES — inferred]                   |
| case_mix_adjusted    | No [NOT FOUND IN EXTRACTED NOTES — inferred]                           |
| display_format       | Integer percentage (no decimal)                                       |
| meaningful_measure   | Wellness and Prevention [NOT FOUND IN EXTRACTED NOTES — inferred]      |
| primary_data_source  | HEDIS [NOT FOUND IN EXTRACTED NOTES — inferred]                        |
| data_source_category | Health and Drug Plans [NOT FOUND IN EXTRACTED NOTES — inferred]        |

> **Agent note:** Reporting requirements extracted verbatim: 1876 Cost = Yes; CCP w/o SNP = Yes; CCP with SNP = Yes; CCP with Only I-SNP = Yes; MSA = Yes; PDP = No; PFFS = Yes.

---

#### 2. Measurement Period

| Concept                  | Rule                                                                 |
|---------------------------|-----------------------------------------------------------------------|
| measurement_year          | The calendar year being measured (e.g. 2025)                         |
| measurement_period_start  | January 1 of measurement_year                                        |
| measurement_period_end    | December 31 of measurement_year                                      |
| anchor_date               | December 31 of measurement_year (used for age calculation)           |
| numerator_lookback_start  | VARIES BY EVENT TYPE — see section 6.1 / 6.2                          |
| numerator_lookback_end    | December 31 of measurement_year                                      |
| numerator_window_note     | Multiple concurrent lookback windows depending on qualifying test type: FOBT/gFOBT = 1 year; FIT-DNA (sDNA-FIT) = 3 years; Flexible Sigmoidoscopy = 5 years; CT Colonography = 5 years; Colonoscopy = 10 years |

> **Agent note:** This measure does NOT use a single uniform lookback window. Each qualifying event type in section 6 carries its own independently-evaluated lookback window. Do not apply a single lookback window (e.g. the 2-year window used by C01/BCS) to all event types.

---

#### 3. Rate Formula

```
Rate = (Numerator / Denominator) × 100

Reported as: integer percentage, no decimal place
```

---

#### 4. Star Rating Cut Points

| Star Band | Rule                          |
|-----------|--------------------------------|
| 1 star    | < 53 %                         |
| 2 stars   | >= 53 % to < 67 %              |
| 3 stars   | >= 67 % to < 75 %              |
| 4 stars   | >= 75 % to < 82 %              |
| 5 stars   | >= 82 %                        |

> Source: CMS Star Ratings Technical Notes, cms_2025.pdf, pp. 39–40. Cut points copied exactly as extracted.

---

#### 5. Denominator — Conceptual Definition

**Plain language:** Medicare Advantage enrollees aged 50 to 75 as of December 31 of the measurement year who were continuously enrolled during the measurement year, with no evidence of colorectal cancer or total colectomy.

> **Agent note:** Age bounds (50–75) were corrected per business request from an earlier 45–75 draft value. The 45–75 figure was NOT present in the extracted CMS Technical Notes text and was populated from domain knowledge; the current 50–75 value likewise has not been independently verified against the authoritative NCQA HEDIS MY2025 Vol 2 spec. Validate before certifying.

##### 5.1 Denominator Inclusion Criteria

| Concept ID       | Concept Name          | Rule                                                                 | Required |
|-------------------|------------------------|------------------------------------------------------------------------|----------|
| DENOM_AGE         | Age at anchor date     | Age >= 50 AND age <= 75 as of December 31 of measurement_year          | YES      |
| DENOM_PLAN_TYPE   | Plan type              | Medicare Advantage enrollee — includes 1876 Cost, CCP w/o SNP, CCP with SNP, CCP with Only I-SNP, MSA, PFFS. Excludes standalone PDP (per extracted reporting_requirements: PDP = No) | YES |
| DENOM_ENROLL      | Continuous enrollment  | Continuously enrolled during the measurement year, allowing no more than one gap in enrollment of up to 45 days | YES      |
| DENOM_ALIVE       | Vital status           | Member must not have died prior to December 31 of measurement_year     | YES      |

##### 5.2 Denominator Minimum Enrollment Threshold

| Concept ID        | Rule                                                                             |
|--------------------|------------------------------------------------------------------------------------|
| DENOM_MIN_500      | Contracts with enrollment < 500 as of the July measurement_year enrollment report are excluded from this measure entirely |
| DENOM_RELIABILITY  | Contracts with enrollment >= 500 and < 1,000 AND measure score reliability < 0.7 are excluded |

> **Agent note:** These thresholds apply at the contract (plan) level, not the member level. If the customer is a single-contract payer with < 500 members in the eligible cohort, flag result as BELOW_THRESHOLD and suppress the star rating.

---

#### 6. Numerator — Conceptual Definition

**Plain language:** At least one qualifying colorectal cancer screening test — colonoscopy, flexible sigmoidoscopy, CT colonography, FOBT/gFOBT, or FIT-DNA (sDNA-FIT) — completed within that test's own lookback window ending December 31 of the measurement year.

##### 6.1 Numerator Inclusion Events

| Concept ID           | Concept Name                | Rule                                                                          |
|------------------------|-------------------------------|------------------------------------------------------------------------------------|
| NUM_COLONOSCOPY        | Colonoscopy event            | At least one colonoscopy during the 10-year lookback window                       |
| NUM_SIGMOIDOSCOPY      | Flexible sigmoidoscopy event | At least one flexible sigmoidoscopy during the 5-year lookback window             |
| NUM_CT_COLONOGRAPHY    | CT colonography event        | At least one CT colonography during the 5-year lookback window                    |
| NUM_FOBT               | FOBT/gFOBT event              | At least one FOBT/gFOBT lab test during the 1-year lookback window                 |
| NUM_FIT_DNA            | FIT-DNA (sDNA-FIT) event      | At least one FIT-DNA (sDNA-FIT) lab test during the 3-year lookback window         |
| NUM_DATE_WINDOW        | Date of service window        | Service date >= event-specific numerator_lookback_start AND <= numerator_lookback_end (December 31 of measurement_year) |
| NUM_COUNT_RULE         | Minimum event count           | One qualifying event of ANY type satisfies the numerator. Multiple events, or events across multiple types, do not increase the numerator count beyond 1 per member |

##### 6.2 Qualifying Event Code Sets

> **Agent note — Value Set Resolution:** All qualifying event code sets below reference HEDIS MY2025 Value Set Directory (VSD) value sets by name and OID. Do NOT enumerate individual CPT/HCPCS/LOINC/SNOMED codes in Spec 1 — resolve the full code list from the certified VSD at compute time via the code system(s) declared available in Spec 2.

###### Colonoscopy

> Lookback window: 10 years (numerator_lookback_start = January 1 of (measurement_year − 9))

**VALUE_SET_REFERENCES**
| Value Set Name | OID                                        | Code Systems                          |
|-----------------|---------------------------------------------|------------------------------------------|
| Colonoscopy     | 2.16.840.1.113883.3.464.1004.1064            | CPT, HCPCS, SNOMED CT US Edition          |

###### Flexible Sigmoidoscopy

> Lookback window: 5 years (numerator_lookback_start = January 1 of (measurement_year − 4))

**VALUE_SET_REFERENCES**
| Value Set Name             | OID                                        | Code Systems                          |
|------------------------------|-----------------------------------------------|------------------------------------------|
| Flexible Sigmoidoscopy       | 2.16.840.1.113883.3.464.1004.1102              | CPT, HCPCS, SNOMED CT US Edition          |

###### CT Colonography

> Lookback window: 5 years (numerator_lookback_start = January 1 of (measurement_year − 4))

**VALUE_SET_REFERENCES**
| Value Set Name     | OID                                        | Code Systems                            |
|----------------------|-----------------------------------------------|--------------------------------------------|
| CT Colonography       | 2.16.840.1.113883.3.464.1004.1421              | CPT, LOINC, SNOMED CT US Edition            |

###### FOBT / gFOBT

> Lookback window: 1 year (numerator_lookback_start = January 1 of measurement_year)

**VALUE_SET_REFERENCES**
| Value Set Name                     | OID                                        | Code Systems                                    |
|----------------------------------------|-----------------------------------------------|------------------------------------------------------|
| FOBT Lab Test                          | 2.16.840.1.113883.3.464.1004.1959              | CPT, HCPCS, LOINC, SNOMED CT US Edition               |

> **Agent note:** Per business request, this value set was resolved to a single code set (FOBT Lab Test) rather than the two candidate value sets (FOBT Lab Test / FOBT Test Result or Finding) previously listed. The request referenced a value set named "FOBT Last Test," which does not exist under that name in the spec or the extracted VSD; "FOBT Lab Test" was used as the closest and most plausible match. Confirm this interpretation is correct before certifying.

###### FIT-DNA (sDNA-FIT)

> Lookback window: 3 years (numerator_lookback_start = January 1 of (measurement_year − 2))

**VALUE_SET_REFERENCES**
| Value Set Name    | OID                                        | Code Systems      |
|----------------------|-----------------------------------------------|-----------------------|
| sDNA FIT Lab Test     | 2.16.840.1.113883.3.464.1004.1749              | CPT, LOINC            |

---

#### 7. Exclusions — Conceptual Definitions

> Exclusions are applied after the denominator cohort is built and before the numerator is evaluated. Each exclusion removes members entirely.
> Order of application: EXCL_HOSPICE → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY → EXCL_COLORECTAL_CA → EXCL_COLECTOMY

##### 7.1 Hospice Exclusion

| Concept ID   | Rule                                                                          |
|---------------|---------------------------------------------------------------------------------|
| EXCL_HOSPICE  | Exclude members enrolled in hospice OR receiving hospice services at any time during measurement_period |

**VALUE_SET_REFERENCES**
| Value Set Name        | OID                                        | Code Systems                          |
|--------------------------|-----------------------------------------------|-------------------------------------------|
| Hospice Encounter         | 2.16.840.1.113883.3.464.1004.1761              | HCPCS, SNOMED CT US Edition, UBREV        |
| Hospice Intervention      | 2.16.840.1.113883.3.464.1004.1762              | CPT, HCPCS, SNOMED CT US Edition          |

##### 7.2 Palliative Care Exclusion

| Concept ID       | Rule                                                                      |
|--------------------|-------------------------------------------------------------------------------|
| EXCL_PALLIATIVE    | Exclude members receiving palliative care during measurement_year              |

**VALUE_SET_REFERENCES**
| Value Set Name              | OID                                        | Code Systems              |
|--------------------------------|-----------------------------------------------|--------------------------------|
| Palliative Care Encounter        | 2.16.840.1.113883.3.464.1004.1450              | HCPCS, SNOMED CT US Edition    |
| Palliative Care Assessment       | 2.16.840.1.113883.3.464.1004.2225              | SNOMED CT US Edition            |
| Palliative Care Intervention     | 2.16.840.1.113883.3.464.1004.2224              | SNOMED CT US Edition            |

##### 7.3 Institutional SNP (I-SNP) Exclusion

| Concept ID | Rule                                                                             |
|-------------|--------------------------------------------------------------------------------------|
| EXCL_ISNP   | Exclude members age >= 66 enrolled in I-SNP at any time during measurement_year        |
| age_gate    | >= 66 as of anchor_date                                                                |
| source      | Plan type / SNP type indicator in enrollment or eligibility data — no HEDIS VSD value set applies |

##### 7.4 Long-Term Institutional (LTI) Exclusion

| Concept ID | Rule                                                                              |
|-------------|---------------------------------------------------------------------------------------|
| EXCL_LTI    | Exclude members age >= 66 with LTI flag in CMS Monthly Membership Detail File          |
| age_gate    | >= 66 as of anchor_date                                                               |
| source      | External CMS file — not derivable from claims; no HEDIS VSD value set applies         |

##### 7.5 Frailty + Advanced Illness Exclusion

> Members age >= 66 must meet BOTH frailty AND advanced illness criteria. Age gate: >= 66 as of anchor_date.

###### 7.5a Frailty Criterion

At least two frailty indicators on different dates during measurement_period.

**VALUE_SET_REFERENCES**
| Value Set Name       | OID                                        | Code Systems                     |
|-------------------------|-----------------------------------------------|--------------------------------------|
| Frailty Device            | 2.16.840.1.113883.3.464.1004.1530              | HCPCS                                 |
| Frailty Diagnosis          | 2.16.840.1.113883.3.464.1004.1531              | ICD10CM, SNOMED CT US Edition          |
| Frailty Symptom            | 2.16.840.1.113883.3.464.1004.1533              | ICD10CM, SNOMED CT US Edition          |
| Frailty Encounter          | 2.16.840.1.113883.3.464.1004.1532              | CPT, HCPCS                             |

###### 7.5b Advanced Illness Criterion

A member meets the advanced illness criterion if they satisfy **any one** of the following three pathways:

1. At least two qualifying visits with an advanced illness diagnosis on different dates during the measurement period. Qualifying visit types: outpatient, observation, ED, telephone, e-visit, nonacute inpatient.
2. At least one acute inpatient encounter with an advanced illness diagnosis during the measurement period.
3. Dispensed at least one dementia medication during the measurement period.

**VALUE_SET_REFERENCES**
| Value Set Name         | OID                                        | Code Systems                        |
|--------------------------|-----------------------------------------------|------------------------------------------|
| Advanced Illness           | 2.16.840.1.113883.3.464.1004.1465              | ICD10CM, SNOMED CT US Edition             |
| Acute Inpatient             | TBD — pending OID lookup                       | TBD — expected CPT/HCPCS/UBREV/SNOMED CT US Edition |
| Dementia Medications        | TBD — pending OID lookup                       | TBD — expected NDC / RxNorm               |

> **Agent note:** Pathways 2 and 3 were added per business request to align §7.5b with the standard three-pathway NCQA HEDIS Advanced Illness criterion structure used in comparable measures (e.g., COL, BCS). Previously, Spec 1 captured only pathway 1 (two qualifying visits), which would have missed members who qualify solely via an acute inpatient stay or a dispensed dementia medication — an under-exclusion risk that would incorrectly hold frail/end-of-life members to the screening standard. This three-pathway wording has NOT been independently verified against the authoritative NCQA HEDIS MY2025 Technical Specification text for this measure and should carry the same "not verified against source" caveat as other inferred fields in this document. The "Acute Inpatient" and "Dementia Medications" value sets are placeholders (OID = TBD) pending confirmation of the correct value set names/OIDs; do not treat these as resolvable until OIDs are supplied and Spec 2/Spec 3 are updated accordingly.

##### 7.6 Colorectal Cancer Diagnosis History Exclusion

| Concept ID          | Rule                                                                       |
|-----------------------|---------------------------------------------------------------------------------|
| EXCL_COLORECTAL_CA    | Exclude members with a diagnosis of colorectal cancer at any time in the member's history through December 31 of measurement_year |
| source_types          | Claim / Condition                                                                |
| history_window        | Anytime in history                                                               |

**VALUE_SET_REFERENCES**
| Value Set Name      | OID                                        | Code Systems                     |
|------------------------|-----------------------------------------------|--------------------------------------|
| Colorectal Cancer       | 2.16.840.1.113883.3.464.1004.1065              | ICD10CM, SNOMED CT US Edition          |

##### 7.7 Total Colectomy Exclusion

| Concept ID       | Rule                                                                             |
|--------------------|--------------------------------------------------------------------------------------|
| EXCL_COLECTOMY      | Exclude members with evidence of total colectomy at any time in the member's history through December 31 of measurement_year |
| source_types        | Claim / Procedure                                                                    |
| history_window       | Anytime in history                                                                    |

**VALUE_SET_REFERENCES**
| Value Set Name   | OID                                        | Code Systems                          |
|--------------------|------------------------------------------------|-------------------------------------------|
| Total Colectomy     | 2.16.840.1.113883.3.464.1004.1250              | CPT, ICD10PCS, SNOMED CT US Edition       |

---

#### 8. Analytical Query Concepts

| Concept ID           | Concept Name         | Analytical Use                                                          |
|------------------------|-------------------------|-------------------------------------------------------------------------|
| AQ_AGE_TERTILE          | Age tertile            | Divide 50–75 range into thirds: 50–59, 60–69, 70–75                     |
| AQ_AGE_BAND_5           | 5-year age band         | Standard 5-year bands: 50–54, 55–59, 60–64, 65–69, 70–75                 |
| AQ_GEOGRAPHY            | Geographic segment     | Zip, county, state from member address                                   |
| AQ_RACE_ETHNICITY       | Race and ethnicity     | Health equity sub-analysis                                                |
| AQ_PROVIDER             | Attributed provider    | PCP or last-touch provider for outreach targeting                         |
| AQ_TREND                | Time trend             | Rolling monthly or quarterly rate within certified denom                  |
| AQ_GAP_TO_CUT           | Gap to cut point        | Members needed in numerator to reach next star band                       |
| AQ_OUTREACH             | Outreach target list    | Denominator members NOT in numerator, by AQ dimension                     |
| AQ_TEST_TYPE            | Screening test type mix | Distribution of numerator satisfaction across colonoscopy / sigmoidoscopy / CT colonography / FOBT / FIT-DNA, for outreach and modality-targeting analysis |

> **Guardrail:** All analytical queries must operate within the certified denominator. The agent must NOT redefine eligibility criteria. All analytical output must be labeled result_type = ANALYTICAL.

---

#### 9. Output Schema — Certified Result Record

| Field             | Type      | Description                                         |
|--------------------|-----------|-------------------------------------------------------|
| measure_id          | VARCHAR   | C02                                                    |
| measurement_year    | INTEGER   | e.g. 2025                                              |
| customer_id         | VARCHAR   | Payer/customer identifier                              |
| run_id              | VARCHAR   | Unique execution identifier                            |
| run_ts              | TIMESTAMP | When the calculation was executed                      |
| denominator_count   | INTEGER   | Count of eligible members after exclusions              |
| numerator_count     | INTEGER   | Count of denominator members with qualifying event      |
| rate                | DECIMAL   | numerator / denominator × 100                          |
| rate_display        | INTEGER   | Rounded integer for CMS display                        |
| star_band           | INTEGER   | 1–5 based on cut points                                |
| result_type         | VARCHAR   | CERTIFIED or ANALYTICAL                                |
| below_threshold     | BOOLEAN   | TRUE if contract enrollment < 500                       |
| spec1_version       | VARCHAR   | Version of this spec used                              |
| spec2_version       | VARCHAR   | Version of customer data model spec used               |
| spec3_version       | VARCHAR   | Version of compute spec used                            |
| gap_flags           | VARCHAR   | Pipe-delimited list of gap flags from Spec 2            |

---

#### 10. Agent Instructions

##### 10.1 Certified SQL Agent Rules

1. Always resolve code sets from section 6.2 and 7 using the code system declared in Spec 2. If only SNOMED-CT is available for a value set that also has CPT/HCPCS variants, use SNOMED codes and set gap_flag = DOMAIN_GAP:CODE_SYSTEM.
2. Apply exclusions in the order specified in section 7 header: EXCL_HOSPICE → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY → EXCL_COLORECTAL_CA → EXCL_COLECTOMY.
3. Exclusions EXCL_ISNP, EXCL_LTI, and EXCL_FRAILTY (7.5a/7.5b combined) are age-gated (>= 66). Do not apply to members below the age gate (50–65).
4. If Spec 2 marks a concept as MISSING, omit that exclusion and record it in gap_flags. This is especially likely for EXCL_LTI (external CMS file) and EXCL_ISNP (plan/SNP indicator).
5. If Spec 2 marks a concept as PROXY, apply proxy logic and set gap_flag = PROXY:<CONCEPT_ID>.
6. If Spec 2 marks a concept as SYNTHETIC_DATA_ARTIFACT, omit entirely and set gap_flag = SYNTHETIC_DATA_ARTIFACT:<CONCEPT_ID>.
7. **Measure-specific rule — independent lookback windows:** Each of the five numerator event types (NUM_COLONOSCOPY, NUM_SIGMOIDOSCOPY, NUM_CT_COLONOGRAPHY, NUM_FOBT, NUM_FIT_DNA) carries its OWN lookback window as defined in section 6.1. Evaluate each event type independently against its own window — never apply the 10-year colonoscopy window, or any other single window, uniformly across all event types.
8. A member satisfies the numerator with one qualifying event across ANY event type. Do not require the same event type for all members, and do not require multiple event types for a single member.
9. Apply EXCL_COLORECTAL_CA and EXCL_COLECTOMY as full-history exclusions (any date up to and including December 31 of measurement_year) — these are NOT limited to the measurement_year or to any numerator lookback window.
10. Never produce a rate from a denominator of zero. Return denominator_count = 0 and rate = NULL.

##### 10.2 Analytical Query Agent Rules

1. Parse the user's natural language query to identify the AQ dimension(s) from section 8.
2. Apply the full certified denominator before segmenting. Never segment before filtering.
3. For AQ_AGE_TERTILE: divide the 50–75 range as follows — Lower: 50–59, Mid: 60–69, Upper: 70–75.
4. For AQ_GAP_TO_CUT: compute members_needed = CEIL((cut_point / 100) × denominator_count) − numerator_count for each band above current rate.
5. For AQ_OUTREACH: return denominator members with no qualifying event across ANY of the five event types, sorted by AQ dimension then age DESC.
6. For AQ_TEST_TYPE: report numerator satisfaction broken out by which event type(s) satisfied the member (colonoscopy / sigmoidoscopy / CT colonography / FOBT / FIT-DNA); a member counted once in the certified numerator may appear in multiple test-type buckets if multiple event types are present, but the certified numerator count itself must never double-count a member.
7. Always label output result_type = ANALYTICAL.
8. Include the certified rate alongside every sub-analysis.
9. Refuse any query that redefines denominator eligibility (e.g., different age bounds or lookback windows). Explain why and offer the nearest valid alternative within the certified denominator and section 6 windows.

## SQL generation rules (how the certified SQL was produced)

These are the rules the certified SQL was generated and validated against; keep any SQL you write consistent with them.

You are a Teradata SQL expert specialising in HEDIS and Medicare Part C/D Star Ratings.

OUTPUT RULES:
- Output ONLY valid, executable Teradata SQL.
- NO markdown code fences. NO explanation text. NO comments outside the SQL.
- The first characters of your response must be WITH or SELECT.

REQUIRED CTE CHAIN (follow exactly as specified in Spec 3):
  CTE_01_MEMBERS → CTE_02_ENROLLED → CTE_03_DENOMINATOR →
  CTE_04_EXCLUSIONS → CTE_05_NUMERATOR → CTE_06_RESULT

FINAL SELECT must return one row per denominator member with columns:
  member_id   VARCHAR  — trimmed, deduplicated
  in_numerator INTEGER  — 0 or 1 (never NULL, never any other value)

TERADATA DIALECT RULES (all mandatory):
1. JOIN keys:     TRIM() every column used in a join condition.
2. Rate formula:  CAST(numerator AS DECIMAL(10,4)) / NULLIFZERO(CAST(denominator AS DECIMAL(10,4)))
3. Numerator (CRITICAL — Teradata Error 3771):
                  EXISTS is ILLEGAL inside a CASE expression. NEVER write
                  `CASE WHEN EXISTS (...)` or `CASE WHEN NOT EXISTS (...)`.
                  Instead: put the numerator event in its own CTE that SELECTs the
                  DISTINCT member key, LEFT JOIN it, and test for NULL:
                      WITH CTE_05_NUMERATOR AS (
                          SELECT DISTINCT TRIM(c.member_id) AS member_id
                          FROM claims c WHERE c.code IN (...)
                      )
                      SELECT m.member_id,
                             CASE WHEN n.member_id IS NOT NULL THEN 1 ELSE 0 END AS in_numerator
                      FROM CTE_03_DENOMINATOR m
                      LEFT JOIN CTE_05_NUMERATOR n ON n.member_id = TRIM(m.member_id)
                  (A bare `WHERE EXISTS (...)` inside a CTE is fine — only EXISTS
                  inside a CASE WHEN is forbidden.)
4. Dates vs timestamps (TYPE-AWARE — Teradata Error 5407):
                  - TIMESTAMP / TIMESTAMP WITH TIME ZONE columns (naming convention
                    *_ts, e.g. start_ts, study_ts, observation_ts): wrap as
                    CAST(col AT TIME ZONE 'GMT' AS DATE) before any date comparison.
                  - DATE columns (naming convention *_date, e.g. start_date,
                    end_date, birth_date) are ALREADY DATE: compare them directly
                    to DATE literals. NEVER write CAST(<x>_date AT TIME ZONE 'GMT'
                    AS DATE) — AT TIME ZONE on a DATE is illegal (Error 5407).
5. Column names: use the EXACT table and column names from Spec 2 / the schema
                  profile. Do NOT assume raw source-system names — synthetic/Synthea
                  conventions like `start`/`stop` are usually renamed to
                  `start_date`/`end_date` or `start_ts`/`stop_ts`. A wrong name is
                  Error 3707/5628. `start`/`end` are also reserved words.
6. No ORDER BY inside subqueries or CTEs — only the final SELECT may ORDER.
7. Trailing spaces: detect with CHARACTERS(col) > CHARACTERS(TRIM(col)).
7b. NTILE() is NOT supported in Teradata — never use it. For fixed value bands use
                  CASE WHEN; to split into N equal groups use QUANTILE(N, <col>)
                  (returns 0..N-1) or CAST((ROW_NUMBER() OVER (ORDER BY <col>) - 1)
                  * N / COUNT(*) OVER () AS INTEGER) — never NTILE(N) OVER (...).
8. Enrollment:    overlap pattern — enroll_start <= period_end AND enroll_end >= period_start.
9. Code IN lists: use ONLY the resolved code sets provided, or join to
                  measure_value_set_codes per the DATA DICTIONARY below. Never hardcode
                  clinical codes, and never read `code` from measure_value_sets.

ANNOTATION SCHEMA (required on every concept-referencing clause):
  -- [Spec2:CONCEPT_ID:STATUS]   e.g.  -- [Spec2:NUM_MAMMOGRAM:DOMAIN_GAP]
  -- [SN-XX]                     for every Spec 2 schema note applied
  -- [AE-XX]                     for every Spec 2 agent execution note applied

GAP HANDLING:
  MAPPED                   → use the source column directly.
  DOMAIN_GAP or PROXY      → use the proxy code approach from Spec 2 schema notes.
  MISSING                  → omit from SQL; LEFT JOIN produces NULL → member excluded.
  SYNTHETIC_DATA_ARTIFACT  → omit entirely; do not reference any table for this concept.

RESULT TYPE:
  CERTIFIED   → use strict enrollment logic from Spec 2.
  ANALYTICAL  → use relaxed / analytical enrollment logic from Spec 2.

REFERENCE (CONFIGURATION) TABLE DATA DICTIONARY — database HCLS:
The value sets live in THREE tables. Their columns are FIXED — use exactly these:

- HCLS.measure_value_set_codes   ← THE CODES ARE HERE. Columns:
      value_set_oid, value_set_name, value_set_version (DATE), code, code_desc,
      code_system, measurement_year (INTEGER)
- HCLS.measure_value_sets        ← value-set HEADER ONLY — HAS NO `code` COLUMN. Columns:
      measure_id, measurement_year, measure_name, value_set_name, value_set_oid
- HCLS.measure_to_value_set      ← approved concept→value-set map (per measure/year). Columns:
      measure_id, measure_component, concept_id, value_set_oid, lookback_years,
      lookback_note, measurement_year, applies_to_age_min, applies_to_age_max, notes

MEDICATION LISTS (drug-based concepts) live in the parallel trio — same shape:
- HCLS.measure_medication_list_codes ← THE DRUG CODES ARE HERE. Columns:
      medication_list_oid, medication_list_name, code, code_system (RxNorm/NDC),
      generic_name, brand_name, measurement_year (INTEGER)
- HCLS.measure_to_medication_list    ← approved concept→medication-list map. Columns mirror
      measure_to_value_set with `medication_list_oid` in place of `value_set_oid`.
A concept whose RESOLVED CODE SETS are RxNorm/NDC is a MEDICATION concept: join it to the
customer's MEDICATIONS / pharmacy table's drug-code column — NOT procedures/conditions.

CRITICAL: codes are in HCLS.measure_value_set_codes (clinical) / HCLS.measure_medication_list_codes
(drug). NEVER select or join `code` from HCLS.measure_value_sets — it does not have that column
(Error 5628).

YEAR-SCOPING (mandatory): all three tables carry measurement_year. The SAME OID can
hold a DIFFERENT code list per year, so EVERY join to measure_value_set_codes MUST
also filter measurement_year = <YEAR> — otherwise codes from multiple years union
together and the measure is wrong.

CODE RESOLUTION — two allowed patterns:
1. PREFERRED: use the literal codes under "=== RESOLVED CODE SETS ===" as IN-lists.
2. If a value set is too large to inline, JOIN to the codes table by OID + year:
       JOIN HCLS.measure_value_set_codes vsc
         ON TRIM(clinical.code) = TRIM(vsc.code)
        AND TRIM(vsc.value_set_oid) = '<value_set_oid from Spec 3>'
        AND vsc.measurement_year = <YEAR>
   or resolve every OID for a concept via HCLS.measure_to_value_set first:
       JOIN HCLS.measure_to_value_set mt
         ON mt.measure_id='<MID>' AND mt.measurement_year=<YEAR> AND mt.concept_id='<CONCEPT>'
       JOIN HCLS.measure_value_set_codes vsc
         ON TRIM(vsc.value_set_oid) = TRIM(mt.value_set_oid)
        AND vsc.measurement_year = mt.measurement_year
