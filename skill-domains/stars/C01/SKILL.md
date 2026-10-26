---
name: stars-c01
description: >-
  STARS measure C01 (Breast Cancer Screening), measurement year 2025. Trigger when the user asks about
  C01 or "Breast Cancer Screening", e.g. "what is our C01 rate", "C01 star rating", "show C01 care gaps",
  "who is missing C01", "break C01 down by age/provider/region", or to build a C01 dashboard.
  Uses the approved, EXPLAIN-validated certified Teradata SQL as the single source of truth for the
  denominator, numerator, rate and star band, and writes analytical queries on top of it.
---

# C01 — Breast Cancer Screening (2025)

Percent of female Medicare Advantage enrollees who had a mammogram to screen for breast cancer in the past two years

- **Program:** STARS
- **Measurement year:** 2025
- **Customer data:** database `DEMO_HealthcareSynthea` (customer id `DEMO_HealthcareSynthea`); reference database `HCLS`
- **Source tables:** `DEMO_HealthcareSynthea.conditions`, `DEMO_HealthcareSynthea.observations`, `DEMO_HealthcareSynthea.patients`, `DEMO_HealthcareSynthea.payer_transitions`, `DEMO_HealthcareSynthea.payers`, `DEMO_HealthcareSynthea.procedures`, `HCLS.measure_value_set_codes`
- **Exported:** 2026-10-26 20:27 UTC from the Star Ratings measure automation app. The SQL below was generated, validated with Teradata `EXPLAIN`, and approved by a human reviewer.

## How to use this skill

Run all SQL with `base_readQuery` (Teradata dialect: use `TOP n`, not `LIMIT`).

1. Treat the **Certified SQL** below as the single source of truth for who is in the denominator and numerator. Never redefine eligibility, exclusions, codes or enrollment.
2. To answer a question (breakdown, filter, trend, gap-to-star, outreach list), reuse the certified CTE chain verbatim up to and including `CTE_05_NUMERATOR` (the "spine") and write only an **analytical tail** that reads from it. Follow the *Analytical query rules* below.
3. Quote the certified rate, star band and cut points from this file; never recompute or invent them. Any analytical result must reconcile to the certified denominator and numerator.
4. For dashboards, build every tile on top of the same spine so all numbers stay consistent with the certified result.

## Certified result

| Item | Value |
|---|---|
| Certified rate | 5.37% |
| Star band | not assigned |
| Denominator | 10000 |
| Numerator | 537 |
| Run time | 2026-07-20 20:54:57 |

### Star cut points

- 1 star: < 53 %
- 2 stars: >= 53 % to < 67 %
- 3 stars: >= 67 % to < 75 %
- 4 stars: >= 75 % to < 82 %
- 5 stars: >= 82 %

## Certified SQL

Run on Teradata as-is to reproduce the certified population. The final SELECT returns one row per denominator member (`member_id`, `in_numerator`).

