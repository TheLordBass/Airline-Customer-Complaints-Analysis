# Airline Operational Performance Analysis

Power BI analysis of 18 months of short-haul operations for **Northline Air**, a fictional UK carrier flying from Manchester, Gatwick and Edinburgh to 18 European destinations.

**Period:** January 2025 – June 2026
**Scope:** 45,886 flight legs, 977 customer complaints, 25 aircraft, 18 airports
**Tools:** Power BI (Power Query / M, DAX)

> The dataset is synthetic and generated for this project. It does not represent any real airline's operations.

The report has two pages: **departure punctuality** and **customer complaints**.

---

## Page 1 — Departure punctuality

### OTP15 averaged 78.2% against an 80% target, missing target in 9 of 18 months

Punctuality is strongly seasonal and the pattern repeats across both years:

| Best months | | Worst months | |
|---|---|---|---|
| Oct 2025 | 81.7% | Feb 2025 | 70.9% |
| May 2026 | 81.4% | Feb 2026 | 71.7% |
| Nov 2025 | 81.3% | Jan 2026 | 72.5% |

Both February troughs sit near 71%, roughly 9 points below the autumn peaks. Winter disruption is the largest single driver of missed target — and because it repeats, it is a planning problem rather than an operational surprise.

### Reactionary delay is the most expensive category per event

| Controllability | Delayed flights | Delay minutes | Share of minutes | Avg mins per delay |
|---|---|---|---|---|
| Controllable | 6,369 | 154,595 | 36.7% | 24.3 |
| Uncontrollable | 6,164 | 149,770 | 35.5% | 24.3 |
| Reactionary | 3,924 | 117,111 | 27.8% | **29.8** |

Reactionary delay — knock-on from a late inbound aircraft — produces the fewest events but the longest average delay. Around 28% of all delay minutes are inherited from earlier in the same aircraft's day rather than originating at the gate, which points at turnaround buffer as the highest-leverage intervention available.

### Station performance does not track station size

| Station | Flights | Delay rate | Avg delay |
|---|---|---|---|
| CDG | 1,453 | 27.9% | 12.5 min |
| AMS | 1,519 | 27.3% | 13.2 min |
| FCO | 1,126 | 26.3% | 12.5 min |
| MAN | 8,108 | 19.3% | 8.8 min |
| TFS | 1,661 | 18.1% | 9.4 min |
| FAO | 1,731 | 18.1% | 8.7 min |

Ranked by count, the largest bases dominate simply because they operate the most flights. Ranked by rate, the three worst stations are congested continental hubs with modest Northline volume, while Manchester — the second-largest base — sits among the better performers. Every station-level chart in this report therefore uses a rate rather than a count.

**Cancellation rate is 0.59%** (270 of 45,886). ALC is weakest at 0.93%, TFS strongest at 0.24%.
**Load factor is 81.7%** across the period.

---

## Page 2 — Customer complaints

977 complaints, £172,360 paid in compensation.

### Complaint volume is not driven by punctuality

The correlation between monthly complaint volume and monthly OTP is **0.11** — effectively none. July 2025 is the highest complaint month of the 18 (77 complaints) but sits mid-table for punctuality, while the two February troughs that dominate the OTP story are unremarkable for complaints.

Breaking complaints down by the delay band of the flight they relate to confirms it:

| Departure delay | Complaints | Flights | Complaints per 1,000 |
|---|---|---|---|
| On time (≤15 min) | 742 | 35,591 | 20.8 |
| 16–60 min | 196 | 8,458 | 23.2 |
| 1–3 hours | 30 | 1,360 | 22.1 |
| 3+ hours | 1 | 80 | 12.5 |

The complaint rate barely moves with delay length. Three quarters of complaints come from flights that departed on time. Whatever is generating customer dissatisfaction here, departure punctuality is not the main driver — which matters, because it means an OTP improvement programme should not be expected to reduce complaint volume.

Complaint volume did fall over the period, averaging 56 a month in the first half of 2025 against 46.5 in the first half of 2026.

### Compensation exposure is volume-driven, not category-driven

| Category | Complaints | Compensation |
|---|---|---|
| Delay | 381 | £65,660 |
| Baggage | 195 | £33,720 |
| Booking | 125 | £23,140 |
| Refund | 107 | £18,760 |
| Cabin service | 103 | £18,110 |
| Seating | 66 | £12,970 |

Average payout is close to flat across categories (£172–£197) and the proportion of complaints attracting a payment sits near 50% throughout. Total compensation therefore tracks complaint volume almost exactly. Delay is the only category large enough to move the total.

### Resolution time is uniformly slow

Average resolution is **22.4 days**, median 22.5, maximum 44. It does not vary meaningfully by category — the spread across all six is 21.8 to 23.2 days.

That flatness is the finding. If resolution time varied by complaint type it would suggest some categories are harder to investigate; that it doesn't points to a capacity or process constraint affecting all case handling equally. A 22-day average is poor by industry standards and would be the first thing to target.

**Caveat:** 23 complaints have no resolution date recorded and are excluded from the average. If those are open cases rather than missing data, the true figure is worse.

### Station complaint rates vary 2.3x

NCL generates 32 complaints per 1,000 flights against a network low of 14 at BFS. As with delay, the ranking inverts when moving from counts to rates — LGW leads on raw complaint volume and sits mid-table on rate.

---

## Data quality

**1,106 of 9,898 delayed flights (11.2%) carry no delay code.** These are flights delayed more than 15 minutes where no cause was recorded at the station.

This is worth reporting in its own right. Unallocated delay means roughly one in nine delay events cannot be attributed, which puts a confidence band around the controllability split above. Any improvement programme targeting controllable delay is working from incomplete attribution.

---

## Methodology

