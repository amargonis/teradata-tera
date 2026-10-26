---
name: stars-c09-2025
description: >-
  STARS measure C09 (Eye Exam to Check for Damage from Diabetes), measurement year 2025. Trigger when the user asks about
  C09 or "Eye Exam to Check for Damage from Diabetes", e.g. "what is our C09 rate", "C09 star rating", "show C09 care gaps",
  "who is missing C09", "break C09 down by age/provider/region", or to build a C09 dashboard.
  Uses the approved, EXPLAIN-validated certified Teradata SQL as the single source of truth for the
  denominator, numerator, rate and star band, and writes analytical queries on top of it.
---

# C09 — Eye Exam to Check for Damage from Diabetes (2025)

Percent of plan members with diabetes who had an eye exam to check for damage from diabetes during the year

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
| Certified rate | 40.75% |
| Star band | 1 |
| Denominator | 719 |
| Numerator | 293 |
| Run time | 2026-08-03 22:56:41 |

### Star cut points

- 1 star: < 57 %
- 2 stars: >= 57 % to < 70 %
- 3 stars: >= 70 % to < 77 %
- 4 stars: >= 77 % to < 83 %
- 5 stars: >= 83 %

## Certified SQL

Run on Teradata as-is to reproduce the certified population. The final SELECT returns one row per denominator member (`member_id`, `in_numerator`).

