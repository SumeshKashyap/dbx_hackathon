# Freight Control Tower — notes for Claude

Hackathon app (Databricks × Accenture App Competition, deadline Fri 30 Oct 2026 23:59 CET) for a freight forwarder: disruption impact + reroute agent, and commercial/Lane insights, with RLS/ABAC. Synthetic forwarder data + real GDACS/USGS alerts → Lakeflow pipeline → warehouse (as the user) / Lakebase → Streamlit app.

## Start of every session

1. Read [claude_notes/PROGRESS.md](./claude_notes/PROGRESS.md) first: phase, next step, and the gotchas borrowed from the Helsinki project.
2. Continue from the first unchecked item under "Next step" unless the user asks for something else.
3. At the end of a session, update PROGRESS.md: tick items, add a dated log entry, rewrite "Next step".

## Rules

- **Never use `main` or the Helsinki catalog `my_databricks_workspace`.** The catalog and CLI profile of the new workspace are in PROGRESS.md once known.
- Use the vocabulary in [claude_notes/CONTEXT.md](./claude_notes/CONTEXT.md) for tables, columns and UI labels (*SLA Breach*, *Delay*, *At Risk*, *Stuck* are different things). When a term settles, add it there; it stays a glossary.
- Hard-to-reverse, surprising trade-offs get an ADR in `docs/adr/` (next number: 0006). Don't silently contradict an ADR; supersede it.
- Requirement IDs (FR-x, NFR-x, R-x, Q-x) in [docs/REQUIREMENTS.md](./docs/REQUIREMENTS.md) are stable. Reference them in code comments and commits.
- Respect the cut line (REQUIREMENTS §2): no stretch item before the MUST demo runs end to end twice.
- Customer data is never read as the app service principal and never synced to Lakebase (ADR-0003).
- Every € or day figure comes from a UC function or the model, never from the LLM (ADR-0005).
- Budget ≤ ₹5,000: stop pipelines, the app and demo sessions after testing, and say so.
- Reuse Helsinki code from `../hsl_live_transit/` where PROGRESS.md points to it, but adapt it; don't copy its domain names.
