# Shipping Control Tower

A freight forwarder's view of every Shipment it moves for its Customers: where each one is, which Disruptions threaten it, what that costs, and how its Lanes perform commercially.

## Language

### Shipments and network

**Shipment**:
One customer booking moved by the forwarder from an origin to a destination, carried in one or more Containers (FCL) or as part of a shared Container (LCL).
_Avoid_: Order, consignment, booking (a Booking is the carrier-side reservation)

**Lane**:
An origin port → destination port pair the forwarder sells, regardless of the route the vessel takes between them.
_Avoid_: Trade lane, route (a Route is the physical path), corridor

**TEU**:
Twenty-foot equivalent unit: the volume unit for Containers. A 20' Container is 1 TEU, a 40' Container is 2 TEU.
_Avoid_: Box count, containers (when volume is meant)

**Container Type**:
The size and load mode of a Shipment's equipment: 20' or 40', FCL (the Customer fills the Container alone) or LCL (the Shipment shares a Container with others). Ocean only.
_Avoid_: Equipment, box type

**Shipment Stage**:
Where a Shipment is in its life: Booked (not yet loaded), In Transit (on board a vessel or flight), At Hub (waiting at a Transshipment Hub between legs), Arrived, Delivered. The stage decides which Reroutes are possible.
_Avoid_: Status (too generic), milestone (a Milestone is the event that changes the stage)

**Carrier**:
The shipping line or airline that operates the vessel or flight a Shipment travels on. The forwarder buys capacity from Carriers but does not steer their vessels.
_Avoid_: Vendor, supplier, liner

**Voyage**:
One sailing of one vessel (or one flight) along a sequence of ports. Many Shipments share a Voyage.
_Avoid_: Trip, sailing, service (a service is the repeating schedule)

**Transshipment Hub**:
A port or airport where a Shipment changes from one Voyage to another (e.g. Singapore, Colombo, Dubai).
_Avoid_: Transfer port, relay port

**Mode**:
How a Shipment travels: ocean or air. Volume is TEU for ocean and chargeable kg for air.
_Avoid_: Transport type, product

### Commercial

**Customer**:
A company that buys transport from the forwarder and is billed for its Shipments.
_Avoid_: Client, shipper (ambiguous with the BCO), account

**Segment**:
The industry a Customer belongs to (e.g. Automotive, Retail, Pharma, Electronics).
_Avoid_: Vertical, industry, sector

**Division**:
A business unit of the forwarder that sells and runs a product line (e.g. Ocean FCL, Ocean LCL, Air Freight).
_Avoid_: Business unit, BU, product line

**Program Manager**:
The forwarder's owner of a Customer Portfolio, responsible for those Customers' Shipments end to end.
_Avoid_: Account manager, PM (ambiguous with project manager), ops user

**Customer Portfolio**:
The set of Customers one Program Manager owns. A Customer belongs to exactly one Customer Portfolio.
_Avoid_: Book of business, account list

**Leadership**:
Forwarder executives who see every Customer, starting from aggregates.
_Avoid_: Management, execs, C-level

### Timeliness

**Committed Delivery Date**:
The date the forwarder promised the Customer for a Shipment's delivery, under the Customer's SLA.
_Avoid_: Due date, deadline, SLA date

**Planned ETA**:
The arrival time in the Shipment's original plan, fixed at booking.
_Avoid_: Original ETA, schedule

**Predicted ETA**:
The current best estimate of a Shipment's arrival, updated as the voyage progresses and Disruptions change it.
_Avoid_: ETA (unqualified), forecast arrival

**Delay**:
How much later the Predicted ETA is than the Planned ETA. A Shipment can be delayed without breaching its SLA.
_Avoid_: Lateness, slippage

**SLA Breach**:
A Shipment whose Predicted ETA (or actual arrival) is after its Committed Delivery Date.
_Avoid_: Late shipment, failure, delayed (when breach is meant)

**At Risk**:
A Shipment not yet in SLA Breach whose predicted chance of breaching is above the configured threshold.
_Avoid_: Risky, flagged, warning

**Stuck**:
A Shipment whose carrier is held in place by a Disruption (e.g. at anchor waiting for a canal to reopen).
_Avoid_: Blocked, halted, stranded

### Disruption impact

**SLA Penalty**:
The credit a Customer's contract grants per day of SLA Breach, as a share of the Shipment's freight revenue, up to a cap.
_Avoid_: Fine, chargeback

**Reroute Cost**:
The extra cost to the forwarder of moving a Shipment on a Reroute instead of its plan.
_Avoid_: Surcharge (a surcharge is billed to the Customer), extra cost

**Disruption Impact**:
The cash one Disruption costs the forwarder: SLA Penalties plus Reroute Cost, always shown as those two parts as well as the total.
_Avoid_: Loss, damage, cost of disruption

**Churn Risk Revenue**:
The annual revenue of Customers whose repeated SLA Breaches make them likely to leave. Shown next to Disruption Impact, never added to it.
_Avoid_: Revenue at risk (too broad), lost revenue

**Disruption**:
An event that slows or stops Shipments in a place for a time: a Chokepoint closure, a natural hazard (volcano, earthquake, cyclone, flood) or port congestion/strike.
_Avoid_: Incident, event, issue, calamity

**Chokepoint**:
A narrow passage many ocean Lanes must use (Suez Canal, Panama Canal, Bab-el-Mandeb, Strait of Malacca), where one Disruption hits many Shipments at once.
_Avoid_: Canal, strait (when the general concept is meant)

**Reroute**:
A change the forwarder makes to how a Shipment travels, chosen to limit Disruption Impact: another sea route, another port, a mode shift to air, or waiting it out (the baseline). Only Booked and At Hub Shipments can be rerouted; an In Transit Shipment can only wait.
_Avoid_: Diversion (a diversion is the Carrier's decision to send its vessel another way), re-plan, alternative

**Reroute Decision**:
A Program Manager's approval of one Reroute for a group of Shipments, recorded with who approved it, when, and the expected Disruption Impact.
_Avoid_: Approval, action, ticket

**Observed Disruption**:
A Disruption taken from a real public alert feed (e.g. GDACS, USGS).
_Avoid_: Real event, live alert

**Injected Disruption**:
A Disruption created on purpose by the Scenario Injector for a demo or drill, always labelled as simulated.
_Avoid_: Fake event, test event, what-if

### Lanes and pricing

**Gross Margin**:
Freight revenue billed to the Customer minus the carrier cost paid for a Shipment; compared across Lanes per TEU (ocean) or per chargeable kg (air).
_Avoid_: Profit, GP, yield

**Lane Potential**:
How attractive a Lane is to grow, from its Gross Margin per volume unit and its volume trend and forecast. Underperforming and high-potential Lanes are the two ends of it.
_Avoid_: Lane score, lane health, opportunity
