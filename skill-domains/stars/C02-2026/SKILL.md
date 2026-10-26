---
name: stars-c02-2026
description: >-
  STARS measure C02 (Colorectal Cancer Screening), measurement year 2026. Trigger when the user asks about
  C02 or "Colorectal Cancer Screening", e.g. "what is our C02 rate", "C02 star rating", "show C02 care gaps",
  "who is missing C02", "break C02 down by age/provider/region", or to build a C02 dashboard.
  Uses the approved, EXPLAIN-validated certified Teradata SQL as the single source of truth for the
  denominator, numerator, rate and star band, and writes analytical queries on top of it.
---

# C02 — Colorectal Cancer Screening (2026)

Percent of Medicare Advantage enrollees who had an appropriate screening for colorectal cancer

- **Program:** STARS  |  **Measurement year:** 2026
- **Customer data:** database `DEMO_HealthcareSynthea`; **reference database:** `HCLS`
- **Source tables:** `DEMO_HealthcareSynthea.conditions`, `DEMO_HealthcareSynthea.observations`, `DEMO_HealthcareSynthea.patients`, `DEMO_HealthcareSynthea.payer_transitions`, `DEMO_HealthcareSynthea.payers`, `DEMO_HealthcareSynthea.procedures`, `HCLS.measure_value_set_codes`
- **Exported:** 2026-10-26 from the Star Ratings measure automation app. The SQL below was generated, validated with Teradata `EXPLAIN`, and approved by a human reviewer.

## How to use this skill

Run all SQL with `base_readQuery` (Teradata dialect: `TOP n`, not `LIMIT`).

1. **Report the certified result** from the table below when asked for the rate or star band. Do not recompute it unless asked to re-run; if asked, run the certified SQL as-is and aggregate `in_numerator`.
2. **Answer analytical questions** (breakdowns, filters, gap-to-star, outreach lists) by reusing the certified spine and writing only an analytical tail. See *Writing analytical queries*.
3. **Build dashboards** by following *Dashboards*.
4. If a request would change who is in the measure, stop and explain; this skill analyses the certified population, it does not redefine it.

## Certified result

| Item | Value |
|---|---|
| Certified rate | 63.07% |
| Star band | not assigned |
| Denominator | 2361 |
| Numerator | 1489 |
| Run time | 2026-10-19 12:33:43 |

### Star cut points

- 1 star: Less than 58%
- 2 stars: Greater than or equal to 58% to less than 71%
- 3 stars: Greater than or equal to 71% to less than 76%
- 4 stars: Greater than or equal to 76% to less than 84%
- 5 stars: Greater than or equal to 84%

## Measure definition (Spec 1, approved)

## Spec 1 — Measure Concept Specification
## C02 · Colorectal Cancer Screening (COL)

---

### 1. Measure Summary

| Field | Value |
|---|---|
| description | Percent of Medicare Advantage enrollees who had an appropriate screening for colorectal cancer |
| direction | Higher is better |
| weighting_category | Process Measure |
| weighting_value | 1 |
| statistical_method | Clustering |
| case_mix_adjusted | No |
| display_format | Percentage with no decimal place |
| meaningful_measure | Wellness and Prevention |
| primary_data_source | HEDIS |
| data_source_category | Health and Drug Plans |

> **Agent note:** Fields `metric`, `measure_reference`, `primary_data_source`, `exclusions`, `data_source_category`, `label_for_stars`, `hedis_label`, `description`, and `label_for_data` were not found verbatim in the extracted CMS Technical Notes text. Values above for `description`, `primary_data_source`, `data_source_category`, and `hedis_label` are inferred from the standard NCQA HEDIS COL measure definition and should be validated against the full NCQA HEDIS MY2024 Volume 2 specification before certification.

### 2. Measurement Period