```sql
WITH
CTE_01_MEMBERS AS (
    SELECT DISTINCT
        TRIM(p.patient_id)        AS patient_id            -- [SN-04][AE-09]
        ,p.birth_date
        ,p.death_date
        ,p.race
        ,p.ethnicity
        ,p.zip
        ,p.city
        ,p.state
        ,p.county
    FROM DEMO_HealthcareSynthea.patients p
    WHERE (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM p.birth_date)) BETWEEN 18 AND 75
        -- [Spec2:DENOM_AGE:MAPPED]
      AND (p.death_date IS NULL OR p.death_date > DATE '2025-12-31')
        -- [Spec2:DENOM_ALIVE:MAPPED]
)
,CTE_ENROLL_GOV_SEGMENTS AS (
    SELECT
        TRIM(pt.patient_id)                                AS patient_id  -- [SN-04]
        ,CAST(pt.start_year AT TIME ZONE 'GMT' AS DATE)     AS seg_start   -- [SN-03]
        ,CAST(pt.end_year   AT TIME ZONE 'GMT' AS DATE)     AS seg_end
    FROM DEMO_HealthcareSynthea.payer_transitions pt
    INNER JOIN DEMO_HealthcareSynthea.payers pay
        ON TRIM(pt.primary_payer_id) = TRIM(pay.payer_id)                 -- [SN-04]
    WHERE pay.payer_type = 'GOVERNMENT'                                    -- [Spec2:DENOM_PLAN_TYPE:PROXY][SN-06]
      AND pay.payer IN ('Medicare','Dual Eligible')
      AND CAST(pt.start_year AT TIME ZONE 'GMT' AS DATE) <= DATE '2025-12-31'
      AND CAST(pt.end_year   AT TIME ZONE 'GMT' AS DATE) >= DATE '2025-01-01'
)
,CTE_ENROLL_ORDERED AS (
    SELECT
        patient_id, seg_start, seg_end
        ,LAG(seg_end) OVER (PARTITION BY patient_id ORDER BY seg_start) AS prev_seg_end
    FROM CTE_ENROLL_GOV_SEGMENTS
)
,CTE_ENROLL_GAPCHECK AS (
    SELECT
        patient_id
        ,MIN(seg_start) AS coverage_start
        ,MAX(seg_end)   AS coverage_end
        ,MAX(CASE WHEN prev_seg_end IS NOT NULL
                    AND seg_start > prev_seg_end + INTERVAL '1' DAY
                  THEN 1 ELSE 0 END) AS has_gap
    FROM CTE_ENROLL_ORDERED
    GROUP BY patient_id
)
,CTE_02_ENROLLED AS (
    SELECT DISTINCT TRIM(g.patient_id) AS patient_id
    FROM CTE_ENROLL_GAPCHECK g
    WHERE g.coverage_start <= DATE '2025-01-01'
      AND g.coverage_end   >= DATE '2025-12-31'
      AND g.has_gap = 0
    -- [Spec2:DENOM_ENROLL:CERTIFIED:PROXY][SN-05][AE-02]
)
,CTE_03_DENOMINATOR AS (
    SELECT
        m.patient_id
        ,m.birth_date
        ,m.race
        ,m.ethnicity
        ,m.zip
        ,m.city
        ,m.state
        ,m.county
        ,(EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) AS age_at_anchor
        ,CASE
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 18 AND 37 THEN 'Lower (18-37)'
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 38 AND 56 THEN 'Mid (38-56)'
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 57 AND 75 THEN 'Upper (57-75)'
         END                                                    AS age_tertile        -- [Spec2:AQ_AGE_TERTILE:MAPPED]
        ,CAST(FLOOR((EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) / NULLIFZERO(5))*5 AS VARCHAR(10))
            || '-' ||
         CAST(FLOOR((EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) / NULLIFZERO(5))*5+4 AS VARCHAR(10))
                                                                 AS age_band_5         -- [Spec2:AQ_AGE_BAND_5:MAPPED]
    FROM CTE_01_MEMBERS m
    INNER JOIN CTE_02_ENROLLED e
        ON TRIM(m.patient_id) = TRIM(e.patient_id)
    WHERE EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.conditions c
        WHERE TRIM(c.patient_id) = TRIM(m.patient_id)
          AND c.code_system = 'SNOMED-CT'
          AND TRIM(c.code) IN (
              SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
              WHERE TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1077'
                AND vsc.measurement_year = 2025)
          AND c.start_date BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
    )
    -- [Spec2:DENOM_DIABETES:MAPPED] two-year window [AE-01]
)
,CTE_04_EXCLUSIONS AS (
    SELECT d.*
    FROM CTE_03_DENOMINATOR d

    WHERE NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.encounters exc_e
        WHERE TRIM(exc_e.patient_id) = TRIM(d.patient_id)
          AND LOWER(exc_e.encounter_class) = 'hospice'
          AND CAST(exc_e.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    )
    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.procedures exc_p
        WHERE TRIM(exc_p.patient_id) = TRIM(d.patient_id)
          AND exc_p.code_system = 'SNOMED-CT'
          AND TRIM(exc_p.code) IN (
              SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
              WHERE TRIM(vsc.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.1761','2.16.840.1.113883.3.464.1004.1762')
                AND vsc.measurement_year = 2025)
          AND CAST(exc_p.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    )
    -- [Spec2:EXCL_HOSPICE:MAPPED]

    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.patients exc_pt
        WHERE TRIM(exc_pt.patient_id) = TRIM(d.patient_id)
          AND exc_pt.death_date IS NOT NULL
          AND exc_pt.death_date BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    )
    -- [Spec2:EXCL_DECEASED:MAPPED]

    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.procedures exc_pal
        WHERE TRIM(exc_pal.patient_id) = TRIM(d.patient_id)
          AND exc_pal.code_system = 'SNOMED-CT'
          AND TRIM(exc_pal.code) IN (
              SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
              WHERE TRIM(vsc.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.1450','2.16.840.1.113883.3.464.1004.2225','2.16.840.1.113883.3.464.1004.2224')
                AND vsc.measurement_year = 2025)
          AND CAST(exc_pal.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    )
    -- [Spec2:EXCL_PALLIATIVE:MAPPED]

    AND NOT (
        d.age_at_anchor >= 66                                    -- [AE-04]
        AND (
            SELECT COUNT(DISTINCT exc_f.start_date)
            FROM DEMO_HealthcareSynthea.conditions exc_f
            WHERE TRIM(exc_f.patient_id) = TRIM(d.patient_id)
              AND exc_f.code_system = 'SNOMED-CT'
              AND TRIM(exc_f.code) IN (
                  SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
                  WHERE TRIM(vsc.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.1531','2.16.840.1.113883.3.464.1004.1533')
                    AND vsc.measurement_year = 2025)
              AND exc_f.start_date BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
        ) >= 2
        -- [Spec2:EXCL_FRAILTY:DOMAIN_GAP:CODE_SYSTEM][AE-08]
        AND (
            (   -- EXCL_ADV_ILL_VISITS (DOMAIN_GAP, two-year window)
                EXISTS (
                    SELECT 1 FROM DEMO_HealthcareSynthea.conditions exc_ai
                    WHERE TRIM(exc_ai.patient_id) = TRIM(d.patient_id)
                      AND exc_ai.code_system = 'SNOMED-CT'
                      AND TRIM(exc_ai.code) IN (
                          SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
                          WHERE TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1465'
                            AND vsc.measurement_year = 2025)
                      AND exc_ai.start_date BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
                )
                AND (
                    SELECT COUNT(DISTINCT CAST(exc_v.start_ts AT TIME ZONE 'GMT' AS DATE))
                    FROM DEMO_HealthcareSynthea.encounters exc_v
                    WHERE TRIM(exc_v.patient_id) = TRIM(d.patient_id)
                      AND TRIM(exc_v.code) IN (
                          SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
                          WHERE TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.2474'
                            AND vsc.measurement_year = 2025)
                      AND CAST(exc_v.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
                ) >= 2
            )
            OR
            (   -- EXCL_ADV_ILL_ACUTE_ENC (PROXY, two-year window)
                EXISTS (
                    SELECT 1 FROM DEMO_HealthcareSynthea.encounters exc_ae
                    WHERE TRIM(exc_ae.patient_id) = TRIM(d.patient_id)
                      AND LOWER(exc_ae.encounter_class) = 'inpatient'
                      AND CAST(exc_ae.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
                      AND EXISTS (
                          SELECT 1 FROM DEMO_HealthcareSynthea.conditions exc_ai2
                          WHERE TRIM(exc_ai2.patient_id) = TRIM(d.patient_id)
                            AND exc_ai2.code_system = 'SNOMED-CT'
                            AND TRIM(exc_ai2.code) IN (
                                SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
                                WHERE TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1465'
                                  AND vsc.measurement_year = 2025)
                            AND exc_ai2.start_date BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
                      )
                )
            )
            OR
            (   -- EXCL_ADV_ILL_ACUTE_DISCH (PROXY, two-year window, stop_ts as discharge substitute)
                EXISTS (
                    SELECT 1 FROM DEMO_HealthcareSynthea.encounters exc_ad
                    WHERE TRIM(exc_ad.patient_id) = TRIM(d.patient_id)
                      AND LOWER(exc_ad.encounter_class) = 'inpatient'
                      AND CAST(exc_ad.end_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
                      AND EXISTS (
                          SELECT 1 FROM DEMO_HealthcareSynthea.conditions exc_ai3
                          WHERE TRIM(exc_ai3.patient_id) = TRIM(d.patient_id)
                            AND exc_ai3.code_system = 'SNOMED-CT'
                            AND TRIM(exc_ai3.code) IN (
                                SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
                                WHERE TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1465'
                                  AND vsc.measurement_year = 2025)
                            AND exc_ai3.start_date BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
                      )
                )
            )
        )
    )
    -- [Spec2:EXCL_FRAILTY_ADVANCED_ILLNESS:PARTIAL]
)
,CTE_NUM_EVENTS_PROC AS (
    SELECT DISTINCT TRIM(p.patient_id) AS patient_id                     -- [SN-04]
    FROM DEMO_HealthcareSynthea.procedures p
    WHERE p.code_system = 'SNOMED-CT'
      AND TRIM(p.code) IN (
          SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
          WHERE TRIM(vsc.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.2550','2.16.840.1.113883.3.464.1004.2584')
            AND vsc.measurement_year = 2025)
      AND CAST(p.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    -- [Spec2:NUM_EYE_EXAM:MAPPED] Retinal Eye Exams / Retinal Imaging [AE-01]
)
,CTE_NUM_EVENTS_OBS AS (
    SELECT DISTINCT TRIM(o.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.observations o
    WHERE TRIM(o.code) IN (
          SELECT TRIM(vsc.code) FROM HCLS.measure_value_set_codes vsc
          WHERE TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.2616'
            AND vsc.measurement_year = 2025)
      AND CAST(o.observation_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    -- [Spec2:NUM_EYE_EXAM:MAPPED] Autonomous Eye Exam Result or Finding [SN-10]
)
,CTE_05_NUMERATOR AS (
    SELECT patient_id FROM CTE_NUM_EVENTS_PROC
    UNION
    SELECT patient_id FROM CTE_NUM_EVENTS_OBS
    -- [Spec2:NUM_COUNT_RULE:MAPPED] one qualifying event, any modality
)
,CTE_06_RESULT AS (
    SELECT
        TRIM(d.patient_id) AS member_id
        ,CASE WHEN n.patient_id IS NOT NULL THEN 1 ELSE 0 END AS in_numerator
        -- [Spec2:NUM_COUNT_RULE:MAPPED][AE-11] LEFT JOIN + IS NOT NULL — never CASE WHEN EXISTS()
    FROM CTE_04_EXCLUSIONS d
    LEFT JOIN CTE_05_NUMERATOR n
        ON TRIM(d.patient_id) = TRIM(n.patient_id)
)
SELECT DISTINCT member_id, in_numerator FROM CTE_06_RESULT;
```

