# Progress & resume guide

> **Resume here.** Read this file top to bottom, then do the first unchecked item under [Next step](#next-step).
> Update it at the end of every working session: tick items, add a dated log entry, rewrite "Next step".

**Last updated:** 2026-10-23 · **Current phase:** Planning done, waiting for your review and the new workspace · **Deadline:** Fri 30 Oct 2026 23:59 CET (Sat 31 Oct 04:29 IST)

---

## Where things live

| File | What it is |
|---|---|
| [../docs/REQUIREMENTS.md](../docs/REQUIREMENTS.md) | Problem, personas, impact method, goals, cut line, FR/NFR, data, architecture, 7-day plan, video storyboard, risks, open questions |
| [CONTEXT.md](./CONTEXT.md) | Glossary: the words to use in tables, columns and UI (e.g. *SLA Breach*, never "late shipment") |
| [../docs/adr/](../docs/adr/) | 0001 real disruptions + synthetic forwarder · 0002 triggered pipeline in demo sessions · 0003 reads as the user, Lakebase for positions + writes · 0004 RLS by Customer Portfolio + ABAC on margin · 0005 engine computes, model explains |
| [../docs/competition-brief.pdf](../docs/competition-brief.pdf) | The competition brief and FAQ (local only, not in git) |
| Reference project | `../../hsl_live_transit/` (Helsinki). Code to borrow: `src/app/{db,warehouse,genie,agent,store,init_db}.py`, `tests/test_app_smoke.py`, `resources/{app,lakebase,warehouse,genie,demo}.yml`, `src/jobs/demo.py`, `scripts/grant_app_access.py` |

### Databricks workspace (fill in on Day 1)

| Item | Value |
|---|---|
| Workspace host | _tbd (Q2)_ |
| CLI profile | _tbd_ |
| Catalog | _tbd_ (never `main`, never `my_databricks_workspace`) |
| Groups | `fct_leadership`, `fct_finance`, `fct_pm_<portfolio>`: _to create_ |
| Demo identities | you + teammate (Q5: who is Leadership, who is a PM) |

---

## Decisions so far (one line each)

- Persona: freight forwarder (ocean + air); two flagships: disruption control tower and commercial/Lane insights.
- Division = forwarder business unit; Segment = Customer industry.
- Data: real GDACS/USGS + synthetic forwarder with fictional names, on a Simulated Clock (ADR-0001).
- Freshness: triggered pipeline about once a minute, only inside demo sessions (ADR-0002).
- Prediction: LightGBM SLA-breach model + deterministic scenario engine + agent (ADR-0005).
- Security: RLS by Customer Portfolio; ABAC masks cost/margin outside Finance/Leadership; identity-driven landing screen (ADR-0004).
- Reads as the user through the warehouse; Lakebase = vessel positions + app writes (ADR-0003).
- Delay ≠ SLA Breach ≠ At Risk ≠ Stuck. Disruption Impact = SLA Penalties + Reroute Cost; Churn Risk Revenue shown separately, never summed.
- Lane Potential = Gross Margin per TEU/kg (by Container Type) + volume trend and forecast.
- Reroutes: other sea route, other port, air, wait, limited by Shipment Stage (In Transit = wait only).
- Disruptions: Chokepoint closure and port strike are Injected (Scenario Injector); natural hazards are Observed (GDACS/USGS).
- Agent: recommend → Program Manager approves → Reroute Decision in Lakebase.
- App: Streamlit + pydeck. Cloud: Azure. Budget ≤ ₹5,000. Evidence: public benchmarks.
- Cut line agreed (REQUIREMENTS §2).

---

## Gotchas borrowed from Helsinki (don't rediscover them)

Numbers in brackets are the Helsinki PROGRESS.md gotcha numbers.

**Bundle and deploy**
1. `mode: development` renames UC schemas to `dev_<user>_<name>`. Use a plain target with `presets.pipelines_development: true` instead. [7]
2. Deploying a continuous pipeline starts it. Stop it after every deploy (relevant if R3 forces continuous mode). [8]
3. App env from a resource is `value_from`, not `valueFrom`; the camelCase key is silently ignored. [18]
4. `bundle run` of the app can't resolve `${resources.postgres_projects.*.name}`; build paths from a variable. [14]
5. App command with a pre-step: `sh -c "python init_db.py; exec streamlit run app.py --server.port $DATABRICKS_APP_PORT --server.address 0.0.0.0"`. Use `$VAR`, never `${VAR:-x}` (the bundle interpolates `${…}`). [23]
6. If a wheel is used: bump `version` on every change and clear `dist/`, because serverless caches wheels by version. [9]

**Apps runtime**
7. The Apps runtime ran **Streamlit 1.38**: no `height=` on `st.pydeck_chart`. Run the smoke test with `--with streamlit==1.38.0` before every deploy. (Re-check the runtime version in the new workspace.) [17]
8. Altair 5 in the runtime: reuse an `alt.Scale` object between layers. [20]
9. `databricks apps logs` fails with a PAT profile (`bad handshake`). Use the app's Logs tab, or an OAuth profile. [15]

**Lakebase**
10. **`default_endpoint_settings` don't apply to the existing `primary` endpoint.** Set `suspend_timeout_duration` to 300 s with `databricks postgres update-endpoint` right after creation and check it. Helsinki billed ~$1.60/day around the clock because of this. [24]
11. App-owned Postgres schemas must be created by the app service principal (`init_db.py` at app start). If you create them yourself, the SP gets `permission denied`. Test SQL on a throwaway branch. [21]
12. Synced tables work in a normal UC catalog; the UC schema name becomes the Postgres schema. [16]
13. **New for this project:** synced tables do not carry UC row filters or masks, so never sync a customer table (ADR-0003). Also check whether a table with a row filter can be a sync source at all (Phase 0).

**Pipelines and serverless**
14. Pipeline progress metrics lag: count rows in the table instead. [5]
15. `.cache()` / `persist` isn't supported on serverless jobs. [11]
16. Serverless notebooks only allow `availableNow` triggers. [4]
17. Changing the columns of a static table joined to a stream kills the stream (`DELTA_SCHEMA_CHANGE_SINCE_ANALYSIS`): restart after reference schema changes. [10]
18. Streaming reads from tables that retention jobs DELETE from need `skipChangeCommits`. [Phase 2 log]

**Agents, Genie, models**
19. UC Python functions have no network: outside APIs (GDACS) go through SQL `http_request` + a UC HTTP connection, or through a job. [25]
20. `CREATE FUNCTION` checks that a `secret()` exists: store a placeholder first. [26]
21. Genie's "right now" needs the feed state spelled out in its instructions (no fresh rows → say unknown, give last seen). [19]
22. Claude endpoints may answer "rate limit of 0" and Agent Bricks Supervisor may be "not available": Partner-powered AI features / account gate. Test on Day 1; fallback `databricks-gpt-oss-120b`. Some models (gemma-3-12b) write tool calls as plain text: don't use them for the agent. [28, 29, FR-17 log]
23. A model that writes SQL needs a guardrail (sqlglot allowlist, fully qualified names) if the app identity has more rights than SELECT. [ADR-0009]

**Cost**
24. One Helsinki demo hour cost about $14–15 (mostly the continuous pipeline and the warehouse). Check `system.billing.usage` per resource after the first session. [log 2026-10-03]

---

## Next step

**Start here next time:**

- [ ] 1. **Your review** of REQUIREMENTS.md, CONTEXT.md and the five ADRs. Change anything; then the plan is frozen.
- [ ] 2. **Create the new Azure workspace** (Premium, Unity Catalog), answer Q2 and Q5, add yourself and your teammate, configure a CLI profile.
- [ ] 3. **Phase 0 platform checks** (REQUIREMENTS §8, Day 1), in this order: OBO auth (R1) → row filter + ABAC with 2 users (R2) → Lakebase sync + 5-min suspend (gotcha 10) → Claude endpoint (R4) → `ai_forecast` (R5) → `searoute` (Q3) → GDACS/USGS reachable → triggered pipeline startup time (R3).
- [ ] 4. Bundle skeleton (`databricks.yml`, schemas, warehouse, Lakebase), copied and adapted from Helsinki.

---

## Log

### 2026-10-23 — Planning (grilling session)
- Read the competition brief (deadline 30 Oct; scoring: business 25, Databricks 25, innovation 20, AI 15, execution 10, clarity 5; mandatory 30-s technical review in the video; Lakebase earns points).
- Grilled the idea: persona, scope, data, freshness, prediction, security, Lakebase split, framework, delay vocabulary, € impact (churn kept separate), Lane Potential, Reroute rules by Shipment Stage, Disruption sources, cut line.
- Wrote CONTEXT.md (glossary), ADR-0001…0005, REQUIREMENTS.md, this file and CLAUDE.md. Moved the brief to `docs/competition-brief.pdf`.
- Nothing created in any workspace.
