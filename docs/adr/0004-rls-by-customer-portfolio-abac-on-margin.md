---
status: accepted
---

# Row filters follow the Customer Portfolio; cost and margin are masked by attribute

A Program Manager sees the Shipments of the Customers in their Customer Portfolio, and Leadership sees every row. A row filter function looks up the viewer (`current_user()` / group membership) in a portfolio mapping table, so reassigning a Customer is a data change, not a code change. Carrier cost and Gross Margin columns carry a governed tag (`commercially_sensitive`), and an ABAC policy masks every tagged column for anyone outside the Finance/Leadership group. This means Program Managers see revenue but not margin, and the policy follows the tag onto every new table. Region-based and Division-based filtering were rejected because they split one Customer across several Program Managers.

## Consequences

- Pipelines, training jobs and the Genie space owner must be able to read all rows. Training runs as a principal outside the filtered groups (or as the owner), never as a demo user.
- If ABAC policies are not available in the workspace, the fallback is a plain UC column mask function on the same columns (R2). The demo story stays the same.
