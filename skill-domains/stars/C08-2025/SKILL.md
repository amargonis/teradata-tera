---
name: stars-c08-2025
description: >-
  STARS measure C08 (Osteoporosis Management), measurement year 2025. Trigger when the user asks about
  C08 or "Osteoporosis Management", e.g. "what is our C08 rate", "C08 star rating", "show C08 care gaps",
  "who is missing C08", "break C08 down by age/provider/region", or to build a C08 dashboard.
  Uses the approved, EXPLAIN-validated certified Teradata SQL as the single source of truth for the
  denominator, numerator, rate and star band, and writes analytical queries on top of it.
---

# C08 — Osteoporosis Management (2025)

Percent of female plan members who broke a bone and got screening or treatment for osteoporosis within 6 months

- **Program:** STARS
- **Measurement year:** 2025
- **Customer data:** database `DEMO_HealthcareSynthea` (customer id `DEMO_HealthcareSynthea`); reference database `HCLS`
- **Source tables:** `DEMO_HealthcareSynthea.conditions`, `DEMO_HealthcareSynthea.encounters`, `DEMO_HealthcareSynthea.medications`, `DEMO_HealthcareSynthea.patients`, `DEMO_HealthcareSynthea.payer_transitions`, `DEMO_HealthcareSynthea.payers`, `DEMO_HealthcareSynthea.procedures`, `HCLS.measure_value_set_codes`
- **Exported:** 2026-10-26 20:52 UTC from the Star Ratings measure automation app. The SQL below was generated, validated with Teradata `EXPLAIN`, and approved by a human reviewer.

## How to use this skill

Run all SQL with `base_readQuery` (Teradata dialect: use `TOP n`, not `LIMIT`).

1. Treat the **Certified SQL** below as the single source of truth for who is in the denominator and numerator. Never redefine eligibility, exclusions, codes or enrollment.
2. To answer a question (breakdown, filter, trend, gap-to-star, outreach list), reuse the certified CTE chain verbatim up to and including `CTE_05_NUMERATOR` (the "spine") and write only an **analytical tail** that reads from it. Follow the *Analytical query rules* below.
3. Quote the certified rate, star band and cut points from this file; never recompute or invent them. Any analytical result must reconcile to the certified denominator and numerator.
4. For dashboards, build every tile on top of the same spine so all numbers stay consistent with the certified result.

## Certified result

No certified run is on record for this measure.

### Star cut points

- 1 star: < 27 %
- 2 stars: >= 27 % to < 39 %
- 3 stars: >= 39 % to < 52 %
- 4 stars: >= 52 % to < 71 %
- 5 stars: >= 71 %

## Certified SQL

Run on Teradata as-is to reproduce the certified population. The final SELECT returns one row per denominator member (`member_id`, `in_numerator`).