```sql
WITH CTE_01_MEMBERS AS (
    SELECT DISTINCT
         TRIM(p.patient_id)   AS patient_id                                      -- [SN-04]
        ,p.birth_date
        ,p.gender
        ,p.death_date
    FROM DEMO_HealthcareSynthea.patients p
    WHERE p.gender = 'F'                                                                       -- [Spec2:DENOM_GENDER:MAPPED]
      AND (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM p.birth_date)) BETWEEN 50 AND 74  -- [Spec2:DENOM_AGE:MAPPED]
      AND (p.death_date IS NULL OR p.death_date > DATE '2025-12-31')                            -- [Spec2:DENOM_ALIVE:MAPPED]
)

,CTE_02A_SEGMENTS AS (
    SELECT DISTINCT
         TRIM(pt.patient_id)                                   AS patient_id
        ,CAST(pt.start_year AT TIME ZONE 'GMT' AS DATE)         AS seg_start   -- [SN-03]
        ,CAST(pt.end_year   AT TIME ZONE 'GMT' AS DATE)         AS seg_end     -- [SN-03]
    FROM DEMO_HealthcareSynthea.payer_transitions pt
    INNER JOIN DEMO_HealthcareSynthea.payers py
        ON TRIM(pt.primary_payer_id) = TRIM(py.payer_id)                       -- [SN-04] [SC-COL-FIX]
    WHERE py.payer_type <> 'NO_INSURANCE'                                     -- [Spec2:DENOM_PLAN_TYPE:PROXY]
      AND CAST(pt.start_year AT TIME ZONE 'GMT' AS DATE) <= DATE '2025-12-31'
      AND CAST(pt.end_year   AT TIME ZONE 'GMT' AS DATE) >= DATE '2025-01-01'
)

,CTE_02B_ORDERED AS (
    SELECT patient_id, seg_start, seg_end,
           LAG(seg_end) OVER (PARTITION BY patient_id ORDER BY seg_start) AS prev_end
    FROM CTE_02A_SEGMENTS
)

,CTE_02C_GRP AS (
    SELECT patient_id, seg_start, seg_end,
           SUM(CASE WHEN prev_end IS NULL OR seg_start > prev_end + INTERVAL '1' DAY
                    THEN 1 ELSE 0 END)
               OVER (PARTITION BY patient_id ORDER BY seg_start ROWS UNBOUNDED PRECEDING) AS island_id
    FROM CTE_02B_ORDERED
)

,CTE_02D_MERGED AS (
    SELECT patient_id, island_id, MIN(seg_start) AS island_start, MAX(seg_end) AS island_end
    FROM CTE_02C_GRP
    GROUP BY patient_id, island_id
)

,CTE_02_ENROLLED AS (
    SELECT DISTINCT TRIM(patient_id) AS patient_id                               -- [Spec2:DENOM_ENROLL:MAPPED] [SN-07]
    FROM CTE_02D_MERGED
    WHERE island_start <= DATE '2025-01-01'
      AND island_end   >= DATE '2025-12-31'
)

,CTE_03_DENOMINATOR AS (
    SELECT
         m.patient_id
        ,m.birth_date
        ,(EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) AS age_at_anchor
    FROM CTE_01_MEMBERS m
    INNER JOIN CTE_02_ENROLLED e
        ON TRIM(m.patient_id) = TRIM(e.patient_id)                               -- [SN-04]
)

,CTE_04_EXCLUSIONS AS (
    SELECT d.*
    FROM CTE_03_DENOMINATOR d
    WHERE NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.procedures pr
        INNER JOIN HCLS.measure_value_set_codes vsc
            ON TRIM(pr.code) = TRIM(vsc.code) AND vsc.code_system = 'SNOMED CT US Edition'
           AND TRIM(vsc.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.1761','2.16.840.1.113883.3.464.1004.1762')
        WHERE TRIM(pr.patient_id) = TRIM(d.patient_id)
          AND CAST(pr.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    )                                                                              -- [Spec2:EXCL_HOSPICE:DOMAIN_GAP]
    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.conditions co
        INNER JOIN HCLS.measure_value_set_codes vsc
            ON TRIM(co.code) = TRIM(vsc.code) AND vsc.code_system = 'SNOMED CT US Edition'
           AND TRIM(vsc.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.1761','2.16.840.1.113883.3.464.1004.1762')
        WHERE TRIM(co.patient_id) = TRIM(d.patient_id)
          AND co.start_date <= DATE '2025-12-31'
          AND (co.end_date IS NULL OR co.end_date >= DATE '2025-01-01')
    )                                                                              -- [Spec2:EXCL_HOSPICE:DOMAIN_GAP]
    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.conditions co2
        INNER JOIN HCLS.measure_value_set_codes vsc2
            ON TRIM(co2.code) = TRIM(vsc2.code) AND vsc2.code_system = 'SNOMED CT US Edition'
           AND TRIM(vsc2.value_set_oid) IN
               ('2.16.840.1.113883.3.464.1004.1450','2.16.840.1.113883.3.464.1004.2225','2.16.840.1.113883.3.464.1004.2224')
        WHERE TRIM(co2.patient_id) = TRIM(d.patient_id)
          AND co2.start_date <= DATE '2025-12-31'
          AND (co2.end_date IS NULL OR co2.end_date >= DATE '2025-01-01')
    )                                                                              -- [Spec2:EXCL_PALLIATIVE:DOMAIN_GAP]
    -- EXCL_ISNP omitted — [Spec2:EXCL_ISNP:MISSING]
    -- EXCL_LTI  omitted — [Spec2:EXCL_LTI:MISSING]
    AND NOT (
        d.age_at_anchor >= 66                                                     -- [Spec1:§7.3/7.4/7.5 age gate]
        AND (SELECT COUNT(DISTINCT co3.start_date) FROM DEMO_HealthcareSynthea.conditions co3
             INNER JOIN HCLS.measure_value_set_codes vsc3
                 ON TRIM(co3.code) = TRIM(vsc3.code) AND vsc3.code_system = 'SNOMED CT US Edition'
                AND TRIM(vsc3.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.1531','2.16.840.1.113883.3.464.1004.1533')
             WHERE TRIM(co3.patient_id) = TRIM(d.patient_id)
               AND co3.start_date BETWEEN DATE '2025-01-01' AND DATE '2025-12-31') >= 2
        AND (SELECT COUNT(DISTINCT co4.start_date) FROM DEMO_HealthcareSynthea.conditions co4
             INNER JOIN HCLS.measure_value_set_codes vsc4
                 ON TRIM(co4.code) = TRIM(vsc4.code) AND vsc4.code_system = 'SNOMED CT US Edition'
                AND TRIM(vsc4.value_set_oid) = '2.16.840.1.113883.3.464.1004.1465'
             WHERE TRIM(co4.patient_id) = TRIM(d.patient_id)
               AND co4.start_date BETWEEN DATE '2025-01-01' AND DATE '2025-12-31') >= 2
    )                                                                              -- [Spec2:EXCL_FRAILTY:DOMAIN_GAP]
    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.conditions co5
        INNER JOIN HCLS.measure_value_set_codes vsc5
            ON TRIM(co5.code) = TRIM(vsc5.code) AND vsc5.code_system = 'SNOMED CT US Edition'
           AND TRIM(vsc5.value_set_oid) IN ('2.16.840.1.113883.3.464.1004.1042','2.16.840.1.113883.3.464.1004.1331')
        WHERE TRIM(co5.patient_id) = TRIM(d.patient_id) AND co5.start_date <= DATE '2025-12-31'
    )                                                                              -- [Spec2:EXCL_MASTECTOMY:DOMAIN_GAP]
    AND NOT (
        EXISTS (SELECT 1 FROM DEMO_HealthcareSynthea.conditions coL
                INNER JOIN HCLS.measure_value_set_codes vscL
                    ON TRIM(coL.code) = TRIM(vscL.code) AND vscL.code_system = 'SNOMED CT US Edition'
                   AND TRIM(vscL.value_set_oid) = '2.16.840.1.113883.3.464.1004.1334'
                WHERE TRIM(coL.patient_id) = TRIM(d.patient_id) AND coL.start_date <= DATE '2025-12-31')
        AND EXISTS (SELECT 1 FROM DEMO_HealthcareSynthea.conditions coR
                INNER JOIN HCLS.measure_value_set_codes vscR
                    ON TRIM(coR.code) = TRIM(vscR.code) AND vscR.code_system = 'SNOMED CT US Edition'
                   AND TRIM(vscR.value_set_oid) = '2.16.840.1.113883.3.464.1004.1335'
                WHERE TRIM(coR.patient_id) = TRIM(d.patient_id) AND coR.start_date <= DATE '2025-12-31')
    )                                                                              -- [Spec2:EXCL_MASTECTOMY:DOMAIN_GAP] EXCL_MAST_L_R
    AND NOT (
        EXISTS (SELECT 1 FROM DEMO_HealthcareSynthea.conditions coAL
                INNER JOIN HCLS.measure_value_set_codes vscAL
                    ON TRIM(coAL.code) = TRIM(vscAL.code) AND vscAL.code_system = 'SNOMED CT US Edition'
                   AND TRIM(vscAL.value_set_oid) = '2.16.840.1.113883.3.464.1004.1329'
                WHERE TRIM(coAL.patient_id) = TRIM(d.patient_id))
        AND EXISTS (SELECT 1 FROM DEMO_HealthcareSynthea.conditions coAR
                INNER JOIN HCLS.measure_value_set_codes vscAR
                    ON TRIM(coAR.code) = TRIM(vscAR.code) AND vscAR.code_system = 'SNOMED CT US Edition'
                   AND TRIM(vscAR.value_set_oid) = '2.16.840.1.113883.3.464.1004.1330'
                WHERE TRIM(coAR.patient_id) = TRIM(d.patient_id))
    )                                                                              -- [Spec2:EXCL_MASTECTOMY:DOMAIN_GAP] EXCL_MAST_ABSENT
    -- EXCL_MAST_MOD_B / EXCL_MAST_2_BILAT / EXCL_MAST_RL omitted — [Spec2:EXCL_MAST_MOD_B/2_BILAT/RL:MISSING] (SN-05)
)

,CTE_MAMMO_EVENTS AS (
    SELECT DISTINCT TRIM(pr.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.procedures pr
    INNER JOIN HCLS.measure_value_set_codes vsc
        ON TRIM(pr.code) = TRIM(vsc.code) AND vsc.code_system = 'SNOMED CT US Edition'
       AND TRIM(vsc.value_set_oid) = '2.16.840.1.113883.3.464.1004.1168'          -- [Spec2:NUM_MAMMOGRAM:MAPPED]
    WHERE CAST(pr.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'  -- [Spec2:NUM_DATE_WINDOW:MAPPED]

    UNION

    SELECT DISTINCT TRIM(ob.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.observations ob
    INNER JOIN HCLS.measure_value_set_codes vsc2
        ON TRIM(ob.code) = TRIM(vsc2.code) AND vsc2.code_system = 'LOINC'
       AND TRIM(vsc2.value_set_oid) = '2.16.840.1.113883.3.464.1004.1168'         -- [Spec2:NUM_MAMMOGRAM:MAPPED]
    WHERE CAST(ob.observation_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'  -- [Spec2:NUM_DATE_WINDOW:MAPPED]
)

,CTE_05_NUMERATOR AS (
    SELECT
         d.patient_id
        ,CASE WHEN m.patient_id IS NOT NULL THEN 1 ELSE 0 END AS in_numerator     -- [Spec1:§10.1.8] LEFT JOIN + IS NOT NULL, no CASE WHEN EXISTS
    FROM CTE_04_EXCLUSIONS d
    LEFT JOIN CTE_MAMMO_EVENTS m
        ON TRIM(d.patient_id) = TRIM(m.patient_id)                               -- [SN-04]
)

,CTE_06_RESULT AS (
    SELECT DISTINCT
         TRIM(n.patient_id) AS member_id
        ,n.in_numerator      AS in_numerator
    FROM CTE_05_NUMERATOR n
)

SELECT
     member_id
    ,in_numerator
FROM CTE_06_RESULT;
```

