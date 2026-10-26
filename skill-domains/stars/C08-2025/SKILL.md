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

- **Program:** STARS  |  **Measurement year:** 2025
- **Customer data:** database `DEMO_HealthcareSynthea`; **reference database:** `HCLS`
- **Source tables:** `DEMO_HealthcareSynthea.conditions`, `DEMO_HealthcareSynthea.encounters`, `DEMO_HealthcareSynthea.medications`, `DEMO_HealthcareSynthea.patients`, `DEMO_HealthcareSynthea.payer_transitions`, `DEMO_HealthcareSynthea.payers`, `DEMO_HealthcareSynthea.procedures`, `HCLS.measure_value_set_codes`
- **Exported:** 2026-10-26 from the Star Ratings measure automation app. The SQL below was generated, validated with Teradata `EXPLAIN`, and approved by a human reviewer.

## How to use this skill

Run all SQL with `base_readQuery` (Teradata dialect: `TOP n`, not `LIMIT`).

1. **Report the certified result** from the table below when asked for the rate or star band. Do not recompute it unless asked to re-run; if asked, run the certified SQL as-is and aggregate `in_numerator`.
2. **Answer analytical questions** (breakdowns, filters, gap-to-star, outreach lists) by reusing the certified spine and writing only an analytical tail. See *Writing analytical queries*.
3. **Build dashboards** by following *Dashboards*.
4. If a request would change who is in the measure, stop and explain; this skill analyses the certified population, it does not redefine it.

## Certified result

No certified run is on record for this measure.

### Star cut points

- 1 star: < 27 %
- 2 stars: >= 27 % to < 39 %
- 3 stars: >= 39 % to < 52 %
- 4 stars: >= 52 % to < 71 %
- 5 stars: >= 71 %

## Measure definition (Spec 1, approved)

## Spec 1 — Measure Concept Specification
## C08 · Osteoporosis Management in Women Who Had a Fracture (OMW)

---

### 1. Measure Summary