## Spine structure (read-only context)

The spine is the certified SQL from the top through `CTE_05_NUMERATOR`. Columns each CTE exposes (never select a column that is not listed):

- `CTE_01_MEMBERS`(birth_date, city, county, death_date, ethnicity, patient_id, race, state, zip)
- `CTE_ENROLL_GOV_SEGMENTS`(patient_id, seg_end, seg_start)
- `CTE_ENROLL_ORDERED`(patient_id, prev_seg_end, seg_end, seg_start)
- `CTE_ENROLL_GAPCHECK`(coverage_end, coverage_start, has_gap, patient_id)
- `CTE_02_ENROLLED`(patient_id)
- `CTE_03_DENOMINATOR`(age_at_anchor, age_band_5, age_tertile, birth_date, city, county, ethnicity, patient_id, race, state, zip)
- `CTE_04_EXCLUSIONS`(age_at_anchor, age_band_5, age_tertile, birth_date, city, county, ethnicity, patient_id, race, state, zip)
- `CTE_NUM_EVENTS_PROC`(patient_id)
- `CTE_NUM_EVENTS_OBS`(patient_id)
- `CTE_05_NUMERATOR`(patient_id)

Member key: `patient_id`

**Rate model for this measure:** `CTE_05_NUMERATOR` is ONLY the DISTINCT set of NUMERATOR members (columns: patient_id). It has NO denominator and NO `in_numerator` column. For this measure: read FROM the eligible-denominator CTE `CTE_04_EXCLUSIONS` (aliased d); LEFT JOIN `CTE_05_NUMERATOR` num ON TRIM(d.patient_id) = TRIM(num.patient_id); derive the flag with CASE WHEN num.patient_id IS NOT NULL THEN 1 ELSE 0 END; denominator = COUNT(DISTINCT d.patient_id); numerator = SUM(that flag).

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
### C09 · Eye Exam to Check for Damage from Diabetes (EED)

