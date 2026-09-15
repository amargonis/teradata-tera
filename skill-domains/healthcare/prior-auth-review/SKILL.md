---
name: prior-auth-review
description: Run a prior authorization care pathway review for a specific PA number. Trigger when the user says "run the prior auth review for", "PA review for", "care pathway review for", "review prior auth", or provides a PA number like PA-2026-003417. Queries the HCLS Teradata database, checks each CPT code against care_pathways prerequisites, verifies claims history, and produces a color-coded gap analysis visualization.
---

# Prior Authorization Care Pathway Review

This skill reviews a prior authorization request by checking its CPT codes against the `HCLS.care_pathways` table to identify missing prerequisite treatments, then verifies whether those prerequisites appear in the member's claims history.

## Trigger phrases

- "Run the prior auth review for PA-2026-003417"
- "Review PA-2026-003417 against care pathways"
- "Care pathway review for PA XXXX"
- "Check prior auth PA-XXXX"

## Extract the PA Number

Parse the PA number from the user's message — format is `PA-YYYY-NNNNNN`. If the user provides a number without the prefix, prepend `PA-`.

---

## Step-by-Step Execution

Execute each step sequentially using `base_readQuery`. Use **TOP** to limit rows (not LIMIT — this is Teradata).

---

### Step 1 — Load PA header

```sql
SELECT auth_request_id, auth_request_num, member_id, ordering_provider_id,
       requested_dos, auth_status_id, create_ts
FROM HCLS.prior_auth_request
WHERE auth_request_num = '{pa_number}'
```

Save: `auth_request_id`, `member_id`, `ordering_provider_id`, `requested_dos`.  
If no row returned → tell the user the PA was not found and stop.

---

### Step 2 — Load member demographics

```sql
SELECT member_id_num, first_name, last_name, dob, gender
FROM HCLS.member
WHERE member_id = {member_id}
```

Save: `member_id_num` (e.g. `MBR-884201`), full name.

---

### Step 3 — Load ordering provider

```sql
SELECT first_name, last_name, npi, primary_specialty, credential
FROM HCLS.provider
WHERE provider_id = {ordering_provider_id}
```

> **Note:** The column is `primary_specialty` — NOT `specialty`.

Save: provider name, `primary_specialty`.

---

### Step 4 — Load CPT procedure codes on the request

```sql
SELECT a.cpt_code, c.cpt_desc
FROM HCLS.auth_cpt_code a
JOIN HCLS.cpt_codes c ON a.cpt_code = c.cpt_code
WHERE a.auth_request_id = {auth_request_id}
```

Save the full list of CPT codes as `{cpt_list}`.

---

### Step 5 — Load ICD-10 diagnosis codes

```sql
SELECT icd_code, diagnosis_sequence, icd_description_override
FROM HCLS.auth_icd_code
WHERE auth_request_id = {auth_request_id}
ORDER BY diagnosis_sequence
```

---

### Step 6 — Load request detail (indications and treatments)

```sql
SELECT * FROM HCLS.auth_request_detail
WHERE auth_request_id = {auth_request_id}
```

---

### Step 7 — Check care pathways for prerequisites

```sql
SELECT cp.cpt_code, cp.cpt_code_prereq, c.cpt_desc AS prereq_desc
FROM HCLS.care_pathways cp
JOIN HCLS.cpt_codes c ON cp.cpt_code_prereq = c.cpt_code
WHERE cp.cpt_code IN ({comma_separated_cpt_codes})
```

- CPT codes that **appear in results** → have prerequisites that must be verified
- CPT codes that **do NOT appear** → no prerequisites, auto-approved (show as GREEN)

Group prerequisites by `cpt_code` for the visualization.

---

### Step 8 — Check claims history for each prerequisite

For **each unique prerequisite CPT code** from Step 7, run:

```sql
SELECT TOP 5
    h.Bill_ID,
    h.Service_Bill_From_Date,
    d.HCPCS_Line_Procedure_Billed_Code,
    d.HCPCS_Line_Procedure_Paid_Code,
    h.Rendering_Bill_Provider_Last_Name_or_Group,
    h.Rendering_Bill_Provider_First_Name
FROM HCLS.WORKERS_COMP_HDR h
JOIN HCLS.WORKERS_COMP_DTL d ON h.Bill_ID = d.Bill_ID
WHERE h.Patient_Account_Number = '{member_id_num}'
  AND d.HCPCS_Line_Procedure_Billed_Code = '{prereq_cpt_code}'
  AND h.Service_Bill_From_Date < '{requested_dos}'
ORDER BY h.Service_Bill_From_Date DESC
```

