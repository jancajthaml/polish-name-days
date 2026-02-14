# PRD-001: Realtime Data Propagation

## Problem / requirement

The current system loads name-day data once on page load and never updates it. If new Polish names are added to the data source, no connected client sees them until the page is manually reloaded and — in the current architecture — the static file is redeployed.

The system must deliver new name-day data to all connected clients in realtime. When a new name-day entry is added to the data source, every open browser session must receive and incorporate that entry without requiring a page reload, browser refresh, or application redeployment. "Realtime" means the update must reach connected clients within seconds to low minutes of the data being committed to the source.

This is the core architectural intent stated by the user: "make the user experience dynamic so if there are new polish names the name days will update realtime."

## Success criteria

- When a new name-day entry is added to the data source, all currently connected clients must display that entry in search results within 60 seconds, without any user action (no reload, no click, no navigation).
- The realtime delivery mechanism must support the initial full-dataset load as well as incremental updates, providing a unified data flow rather than two separate channels.
- The propagation mechanism must work across all modern browsers on both mobile and desktop devices.

## Out of scope

- Defining which specific realtime protocol or technology to use (that is a technology decision, not a requirement).
- Offline support or data caching for disconnected clients.
- Delivery of name-day deletions or modifications (the user's intent is about additions of new names; edit/delete semantics are not stated).
