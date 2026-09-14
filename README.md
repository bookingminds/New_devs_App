# Property Revenue Dashboard — debugging notes

Branch `solution`, one fix per commit, backend only. The frontend is untouched.

## What I did

| # | Symptom | Root cause | Fix |
|---|---------|------------|-----|
| 1 | Totals don't match the clients' records | `backend/app/core/database_pool.py` built the DB URL from settings fields that don't exist (`supabase_db_*`), used `QueuePool` (not allowed with the async engine) and `get_session` was `async def`, so `async with` got a coroutine. The pool never worked and every request fell back to hardcoded mock numbers. | Build the URL from `settings.database_url` with the asyncpg driver, drop `QueuePool`, make `get_session` a plain method. |
| 2 | Ocean Rentals sometimes sees another company's revenue | `backend/app/services/cache.py` cached under `revenue:{property_id}`. Property ids repeat across tenants (`prop-001` exists for both), so the first caller filled the cache and the other tenant read it for 5 minutes. | Key is now `revenue:{tenant_id}:{property_id}:{period}`. |
| 3 | Totals "a few cents off" | `backend/app/api/v1/dashboard.py` returned `float(total)`; amounts are stored as `NUMERIC(10,3)`, so sub-cent values reached the client and there was no rounding policy. | Round once, at the API boundary, with `Decimal.quantize(0.01, ROUND_HALF_UP)`. |
| 4 | March totals differ from the client's books | Monthly revenue was a stub returning 0 with naive UTC month boundaries. `res-tz-1` checks in at `2024-02-29 23:30 UTC`, which is `2024-03-01 00:30` in Europe/Paris: March in the client's books, February in UTC. | `calculate_total_revenue` takes optional `month`/`year` and filters on `check_in_date AT TIME ZONE properties.timezone`. Exposed as `?month=&year=` on `/api/v1/dashboard/summary` (both or neither, else 400). |
| 5 | Wrong numbers were invisible for weeks | On any DB error the service returned a fixed table of numbers, identical for every tenant. | Log and re-raise. A visible 500 beats a plausible wrong figure on a board report. Nothing is cached on failure. |
| 6 | (found while fixing 1) | A new engine with a 20-connection pool was created on every request. | Reuse the process-wide `db_pool`. |

## How to run

```bash
docker-compose up --build
# Frontend: http://localhost:3000   Backend docs: http://localhost:8000/docs
```

Quick API check (replace the token from the login response):

```bash
curl -s -X POST http://localhost:8000/api/v1/auth/login -H 'Content-Type: application/json' \
  -d '{"email":"ocean@propertyflow.com","password":"client_b_2024"}'
curl -s 'http://localhost:8000/api/v1/dashboard/summary?property_id=prop-001' -H "Authorization: Bearer $TOKEN"
curl -s 'http://localhost:8000/api/v1/dashboard/summary?property_id=prop-001&month=3&year=2024' -H "Authorization: Bearer $TOKEN"
```

## How I verified

Expected values come straight from `database/seed.sql`:

| Client | Property | All-time | March 2024 |
|--------|----------|----------|------------|
| Sunset (tenant-a) | prop-001 | 2250.00 / 4 | 2250.00 / 4 (includes `res-tz-1`, 1 March in Paris) |
| Sunset (tenant-a) | prop-002 | 4975.50 / 4 | 4975.50 / 4 |
| Sunset (tenant-a) | prop-003 | 6100.50 / 2 | 6100.50 / 2 |
| Ocean (tenant-b) | prop-001 | 0.00 / 0 | 0.00 / 0 |
| Ocean (tenant-b) | prop-004 | 1776.50 / 4 | 1776.50 / 4 |
| Ocean (tenant-b) | prop-005 | 3256.00 / 3 | 3256.00 / 3 |

Before the fixes both clients got the same mock numbers for every property (prop-001 = 1000.00 / 3).
After fix 1 Ocean still saw Sunset's totals (the cache leak); after fix 2 each tenant sees only its own.
Precision was checked with a temporary `100.005` reservation: the API returned `100.005` before and `100.01` after.
February 2024 returns 0 for every property, because the only late-February check-in is March in the property's timezone.
Stopping the `db` container now yields HTTP 500 instead of invented revenue.

## Assumptions

- "Revenue for March" means reservations whose **check-in** falls in March **in the property's timezone**. No nightly proration across month boundaries.
- Rounding policy is half-up to whole cents, applied once at the API boundary. Storage keeps 3 decimals as designed.
- The UI has no month selector, so the month filter lives on the API (see `/docs`); I kept the frontend out of the diff on purpose.
- Single currency (USD) as in the seed data; no FX conversion.

## With more time

- pytest coverage for the four behaviours: tenant isolation of the cache, rounding, timezone month boundary, no silent fallback.
- A month selector in the dashboard UI.
- Invalidate the revenue cache on reservation writes instead of relying on the 5-minute TTL.
- Remove the `default_tenant` fallback in `dashboard.py` and reject requests with no resolved tenant.
- Return money as a string or integer cents end to end.

## Tooling

I used Claude Code as a pair programmer under my direction, one change at a time, and verified each change against the running stack before committing. I can walk through every line.