| Field | Value |
|---|---|
| description | Percent of female plan members who broke a bone and got screening or treatment for osteoporosis within 6 months |
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
| index_event_start_date (IESD) | Date of the qualifying fracture identified during the intake period. Each member has an individual IESD — this is NOT a fixed calendar date |
| intake_period | Per HEDIS OMW specification: the 12-month period from July 1 of the year prior to measurement_year through June 30 of measurement_year (source: measure_reference p.232) — **agent note: confirm exact intake_period boundaries against the current-year NCQA technical specification; not explicitly restated in the extracted CMS Technical Notes** |
| numerator_lookback_start | IESD (member's individual fracture date) |
| numerator_lookback_end | IESD + 180 days (6 months) |
| numerator_window_note | Six-month (180-day) window following each member's individual index fracture event (IESD). This window is member-specific, NOT calendar-year aligned. Do not apply a fixed January 1–December 31 window to the numerator. |

> **Agent note:** This measure uses a floating, member-specific index event (fracture) rather than a fixed measurement-year window for both numerator evaluation and several exclusion lookbacks (see §7). Do not reuse the C01 fixed two-calendar-year pattern.

### 3. Rate Formula

```
Rate = (Numerator / Denominator) × 100

Reported as: Percentage with no decimal place
```

### 4. Star Rating Cut Points

| Star Band | Rule |
|---|---|
| 1 star | < 27 % |
| 2 stars | >= 27 % to < 39 % |
| 3 stars | >= 39 % to < 52 % |
| 4 stars | >= 52 % to < 71 % |
| 5 stars | >= 71 % |

### 5. Denominator — Conceptual Definition

**Plain language:** Female Medicare Advantage enrollees aged 67–85 as of December 31 of the measurement year who suffered a qualifying fracture (index event) during the intake period.

#### 5.1 Denominator Inclusion Criteria

| Concept ID | Concept Name | Rule | Required |
|---|---|---|---|
| DENOM_GENDER | Member sex | Female | YES |
| DENOM_AGE | Age at anchor date | Age >= 67 AND age <= 85 as of December 31 of measurement_year | YES |
| DENOM_PLAN_TYPE | Plan type | Medicare Advantage (MA) enrollee | YES |
| DENOM_ENROLL | Continuous enrollment | Continuous enrollment from the IESD through 180 days following the IESD (no enrollment gap permitted) — per HEDIS OMW specification | YES |
| DENOM_ALIVE | Vital status | Member must not have died prior to end of the 180-day numerator window (see EXCL_DEATH in §7) | YES |
| DENOM_EVENT | Index fracture event | Member must have at least one qualifying fracture diagnosis/event during the intake_period. Fracture code set: **TBD — not present in current MY2025 VSD listing; source references "Fracture Value Set" not found in available value sets** | YES |

#### 5.2 Denominator Minimum Enrollment Threshold

| Concept ID | Rule |
|---|---|
| DENOM_MIN_500 | Contracts with enrollment < 500 as of the July 2023 enrollment report are excluded from this measure entirely |
| DENOM_RELIABILITY | Contracts with enrollment >= 500 and < 1,000 AND measure score reliability < 0.7 are excluded |

### 6. Numerator — Conceptual Definition

**Plain language:** Members in the denominator who received either a bone mineral density (BMD) test or a prescription/dispensed drug to treat osteoporosis within 6 months (180 days) after their qualifying fracture (IESD).

#### 6.1 Numerator Inclusion Events

| Concept ID | Concept Name | Rule |
|---|---|---|
| NUM_BMD_TEST | BMD test event | At least one qualifying bone mineral density test during the 180-day numerator lookback window following IESD |
| NUM_OSTEO_RX | Osteoporosis treatment event | At least one qualifying osteoporosis therapy dispensed/active prescription during the 180-day numerator lookback window following IESD |
| NUM_DATE_WINDOW | Date of service window | Service/dispense date >= IESD AND <= (IESD + 180 days) |
| NUM_COUNT_RULE | Minimum event count | One qualifying event of EITHER type (BMD test OR osteoporosis treatment) satisfies the numerator. Multiple events do not increase the numerator count beyond 1 per member |

#### 6.2 Qualifying Event Code Sets

##### Event Type 1 — Bone Mineral Density (BMD) Test

> Lookback window: 180 days (6 months) following IESD

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|---|---|---|
| TBD | TBD | The source specification references a "Bone Mineral Density Tests Value Set." No matching value set was found in the available MY2025 HEDIS VSD listing. Flag as DOMAIN_GAP:CODE_SYSTEM pending confirmation of correct value set name/OID from NCQA publisher documentation. |

##### Event Type 2 — Osteoporosis Treatment (Medication)

> Lookback window: 180 days (6 months) following IESD

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|---|---|---|
| TBD | TBD | The source specification references an "Osteoporosis Medications Value Set" / "Osteoporosis Medications List." No matching value set was found in the available MY2025 HEDIS VSD listing. Flag as DOMAIN_GAP:CODE_SYSTEM pending confirmation. |

> **Agent note:** Both event types are drawn from the same 180-day post-fracture window. A member satisfying either event type independently qualifies for the numerator — do not require both.

### 7. Exclusions — Conceptual Definitions

> Exclusions are applied after the denominator cohort (including the qualifying fracture/IESD) is identified and before the numerator is evaluated.
> Order of application: EXCL_PRIOR_BMD → EXCL_PRIOR_OSTEO_CLAIM → EXCL_PRIOR_OSTEO_RX → EXCL_HOSPICE → EXCL_DEATH → EXCL_PALLIATIVE → EXCL_ISNP → EXCL_LTI → EXCL_FRAILTY_ADVANCED_67_80 → EXCL_FRAILTY_81PLUS

#### 7.1 Prior BMD Test Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_PRIOR_BMD | Exclude members who had a BMD test during the 730 days (24 months) prior to the IESD |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|---|---|---|
| TBD | TBD | Same "Bone Mineral Density Tests" value set referenced in §6.2 Event Type 1. Not found in current VSD listing — flag as DOMAIN_GAP:CODE_SYSTEM. |

#### 7.2 Prior Osteoporosis Therapy Claim Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_PRIOR_OSTEO_CLAIM | Exclude members who had a claim/encounter for osteoporosis therapy during the 365 days (12 months) prior to the IESD |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|---|---|---|
| TBD | TBD | Same "Osteoporosis Medications" value set referenced in §6.2 Event Type 2. Not found in current VSD listing — flag as DOMAIN_GAP:CODE_SYSTEM. |

#### 7.3 Prior Osteoporosis Prescription Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_PRIOR_OSTEO_RX | Exclude members who received a dispensed prescription or had an active prescription to treat osteoporosis during the 365 days (12 months) prior to the IESD |

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Note |
|---|---|---|
| TBD | TBD | Same "Osteoporosis Medications List" referenced in §6.2 Event Type 2. Not found in current VSD listing — flag as DOMAIN_GAP:CODE_SYSTEM. |

#### 7.4 Hospice Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_HOSPICE | Exclude members in hospice or using hospice services any time during the measurement year |

**VALUE_SET_REFERENCES**
| Value Set Name | OID |
|---|---|
| Hospice Encounter | 2.16.840.1.113883.3.464.1004.1761 |
| Hospice Intervention | 2.16.840.1.113883.3.464.1004.1762 |

#### 7.5 Death Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_DEATH | Exclude members who died any time during the measurement year |
| source | Enrollment/eligibility file — vital status flag |

#### 7.6 Palliative Care Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_PALLIATIVE | Exclude members who received palliative care any time during the intake period through the end of the measurement year |

**VALUE_SET_REFERENCES**
| Value Set Name | OID |
|---|---|
| Palliative Care Assessment | 2.16.840.1.113883.3.464.1004.2225 |
| Palliative Care Encounter | 2.16.840.1.113883.3.464.1004.1450 |
| Palliative Care Intervention | 2.16.840.1.113883.3.464.1004.2224 |

#### 7.7 Institutional SNP (I-SNP) Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_ISNP | Exclude members aged >= 67 as of December 31 of measurement_year enrolled in an Institutional SNP (I-SNP) at any time during the measurement_year |
| age_gate | >= 67 as of anchor_date |
| source | Plan type / SNP type indicator in enrollment or eligibility data |

#### 7.8 Long-Term Institutional (LTI) Exclusion

| Concept ID | Rule |
|---|---|
| EXCL_LTI | Exclude members aged >= 67 as of December 31 of measurement_year living long-term in an institution any time during the measurement_year |
| age_gate | >= 67 as of anchor_date |
| source | External CMS Monthly Membership Detail Data File — LTI flag field — not derivable from claims |

#### 7.9 Frailty + Advanced Illness Exclusion (Age 67–80)

> Members aged 67–80 as of December 31 of measurement_year must meet BOTH the frailty criterion AND the advanced illness criterion to be excluded.

##### 7.9a Frailty Criterion

| Concept ID | Rule |
|---|---|
| EXCL_FRAILTY | At least two frailty indicators with different dates of service during the intake period through the end of the measurement_year |

**VALUE_SET_REFERENCES**
| Value Set Name | OID |
|---|---|
| Frailty Device | 2.16.840.1.113883.3.464.1004.1530 |
| Frailty Diagnosis | 2.16.840.1.113883.3.464.1004.1531 |
| Frailty Encounter | 2.16.840.1.113883.3.464.1004.1532 |
| Frailty Symptom | 2.16.840.1.113883.3.464.1004.1533 |

##### 7.9b Advanced Illness Criterion

| Concept ID | Rule |
|---|---|
| EXCL_ADVANCED_ILLNESS | Any of the following during measurement_year or the year prior to measurement_year: (a) at least two qualifying visits with an advanced illness diagnosis on different dates of service; (b) at least one acute inpatient encounter with an advanced illness diagnosis; (c) at least one acute inpatient discharge with an advanced illness diagnosis on the discharge claim; OR (d) a dispensed dementia medication |

**Qualifying Visit Types (criterion a):** Outpatient, observation, ED, telephone, e-visit/virtual check-in, nonacute inpatient encounter, nonacute inpatient discharge

**VALUE_SET_REFERENCES**
| Value Set Name | OID | Use |
|---|---|---|
| Advanced Illness | 2.16.840.1.113883.3.464.1004.1465 | Advanced illness diagnosis |
| Outpatient, ED, Telephone, Acute Inpatient and Nonacute Inpatient | 2.16.840.1.113883.3.464.1004.2474 | Qualifying visit types (criterion a) |
| Acute Inpatient | 2.16.840.1.113883.3.464.1004.1810 | Acute inpatient encounter (criterion b/c) |
| TBD (Dementia Medication) | TBD | Dispensed dementia medication (criterion d) — not found in current VSD listing; flag as DOMAIN_GAP:CODE_SYSTEM |

#### 7.10 Frailty-Only Exclusion (Age 81+)

| Concept ID | Rule |
|---|---|
| EXCL_FRAILTY_81PLUS | Exclude members aged >= 81 as of December 31 of measurement_year with at least two frailty indicators with different dates of service during the intake period through the end of the measurement_year |
| age_gate | >= 81 as of anchor_date |

**VALUE_SET_REFERENCES**
| Value Set Name | OID |
|---|---|
| Frailty Device | 2.16.840.1.113883.3.464.1004.1530 |
| Frailty Diagnosis | 2.16.840.1.113883.3.464.1004.1531 |
| Frailty Encounter | 2.16.840.1.113883.3.464.1004.1532 |
| Frailty Symptom | 2.16.840.1.113883.3.464.1004.1533 |

### 8. Analytical Query Concepts

| Concept ID | Concept Name | Analytical Use |
|---|---|---|
| AQ_AGE_TERTILE | Age tertile | Divide 67–85 range into thirds: 67–72, 73–79, 80–85 |
| AQ_AGE_BAND_5 | 5-year age band | Standard 5-year bands: 67–71, 72–76, 77–81, 82–85 |
| AQ_GEOGRAPHY | Geographic segment | Zip, county, state from member address |
| AQ_RACE_ETHNICITY | Race and ethnicity | Health equity sub-analysis |
| AQ_PROVIDER | Attributed provider | PCP or last-touch provider for outreach targeting |
| AQ_TREND | Time trend | Rolling monthly or quarterly rate within certified denominator |
| AQ_GAP_TO_CUT | Gap to cut point | Members needed in numerator to reach next star band |
| AQ_OUTREACH | Outreach target list | Denominator members NOT in numerator, by AQ dimension |
| AQ_TREATMENT_TYPE | BMD test vs. medication | Segment numerator-satisfying members by which event type (BMD test / osteoporosis Rx) satisfied the numerator |
| AQ_DAYS_TO_TREATMENT | Time-to-treatment | Distribution of days elapsed between IESD and qualifying numerator event |

> **Guardrail:** All analytical queries must operate within the certified denominator. The agent must NOT redefine eligibility criteria. All analytical output must be labeled result_type = ANALYTICAL.

## Certified SQL

Running this as-is reproduces the certified population: one row per denominator member (`member_id`, `in_numerator`).

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

## Spine structure

The spine is the certified SQL from the top through `CTE_05_NUMERATOR`. Columns each CTE exposes (never select a column not listed):

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