## Spine structure (read-only context)

The spine is the certified SQL from the top through `CTE_05_NUMERATOR`. Columns each CTE exposes (never select a column that is not listed):

- `CTE_01_MEMBERS`(birth_date, death_date, gender, patient_id)
- `CTE_02A_SEGMENTS`(patient_id, seg_end, seg_start)
- `CTE_02B_ORDERED`(patient_id, prev_end, seg_end, seg_start)
- `CTE_02C_GRP`(island_id, patient_id, seg_end, seg_start)
- `CTE_02D_MERGED`(island_end, island_id, island_start, patient_id)
- `CTE_02_ENROLLED`(patient_id)
- `CTE_03_DENOMINATOR`(age_at_anchor, birth_date, patient_id)
- `CTE_04_EXCLUSIONS`(age_at_anchor, birth_date, patient_id)
- `CTE_MAMMO_EVENTS`(patient_id)
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
### C01 · Breast Cancer Screening (BCS)

---

#### Document Control

| Field            | Value                                              |
|------------------|-----------------------------------------------------|
| spec_type        | MEASURE_CONCEPT                                    |
| spec_version     | 1.0                                                |
| measure_id       | C01                                                 |
| measure_label    | Breast Cancer Screening                             |
| hedis_label      | BCS                                                 |
| program          | Medicare Advantage Part C Star Ratings              |
| reference        | NCQA HEDIS Measurement Year 2023 Technical Specifications Volume 2, p.606 |
| cms_cmit         | 00093-02-C-PARTC                                    |
| authored_date    | 2026-07-15                                          |
| status           | DRAFT                                               |

