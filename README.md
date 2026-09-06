# Airline Customer Complaints Analysis

Power BI analysis of 18 months of customer complaints for **Northline Air**, a fictional UK short-haul carrier operating from Manchester, Gatwick and Edinburgh to 18 European destinations.

**Period:** January 2025 – June 2026  
**Scope:** 977 complaints across 45,886 flight legs, £172,360 in compensation  
**Tools:** Power BI (Power Query / M, DAX)

> The dataset is synthetic and generated for this project. It does not represent any real airline's operations.

**Companion project:** flight-level operational data and departure punctuality analysis for the same carrier and period — [Airline Departure Punctuality Analysis](https://github.com/TheLordBass/Airline-Departure-Punctuality-Analysis). Several measures on this page use flight volumes from that dataset as a denominator.

---

## The dashboard

![Customer complaints dashboard](Airline%20complaints.png)

One page: four KPI cards across the top, complaint volume and compensation on a combined axis by category, channel mix, the monthly time series, and station complaint rates normalised per 1,000 flights.

**At a glance:** 977 complaints, £172.36K paid out, an average resolution time of 22.37 days, and Delay as the most common category. The category chart is the clearest single visual — complaint count and compensation fall together across all six categories, which is the point made below about exposure being volume-driven.

---

## Headline findings

### Complaint volume is not driven by punctuality

The correlation between monthly complaint volume and monthly on-time performance is **0.11** — effectively none. July 2025 is the highest complaint month of the 18 (77 complaints) but sits mid-table for punctuality, while the worst two months for departure delay are unremarkable for complaints.

Breaking complaints down by the delay band of the flight they relate to confirms it:

| Departure delay | Complaints | Flights | Complaints per 1,000 |
|---|---|---|---|
| On time (≤15 min) | 742 | 35,591 | 20.8 |
| 16–60 min | 196 | 8,458 | 23.2 |
| 1–3 hours | 30 | 1,360 | 22.1 |
| 3+ hours | 1 | 80 | 12.5 |

The complaint rate barely moves with delay length, and three quarters of complaints come from flights that departed on time.

This matters operationally. "Delay" is the most common complaint *category*, which invites the assumption that reducing delay would reduce complaints. The data does not support that — an OTP improvement programme should not be expected to move complaint volume. Whatever is generating dissatisfaction here sits elsewhere in the journey.

Complaint volume did fall over the period: an average of 56 a month in the first half of 2025 against 46.5 in the first half of 2026.

### Compensation exposure is volume-driven, not category-driven

| Category | Complaints | Compensation |
|---|---|---|
| Delay | 381 | £65,660 |
| Baggage | 195 | £33,720 |
| Booking | 125 | £23,140 |
| Refund | 107 | £18,760 |
| Cabin service | 103 | £18,110 |
| Seating | 66 | £12,970 |

Average payout is close to flat across categories (£172–£197), and the share of complaints attracting a payment sits near 50% throughout. Total compensation therefore tracks complaint volume almost exactly.

The practical consequence: there is no high-cost complaint type to target. Delay is the only category large enough to move the total, and it does so through sheer volume rather than payout size.

### Resolution time is uniformly slow

Average resolution is **22.4 days**, median 22.5, maximum 44. It does not vary meaningfully by category — the spread across all six is 21.8 to 23.2 days.

The flatness is the finding. If resolution time varied by complaint type it would suggest some cases are genuinely harder to investigate. That it doesn't points to a capacity or process constraint affecting all case handling equally, which is a different problem with a different fix.

A 22-day average is poor by industry standards and is arguably the most actionable issue on this dashboard.

**Caveat:** 23 complaints have no resolution date recorded and are excluded from the average. If those are open cases rather than missing data, the true figure is worse.

### Station complaint rates vary 2.3x

NCL generates 32 complaints per 1,000 flights against a network low of 14 at BFS.

As with most operational metrics, the ranking inverts when moving from counts to rates. Gatwick leads on raw complaint volume purely because it operates the most flights, and sits mid-table once normalised.

### Channel mix

Email accounts for 43% of complaints (421), web form 29% (287), phone 16% (155) and social 12% (114). Two thirds of contact arrives through asynchronous written channels, which is consistent with the long resolution times — these are cases worked in a queue, not resolved at first contact.

---

## Methodology

### Cleaning (Power Query)

| Issue | Volume | Treatment |
|---|---|---|
| Channel casing inconsistency (`Email`/`email`, `Web form`/`WEB FORM`) | 84 rows | Trim + proper case |
| Airport codes with leading whitespace and mixed case | 90 distinct values for 18 airports | Trim + uppercase |
| Duplicate flight records in the linked flights table | 642 rows | `Table.Distinct` on `flight_id` |
| `-999` sentinel in delay columns | 91 rows | Replaced with `null` |
| Missing resolution dates | 23 rows | Retained, excluded from averages, flagged as a caveat |

The channel casing issue is worth noting because it was invisible until the data was charted — four logical channels were rendering as six slices, and the totals did not reconcile to 977. Categorical drift of this kind does not throw an error; it silently fragments every visual that uses the field.

### Decisions worth explaining

**Complaints are dated by flight date, not submission date.** The complaints table has no date column of its own and inherits date context through its relationship to `flights`. The time series therefore shows which operating periods generated complaints, not when customers wrote in. For an operational view this is the more useful framing, but it is not the same thing as complaint arrival volume and should not be read as a contact-centre demand curve.

**Station charts use complaints per 1,000 flights, not counts.** Raw complaint counts rank almost perfectly by flight volume, which reproduces the route network rather than revealing anything about service quality.

**Time-series axes start at zero.** Complaint volume ranges 37–77 a month. On a truncated axis this reads as high volatility; on a zero-based axis as a stable band with a mild downward drift. The latter is the accurate impression, and the difference is large enough to change how a reader interprets the chart.

**`-999` was nulled, not filtered.** Removing those rows would drop 91 real flights from the denominator of every rate on this page. Replacing with null preserves the flight while excluding it from any average.

**Compensation is shown against complaint volume on a combined axis.** An earlier version had these as two separate bar charts, which duplicated the same ranking twice. Combining them into a line-and-column chart makes the relationship — that the two track together — the point of the visual rather than something the reader has to infer across two charts.

### Data model

- `complaints` — fact table, related to `flights` on `flight_id`
- `flights` — provides origin station, delay band and date context
- `Date Table` — generated with `CALENDAR()`, marked as a date table, related to `flights[departure_date]`
- `airports` — station reference

`Month Year` is set to sort by a `yyyy-mm` column so the time axis orders chronologically rather than alphabetically.

### Core measures

```dax
Number of Complaints = COUNTROWS( complaints )

Total Compensation Paid = SUM( complaints[compensation_paid_gbp] )

Average Resolution Time = AVERAGE( complaints[days_to_resolve] )

Eligible Flights =
CALCULATE(
    COUNTROWS( flights ),
    flights[cancelled] = 0,
    NOT ISBLANK( flights[departure_delay_mins] )
)

Complaints per 1000 Flights =
DIVIDE( COUNTROWS( complaints ), [Eligible Flights] ) * 1000
```

---

## Definitions

**Complaints per 1,000 flights** — complaint count normalised by departures from that station, so stations of different sizes can be compared.

**Days to resolve** — calendar days between complaint receipt and case closure. Open cases have no value and are excluded.

**Compensation paid** — total settlement value, including statutory delay compensation and goodwill payments. Around half of complaints result in no payment.

---

## Repository contents

| File | Description |
|---|---|
| `Airline data.pbix` | Power BI report, including all Power Query steps and DAX measures |
| `Airline complaints.png` | Screenshot of the report page |

---

## Possible extensions

**Complaint driver analysis.** Punctuality explains almost none of the variation in complaint volume, which leaves the actual driver unidentified. Testing against load factor, aircraft age, route, time of day and cabin crew rostering would turn an open question into an answer, and is the most valuable next step.

**Resolution time distribution.** The 22.4-day average is stable across every dimension tested, but an average conceals its tail. A distribution view would show whether the process is uniformly slow or whether a subset of cases runs far longer.

**Repeat complainants.** The current model treats every complaint as independent. Identifying customers who complain more than once would distinguish systemic service failure from isolated incidents.