| Concept | Rule |
|---|---|
| measurement_year | 2024 (per data_time_frame 01/01/2024 – 12/31/2024) |
| measurement_period_start | January 1, 2024 |
| measurement_period_end | December 31, 2024 |
| anchor_date | December 31, 2024 (used for age calculation) |
| numerator_lookback_start | Varies by qualifying event type — see Section 6.1. Ranges from January 1 of measurement_year (FOBT) to January 1 of (measurement_year − 9) (Colonoscopy) |
| numerator_lookback_end | December 31 of measurement_year (all event types) |
| numerator_window_note | Multiple qualifying event types, each with its OWN lookback window: FOBT = 1 year (measurement year only); FIT-DNA = 3 years (measurement year + 2 prior years); Flexible Sigmoidoscopy = 5 years (measurement year + 4 prior years); CT Colonography = 5 years (measurement year + 4 prior years); Colonoscopy = 10 years (measurement year + 9 prior years) |

> **Agent note:** Do NOT apply a single lookback window to all event types. Each NUM_[EVENT] row in Section 6.1 carries its own window. This mirrors the COL guidance in the Spec 1 template — resolve from source spec, never assume a uniform two-year window.

### 3. Rate Formula

```
Rate = (Numerator / Denominator) × 100

Reported as: Percentage with no decimal place
```

### 4. Star Rating Cut Points

| Star Band | Rule |
|---|---|
| 1 star | Less than 58% |
| 2 stars | Greater than or equal to 58% to less than 71% |
| 3 stars | Greater than or equal to 71% to less than 76% |
| 4 stars | Greater than or equal to 76% to less than 84% |
| 5 stars | Greater than or equal to 84% |

> Source: CMS 2026 Star Ratings Technical Notes (measurement year 2024 data). Cut points change annually — always pull from the correct year's Technical Notes.
> **Major disaster provision:** Higher of measure star (2025–2026) applied for contracts with 25% or more enrolled membership affected by 2024 disasters, per CMS Technical Notes.

### 5. Denominator — Conceptual Definition

**Plain language:** Medicare Advantage enrollees aged 45 to 75 as of December 31 of the measurement year who were continuously enrolled during the measurement year and had an appropriate colorectal cancer screening test performed within the applicable lookback window for that test type.

#### 5.1 Denominator Inclusion Criteria

| Concept ID | Concept Name | Rule | Required |
|---|---|---|---|
| DENOM_AGE | Age at anchor date | Age >= 45 AND age <= 75 as of December 31 of measurement_year | YES |
| DENOM_PLAN_TYPE | Plan type | Medicare Advantage (MA) enrollee — includes 1876 Cost, CCP w/o SNP, CCP with SNP, CCP with Only I-SNP, MSA, PFFS. Excludes standalone PDP | YES |
| DENOM_ENROLL | Continuous enrollment | Continuously enrolled during the measurement year, with allowable gap no greater than one month | YES |
| DENOM_ALIVE | Vital status | Member must not have died prior to December 31 of measurement_year | YES |

> **Agent note:** Reporting requirements table indicates this measure is reported for 1876 Cost, CCP w/o SNP, CCP with SNP, CCP with Only I-SNP, MSA, and PFFS contract types; it is NOT reported for standalone PDP contracts.

#### 5.2 Denominator Minimum Enrollment Threshold

| Concept ID | Rule |
|---|---|
| DENOM_MIN_500 | Contracts with enrollment < 500 as of the July enrollment report are excluded |
| DENOM_RELIABILITY | Contracts with enrollment >= 500 and < 1,000 AND reliability < 0.7 are excluded |

### 6. Numerator — Conceptual Definition

**Plain language:** At least one qualifying colorectal cancer screening test — FOBT, FIT-DNA, flexible sigmoidoscopy, CT colonography, or colonoscopy — completed within the lookback window specific to that test type, ending December 31 of the measurement year.

#### 6.1 Numerator Inclusion Events

