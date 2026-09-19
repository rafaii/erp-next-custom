Here's a business-owner dashboard built around the one thing that makes rental arbitrage different from normal property management: the **spread** between your fixed master lease obligation and the variable rent you collect from subtenants.

## Why This Dashboard Is Different

In a master lease arbitrage model, you pay one fixed rent to the building owner and re-rent individual units to tenants — your entire business lives or dies on the gap between those two numbers. A generic P&L or occupancy report doesn't surface that spread clearly, so the dashboard's job is to answer, in plain language, three questions every morning: _Am I making money on this building? Which units are costing me money? Do I have enough cash if something breaks?_

## Critical Metrics for an Arbitrage Owner

|Metric|Plain-language meaning|Why it's critical|
|---|---|---|
|Monthly spread (margin)|Rent I collect minus rent I pay the landlord|The core profit driver of arbitrage [](https://blog.iq.dwellsy.com/master-lease-agreement-definition-key-clauses-and-how-it-works/)|
|Occupancy rate|% of units currently rented out|Empty units still cost you master-lease rent|
|Master lease obligation|Fixed rent + date due to building owner|Your biggest unavoidable expense|
|Rent collected vs. expected|Actual tenant payments vs. what's billed|Surfaces late/missing payments early [](https://www.ube.ac.uk/whats-happening/articles/master-leasing/)|
|Vacancy loss|Rent income lost from empty units this month|Quantifies the cost of vacancy in dollars, not just % [](https://www.toucantoco.com/en/blog/financial-dashboard-for-small-businesses)|
|Cash balance / runway|Money on hand ÷ average monthly burn|Tells you how many months you can survive a bad stretch [](https://www.inetsoft.com/info/rental-property-management-dashboards/)|
|Lease expirations (yours & tenants')|Countdown to master lease renewal and tenant lease-end dates|Prevents surprise loss of the whole building or a big tenant [](https://www.mrisoftware.com/blog/content_module-id63053/)|
|Maintenance/repair spend|Money spent fixing units this month|Arbitrage margins are thin; repairs eat into spread fast|
|Per-unit profitability|Which units are profitable vs. losing money|Lets owner make unit-level decisions (raise rent, drop unit)|
|Overdue tenants|Who hasn't paid and how many days late|Direct action item, not just a report line|

## Dashboard Wireframe

text

`┌─ Business Health — Riverside Building (Real Estate Arbitrage) ── [This Month v] │                                                        Last updated: 2 hrs ago ├────────────────────────────────────────────────────────────────────────────── │  HOW AM I DOING THIS MONTH? │ ┌───────────────────┐ ┌───────────────────┐ ┌───────────────────┐ ┌─────────┐ │ │ MONEY YOU MADE     │ │ RENT YOU PAY       │ │ YOUR PROFIT        │ │ CASH ON │ │ │ (Rent Collected)   │ │ (Master Lease)     │ │ (The Spread)       │ │ HAND    │ │ │  $18,400            │ │  $12,000            │ │  $6,400  ▲ 8%       │ │ $9,200  │ │ │  from 11 tenants    │ │  due in 6 days      │ │  Good — above avg   │ │ 2.3 mo  │ │ └───────────────────┘ └───────────────────┘ └───────────────────┘ └─────────┘ ├────────────────────────────────────────────────────────────────────────────── │  ARE MY UNITS FILLED? │ ┌────────────────────────────────────────────────────────────────────────┐ │ │  Occupancy: 11 / 13 units rented (85%)          [see empty units >]     │ │ │  ██████████████████████████████████░░░░░░  85%                        │ │ │  2 empty units = losing ~$2,100/month in potential rent                │ │ └────────────────────────────────────────────────────────────────────────┘ ├────────────────────────────────────────────────────────────────────────────── │  WHO OWES ME MONEY?                              WHAT'S COMING UP? │ ┌──────────────────────────────────┐             ┌──────────────────────────┐ │ │ ⚠ Unit 4B — John K.  $1,200 (7d late)│         │ Master lease renews in   │ │ │ ⚠ Unit 2A — Maria S. $950 (2d late) │          │  4 months  [Review terms]│ │ │ ✓ 9 tenants paid on time             │         │ Tenant lease ending soon:│ │ │                    [Send reminder]  │          │  Unit 6C - Sept 15       │ │ └──────────────────────────────────┘             └──────────────────────────┘ ├────────────────────────────────────────────────────────────────────────────── │  WHICH UNITS ARE MAKING OR LOSING MONEY? │ ┌──────┬──────────────┬───────────────┬──────────────┬───────────────────┐ │ │ Unit │ Tenant Rent  │ Cost per unit* │ Profit       │ Status            │ │ ├──────┼──────────────┼───────────────┼──────────────┼───────────────────┤ │ │ 1A   │ $1,650       │ $920          │ $730         │ 🟢 Profitable      │ │ │ 2A   │ $1,600       │ $920          │ $680         │ 🟢 Profitable      │ │ │ 4B   │ $1,200       │ $920          │ $280         │ 🟡 Thin margin     │ │ │ 3C   │ $0 (empty)   │ $920          │ -$920        │ 🔴 Vacant — costing │ │ └──────┴──────────────┴───────────────┴──────────────┴───────────────────┘ │  *Cost per unit = master lease rent ÷ total units + shared repairs ├────────────────────────────────────────────────────────────────────────────── │  MONEY SPENT ON REPAIRS/MAINTENANCE THIS MONTH:  $850   [view details >] │  ██████░░░░░░░░░░░░  13% of collected rent — within normal range ├────────────────────────────────────────────────────────────────────────────── │  TREND — Last 6 Months │  Profit:  $4,200 → $4,800 → $5,100 → $5,900 → $6,000 → $6,400   📈 trending up └──────────────────────────────────────────────────────────────────────────────`

## Design Rules for the Non-Accountant View

- **Lead with dollars, not accounting terms** — "Money you made" and "Rent you pay," never "Revenue" and "COGS." Save the technical language for the accountant-facing module you already built.
    
- **Traffic-light coloring everywhere** — green (profitable/on-time), yellow (thin margin/slightly late), red (vacant/overdue) — so the owner scans in seconds, not minutes.
    
- **Turn every risk into an action button** — "Send reminder," "Review terms," "See empty units" — a dashboard for owners should prompt decisions, not just display numbers.[](https://www.inetsoft.com/info/rental-property-management-dashboards/)
    
- **The spread card is the hero metric** — place "Your Profit (The Spread)" front and center since it's the single number that tells an arbitrage operator if the deal is working.[](https://blog.iq.dwellsy.com/master-lease-agreement-definition-key-clauses-and-how-it-works/)
    
- **Surface the master lease renewal countdown prominently** — losing the master lease is the single biggest existential risk in this model, so it deserves permanent visibility, not a buried report.
    

I'll add this to your wireframe reference file as a new section so your design team has it alongside the accounting screens.

## Purpose

This dashboard sits alongside the Accounting module but is designed for the business owner/operator role, not the bookkeeper. It answers three questions in plain language every time the owner logs in:

1. Am I making money on this building?
    
2. Which units are costing me money?
    
3. Do I have enough cash if something breaks?
    

It intentionally avoids accounting terminology (no "GL," "P&L," "COGS") and instead uses the language of the arbitrage business model: rent collected, rent paid, spread, occupancy, vacancy loss.

---

## Core Concept: The Arbitrage Spread

The business model: lease an entire building at a fixed rent from the owner, then sublease individual units to tenants. Profit = total rent collected from tenants minus the fixed master lease rent minus operating costs (repairs, utilities, management). This "spread" is the single most important number on the dashboard and should always be the largest, most visually dominant metric.

---

## Wireframe

text

`+-- Business Health -- Riverside Building (Real Estate Arbitrage) -- [This Month v] |                                                        Last updated: 2 hrs ago +-------------------------------------------------------------------------------- |  HOW AM I DOING THIS MONTH? | +-------------------+ +-------------------+ +-------------------+ +---------+ | | MONEY YOU MADE     | | RENT YOU PAY       | | YOUR PROFIT        | | CASH ON | | | (Rent Collected)   | | (Master Lease)     | | (The Spread)       | | HAND    | | |  $18,400            | |  $12,000            | |  $6,400  (up 8%)    | | $9,200  | | |  from 11 tenants    | |  due in 6 days      | |  Good - above avg   | | 2.3 mo  | | +-------------------+ +-------------------+ +-------------------+ +---------+ +-------------------------------------------------------------------------------- |  ARE MY UNITS FILLED? | +----------------------------------------------------------------------+ | |  Occupancy: 11 / 13 units rented (85%)          [see empty units >]   | | |  [======================================------]  85%                 | | |  2 empty units = losing ~$2,100/month in potential rent               | | +----------------------------------------------------------------------+ +-------------------------------------------------------------------------------- |  WHO OWES ME MONEY?                              WHAT'S COMING UP? | +------------------------------------+           +--------------------------+ | | (!) Unit 4B - John K.  $1,200 (7d late)|        | Master lease renews in   | | | (!) Unit 2A - Maria S. $950 (2d late) |         |  4 months  [Review terms]| | | (ok) 9 tenants paid on time            |        | Tenant lease ending soon:| | |                    [Send reminder]     |        |  Unit 6C - Sept 15       | | +------------------------------------+           +--------------------------+ +-------------------------------------------------------------------------------- |  WHICH UNITS ARE MAKING OR LOSING MONEY? | +------+--------------+---------------+--------------+-------------------+ | | Unit | Tenant Rent  | Cost per unit* | Profit       | Status            | | +------+--------------+---------------+--------------+-------------------+ | | 1A   | $1,650       | $920          | $730         | Green: Profitable  | | | 2A   | $1,600       | $920          | $680         | Green: Profitable  | | | 4B   | $1,200       | $920          | $280         | Yellow: Thin margin| | | 3C   | $0 (empty)   | $920          | -$920        | Red: Vacant - cost | | +------+--------------+---------------+--------------+-------------------+ |  *Cost per unit = master lease rent / total units + shared repairs +-------------------------------------------------------------------------------- |  MONEY SPENT ON REPAIRS/MAINTENANCE THIS MONTH:  $850   [view details >] |  [======------------]  13% of collected rent - within normal range +-------------------------------------------------------------------------------- |  TREND - Last 6 Months |  Profit:  $4,200 -> $4,800 -> $5,100 -> $5,900 -> $6,000 -> $6,400   trending up +--------------------------------------------------------------------------------`

---

## Critical Metrics Reference

|Metric|Plain-language label|Why critical|
|---|---|---|

|Metric|Plain-language label|Why critical|
|---|---|---|
|Monthly spread|"Your Profit (The Spread)"|Core profit driver — rent collected minus rent paid minus costs|
|Occupancy rate|"Are my units filled?"|Vacant units still incur master-lease cost|
|Master lease obligation|"Rent you pay" + due date|Biggest fixed, unavoidable expense|
|Rent collected vs expected|"Money you made"|Surfaces shortfalls immediately|
|Vacancy loss|"2 empty units = losing $X/month"|Converts vacancy into a concrete dollar cost|
|Cash balance / runway|"Cash on hand" in months|Tells owner how long they can survive a shock|
|Lease expirations|"What's coming up"|Master lease renewal and tenant lease-end countdowns|
|Maintenance spend|"% of collected rent"|Thin arbitrage margins are sensitive to repair costs|
|Per-unit profitability|Unit-by-unit table with traffic-light status|Enables unit-level decisions (raise rent, drop unit)|
|Overdue tenants|"Who owes me money" with days late|Direct, actionable list, not a buried report|

---

## Design Rules

1. Lead with dollar amounts and plain phrases, never accounting jargon.
    
2. Traffic-light coloring (green/yellow/red) for instant scanning.
    
3. Every risk item pairs with an action button (Send reminder, Review terms, See empty units).
    
4. The Spread is always the largest, most prominent card on the page.
    
5. Master lease renewal countdown is permanently visible — losing the master lease is the top existential risk in this business model.
    
6. Trend line at the bottom shows direction of travel (improving/declining) without requiring the owner to interpret a full chart.

## Key Takeaway for Your Architecture

This dashboard should pull from the _same_ underlying GL data as your Accounting module — it's just a different lens. The "Rent Collected" card is really the sum of paid Sales Invoices against tenant customers, "Rent You Pay" is your Purchase Invoice/recurring bill to the building owner, and "Your Profit (Spread)" is effectively a mini P&L filtered to just those two account groups. That means you don't need a separate data pipeline — build a reporting layer that aggregates existing GL Entries into these owner-friendly cards, so the accountant's numbers and the owner's dashboard numbers never drift out of sync.

One thing worth deciding now: whether "Cost per unit" in the per-unit profitability table should be a straight pro-rata split of the master lease (as shown) or should also allocate a share of utilities/management fees — that choice affects whether individual "unprofitable" units are really losing money or just absorbing shared costs unevenly, and it's worth exposing as a toggle for power users while keeping the default view simple for the plain-language dashboard.