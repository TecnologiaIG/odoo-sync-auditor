---
name: sync-detect-gaps
description: Finds records in System A without a corresponding record in System B (and vice versa). Detects orphaned Pipedrive deals without Odoo leads, Odoo partners without their external mapping, hr.employee without Entra users, etc.
triggers:
  - "detect gaps"
  - "find orphans"
  - "audit missing records"
---

# Sync Detect Gaps

Finds records that exist on one side of the sync but not the other. Helps identify when sync silently failed (an exception was swallowed, a record was filtered out, the external system was down at sync time).

## When to use

- After a sync incident (worker crashed, API was rate-limited)
- Monthly cadence to catch slow accumulation of orphans
- During HR audit (Entra users without hr.employee → "ghost" accounts)
- During CRM cleanup (Pipedrive deals never made it to Odoo → lost revenue tracking)

## Inputs

- `odoo_model`: e.g. `res.partner`, `crm.lead`, `hr.employee`
- `external_system_name`: human label (e.g. "Pipedrive", "Entra ID")
- `external_id_field`: field on Odoo with external PK
- `external_fetch_all_ids_fn`: how to list ALL external PKs (depends on your MCP / API)
- `direction`: `"odoo_missing"` (records in external without Odoo counterpart), `"external_missing"` (records in Odoo without external counterpart), or `"both"` (default)

## Flow

### 1. Snapshot Odoo IDs

```python
odoo_records = mcp__odoo__search_records(
    model=odoo_model,
    domain=[(external_id_field, "!=", False)],
    fields=["id", external_id_field]
)
odoo_external_ids = set(r[external_id_field] for r in odoo_records)
```

### 2. Snapshot external IDs

```python
# Call your external system's "list all IDs" endpoint
external_ids = set(external_fetch_all_ids_fn())
```

### 3. Set operations

```python
in_external_not_odoo = external_ids - odoo_external_ids  # Odoo missing
in_odoo_not_external = odoo_external_ids - external_ids  # External missing
```

### 4. Drill into the gaps

For `in_external_not_odoo`: fetch each external record to understand WHY it didn't sync. Common reasons:
- Created after the last sync window (legitimate, will catch up)
- Marked "deleted" in external but only soft-delete (sync filter excluded it)
- Required field missing (sync skipped with exception)
- Permission error (token scope didn't cover this record)

For `in_odoo_not_external`: fetch each Odoo record. Common reasons:
- Manually created in Odoo (not from sync)
- Source was deleted in external after sync
- Wrong `external_id` (sync wrote wrong PK)

### 5. Output

```json
{
  "odoo_model": "crm.lead",
  "external_system": "Pipedrive",
  "snapshot_time": "2026-05-26T09:00:00-06:00",
  "in_external_not_odoo": [
    {
      "external_id": "deal_45102",
      "external_summary": {
        "title": "Apto 504 Distrito SA - Lead 45102",
        "value": 175000,
        "created_at": "2026-05-25T14:30:00",
        "owner": "Carolina M."
      },
      "likely_cause": "Created 2026-05-25, last sync ran 2026-05-25 12:00. Will catch up next sync.",
      "severity": "info"
    },
    {
      "external_id": "deal_44891",
      "external_summary": { "...": "..." },
      "likely_cause": "Sync error: required field 'partner_id' missing. Found in sync log 2026-05-22.",
      "severity": "warning",
      "recommended_action": "Create partner in Odoo, then re-run sync for deal_44891."
    }
  ],
  "in_odoo_not_external": [
    {
      "odoo_id": 7823,
      "external_id_stored": "deal_99999",
      "likely_cause": "External_id points to a deleted/non-existent Pipedrive deal. Possible typo or manual edit.",
      "severity": "warning",
      "recommended_action": "Verify with sales team. If lead is legitimate, clear external_id field."
    }
  ],
  "summary": {
    "total_in_external_not_odoo": 23,
    "total_in_odoo_not_external": 7,
    "by_severity": {"critical": 0, "warning": 5, "info": 25}
  }
}
```

## Severity rules

- `critical` (🔴): > 50 gaps OR gap blocks accounting (e.g. invoice issued but partner missing)
- `warning` (🟡): 10-50 gaps OR known cause (sync error logged)
- `info` (🟢): < 10 gaps OR cause is sync-lag (will self-resolve)

## Common gap patterns

| Gap | Direction | Likely cause |
|---|---|---|
| `hr.employee` ↔ Entra ID | external_missing | HR onboarding skipped sync step |
| `crm.lead` ↔ Pipedrive deal | odoo_missing | Sync exception (check logs) |
| `res.partner` ↔ external CRM | both | Different match keys (NIT vs email) |
| `sale.order` ↔ source platform | odoo_missing | New record created post-sync window |

## Recommended action template

```
GAP DETECTED: <model> <id> ↔ <external_system>
Direction: <odoo_missing | external_missing>
Severity: <severity>

Context:
  <details from drill-down>

Recommended:
  1. <step 1>
  2. <step 2>

If unsure, escalate to: <team>
```

## Performance notes

- Both snapshots happen in parallel.
- For > 100k records on either side, use pagination + cursor-based iteration.
- Schedule weekly (gaps accumulate slowly; daily is overkill).
