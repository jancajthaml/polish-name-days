# Validation Matrix — Polish Name Days (Realtime)

## Validation matrix

| WP | Validation entry | PRD / ADR covered | Method | Pass criterion |
|----|------------------|-------------------|--------|----------------|
| WP-001 | Firebase project accessible | [ADR-01-01](../../GENESIS_SUPPORT/adr/01-data/adr-01-01-realtime-data-service.md), [ADR-03-01](../../GENESIS_SUPPORT/adr/03-infrastructure/adr-03-01-hosting-strategy.md) | Manual: open Firebase Console, confirm project exists | Project visible; RTDB enabled; no errors |
| WP-001 | Security rules enforced | [ADR-01-02](../../GENESIS_SUPPORT/adr/01-data/adr-01-02-data-schema.md) | Manual: attempt unauthenticated write via REST API | Write rejected (403); read succeeds without auth |
| WP-001 | Spark plan active | [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md) | Manual: check Firebase project settings | Billing shows Spark (free) plan |
| WP-002 | All CSV data in Firebase | [ADR-01-02](../../GENESIS_SUPPORT/adr/01-data/adr-01-02-data-schema.md), [PRD-002](../../GENESIS_SUPPORT/prd/prd-002-data-management.md) | Script or manual: count entries in `/nameDays` | Count ≥ 4700 unique names; total name-date pairs ≥ 6602 |
| WP-002 | Schema correct | [ADR-01-02](../../GENESIS_SUPPORT/adr/01-data/adr-01-02-data-schema.md) | Manual: inspect sample entries in Firebase Console | Each name key has indexed date sub-object; dates match `DD.MM` pattern |
| WP-002 | fallback.json exists and valid | [ADR-02-01](../../GENESIS_SUPPORT/adr/02-frontend/adr-02-01-client-architecture.md) | Script: `JSON.parse` on `fallback.json` | Parse succeeds; structure is `{ name: ["DD.MM", ...], ... }` |
| WP-003 | Firebase onValue fires | [ADR-01-01](../../GENESIS_SUPPORT/adr/01-data/adr-01-01-realtime-data-service.md), [PRD-001](../../GENESIS_SUPPORT/prd/prd-001-realtime-data-propagation.md) | Manual: load page, check network/console | Firebase connection established; initial snapshot received |
| WP-003 | Realtime update propagates | [PRD-001](../../GENESIS_SUPPORT/prd/prd-001-realtime-data-propagation.md) | Manual: add name via Console while page open | New name appears in search results within 60s, no reload |
| WP-003 | Dual-trigger rendering | [ADR-02-01](../../GENESIS_SUPPORT/adr/02-frontend/adr-02-01-client-architecture.md), [ADR-02-02](../../GENESIS_SUPPORT/adr/02-frontend/adr-02-02-search-preservation.md), [PRD-003](../../GENESIS_SUPPORT/prd/prd-003-search-with-dynamic-data.md) | Manual: type partial name, add matching name via Console | New name appears without typing again |
| WP-003 | Polish diacritics preserved | [PRD-003](../../GENESIS_SUPPORT/prd/prd-003-search-with-dynamic-data.md), [ADR-02-02](../../GENESIS_SUPPORT/adr/02-frontend/adr-02-02-search-preservation.md) | Manual: search "Zofia" and "Żofia" | Both return equivalent matches |
| WP-003 | Fallback on timeout | [PRD-004](../../GENESIS_SUPPORT/prd/prd-004-system-resilience.md), [ADR-02-01](../../GENESIS_SUPPORT/adr/02-frontend/adr-02-01-client-architecture.md) | Manual: block Firebase, load page, wait 5s | fallback.json loads; search works on static data |
| WP-003 | Malformed data skipped | [PRD-004](../../GENESIS_SUPPORT/prd/prd-004-system-resilience.md), [ADR-02-01](../../GENESIS_SUPPORT/adr/02-frontend/adr-02-01-client-architecture.md) | Manual: write invalid entry (if rules allow) or simulate | Client does not crash; malformed entry not in search results |
| WP-003 | Search responsiveness | [PRD-003](../../GENESIS_SUPPORT/prd/prd-003-search-with-dynamic-data.md) | Manual: type in search, observe result delay | Results appear within 200ms of keystroke |
| WP-004 | Site loads from GitHub Pages | [ADR-03-01](../../GENESIS_SUPPORT/adr/03-infrastructure/adr-03-01-hosting-strategy.md), [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md) | Manual: open deployed URL | Page loads; no 404 or blank |
| WP-004 | End-to-end PRD satisfaction | [PRD-001](../../GENESIS_SUPPORT/prd/prd-001-realtime-data-propagation.md), [PRD-002](../../GENESIS_SUPPORT/prd/prd-002-data-management.md), [PRD-003](../../GENESIS_SUPPORT/prd/prd-003-search-with-dynamic-data.md), [PRD-004](../../GENESIS_SUPPORT/prd/prd-004-system-resilience.md), [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md) | Manual: run all validation entries above on live site | All pass |