| Concept ID | Concept Name | Rule |
|---|---|---|
| NUM_FOBT | FOBT event | At least one qualifying FOBT (gFOBT or FIT) during measurement_year (1-year lookback) |
| NUM_FITDNA | FIT-DNA event | At least one qualifying FIT-DNA (sDNA-FIT) test during measurement_year or 2 prior years (3-year lookback) |
| NUM_FLEXSIG | Flexible sigmoidoscopy event | At least one qualifying flexible sigmoidoscopy during measurement_year or 4 prior years (5-year lookback) |
| NUM_CTCOLON | CT colonography event | At least one qualifying CT colonography during measurement_year or 4 prior years (5-year lookback) |
| NUM_COLONOSCOPY | Colonoscopy event | At least one qualifying colonoscopy during measurement_year or 9 prior years (10-year lookback) |
| NUM_DATE_WINDOW | Date of service | Service date must fall within the specific lookback window defined for that event type |
| NUM_COUNT_RULE | Minimum event count | One qualifying event of ANY type satisfies the numerator. Additional events do not increase the count |

#### 6.2 Qualifying Event Code Sets

> Reference the following HEDIS Value Set Directory (VSD) entries by name and OID. Do not enumerate individual codes here — resolve at compute time via Spec 2/Spec 3 code system bindings.

##### FOBT

> Lookback window: 1 year (measurement_year only)

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| FOBT Lab Test | 2.16.840.1.113883.3.464.1004.1959 | CPT, HCPCS, LOINC |
| FOBT Test Result or Finding | 2.16.840.1.113883.3.464.1004.1960 | SNOMED CT US Edition |

##### FIT-DNA

> Lookback window: 3 years (measurement_year + 2 prior years)

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| sDNA FIT Lab Test | 2.16.840.1.113883.3.464.1004.1749 | CPT, LOINC |

##### Flexible Sigmoidoscopy

> Lookback window: 5 years (measurement_year + 4 prior years)

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Flexible Sigmoidoscopy | 2.16.840.1.113883.3.464.1004.1102 | CPT, HCPCS, SNOMED CT US Edition |

##### CT Colonography

> Lookback window: 5 years (measurement_year + 4 prior years)

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| CT Colonography | 2.16.840.1.113883.3.464.1004.1421 | CPT, LOINC |

##### Colonoscopy

> Lookback window: 10 years (measurement_year + 9 prior years)

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Colonoscopy | 2.16.840.1.113883.3.464.1004.1064 | CPT, HCPCS, SNOMED CT US Edition |

> **Agent note:** No individual CPT/HCPCS/LOINC/SNOMED codes are listed in this spec. All code resolution must occur via the named Value Set OID at compute time using the code system(s) declared available in Spec 2.

### 7. Exclusions — Conceptual Definitions

> Exclusions are applied after the denominator cohort is built and before the numerator is evaluated. Each exclusion removes members entirely.
> Order of application: EXCL_HOSPICE → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY → EXCL_COLORECTAL_HISTORY → EXCL_COLECTOMY

#### 7.1 Hospice Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_HOSPICE | Exclude members in hospice or using hospice services during measurement_period |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Hospice Encounter | 2.16.840.1.113883.3.464.1004.1761 | HCPCS, SNOMED CT US Edition, UBREV |
| Hospice Intervention | 2.16.840.1.113883.3.464.1004.1762 | CPT, HCPCS, SNOMED CT US Edition |

#### 7.2 Palliative Care Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_PALLIATIVE | Exclude members receiving palliative care during measurement_year |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Palliative Care Encounter | 2.16.840.1.113883.3.464.1004.1450 | HCPCS, SNOMED CT US Edition |
| Palliative Care Assessment | 2.16.840.1.113883.3.464.1004.2225 | SNOMED CT US Edition |
| Palliative Care Intervention | 2.16.840.1.113883.3.464.1004.2224 | SNOMED CT US Edition |

#### 7.3 Institutional SNP (I-SNP) Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_ISNP | Exclude members age >= 66 enrolled in I-SNP at any time during measurement_year |
| age_gate | >= 66 as of anchor_date |
| source | Plan type / SNP type indicator in enrollment or eligibility data — not a HEDIS VSD code set |

#### 7.4 Long-Term Institutional (LTI) Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_LTI | Exclude members age >= 66 with LTI flag in CMS Monthly Membership Detail File |
| age_gate | >= 66 as of anchor_date |
| source | External CMS file — not derivable from claims or HEDIS VSD |