### Cleaning (Power Query)

The source extract carried the problems typical of data assembled from several operational systems. Each was handled as a named applied step so the transformation history is auditable.

| Issue | Volume | Treatment |
|---|---|---|
| Duplicate flight records | 642 rows | `Table.Distinct` on `flight_id` |
| Airport codes with leading whitespace and mixed case | 90 distinct values for 18 airports | Trim + uppercase |
| Aircraft registrations missing hyphen or lowercased | ~1,850 rows | Conditional transform re-inserting the hyphen |
| Delay causes recorded as free text instead of code | 8 variants | Mapped to IATA codes via `Record.FieldOrDefault` |
| `-999` sentinel in delay columns | 91 rows | Replaced with `null` |
| Complaint channel casing (`Email`/`email`, `Web form`/`WEB FORM`) | 84 rows | Trim + proper case |
| Empty trailing column from CSV | — | Removed |

### Decisions worth explaining

**`-999` was nulled, not filtered.** Removing those rows would drop 91 real flights from every count. Replacing with null preserves the flight while excluding it from any average, which is the correct treatment for a sentinel value.

**`departure_date` was used as the date key, not `flight_date`.** The two disagree on 1,047 flights — not an error, but late-evening departures where the operating day and the actual departure timestamp fall either side of midnight. Since every measure concerns departure punctuality, the date dimension is aligned to the departure timestamp. `flight_date` also arrived in two formats (`dd/mm/yyyy` and `dd-MMM-yy`) and was dropped from the model to prevent accidental use.

**Complaints are dated by flight date, not submission date.** The complaints table has no date column of its own and inherits its date context through the relationship to `flights`. This means the time series shows which operating periods generated complaints, not when customers wrote in.

**Cancelled flights are excluded from OTP but included in cancellation rate.** A cancelled flight has no departure time and cannot be on or off time. Cancellation rate needs them in the denominator or the metric is meaningless. The two are reported separately rather than combined.

**Blank delay values are excluded from the OTP population.** DAX coerces `BLANK()` to zero in a comparison, so `departure_delay_mins <= 15` silently counts unrecorded flights as on time. An explicit `NOT ISBLANK()` filter removes 397 flights, giving 45,489 eligible. Without it OTP overstates by roughly 0.3 points.

**OTP15 and Delay Rate share a denominator.** Both calculate against a single `Eligible Flights` measure so they always sum to exactly 100%. An earlier version had the two drifting apart on their filter conditions; sharing the base makes that failure impossible.

**Station-level charts use rates, not counts.** Both delay and complaint counts rank almost perfectly by flight volume, which reproduces the route network rather than revealing performance. Every station chart uses a per-flight or per-1,000-flight rate.

**Time-series axes start at zero.** Complaint volume ranges 37–77 a month; on a truncated axis this reads as high volatility, on a zero-based axis as a stable band with a mild downward drift. The latter is the accurate impression.

### Data model

Star schema with `flights` as the fact table.

- `Date Table` — generated with `CALENDAR()`, marked as a date table, related to `flights[departure_date]`
- `complaints` — related to `flights` on `flight_id`, inheriting date context through it
- `delay_codes` — IATA-style reference providing `delay_category` and `controllability`
- `aircraft` — fleet register
- `airports` — station reference

`Month Year` is set to sort by a `yyyy-mm` column so time axes order chronologically rather than alphabetically.

### Core measures

```dax
Eligible Flights =
CALCULATE(
    COUNTROWS( flights ),
    flights[cancelled] = 0,
    NOT ISBLANK( flights[departure_delay_mins] )
)

OTP15 % =
VAR OnTime =
    CALCULATE(
        COUNTROWS( flights ),
        flights[cancelled] = 0,
        NOT ISBLANK( flights[departure_delay_mins] ),
        flights[departure_delay_mins] <= 15
    )
RETURN DIVIDE( OnTime, [Eligible Flights] )

Delay Rate % =
VAR Delayed =
    CALCULATE(
        COUNTROWS( flights ),
        flights[cancelled] = 0,
        NOT ISBLANK( flights[departure_delay_mins] ),
        flights[departure_delay_mins] > 15
    )
RETURN DIVIDE( Delayed, [Eligible Flights] )

Cancellation Rate % =
DIVIDE(
    CALCULATE( COUNTROWS( flights ), flights[cancelled] = 1 ),
    COUNTROWS( flights )
)

Complaints per 1000 Flights =
DIVIDE( COUNTROWS( complaints ), [Eligible Flights] ) * 1000
```

---

## Definitions

**OTP15** — share of operated flights departing within 15 minutes of schedule. The standard industry punctuality measure. Cancelled flights excluded.

**Delay Rate** — share of operated flights departing more than 15 minutes late. The exact complement of OTP15.

**Controllability** — whether the delay cause is within the airline's control (technical, crew, boarding, ramp handling), outside it (weather, ATC, airport congestion), or reactionary (knock-on from a late inbound aircraft).

**Load Factor** — passengers boarded as a share of seats available.

---

## Repository contents

| File | Description |
|---|---|
| `Airline data.pbix` | Power BI report, including all Power Query steps and DAX measures |

---

## Possible extensions

**Delay propagation.** The reactionary finding above is descriptive. Sequencing flights by aircraft and date would quantify the knock-on directly — how many downstream minutes an average morning delay generates, and at what turnaround buffer propagation stops. This is the most operationally useful question the dataset can answer and is the intended next iteration.

**Cost of delay.** The source data includes fuel, landing and handling costs at flight level, supporting a cost-per-delay-minute view and a return-on-investment case for additional turnaround buffer.

**Complaint driver analysis.** Punctuality explains very little of complaint volume. Testing against load factor, aircraft age, route and time of day would identify what actually does.
