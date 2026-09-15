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

#### Page structure — render these sections in order:

**A. Top bar**
- Full-width bar, background `#ffffff`, `border-bottom: 3px solid #FF5F02`, padding 12px 24px
- Left: "TERADATA" in `#FF5F02`, bold monospace, letter-spacing .12em, font-size 13px
- Right: "AI Studio · Prior Auth Review" in `#6b7280`, font-size 12px

**B. Header card**
- Background `--surface`, border `1px solid --border`, border-radius 10px, padding 24px, margin 20px auto, max-width 800px
- Top row: PA number in `--orange` bold large (22px), and a status pill on the right:
  - RED pill (`background: --red-bg`, `border: 1px solid --red`, `color: --red`) with text "⛔ PEND" if any RED
  - AMBER pill with "⚠ PEND" if only AMBER
  - GREEN pill with "✓ APPROVED" if all GREEN
- Below that: a 2-column grid of label/value pairs:
  - Member: `{first_name} {last_name}` · `{member_id_num}` · `{gender}`
  - Provider: `Dr. {last_name}, {credential}` · `{primary_specialty}`
  - Requested DOS: `{requested_dos}`
  - ICD-10: comma-separated list of icd_code values
- Member and provider initials shown as colored circular avatars (2-letter initials, `background: #2e3550`, `color: --coral`)

**C. Metrics strip**
- 4 equal tiles in a row (flex, gap 12px), each `background: --surface2`, border-radius 8px, padding 16px 20px, margin 0 auto 20px, max-width 800px
- Tile 1 — "CPT CODES REQUESTED" / big number
- Tile 2 — "PREREQUISITES REQUIRED" / big number
- Tile 3 — "PREREQUISITES SATISFIED" / `{n} of {total}` in GREEN if all satisfied, RED if 0, AMBER otherwise
- Tile 4 — "OVERALL STATUS" / "PEND" in RED or "APPROVED" in GREEN (large, bold)
- Label: `--muted`, 10px, uppercase, letter-spacing .1em. Number: 28px bold, `--text`

**D. Care Pathway Gap Analysis section**
- Section header: `--coral`, bold, 13px uppercase, letter-spacing .1em, with a 2px `--orange` left border, padding-left 12px, margin 0 auto 14px, max-width 800px
- One card per CPT code that HAS prerequisites (`background: --surface`, border-radius 10px, border `1px solid --border`, max-width 800px, margin 0 auto 16px):
  - Card header row: CPT code in `--coral` monospace bold, description in `--text`, and a right-side pill showing prerequisite count (e.g. "2 prerequisites")
  - **Care pathway flow** — render the prerequisite chain as a visual horizontal sequence:
    - Each prerequisite is a node: rounded box, border color = status color, label = CPT code + short description
    - Between nodes: `→` arrow in `--muted`
    - After last prerequisite: `→` then the requested CPT code node in `--orange` border (the goal)
    - Below each prerequisite node: status badge (RED/AMBER/GREEN pill) + evidence text
      - GREEN: "✓ Claim: {date} · {provider}" in `--green`
      - AMBER: "⚠ Documented in request — no claim" in `--amber`
      - RED: "✗ Not found in claims history" in `--red`
  - If the flow wraps on narrow screens, stack nodes vertically with `↓` arrows

**E. Auto-Approved section**
- Section header same style as D
- Horizontal flex row of small green pills, each: CPT code + short description, `border: 1px solid --green`, `background: --green-bg`, `color: --green`, border-radius 20px, padding 6px 14px, font-size 13px

**F. Action Required panel** (only if any RED or AMBER)
- Full-width card, `border-left: 4px solid --red` (or `--amber` if no RED), `background: --surface2`, border-radius 8px, padding 20px 24px, max-width 800px, margin 0 auto 20px
- Heading: "Documentation Required" in `--red` or `--amber`
- Bullet list of what the provider must submit for each RED/AMBER prerequisite
- Applicable denial codes in a small monospace row: `44 · 0F · 0U` each in a grey pill with tooltip-style description below

**G. Legend strip**
- Small horizontal row at bottom of the 800px container
- Three items: `● GREEN — Prerequisite satisfied in claims`, `● AMBER — Documented, no claim`, `● RED — Not found, documentation required`
- Font-size 11px, `--muted`, monospace dots colored accordingly

---

#### Full example structure (for PA-2026-003417):

```
[Teradata orange top bar]

[Header card]
  PA-2026-003417                              [⛔ PEND]
  Member:   Robert Martinez · MBR-884201 · Male
  Provider: Dr. Chen, MD · Orthopedic Surgery
  DOS:      2026-03-25
  ICD-10:   S43.431A, M25.511

[Metrics strip]
  CPT Codes: 4  |  Prerequisites Required: 2  |  Satisfied: 0 of 2 (RED)  |  Status: PEND

[Section: Care Pathway Gap Analysis]
  [CPT 73223 card — MRI Upper Extremity Joint]
    [Flow: 97001 PT Eval] → [97110 Therapeutic Ex] → [73223 MRI ★]
               ✗ RED                    ✗ RED              (goal)

[Section: Auto-Approved Codes]
  [✓ 77002]  [✓ 23350]  [✓ 20610]

[Action Required panel — red border]
  Provider must submit: PT evaluation records (97001), Therapeutic exercise records (97110)
  Denial codes: 44 · 0F · 0U

[Legend]
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
