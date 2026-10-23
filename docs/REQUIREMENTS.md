# Freight Control Tower — Requirements & Plan

A Databricks App for a freight forwarder (ocean + air). When a Disruption hits, it finds the affected Shipments within seconds, prices the Disruption Impact, and lets an agent propose Reroutes for a Program Manager to approve. The same data model answers the commercial questions: which Lanes, Container Types, Customers, Segments and Divisions make or lose money, and which Lanes to grow.

- Competition: Databricks × Accenture App Competition ([competition-brief.pdf](./competition-brief.pdf), kept local, not in git). **Deadline: Fri 30 Oct 2026, 23:59 CET (= Sat 31 Oct 04:29 IST). Plan to submit on 30 Oct by 20:00 IST.**
- Vocabulary: [CONTEXT.md](../claude_notes/CONTEXT.md). Terms in **bold** are defined there.
- Decisions: [docs/adr/](./adr/).
- Status and next step: [PROGRESS.md](../claude_notes/PROGRESS.md).
- Working name "Freight Control Tower"; rename freely before the video.

---

## 1. Problem, users, impact

### 1.1 The problem (for the write-up)
When a canal closes, a volcano erupts or a port strikes, a forwarder's ops team spends **hours to days** working through carrier advisories, TMS exports and spreadsheets to answer four questions: *Which of our Shipments are affected? Which Customers will see an SLA Breach? What will it cost us? What should we do?* By the time they answer, rebooking capacity is gone and Customers have heard about it from the news first. Separately, commercial teams price Lanes from quarterly reports and can't easily see which Lanes and Container Types erode margin.

**Current workaround:** carrier emails plus manual Excel lookups against TMS exports; impact estimated after the fact; Lane profitability reviewed quarterly in BI decks.

### 1.2 Public evidence (verify every figure and add sources before submission; Q1)
- *Ever Given*, Suez Canal, 23–29 Mar 2021: about 6 days, ~370 ships queued, widely cited ~$9.6bn of trade held per day.
- Red Sea / Bab-el-Mandeb, from Dec 2023: most Asia–Europe services diverted via the Cape of Good Hope, about +10–14 days per voyage.
- Panama Canal drought, 2023: draft limits and fewer daily transits.
- Eyjafjallajökull, Apr 2010: European airspace closures; this is the air-freight volcano case.

### 1.3 Users
| Persona | Sees | Main jobs |
|---|---|---|
| **Program Manager** (one per **Customer Portfolio**) | Only their Customers' Shipments, row by row; revenue but not cost/margin (ADR-0004) | Triage affected Shipments, approve Reroutes, notify Customers |
| **Leadership** | All Customers, starting from aggregates; can drill down; sees margin | Exposure across the network, Disruption Impact, Lane and Segment performance |
| Admin / demo operator | Scenario Injector | Inject a Chokepoint closure or port strike for demos and drills |

### 1.4 Impact estimate (method; numbers are filled in from the synthetic forwarder in Phase 5)
- **Time to answer** "which Shipments, what cost, what to do": baseline 4–8 h of an ops analyst per Disruption (assumption, stated as such) → < 1 min in the app.
- **Avoided SLA Penalties**: penalties if every affected Shipment waits, minus penalties after the approved Reroutes, minus Reroute Cost. Computed per Disruption by the engine (FR-7), so the write-up can show the calculation.
- **Margin uplift from the Lane view**: e.g. repricing or shifting volume on the bottom-decile Lanes by Gross Margin per TEU. Shown as an illustrative calculation.

---

## 2. Goals, non-goals, cut line

### Goals
1. A live operations picture: vessels and flights on a map, Shipments by **Shipment Stage**, **Delay**, **SLA Breach**, **At Risk** and **Stuck** counts (FR-9, FR-10).
2. Disruption impact in seconds: Observed Disruptions (GDACS/USGS) and Injected Disruptions are matched to Shipments, with recomputed Predicted ETAs, SLA Penalties, Reroute Cost and **Churn Risk Revenue** shown separately (FR-6, FR-7).
3. Prediction: an SLA-breach model scores every open Shipment (FR-8).
4. Agent: proposes valid Reroutes per Shipment Stage with €/day trade-offs; the Program Manager approves; the approval is written to Lakebase (FR-12).
5. Commercial insight: revenue, cost, Gross Margin by Customer, Segment, Division, country and region; Lane Potential with Container Type profitability and a volume forecast (FR-11).
6. Governance as a feature: the same app shows different rows (RLS by Customer Portfolio) and masked columns (ABAC) depending on who is logged in (FR-4).
7. Ask anything: a Genie space over the gold tables, queried as the user (FR-13).

