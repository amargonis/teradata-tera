---
name: category-health-check
description: Run a retail category shelf health check for a product department. Trigger when the user says "run the category health check for", "shelf health for", "inventory check for [department]", or provides a department name like "produce" or "beverages". Queries the retaildbs Teradata database, identifies top-selling products, checks store inventory levels, and produces a stockout risk report with reorder recommendations.
---

# Retail Category Shelf Health Check

This skill analyzes a retail product department's shelf health by identifying the top-selling products, checking current inventory levels across stores, and flagging items at risk of stockout.

## Trigger phrases

- "Run the category health check for produce"
- "Shelf health for beverages"
- "Inventory check for the dairy department"
- "Category analysis for snacks"

## Extract the Department Name

Parse the department name from the user's message. Match it against `retaildbs.departments`.

---

## Step-by-Step Execution

Execute each step sequentially using `base_readQuery`. Use **TOP** to limit rows (not LIMIT — this is Teradata).

---

### Step 1 — Find the department

```sql
SELECT department_id, department
FROM retaildbs.departments
WHERE LOWER(TRIM(department)) LIKE LOWER('%{department_name}%')
```

Save: `department_id`, full `department` name.  
If no row returned → run `SELECT department_id, department FROM retaildbs.departments ORDER BY department` and ask the user to choose.

---

### Step 2 — Get products in this department

```sql
SELECT TOP 30
    p.product_id,
    p.product_name,
    a.aisle_id,
    a.aisle
FROM retaildbs.products p
JOIN retaildbs.aisles a ON p.aisle_id = a.aisle_id
WHERE p.department_id = {department_id}
ORDER BY p.product_name
```

Save: full product list, count as `{product_count}`, product IDs as `{all_product_ids}`.

---

### Step 3 — Identify top-ordered products in this department

```sql
SELECT TOP 10
    p.product_id,
    p.product_name,
    COUNT(op.order_id) AS total_orders,
    SUM(CASE WHEN op.reordered = 1 THEN 1 ELSE 0 END) AS reorder_count,
    CAST(SUM(CASE WHEN op.reordered = 1 THEN 1 ELSE 0 END) AS FLOAT) /
        NULLIF(COUNT(op.order_id), 0) * 100 AS reorder_rate_pct
FROM retaildbs.order_products op
JOIN retaildbs.products p ON op.product_id = p.product_id
WHERE p.department_id = {department_id}
GROUP BY p.product_id, p.product_name
ORDER BY total_orders DESC
```

Save: top 10 products as `{top_products}`, their IDs as `{top_product_ids}` (comma-separated for use in IN clauses).

---

### Step 4 — Check inventory levels for top products

```sql
SELECT
    spi.product_id,
    COUNT(DISTINCT spi.store_id) AS stores_stocking,
    SUM(spi.quantity_on_hand) AS total_stock,
    AVG(spi.quantity_on_hand) AS avg_stock_per_store,
    MIN(spi.quantity_on_hand) AS min_stock
FROM retaildbs.store_product_inventory spi
WHERE spi.product_id IN ({top_product_ids})
GROUP BY spi.product_id
ORDER BY total_stock ASC
```

> **Note:** Column name may be `quantity_on_hand`, `quantity_available`, or `inventory_qty`. If the query fails, run `SELECT TOP 1 * FROM retaildbs.store_product_inventory` to discover the actual column name and retry.

Save: stock metrics per product.

---

### Step 5 — Get pricing for top products

```sql
SELECT product_id, price
FROM retaildbs.products_price
WHERE product_id IN ({top_product_ids})
```

> **Note:** Column may be `price`, `unit_price`, or `retail_price`. Discover with `SELECT TOP 1 * FROM retaildbs.products_price` if needed.

Save: price per product.

---

### Step 6 — Get store count for context

```sql
SELECT COUNT(DISTINCT store_id) AS total_stores
FROM retaildbs.retail_stores
```