---

#### Document Control

| Field            | Value                                              |
|------------------|----------------------------------------------------|
| spec_type        | MEASURE_CONCEPT                                    |
| spec_version     | 1.0                                                |
| measure_id       | C09                                                 |
| measure_label    | Eye Exam to Check for Damage from Diabetes          |
| hedis_label      | Eye Exam for Patients with Diabetes (EED)           |
| program          | Medicare Advantage Part C Star Ratings              |
| reference        | NCQA HEDIS Measurement Year 2023 Technical Specifications Volume 2, p.203 |
| cms_cmit         | 00203-02-C-PARTC                                    |
| authored_date    | 2026-08-03                                          |
| status           | ACTIVE                                               |

---

#### 1. Measure Summary

| Field                | Value                                              |
|----------------------|----------------------------------------------------|
| description          | Percent of plan members with diabetes who had an eye exam to check for damage from diabetes during the year |
| direction            | Higher is better                                    |
| weighting_category   | Process Measure                                     |
| weighting_value      | 1                                                    |
| statistical_method   | Clustering                                          |
| case_mix_adjusted    | No                                                   |
| display_format       | Percentage with no decimal place                    |
| meaningful_measure   | Chronic Conditions                                  |
| primary_data_source  | HEDIS                                                |
| data_source_category | Health and Drug Plans                               |

---

#### 2. Measurement Period

| Concept                  | Rule                                                        |
|---------------------------|-------------------------------------------------------------|
| measurement_year         | The calendar year being measured (e.g. 2023)                |
| measurement_period_start | January 1 of measurement_year                                |
| measurement_period_end   | December 31 of measurement_year                              |
| anchor_date              | December 31 of measurement_year (used for age calculation)   |
| numerator_lookback_start | January 1 of measurement_year                                 |
| numerator_lookback_end   | December 31 of measurement_year                               |
| numerator_window_note    | One calendar year — retinal eye exam must occur during the measurement year itself. Diabetes-diagnosis identification for the denominator (§5) uses a two-year window (measurement year or year prior). Do NOT apply the two-year diabetes-identification window to the numerator eye exam event. |

> **Agent note:** The extracted CMS Technical Notes describe a single-year numerator lookback (eye exam performed "during the measurement year"). The full NCQA HEDIS EED specification (p.203, referenced but not fully excerpted here) contains additional nuance around negative retinopathy findings identified in the prior year satisfying the numerator in certain circumstances. Until that language is confirmed from the full NCQA source, the agent must apply the single-year window stated in the CMS Technical Notes and flag any prior-year eye exam data as gap_flag = SPEC_CLARIFICATION_NEEDED:NUM_LOOKBACK.

---

#### 3. Rate Formula

```
Rate = (Numerator / Denominator) × 100

Reported as: Percentage with no decimal place
```

---

#### 4. Star Rating Cut Points

| Star Band | Rule                          |
|-----------|-------------------------------|
| 1 star    | < 57 %                        |
| 2 stars   | >= 57 % to < 70 %              |
| 3 stars   | >= 70 % to < 77 %              |
| 4 stars   | >= 77 % to < 83 %              |
| 5 stars   | >= 83 %                        |

---

#### 5. Denominator — Conceptual Definition

**Plain language:** Medicare Advantage enrollees aged 18–75 with diabetes (type 1 or type 2) diagnosed during the measurement year or the year prior, who were continuously enrolled during the measurement year.

