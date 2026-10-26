---
name: stars-c01-2025
description: >-
  STARS measure C01 (Breast Cancer Screening), measurement year 2025. Trigger when the user asks about
  C01 or "Breast Cancer Screening", e.g. "what is our C01 rate", "C01 star rating", "show C01 care gaps",
  "who is missing C01", "break C01 down by age/provider/region", or to build a C01 dashboard.
  Uses the approved, EXPLAIN-validated certified Teradata SQL as the single source of truth for the
  denominator, numerator, rate and star band, and writes analytical queries on top of it.
---

# C01 — Breast Cancer Screening (2025)

Percent of female Medicare Advantage enrollees who had a mammogram to screen for breast cancer in the past two years

- **Program:** STARS  |  **Measurement year:** 2025
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

## Measure definition (Spec 1, approved)

## Spec 1 — Measure Concept Specification
## C01 · Breast Cancer Screening (BCS)

---

### 1. Measure Summary

| Field | Value |
|---|---|
| description | Percent of female Medicare Advantage enrollees who had a mammogram to screen for breast cancer in the past two years |
| direction | Higher is better |
| weighting_category | Process Measure |
| weighting_value | 1 |
| statistical_method | Clustering |
| case_mix_adjusted | No |
| display_format | Percentage with no decimal place |
| meaningful_measure | Wellness and Prevention |
| primary_data_source | HEDIS |
| data_source_category | Health and Drug Plans |

> **Agent note (data discrepancy):** The extracted `label_for_stars`/`description` field states "aged 52-74," while the extracted `metric` field (the authoritative rate definition) states "50 to 74 years of age." This spec uses the metric definition (age 50–74) as authoritative, consistent with published HEDIS BCS age criteria. Flag this discrepancy if source documents are re-extracted in future measurement years.

### 2. Measurement Period

| Concept | Rule |
|---|---|
| measurement_year | The calendar year being measured (e.g. 2023) |
| measurement_period_start | January 1 of measurement_year |
| measurement_period_end | December 31 of measurement_year |
| anchor_date | December 31 of measurement_year (used for age calculation) |
| numerator_lookback_start | January 1 of (measurement_year − 1) |
| numerator_lookback_end | December 31 of measurement_year |
| numerator_window_note | Two full calendar years: prior year and measurement year ("past two years" per metric definition) |

### 3. Rate Formula

```
Rate = (Numerator / Denominator) × 100

Reported as: percentage with no decimal place
```

### 4. Star Rating Cut Points

| Star Band | Rule |
|---|---|
| 1 star | < 53 % |
| 2 stars | >= 53 % to < 67 % |
| 3 stars | >= 67 % to < 75 % |
| 4 stars | >= 75 % to < 82 % |
| 5 stars | >= 82 % |

> Source: CMS Star Ratings Technical Notes, 2025 (measurement year 2023 data). Cut points change annually — always pull from the correct year's Technical Notes.

### 5. Denominator — Conceptual Definition

**Plain language:** Female Medicare Advantage enrollees aged 50 to 74 as of December 31 of the measurement year who were continuously enrolled and alive through the measurement period.

#### 5.1 Denominator Inclusion Criteria

| Concept ID | Concept Name | Rule | Required |
|---|---|---|---|
| DENOM_GENDER | Member sex | Female | YES |
| DENOM_AGE | Age at anchor date | Age >= 50 AND age <= 74 as of December 31 of measurement_year | YES |
| DENOM_PLAN_TYPE | Plan type | Medicare Advantage (MA) enrollee — per reporting_requirements table (1876 Cost, CCP w/o SNP, CCP with SNP, CCP with Only I-SNP, MSA, PFFS). Excludes standalone PDP | YES |
| DENOM_ENROLL | Continuous enrollment | Enrolled for the full measurement year (January 1 – December 31) | YES |
| DENOM_ALIVE | Vital status | Member must not have died prior to December 31 of measurement_year | YES |

#### 5.2 Denominator Minimum Enrollment Threshold

| Concept ID | Rule |
|---|---|
| DENOM_MIN_500 | Contracts whose enrollment was less than 500 as of the July measurement_year enrollment report are excluded from this measure entirely |
| DENOM_RELIABILITY | Contracts whose enrollment was >= 500 and < 1,000 as of the July report AND measure score reliability < 0.7 are excluded |

> **Agent note:** These thresholds apply at the contract (plan) level, not the member level. If the customer is a single-contract payer with < 500 members in the eligible cohort, flag result as BELOW_THRESHOLD and suppress the star rating.

### 6. Numerator — Conceptual Definition

**Plain language:** At least one mammogram to screen for breast cancer during the past two years (January 1 of the year prior to the measurement year through December 31 of the measurement year).

#### 6.1 Numerator Inclusion Events

