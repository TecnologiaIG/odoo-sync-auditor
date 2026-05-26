# odoo-sync-auditor

Audit synchronization health between [Odoo](https://www.odoo.com) (CRM + Sales) and any external system that feeds it — other CRMs, ERPs, real-estate platforms, marketing tools, etc.

Most Odoo integrations are write-once and trust-forever — Pipedrive deal becomes Odoo lead, source platform "stale" sale becomes Odoo SO, and nobody ever checks the bidirectional state again. Months later, you discover Q1.2M of price drift, hundreds of duplicate partners, and CRM leads pointing at deleted opportunities.

This plugin packages the three audit patterns that actually catch problems.

## Skills

| Slash | What it does |
|---|---|
| `/sync-detect-duplicates` | Find duplicate `res.partner` and `crm.lead` records by NIT / email / phone / external_id collisions |
| `/sync-detect-drift` | Compare values between Odoo and the source system (price, status, owner) — flag mismatches |
| `/sync-detect-gaps` | Find records in System A without a counterpart in System B (and vice versa) |

## Use cases

- **Pre-month-end close**: catch drift in `sale.order` prices before they hit accounting.
- **CRM migration**: validate that Pipedrive → Odoo migration is consistent at deals + persons levels.
- **Multi-source consolidation**: audit a real-estate platform feeding both Pipedrive (sales pipeline) and Odoo (operations).
- **Compliance**: report duplicate `res.partner` by NIT for tax authority audits.

## Installation

### As a Claude Code plugin

Add to your `~/.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "odoo-sync-auditor": {
      "source": {
        "source": "github",
        "repo": "TecnologiaIG/odoo-sync-auditor"
      }
    }
  }
}
```

Then in Claude Code:

```
/plugin install odoo-sync-auditor@odoo-sync-auditor
```

### Prerequisites

This plugin **depends on**:

- An MCP server for Odoo (e.g. the official `odoo` Cowork connector, or the open-source [`odoo-mcp`](https://github.com/odoo-mcp)). The skills call `mcp__odoo__search_records` and `mcp__odoo__get_record`.
- An MCP server (or REST endpoint) for your external system. The skills are agnostic — they don't ship integration code for Pipedrive, HubSpot, etc. You provide the read interface.

The skills include hooks for configuring the "external system" side via prompts.

## Pattern: external_id mapping

Every audit assumes you have an `external_id` field (or equivalent) on Odoo records that points back to the source system's primary key. If you don't, the auditor falls back to fuzzy matching on:

1. NIT / tax ID (Latin America)
2. Email (lowercased, trimmed)
3. Phone (E.164 normalized)
4. Name + city (fuzzy ≥ 90% ratio)

This is configurable per-skill.

## Severity classification

Each detected issue is classified:

| Severity | Criteria |
|---|---|
| 🔴 Critical | Drift > $10k OR > 50 affected records OR blocks month-end |
| 🟡 Warning | Drift $1k-$10k OR 10-50 records |
| 🟢 Info | Cosmetic, < 10 records |

You can override the thresholds via skill parameters.

## Origin

Extracted from production usage at a Guatemalan real-estate group syncing 4 systems (a property-management SaaS, Pipedrive, Odoo, Oracle APEX). The patterns are agnostic — no schemas leak through.

## License

MIT. See [LICENSE](LICENSE).

## Contributing

Issues and PRs welcome. New audit patterns are especially appreciated — if you've battled a recurring sync issue and built a check for it, send a PR.
