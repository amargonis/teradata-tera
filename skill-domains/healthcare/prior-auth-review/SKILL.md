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

Copy the HTML template below exactly. Replace only the `{PLACEHOLDER}` tokens with real values from the data collected in Steps 1–10. Do not regenerate the CSS or structure — substitute values only.

For `{PREREQ_NODES}`: for each prerequisite CPT code, output one node block and one arrow. Use class `red`, `amber`, or `green` based on claims result.
For `{PREREQ_DETAIL_ROWS}`: one `<tr>` per prerequisite with CPT, description, rationale, and status badge.
For `{CLAIMS_ROWS}`: one `<tr>` per claim found, or the empty-state row if none.
For `{NOTE_CARDS}`: one `.note-card` div per clinical note.
For `{AUTO_PILLS}`: one `.auto-pill` span per auto-approved CPT code.
For `{ACTION_ITEMS}`: one `<li>` per RED/AMBER prerequisite describing what provider must submit.
For `{ICD_ROWS}`: one `<tr>` per ICD-10 code.

---

#### HTML template (substitute placeholders only)

```html
<!DOCTYPE html>
<html><head><meta charset="utf-8"><meta name="viewport" content="width=device-width,initial-scale=1">
<title>Prior Auth Review — {PA_NUMBER}</title>
<style>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0}
body{background:#f0f2f5;color:#1a1d23;font-family:'Segoe UI',system-ui,sans-serif;font-size:14px;line-height:1.6;padding-inline:16px}
.page{max-width:860px;margin:0 auto;padding-bottom:48px}
.topbar{background:#fff;border-bottom:3px solid #FF5F02;padding:11px 0;margin-bottom:20px;display:flex;align-items:center;justify-content:space-between}
.topbar-brand{font-family:'Courier New',monospace;font-size:13px;font-weight:500;color:#FF5F02;letter-spacing:.12em}
.topbar-right{display:flex;align-items:center;gap:16px}
.card{background:#fff;border:1px solid #e2e5ea;border-radius:8px;padding:20px 24px;margin-bottom:14px}
.card-title{font-size:11px;letter-spacing:.1em;text-transform:uppercase;color:#9ca3af;font-weight:600;margin-bottom:14px;display:flex;align-items:center;gap:8px}
.card-title::after{content:'';flex:1;height:1px;background:#f0f2f5}
.header-top{display:flex;align-items:flex-start;justify-content:space-between;gap:16px;margin-bottom:18px;padding-bottom:18px;border-bottom:1px solid #f0f2f5}
.pa-num{font-family:'Courier New',monospace;font-size:21px;font-weight:500;color:#1a1d23}
.pa-sub{font-size:12px;color:#9ca3af;margin-top:2px}
.status-pill-red{display:inline-flex;align-items:center;gap:6px;border:1px solid #fca5a5;background:#fef2f2;color:#dc2626;border-radius:6px;padding:7px 16px;font-size:12px;font-weight:600;white-space:nowrap;flex-shrink:0;font-family:'Courier New',monospace}
.status-pill-amber{display:inline-flex;align-items:center;gap:6px;border:1px solid #fcd34d;background:#fffbeb;color:#d97706;border-radius:6px;padding:7px 16px;font-size:12px;font-weight:600;white-space:nowrap;flex-shrink:0;font-family:'Courier New',monospace}
.status-pill-green{display:inline-flex;align-items:center;gap:6px;border:1px solid #86efac;background:#f0fdf4;color:#16a34a;border-radius:6px;padding:7px 16px;font-size:12px;font-weight:600;white-space:nowrap;flex-shrink:0;font-family:'Courier New',monospace}
.header-grid{display:grid;grid-template-columns:1fr 1fr;gap:14px 32px}
.hfield label{font-size:10px;letter-spacing:.1em;text-transform:uppercase;color:#9ca3af;display:block;margin-bottom:3px}
.hfield-val{font-size:13px;color:#374151;font-weight:500}
.hfield-sub{font-size:12px;color:#6b7280}
.avatar-row{display:flex;align-items:center;gap:10px}
.avatar{width:32px;height:32px;border-radius:50%;background:#f0f2f5;color:#FF5F02;font-size:11px;font-weight:700;display:flex;align-items:center;justify-content:center;flex-shrink:0;border:1px solid #e2e5ea}
.metrics{display:grid;grid-template-columns:repeat(4,1fr);gap:10px;margin-bottom:14px}
.tile{background:#fff;border:1px solid #e2e5ea;border-radius:8px;padding:14px 16px}
.tile-label{font-size:10px;letter-spacing:.08em;text-transform:uppercase;color:#9ca3af;margin-bottom:6px}
.tile-val{font-size:24px;font-weight:600;font-family:'Courier New',monospace;color:#1a1d23}
.tile-val.red{color:#dc2626}.tile-val.green{color:#16a34a}.tile-val.amber{color:#d97706}
.tbl{width:100%;border-collapse:collapse}
.tbl th{font-size:10px;letter-spacing:.08em;text-transform:uppercase;color:#9ca3af;font-weight:600;text-align:left;padding:7px 10px;background:#fafbfc;border-bottom:1px solid #e2e5ea}
.tbl td{padding:9px 10px;border-bottom:1px solid #f5f6f8;font-size:13px;color:#374151;vertical-align:middle}
.tbl tr:last-child td{border-bottom:none}
.seq{display:inline-flex;width:20px;height:20px;border-radius:50%;font-size:10px;font-weight:700;align-items:center;justify-content:center}
.seq.primary{background:#FF5F02;color:#fff}.seq.secondary{background:#6b7280;color:#fff}
.primary-tag{font-size:10px;background:#fff7ed;border:1px solid #fed7aa;color:#ea580c;border-radius:4px;padding:1px 6px;margin-left:6px;font-weight:500}
.mono{font-family:'Courier New',monospace;font-size:12px}
.journey{display:flex;align-items:flex-start;gap:0;overflow-x:auto;padding-bottom:4px}
.journey-step{display:flex;flex-direction:column;align-items:center;min-width:148px}
.step-node{border:2px solid;border-radius:8px;padding:12px 12px 10px;text-align:center;width:136px;display:flex;flex-direction:column;align-items:center;gap:4px}
.step-node.red{border-color:#fca5a5;background:#fef9f9}
.step-node.amber{border-color:#fcd34d;background:#fffbeb}
.step-node.green{border-color:#86efac;background:#f0fdf4}
.step-node.goal{border-color:#fdba74;background:#fff7ed}
.step-num{width:20px;height:20px;border-radius:50%;font-size:10px;font-weight:700;line-height:20px;text-align:center;flex-shrink:0}
.step-node.red .step-num{background:#fca5a5;color:#dc2626}
.step-node.amber .step-num{background:#fcd34d;color:#d97706}
.step-node.green .step-num{background:#86efac;color:#16a34a}
.step-node.goal .step-num{background:#fdba74;color:#ea580c}
.step-cpt{font-family:'Courier New',monospace;font-size:13px;font-weight:500}
.step-node.red .step-cpt{color:#dc2626}.step-node.amber .step-cpt{color:#d97706}
.step-node.green .step-cpt{color:#16a34a}.step-node.goal .step-cpt{color:#FF5F02}
.step-desc{font-size:11px;color:#6b7280;line-height:1.3}
.step-status{font-size:11px;font-weight:600;margin-top:8px;text-align:center}
.step-status.red{color:#dc2626}.step-status.green{color:#16a34a}
.step-status.amber{color:#d97706}.step-status.goal{color:#9ca3af;font-weight:400}
.journey-arrow{font-size:18px;color:#d1d5db;align-self:center;padding:0 4px;margin-top:-18px;flex-shrink:0}
.prereq-detail{margin-top:16px;padding-top:14px;border-top:1px solid #f0f2f5}
.badge{display:inline-flex;align-items:center;gap:4px;border-radius:6px;padding:3px 9px;font-size:11px;font-weight:600;border:1px solid;font-family:'Courier New',monospace}
.badge.red{background:#fef2f2;border-color:#fca5a5;color:#dc2626}
.badge.green{background:#f0fdf4;border-color:#86efac;color:#16a34a}
.badge.amber{background:#fffbeb;border-color:#fcd34d;color:#d97706}
.empty-cell{text-align:center;padding:20px!important;color:#9ca3af}
.note-card{border:1px solid #e2e5ea;border-radius:6px;padding:14px 16px;margin-bottom:10px}
.note-card:last-child{margin-bottom:0}
.note-hdr{display:flex;align-items:center;margin-bottom:8px}
.note-type{font-family:'Courier New',monospace;font-size:11px;font-weight:500;color:#FF5F02;text-transform:uppercase;letter-spacing:.06em}
.note-date{font-size:11px;color:#9ca3af;margin-left:auto}
.note-text{font-size:13px;color:#374151;line-height:1.6}
.note-test{margin-top:8px;padding-top:8px;border-top:1px solid #f0f2f5;display:flex;gap:16px}
.note-test-item label{font-size:10px;text-transform:uppercase;letter-spacing:.08em;color:#9ca3af;display:block;margin-bottom:2px}
.note-test-item span{font-size:12px;color:#374151;font-weight:500}
.auto-row{display:flex;flex-wrap:wrap;gap:8px}
.auto-pill{display:inline-flex;align-items:center;gap:6px;border:1px solid #86efac;background:#f0fdf4;border-radius:6px;padding:7px 12px;font-size:13px}
.auto-code{font-family:'Courier New',monospace;font-weight:500;color:#16a34a;font-size:12px}
.action-panel{border:1px solid #fecaca;border-left:3px solid #dc2626;background:#fff;border-radius:8px;padding:18px 22px;margin-bottom:14px}
.action-heading{font-size:13px;font-weight:600;color:#dc2626;margin-bottom:12px}
.action-list{list-style:none;display:flex;flex-direction:column;gap:8px}
.action-list li{display:flex;align-items:flex-start;gap:8px;font-size:13px;color:#374151}
.action-list li::before{content:'→';color:#dc2626;flex-shrink:0;font-weight:700;margin-top:1px}
.denial-row{display:flex;gap:8px;flex-wrap:wrap;margin-top:14px;padding-top:14px;border-top:1px solid #fee2e2}
.denial-chip{border:1px solid #e2e5ea;border-radius:6px;padding:8px 14px;background:#fafbfc}
.denial-code{font-family:'Courier New',monospace;font-size:15px;font-weight:600;color:#FF5F02}
.denial-desc{font-size:11px;color:#6b7280;margin-top:2px;line-height:1.4;max-width:180px}
.legend{display:flex;flex-wrap:wrap;gap:16px;padding-top:14px;border-top:1px solid #e2e5ea}
.legend-item{display:flex;align-items:center;gap:6px;font-size:11px;color:#9ca3af}
.dot{width:8px;height:8px;border-radius:50%;flex-shrink:0}
@media(max-width:600px){.metrics{grid-template-columns:1fr 1fr}.header-grid{grid-template-columns:1fr}.journey{flex-direction:column;align-items:center}.journey-arrow{transform:rotate(90deg)}}
</style></head><body>

<div style="background:#fff;border-bottom:3px solid #FF5F02;padding:11px 16px;margin-bottom:0">
  <div style="max-width:860px;margin:0 auto;display:flex;align-items:center;justify-content:space-between">
    <span style="font-family:'Courier New',monospace;font-size:13px;font-weight:500;color:#FF5F02;letter-spacing:.12em">TERADATA</span>
    <div style="display:flex;align-items:center;gap:16px">
      <span style="font-size:12px;color:#6b7280">AI Studio · Prior Auth Review</span>
      <span style="font-family:'Courier New',monospace;font-size:11px;color:#9ca3af">Generated {GENERATED_DATE}</span>
    </div>
  </div>
</div>

<div class="page" style="padding-top:20px">

  <div class="card">
    <div class="header-top">
      <div>
        <div class="pa-num">{PA_NUMBER}</div>
        <div class="pa-sub">Auth Status: {AUTH_STATUS} · Submitted {SUBMITTED_DATE}</div>
      </div>
      {STATUS_PILL}
    </div>
    <div class="header-grid">
      <div class="hfield"><label>Member</label>
        <div class="avatar-row"><div class="avatar">{MEMBER_INITIALS}</div>
        <div><div class="hfield-val">{MEMBER_NAME}</div><div class="hfield-sub">{MEMBER_ID} · {GENDER}</div></div></div>
      </div>
      <div class="hfield"><label>Ordering Provider</label>
        <div class="avatar-row"><div class="avatar">{PROVIDER_INITIALS}</div>
        <div><div class="hfield-val">{PROVIDER_NAME}</div><div class="hfield-sub">{PRIMARY_SPECIALTY} · NPI {NPI}</div></div></div>
      </div>
      <div class="hfield"><label>Requested Date of Service</label>
        <div class="hfield-val">{REQUESTED_DOS}</div></div>
      <div class="hfield"><label>Auth Request ID</label>
        <div class="hfield-val mono">{AUTH_REQUEST_ID}</div>
        <div class="hfield-sub">Created {CREATED_DATE}</div></div>
    </div>
  </div>

  <div class="metrics">
    <div class="tile"><div class="tile-label">CPT Codes Requested</div><div class="tile-val">{CPT_COUNT}</div></div>
    <div class="tile"><div class="tile-label">Prerequisites Required</div><div class="tile-val">{PREREQ_COUNT}</div></div>
    <div class="tile"><div class="tile-label">Prerequisites Satisfied</div><div class="tile-val {SATISFIED_COLOR}">{SATISFIED_RATIO}</div></div>
    <div class="tile"><div class="tile-label">Claims Found</div><div class="tile-val {CLAIMS_COLOR}">{CLAIMS_COUNT}</div></div>
  </div>

  <div class="card">
    <div class="card-title">ICD-10 Diagnoses</div>
    <table class="tbl">
      <thead><tr><th>Seq</th><th>ICD-10 Code</th><th>Description</th><th>Type</th></tr></thead>
      <tbody>{ICD_ROWS}</tbody>
    </table>
  </div>

  <div class="card">
    <div class="card-title">Care Pathway Journey — CPT {JOURNEY_CPT_CODE}</div>
    <div style="font-size:12px;color:#6b7280;margin-bottom:16px">The following prerequisite treatments must be documented before authorization can proceed.</div>
    <div class="journey">{PREREQ_NODES}</div>
    <div class="prereq-detail">
      <div style="font-size:11px;letter-spacing:.08em;text-transform:uppercase;color:#9ca3af;font-weight:600;margin-bottom:10px">Prerequisite Detail</div>
      <table class="tbl">
        <thead><tr><th>CPT</th><th>Name</th><th>Clinical Rationale</th><th>Status</th></tr></thead>
        <tbody>{PREREQ_DETAIL_ROWS}</tbody>
      </table>
    </div>
  </div>

  <div class="card">
    <div class="card-title">Claims History</div>
    <div style="font-size:12px;color:#6b7280;margin-bottom:12px">Searched WORKERS_COMP_HDR/DTL for member <span class="mono">{MEMBER_ID}</span> with service date before <span class="mono">{REQUESTED_DOS}</span>. Prerequisites searched: {SEARCHED_CODES}</div>
    <table class="tbl">
      <thead><tr><th>Bill ID</th><th>Service Date</th><th>CPT Billed</th><th>CPT Paid</th><th>Rendering Provider</th></tr></thead>
      <tbody>{CLAIMS_ROWS}</tbody>
    </table>
  </div>

  <div class="card">
    <div class="card-title">Clinical Notes</div>
    {NOTE_CARDS}
  </div>

  <div class="card">
    <div class="card-title">Auto-Approved — No Prerequisites Required</div>
    <div class="auto-row">{AUTO_PILLS}</div>
  </div>

  <div class="action-panel">
    <div class="action-heading">⚠ Documentation Required — Provider Action Needed</div>
    <ul class="action-list">{ACTION_ITEMS}</ul>
    <div class="denial-row">
      <div class="denial-chip"><div class="denial-code">44</div><div class="denial-desc">Documentation of conservative treatment failure required</div></div>
      <div class="denial-chip"><div class="denial-code">0F</div><div class="denial-desc">Not medically necessary per submitted information</div></div>
      <div class="denial-chip"><div class="denial-code">0U</div><div class="denial-desc">Additional patient information required — please resubmit</div></div>
    </div>
  </div>

  <div class="legend">
    <div class="legend-item"><div class="dot" style="background:#16a34a"></div>GREEN — Prerequisite satisfied in claims history</div>
    <div class="legend-item"><div class="dot" style="background:#d97706"></div>AMBER — Documented in request, no claim found</div>
    <div class="legend-item"><div class="dot" style="background:#dc2626"></div>RED — Not found; provider must submit documentation</div>
  </div>

</div></body></html>
```

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