```sql
WITH
CTE_00_IESD AS (
    SELECT
        TRIM(c.patient_id)      AS patient_id
        ,MIN(c.start_date)      AS iesd
    FROM DEMO_HealthcareSynthea.conditions c
    WHERE c.code_system = 'SNOMED-CT'
      AND TRIM(c.code) IN (
            SELECT TRIM(code) FROM HCLS.measure_value_set_codes
            WHERE value_set_oid = 'FRACTURE_VALUE_SET_OID_PENDING_CONFIRMATION'
              AND code_system = 'SNOMED-CT'
              AND measurement_year = 2025)
      AND c.start_date BETWEEN DATE '2024-07-01' AND DATE '2025-06-30'
      -- [Spec2:DENOM_EVENT:DOMAIN_GAP:CODE_SYSTEM] [AE-02] blocking gap — 0 rows expected today
    GROUP BY TRIM(c.patient_id)
)

,CTE_01_MEMBERS AS (
    SELECT DISTINCT
        TRIM(p.patient_id)   AS patient_id                        -- [SN-03]
        ,p.birth_date
        ,p.gender
        ,p.death_date
        ,p.race
        ,p.ethnicity
        ,p.zip
        ,p.city
        ,p.state
        ,p.county
        ,f.iesd
    FROM DEMO_HealthcareSynthea.patients p
    INNER JOIN CTE_00_IESD f
        ON TRIM(p.patient_id) = f.patient_id                        -- [Spec2:DENOM_EVENT:DOMAIN_GAP]
    WHERE p.gender = 'F'                                            -- [Spec2:DENOM_GENDER:MAPPED]
      AND (EXTRACT(YEAR FROM DATE '2025-12-31')
           - EXTRACT(YEAR FROM p.birth_date)) BETWEEN 67 AND 85     -- [Spec2:DENOM_AGE:MAPPED]
      AND (p.death_date IS NULL
           OR p.death_date > (f.iesd + 180))                        -- [Spec2:DENOM_ALIVE:MAPPED] [SN-09]
)

,CTE_ENROLLMENT_SEGMENTS AS (
    SELECT DISTINCT
        TRIM(pt.patient_id)                                              AS patient_id
        ,m.iesd
        ,CAST(CAST(pt.start_year AS VARCHAR(4)) || '-01-01' AS DATE)     AS seg_start   -- [SN-02] start_year/end_year are INTEGER year values, not TIMESTAMP/DATE
        ,CAST(CAST(pt.end_year   AS VARCHAR(4)) || '-12-31' AS DATE)     AS seg_end     -- [SN-02]
    FROM DEMO_HealthcareSynthea.payer_transitions pt
    INNER JOIN CTE_01_MEMBERS m ON TRIM(pt.patient_id) = m.patient_id
    INNER JOIN DEMO_HealthcareSynthea.payers py ON TRIM(pt.primary_payer_id) = TRIM(py.payer_id)  -- [SN-03]
    WHERE py.payer_type = 'GOVERNMENT'                               -- [Spec2:DENOM_PLAN_TYPE:PROXY]
      AND py.payer IN ('Medicare', 'Dual Eligible')
      AND CAST(CAST(pt.start_year AS VARCHAR(4)) || '-01-01' AS DATE) <= (m.iesd + 180)
      AND CAST(CAST(pt.end_year   AS VARCHAR(4)) || '-12-31' AS DATE) >= m.iesd
)

,CTE_ENROLLMENT_GAPS AS (
    SELECT
        patient_id, iesd, seg_start, seg_end
        ,LAG(seg_end) OVER (PARTITION BY patient_id ORDER BY seg_start) AS prev_seg_end
    FROM CTE_ENROLLMENT_SEGMENTS
)

,CTE_ENROLLMENT_CHECK AS (
    SELECT
        patient_id, iesd
        ,MIN(seg_start) AS earliest_start
        ,MAX(seg_end)   AS latest_end
        ,MAX(CASE WHEN prev_seg_end IS NOT NULL
                   AND seg_start > (prev_seg_end + 1)
                  THEN 1 ELSE 0 END) AS has_gap                      -- [SN-06] [AE-14]
    FROM CTE_ENROLLMENT_GAPS
    GROUP BY patient_id, iesd
)

,CTE_02_ENROLLED AS (
    SELECT DISTINCT TRIM(patient_id) AS patient_id                   -- [Spec2:DENOM_ENROLL:PROXY]
    FROM CTE_ENROLLMENT_CHECK
    WHERE earliest_start <= iesd
      AND latest_end >= (iesd + 180)
      AND COALESCE(has_gap, 0) = 0
)

,CTE_03_DENOMINATOR AS (
    SELECT
        m.patient_id, m.birth_date, m.death_date, m.race, m.ethnicity
        ,m.zip, m.city, m.state, m.county, m.iesd
        ,(EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date))  AS age_at_anchor
        ,CASE
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 67 AND 72 THEN 'Lower (67-72)'
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 73 AND 79 THEN 'Mid (73-79)'
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 80 AND 85 THEN 'Upper (80-85)'
         END AS age_tertile                                          -- [Spec2:AQ_AGE_TERTILE:MAPPED]
        ,CASE
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 67 AND 71 THEN '67-71'
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 72 AND 76 THEN '72-76'
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 77 AND 81 THEN '77-81'
            WHEN (EXTRACT(YEAR FROM DATE '2025-12-31') - EXTRACT(YEAR FROM m.birth_date)) BETWEEN 82 AND 85 THEN '82-85'
         END AS age_band_5                                            -- [Spec2:AQ_AGE_BAND_5:MAPPED]
    FROM CTE_01_MEMBERS m
    INNER JOIN CTE_02_ENROLLED e ON m.patient_id = e.patient_id
)

,CTE_04_EXCLUSIONS AS (
    SELECT d.*
    FROM CTE_03_DENOMINATOR d
    WHERE NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.procedures pb
        WHERE TRIM(pb.patient_id) = d.patient_id
          AND pb.code_system = 'SNOMED-CT'
          AND TRIM(pb.code) IN (
                SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                WHERE value_set_oid = 'BMD_VALUE_SET_OID_PENDING_CONFIRMATION'
                  AND code_system = 'SNOMED-CT'
                  AND measurement_year = 2025)
          AND CAST(pb.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN (d.iesd - 730) AND (d.iesd - 1)
    )   -- [Spec2:EXCL_PRIOR_BMD:DOMAIN_GAP:CODE_SYSTEM]
      AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.encounters ec
        WHERE TRIM(ec.patient_id) = d.patient_id
          AND TRIM(ec.code) IN (
                SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                WHERE value_set_oid = 'OSTEO_MED_VALUE_SET_OID_PENDING_CONFIRMATION'
                  AND code_system = 'SNOMED-CT'
                  AND measurement_year = 2025)
          AND CAST(ec.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN (d.iesd - 365) AND (d.iesd - 1)
    )   -- [Spec2:EXCL_PRIOR_OSTEO_CLAIM:DOMAIN_GAP:CODE_SYSTEM]
      AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.medications mx
        WHERE TRIM(mx.patient_id) = d.patient_id
          AND TRIM(mx.code) IN (
                SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                WHERE value_set_oid = 'OSTEO_MED_VALUE_SET_OID_PENDING_CONFIRMATION'
                  AND measurement_year = 2025)
          AND CAST(mx.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN (d.iesd - 365) AND (d.iesd - 1)
    )   -- [Spec2:EXCL_PRIOR_OSTEO_RX:DOMAIN_GAP:CODE_SYSTEM]
      AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.encounters eh
        WHERE TRIM(eh.patient_id) = d.patient_id
          AND TRIM(eh.code) IN (
                SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                WHERE value_set_oid IN ('2.16.840.1.113883.3.464.1004.1761','2.16.840.1.113883.3.464.1004.1762')
                  AND code_system = 'SNOMED-CT'
                  AND measurement_year = 2025)
          AND CAST(eh.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2025-01-01' AND DATE '2025-12-31'
    )   -- [Spec2:EXCL_HOSPICE:DOMAIN_GAP:CODE_SYSTEM] [SN-10]
      AND (d.death_date IS NULL
           OR d.death_date NOT BETWEEN DATE '2025-01-01' AND DATE '2025-12-31')   -- [Spec2:EXCL_DEATH:MAPPED]
      AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.conditions cp
        WHERE TRIM(cp.patient_id) = d.patient_id
          AND TRIM(cp.code) IN (
                SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                WHERE value_set_oid IN ('2.16.840.1.113883.3.464.1004.2225','2.16.840.1.113883.3.464.1004.1450','2.16.840.1.113883.3.464.1004.2224')
                  AND code_system = 'SNOMED-CT'
                  AND measurement_year = 2025)
          AND cp.start_date BETWEEN DATE '2024-07-01' AND DATE '2025-12-31'
    )   -- [Spec2:EXCL_PALLIATIVE:DOMAIN_GAP:CODE_SYSTEM] [SN-10]
      AND NOT (
        d.age_at_anchor BETWEEN 67 AND 80
        AND (
            SELECT COUNT(DISTINCT c.start_date)
            FROM DEMO_HealthcareSynthea.conditions c
            WHERE TRIM(c.patient_id) = d.patient_id
              AND TRIM(c.code) IN (
                    SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                    WHERE value_set_oid IN ('2.16.840.1.113883.3.464.1004.1530','2.16.840.1.113883.3.464.1004.1531',
                                             '2.16.840.1.113883.3.464.1004.1532','2.16.840.1.113883.3.464.1004.1533')
                      AND code_system = 'SNOMED-CT'
                      AND measurement_year = 2025)
              AND c.start_date BETWEEN DATE '2024-07-01' AND DATE '2025-12-31'
        ) >= 2
        AND (
            EXISTS (
                SELECT 1 FROM DEMO_HealthcareSynthea.encounters ai
                WHERE TRIM(ai.patient_id) = d.patient_id
                  AND TRIM(ai.code) IN (
                        SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                        WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1810'
                          AND code_system = 'SNOMED-CT'
                          AND measurement_year = 2025)
                  AND EXISTS (
                        SELECT 1 FROM DEMO_HealthcareSynthea.conditions aidx
                        WHERE TRIM(aidx.patient_id) = d.patient_id
                          AND TRIM(aidx.code) IN (
                                SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                                WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1465'
                                  AND code_system = 'SNOMED-CT'
                                  AND measurement_year = 2025))
                  AND (CAST(ai.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
                       OR CAST(ai.end_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2024-01-01' AND DATE '2025-12-31')
            )
            OR (
                SELECT COUNT(DISTINCT CAST(v.start_ts AT TIME ZONE 'GMT' AS DATE))
                FROM DEMO_HealthcareSynthea.encounters v
                WHERE TRIM(v.patient_id) = d.patient_id
                  AND TRIM(v.code) IN (
                        SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                        WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.2474'
                          AND code_system = 'SNOMED-CT'
                          AND measurement_year = 2025)
                  AND EXISTS (
                        SELECT 1 FROM DEMO_HealthcareSynthea.conditions vdx
                        WHERE TRIM(vdx.patient_id) = d.patient_id
                          AND TRIM(vdx.code) IN (
                                SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                                WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1465'
                                  AND code_system = 'SNOMED-CT'
                                  AND measurement_year = 2025))
                  AND CAST(v.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN DATE '2024-01-01' AND DATE '2025-12-31'
            ) >= 2
            -- criterion (d) dementia medication omitted — [Spec2:EXCL_ADVANCED_ILLNESS:DOMAIN_GAP] [AE-09]
        )
      )   -- [Spec2:EXCL_FRAILTY_ADVANCED_67_80:DOMAIN_GAP(partial):CODE_SYSTEM] [AE-08]
      AND NOT (
        d.age_at_anchor >= 81
        AND (
            SELECT COUNT(DISTINCT c81.start_date)
            FROM DEMO_HealthcareSynthea.conditions c81
            WHERE TRIM(c81.patient_id) = d.patient_id
              AND TRIM(c81.code) IN (
                    SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                    WHERE value_set_oid IN ('2.16.840.1.113883.3.464.1004.1530','2.16.840.1.113883.3.464.1004.1531',
                                             '2.16.840.1.113883.3.464.1004.1532','2.16.840.1.113883.3.464.1004.1533')
                      AND code_system = 'SNOMED-CT'
                      AND measurement_year = 2025)
              AND c81.start_date BETWEEN DATE '2024-07-01' AND DATE '2025-12-31'
        ) >= 2
      )   -- [Spec2:EXCL_FRAILTY_81PLUS:DOMAIN_GAP:CODE_SYSTEM] [AE-08]
)

,CTE_BMD_EVENTS AS (
    SELECT DISTINCT TRIM(pr.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.procedures pr
    INNER JOIN CTE_04_EXCLUSIONS d ON TRIM(pr.patient_id) = d.patient_id
    WHERE pr.code_system = 'SNOMED-CT'
      AND TRIM(pr.code) IN (
            SELECT TRIM(code) FROM HCLS.measure_value_set_codes
            WHERE value_set_oid = 'BMD_VALUE_SET_OID_PENDING_CONFIRMATION'
              AND code_system = 'SNOMED-CT'
              AND measurement_year = 2025)
      AND CAST(pr.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN d.iesd AND (d.iesd + 180)
)   -- [Spec2:NUM_BMD_TEST:DOMAIN_GAP:CODE_SYSTEM]

,CTE_OSTEO_RX_EVENTS AS (
    SELECT DISTINCT TRIM(mx.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.medications mx
    INNER JOIN CTE_04_EXCLUSIONS d ON TRIM(mx.patient_id) = d.patient_id
    WHERE TRIM(mx.code) IN (
            SELECT TRIM(code) FROM HCLS.measure_value_set_codes
            WHERE value_set_oid = 'OSTEO_MED_VALUE_SET_OID_PENDING_CONFIRMATION'
              AND measurement_year = 2025)
      AND CAST(mx.start_ts AT TIME ZONE 'GMT' AS DATE) BETWEEN d.iesd AND (d.iesd + 180)
)   -- [Spec2:NUM_OSTEO_RX:DOMAIN_GAP:CODE_SYSTEM]

,CTE_05_NUMERATOR AS (
    SELECT
        d.patient_id
        ,CASE WHEN bmd.patient_id IS NOT NULL
               OR rx.patient_id IS NOT NULL
              THEN 1 ELSE 0 END AS in_numerator      -- [Spec2:NUM_COUNT_RULE:MAPPED] [AE-06]
    FROM CTE_04_EXCLUSIONS d
    LEFT JOIN CTE_BMD_EVENTS bmd     ON d.patient_id = bmd.patient_id
    LEFT JOIN CTE_OSTEO_RX_EVENTS rx ON d.patient_id = rx.patient_id
)

,CTE_06_RESULT AS (
    SELECT
        TRIM(n.patient_id) AS member_id
        ,CASE WHEN n.in_numerator IS NULL THEN 0 ELSE n.in_numerator END AS in_numerator
    FROM CTE_05_NUMERATOR n
)

,CTE_07_AGGREGATE AS (
    SELECT
        r.member_id                         AS member_id
        ,MAX(r.in_numerator)                AS in_numerator          -- [Spec2:AGG_DEDUP:MAPPED] one row per member
    FROM CTE_06_RESULT r
    GROUP BY r.member_id
)

SELECT
    a.member_id
    ,CASE WHEN a.in_numerator IS NULL THEN 0 ELSE a.in_numerator END AS in_numerator
FROM CTE_07_AGGREGATE a
```