---

#### 1. Measure Summary

| Field                | Value                                              |
|----------------------|------------------------------------------------------|
| description          | Percent of female Medicare Advantage enrollees who had a mammogram to screen for breast cancer in the past two years |
| direction            | Higher is better                                    |
| weighting_category   | Process Measure                                     |
| weighting_value      | 1                                                    |
| statistical_method   | Clustering                                          |
| case_mix_adjusted    | No                                                   |
| display_format       | Percentage with no decimal place                    |
| meaningful_measure   | Wellness and Prevention                             |
| primary_data_source  | HEDIS                                                |
| data_source_category | Health and Drug Plans                               |

> **Agent note (data discrepancy):** The extracted `label_for_stars`/`description` field states "aged 52-74," while the extracted `metric` field (the authoritative rate definition) states "50 to 74 years of age." This spec uses the metric definition (age 50–74) as authoritative, consistent with published HEDIS BCS age criteria. Flag this discrepancy if source documents are re-extracted in future measurement years.

---

#### 2. Measurement Period

| Concept                  | Rule                                                        |
|--------------------------|-------------------------------------------------------------|
| measurement_year         | The calendar year being measured (e.g. 2023)                |
| measurement_period_start | January 1 of measurement_year                                |
| measurement_period_end   | December 31 of measurement_year                              |
| anchor_date              | December 31 of measurement_year (used for age calculation)  |
| numerator_lookback_start | January 1 of (measurement_year − 1)                          |
| numerator_lookback_end   | December 31 of measurement_year                              |
| numerator_window_note    | Two full calendar years: prior year and measurement year ("past two years" per metric definition) |