##### 5.1 Denominator Inclusion Criteria

| Concept ID       | Concept Name          | Rule                                                                                       | Required |
|------------------|-----------------------|----------------------------------------------------------------------------------------------|----------|
| DENOM_AGE        | Age at anchor date    | Age >= 18 AND age <= 75 as of December 31 of measurement_year                               | YES      |
| DENOM_PLAN_TYPE  | Plan type             | Medicare Advantage (MA) enrollee                                                            | YES      |
| DENOM_DIABETES   | Diabetes diagnosis    | Diagnosis of diabetes (type 1 or type 2), in any setting, during measurement_year OR the year prior to measurement_year | YES      |
| DENOM_ENROLL     | Continuous enrollment | Continuously enrolled during measurement_year (per HEDIS continuous enrollment allowance)   | YES      |
| DENOM_ALIVE      | Vital status          | Member must not have died at any time during measurement_year (see EXCL_DECEASED, §7)        | YES      |

> **Agent note:** DENOM_DIABETES is identified using the "Diabetes" value set (ICD10CM, SNOMED CT US Edition). Standard HEDIS diabetes-population identification also permits pharmacy-based identification (dispensed insulin or oral hypoglycemic/antihyperglycemic medications); no corresponding NDC-based value set is present in the available MY2025 HEDIS VSD list. If Spec 2 cannot supply pharmacy-based diabetes identification, flag gap_flag = DOMAIN_GAP:DENOM_DIABETES_PHARMACY.

##### 5.2 Denominator Minimum Enrollment Threshold

| Concept ID        | Rule                                                                            |
|-------------------|-----------------------------------------------------------------------------------|
| DENOM_MIN_500     | Contracts with enrollment < 500 as of the July 2023 enrollment report are excluded from this measure entirely |
| DENOM_RELIABILITY | Contracts with enrollment >= 500 and < 1,000 AND measure score reliability < 0.7 are excluded |

---

#### 6. Numerator — Conceptual Definition

**Plain language:** At least one retinal eye exam performed to check for diabetic damage (retinopathy) during the measurement year.

##### 6.1 Numerator Inclusion Events

| Concept ID      | Concept Name         | Rule                                                              |
|-----------------|-----------------------|---------------------------------------------------------------------|
| NUM_EYE_EXAM    | Retinal eye exam event | At least one qualifying retinal eye exam during the numerator lookback window |
| NUM_DATE_WINDOW | Date of service      | Service date >= numerator_lookback_start AND <= numerator_lookback_end |
| NUM_COUNT_RULE  | Minimum event count  | One qualifying event satisfies the numerator. Additional events do not increase the numerator count beyond 1 per member |

##### 6.2 Qualifying Event Code Sets

###### Retinal Eye Exam

> Lookback window: 1 year (measurement year only)

**VALUE_SET_REFERENCES**
| Value Set Name                                  | OID                                              | Code Systems                          |
|--------------------------------------------------|---------------------------------------------------|----------------------------------------|
| Retinal Eye Exams                                | 2.16.840.1.113883.3.464.1004.2550                 | CPT, HCPCS, SNOMED CT US Edition       |
| Retinal Imaging                                  | 2.16.840.1.113883.3.464.1004.2584                 | CPT, SNOMED CT US Edition              |
| Autonomous Eye Exam Result or Finding            | 2.16.840.1.113883.3.464.1004.2616                 | LOINC                                   |

> **Agent note:** "Autonomous Eye Exam Result or Finding" captures results from AI/autonomous retinal imaging devices; treat any recorded result (positive or negative for retinopathy) as a qualifying numerator event, per current NCQA guidance on autonomous exam technology. "Diabetic Retinopathy Severity Level" (LOINC, OID 2.16.840.1.113883.3.464.1004.2555) may be used by Spec 2 as a supporting result value set if the customer data model separates exam procedure from exam finding, but does not independently satisfy the numerator without a corresponding exam event.

---

#### 7. Exclusions — Conceptual Definitions

> Exclusions are applied after the denominator cohort is built and before the numerator is evaluated.
> Order of application: EXCL_DIABETES_EXCEPTION → EXCL_HOSPICE → EXCL_DECEASED → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY_ADVANCED_ILLNESS

##### 7.1 Hospice Exclusion

| Concept ID   | Rule                                                                          |
|--------------|-------------------------------------------------------------------------------|
| EXCL_HOSPICE | Exclude members in hospice or using hospice services any time during measurement_year |
| source_types | Encounter · Claim · Enrollment flag                                           |