## Spine structure (read-only context)

The spine is the certified SQL from the top through `CTE_05_NUMERATOR`. Columns each CTE exposes (never select a column that is not listed):

- `CTE_00_IESD`(iesd, patient_id)
- `CTE_01_MEMBERS`(birth_date, city, county, death_date, ethnicity, gender, iesd, patient_id, race, state, zip)
- `CTE_ENROLLMENT_SEGMENTS`(iesd, patient_id, seg_end, seg_start)
- `CTE_ENROLLMENT_GAPS`(iesd, patient_id, prev_seg_end, seg_end, seg_start)
- `CTE_ENROLLMENT_CHECK`(earliest_start, has_gap, iesd, latest_end, patient_id)
- `CTE_02_ENROLLED`(patient_id)
- `CTE_03_DENOMINATOR`(age_at_anchor, age_band_5, age_tertile, birth_date, city, county, death_date, ethnicity, iesd, patient_id, race, state, zip)
- `CTE_04_EXCLUSIONS`(age_at_anchor, age_band_5, age_tertile, birth_date, city, county, death_date, ethnicity, iesd, patient_id, race, state, zip)
- `CTE_BMD_EVENTS`(patient_id)
- `CTE_OSTEO_RX_EVENTS`(patient_id)
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
### C08 · Osteoporosis Management in Women Who Had a Fracture (OMW)