---

#### 3. Rate Formula

```
Rate = (Numerator / Denominator) × 100

Reported as: percentage with no decimal place
```

---

#### 4. Star Rating Cut Points

| Star Band | Rule                         |
|-----------|-------------------------------|
| 1 star    | < 53 %                        |
| 2 stars   | >= 53 % to < 67 %              |
| 3 stars   | >= 67 % to < 75 %              |
| 4 stars   | >= 75 % to < 82 %              |
| 5 stars   | >= 82 %                        |

> Source: CMS Star Ratings Technical Notes, 2025 (measurement year 2023 data). Cut points change annually — always pull from the correct year's Technical Notes.

---

#### 5. Denominator — Conceptual Definition

**Plain language:** Female Medicare Advantage enrollees aged 50 to 74 as of December 31 of the measurement year who were continuously enrolled and alive through the measurement period.

##### 5.1 Denominator Inclusion Criteria

| Concept ID      | Concept Name              | Rule                                                                 | Required |
|-----------------|----------------------------|------------------------------------------------------------------------|----------|
| DENOM_GENDER    | Member sex                 | Female                                                                  | YES      |
| DENOM_AGE       | Age at anchor date          | Age >= 50 AND age <= 74 as of December 31 of measurement_year          | YES      |
| DENOM_PLAN_TYPE | Plan type                   | Medicare Advantage (MA) enrollee — per reporting_requirements table (1876 Cost, CCP w/o SNP, CCP with SNP, CCP with Only I-SNP, MSA, PFFS). Excludes standalone PDP | YES |
| DENOM_ENROLL    | Continuous enrollment       | Enrolled for the full measurement year (January 1 – December 31)       | YES      |
| DENOM_ALIVE     | Vital status                | Member must not have died prior to December 31 of measurement_year     | YES      |

##### 5.2 Denominator Minimum Enrollment Threshold

| Concept ID        | Rule                                                                                 |
|--------------------|----------------------------------------------------------------------------------------|
| DENOM_MIN_500      | Contracts whose enrollment was less than 500 as of the July measurement_year enrollment report are excluded from this measure entirely |
| DENOM_RELIABILITY  | Contracts whose enrollment was >= 500 and < 1,000 as of the July report AND measure score reliability < 0.7 are excluded |

> **Agent note:** These thresholds apply at the contract (plan) level, not the member level. If the customer is a single-contract payer with < 500 members in the eligible cohort, flag result as BELOW_THRESHOLD and suppress the star rating.

---

#### 6. Numerator — Conceptual Definition

**Plain language:** At least one mammogram to screen for breast cancer during the past two years (January 1 of the year prior to the measurement year through December 31 of the measurement year).

##### 6.1 Numerator Inclusion Events

| Concept ID        | Concept Name              | Rule                                                                              |
|-------------------|----------------------------|-------------------------------------------------------------------------------------|
| NUM_MAMMOGRAM     | Mammogram event            | At least one qualifying mammogram service during the numerator lookback window      |
| NUM_DATE_WINDOW   | Date of service window     | Service date >= numerator_lookback_start AND <= numerator_lookback_end              |
| NUM_COUNT_RULE    | Minimum event count        | One qualifying event satisfies the numerator. Multiple events do not increase the numerator count beyond 1 per member |

##### 6.2 Qualifying Event Code Sets

###### Mammogram

> Lookback window: two calendar years (prior year + measurement year)

**VALUE_SET_REFERENCES**

| Value Set Name | OID                                          | Code Systems                        |
|-----------------|-----------------------------------------------|----------------------------------------|
| Mammography     | 2.16.840.1.113883.3.464.1004.1168             | CPT, LOINC, SNOMED CT US Edition       |

> **Agent note:** Both screening and diagnostic mammograms are captured within the "Mammography" value set and both satisfy the numerator event requirement per HEDIS BCS specifications. Do not restrict to screening-only codes.

---

#### 7. Exclusions — Conceptual Definitions