**Classification rules:**

| Result | Status | Color |
|--------|--------|-------|
| At least one claim row returned, service date before requested DOS | ✅ GREEN — Satisfied | `#27ae60` |
| No claim found, but treatment documented in `auth_request_detail` (treatment_type_id not null) | ⚠️ AMBER — Partial | `#f39c12` |
| No claim found, no treatment documentation | ⛔ RED — Missing | `#e74c3c` |

> **Known data note:** In this environment `Patient_Account_Number` in `WORKERS_COMP_HDR` does not map to `member_id_num` — the claims query will return zero rows for all members. This is expected. Prerequisites will show RED, which tells the correct demo story: *"provider needs to submit evidence of conservative treatment."*

---

### Step 9 — Check treatment documentation for AMBER classification

```sql
SELECT ard.treatment_type_id, tt.treatment_code, tt.treatment_name,
       pd.duration_code, pd.duration_label
FROM HCLS.auth_request_detail ard
JOIN HCLS.treatment_type tt ON ard.treatment_type_id = tt.treatment_type_id
LEFT JOIN HCLS.prior_treatment_duration pd ON ard.duration_id = pd.duration_id
WHERE ard.auth_request_id = {auth_request_id}
  AND ard.treatment_type_id IS NOT NULL
```

If `treatment_code = 'PHYSICAL_THERAPY'` is documented and the corresponding CPT codes (97001, 97110) are absent from claims → upgrade those prerequisites from RED to AMBER.

---

### Step 10 — Load clinical notes

```sql
SELECT TOP 5 note_type, note_text, prior_test_name, prior_test_result,
       create_ts AS note_date
FROM HCLS.auth_clinical_notes
WHERE auth_request_id = {auth_request_id}
ORDER BY create_ts DESC
```

> **Note:** The date column is `create_ts` aliased as `note_date` — there is no `note_date` column.

---

### Step 11 — Produce the visualization

Generate a complete standalone HTML page. Use only inline `<style>` — no external CSS or JS dependencies. The page must render correctly on its own when saved as an .html file.

---

#### Visual design tokens

```
Page background:  #f4f5f7
Card background:  #ffffff
Card border:      #e5e7eb
Inner row bg:     #fafafa
Text primary:     #1a1d23
Text secondary:   #374151
Muted / labels:   #6b7280 / #9ca3af
Orange accent:    #FF5F02  (top bar rule, CPT code labels only)
Green:            #16a34a  · green-bg #f0fdf4  · green-border #86efac
Amber:            #d97706  · amber-bg #fffbeb  · amber-border #fcd34d
Red:              #dc2626  · red-bg   #fef2f2  · red-border   #fca5a5
```

Use `font-family: 'Segoe UI', system-ui, sans-serif` for body text and `font-family: 'Courier New', monospace` for CPT codes, PA numbers, and metric values. Light background only — do NOT use dark backgrounds.

---

#### Page structure — render these 10 sections in order (max-width 860px centered):

Use `font-family: 'Segoe UI', system-ui, sans-serif` for body and `font-family: 'Courier New', monospace` for codes, PA numbers, and metric values. Light background only — do NOT use dark backgrounds.

**A. Top bar** — white bg, `border-bottom: 3px solid #FF5F02`, padding 11px 16px. Left: "TERADATA" in `#FF5F02` monospace. Right: "AI Studio · Prior Auth Review" in `#6b7280` + generated timestamp in `#9ca3af` monospace.

**B. Header card** — white, `border: 1px solid #e2e5ea`, border-radius 8px, padding 20px 24px.
- Top row: PA number monospace 21px + "Auth Status: Pending Review" sub-label; status pill right-aligned:
  - RED: `bg:#fef2f2 border:#fca5a5 color:#dc2626` "⛔ PEND — Gaps Found"
  - AMBER: `bg:#fffbeb border:#fcd34d color:#d97706` "⚠ PEND — Partial"
  - GREEN: `bg:#f0fdf4 border:#86efac color:#16a34a` "✓ APPROVED"