---

#### Document Control

| Field            | Value                                              |
|------------------|----------------------------------------------------|
| spec_type        | MEASURE_CONCEPT                                    |
| spec_version     | 1.0                                                 |
| measure_id       | C08                                                 |
| measure_label    | Osteoporosis Management                             |
| hedis_label      | OMW                                                  |
| program          | Medicare Advantage Part C Star Ratings              |
| reference        | NCQA HEDIS Measurement Year 2023 Technical Specifications Volume 2, p.232 |
| cms_cmit         | 00484-02-C-PARTC                                    |
| authored_date    | 2026-07-27                                          |
| status           | ACTIVE                                               |

---

#### 1. Measure Summary

| Field                | Value                                                            |
|----------------------|-------------------------------------------------------------------|
| description          | Percent of female plan members who broke a bone and got screening or treatment for osteoporosis within 6 months |
| direction            | Higher is better                                                   |
| weighting_category   | Process Measure                                                    |
| weighting_value      | 1                                                                   |
| statistical_method   | Clustering                                                          |
| case_mix_adjusted    | No                                                                  |
| display_format       | Percentage with no decimal place                                   |
| meaningful_measure   | Chronic Conditions                                                  |
| primary_data_source  | HEDIS                                                               |
| data_source_category | Health and Drug Plans                                               |