> Exclusions are applied after the denominator cohort is built and before the numerator is evaluated. Each exclusion removes members from the denominator entirely.
> Order of application: EXCL_HOSPICE → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY → EXCL_MASTECTOMY

##### 7.1 Hospice Exclusion

| Concept ID   | Rule                                                                          |
|--------------|---------------------------------------------------------------------------------|
| EXCL_HOSPICE | Exclude members in hospice or using hospice services any time during the measurement_period |
| source_types | Encounter · Claim · Enrollment flag                                            |

**VALUE_SET_REFERENCES**

| Value Set Name       | OID                                        | Code Systems                    |
|-----------------------|-----------------------------------------------|-------------------------------------|
| Hospice Encounter     | 2.16.840.1.113883.3.464.1004.1761             | HCPCS, SNOMED CT US Edition, UBREV  |
| Hospice Intervention  | 2.16.840.1.113883.3.464.1004.1762             | CPT, HCPCS, SNOMED CT US Edition    |

##### 7.2 Palliative Care Exclusion

| Concept ID      | Rule                                                                      |
|------------------|------------------------------------------------------------------------------|
| EXCL_PALLIATIVE | Exclude members receiving palliative care any time during the measurement_period |
| source_types    | Encounter/claim with palliative care code · Palliative flag                   |

**VALUE_SET_REFERENCES**

| Value Set Name              | OID                                        | Code Systems           |
|-------------------------------|-----------------------------------------------|----------------------------|
| Palliative Care Encounter     | 2.16.840.1.113883.3.464.1004.1450             | HCPCS, SNOMED CT US Edition |
| Palliative Care Assessment    | 2.16.840.1.113883.3.464.1004.2225             | SNOMED CT US Edition       |
| Palliative Care Intervention  | 2.16.840.1.113883.3.464.1004.2224             | SNOMED CT US Edition       |

##### 7.3 Institutional SNP (I-SNP) Exclusion

| Concept ID | Rule                                                                              |
|-------------|---------------------------------------------------------------------------------|
| EXCL_ISNP  | Exclude members age >= 66 as of December 31 of measurement_year who were enrolled in an Institutional SNP (I-SNP) at any time during the measurement_year |
| age_gate   | >= 66 as of anchor_date                                                          |
| source     | Plan type / SNP type indicator in enrollment or eligibility data — not a HEDIS clinical value set |

##### 7.4 Long-Term Institutional (LTI) Exclusion

| Concept ID | Rule                                                                              |
|-------------|-----------------------------------------------------------------------------------|
| EXCL_LTI   | Exclude members age >= 66 as of December 31 of measurement_year with an LTI flag in the CMS Monthly Membership Detail Data File at any time during the measurement_year. Use the run date of the file to determine LTI status. |
| age_gate   | >= 66 as of anchor_date                                                           |
| source     | External CMS Monthly Membership Detail Data File — not derivable from claims     |

##### 7.5 Frailty + Advanced Illness Exclusion

> Members age >= 66 as of December 31 of measurement_year must meet BOTH the frailty criterion AND the advanced illness criterion during the measurement_year to be excluded.

###### 7.5a Frailty Criterion

| Concept ID   | Rule                                                                                                    |
|---------------|-------------------------------------------------------------------------------------------------------------|
| EXCL_FRAILTY | At least two indications of frailty with different dates of service during the measurement_period          |

**VALUE_SET_REFERENCES**

| Value Set Name    | OID                                        | Code Systems              |
|---------------------|------------------------------------------------|--------------------------------|
| Frailty Device      | 2.16.840.1.113883.3.464.1004.1530             | HCPCS                          |
| Frailty Diagnosis   | 2.16.840.1.113883.3.464.1004.1531             | ICD10CM, SNOMED CT US Edition  |
| Frailty Encounter   | 2.16.840.1.113883.3.464.1004.1532             | CPT, HCPCS                     |
| Frailty Symptom     | 2.16.840.1.113883.3.464.1004.1533             | ICD10CM, SNOMED CT US Edition  |

###### 7.5b Advanced Illness Criterion

| Concept ID              | Rule                                                                                                        |
|---------------------------|-----------------------------------------------------------------------------------------------------------------|
| EXCL_ADVANCED_ILLNESS   | At least two qualifying visits with an advanced illness diagnosis, on different dates of service, during the measurement_year. Visit type need not be the same for both visits. |

**Qualifying Visit Types:** Outpatient visit · Observation visit · Emergency department visit · Telephone visit · e-visit/virtual check-in · Nonacute inpatient encounter · Nonacute inpatient discharge