**VALUE_SET_REFERENCES**
| Value Set Name        | OID                                             | Code Systems                     |
|------------------------|--------------------------------------------------|------------------------------------|
| Hospice Encounter     | 2.16.840.1.113883.3.464.1004.1761                | HCPCS, SNOMED CT US Edition, UBREV |
| Hospice Intervention  | 2.16.840.1.113883.3.464.1004.1762                | CPT, HCPCS, SNOMED CT US Edition   |

##### 7.2 Deceased Member Exclusion

| Concept ID     | Rule                                                             |
|----------------|-------------------------------------------------------------------|
| EXCL_DECEASED  | Exclude members who died at any time during measurement_year      |
| source_types   | Enrollment/eligibility file — date of death field                 |

##### 7.3 Palliative Care Exclusion

| Concept ID      | Rule                                                                      |
|-----------------|-----------------------------------------------------------------------------|
| EXCL_PALLIATIVE | Exclude members receiving palliative care any time during measurement_year |
| source_types    | Encounter/claim with palliative care code · Palliative flag                |

**VALUE_SET_REFERENCES**
| Value Set Name              | OID                                             | Code Systems       |
|-------------------------------|---------------------------------------------------|----------------------|
| Palliative Care Encounter    | 2.16.840.1.113883.3.464.1004.1450                | HCPCS, SNOMED CT US Edition |
| Palliative Care Assessment   | 2.16.840.1.113883.3.464.1004.2225                | SNOMED CT US Edition |
| Palliative Care Intervention | 2.16.840.1.113883.3.464.1004.2224                | SNOMED CT US Edition |

##### 7.4 Institutional SNP (I-SNP) Exclusion

| Concept ID | Rule                                                                              |
|------------|-------------------------------------------------------------------------------------|
| EXCL_ISNP  | Exclude members age >= 66 enrolled in I-SNP at any time during measurement_year     |
| age_gate   | >= 66 as of December 31 of measurement_year                                          |
| source     | Plan/SNP type indicator in enrollment/eligibility data — no clinical code set applies |

##### 7.5 Long-Term Institutional (LTI) Exclusion

| Concept ID | Rule                                                                              |
|------------|-----------------------------------------------------------------------------------|
| EXCL_LTI   | Exclude members age >= 66 with LTI flag in CMS Monthly Membership Detail Data File any time during measurement_year |
| age_gate   | >= 66 as of December 31 of measurement_year                                        |
| source     | External CMS file — not derivable from claims                                       |

##### 7.6 Frailty + Advanced Illness Exclusion

> Members age >= 66 as of December 31 of measurement_year must meet BOTH the frailty criterion AND the advanced illness criterion to be excluded.

###### 7.6a Frailty Criterion

| Concept ID     | Rule                                                                          |
|----------------|----------------------------------------------------------------------------------|
| EXCL_FRAILTY   | At least two frailty indicators with different dates of service during measurement_year |

**VALUE_SET_REFERENCES**
| Value Set Name    | OID                                             | Code Systems                     |
|---------------------|---------------------------------------------------|------------------------------------|
| Frailty Device     | 2.16.840.1.113883.3.464.1004.1530                 | HCPCS                              |
| Frailty Diagnosis  | 2.16.840.1.113883.3.464.1004.1531                 | ICD10CM, SNOMED CT US Edition       |
| Frailty Encounter  | 2.16.840.1.113883.3.464.1004.1532                 | CPT, HCPCS                          |
| Frailty Symptom    | 2.16.840.1.113883.3.464.1004.1533                 | ICD10CM, SNOMED CT US Edition       |

###### 7.6b Advanced Illness Criterion

> Any ONE of the following four patterns during measurement_year OR the year prior to measurement_year (count services across both years where noted) satisfies this criterion:

| Concept ID              | Pattern                                                                                              |
|--------------------------|--------------------------------------------------------------------------------------------------------|
| EXCL_ADV_ILL_VISITS      | At least two qualifying visits (outpatient, observation, ED, telephone, e-visit/virtual check-in, nonacute inpatient encounter, nonacute inpatient discharge) on different dates of service, with an advanced illness diagnosis, counted across measurement_year and the year prior |
| EXCL_ADV_ILL_ACUTE_ENC   | At least one acute inpatient encounter with an advanced illness diagnosis                             |
| EXCL_ADV_ILL_ACUTE_DISCH | At least one acute inpatient discharge with an advanced illness diagnosis on the discharge claim       |
| EXCL_ADV_ILL_RX          | A dispensed dementia medication during measurement_year or the year prior                              |

