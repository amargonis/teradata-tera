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

- **Program:** STARS  |  **Measurement year:** 2025
- **Customer data:** database `DEMO_HealthcareSynthea`; **reference database:** `HCLS`
- **Source tables:** `DEMO_HealthcareSynthea.conditions`, `DEMO_HealthcareSynthea.encounters`, `DEMO_HealthcareSynthea.observations`, `DEMO_HealthcareSynthea.patients`, `DEMO_HealthcareSynthea.payer_transitions`, `DEMO_HealthcareSynthea.payers`, `DEMO_HealthcareSynthea.procedures`, `HCLS.measure_value_set_codes`
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

## Measure definition (Spec 1, approved)

## Spec 1 — Measure Concept Specification
## C09 · Eye Exam to Check for Damage from Diabetes (EED)

---

### 1. Measure Summary

| Field | Value |
|---|---|
| description | Percent of plan members with diabetes who had an eye exam to check for damage from diabetes during the year |
| direction | Higher is better |
| weighting_category | Process Measure |
| weighting_value | 1 |
| statistical_method | Clustering |
| case_mix_adjusted | No |
| display_format | Percentage with no decimal place |
| meaningful_measure | Chronic Conditions |
| primary_data_source | HEDIS |
| data_source_category | Health and Drug Plans |

### 2. Measurement Period

| Concept | Rule |
|---|---|
| measurement_year | The calendar year being measured (e.g. 2023) |
| measurement_period_start | January 1 of measurement_year |
| measurement_period_end | December 31 of measurement_year |
| anchor_date | December 31 of measurement_year (used for age calculation) |
| numerator_lookback_start | January 1 of measurement_year |
| numerator_lookback_end | December 31 of measurement_year |
| numerator_window_note | One calendar year — retinal eye exam must occur during the measurement year itself. Diabetes-diagnosis identification for the denominator (§5) uses a two-year window (measurement year or year prior). Do NOT apply the two-year diabetes-identification window to the numerator eye exam event. |

> **Agent note:** The extracted CMS Technical Notes describe a single-year numerator lookback (eye exam performed "during the measurement year"). The full NCQA HEDIS EED specification (p.203, referenced but not fully excerpted here) contains additional nuance around negative retinopathy findings identified in the prior year satisfying the numerator in certain circumstances. Until that language is confirmed from the full NCQA source, the agent must apply the single-year window stated in the CMS Technical Notes and flag any prior-year eye exam data as gap_flag = SPEC_CLARIFICATION_NEEDED:NUM_LOOKBACK.

### 3. Rate Formula

```
Rate = (Numerator / Denominator) × 100

Reported as: Percentage with no decimal place
```

### 4. Star Rating Cut Points

| Star Band | Rule |
|---|---|
| 1 star | < 57 % |
| 2 stars | >= 57 % to < 70 % |
| 3 stars | >= 70 % to < 77 % |
| 4 stars | >= 77 % to < 83 % |
| 5 stars | >= 83 % |

### 5. Denominator — Conceptual Definition

**Plain language:** Medicare Advantage enrollees aged 18–75 with diabetes (type 1 or type 2) diagnosed during the measurement year or the year prior, who were continuously enrolled during the measurement year.

#### 5.1 Denominator Inclusion Criteria

| Concept ID | Concept Name | Rule | Required |
|---|---|---|---|
| DENOM_AGE | Age at anchor date | Age >= 18 AND age <= 75 as of December 31 of measurement_year | YES |
| DENOM_PLAN_TYPE | Plan type | Medicare Advantage (MA) enrollee | YES |
| DENOM_DIABETES | Diabetes diagnosis | Diagnosis of diabetes (type 1 or type 2), in any setting, during measurement_year OR the year prior to measurement_year | YES |
| DENOM_ENROLL | Continuous enrollment | Continuously enrolled during measurement_year (per HEDIS continuous enrollment allowance) | YES |
| DENOM_ALIVE | Vital status | Member must not have died at any time during measurement_year (see EXCL_DECEASED, §7) | YES |

> **Agent note:** DENOM_DIABETES is identified using the "Diabetes" value set (ICD10CM, SNOMED CT US Edition). Standard HEDIS diabetes-population identification also permits pharmacy-based identification (dispensed insulin or oral hypoglycemic/antihyperglycemic medications); no corresponding NDC-based value set is present in the available MY2025 HEDIS VSD list. If Spec 2 cannot supply pharmacy-based diabetes identification, flag gap_flag = DOMAIN_GAP:DENOM_DIABETES_PHARMACY.

#### 5.2 Denominator Minimum Enrollment Threshold

| Concept ID | Rule |
|---|---|
| DENOM_MIN_500 | Contracts with enrollment < 500 as of the July 2023 enrollment report are excluded from this measure entirely |
| DENOM_RELIABILITY | Contracts with enrollment >= 500 and < 1,000 AND measure score reliability < 0.7 are excluded |

### 6. Numerator — Conceptual Definition

**Plain language:** At least one retinal eye exam performed to check for diabetic damage (retinopathy) during the measurement year.