#### 7.5 Frailty + Advanced Illness Exclusion

> Members age >= 66 must meet BOTH frailty AND advanced illness criteria on different dates during measurement_period.
> Age gate: >= 66 as of anchor_date.

##### 7.5a Frailty Criterion

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Frailty Device | 2.16.840.1.113883.3.464.1004.1530 | HCPCS |
| Frailty Diagnosis | 2.16.840.1.113883.3.464.1004.1531 | ICD10CM, SNOMED CT US Edition |
| Frailty Encounter | 2.16.840.1.113883.3.464.1004.1532 | CPT, HCPCS |
| Frailty Symptom | 2.16.840.1.113883.3.464.1004.1533 | ICD10CM, SNOMED CT US Edition |

##### 7.5b Advanced Illness Criterion

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Advanced Illness | 2.16.840.1.113883.3.464.1004.1465 | ICD10CM, SNOMED CT US Edition |

**Qualifying Visit Types for Advanced Illness:** Outpatient, Observation, ED, Telephone, E-visit, Nonacute Inpatient (per standard HEDIS visit-type value sets, e.g. "Outpatient, ED, Telephone, Acute Inpatient and Nonacute Inpatient" [OID 2.16.840.1.113883.3.464.1004.2474]).

#### 7.6 Colorectal Cancer / Total Colectomy History Exclusion

> Exclude members with evidence of a diagnosis of colorectal cancer or a history of total colectomy at any time in the member's history through December 31 of the measurement_year.

| Concept ID | Rule |
|---|---|
| EXCL_COLORECTAL_CANCER_HISTORY | Diagnosis of colorectal cancer or history of colorectal cancer, any date |
| EXCL_TOTAL_COLECTOMY | Procedure or diagnosis indicating total colectomy, any date |
| history_window | Anytime in member history through end of measurement_year |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Colorectal Cancer and History of Colorectal Cancer | 2.16.840.1.113883.3.464.1004.1065 | ICD10CM, SNOMED CT US Edition |
| Total Colectomy | 2.16.840.1.113883.3.464.1004.1250 | CPT, ICD10PCS, SNOMED CT US Edition |

### 8. Analytical Query Concepts

| Concept ID | Concept Name | Analytical Use |
|---|---|---|
| AQ_AGE_TERTILE | Age tertile | Divide 45–75 range into thirds: 45–55, 56–65, 66–75 |
| AQ_AGE_BAND_5 | 5-year age band | Standard 5-year bands: 45–49, 50–54, 55–59, 60–64, 65–69, 70–75 |
| AQ_GEOGRAPHY | Geographic segment | Zip, county, state from member address |
| AQ_RACE_ETHNICITY | Race and ethnicity | Health equity sub-analysis |
| AQ_PROVIDER | Attributed provider | PCP or last-touch provider for outreach targeting |
| AQ_TREND | Time trend | Rolling monthly or quarterly rate within certified denominator |
| AQ_GAP_TO_CUT | Gap to cut point | Members needed in numerator to reach next star band |
| AQ_OUTREACH | Outreach target list | Denominator members NOT in numerator, by AQ dimension |
| AQ_TEST_TYPE | Screening test type | Breaks out numerator by qualifying event type (FOBT / FIT-DNA / Flex Sig / CT Colonography / Colonoscopy) — used to identify which screening modality drove compliance |

> **Guardrail:** All analytical queries must operate within the certified denominator. The agent must NOT redefine eligibility criteria. All analytical output must be labeled result_type = ANALYTICAL.

## Certified SQL

Running this as-is reproduces the certified population: one row per denominator member (`member_id`, `in_numerator`).