**VALUE_SET_REFERENCES**

| Value Set Name    | OID                                        | Code Systems               |
|---------------------|------------------------------------------------|---------------------------------|
| Advanced Illness    | 2.16.840.1.113883.3.464.1004.1465             | ICD10CM, SNOMED CT US Edition   |

##### 7.6 Bilateral Mastectomy Exclusion

> Exclude members with evidence of bilateral mastectomy (or both right and left unilateral mastectomies) at any time in the member's history through December 31 of the measurement_year. Any one of the following patterns satisfies the exclusion:

| Concept ID        | Pattern                                                                    |
|---------------------|--------------------------------------------------------------------------------|
| EXCL_MAST_BILAT    | Bilateral mastectomy procedure code (any date in history)                       |
| EXCL_MAST_MOD_B    | Unilateral mastectomy with a bilateral modifier (same procedure)               |
| EXCL_MAST_2_BILAT  | Two unilateral mastectomy procedures each found with a bilateral modifier (same procedure) |
| EXCL_MAST_RL       | Unilateral mastectomy with right-side modifier + unilateral mastectomy with left-side modifier (same or different dates) |
| EXCL_MAST_ABSENT   | Absence of left breast + absence of right breast (same or different dates)     |
| EXCL_MAST_L_R      | Left unilateral mastectomy (any date) + right unilateral mastectomy (any date)  |
| EXCL_MAST_HIST     | History of bilateral mastectomy diagnosis code                                 |

**VALUE_SET_REFERENCES**

| Value Set Name                     | OID                                        | Code Systems                          |
|---------------------------------------|------------------------------------------------|--------------------------------------------|
| Bilateral Mastectomy                  | 2.16.840.1.113883.3.464.1004.1042             | ICD10PCS, SNOMED CT US Edition             |
| Unilateral Mastectomy                 | 2.16.840.1.113883.3.464.1004.1256             | CPT                                        |
| Unilateral Mastectomy Left            | 2.16.840.1.113883.3.464.1004.1334             | ICD10PCS, SNOMED CT US Edition             |
| Unilateral Mastectomy Right           | 2.16.840.1.113883.3.464.1004.1335             | ICD10PCS, SNOMED CT US Edition             |
| Clinical Unilateral Mastectomy        | 2.16.840.1.113883.3.464.1004.1948             | SNOMED CT US Edition                       |
| Absence of Left Breast                | 2.16.840.1.113883.3.464.1004.1329             | ICD10CM, SNOMED CT US Edition              |
| Absence of Right Breast               | 2.16.840.1.113883.3.464.1004.1330             | ICD10CM, SNOMED CT US Edition              |
| History of Bilateral Mastectomy       | 2.16.840.1.113883.3.464.1004.1331             | ICD10CM, SNOMED CT US Edition              |

> **Agent note:** CPT modifiers LT (left), RT (right), and 50 (bilateral) applied to Unilateral Mastectomy procedure codes are required to distinguish laterality patterns (EXCL_MAST_MOD_B, EXCL_MAST_2_BILAT, EXCL_MAST_RL). If Spec 2 does not map claim modifiers, flag as MISSING and record gap_flag = MISSING:CLAIM_MODIFIER.

---

#### 8. Analytical Query Concepts

| Concept ID        | Concept Name         | Analytical Use                                                        |
|--------------------|-------------------------|-----------------------------------------------------------------------|
| AQ_AGE_TERTILE    | Age tertile             | Divide 50–74 range into thirds: 50–57, 58–65, 66–74                    |
| AQ_AGE_BAND_5     | 5-year age band         | Standard 5-year bands: 50–54, 55–59, 60–64, 65–69, 70–74                |
| AQ_GEOGRAPHY      | Geographic segment      | Zip code, county, state — from member address                          |
| AQ_RACE_ETHNICITY | Race and ethnicity      | For health equity analysis                                             |
| AQ_PROVIDER       | Attributed provider     | PCP or attributed provider for outreach targeting                      |
| AQ_TREND          | Time trend              | Rolling monthly or quarterly rate within the certified denominator     |
| AQ_GAP_TO_CUT     | Gap to cut point        | Members needed in numerator to reach next star band                    |
| AQ_OUTREACH       | Outreach target list    | Denominator members NOT in numerator, segmented by AQ dimension        |
| AQ_TEST_TYPE      | Screening modality      | Not applicable for this measure — single modality (Mammography). Retained for schema parity with COL-type measures. |

