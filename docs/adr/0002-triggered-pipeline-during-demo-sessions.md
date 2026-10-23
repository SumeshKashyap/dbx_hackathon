---
status: accepted
---

# Triggered pipeline on a Simulated Clock, run only inside demo sessions

Ocean Shipments change state over hours, not seconds, so "near real time" means the views are at most a few simulated hours behind, not sub-second. A generator job writes events to a UC Volume and a triggered Lakeflow pipeline processes them about once a minute. Both run only inside a bounded `demo_session` job (the Helsinki pattern), which keeps the project under its ₹5,000 budget. This deliberately deviates from Helsinki's continuous MQTT pipeline: do not "upgrade" it to continuous without re-checking cost.

## Consequences

- If a serverless triggered update takes longer than its interval (startup overhead), the demo session switches the same pipeline to continuous mode for the session only (R3). This is measured in Phase 0.