```sql
WITH
CTE_01_MEMBERS AS (
    SELECT DISTINCT
        TRIM(p.patient_id)      AS patient_id                       -- [SN-04]
        ,p.birth_date
        ,p.race                                                     -- [Spec2:AQ_RACE_ETHNICITY:MAPPED]
        ,p.ethnicity                                                 -- [Spec2:AQ_RACE_ETHNICITY:MAPPED]
        ,p.zip                                                       -- [Spec2:AQ_GEOGRAPHY:MAPPED]
        ,p.city                                                      -- [Spec2:AQ_GEOGRAPHY:MAPPED]
        ,p.state                                                     -- [Spec2:AQ_GEOGRAPHY:MAPPED]
        ,p.county                                                    -- [Spec2:AQ_GEOGRAPHY:MAPPED]
    FROM DEMO_HealthcareSynthea.patients p
    WHERE (EXTRACT(YEAR FROM DATE '2026-12-31') - EXTRACT(YEAR FROM p.birth_date))
              BETWEEN 45 AND 75                                      -- [Spec2:DENOM_AGE:MAPPED]
      AND (p.death_date IS NULL OR p.death_date > DATE '2026-12-31') -- [Spec2:DENOM_ALIVE:MAPPED]
)
,enroll_segments AS (
    SELECT
        TRIM(pt.patient_id) AS patient_id,                           -- [SN-04]
        CAST(pt.start_year AS DATE) AS seg_start,                     -- [SN-03]
        CAST(pt.end_year   AS DATE) AS seg_end                        -- [SN-03]
    FROM DEMO_HealthcareSynthea.payer_transitions pt
    INNER JOIN DEMO_HealthcareSynthea.payers py
        ON TRIM(pt.primary_payer_id) = TRIM(py.payer_id)             -- [SN-04]
    WHERE py.payer_type = 'GOVERNMENT'                               -- [Spec2:DENOM_PLAN_TYPE:PROXY] [SN-06]
      AND py.payer IN ('Medicare','Dual Eligible')                   -- [Spec2:DENOM_PLAN_TYPE:PROXY]
      AND CAST(pt.start_year AS DATE) <= DATE '2026-12-31'
      AND CAST(pt.end_year   AS DATE) >= DATE '2026-01-01'
)
,gapped AS (
    SELECT
        patient_id, seg_start, seg_end,
        seg_start - LAG(seg_end) OVER (PARTITION BY patient_id ORDER BY seg_start) AS gap_days
    FROM enroll_segments
)
,episoded AS (
    SELECT
        patient_id, seg_start, seg_end,
        SUM(CASE WHEN gap_days IS NULL OR gap_days <= 31 THEN 0 ELSE 1 END)
            OVER (PARTITION BY patient_id ORDER BY seg_start ROWS UNBOUNDED PRECEDING) AS episode_id
    FROM gapped
)
,episode_span AS (
    SELECT patient_id, episode_id,
           MIN(seg_start) AS episode_start,
           MAX(seg_end)   AS episode_end
    FROM episoded
    GROUP BY patient_id, episode_id
)
,CTE_02_ENROLLED AS (
    SELECT DISTINCT TRIM(patient_id) AS patient_id                   -- [Spec2:DENOM_ENROLL:PROXY:CERTIFIED_STITCHED] [SN-07]
    FROM episode_span
    WHERE episode_start <= DATE '2026-01-01'
      AND episode_end   >= DATE '2026-12-31'
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
        ,(EXTRACT(YEAR FROM DATE '2026-12-31') - EXTRACT(YEAR FROM m.birth_date)) AS age_at_anchor
    FROM CTE_01_MEMBERS m
    INNER JOIN CTE_02_ENROLLED e
        ON TRIM(m.patient_id) = TRIM(e.patient_id)                     -- [SN-04]
)
,CTE_04_EXCLUSIONS AS (
    SELECT d.*
    FROM CTE_03_DENOMINATOR d
    WHERE NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.procedures hp
        WHERE TRIM(hp.patient_id) = TRIM(d.patient_id)
          AND hp.code_system = 'SNOMED-CT'
          AND TRIM(hp.code) IN (
              SELECT TRIM(code) FROM HCLS.measure_value_set_codes
              WHERE value_set_oid IN ('2.16.840.1.113883.3.464.1004.1761','2.16.840.1.113883.3.464.1004.1762')
                AND measurement_year = 2026)
          AND CAST(hp.start_ts AT TIME ZONE 'GMT' AS DATE)
                BETWEEN DATE '2026-01-01' AND DATE '2026-12-31'
    )                                                                   -- [Spec2:EXCL_HOSPICE:MAPPED]
    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.conditions pc
        WHERE TRIM(pc.patient_id) = TRIM(d.patient_id)
          AND pc.code_system = 'SNOMED-CT'
          AND TRIM(pc.code) IN (
              SELECT TRIM(code) FROM HCLS.measure_value_set_codes
              WHERE value_set_oid IN (
                  '2.16.840.1.113883.3.464.1004.1450',
                  '2.16.840.1.113883.3.464.1004.2225',
                  '2.16.840.1.113883.3.464.1004.2224')
                AND measurement_year = 2026)
          AND pc.start_date <= DATE '2026-12-31'
    )                                                                   -- [Spec2:EXCL_PALLIATIVE:MAPPED]
    AND NOT (
        d.age_at_anchor >= 66
        AND EXISTS (
            SELECT 1 FROM DEMO_HealthcareSynthea.conditions fc
            WHERE TRIM(fc.patient_id) = TRIM(d.patient_id)
              AND fc.code_system = 'SNOMED-CT'
              AND TRIM(fc.code) IN (
                  SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                  WHERE value_set_oid IN (
                      '2.16.840.1.113883.3.464.1004.1531',
                      '2.16.840.1.113883.3.464.1004.1533')
                    AND measurement_year = 2026)
              AND fc.start_date <= DATE '2026-12-31'
        )
        AND EXISTS (
            SELECT 1 FROM DEMO_HealthcareSynthea.conditions ai
            WHERE TRIM(ai.patient_id) = TRIM(d.patient_id)
              AND ai.code_system = 'SNOMED-CT'
              AND TRIM(ai.code) IN (
                  SELECT TRIM(code) FROM HCLS.measure_value_set_codes
                  WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1465'
                    AND measurement_year = 2026)
              AND ai.start_date <= DATE '2026-12-31'
        )
    )                                                                   -- [Spec2:EXCL_FRAILTY:DOMAIN_GAP:PARTIAL]
    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.conditions crc
        WHERE TRIM(crc.patient_id) = TRIM(d.patient_id)
          AND crc.code_system = 'SNOMED-CT'
          AND TRIM(crc.code) IN (
              SELECT TRIM(code) FROM HCLS.measure_value_set_codes
              WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1065'
                AND measurement_year = 2026)
          AND crc.start_date <= DATE '2026-12-31'
    )                                                                   -- [Spec2:EXCL_COLORECTAL_CANCER_HISTORY:MAPPED]
    AND NOT EXISTS (
        SELECT 1 FROM DEMO_HealthcareSynthea.procedures col
        WHERE TRIM(col.patient_id) = TRIM(d.patient_id)
          AND col.code_system = 'SNOMED-CT'
          AND TRIM(col.code) IN (
              SELECT TRIM(code) FROM HCLS.measure_value_set_codes
              WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1250'
                AND measurement_year = 2026)
          AND CAST(col.start_ts AT TIME ZONE 'GMT' AS DATE) <= DATE '2026-12-31'
    )                                                                   -- [Spec2:EXCL_TOTAL_COLECTOMY:MAPPED]
)
,CTE_FOBT_EVENTS AS (
    SELECT DISTINCT TRIM(o.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.observations o
    WHERE TRIM(o.code) IN (
        SELECT TRIM(code) FROM HCLS.measure_value_set_codes
        WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1959' AND measurement_year = 2026)
      AND CAST(o.observation_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2026-01-01' AND DATE '2026-12-31'            -- [Spec2:NUM_FOBT:MAPPED]
    UNION
    SELECT DISTINCT TRIM(c.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.conditions c
    WHERE c.code_system = 'SNOMED-CT'
      AND TRIM(c.code) IN (
        SELECT TRIM(code) FROM HCLS.measure_value_set_codes
        WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1960' AND measurement_year = 2026)
      AND c.start_date BETWEEN DATE '2026-01-01' AND DATE '2026-12-31' -- [Spec2:NUM_FOBT:MAPPED]
)
,CTE_FITDNA_EVENTS AS (
    SELECT DISTINCT TRIM(o.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.observations o
    WHERE TRIM(o.code) IN (
        SELECT TRIM(code) FROM HCLS.measure_value_set_codes
        WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1749' AND measurement_year = 2026)
      AND CAST(o.observation_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2024-01-01' AND DATE '2026-12-31'            -- [Spec2:NUM_FITDNA:MAPPED]
)
,CTE_FLEXSIG_EVENTS AS (
    SELECT DISTINCT TRIM(p.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.procedures p
    WHERE p.code_system = 'SNOMED-CT'
      AND TRIM(p.code) IN (
        SELECT TRIM(code) FROM HCLS.measure_value_set_codes
        WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1102' AND measurement_year = 2026)
      AND CAST(p.start_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2022-01-01' AND DATE '2026-12-31'            -- [Spec2:NUM_FLEXSIG:MAPPED]
)
,CTE_COLONOSCOPY_EVENTS AS (
    SELECT DISTINCT TRIM(p.patient_id) AS patient_id
    FROM DEMO_HealthcareSynthea.procedures p
    WHERE p.code_system = 'SNOMED-CT'
      AND TRIM(p.code) IN (
        SELECT TRIM(code) FROM HCLS.measure_value_set_codes
        WHERE value_set_oid = '2.16.840.1.113883.3.464.1004.1064' AND measurement_year = 2026)
      AND CAST(p.start_ts AT TIME ZONE 'GMT' AS DATE)
            BETWEEN DATE '2017-01-01' AND DATE '2026-12-31'            -- [Spec2:NUM_COLONOSCOPY:MAPPED] [SN-10]
)
,CTE_NUM_EVENTS_ALL AS (
    SELECT patient_id FROM CTE_FOBT_EVENTS
    UNION
    SELECT patient_id FROM CTE_FITDNA_EVENTS
    UNION
    SELECT patient_id FROM CTE_FLEXSIG_EVENTS
    UNION
    SELECT patient_id FROM CTE_COLONOSCOPY_EVENTS
)
,CTE_05_NUMERATOR AS (
    SELECT
        d.*
        ,CASE WHEN n.patient_id IS NOT NULL THEN 1 ELSE 0 END AS in_numerator  -- [Spec2:NUM_COUNT_RULE:MAPPED]
    FROM CTE_04_EXCLUSIONS d
    LEFT JOIN CTE_NUM_EVENTS_ALL n
        ON TRIM(d.patient_id) = TRIM(n.patient_id)                     -- [SN-04]
)
,CTE_06_RESULT AS (
    SELECT
        TRIM(patient_id) AS member_id,
        in_numerator
    FROM CTE_05_NUMERATOR
)
SELECT
    member_id,
    in_numerator
FROM CTE_06_RESULT;
```