> **Guardrail:** All analytical queries must operate on the certified denominator population. The analytical agent MUST NOT redefine eligibility criteria (age bounds, sex, enrollment rules, exclusions). All analytical results must be labeled result_type = ANALYTICAL, never CERTIFIED.

---

#### 9. Output Schema — Certified Result Record

| Field              | Type      | Description                                          |
|---------------------|-----------|--------------------------------------------------------|
| measure_id         | VARCHAR   | C01                                                    |
| measurement_year   | INTEGER   | e.g. 2023                                              |
| customer_id        | VARCHAR   | Payer/customer identifier                              |
| run_id             | VARCHAR   | Unique execution identifier                            |
| run_ts             | TIMESTAMP | When the calculation was executed                      |
| denominator_count  | INTEGER   | Count of eligible members after exclusions             |
| numerator_count    | INTEGER   | Count of denominator members with qualifying event     |
| rate               | DECIMAL   | numerator / denominator × 100                          |
| rate_display       | INTEGER   | Rounded integer for CMS display                        |
| star_band          | INTEGER   | 1–5 based on cut points                                |
| result_type        | VARCHAR   | CERTIFIED or ANALYTICAL                                |
| below_threshold    | BOOLEAN   | TRUE if contract enrollment < 500                      |
| spec1_version      | VARCHAR   | Version of this spec used                              |
| spec2_version      | VARCHAR   | Version of customer data model spec used               |
| spec3_version      | VARCHAR   | Version of compute spec used                            |
| gap_flags          | VARCHAR   | Pipe-delimited list of gap flags from Spec 2            |

---

#### 10. Agent Instructions

##### 10.1 Certified SQL Agent Rules

1. Always resolve code sets from section 6.2/7.x using the value set references and the code system declared in Spec 2. If only SNOMED-CT is available where CPT/LOINC/HCPCS is expected, use SNOMED codes from the referenced value set and set gap_flag = DOMAIN_GAP:CODE_SYSTEM.
2. Apply exclusions in the order specified in section 7 header: EXCL_HOSPICE → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY → EXCL_MASTECTOMY.
3. Exclusions EXCL_ISNP, EXCL_LTI, and EXCL_FRAILTY (including EXCL_ADVANCED_ILLNESS) are age-gated (>= 66 as of anchor_date). Do not apply to members aged 50–65.
4. If Spec 2 marks a concept as MISSING (e.g. LTI flag, claim modifiers for mastectomy laterality), omit that exclusion and record it in gap_flags.
5. If Spec 2 marks a concept as PROXY, apply the proxy logic and set gap_flag = PROXY:<CONCEPT_ID>.
6. If Spec 2 marks a concept as SYNTHETIC_DATA_ARTIFACT, omit entirely and set gap_flag = SYNTHETIC_DATA_ARTIFACT:<CONCEPT_ID>.
7. The numerator lookback window is exactly two calendar years (January 1 of measurement_year − 1 through December 31 of measurement_year). Do not use a rolling 24-month window from the member's index date.
8. A member satisfies the numerator with one qualifying mammogram event (screening or diagnostic). Do not count events; evaluate existence only.
9. For EXCL_MASTECTOMY, evaluate all seven patterns (EXCL_MAST_BILAT through EXCL_MAST_HIST) independently; satisfying any one pattern excludes the member. Do not require all patterns to be present.
10. Never produce a rate from a denominator of zero. Return denominator_count = 0 and rate = NULL.

##### 10.2 Analytical Query Agent Rules

1. Parse the user's natural language query to identify the intended AQ dimension(s) from section 8.
2. Apply the full certified denominator logic before segmenting. Never segment before filtering.
3. For AQ_AGE_TERTILE: divide the 50–74 range as follows — Lower: 50–57, Mid: 58–65, Upper: 66–74.
4. For AQ_GAP_TO_CUT: compute members_needed = CEIL((cut_point_rate / 100) × denominator_count) − numerator_count for each star band boundary above the current rate.
5. For AQ_OUTREACH: return denominator members where no qualifying numerator event exists, sorted by the requested AQ dimension then age DESC.
6. Always label output with result_type = ANALYTICAL.
7. Include the certified rate alongside any sub-analysis for comparison context.
8. If the user's query would require redefining denominator eligibility (e.g. "show me women aged 45–50" or "include male members"), refuse the redefinition and explain that the certified denominator is female members aged 50–74. Offer to run the analysis on the nearest valid tertile or band instead.

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
