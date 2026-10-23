---
status: accepted
---

# The scenario engine and the model compute the numbers; the language model only explains and asks for approval

Every € and day figure (Predicted ETA, Delay, SLA Penalty, Reroute Cost, the valid Reroutes per Shipment Stage) comes from deterministic Unity Catalog functions and the SLA-breach model. The agent's language model picks tools, ranks the options they return and writes the explanation and the customer notification. It never calculates or invents a figure itself. A Reroute only takes effect after a Program Manager approves it (recommend → approve → act). This keeps the impact numbers reproducible for the write-up and unit-testable, and borrows Helsinki's split (rules decide, model explains; Helsinki FR-15.3 and ADR-0007).

## Considered Options

- **Fully automatic rerouting under a cost threshold**: more "automation", but not credible for a forwarder's customer commitments.
- **LLM reasons about impact directly**: less code, but the figures can't be reproduced and change from run to run.

## Consequences

- The agent loop runs inside the app on a Foundation Model endpoint (Helsinki ADR-0008), using Claude if the new workspace allows it and `databricks-gpt-oss-120b` otherwise.