| Concept ID | Concept Name | Rule |
|---|---|---|
| NUM_MAMMOGRAM | Mammogram event | At least one qualifying mammogram service during the numerator lookback window |
| NUM_DATE_WINDOW | Date of service window | Service date >= numerator_lookback_start AND <= numerator_lookback_end |
| NUM_COUNT_RULE | Minimum event count | One qualifying event satisfies the numerator. Multiple events do not increase the numerator count beyond 1 per member |

#### 6.2 Qualifying Event Code Sets

##### Mammogram

> Lookback window: two calendar years (prior year + measurement year)

**VALUE_SET_REFERENCES**

| Value Set Name | OID | Code Systems |
|---|---|---|
| Mammography | 2.16.840.1.113883.3.464.1004.1168 | CPT, LOINC, SNOMED CT US Edition |

> **Agent note:** Both screening and diagnostic mammograms are captured within the "Mammography" value set and both satisfy the numerator event requirement per HEDIS BCS specifications. Do not restrict to screening-only codes.

### 7. Exclusions — Conceptual Definitions

> Exclusions are applied after the denominator cohort is built and before the numerator is evaluated. Each exclusion removes members from the denominator entirely.
> Order of application: EXCL_HOSPICE → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY → EXCL_MASTECTOMY

#### 7.1 Hospice Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_HOSPICE | Exclude members in hospice or using hospice services any time during the measurement_period |
| source_types | Encounter · Claim · Enrollment flag |

**VALUE_SET_REFERENCES**

| Value Set Name | OID | Code Systems |
|---|---|---|
| Hospice Encounter | 2.16.840.1.113883.3.464.1004.1761 | HCPCS, SNOMED CT US Edition, UBREV |
| Hospice Intervention | 2.16.840.1.113883.3.464.1004.1762 | CPT, HCPCS, SNOMED CT US Edition |

#### 7.2 Palliative Care Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_PALLIATIVE | Exclude members receiving palliative care any time during the measurement_period |
| source_types | Encounter/claim with palliative care code · Palliative flag |

**VALUE_SET_REFERENCES**

| Value Set Name | OID | Code Systems |
|---|---|---|
| Palliative Care Encounter | 2.16.840.1.113883.3.464.1004.1450 | HCPCS, SNOMED CT US Edition |
| Palliative Care Assessment | 2.16.840.1.113883.3.464.1004.2225 | SNOMED CT US Edition |
| Palliative Care Intervention | 2.16.840.1.113883.3.464.1004.2224 | SNOMED CT US Edition |

#### 7.3 Institutional SNP (I-SNP) Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_ISNP | Exclude members age >= 66 as of December 31 of measurement_year who were enrolled in an Institutional SNP (I-SNP) at any time during the measurement_year |
| age_gate | >= 66 as of anchor_date |
| source | Plan type / SNP type indicator in enrollment or eligibility data — not a HEDIS clinical value set |

#### 7.4 Long-Term Institutional (LTI) Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_LTI | Exclude members age >= 66 as of December 31 of measurement_year with an LTI flag in the CMS Monthly Membership Detail Data File at any time during the measurement_year. Use the run date of the file to determine LTI status. |
| age_gate | >= 66 as of anchor_date |
| source | External CMS Monthly Membership Detail Data File — not derivable from claims |

#### 7.5 Frailty + Advanced Illness Exclusion

> Members age >= 66 as of December 31 of measurement_year must meet BOTH the frailty criterion AND the advanced illness criterion during the measurement_year to be excluded.

##### 7.5a Frailty Criterion

| Concept ID | Rule |
|---|---|
| EXCL_FRAILTY | At least two indications of frailty with different dates of service during the measurement_period |

**VALUE_SET_REFERENCES**

| Value Set Name | OID | Code Systems |
|---|---|---|
| Frailty Device | 2.16.840.1.113883.3.464.1004.1530 | HCPCS |
| Frailty Diagnosis | 2.16.840.1.113883.3.464.1004.1531 | ICD10CM, SNOMED CT US Edition |
| Frailty Encounter | 2.16.840.1.113883.3.464.1004.1532 | CPT, HCPCS |
| Frailty Symptom | 2.16.840.1.113883.3.464.1004.1533 | ICD10CM, SNOMED CT US Edition |

##### 7.5b Advanced Illness Criterion

| Concept ID | Rule |
|---|---|
| EXCL_ADVANCED_ILLNESS | At least two qualifying visits with an advanced illness diagnosis, on different dates of service, during the measurement_year. Visit type need not be the same for both visits. |

**Qualifying Visit Types:** Outpatient visit · Observation visit · Emergency department visit · Telephone visit · e-visit/virtual check-in · Nonacute inpatient encounter · Nonacute inpatient discharge

**VALUE_SET_REFERENCES**