## Spine structure

The spine is the certified SQL from the top through `CTE_05_NUMERATOR`. Columns each CTE exposes (never select a column not listed):

- `CTE_01_MEMBERS`(birth_date, city, county, ethnicity, patient_id, race, state, zip)
- `enroll_segments`(patient_id, seg_end, seg_start)
- `gapped`(gap_days, patient_id, seg_end, seg_start)
- `episoded`(episode_id, patient_id, seg_end, seg_start)
- `episode_span`(episode_end, episode_id, episode_start, patient_id)
- `CTE_02_ENROLLED`(patient_id)
- `CTE_03_DENOMINATOR`(age_at_anchor, birth_date, city, county, ethnicity, patient_id, race, state, zip)
- `CTE_04_EXCLUSIONS`(age_at_anchor, birth_date, city, county, ethnicity, patient_id, race, state, zip)
- `CTE_FOBT_EVENTS`(patient_id)
- `CTE_FITDNA_EVENTS`(patient_id)
- `CTE_FLEXSIG_EVENTS`(patient_id)
- `CTE_COLONOSCOPY_EVENTS`(patient_id)
- `CTE_NUM_EVENTS_ALL`(patient_id)
- `CTE_05_NUMERATOR`(age_at_anchor, birth_date, city, county, ethnicity, in_numerator, patient_id, race, state, zip)

