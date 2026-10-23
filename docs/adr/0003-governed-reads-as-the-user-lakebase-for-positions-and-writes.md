---
status: accepted
---

# Customer data is read as the viewer through the SQL warehouse; Lakebase holds only positions and the app's writes

Row filters (by Customer Portfolio) and ABAC column masks live in Unity Catalog and only take effect when the query runs as the person looking. So every read of Shipment, Customer or commercial data goes through the SQL warehouse with Databricks Apps on-behalf-of-user authorization, and so do Genie and the agent's tools, which means a Program Manager's agent cannot see another portfolio. Lakebase synced tables do not carry UC row filters or masks, so only vessel and flight positions (no customer data, comparable to public AIS) are synced to Lakebase for the live map, and Lakebase also stores what the app writes (Reroute Decisions, Injected Disruptions, notification drafts), owned by the app's service principal.

## Considered Options

- **Shipments in Lakebase with Postgres row-level security**: a faster map, but it duplicates the security model in Postgres roles for every user, and the two models would drift apart.
- **Everything as the app service principal, filtered in app code**: the simplest option, but then the app enforces security instead of the platform, which defeats the RLS/ABAC demo.
- **Warehouse only**: loses Lakebase, which the competition FAQ explicitly rewards.

## Consequences

- The map shows ships without customer data. Shipment pins and counts on the map come from the warehouse (as the user) and are refreshed less often than positions.
- Reroute Decisions are written by the service principal, with `approved_by` taken from the viewer's identity header.
- On-behalf-of-user auth and its scopes (`sql`, Genie) must be available in the new workspace; this is checked in Phase 0 (R1).