- 2-column grid: Member (avatar initials + name + member_id_num + gender) · Provider (avatar + Dr. name + credential + primary_specialty + NPI) · Requested DOS + days since submission · Auth Request ID + created date

**C. Metrics strip** — 4-column grid, white tiles, `border: 1px solid #e2e5ea`, border-radius 8px, padding 14px 16px.
- Tile 1: "CPT CODES REQUESTED" / count
- Tile 2: "PREREQUISITES REQUIRED" / count
- Tile 3: "PREREQUISITES SATISFIED" / `{n} / {total}` — RED if 0, GREEN if complete, AMBER if partial
- Tile 4: "CLAIMS FOUND" / count — RED if 0
- Labels 10px uppercase `#9ca3af`. Values 24px monospace bold.

**D. ICD-10 Diagnoses card** — table: Seq (filled circle, orange=primary gray=secondary) · ICD-10 Code (monospace) · Full Description · Type ("Primary" orange badge or "Secondary" gray text).

**E. Care Pathway Journey card** (one per CPT code with prerequisites) — title "CARE PATHWAY JOURNEY — CPT {code}".
- Intro: "The following prerequisite treatments must be documented before authorization can proceed."
- Horizontal node flow separated by `→` arrows (`#d1d5db`):
  - Prerequisite nodes: `border: 2px solid {status-color}`, status bg, border-radius 8px, width 136px. Inside box (top→bottom): step number circle · CPT code monospace bold · short description `#6b7280`. Below box: status text (RED "✗ Not in claims" · GREEN "✓ Claim: {date} · {provider}" · AMBER "⚠ Documented").
  - Goal node: `border: 2px solid #fdba74`, `bg:#fff7ed`, star marker, CPT in `#FF5F02`, "Authorization goal" below.
- Prerequisite Detail table below flow: CPT code · Name · Clinical rationale · Status badge (colored pill).

**F. Claims History card** — title "CLAIMS HISTORY".
- Sub-text: searched member `{member_id_num}` for `{prereq_codes}` before `{requested_dos}`.
- Table: Bill ID · Service Date · CPT Billed · CPT Paid · Rendering Provider · Status.
- If empty: centered "No claims found for this member" with explanation note.

**G. Clinical Notes card** — title "CLINICAL NOTES". One sub-card per note:
- Header: note_type in `#FF5F02` monospace uppercase + note_date right-aligned `#9ca3af`.
- Body: note_text. If prior_test_name not null: add test name + result row.

**H. Auto-Approved section** — title "AUTO-APPROVED — NO PREREQUISITES REQUIRED". Green pills: `border:1px solid #86efac bg:#f0fdf4`, border-radius 6px — CPT code in `#16a34a` monospace + description.

**I. Action Required panel** (only if RED or AMBER) — `border:1px solid #fecaca border-left:3px solid #dc2626`, white bg, border-radius 8px.
- Heading "⚠ Documentation Required — Provider Action Needed" in `#dc2626`.
- Bulleted list (→ red prefix) — one item per RED/AMBER prerequisite with specific submission instructions.
- Denial code chips: 44 (Documentation of conservative treatment failure required) · 0F (Not medically necessary) · 0U (Additional patient information required).

**J. Legend** — `border-top: 1px solid #e2e5ea`, 11px `#9ca3af`. Three colored dots: GREEN · AMBER · RED with descriptions.

---

#### Full example output (PA-2026-003417):

