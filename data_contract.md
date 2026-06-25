# Data Contract — NYC Taxi Trips

## Grain

One row per taxi trip.

## Required Fields

| Column | Required | Rule |
|---|---:|---|
| pickup_ts | Yes | Must be timestamp |
| dropoff_ts | Yes | Must be greater than pickup_ts |
| pickup_ts | Yes | Must fall within +/- 1 day of the ingestion batch's month (catches meter clock errors) |
| pickup_location_id | Yes | Must exist in zone dimension |
| dropoff_location_id | Yes | Must exist in zone dimension |
| trip_distance | Yes | Must be > 0 |
| fare_amount | Yes | Must be >= 0 |
| total_amount | Yes | Must be >= 0 |

## SLA

- Monthly files should be processed within 2 hours of availability.
- Gold marts should be refreshed after all quality rules pass.
- Failed batches should be safe to rerun.