| Value Set Name | OID | Code Systems |
|---|---|---|
| Advanced Illness | 2.16.840.1.113883.3.464.1004.1465 | ICD10CM, SNOMED CT US Edition |

#### 7.6 Bilateral Mastectomy Exclusion

> Exclude members with evidence of bilateral mastectomy (or both right and left unilateral mastectomies) at any time in the member's history through December 31 of the measurement_year. Any one of the following patterns satisfies the exclusion:

| Concept ID | Pattern |
|---|---|
| EXCL_MAST_BILAT | Bilateral mastectomy procedure code (any date in history) |
| EXCL_MAST_MOD_B | Unilateral mastectomy with a bilateral modifier (same procedure) |
| EXCL_MAST_2_BILAT | Two unilateral mastectomy procedures each found with a bilateral modifier (same procedure) |
| EXCL_MAST_RL | Unilateral mastectomy with right-side modifier + unilateral mastectomy with left-side modifier (same or different dates) |
| EXCL_MAST_ABSENT | Absence of left breast + absence of right breast (same or different dates) |
| EXCL_MAST_L_R | Left unilateral mastectomy (any date) + right unilateral mastectomy (any date) |
| EXCL_MAST_HIST | History of bilateral mastectomy diagnosis code |

**VALUE_SET_REFERENCES**

| Value Set Name | OID | Code Systems |
|---|---|---|
| Bilateral Mastectomy | 2.16.840.1.113883.3.464.1004.1042 | ICD10PCS, SNOMED CT US Edition |
| Unilateral Mastectomy | 2.16.840.1.113883.3.464.1004.1256 | CPT |
| Unilateral Mastectomy Left | 2.16.840.1.113883.3.464.1004.1334 | ICD10PCS, SNOMED CT US Edition |
| Unilateral Mastectomy Right | 2.16.840.1.113883.3.464.1004.1335 | ICD10PCS, SNOMED CT US Edition |
| Clinical Unilateral Mastectomy | 2.16.840.1.113883.3.464.1004.1948 | SNOMED CT US Edition |
| Absence of Left Breast | 2.16.840.1.113883.3.464.1004.1329 | ICD10CM, SNOMED CT US Edition |
| Absence of Right Breast | 2.16.840.1.113883.3.464.1004.1330 | ICD10CM, SNOMED CT US Edition |
| History of Bilateral Mastectomy | 2.16.840.1.113883.3.464.1004.1331 | ICD10CM, SNOMED CT US Edition |

> **Agent note:** CPT modifiers LT (left), RT (right), and 50 (bilateral) applied to Unilateral Mastectomy procedure codes are required to distinguish laterality patterns (EXCL_MAST_MOD_B, EXCL_MAST_2_BILAT, EXCL_MAST_RL). If Spec 2 does not map claim modifiers, flag as MISSING and record gap_flag = MISSING:CLAIM_MODIFIER.

### 8. Analytical Query Concepts

| Concept ID | Concept Name | Analytical Use |
|---|---|---|
| AQ_AGE_TERTILE | Age tertile | Divide 50–74 range into thirds: 50–57, 58–65, 66–74 |
| AQ_AGE_BAND_5 | 5-year age band | Standard 5-year bands: 50–54, 55–59, 60–64, 65–69, 70–74 |
| AQ_GEOGRAPHY | Geographic segment | Zip code, county, state — from member address |
| AQ_RACE_ETHNICITY | Race and ethnicity | For health equity analysis |
| AQ_PROVIDER | Attributed provider | PCP or attributed provider for outreach targeting |
| AQ_TREND | Time trend | Rolling monthly or quarterly rate within the certified denominator |
| AQ_GAP_TO_CUT | Gap to cut point | Members needed in numerator to reach next star band |
| AQ_OUTREACH | Outreach target list | Denominator members NOT in numerator, segmented by AQ dimension |
| AQ_TEST_TYPE | Screening modality | Not applicable for this measure — single modality (Mammography). Retained for schema parity with COL-type measures. |

> **Guardrail:** All analytical queries must operate on the certified denominator population. The analytical agent MUST NOT redefine eligibility criteria (age bounds, sex, enrollment rules, exclusions). All analytical results must be labeled result_type = ANALYTICAL, never CERTIFIED.

## Certified SQL

Running this as-is reproduces the certified population: one row per denominator member (`member_id`, `in_numerator`).

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

## Spine structure

The spine is the certified SQL from the top through `CTE_05_NUMERATOR`. Columns each CTE exposes (never select a column not listed):

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

**Rate model:** `CTE_05_NUMERATOR` is member-grain: one row per DENOMINATOR member (columns: in_numerator, patient_id) with a 0/1 `in_numerator` flag. Read FROM `CTE_05_NUMERATOR`; denominator = COUNT(DISTINCT patient_id); numerator = SUM(in_numerator).

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