```
[White top bar — orange bottom border]
  TERADATA                    AI Studio · Prior Auth Review    Generated 2026-03-20 09:14

[Header card]
  PA-2026-003417  ·  Auth Status: Pending Review              [⛔ PEND — Gaps Found]
  [RM] Robert Martinez · MBR-884201 · Male
  [SC] Dr. Sarah Chen, MD · Orthopedic Surgery · NPI 1234567890
  DOS: 2026-03-25   Auth Request ID: 1

[Metrics: 4 tiles]
  CPT Codes: 4  |  Prerequisites Required: 2  |  Satisfied: 0/2 (RED)  |  Claims Found: 0 (RED)

[ICD-10 Diagnoses]
  ① S43.431A — Superior glenoid labrum lesion right shoulder [Primary]
  ② M25.511  — Pain in right shoulder

[Care Pathway Journey — CPT 73223]
  [① 97001 PT Evaluation] → [② 97110 Therapeutic Exercises] → [★ 73223 MRI goal]
     ✗ Not in claims           ✗ Not in claims
  Detail: 97001 · PT Evaluation · Baseline required [✗ Missing]
          97110 · Therapeutic Exercises · 6-week course required [✗ Missing]

[Claims History]
  No claims found for MBR-884201 (searched 97001, 97110 before 2026-03-25)

[Clinical Notes]
  PHYSICIAN NOTE  2026-03-19
  Patient presents with right shoulder pain and limited ROM...
  PRIOR TEST RESULT  2026-03-15
  X-ray right shoulder — no acute fracture

[Auto-Approved]
  [✓ 77002 Fluoroscopic Guidance]  [✓ 23350 Injection Shoulder]  [✓ 20610 Arthrocentesis]

[Action Required — red left border]
  → Submit PT evaluation records (97001): dates, therapist notes, functional baseline
  → Submit therapeutic exercise records (97110): frequency, 6-week duration, outcomes
  [44] [0F] [0U]

[Legend: ● GREEN ● AMBER ● RED]
```

---

### Step 12 — Text summary

After the visualization, provide a concise text summary:

- **Overall status**: PENDING / PEND / APPROVED
- **Member**: name, ID
- **Provider**: name, specialty
- **Gaps**: for each RED/AMBER prerequisite — what the provider must submit
- **Applicable denial codes:**
  - `44` — Documentation of conservative treatment failure is required
  - `0F` — Not medically necessary
  - `0U` — Additional patient information required
- **Recommended next action**

---

## Reference: HCLS Table Schema (verified column names)

| Table | Key columns |
|-------|-------------|
| `HCLS.prior_auth_request` | `auth_request_id`, `auth_request_num`, `member_id`, `ordering_provider_id`, `requested_dos`, `auth_status_id` |
| `HCLS.member` | `member_id`, `member_id_num`, `first_name`, `last_name`, `dob`, `gender` |
| `HCLS.provider` | `provider_id`, `npi`, `first_name`, `last_name`, `primary_specialty`, `credential` |
| `HCLS.auth_cpt_code` | `auth_request_id`, `cpt_code` |
| `HCLS.cpt_codes` | `cpt_code`, `cpt_desc` |
| `HCLS.auth_icd_code` | `auth_request_id`, `icd_code`, `diagnosis_sequence`, `icd_description_override` |
| `HCLS.auth_request_detail` | `auth_request_id`, `treatment_type_id`, `duration_id`, `indication_id` |
| `HCLS.treatment_type` | `treatment_type_id`, `treatment_code`, `treatment_name` |
| `HCLS.prior_treatment_duration` | `duration_id`, `duration_code`, `duration_label` |
| `HCLS.care_pathways` | `cpt_code`, `cpt_code_prereq` |
| `HCLS.auth_clinical_notes` | `note_id`, `auth_request_id`, `note_type`, `note_text`, `prior_test_name`, `prior_test_dt`, `prior_test_result`, `create_ts` |
| `HCLS.WORKERS_COMP_HDR` | `Bill_ID`, `Patient_Account_Number`, `Service_Bill_From_Date`, `Rendering_Bill_Provider_Last_Name_or_Group` |
| `HCLS.WORKERS_COMP_DTL` | `Bill_ID`, `HCPCS_Line_Procedure_Billed_Code`, `HCPCS_Line_Procedure_Paid_Code` |
| `HCLS.preauth_codes` | denial/pend reason codes |

## Demo test case

- **PA number:** `PA-2026-003417` (`auth_request_id = 1`)
- **Member:** Robert Martinez, `MBR-884201`
- **Provider:** Dr. Sarah Chen, Orthopedic Surgery
- **Requested DOS:** 2026-03-25
- **CPT codes:** `73223` (shoulder MRI), `77002`, `23350`, `20610`
- **Expected result:**
  - `73223` → requires `97001` (PT Evaluation) and `97110` (Therapeutic Exercises) → both RED
  - `77002`, `23350`, `20610` → no prerequisites → auto-approved GREEN

## Teradata SQL rules

- Use `TOP N` to limit rows, never `LIMIT N`
- Qualify all tables with the schema: `HCLS.table_name`
- Use `EXPLAIN {sql}` to debug slow queries before re-running
- String comparisons are case-sensitive