Save: `{total_stores}` for use in report context.

---

### Step 7 — Classify stockout risk

Using data from Steps 3 and 4, classify each top product in context (no additional query needed):

| Condition | Status | Action |
|-----------|--------|--------|
| `avg_stock_per_store` ≥ 20 | 🟢 Well-stocked | No action needed |
| `avg_stock_per_store` 5–19 | 🟡 Monitor | Flag for next delivery review |
| `avg_stock_per_store` < 5 | 🔴 Reorder needed | Immediate restock required |
| No inventory record found | ⚪ Not tracked | Verify with warehouse |

---

### Step 8 — Present findings as structured text

Do NOT generate HTML or a dashboard. Output a clean structured text summary using the data collected in Steps 1–7.

Use this format:

---

**CATEGORY HEALTH REPORT — {department}**
Generated: {today's date}

**DEPARTMENT OVERVIEW**
| Field | Value |
|-------|-------|
| Department | {department} |
| Products in Department | {product_count} |
| Top Products Analyzed | 10 |
| Stores in Network | {total_stores} |

**METRICS**
- 🔴 Reorder Needed: {red_count}
- 🟡 Monitor: {amber_count}
- 🟢 Well-Stocked: {green_count}
- ⚪ Not Tracked: {grey_count}

**TOP PRODUCTS BY ORDER VOLUME**
| Rank | Product | Orders | Reorder Rate | Price | Avg Stock/Store | Status |
|------|---------|--------|--------------|-------|-----------------|--------|
| 1 | {product_name} | {total_orders} | {reorder_rate_pct:.1f}% | ${price} | {avg_stock:.0f} | 🔴/🟡/🟢 |
...

**STOCKOUT RISK ALERTS**
For each 🔴 product:
→ {product_name} — avg {avg_stock:.0f} units/store across {stores_stocking} stores · {total_orders} customer orders on record · **Restock immediately**

**WELL-STOCKED PRODUCTS**
For each 🟢 product:
✓ {product_name} — {avg_stock:.0f} avg units/store · no action needed

**RECOMMENDED ACTIONS**
→ Immediate reorder: list 🔴 products
→ Watch list: list 🟡 products
→ {green_count} products require no action

---

### Step 9 — Suggest next steps

End your response with this message to the user:

> **Next Step — Visual Dashboard**
> To generate the full category health dashboard with inventory charts and product rankings, type:
> **"Generate the category dashboard for {department}"**

---

## Reference: retaildbs Table Schema

| Table | Key columns |
|-------|-------------|
| `retaildbs.departments` | `department_id`, `department` |
| `retaildbs.aisles` | `aisle_id`, `aisle` |
| `retaildbs.products` | `product_id`, `product_name`, `aisle_id`, `department_id` |
| `retaildbs.products_price` | `product_id`, `price` |
| `retaildbs.orders` | `order_id`, `user_id`, `order_number`, `order_dow`, `order_hour_of_day`, `days_since_prior_order` |
| `retaildbs.order_products` | `order_id`, `product_id`, `add_to_cart_order`, `reordered` |
| `retaildbs.store_product_inventory` | `store_id`, `product_id`, `quantity_on_hand` |
| `retaildbs.retail_stores` | `store_id`, `store_name` |

## Demo test case

- **Department:** `produce`
- **Trigger:** `Run the category health check for produce`
- **Expected result:**
  - Bananas, organic berries, and leafy greens appear as top sellers
  - Some high-velocity products show low avg_stock_per_store → 🔴
  - Products with strong inventory buffers → 🟢

## Teradata SQL rules

- Use `TOP N` to limit rows — never `LIMIT N`
- Qualify all tables with the schema: `retaildbs.table_name`
- Use `LOWER()` for case-insensitive department name matching
- Use `NULLIF(x, 0)` to avoid division by zero in ratio calculations
- Use `CAST(... AS FLOAT)` before division to avoid integer truncation
