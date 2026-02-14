# PRD-002: Data Management

## Problem / requirement

The current system stores name-day data as a static CSV file committed to a git repository. Adding new names requires editing the file, committing, pushing, and waiting for GitHub Pages to redeploy. There is no runtime data management capability.

For the realtime experience to work, there must be a way to add new name-day entries to the data source such that the addition triggers realtime propagation to connected clients. The data management path must accept new name-day entries (a Polish name and one or more associated dates) and persist them durably so they survive system restarts and are available to all future clients.

The user stated "if there are new polish names" — implying an ongoing ability to introduce new names into the system, not a one-time migration.

## Success criteria

- A new name-day entry (name + date) can be added to the system's data source without requiring a code change, git commit, or redeployment.
- Added entries must be durably persisted — they must survive restarts of any system component and be available to all clients who connect after the addition.
- The data source must preserve the semantic contract: each entry is a Polish name associated with one or more dates in DD.MM format.
- The data management mechanism must validate that entries conform to the expected schema (non-empty name, valid DD.MM date) and reject malformed entries.

## Out of scope

- A full-featured admin UI for data management (the mechanism may be an API, a console, a direct database write, or any other path — the form factor is a technology decision).
- User authentication or role-based access control for data management (no compliance requirements exist; access control is not stated as a requirement).
- Bulk import/export capabilities.
- Editing or deleting existing name-day entries.