Member key: `patient_id`

**Rate model:** `CTE_05_NUMERATOR` is member-grain: one row per DENOMINATOR member (columns: age_at_anchor, birth_date, city, county, ethnicity, in_numerator, patient_id, race, state, zip) with a 0/1 `in_numerator` flag. Read FROM `CTE_05_NUMERATOR`; denominator = COUNT(DISTINCT patient_id); numerator = SUM(in_numerator).

## Writing analytical queries

Assemble: certified spine verbatim, then your tail (`, extra_cte AS (...)` and a final `SELECT`) reading from `CTE_05_NUMERATOR` (or the denominator CTE named in the rate model).

- **The spine is frozen.** Reuse the certified CTEs verbatim through `CTE_05_NUMERATOR`; never modify, rebuild or re-derive eligibility, exclusions, codes or enrollment. Add only a tail (extra CTEs and a final SELECT) that reads from the spine, so every result is a subset or aggregation of the certified population.
- **Follow the rate model above exactly.** It states which CTE is the denominator and how the numerator flag is formed for this measure.
- **Enrichment** (a dimension not already on the spine, such as provider, facility or payer): `LEFT JOIN` on the member key, one row per member. Never `INNER JOIN`, and never filter on the joined table, or the denominator changes.
- **Names:** use only columns listed in the spine structure and tables you have looked up (e.g. with `base_columnDescription`). Never guess a column; fully qualify tables as `db.table`.
- **Never invent numbers.** Certified rate, star band and cut points come from this file. Gap-to-next-star math uses only the cut points above.
- **Questions that redefine the measure** (different age range, codes, enrollment, exclusions) are scenario analysis, not analytics. Say so instead of altering the certified logic.
- **Reconcile:** an overall rate computed from your tail must equal the certified denominator, numerator and rate.
- Validate with `EXPLAIN` before running anything large; use `TOP n` while exploring.

