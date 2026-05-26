---
name: sync-detect-duplicates
description: Detects duplicate records in Odoo (res.partner, crm.lead, sale.order) caused by sync issues. Matches by external_id, NIT/tax ID, email, phone, or fuzzy name. Reports with severity and merge recommendations.
triggers:
  - "detect duplicates"
  - "find duplicate partners"
  - "audit duplicates"
---

# Sync Detect Duplicates

Finds duplicate records in Odoo that typically arise when sync from an external system creates a new partner/lead instead of updating an existing one.

## When to use

- After a bulk import (CSV upload, migration script)
- When users complain "the customer shows up twice in CRM"
- Before month-end (duplicates inflate counts and split invoices)
- During audit prep (tax authorities flag duplicate NITs)

## Inputs

- `model`: Odoo model to audit. Common: `res.partner`, `crm.lead`, `sale.order`
- `match_strategy`: ordered list, default `["external_id", "nit", "email", "phone", "fuzzy_name"]`
- `min_similarity`: for fuzzy_name, 0.0-1.0 (default 0.90)
- `scope`: optional Odoo domain filter (e.g. `[("company_id", "=", 4)]` for one company)

## Strategies (in priority order)

### 1. external_id collision

Two records pointing to the same source-system PK:

```python
mcp__odoo__search_records(
    model="res.partner",
    domain=[("external_id", "!=", False)],
    fields=["id", "name", "external_id"]
)
```

Group by `external_id`. Any group with > 1 record is a duplicate (the sync wrote the same source twice).

### 2. NIT / tax ID exact match

```python
mcp__odoo__search_records(
    model="res.partner",
    domain=[("vat", "!=", False)],
    fields=["id", "name", "vat", "country_id"]
)
```

Group by `(country_id, vat)`. Any group with > 1 is a duplicate.

**Gotcha**: country matters because the same VAT in different countries is legitimately different.

### 3. Email exact match (lowercased, trimmed)

```python
mcp__odoo__search_records(
    model="res.partner",
    domain=[("email", "!=", False)],
    fields=["id", "name", "email"]
)
```

Normalize: `email.lower().strip()`. Group by normalized email. Any group with > 1 is a duplicate.

### 4. Phone E.164 match

```python
# Use phonenumbers lib or regex to normalize to E.164
# Group by normalized phone
```

### 5. Fuzzy name + city

For records without strong identifiers, fall back to:

```python
from difflib import SequenceMatcher

def similarity(a, b):
    return SequenceMatcher(None, a.lower(), b.lower()).ratio()
```

Group by city. Within each city, compare all name pairs. Pairs with similarity ≥ `min_similarity` are candidates.

**Warning**: high false-positive rate. Always show to a human before merging.

## Output

```json
{
  "model": "res.partner",
  "total_records": 12453,
  "duplicate_groups": [
    {
      "match_strategy": "nit",
      "key": "GT-1234567-8",
      "records": [
        {"id": 1845, "name": "ACME Corp", "external_id": null, "create_date": "2024-01-15"},
        {"id": 9234, "name": "ACME Corporation", "external_id": "pipedrive_1234", "create_date": "2026-03-22"}
      ],
      "severity": "warning",
      "recommended_action": "merge into id=1845 (older); transfer external_id from 9234"
    }
  ],
  "summary": {
    "by_strategy": {"external_id": 2, "nit": 8, "email": 14, "phone": 3, "fuzzy_name": 21},
    "total_duplicate_groups": 48,
    "estimated_records_to_merge": 102
  }
}
```

## Recommended merge action

Do **NOT** auto-merge. Always present to a human for review. The output includes a recommendation but the actual merge should be:

1. Pick the canonical record (usually oldest, or the one with most invoices/SOs linked)
2. `mcp__odoo__update_record` to move related records (`account.move.partner_id`, `sale.order.partner_id`, etc.) to the canonical
3. Archive the duplicates (`active=False`) instead of deleting (preserves audit trail)

A future skill `/sync-merge-duplicates` could automate steps 2-3 with confirmation.

## Severity rules

- `critical` (🔴): > 50 duplicate groups OR duplicate has open `sale.order` or `account.move`
- `warning` (🟡): 10-50 groups, no financial impact
- `info` (🟢): < 10 groups, all archived or no relations

## Performance notes

- For `res.partner` > 50k records, run strategies sequentially, not in parallel — Odoo's ORM doesn't love concurrent searches.
- Use `scope` to filter to one `company_id` at a time for multi-company tenants.
- Cache results to a local file if you'll review iteratively.