## Integration validation

| Integration point | Producing WP | Consuming WP | Validation method | Pass criterion |
|-------------------|--------------|--------------|-------------------|----------------|
| Firebase config → client | WP-001 | WP-003 | Manual: load page with valid config | Firebase SDK initialises; connection to RTDB established |
| Firebase data schema → client | WP-002 | WP-003 | Manual: after WP-002, load page | Client transforms snapshot to `{ name: [dates] }`; search returns correct dates for known names |
| fallback.json → client | WP-002 | WP-003 | Manual: disable Firebase, load page | Client loads fallback after timeout; search works with fallback data |

## Regression criteria

After each work package completes, the following must still hold:

| After WP | Regression check |
|----------|------------------|
| WP-001 | N/A — no prior behaviour |
| WP-002 | Firebase rules still enforce read/write; CSV file unchanged in repo |
| WP-003 | Static site still loads from GitHub Pages (prior to cutover); CSV-based version still works if reverted |
| WP-004 | All previous WP validations still pass on live site |

**Monotonic progress**: Each WP adds capability. Rollback to any prior state (git revert) restores previous behaviour. No WP removes a previously working capability except by design (e.g. CSV fetch replaced by Firebase — rollback restores CSV).

## Completion criteria

The transformation is **complete** when:

1. **PRD coverage**: Every PRD success criterion maps to at least one validation entry that passes.
2. **ADR coverage**: Every ADR decision is exercised by at least one validation entry.
3. **Integration**: All three integration points pass validation.
4. **Regression**: No regression criteria violated.
5. **Operational**: Site is live on GitHub Pages; operator can add names via Firebase Console without code change; cold start deploy achievable within 1 hour per [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md).

### PRD → validation mapping

| PRD | Success criterion | Validation entry |
|-----|-------------------|------------------|
| [PRD-001](../../GENESIS_SUPPORT/prd/prd-001-realtime-data-propagation.md) | New entry reaches clients within 60s, no user action | WP-003: Realtime update propagates |
| [PRD-001](../../GENESIS_SUPPORT/prd/prd-001-realtime-data-propagation.md) | Unified data flow (initial + incremental) | WP-003: Firebase onValue fires (covers both) |
| [PRD-001](../../GENESIS_SUPPORT/prd/prd-001-realtime-data-propagation.md) | Works on all modern browsers | WP-004: End-to-end (manual on target browsers) |
| [PRD-002](../../GENESIS_SUPPORT/prd/prd-002-data-management.md) | Add without code change/redeploy | WP-004: Add via Console, verify propagation |
| [PRD-002](../../GENESIS_SUPPORT/prd/prd-002-data-management.md) | Durable persistence | WP-002: Data in Firebase; WP-004: persist across reload |
| [PRD-002](../../GENESIS_SUPPORT/prd/prd-002-data-management.md) | Schema preserved | WP-002: Schema correct; WP-003: search returns dates |
| [PRD-002](../../GENESIS_SUPPORT/prd/prd-002-data-management.md) | Validate schema, reject malformed | WP-001: Security rules; WP-003: Malformed data skipped |
| [PRD-003](../../GENESIS_SUPPORT/prd/prd-003-search-with-dynamic-data.md) | New name findable without reload | WP-003: Dual-trigger rendering |
| [PRD-003](../../GENESIS_SUPPORT/prd/prd-003-search-with-dynamic-data.md) | Diacritics for new names | WP-003: Polish diacritics preserved |
| [PRD-003](../../GENESIS_SUPPORT/prd/prd-003-search-with-dynamic-data.md) | Top matches, per keystroke | WP-003: Search responsiveness; WP-004: manual |
| [PRD-003](../../GENESIS_SUPPORT/prd/prd-003-search-with-dynamic-data.md) | ≤200ms at 5k and 10k scale | WP-003: Search responsiveness |
| [PRD-004](../../GENESIS_SUPPORT/prd/prd-004-system-resilience.md) | Fallback if source unavailable at load | WP-003: Fallback on timeout |
| [PRD-004](../../GENESIS_SUPPORT/prd/prd-004-system-resilience.md) | Auto-reconnect on drop | WP-003: Firebase SDK handles; manual: disconnect/reconnect |
| [PRD-004](../../GENESIS_SUPPORT/prd/prd-004-system-resilience.md) | Discard malformed entries | WP-003: Malformed data skipped |
| [PRD-004](../../GENESIS_SUPPORT/prd/prd-004-system-resilience.md) | Function on last-known during outage | WP-003: Fallback on timeout |
| [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md) | ≤3 components | WP-004: Component count check |
| [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md) | No specialist expertise | WP-004: Documentation review |
| [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md) | Free-tier cost | WP-001: Spark plan |
| [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md) | 1-hour cold start | WP-004: Deploy verification |
| [PRD-005](../../GENESIS_SUPPORT/prd/prd-005-operational-simplicity.md) | Data decoupled from code deploy | WP-004: Add via Console, no git push for data |