Example questions this supports:
- Rate by age band, gender, race, county or other member attribute on the spine, with the gap to the next star band.
- Open care gaps: denominator members with `in_numerator = 0`, with contact and demographic columns for outreach.
- Members needed to reach the next star band: `CEIL(next_cut/100 * denominator) - numerator`.
- Rate by provider, facility or payer via a `LEFT JOIN` enrichment.

## Dashboards

1. Pick the dimensions (e.g. age band, county, provider) and confirm each is on the spine or can be added by `LEFT JOIN` enrichment.
2. Write one query per tile, all built on the same spine, each returning aggregated rows (not member rows) unless the tile is an outreach list.
3. Include a headline tile with the certified rate and star band taken from this file, and check that the breakdown tiles sum to the certified denominator and numerator.
4. Show cut points as reference lines on rate charts, and flag segments below the next star threshold.
5. Keep outreach lists to the columns needed, and cap rows with `TOP n`.

## Teradata notes

- `CASE WHEN EXISTS (...)` is illegal (error 3771). Use a `LEFT JOIN` and test `IS NOT NULL`.
- `AT TIME ZONE` is valid only on TIMESTAMP columns (`*_ts`): `CAST(col AT TIME ZONE 'GMT' AS DATE)`. DATE columns (`*_date`) compare directly (error 5407).
- `NTILE` is not supported. Use `CASE WHEN` bands or `QUANTILE(n, col)`.
- Wrap rate denominators in `NULLIFZERO()`; `TRIM()` every join key; no `ORDER BY` inside CTEs.
- Do not use reserved words as aliases or output names (`value`, `year`, `month`, `day`, `level`, `rank`, `row`, `type`, `count`, `zone`, `title`, `summary`, `lt`, `gt`, `eq`, `in`, `is`, ...). Use `member_count`, `age_band`, etc.
- Dates in the certified SQL are literals for the measurement year; keep them when reusing the spine.

Reference (configuration) tables used by the certified SQL:
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