**VALUE_SET_REFERENCES**
| Value Set Name                                                            | OID                                             | Code Systems                             | Applies To            |
|-----------------------------------------------------------------------------|----------------------------------------------------|---------------------------------------------|-------------------------|
| Advanced Illness                                                          | 2.16.840.1.113883.3.464.1004.1465                 | ICD10CM, SNOMED CT US Edition               | Diagnosis basis for all patterns |
| Outpatient, ED, Telephone, Acute Inpatient and Nonacute Inpatient         | 2.16.840.1.113883.3.464.1004.2474                 | CPT, HCPCS, SNOMED CT US Edition, UBREV     | EXCL_ADV_ILL_VISITS      |
| Acute Inpatient                                                           | 2.16.840.1.113883.3.464.1004.1810                 | CPT, SNOMED CT US Edition                   | EXCL_ADV_ILL_ACUTE_ENC / EXCL_ADV_ILL_ACUTE_DISCH |
| Acute Inpatient POS                                                       | 2.16.840.1.113883.3.464.1004.1027                 | POS                                          | EXCL_ADV_ILL_ACUTE_ENC / EXCL_ADV_ILL_ACUTE_DISCH |
| Dementia Medication (Dispensed)                                          | TBD                                                | NDC                                           | EXCL_ADV_ILL_RX          |

> **Agent note:** No pharmacy/NDC-based value set for "dispensed dementia medication" is present in the available MY2025 HEDIS VSD list. If Spec 2 cannot supply this concept, omit EXCL_ADV_ILL_RX from evaluation and set gap_flag = DOMAIN_GAP:EXCL_ADV_ILL_RX. This does not invalidate the other three advanced-illness patterns.

##### 7.7 Diabetes Diagnosis Exception (Required)

| Concept ID              | Rule                                                                                                                          |
|--------------------------|----------------------------------------------------------------------------------------------------------------------------------|
| EXCL_DIABETES_EXCEPTION | Exclude members who did NOT have a diagnosis of diabetes, in any setting, during measurement_year or the year prior AND who HAD a diagnosis of polycystic ovarian syndrome, gestational diabetes, or steroid-induced diabetes, in any setting, during measurement_year or the year prior |
| source_types             | Diagnosis / Condition (any claim setting)                                                                                       |
| history_window           | Measurement year and year prior (two calendar years)                                                                            |

###### Diabetes Diagnosis Exception Code Sets

**VALUE_SET_REFERENCES**
| Value Set Name                        | OID                                             | Code Systems                  |
|-----------------------------------------|----------------------------------------------------|----------------------------------|
| Diabetes                               | 2.16.840.1.113883.3.464.1004.1077                 | ICD10CM, SNOMED CT US Edition    |
| Polycystic Ovarian Syndrome            | TBD                                                | —                                 |
| Gestational Diabetes                   | TBD                                                | —                                 |
| Steroid-Induced Diabetes               | TBD                                                | —                                 |

> **Agent note:** No dedicated value set for polycystic ovarian syndrome, gestational diabetes, or steroid-induced diabetes is present in the available MY2025 HEDIS VSD list. Flag as gap_flag = DOMAIN_GAP:EXCL_DIABETES_EXCEPTION_CODES until Spec 2 supplies a mapped code set from the full NCQA specification.

---

#### 8. Analytical Query Concepts

| Concept ID            | Concept Name         | Analytical Use                                                        |
|------------------------|-----------------------|--------------------------------------------------------------------------|
| AQ_AGE_TERTILE        | Age tertile           | Divide 18–75 range into thirds: 18–37, 38–56, 57–75                    |
| AQ_AGE_BAND_5         | 5-year age band       | Standard 5-year bands across 18–75 (e.g. 18–22, 23–27, ... 71–75)       |
| AQ_GEOGRAPHY          | Geographic segment    | Zip, county, state from member address                                  |
| AQ_RACE_ETHNICITY     | Race and ethnicity    | Health equity sub-analysis                                              |
| AQ_PROVIDER           | Attributed provider   | PCP or eye care specialist for outreach targeting                       |
| AQ_TREND              | Time trend            | Rolling monthly or quarterly rate within certified denom                |
| AQ_GAP_TO_CUT         | Gap to cut point      | Members needed in numerator to reach next star band                     |
| AQ_OUTREACH           | Outreach target list  | Denominator members NOT in numerator, by AQ dimension                   |
| AQ_EXAM_MODALITY      | Exam modality         | Segment numerator events by exam type (in-person retinal exam, retinal imaging, autonomous/AI exam) |

> **Guardrail:** All analytical queries must operate within the certified
> denominator. The agent must NOT redefine eligibility criteria.
> All analytical output must be labeled result_type = ANALYTICAL.

---

#### 9. Output Schema — Certified Result Record

