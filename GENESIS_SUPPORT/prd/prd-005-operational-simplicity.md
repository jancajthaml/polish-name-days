# PRD-005: Operational Simplicity

## Problem / requirement

The current system has zero operational overhead: a static HTML file and a CSV file served by GitHub Pages. There is no server to maintain, no database to back up, no service to monitor, no dependency to update, no CI/CD pipeline to manage. This extreme simplicity is consistent with a solo developer or very small team operator profile.

Introducing realtime capability necessarily adds components. But the total system must remain operatable by the same operator profile. The number of new infrastructure components must be minimised. Each component must justify its existence. The ongoing maintenance burden (updates, monitoring, incident response, cost management) must be proportional to the value delivered — a name-day lookup tool, not a mission-critical enterprise system.

## Success criteria

- The total number of distinct infrastructure components (excluding the frontend itself) must not exceed three.
- No component may require dedicated infrastructure expertise (e.g., database administration, Kubernetes operation, message broker tuning) to operate.
- The ongoing operational cost must remain within free-tier or near-zero-cost boundaries for the expected usage (small public website, low traffic, small dataset).
- A new developer must be able to understand the complete system architecture and deploy it from a cold start within one hour, using only the project documentation and standard tooling.
- The system must not require a custom CI/CD pipeline to deploy data updates (data management must be decoupled from code deployment).

## Out of scope

- High-availability or multi-region deployment (the current system is single-region, single-provider).
- Automated scaling infrastructure.
- Formal SLA commitments.
- Dedicated monitoring or alerting systems.
