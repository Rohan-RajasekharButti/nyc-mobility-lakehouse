# Analytics Query Pack — Results

Run against `gold.fct_taxi_trips` / `gold.dim_zone` / `gold.fct_airport_trips`, built from the real May 2024 NYC TLC yellow taxi trip file (3,618,292 clean trips after validation). Full queries live in [`sql/analytics_queries.sql`](../sql/analytics_queries.sql).

## 1. Peak pickup demand by hour and zone

Business question: *where and when should we position more vehicles?*

| zone_name             | borough   |   pickup_hour |   trips |
|:-----------------------|:----------|--------------:|--------:|
| Midtown Center        | Manhattan |            18 |   15124 |
| Midtown Center        | Manhattan |            17 |   14580 |
| Upper East Side South | Manhattan |            18 |   14232 |
| Upper East Side South | Manhattan |            17 |   14146 |
| Upper East Side North | Manhattan |            15 |   14144 |
| Upper East Side South | Manhattan |            15 |   14051 |
| Upper East Side South | Manhattan |            14 |   13913 |
| Upper East Side South | Manhattan |            16 |   13309 |
| Midtown Center        | Manhattan |            19 |   13144 |
| Upper East Side North | Manhattan |            18 |   12955 |

**Reading:** demand is overwhelmingly an evening-rush, Manhattan-core story — 4–7pm pickups in Midtown and the Upper East Side dominate every other zone/hour combination in the data.

## 2. Revenue per mile by route (min. 50 trips)

Business question: *which routes look the most profitable per mile?*

| pickup_location_id | dropoff_location_id | trips | revenue ($) | revenue/mile ($) |
|---:|---:|---:|---:|---:|
| 1 (Newark Airport) | 1 | 110 | 9,765.91 | 2,943.29 |
| 197 | 197 | 94 | 3,992.46 | 1,115.13 |
| 216 | 216 | 221 | 13,606.06 | 1,114.46 |
| 132 (JFK) | 132 | 4,306 | 191,350.81 | 727.82 |

**Reading — and the catch:** this metric is dominated by *same-zone* trips, where `trip_distance` is recorded as near-zero but the flat/negotiated fare is high (airport trips especially). Taken at face value this looks like "park a cab at zone 1 and print money," but it's really a metric artifact — short or zero-distance trips break a per-mile denominator. Production use of this query should exclude `pickup_location_id = dropoff_location_id` or floor `trip_distance` at a sane minimum before ranking routes. Flagging this kind of metric trap is more valuable in an interview than the number itself.

## 3. Fare outliers

Business question: *which trips look like meter errors or potential fraud?*

**29,805 trips** (0.82% of clean trips) have either `total_amount > $300` or an implied rate over $50/mile. These are flagged for manual review, not auto-deleted — a $300+ fare from JFK to a far Manhattan zone with bad traffic is plausible; a $300 fare for a 0.2-mile trip is not. The query returns full row detail so an analyst can triage rather than trusting a blanket cutoff.

## 4. Top zones by revenue

Business question: *where does the money actually come from?*

| zone_name | borough | trip_count | total_revenue ($) | avg_tip_% |
|:---|:---|---:|---:|---:|
| JFK Airport | Queens | 157,916 | 13,119,500 | 14.1% |
| LaGuardia Airport | Queens | 124,138 | 8,626,920 | 21.0% |
| Midtown Center | Manhattan | 170,773 | 4,405,430 | 20.2% |
| Upper East Side South | Manhattan | 179,968 | 3,806,180 | 21.6% |
| Upper East Side North | Manhattan | 165,296 | 3,580,100 | 21.1% |

**Reading:** JFK alone generates ~1.5x the revenue of the next-highest zone despite the flat-rate fare structure, and its average tip percentage (14.1%) is notably lower than every Manhattan zone — flat-rate airport fares appear to suppress tipping behavior relative to metered rides.

## 5. Airport trip mix

Business question: *how do the three airports compare operationally?*

| airport_code | trip_count | avg_fare ($) | avg_duration (min) | avg_distance (mi) |
|:---|---:|---:|---:|---:|
| JFK_LGA | 357,562 | 77.76 | 42.3 | 13.29 |
| EWR | 9,927 | 129.08 | 44.0 | 17.37 |

**Reading:** Newark trips are far less frequent (2.7% of airport volume) but cost ~66% more per trip on average — consistent with EWR's flat-fare-plus-tolls structure and longer average distance from Manhattan. JFK/LaGuardia are grouped here because both fall inside Queens pickup/dropoff zones in the source data; splitting them would need a finer-grained zone-to-airport mapping than `dim_zone` currently provides.