| Field            | Type      | Description                                         |
|------------------|-----------|-----------------------------------------------------|
| measure_id       | VARCHAR   | C09                                                  |
| measurement_year | INTEGER   | e.g. 2023                                            |
| customer_id      | VARCHAR   | Payer/customer identifier                           |
| run_id           | VARCHAR   | Unique execution identifier                          |
| run_ts           | TIMESTAMP | When the calculation was executed                   |
| denominator_count | INTEGER  | Count of eligible members after exclusions          |
| numerator_count  | INTEGER   | Count of denominator members with qualifying event  |
| rate             | DECIMAL   | numerator / denominator × 100                       |
| rate_display     | INTEGER   | Rounded integer for CMS display                     |
| star_band        | INTEGER   | 1–5 based on cut points                             |
| result_type      | VARCHAR   | CERTIFIED or ANALYTICAL                             |
| below_threshold  | BOOLEAN   | TRUE if contract enrollment < 500                    |
| spec1_version    | VARCHAR   | Version of this spec used                           |
| spec2_version    | VARCHAR   | Version of customer data model spec used            |
| spec3_version    | VARCHAR   | Version of compute spec used                        |
| gap_flags        | VARCHAR   | Pipe-delimited list of gap flags from Spec 2         |

---

#### 10. Agent Instructions

##### 10.1 Certified SQL Agent Rules

1. Always resolve code sets from section 6.2 and 7.x using the value set / code system declared in Spec 2. If only SNOMED-CT is available where CPT/HCPCS is preferred, use SNOMED codes and set gap_flag = DOMAIN_GAP:CODE_SYSTEM.
2. Apply exclusions in the order specified in section 7 header: EXCL_DIABETES_EXCEPTION → EXCL_HOSPICE → EXCL_DECEASED → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY_ADVANCED_ILLNESS.
3. Exclusions EXCL_ISNP, EXCL_LTI, and EXCL_FRAILTY_ADVANCED_ILLNESS are age-gated (>= 66 as of December 31 of measurement_year). Do not apply to members aged 18–65.
4. The advanced illness criterion (§7.6b) has FOUR independent qualifying patterns; a member meets the criterion if ANY ONE pattern is satisfied. Do not require all four.
5. The advanced illness visit-count pattern (EXCL_ADV_ILL_VISITS) spans TWO calendar years (measurement_year and year prior) — do not restrict to measurement_year alone. The frailty criterion (§7.6a) spans ONLY measurement_year — do not extend it to the prior year.
6. The diabetes diagnosis exception (EXCL_DIABETES_EXCEPTION, §7.7) also spans a two-year window (measurement_year and year prior) and is independent of the age-gated exclusions — apply to all denominator members regardless of age.
7. If Spec 2 marks a concept as MISSING, omit that exclusion and record it in gap_flags. In particular, EXCL_ADV_ILL_RX (dementia medication) and the PCOS/gestational/steroid-induced diabetes code sets are commonly MISSING from claims-only data models — do not fail the pipeline; flag and continue.
8. If Spec 2 marks a concept as PROXY, apply proxy logic and set gap_flag = PROXY:<CONCEPT_ID>.
9. If Spec 2 marks a concept as SYNTHETIC_DATA_ARTIFACT, omit entirely and set gap_flag = SYNTHETIC_DATA_ARTIFACT:<CONCEPT_ID>.
10. The numerator lookback window is exactly one calendar year (measurement_year only). Do not apply a two-year window to the eye exam event, even though the denominator diabetes-identification window is two years.
11. A member satisfies the numerator with one qualifying event across any of the qualifying event value sets in §6.2 (in-person retinal exam, retinal imaging, or autonomous/AI exam result). Do not require a specific modality.
12. Never produce a rate from a denominator of zero. Return denominator_count = 0 and rate = NULL.

##### 10.2 Analytical Query Agent Rules

1. Parse the user's natural language query to identify the AQ dimension(s) from section 8.
2. Apply the full certified denominator before segmenting. Never segment before filtering.
3. For AQ_AGE_TERTILE: divide the 18–75 range into three equal bands — Lower: 18–37, Mid: 38–56, Upper: 57–75.
4. For AQ_GAP_TO_CUT: compute members_needed = CEIL((cut_point / 100) × denominator_count) − numerator_count for each band above current rate.
5. For AQ_OUTREACH: return denominator members with no qualifying event, sorted by AQ dimension then age DESC.
6. For AQ_EXAM_MODALITY: segment numerator members by which qualifying value set (Retinal Eye Exams / Retinal Imaging / Autonomous Eye Exam Result) satisfied the event; a member satisfying multiple modalities counts once in the numerator but may appear in multiple modality segments for descriptive purposes only.
7. Always label output result_type = ANALYTICAL.
8. Include the certified rate alongside every sub-analysis.
9. Refuse any query that redefines denominator eligibility (e.g. age range outside 18–75, or excluding the diabetes-diagnosis requirement). Explain why and offer the nearest valid alternative within the certified denominator.

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
