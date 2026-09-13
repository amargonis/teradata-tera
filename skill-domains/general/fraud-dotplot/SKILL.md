---
name: anomaly-dotplot
description: Create an interactive anomaly dot plot showing total bill charges vs anomaly score for all alerted providers in a specific cluster. Use this skill whenever the user says "show me the anomaly analysis for cluster X", "anomaly dot plot for cluster X", "plot the anomalies for cluster X", "show anomalies in cluster X", or any request to visualize or explore anomaly scores, bill charges, or contributing features for providers within a specific cluster ID. This skill queries HCLS anomaly_results, alerts, anomaly_result_details, and WORKERS_COMP_HDR, then renders an interactive scatter/dot plot where hovering over each provider dot reveals the top 3 features driving their anomaly score.
---

# Anomaly Dot Plot — by Cluster

Produces an interactive dot plot of alerted providers in a given cluster, with total bill charges on the Y-axis, anomaly score on the X-axis, and hover tooltips showing the top 3 contributing features per provider.

## Trigger

User says something like:
- "Show me the anomaly analysis for cluster 2"
- "Plot the anomalies for cluster 4"
- "Anomaly dot plot for cluster X"
- "Show me the anomaly dot plot for cluster X"

## Extract Cluster ID

Parse the cluster ID integer from the user's message.

---

## Step-by-Step Execution

### 1. Get alerted providers in the cluster with billing totals

```sql
SELECT
  ar.object_id AS provider_npi,
  ar.cluster_id,
  CAST(ar.anomaly_score AS FLOAT) AS anomaly_score,
  SUM(CAST(h.Total_Charge_Per_Bill AS DECIMAL(18,2))) AS total_bill_charges
FROM HCLS.anomaly_results ar
JOIN HCLS.alerts a
  ON ar.object_type = a.object_type
  AND ar.object_id = a.object_id
JOIN HCLS.WORKERS_COMP_HDR h
  ON ar.object_id = h.Rendering_Bill_Provider_National_Provider_ID
WHERE ar.cluster_id = {cluster_id}
GROUP BY ar.object_id, ar.cluster_id, ar.anomaly_score
ORDER BY ar.anomaly_score DESC
```

If no results, inform user no alerted providers exist in that cluster.

### 2. Get top 3 contributing features per provider

```sql
SELECT
  ard.object_id AS provider_npi,
  ard.feature,
  CAST(ard.feature_value AS FLOAT) AS feature_value,
  CAST(ard.feature_score AS FLOAT) AS feature_score,
  ROW_NUMBER() OVER (PARTITION BY ard.object_id ORDER BY CAST(ard.feature_score AS FLOAT) DESC) AS feature_rank
FROM HCLS.anomaly_result_details ard
JOIN HCLS.alerts a
  ON ard.object_type = a.object_type
  AND ard.object_id = a.object_id
WHERE CAST(ard.feature_score AS FLOAT) > 0
  AND ard.object_id IN ({comma_separated_provider_npis_from_step_1})
QUALIFY feature_rank <= 3
ORDER BY ard.object_id, feature_rank
```

### 3. Map feature names to readable labels

Use this pattern to translate feature names into human-readable labels:

| Feature prefix | Label |
|---|---|
| `avg01_..._in_*` | Avg Charge/Claim (Xd In) |
| `avg02_..._in_*` | Avg Paid/Claim (Xd In) |
| `avg01_..._out_*` | Avg Charge Outbound (Xd) |
| `avg02_..._out_*` | Avg Paid/Claim (Xd Out) |
| `sum01_..._in_*` | Total Charged (Xd In) |
| `sum02_..._in_*` | Total Billed (Xd In) |
| `sum01_..._out_*` | Total Charged Outbound (Xd) |
| `sum02_..._out_*` | Total Billed Outbound (Xd) |
| `cnt0*_..._in_*` | Claim Count (Xd In) |
| `cnt0*_..._out_*` | Outbound Claim Count (Xd) |
| `dcnt01_..._in_*` | Distinct Patients (Xd In) |
| `dcnt01_..._out_*` | Distinct Patients Outbound (Xd) |
| `dcnt02_..._in_*` | Distinct Payers (Xd In) |
| `dcnt02_..._out_*` | Distinct Payers Outbound (Xd) |
| `std01_..._in_*` | Charge Volatility (Xd In) |
| `std01_..._out_*` | Charge Volatility (Xd Out) |
| `max01_..._in_*` | Max Single Claim (Xd In) |
| `max01_..._out_*` | Max Claim Outbound (Xd) |

Where `X` = the timeframe number extracted from the feature name (e.g. `_30`, `_90`, `_180`, `_365`).

Format feature_value as currency ($X,XXX) for sum/avg/max features, plain number for cnt/dcnt features.

---

## Visualization

