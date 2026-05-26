---
name: sync-detect-drift
description: Compares values between Odoo records and an external source-of-truth system. Detects price drift on sale.order, status mismatches on crm.lead, owner divergence, etc. Reports with reconciliation recommendations.
triggers:
  - "detect drift"
  - "audit price drift"
  - "compare odoo external"
---

# Sync Detect Drift

Detects when an Odoo record has diverged from the external system that originally fed it. Classic case: external system updates a property price; the original SO in Odoo still has the old price; nobody notices until month-end accounting flags a Q1.2M revenue shortfall.

## When to use

- Monthly cadence (run before close)
- After bulk price updates in the source system
- When users report "the SO total doesn't match the quote"
- During sync system migration (validate new sync covers all fields)

## Inputs

- `odoo_model`: e.g. `sale.order`, `crm.lead`, `res.partner`
- `external_id_field`: field on Odoo record pointing to external system PK (e.g. `x_pipedrive_id`, `x_external_id`)
- `compare_fields`: dict mapping Odoo field → external system field. E.g.:
  ```json
  {
    "amount_total": "deal.value",
    "user_id.name": "deal.owner.name",
    "state": "deal.status_mapped"
  }
  ```
- `tolerance`: dict of field → tolerance (default 0 = exact match). E.g. `{"amount_total": 1.0}` for $1 tolerance.
- `external_fetch_fn`: how to fetch from external (depends on your MCP / API)

## Flow

### 1. Get Odoo records with external_id set

```python
records = mcp__odoo__search_records(
    model=odoo_model,
    domain=[(external_id_field, "!=", False)],
    fields=["id", external_id_field, ...compare_fields.keys()]
)
```

### 2. Fetch external records in batch

For each `external_id`, call the external system (parallelize). Build a dict `{external_id: external_record}`.

### 3. Compare field by field

For each Odoo record:

```python
odoo_value = record.get(odoo_field)
external_value = external_record.get(external_field)
tolerance = tolerances.get(odoo_field, 0)

if abs(odoo_value - external_value) > tolerance:
    drift_detected = True
```

For string fields, use exact match (or normalized comparison if `tolerance="fuzzy"`).

### 4. Cross-check against the opportunity (gotcha)

For `sale.order.amount_total` drift, **always read the linked `crm.lead.name`** before reporting. The opportunity title often documents legitimate reasons for the drift:

- "Transferred to Apt 101 - 2023-10-19" → SO will be canceled, drift is fake
- "Customer canceled — refund Q15k" → drift reflects a refund, not a bug

Skipping this step caused us to report Q1.25M of "drift" that was actually canceled-but-not-deleted opportunities.

### 5. Output

```json
{
  "model": "sale.order",
  "total_compared": 1247,
  "drift_groups": [
    {
      "odoo_id": 5078,
      "external_id": "deal_18234",
      "field": "amount_total",
      "odoo_value": 145000.00,
      "external_value": 152500.00,
      "delta": 7500.00,
      "opportunity_name": "Apto 304 Albura - cierre",
      "severity": "warning",
      "recommended_action": "Verify with sales: did the customer accept the new price? If yes, update SO. If no, update source system."
    }
  ],
  "summary": {
    "total_drift": 23,
    "by_field": {"amount_total": 12, "user_id.name": 8, "state": 3},
    "total_drift_amount": 87420.50
  }
}
```

## Severity rules

- `critical` (🔴): drift > $10k OR delta > 10% of total OR field affects accounting (`amount_total`, `partner_id`)
- `warning` (🟡): drift $1k-$10k OR 10-50% of total
- `info` (🟢): cosmetic drift (name, description, custom text fields)

## Reconciliation strategies

The output is informational — actual reconciliation is human-decided:

- **External wins**: update Odoo to match external. Use when source system is authoritative.
- **Odoo wins**: update external to match Odoo. Use when Odoo is authoritative (e.g. accounting).
- **Both wrong**: investigate further; might be a third system that fed both.
- **Drift is real / intentional**: document in `crm.lead.name` and exclude from future audits.

## Performance

- Batch external fetches (avoid 1-at-a-time HTTP).
- For > 10k records, run in chunks of 1000 with progress logs.
- Cache the comparison result so re-runs only check changed records.

## Excluded automatically

- Records in `state="cancel"`, `state="done"` (or equivalent terminal states)
- Records with `active=False`
- Records updated in the last hour (sync may be mid-flight)
