# ADR-01-01: Realtime Data Service — Firebase Realtime Database

## Context

The topology requires a single managed service that combines data storage and realtime push delivery (see [02-topology.md](../../02-topology.md)). The current system stores data in a static CSV file with no write path and no push capability. PRD-001 requires realtime propagation of new entries to all connected clients within 60 seconds. PRD-002 requires a data management path that does not involve code changes or redeployment. PRD-005 requires the service to be operatable by a solo developer within free-tier cost boundaries.

See also: [PRD-001](../../prd/prd-001-realtime-data-propagation.md), [PRD-002](../../prd/prd-002-data-management.md), [PRD-005](../../prd/prd-005-operational-simplicity.md)

## Decision

We choose **Firebase Realtime Database (RTDB)** as the single Realtime Data Service component identified in the topology.

Firebase RTDB provides:
- **Realtime subscriptions**: Clients attach a listener and receive the initial data snapshot followed by incremental patches on every change — matching the topology's unified data channel requirement.
- **Built-in reconnection**: The SDK automatically reconnects after network interruptions and re-synchronises state — addressing FS-3 without custom code.
- **Protocol fallback**: The SDK uses WebSocket as primary transport and falls back to long-polling on browsers/proxies that block WebSocket — addressing FS-8.
- **Web console for data management**: The operator can add, edit, and view data through the Firebase Console without building a custom admin interface — satisfying PRD-002 with zero additional components.
- **REST API**: Data can also be read/written via HTTPS REST, providing an escape hatch for scripted imports or alternative tooling.
- **Free tier (Spark plan)**: 1 GB stored data, 10 GB/month download, 100 simultaneous connections — more than sufficient for ~6600 name-date pairs and a small-audience public site.
- **CDN-hosted SDK**: The client library is loaded via a `<script>` tag from Google's CDN (~40 KB gzipped), preserving the zero-build approach.
- **Security rules**: Read access is set to public (name-day data is not sensitive). Write access is restricted to the authenticated operator.

### Security rules configuration

```json
{
  "rules": {
    "nameDays": {
      ".read": true,
      ".write": "auth != null"
    }
  }
}
```

### Alternatives considered

| Alternative | Why rejected |
|------------|-------------|
| Supabase (PostgreSQL + Realtime) | More powerful but heavier: requires PostgreSQL knowledge, Realtime channel configuration, and understanding of row-level security. Disproportionate for a flat name→dates store. |
| Firebase Cloud Firestore | Higher latency for simple reads, more complex query model (collections/documents vs. JSON tree), more expensive per-operation pricing. RTDB is simpler for this small, flat dataset. |
| Custom WebSocket server | Violates operator profile: requires hosting, maintaining, and monitoring a server. Violates cost constraint: needs rented infrastructure. |
| SSE + custom backend | Same issues as custom WebSocket: requires a hosted server component. |
| PubNub / Pusher | Messaging services without built-in persistence. Would require a separate data store, adding a component and violating the topology's absorption principle. |
| Periodic HTTP polling | Not truly realtime. Would satisfy "eventual" delivery but not the ≤60-second propagation criterion under normal conditions. Acceptable only as a fallback. |

## Failure modes and mitigations

| Failure mode | Mitigation |
|-------------|-----------|
| Firebase RTDB service outage (FS-1) | The client SDK caches the last-received snapshot in memory. Search continues on stale data. A static JSON snapshot can be bundled as a fallback for initial load if RTDB is unreachable (see [ADR-02-01](../02-frontend/adr-02-01-client-architecture.md)). |
| Malformed data written to RTDB (FS-2) | Client-side validation: the listener callback validates each entry (non-empty string name, DD.MM date regex) before merging into the in-memory dataset. Malformed entries are skipped. Firebase Security Rules can enforce schema via `.validate` rules. |
| Realtime connection drops mid-session (FS-3) | Firebase SDK handles automatic reconnection and re-synchronisation natively. No custom reconnection logic needed. |
| Firebase pricing changes or free-tier removal (FS-4) | Escape hatch: the data is a simple JSON object exportable via REST API or Firebase Console. The client code interacts through a thin abstraction (listener callback), making it feasible to replace Firebase with any service that provides a similar subscribe-to-changes API. The data volume (~6600 records) is trivially portable. |
| Firebase SDK blocked by corporate firewall or ad blocker (FS-8 variant) | The SDK falls back to long-polling over HTTPS (port 443), which is rarely blocked. If the entire `firebaseio.com` domain is blocked, the client falls back to the bundled static snapshot. |
| 100 simultaneous connection limit on Spark plan | For a small-audience name-day lookup tool, 100 concurrent connections is adequate. If exceeded, additional clients get connection errors. Mitigation: upgrade to Blaze (pay-as-you-go) plan only if usage actually grows to this level. Do not pre-optimise. |

## Consequences

- The system gains a dependency on Google Firebase. This is a managed service dependency, not a code dependency — the data is portable JSON and the client interaction surface is a single listener.
- The operator must create a Firebase project and configure security rules. This is a one-time setup taking ~15 minutes.
- Data format changes from CSV to JSON (see [ADR-01-02](adr-01-02-data-schema.md)).
- The existing `name-days.csv` data must be migrated to Firebase RTDB as a one-time import.
- The `fetch()` call in the current code is replaced by a Firebase `onValue()` listener.
