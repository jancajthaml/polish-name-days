# PRD-004: System Resilience

## Problem / requirement

The current system has a simple failure model: if the CSV fetch fails on page load, the application shows nothing. There is no error handling, no retry, no fallback. This is tolerable for a static site — the data either loads or it does not.

Introducing realtime infrastructure adds new failure modes: the realtime data source may become unavailable, the client's connection may drop and need reconnection, malformed data may be pushed, and the realtime provider itself may experience outages. The system must handle these failures gracefully — it must never become worse than the current static experience. A user who opens the page must always see name-day data (even if stale), and a user whose realtime connection drops must recover automatically.

## Success criteria

- If the realtime data source is temporarily unavailable at page load, the application must still display name-day data from a fallback source (e.g., last-known dataset, static snapshot) rather than showing a blank page.
- If the client's realtime connection drops mid-session, it must automatically reconnect and reconcile its state (receive any updates missed during disconnection) without user intervention.
- Malformed data entries received via the realtime channel must be silently discarded by the client without corrupting the in-memory dataset or breaking the search index.
- The system must function (search and display) using its last-known dataset even during prolonged realtime source outages.

## Out of scope

- Guaranteed exactly-once delivery semantics (at-least-once with client-side deduplication is sufficient for name-day data).
- Monitoring, alerting, or observability infrastructure (the operator profile does not support these).
- Automated failover between multiple realtime providers.