---

#### 2. Measurement Period

| Concept                  | Rule                                                                                 |
|---------------------------|---------------------------------------------------------------------------------------|
| measurement_year         | The calendar year being measured (e.g. 2023)                                        |
| measurement_period_start | January 1 of measurement_year                                                        |
| measurement_period_end   | December 31 of measurement_year                                                      |
| anchor_date              | December 31 of measurement_year (used for age calculation)                          |
| index_event_start_date (IESD) | Date of the qualifying fracture identified during the intake period. Each member has an individual IESD — this is NOT a fixed calendar date |
| intake_period            | Per HEDIS OMW specification: the 12-month period from July 1 of the year prior to measurement_year through June 30 of measurement_year (source: measure_reference p.232) — **agent note: confirm exact intake_period boundaries against the current-year NCQA technical specification; not explicitly restated in the extracted CMS Technical Notes** |
| numerator_lookback_start | IESD (member's individual fracture date)                                             |
| numerator_lookback_end   | IESD + 180 days (6 months)                                                            |
| numerator_window_note    | Six-month (180-day) window following each member's individual index fracture event (IESD). This window is member-specific, NOT calendar-year aligned. Do not apply a fixed January 1–December 31 window to the numerator. |

> **Agent note:** This measure uses a floating, member-specific index event (fracture) rather than a fixed measurement-year window for both numerator evaluation and several exclusion lookbacks (see §7). Do not reuse the C01 fixed two-calendar-year pattern.

---

#### 3. Rate Formula

```
Rate = (Numerator / Denominator) × 100

Reported as: Percentage with no decimal place
```

---

#### 4. Star Rating Cut Points

| Star Band | Rule                         |
|-----------|------------------------------|
| 1 star    | < 27 %                       |
| 2 stars   | >= 27 % to < 39 %             |
| 3 stars   | >= 39 % to < 52 %             |
| 4 stars   | >= 52 % to < 71 %             |
| 5 stars   | >= 71 %                       |

---

#### 5. Denominator — Conceptual Definition

**Plain language:** Female Medicare Advantage enrollees aged 67–85 as of December 31 of the measurement year who suffered a qualifying fracture (index event) during the intake period.

##### 5.1 Denominator Inclusion Criteria

| Concept ID       | Concept Name          | Rule                                                                        | Required |
|------------------|-----------------------|-------------------------------------------------------------------------------|----------|
| DENOM_GENDER     | Member sex             | Female                                                                        | YES      |
| DENOM_AGE        | Age at anchor date     | Age >= 67 AND age <= 85 as of December 31 of measurement_year                | YES      |
| DENOM_PLAN_TYPE  | Plan type              | Medicare Advantage (MA) enrollee                                             | YES      |
| DENOM_ENROLL     | Continuous enrollment  | Continuous enrollment from the IESD through 180 days following the IESD (no enrollment gap permitted) — per HEDIS OMW specification | YES |
| DENOM_ALIVE      | Vital status           | Member must not have died prior to end of the 180-day numerator window (see EXCL_DEATH in §7) | YES |
| DENOM_EVENT      | Index fracture event   | Member must have at least one qualifying fracture diagnosis/event during the intake_period. Fracture code set: **TBD — not present in current MY2025 VSD listing; source references "Fracture Value Set" not found in available value sets** | YES |

##### 5.2 Denominator Minimum Enrollment Threshold

| Concept ID        | Rule                                                                            |
|-------------------|-----------------------------------------------------------------------------------|
| DENOM_MIN_500     | Contracts with enrollment < 500 as of the July 2023 enrollment report are excluded from this measure entirely |
| DENOM_RELIABILITY | Contracts with enrollment >= 500 and < 1,000 AND measure score reliability < 0.7 are excluded |

---

#### 6. Numerator — Conceptual Definition

**Plain language:** Members in the denominator who received either a bone mineral density (BMD) test or a prescription/dispensed drug to treat osteoporosis within 6 months (180 days) after their qualifying fracture (IESD).

##### 6.1 Numerator Inclusion Events

| Concept ID        | Concept Name              | Rule                                                                                |
|--------------------|-----------------------------|----------------------------------------------------------------------------------------|
| NUM_BMD_TEST       | BMD test event              | At least one qualifying bone mineral density test during the 180-day numerator lookback window following IESD |
| NUM_OSTEO_RX       | Osteoporosis treatment event | At least one qualifying osteoporosis therapy dispensed/active prescription during the 180-day numerator lookback window following IESD |
| NUM_DATE_WINDOW    | Date of service window       | Service/dispense date >= IESD AND <= (IESD + 180 days)                                 |
| NUM_COUNT_RULE     | Minimum event count          | One qualifying event of EITHER type (BMD test OR osteoporosis treatment) satisfies the numerator. Multiple events do not increase the numerator count beyond 1 per member |

##### 6.2 Qualifying Event Code Sets

###### Event Type 1 — Bone Mineral Density (BMD) Test

> Lookback window: 180 days (6 months) following IESD

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|-----------------|-----|------|
| TBD             | TBD | The source specification references a "Bone Mineral Density Tests Value Set." No matching value set was found in the available MY2025 HEDIS VSD listing. Flag as DOMAIN_GAP:CODE_SYSTEM pending confirmation of correct value set name/OID from NCQA publisher documentation. |

###### Event Type 2 — Osteoporosis Treatment (Medication)

> Lookback window: 180 days (6 months) following IESD

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|-----------------|-----|------|
| TBD             | TBD | The source specification references an "Osteoporosis Medications Value Set" / "Osteoporosis Medications List." No matching value set was found in the available MY2025 HEDIS VSD listing. Flag as DOMAIN_GAP:CODE_SYSTEM pending confirmation. |

> **Agent note:** Both event types are drawn from the same 180-day post-fracture window. A member satisfying either event type independently qualifies for the numerator — do not require both.

---

#### 7. Exclusions — Conceptual Definitions

> Exclusions are applied after the denominator cohort (including the qualifying fracture/IESD) is identified and before the numerator is evaluated.
> Order of application: EXCL_PRIOR_BMD → EXCL_PRIOR_OSTEO_CLAIM → EXCL_PRIOR_OSTEO_RX → EXCL_HOSPICE → EXCL_DEATH → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY_ADVANCED_67_80 → EXCL_FRAILTY_81PLUS

##### 7.1 Prior BMD Test Exclusion

| Concept ID        | Rule                                                                                          |
|--------------------|-------------------------------------------------------------------------------------------------|
| EXCL_PRIOR_BMD     | Exclude members who had a BMD test during the 730 days (24 months) prior to the IESD           |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|-----------------|-----|------|
| TBD             | TBD | Same "Bone Mineral Density Tests" value set referenced in §6.2 Event Type 1. Not found in current VSD listing — flag as DOMAIN_GAP:CODE_SYSTEM. |

##### 7.2 Prior Osteoporosis Therapy Claim Exclusion

| Concept ID              | Rule                                                                                    |
|--------------------------|--------------------------------------------------------------------------------------------|
| EXCL_PRIOR_OSTEO_CLAIM   | Exclude members who had a claim/encounter for osteoporosis therapy during the 365 days (12 months) prior to the IESD |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|-----------------|-----|------|
| TBD             | TBD | Same "Osteoporosis Medications" value set referenced in §6.2 Event Type 2. Not found in current VSD listing — flag as DOMAIN_GAP:CODE_SYSTEM. |

##### 7.3 Prior Osteoporosis Prescription Exclusion

| Concept ID           | Rule                                                                                          |
|-----------------------|--------------------------------------------------------------------------------------------------|
| EXCL_PRIOR_OSTEO_RX   | Exclude members who received a dispensed prescription or had an active prescription to treat osteoporosis during the 365 days (12 months) prior to the IESD |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|-----------------|-----|------|
| TBD             | TBD | Same "Osteoporosis Medications List" referenced in §6.2 Event Type 2. Not found in current VSD listing — flag as DOMAIN_GAP:CODE_SYSTEM. |

##### 7.4 Hospice Exclusion

| Concept ID    | Rule                                                                                          |
|---------------|--------------------------------------------------------------------------------------------------|
| EXCL_HOSPICE  | Exclude members in hospice or using hospice services any time during the measurement year        |

**VALUE_SET_REFERENCES**
| Value Set Name       | OID                                    |
|-----------------------|------------------------------------------|
| Hospice Encounter     | 2.16.840.1.113883.3.464.1004.1761       |
| Hospice Intervention  | 2.16.840.1.113883.3.464.1004.1762       |

##### 7.5 Death Exclusion

| Concept ID  | Rule                                                          |
|-------------|-----------------------------------------------------------------|
| EXCL_DEATH  | Exclude members who died any time during the measurement year   |
| source      | Enrollment/eligibility file — vital status flag                 |

##### 7.6 Palliative Care Exclusion

| Concept ID       | Rule                                                                                                 |
|-------------------|---------------------------------------------------------------------------------------------------------|
| EXCL_PALLIATIVE   | Exclude members who received palliative care any time during the intake period through the end of the measurement year |

**VALUE_SET_REFERENCES**
| Value Set Name              | OID                                     |
|-------------------------------|-------------------------------------------|
| Palliative Care Assessment    | 2.16.840.1.113883.3.464.1004.2225        |
| Palliative Care Encounter     | 2.16.840.1.113883.3.464.1004.1450        |
| Palliative Care Intervention  | 2.16.840.1.113883.3.464.1004.2224        |

##### 7.7 Institutional SNP (I-SNP) Exclusion

| Concept ID | Rule                                                                                                          |
|------------|-------------------------------------------------------------------------------------------------------------|
| EXCL_ISNP  | Exclude members aged >= 67 as of December 31 of measurement_year enrolled in an Institutional SNP (I-SNP) at any time during the measurement_year |
| age_gate   | >= 67 as of anchor_date                                                                                      |
| source     | Plan type / SNP type indicator in enrollment or eligibility data                                             |

##### 7.8 Long-Term Institutional (LTI) Exclusion

| Concept ID | Rule                                                                                                                     |
|------------|---------------------------------------------------------------------------------------------------------------------------|
| EXCL_LTI   | Exclude members aged >= 67 as of December 31 of measurement_year living long-term in an institution any time during the measurement_year |
| age_gate   | >= 67 as of anchor_date                                                                                                    |
| source     | External CMS Monthly Membership Detail Data File — LTI flag field — not derivable from claims                            |

##### 7.9 Frailty + Advanced Illness Exclusion (Age 67–80)

> Members aged 67–80 as of December 31 of measurement_year must meet BOTH the frailty criterion AND the advanced illness criterion to be excluded.

###### 7.9a Frailty Criterion

| Concept ID     | Rule                                                                                                              |
|-----------------|---------------------------------------------------------------------------------------------------------------------|
| EXCL_FRAILTY    | At least two frailty indicators with different dates of service during the intake period through the end of the measurement_year |

**VALUE_SET_REFERENCES**
| Value Set Name    | OID                                    |
|--------------------|-------------------------------------------|
| Frailty Device     | 2.16.840.1.113883.3.464.1004.1530        |
| Frailty Diagnosis   | 2.16.840.1.113883.3.464.1004.1531        |
| Frailty Encounter   | 2.16.840.1.113883.3.464.1004.1532        |
| Frailty Symptom     | 2.16.840.1.113883.3.464.1004.1533        |

###### 7.9b Advanced Illness Criterion

| Concept ID              | Rule                                                                                                                                     |
|--------------------------|---------------------------------------------------------------------------------------------------------------------------------------------|
| EXCL_ADVANCED_ILLNESS    | Any of the following during measurement_year or the year prior to measurement_year: (a) at least two qualifying visits with an advanced illness diagnosis on different dates of service; (b) at least one acute inpatient encounter with an advanced illness diagnosis; (c) at least one acute inpatient discharge with an advanced illness diagnosis on the discharge claim; OR (d) a dispensed dementia medication |

**Qualifying Visit Types (criterion a):** Outpatient, observation, ED, telephone, e-visit/virtual check-in, nonacute inpatient encounter, nonacute inpatient discharge

**VALUE_SET_REFERENCES**
| Value Set Name                                                          | OID                                    | Use                                    |
|----------------------------------------------------------------------------|-------------------------------------------|-------------------------------------------|
| Advanced Illness                                                          | 2.16.840.1.113883.3.464.1004.1465        | Advanced illness diagnosis                |
| Outpatient, ED, Telephone, Acute Inpatient and Nonacute Inpatient          | 2.16.840.1.113883.3.464.1004.2474        | Qualifying visit types (criterion a)      |
| Acute Inpatient                                                            | 2.16.840.1.113883.3.464.1004.1810        | Acute inpatient encounter (criterion b/c) |
| TBD (Dementia Medication)                                                  | TBD                                       | Dispensed dementia medication (criterion d) — not found in current VSD listing; flag as DOMAIN_GAP:CODE_SYSTEM |

##### 7.10 Frailty-Only Exclusion (Age 81+)

| Concept ID           | Rule                                                                                                             |
|------------------------|---------------------------------------------------------------------------------------------------------------------|
| EXCL_FRAILTY_81PLUS   | Exclude members aged >= 81 as of December 31 of measurement_year with at least two frailty indicators with different dates of service during the intake period through the end of the measurement_year |
| age_gate               | >= 81 as of anchor_date                                                                                            |

**VALUE_SET_REFERENCES**
| Value Set Name    | OID                                    |
|--------------------|-------------------------------------------|
| Frailty Device     | 2.16.840.1.113883.3.464.1004.1530        |
| Frailty Diagnosis   | 2.16.840.1.113883.3.464.1004.1531        |
| Frailty Encounter   | 2.16.840.1.113883.3.464.1004.1532        |
| Frailty Symptom     | 2.16.840.1.113883.3.464.1004.1533        |

---

#### 8. Analytical Query Concepts

| Concept ID          | Concept Name             | Analytical Use                                                              |
|-----------------------|-----------------------------|---------------------------------------------------------------------------------|
| AQ_AGE_TERTILE       | Age tertile                | Divide 67–85 range into thirds: 67–72, 73–79, 80–85                            |
| AQ_AGE_BAND_5        | 5-year age band             | Standard 5-year bands: 67–71, 72–76, 77–81, 82–85                              |
| AQ_GEOGRAPHY         | Geographic segment          | Zip, county, state from member address                                        |
| AQ_RACE_ETHNICITY    | Race and ethnicity          | Health equity sub-analysis                                                     |
| AQ_PROVIDER          | Attributed provider         | PCP or last-touch provider for outreach targeting                             |
| AQ_TREND             | Time trend                  | Rolling monthly or quarterly rate within certified denominator                 |
| AQ_GAP_TO_CUT        | Gap to cut point            | Members needed in numerator to reach next star band                           |
| AQ_OUTREACH          | Outreach target list        | Denominator members NOT in numerator, by AQ dimension                        |
| AQ_TREATMENT_TYPE    | BMD test vs. medication     | Segment numerator-satisfying members by which event type (BMD test / osteoporosis Rx) satisfied the numerator |
| AQ_DAYS_TO_TREATMENT | Time-to-treatment           | Distribution of days elapsed between IESD and qualifying numerator event      |

> **Guardrail:** All analytical queries must operate within the certified denominator. The agent must NOT redefine eligibility criteria. All analytical output must be labeled result_type = ANALYTICAL.

---

#### 9. Output Schema — Certified Result Record

| Field             | Type      | Description                                          |
|-------------------|-----------|-------------------------------------------------------|
| measure_id        | VARCHAR   | C08                                                    |
| measurement_year  | INTEGER   | e.g. 2023                                              |
| customer_id       | VARCHAR   | Payer/customer identifier                              |
| run_id            | VARCHAR   | Unique execution identifier                            |
| run_ts            | TIMESTAMP | When the calculation was executed                      |
| denominator_count | INTEGER   | Count of eligible members after exclusions             |
| numerator_count   | INTEGER   | Count of denominator members with qualifying event     |
| rate              | DECIMAL   | numerator / denominator × 100                          |
| rate_display      | INTEGER   | Rounded integer for CMS display                        |
| star_band         | INTEGER   | 1–5 based on cut points                                |
| result_type       | VARCHAR   | CERTIFIED or ANALYTICAL                                |
| below_threshold   | BOOLEAN   | TRUE if contract enrollment < 500                       |
| spec1_version     | VARCHAR   | Version of this spec used                              |
| spec2_version     | VARCHAR   | Version of customer data model spec used               |
| spec3_version     | VARCHAR   | Version of compute spec used                            |
| gap_flags         | VARCHAR   | Pipe-delimited list of gap flags from Spec 2            |

---

#### 10. Agent Instructions

##### 10.1 Certified SQL Agent Rules

1. Always resolve code sets from section 6.2/7.x using the code system declared in Spec 2. If only SNOMED-CT is available where CPT/HCPCS is expected, set gap_flag = DOMAIN_GAP:CODE_SYSTEM.
2. The fracture (index) event, BMD test value set, osteoporosis medication value set, and dementia medication value set are currently marked TBD — not resolvable from the MY2025 VSD listing provided. Do not fabricate codes. Set gap_flag = DOMAIN_GAP:CODE_SYSTEM for each and surface to the data steward for confirmation before certifying results.
3. This measure does NOT use a fixed calendar-year numerator window. Compute IESD per member as the date of the qualifying fracture event, then evaluate the numerator window as IESD through IESD + 180 days. Do not substitute a January 1–December 31 window.
4. Exclusions EXCL_PRIOR_BMD, EXCL_PRIOR_OSTEO_CLAIM, and EXCL_PRIOR_OSTEO_RX are evaluated relative to each member's individual IESD (730 days and 365 days prior, respectively) — NOT relative to the measurement year boundaries.
5. Apply exclusions in the order specified in the §7 header.
6. EXCL_ISNP and EXCL_LTI are age-gated (>= 67 as of anchor_date).
7. EXCL_FRAILTY_ADVANCED_67_80 applies only to members aged 67–80 as of anchor_date; EXCL_FRAILTY_81PLUS applies only to members aged 81+ as of anchor_date. These two frailty-based exclusions are mutually exclusive by age band — do not apply both to the same member.
8. For the advanced illness criterion (§7.9b), a member is excluded if ANY ONE of the four sub-criteria (two qualifying visits, one acute inpatient encounter, one acute inpatient discharge, or a dispensed dementia medication) is met — these are OR conditions, not AND.
9. If Spec 2 marks a concept as MISSING, omit that exclusion and record it in gap_flags.
10. If Spec 2 marks a concept as PROXY, apply proxy logic and set gap_flag = PROXY:<CONCEPT_ID>.
11. If Spec 2 marks a concept as SYNTHETIC_DATA_ARTIFACT, omit entirely and set gap_flag = SYNTHETIC_DATA_ARTIFACT:<CONCEPT_ID>.
12. A member satisfies the numerator with one qualifying event across either event type (BMD test OR osteoporosis treatment). Do not require both event types.
13. Never produce a rate from a denominator of zero. Return denominator_count = 0 and rate = NULL.

##### 10.2 Analytical Query Agent Rules

1. Parse the user's natural language query to identify the AQ dimension(s) from section 8.
2. Apply the full certified denominator (including the fracture/IESD identification and all exclusions) before segmenting. Never segment before filtering.
3. For AQ_AGE_TERTILE: divide the 67–85 range as follows — Lower: 67–72, Mid: 73–79, Upper: 80–85.
4. For AQ_GAP_TO_CUT: compute members_needed = CEIL((cut_point / 100) × denominator_count) − numerator_count for each band above the current rate.
5. For AQ_TREATMENT_TYPE: segment numerator-satisfying members into "BMD test only," "osteoporosis treatment only," or "both," based on which qualifying event(s) occurred in the member's 180-day window.
6. For AQ_OUTREACH: return denominator members with no qualifying event in their individual 180-day window, sorted by AQ dimension then age DESC.
7. Always label output result_type = ANALYTICAL.
8. Include the certified rate alongside every sub-analysis.
9. Refuse any query that redefines denominator eligibility (e.g., changing the age band, sex restriction, or fracture qualification). Explain why and offer the nearest valid alternative (e.g., an age tertile sub-analysis).

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