### Non-goals
- Real customer or carrier data, or real company names (ADR-0001).
- Live AIS ingestion (post-hackathon option, ADR-0001).
- Road freight; multi-leg door-to-door trucking beyond a fixed on-carriage cost/time for an alternative port.
- Booking with real carriers; the "act" in FR-12 records the decision and drafts the notification, it does not call carrier APIs.
- 24/7 operation: everything runs on demand (NFR-3).
- A PM what-if sandbox (Injected Disruptions cover the demo need).

### Cut line
**MUST (by Wed 28 Oct):** FR-1–FR-10, FR-11.1–11.3, FR-12.1–12.5, FR-13, FR-14.
**STRETCH (only after the MUST list works end to end):** notification drafts (FR-12.6), port congestion/strike as an Injected type (FR-6.3), the Churn Risk Revenue KPI (FR-7.4), an AI/BI dashboard (FR-15), flights moving on the map (FR-9.3).

---

## 3. Environment and constraints

| Item | Decision |
|---|---|
| Workspace | **New** Azure Databricks workspace, not created yet. Host, CLI profile and catalog go here once known (Q2). Prefer a region where Claude Foundation Model endpoints are enabled (Helsinki's Germany West Central blocked them) |
| Catalog | New project catalog (name: Q2), never `main`, never the Helsinki catalog |
| Schemas | `fct_ref` (reference data), `fct_ops` (pipeline tables), `fct_sec` (portfolio mapping + filter/mask functions), `fct_ml` (models, features). Names provisional |
| Ingestion | Generator job writes JSON events to a UC Volume; Auto Loader in a Lakeflow Declarative Pipeline (ADR-0001, ADR-0002) |
| Pipeline | Lakeflow Declarative Pipelines, serverless, triggered about once a minute inside demo sessions (ADR-0002) |
| ML | MLflow + UC model registry; LightGBM; batch scoring inside the pipeline (no always-on serving endpoint) |
| Live map serving | Lakebase synced table with positions only (ADR-0003) |
| Governed reads | Serverless SQL warehouse with on-behalf-of-user auth (ADR-0003) |
| App writes | Lakebase tables owned by the app service principal (Reroute Decisions, Injected Disruptions, drafts) |
| Security | UC row filters + ABAC policies on governed tags (ADR-0004) |
| Agent | Tools as UC functions; tool-calling loop in the app; Claude if available, else `databricks-gpt-oss-120b` (ADR-0005) |
| Chat | Genie space via the Conversation API, as the user |
| App | Databricks App, Streamlit + pydeck (runtime Streamlit 1.38, see gotchas) |
| Packaging | Databricks Asset Bundle, one `dev` target (no `mode: development`, see gotchas) |
| Budget | **≤ ₹5,000 total** (NFR-3) |
| Time | 7 days, one implementer; ideas shared with one teammate |

---

## 4. Data

### 4.1 Real reference data (public)
- **Ports**: UN/LOCODE or the World Port Index (name, country, lat/lon). About 40 major ports are enough.
- **Airports**: about 15 cargo hubs (IATA, lat/lon).
- **Chokepoints**: Suez, Panama, Bab-el-Mandeb, Malacca, Gibraltar, Cape of Good Hope (as the detour), each as a small polygon.
- **Sea routes**: port-to-port polylines generated once with the `searoute` Python package (licence and Chokepoint passage are checked in Phase 0, Q3), so we know which Lanes pass which Chokepoint.

### 4.2 Real disruption feeds
- **GDACS** events API / RSS: volcano (VO), tropical cyclone (TC), flood (FL), earthquake (EQ), drought; alert level, coordinates, dates. No key.
- **USGS** earthquake GeoJSON summary feed. No key.
- Polled by a small job every few minutes during demo sessions (and once a day otherwise). The impact radius per type and alert level is a configuration table.

### 4.3 Synthetic forwarder (generator, seeded and reproducible)
| Entity | Size (default) | Notes |
|---|---|---|
| Customer | 120 | fictional names; Segment, country, region, annual revenue, SLA terms (penalty % per day, cap %) |
| Customer Portfolio / Program Manager | 6 portfolios | 2 mapped to real demo identities (you + teammate), the rest to placeholder emails |
| Division | 3 | Ocean FCL, Ocean LCL, Air Freight |
| Carrier | 8 ocean + 4 air | fictional |
| Lane | ~60 ocean + ~15 air | Asia–Europe, Transpacific, Transatlantic, intra-Asia, Latin America |
| Voyage | rolling schedule | services per Lane with Transshipment Hubs |
| Shipment history | ~2 years, ~150k | completed, with outcomes: the training data and Lane analytics |
| Open Shipments | ~3,000 | Booked / In Transit / At Hub at simulation start |

Prices and costs: buy rate per TEU/kg per Lane × Container Type with seasonality and noise, and sell rate = buy × margin factor by Customer and Segment. Some Lanes are deliberately underpriced and some are growing fast, so the Lane view has something to find.

Delays in history: base variance per Carrier and Lane plus seeded past Disruptions (Red Sea style), so the model has real signal.

---

## 5. Functional requirements

### FR-1 Generator and Simulated Clock
- FR-1.1 A seeded generator builds the reference, history and open-Shipment tables (§4.3) in one run (backfill job).
- FR-1.2 A demo-session generator advances the Simulated Clock at a configurable rate (default 1 real minute = 1 simulated hour) and writes events to a Volume: vessel/flight positions, Milestones (loaded, departed, arrived at hub, departed hub, arrived, delivered) and new Bookings.
- FR-1.3 Generated vessel positions follow the `searoute` polylines. A Carrier diversion (e.g. via the Cape when Suez is closed) switches the polyline and adds the extra days.
- FR-1.4 The generator reads active Disruptions (observed + injected) and applies them: Stuck vessels hold position at the Chokepoint/port, and diverted Voyages take the detour.

### FR-2 Bronze / silver / gold pipeline
- FR-2.1 Bronze: Auto Loader over the Volume, raw JSON plus file metadata, append-only.
- FR-2.2 Silver: typed `silver_positions`, `silver_milestones`, `silver_disruptions` (observed + injected merged, with `source` = observed | injected), deduplicated, with expectations (non-null ids, coordinates in range).
- FR-2.3 Gold `gold_shipment_current`: one row per open Shipment with Stage, current Voyage, Planned ETA, Predicted ETA, Delay, Committed Delivery Date, SLA Breach flag, breach probability (FR-8), At Risk, Stuck, active Disruption id(s), revenue, carrier cost, Gross Margin.
- FR-2.4 Gold `gold_vessel_current`: latest position per vessel/flight (synced to Lakebase, FR-9).
- FR-2.5 Gold commercial tables: `gold_lane_weekly` (Lane × Container Type × week: Shipments, TEU / chargeable kg, revenue, cost, Gross Margin, SLA Breach rate) and `gold_customer_monthly` (Customer, Segment, Division, country, region).

### FR-3 Reference job
- FR-3.1 Loads ports, airports, Chokepoints and sea-route polylines with the Lane → Chokepoint passage table. A failed load never truncates the existing tables.

### FR-4 Security (ADR-0004)
- FR-4.1 `fct_sec.portfolio_members` maps users/groups to Customer Portfolios; `fct_sec.customer_portfolio` maps Customers to portfolios.
- FR-4.2 A row filter on every table with `customer_id` (silver/gold Shipment tables and commercial tables) lets a member of `fct_leadership` see everything and anyone else only their portfolios' Customers.
- FR-4.3 Governed tag `commercially_sensitive` on carrier cost and Gross Margin columns; an ABAC policy masks them (NULL) for anyone outside `fct_finance` / `fct_leadership`. Fallback: column mask functions (R2).
- FR-4.4 The app shows a "Signed in as … · Portfolio … · margin hidden" badge, so the video makes the security visible.
- FR-4.5 Acceptance: the same app page, opened by the two demo identities side by side, shows different Shipment counts and masked vs. visible margin.

### FR-5 Disruption feeds
- FR-5.1 A job polls GDACS and USGS, keeps events with an alert level ≥ the configured level, and writes them to bronze.
- FR-5.2 Each Disruption gets a footprint (radius by type and alert level from configuration, or a Chokepoint polygon) and a start/expected end.

### FR-6 Scenario Injector (admin only)
- FR-6.1 An admin-only app panel creates an Injected Disruption: Chokepoint closure (which Chokepoint, start = now in Simulated Time, duration in days). It is stored in Lakebase and picked up by the generator and pipeline.
- FR-6.2 Every Injected Disruption is labelled "simulated" everywhere it appears.
- FR-6.3 *(STRETCH)* Port congestion/strike: a port and extra dwell days.

### FR-7 Scenario engine (deterministic, UC functions; ADR-0005)
- FR-7.1 `affected_shipments(disruption_id)`: open Shipments whose remaining route (Voyage polyline, or the Lane polyline if Booked) intersects the footprint during its active window.
- FR-7.2 Recomputes the Predicted ETA per affected Shipment: Chokepoint closure → min(waiting for reopening, Carrier diversion detour); hazard at a port/airport → closure days; air via a volcano ash zone → airport closure days.
- FR-7.3 `disruption_impact(disruption_id)`: counts (affected, new SLA Breaches, At Risk, Stuck), SLA Penalties (Customer terms, capped) and Reroute Cost of the approved decisions. Shown as parts and total.
- FR-7.4 *(STRETCH)* Churn Risk Revenue: annual revenue of Customers with ≥ N SLA Breaches in 90 days, shown next to the impact, never summed.
- FR-7.5 `reroute_options(shipment_ids)`: only options valid for the Shipment Stage (Booked → other service/route, other port + on-carriage, air; At Hub → other connecting Voyage, air; In Transit → wait only), each with the new Predicted ETA, Reroute Cost, the SLA Penalty after the Reroute, and the net € vs. waiting.
- FR-7.6 Unit tests cover the stage rules, penalty caps and the "wait" baseline.

### FR-8 SLA-breach prediction
- FR-8.1 A training notebook/job builds features from history (Lane, Carrier, Mode, Container Type, season, Transshipment count, buffer days to the Committed Delivery Date, Chokepoint passage, active Disruption on the remaining route, current Delay), trains LightGBM, logs to MLflow and registers in UC with alias `champion`.
- FR-8.2 The pipeline scores every open Shipment on each update (`breach_probability`); At Risk = probability ≥ threshold (default 0.5, configurable).
- FR-8.3 The app shows the top drivers (SHAP or feature importance) for a selected Shipment.
- FR-8.4 The model card in the write-up states the AUC on a time-based holdout and that the labels come from the simulator (ADR-0001).

### FR-9 Live map (Lakebase)
- FR-9.1 Vessels from the Lakebase synced table, coloured by status (normal / delayed / stuck), with Chokepoints and active Disruption footprints drawn (Observed vs. Injected styled differently).
- FR-9.2 Clicking a vessel lists its Shipments **from the warehouse, as the user** (RLS applies, so a PM sees only their own).
- FR-9.3 *(STRETCH)* Flights moving on the map.

### FR-10 Operations views
- FR-10.1 **Leadership overview**: KPI tiles (open Shipments, Delayed, SLA Breach, At Risk, Stuck, Disruption Impact €, revenue in transit), trends, breakdown by region / Division / Segment, and the active Disruptions ranked by impact.
- FR-10.2 **Program Manager workbench**: the Shipments in the viewer's portfolios, filterable by Disruption, Stage, Customer and At Risk, sorted by breach probability; a Shipment detail with timeline, drivers (FR-8.3) and Reroute options.
- FR-10.3 The landing screen is chosen from the viewer's group (Leadership → overview, otherwise the workbench); both screens stay reachable through tabs, and the data still obeys RLS.

### FR-11 Commercial and Lane insights
- FR-11.1 Revenue, carrier cost and Gross Margin by Customer, Segment, Division, country and region, with period filters (margin masked per FR-4.3).
- FR-11.2 Lane table: Shipments, TEU / chargeable kg, Gross Margin per TEU / per kg, split by Container Type (20'/40', FCL/LCL), with a scatter of Gross Margin per TEU vs. volume growth.
- FR-11.3 Lane Potential: 12-week volume trend plus an 8-week forecast (`ai_forecast`; fallback: a Prophet/linear job, R5). Lanes are classified as *high potential* (growing, margin ≥ median), *underperforming* (margin in the bottom quartile or shrinking) or *steady*.
- FR-11.4 A "Why?" button asks the agent/Genie to explain a Lane's classification from the data.

### FR-12 Reroute agent (ADR-0005)
- FR-12.1 Tool-calling loop in the app (Helsinki `agent.py` pattern) with tools = the FR-7 UC functions plus a Genie tool, run **as the user** (ADR-0003).
- FR-12.2 Given a Disruption (or a selection of Shipments), the agent groups the affected Shipments (Customer × Stage × Voyage), calls `reroute_options` and recommends one option per group with a one-paragraph reason that quotes the tool's figures.
- FR-12.3 The UI shows the recommendation as a table (group, option, new Predicted ETA, Reroute Cost, net €) with **Approve** / **Reject** per row.
- FR-12.4 Approve writes a **Reroute Decision** to Lakebase (`approved_by` from the identity header, timestamp, Shipment ids, option, expected figures); the Shipments then show "Reroute approved" and the impact KPIs update.
- FR-12.5 "Tools the agent used" expander (as in Helsinki) for transparency in the video.
- FR-12.6 *(STRETCH)* The agent drafts a Customer notification email per Customer, stored in Lakebase; nothing is sent.

### FR-13 Ask (Genie)
- FR-13.1 A Genie space over the gold tables with the CONTEXT.md vocabulary in its instructions and 5–8 example SQLs (Leadership and Lane questions).
- FR-13.2 An Ask tab uses the Conversation API as the user, so a PM's Genie answers obey RLS and ABAC (shown in the video).
- FR-13.3 The instructions say how to answer when no demo session is live (as Helsinki gotcha 19 taught).

### FR-14 Demo session
- FR-14.1 `demo_session` job: starts the generator, the GDACS poller, the pipeline schedule, the Lakebase sync and the app; holds for N minutes (default 30); always stops everything (`ALL_DONE`).
- FR-14.2 `demo_stop` job.
- FR-14.3 A `demo_reset` job restores the open-Shipment state and Simulated Clock to the seeded start, so the video can be re-recorded.

### FR-15 *(STRETCH)* AI/BI dashboard
- FR-15.1 A published Lakeview dashboard over the commercial gold tables (Leadership).

---

## 6. Non-functional requirements

- **NFR-1 Freshness**: a Milestone or position written by the generator shows in the app within ≤ 2 min real time (measured in Phase 1).
- **NFR-2 Speed**: warehouse views ≤ 3 s; map refresh ≤ 1 s from Lakebase; agent recommendation ≤ 30 s.
- **NFR-3 Cost**: whole project ≤ ₹5,000. Serverless only; warehouse 2X-Small with 5-min auto-stop; Lakebase scale-to-zero with a **5-min suspend set on the existing `primary` endpoint** (Helsinki gotcha 24); no always-on serving endpoints; nothing runs outside demo sessions. Check `system.billing.usage` daily.
- **NFR-4 Security**: no Shipment, Customer or commercial read as the service principal (ADR-0003); the app service principal has no SELECT on unfiltered gold tables except through the positions sync.
- **NFR-5 Reproducibility**: one `databricks bundle deploy -t dev` plus `setup_*` scripts; seeded generator.
- **NFR-6 Honesty**: Injected Disruptions and synthetic data are labelled; every € figure can be traced to a function and its inputs.
- **NFR-7 Tests**: unit tests for the generator rules, the scenario engine (FR-7.6), the agent loop and a Streamlit 1.38 smoke test (Helsinki pattern).

---

## 7. Architecture (target)

```
GDACS / USGS ──poll job──┐
Generator (Sim Clock) ───┼─► UC Volume ─► Lakeflow pipeline (Auto Loader)
Scenario Injector (Lakebase) ─┘        bronze → silver → gold ──► batch scoring (UC model)
                                                   │
                         ┌─────────────────────────┼──────────────────────────┐
             gold_vessel_current          gold_* (row filters + ABAC)     UC functions
              (synced → Lakebase)          SQL warehouse, as the user      (scenario engine)
                         │                         │                          │
                         └──────────── Databricks App (Streamlit) ────────────┘
                              map · Leadership · PM workbench · Lanes · Ask (Genie) · Agent
                                         │ Approve
                                         ▼
                                Lakebase: Reroute Decisions, Injected Disruptions
```

---

## 8. Delivery plan (7 days)

| Day | Date | Phase | Exit check |
|---|---|---|---|
| 0 | Fri 23 Oct | **Planning** (this doc) · create the workspace | Docs reviewed; workspace host + catalog known |
| 1 | Sat 24 Oct | **Phase 0 Platform checks + bundle**: OBO auth with `sql` scope, row filter + ABAC on a toy table seen by 2 users, Lakebase sync, Claude endpoint, `ai_forecast`, `searoute`, GDACS reachable, triggered pipeline startup time | Each check passed or a fallback chosen (R1–R5) |
| 1–2 | Sat–Sun | **Phase 1 Data**: reference job, generator backfill + Simulated Clock, pipeline bronze→gold, security (FR-4) | Two users see different rows; NFR-1 measured |
| 2–3 | Sun–Mon | **Phase 2 Intelligence**: scenario engine UC functions + tests, SLA-breach model + scoring, Lane Potential + forecast | Suez injection → affected count + € in SQL |
| 3–4 | Mon–Tue | **Phase 3 App**: map (Lakebase), Leadership, PM workbench, Lanes, Ask (Genie), Scenario Injector | All tabs work as both users |
| 4–5 | Tue–Wed | **Phase 4 Agent**: reroute loop, approve → Lakebase, tools-used expander; demo_session/stop/reset | End-to-end demo story runs twice in a row |
| 6 | Thu 29 Oct | **Phase 5 Polish + video**: impact numbers, stretch items only if green, record the 3-min video incl. the 30-s technical review | Video uploaded with public access |
| 7 | Fri 30 Oct | **Write-up + submit**: 1–2 page description, check links, submit by 20:00 IST | Submitted |

### Video storyboard (3 min, draft)
1. 0:00–0:25 Problem: "Suez is blocked again: which of our 3,000 Shipments are hit, and what will it cost?" (public facts).
2. 0:25–1:30 App: Leadership overview → Inject "Suez closed 10 days" → KPIs move → PM workbench (second browser, fewer rows, margin hidden) → agent recommends → Approve → impact drops.
3. 1:30–2:10 Lanes: underperforming vs. high-potential Lanes, 40' FCL vs. LCL margin, ask Genie "which Segment has the weakest margin?".
4. 2:10–2:30 Result: time-to-answer and € avoided, with the calculation.
5. 2:30–3:00 **Technical review** (mandatory): bundle resources, pipeline graph, UC lineage, row filter / ABAC policy, model in UC, Lakebase tables, app resources.

---

## 9. Risks

| ID | Risk | Mitigation |
|---|---|---|
| R1 | On-behalf-of-user auth (scopes, Genie via OBO) not available or behaves differently in the new workspace | Check on Day 1 (first Phase 0 item). Fallback: the app reads as its service principal and filters by the viewer's portfolio in a UC function that takes the viewer's email, while the platform RLS/ABAC is shown with each user opening Genie / the SQL editor directly. This weakens ADR-0003, so supersede it with a new ADR if it comes to that |
| R2 | ABAC policies / governed tags not enabled | Plain UC column mask functions; same demo |
| R3 | Serverless triggered update slower than 1 min | Continuous pipeline for the session only (ADR-0002) |
| R4 | Claude endpoints blocked (Helsinki gotcha 29) | `databricks-gpt-oss-120b`, proven with tool calling |
| R5 | `ai_forecast` unavailable | Small forecasting job (linear/Prophet) writing to a table |
| R6 | Scope: two flagships + air, solo, 7 days | Cut line (§2); stretch only after the MUST demo runs twice |
| R7 | Budget overrun | NFR-3 checks; daily billing query; stop everything after each session |
| R8 | GDACS has nothing relevant during recording | The video uses an Injected Suez closure; Observed Disruptions are shown as context |
| R9 | A synthetic model looks "too good" | Add noise; show a time-based holdout honestly; state in the write-up that labels are simulated |

## 10. Open questions

- **Q1** Sources for the public figures in §1.2 (collect links on Day 6).
- **Q2** New workspace: host, region, CLI profile name, catalog name, and whether you are account admin (to create groups `fct_leadership`, `fct_finance`, `fct_pm_*`).
- **Q3** `searoute` licence and whether its network routes Asia–Europe through Suez by default (and can be forced via the Cape).
- **Q4** App and project name for the video.
- **Q5** Which demo identity is Leadership and which is a Program Manager (you or your teammate)?