Produce a `show_widget` interactive dot plot using a plain `<canvas>` element (no Chart.js — draw manually for full tooltip control).

### Canvas layout
- Total canvas: 692 × 388px inside a 724px × 420px wrapper
- Padding: top=20, right=30, bottom=55, left=80
- Y-axis: **log scale** of total_bill_charges (use `Math.log1p`)
- X-axis: anomaly_score, range from (min_score - 0.02) to (max_score + 0.02)

### Dot styling
- Color by cluster using the Teradata brand palette:
  - Cluster 0: `#7ED321` (brand green accent)
  - Cluster 1: `#4A90E2` (brand blue accent)
  - Cluster 2: `#FF5F02` (Teradata Orange — highest risk)
  - Cluster 3: `#00233C` (Teradata Navy)
  - Cluster 4: `#D8BFD8` (brand lavender)
  - Other clusters: `#888888`
- Radius: 9px for highest-risk cluster (2), 7px otherwise
- Opacity: 0.85, white stroke 1px

### Axes
- Y-axis ticks: $1K, $10K, $100K, $1M, $10M (formatted)
- X-axis ticks: evenly spaced score values, 1 decimal
- Axis labels: "Anomaly Score →" (bottom), "Total Bill Charges (log) →" (left, rotated)
- Grid lines: `#e0e0e0`, 0.5px
- Axis tick color: `#00233C` (Teradata Navy)

### Hover tooltip
On `mousemove`, detect nearest dot within 14px. Show a fixed-position tooltip containing:
- Provider NPI (in `#FF5F02`)
- Cluster | Score | Total Billed
- Top 3 features with label, value, and a relative horizontal bar (width = score / max_score * 100%)

Highlight the hovered dot with a navy ring (radius+2, `#00233C` stroke 2px). Redraw on `mouseleave` to clear highlight.

### Page styling (Teradata brand)
```css
body { background: #ffffff; }
.wrap { background:#ffffff; padding:20px; max-width:800px; font-family:Inter,sans-serif; }
.header { color:#FF5F02; font-size:20px; font-weight:600; }
.sub { color:#00233C; font-size:13px; }
.chart-wrapper { background:#f5f5f5; border:1px solid #e0e0e0; border-radius:10px; padding:16px; }
.tooltip { background:#ffffff; border:1px solid #FF5F02; border-radius:8px; padding:10px 12px; font-size:11px; color:#00233C; max-width:240px; box-shadow: 0 2px 8px rgba(0,35,60,0.12); }
```

Header text: **"Teradata Anomaly Score Analysis — Cluster {X}"** in `#FF5F02`

---

## Provider Table

Below the dot plot, render an HTML table inside the same `show_widget` listing all alerted providers sorted by anomaly score descending. Include the following columns:

| Column | Description |
|---|---|
| Rank | Row number, 1 = highest score |
| Provider NPI | The NPI identifier |
| Anomaly Score | Formatted to 4 decimal places |
| Total Billed | Formatted as $X,XXX,XXX |
| Top Feature | The #1 contributing feature label (human-readable, from the mapping table) |
| Feature Value | The value of that top feature, formatted appropriately |

### Table styling (Teradata brand)
```css
.tbl { width:100%; border-collapse:collapse; font-size:12px; margin-top:14px; }
.tbl th { background:#00233C; color:#ffffff; text-align:left; padding:7px 8px; }
.tbl td { color:#00233C; padding:5px 8px; border-bottom:1px solid #e0e0e0; }
.tbl tr:hover td { background:#f5f5f5; }
.tbl .npi { color:#00233C; font-family:monospace; font-weight:500; }
.tbl .score { color:#FF5F02; font-weight:600; }
.tbl .billed { color:#3a7d44; }
.tbl .feat { color:#555555; font-size:11px; }
```

Highlight any provider with anomaly_score > 0.4 by adding a left border: `border-left: 3px solid #FF5F02` on its `<tr>`.

---

## Output Format

1. The `show_widget` containing: the interactive dot plot + the provider table below it
2. A brief prose summary (3–5 sentences) calling out:
   - Total number of alerted providers in the cluster
   - The highest-scoring provider (NPI + score + charges)
   - The dominant feature theme(s) driving anomalies (e.g., "outbound billing volatility", "inflated avg paid-per-claim")
   - Any providers combining high score AND high charges (top-right quadrant) as highest priority

---

## Notes

- If the cluster has many providers (>50), limit Step 1 to TOP 50 by anomaly_score DESC to keep the visualization readable.
- If the cluster has zero alerted providers, return a message explaining this and suggest checking the overall alert summary.
- The `QUALIFY` keyword is Teradata-specific — use it for the ROW_NUMBER filter rather than a subquery.
- feature_score represents the anomaly contribution magnitude — higher = more anomalous for that feature.
- The log scale on Y is important — bill charges span several orders of magnitude across providers.