#### 6.1 Numerator Inclusion Events

| Concept ID | Concept Name | Rule |
|---|---|---|
| NUM_EYE_EXAM | Retinal eye exam event | At least one qualifying retinal eye exam during the numerator lookback window |
| NUM_DATE_WINDOW | Date of service | Service date >= numerator_lookback_start AND <= numerator_lookback_end |
| NUM_COUNT_RULE | Minimum event count | One qualifying event satisfies the numerator. Additional events do not increase the numerator count beyond 1 per member |

#### 6.2 Qualifying Event Code Sets

##### Retinal Eye Exam

> Lookback window: 1 year (measurement year only)

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Retinal Eye Exams | 2.16.840.1.113883.3.464.1004.2550 | CPT, HCPCS, SNOMED CT US Edition |
| Retinal Imaging | 2.16.840.1.113883.3.464.1004.2584 | CPT, SNOMED CT US Edition |
| Autonomous Eye Exam Result or Finding | 2.16.840.1.113883.3.464.1004.2616 | LOINC |

> **Agent note:** "Autonomous Eye Exam Result or Finding" captures results from AI/autonomous retinal imaging devices; treat any recorded result (positive or negative for retinopathy) as a qualifying numerator event, per current NCQA guidance on autonomous exam technology. "Diabetic Retinopathy Severity Level" (LOINC, OID 2.16.840.1.113883.3.464.1004.2555) may be used by Spec 2 as a supporting result value set if the customer data model separates exam procedure from exam finding, but does not independently satisfy the numerator without a corresponding exam event.

### 7. Exclusions — Conceptual Definitions

> Exclusions are applied after the denominator cohort is built and before the numerator is evaluated.
> Order of application: EXCL_DIABETES_EXCEPTION → EXCL_HOSPICE → EXCL_DECEASED → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY_ADVANCED_ILLNESS

#### 7.1 Hospice Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_HOSPICE | Exclude members in hospice or using hospice services any time during measurement_year |
| source_types | Encounter · Claim · Enrollment flag |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Hospice Encounter | 2.16.840.1.113883.3.464.1004.1761 | HCPCS, SNOMED CT US Edition, UBREV |
| Hospice Intervention | 2.16.840.1.113883.3.464.1004.1762 | CPT, HCPCS, SNOMED CT US Edition |

#### 7.2 Deceased Member Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_DECEASED | Exclude members who died at any time during measurement_year |
| source_types | Enrollment/eligibility file — date of death field |

#### 7.3 Palliative Care Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_PALLIATIVE | Exclude members receiving palliative care any time during measurement_year |
| source_types | Encounter/claim with palliative care code · Palliative flag |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Palliative Care Encounter | 2.16.840.1.113883.3.464.1004.1450 | HCPCS, SNOMED CT US Edition |
| Palliative Care Assessment | 2.16.840.1.113883.3.464.1004.2225 | SNOMED CT US Edition |
| Palliative Care Intervention | 2.16.840.1.113883.3.464.1004.2224 | SNOMED CT US Edition |

#### 7.4 Institutional SNP (I-SNP) Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_ISNP | Exclude members age >= 66 enrolled in I-SNP at any time during measurement_year |
| age_gate | >= 66 as of December 31 of measurement_year |
| source | Plan/SNP type indicator in enrollment/eligibility data — no clinical code set applies |

#### 7.5 Long-Term Institutional (LTI) Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_LTI | Exclude members age >= 66 with LTI flag in CMS Monthly Membership Detail Data File any time during measurement_year |
| age_gate | >= 66 as of December 31 of measurement_year |
| source | External CMS file — not derivable from claims |

#### 7.6 Frailty + Advanced Illness Exclusion

> Members age >= 66 as of December 31 of measurement_year must meet BOTH the frailty criterion AND the advanced illness criterion to be excluded.

##### 7.6a Frailty Criterion

| Concept ID | Rule |
|---|---|
| EXCL_FRAILTY | At least two frailty indicators with different dates of service during measurement_year |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Frailty Device | 2.16.840.1.113883.3.464.1004.1530 | HCPCS |
| Frailty Diagnosis | 2.16.840.1.113883.3.464.1004.1531 | ICD10CM, SNOMED CT US Edition |
| Frailty Encounter | 2.16.840.1.113883.3.464.1004.1532 | CPT, HCPCS |
| Frailty Symptom | 2.16.840.1.113883.3.464.1004.1533 | ICD10CM, SNOMED CT US Edition |

##### 7.6b Advanced Illness Criterion

> Any ONE of the following four patterns during measurement_year OR the year prior to measurement_year (count services across both years where noted) satisfies this criterion:

| Concept ID | Pattern |
|---|---|
| EXCL_ADV_ILL_VISITS | At least two qualifying visits (outpatient, observation, ED, telephone, e-visit/virtual check-in, nonacute inpatient encounter, nonacute inpatient discharge) on different dates of service, with an advanced illness diagnosis, counted across measurement_year and the year prior |
| EXCL_ADV_ILL_ACUTE_ENC | At least one acute inpatient encounter with an advanced illness diagnosis |
| EXCL_ADV_ILL_ACUTE_DISCH | At least one acute inpatient discharge with an advanced illness diagnosis on the discharge claim |
| EXCL_ADV_ILL_RX | A dispensed dementia medication during measurement_year or the year prior |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems | Applies To |
|---|---|---|---|
| Advanced Illness | 2.16.840.1.113883.3.464.1004.1465 | ICD10CM, SNOMED CT US Edition | Diagnosis basis for all patterns |
| Outpatient, ED, Telephone, Acute Inpatient and Nonacute Inpatient | 2.16.840.1.113883.3.464.1004.2474 | CPT, HCPCS, SNOMED CT US Edition, UBREV | EXCL_ADV_ILL_VISITS |
| Acute Inpatient | 2.16.840.1.113883.3.464.1004.1810 | CPT, SNOMED CT US Edition | EXCL_ADV_ILL_ACUTE_ENC / EXCL_ADV_ILL_ACUTE_DISCH |
| Acute Inpatient POS | 2.16.840.1.113883.3.464.1004.1027 | POS | EXCL_ADV_ILL_ACUTE_ENC / EXCL_ADV_ILL_ACUTE_DISCH |
| Dementia Medication (Dispensed) | TBD | NDC | EXCL_ADV_ILL_RX |

> **Agent note:** No pharmacy/NDC-based value set for "dispensed dementia medication" is present in the available MY2025 HEDIS VSD list. If Spec 2 cannot supply this concept, omit EXCL_ADV_ILL_RX from evaluation and set gap_flag = DOMAIN_GAP:EXCL_ADV_ILL_RX. This does not invalidate the other three advanced-illness patterns.

#### 7.7 Diabetes Diagnosis Exception (Required)

| Concept ID | Rule |
|---|---|
| EXCL_DIABETES_EXCEPTION | Exclude members who did NOT have a diagnosis of diabetes, in any setting, during measurement_year or the year prior AND who HAD a diagnosis of polycystic ovarian syndrome, gestational diabetes, or steroid-induced diabetes, in any setting, during measurement_year or the year prior |
| source_types | Diagnosis / Condition (any claim setting) |
| history_window | Measurement year and year prior (two calendar years) |

##### Diabetes Diagnosis Exception Code Sets

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Code Systems |
|---|---|---|
| Diabetes | 2.16.840.1.113883.3.464.1004.1077 | ICD10CM, SNOMED CT US Edition |
| Polycystic Ovarian Syndrome | TBD | — |
| Gestational Diabetes | TBD | — |
| Steroid-Induced Diabetes | TBD | — |

> **Agent note:** No dedicated value set for polycystic ovarian syndrome, gestational diabetes, or steroid-induced diabetes is present in the available MY2025 HEDIS VSD list. Flag as gap_flag = DOMAIN_GAP:EXCL_DIABETES_EXCEPTION_CODES until Spec 2 supplies a mapped code set from the full NCQA specification.

### 8. Analytical Query Concepts

| Concept ID | Concept Name | Analytical Use |
|---|---|---|
| AQ_AGE_TERTILE | Age tertile | Divide 18–75 range into thirds: 18–37, 38–56, 57–75 |
| AQ_AGE_BAND_5 | 5-year age band | Standard 5-year bands across 18–75 (e.g. 18–22, 23–27, ... 71–75) |
| AQ_GEOGRAPHY | Geographic segment | Zip, county, state from member address |
| AQ_RACE_ETHNICITY | Race and ethnicity | Health equity sub-analysis |
| AQ_PROVIDER | Attributed provider | PCP or eye care specialist for outreach targeting |
| AQ_TREND | Time trend | Rolling monthly or quarterly rate within certified denom |
| AQ_GAP_TO_CUT | Gap to cut point | Members needed in numerator to reach next star band |
| AQ_OUTREACH | Outreach target list | Denominator members NOT in numerator, by AQ dimension |
| AQ_EXAM_MODALITY | Exam modality | Segment numerator events by exam type (in-person retinal exam, retinal imaging, autonomous/AI exam) |

> **Guardrail:** All analytical queries must operate within the certified
> denominator. The agent must NOT redefine eligibility criteria.
> All analytical output must be labeled result_type = ANALYTICAL.

## Certified SQL

Running this as-is reproduces the certified population: one row per denominator member (`member_id`, `in_numerator`).

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

## Spine structure

The spine is the certified SQL from the top through `CTE_05_NUMERATOR`. Columns each CTE exposes (never select a column not listed):

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

**Rate model:** `CTE_05_NUMERATOR` is ONLY the DISTINCT set of NUMERATOR members (columns: patient_id). It has NO denominator and NO `in_numerator` column. For this measure: read FROM the eligible-denominator CTE `CTE_04_EXCLUSIONS` (aliased d); LEFT JOIN `CTE_05_NUMERATOR` num ON TRIM(d.patient_id) = TRIM(num.patient_id); derive the flag with CASE WHEN num.patient_id IS NOT NULL THEN 1 ELSE 0 END; denominator = COUNT(DISTINCT d.patient_id); numerator = SUM(that flag).

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
