# Sensor Readings Service

A REST service that accepts temperature readings from field devices, stores them,
and lets an operator ask what a device has reported and how it has been behaving.

Python 3.11, FastAPI, pytest. Deployed to DigitalOcean App Platform from the Dockerfile.

## A reading

    {
      "reading_id": "a1b2c3",          // unique, chosen by the device
      "device_id": "device-42",
      "timestamp": "2026-09-20T14:03:00Z",
      "temperature_c": 21.5
    }

## Endpoints

| Method | Path                                | Purpose                                    | Success | Errors |
|--------|-------------------------------------|--------------------------------------------|---------|--------|
| GET    | /healthz                            | Platform health check                      | 200     |        |
| POST   | /v1/readings                        | Accept one reading                         | 201     | 422 invalid |
| GET    | /v1/devices/{device_id}/readings    | That device's readings, newest first       | 200     | 404 unknown device |
| GET    | /v1/devices/{device_id}/stats       | count, min, max, avg temperature           | 200     | 404 unknown device |

Later, if time: POST /v1/readings/batch (devices send bursts after being offline).

## Validation rules

Derived from the brief: devices resend, send garbage when a sensor fails, and have bad clocks.

- All four fields required, correct types. (FastAPI/Pydantic handles this: 422.)
- `device_id` and `reading_id`: non-empty strings.
- `temperature_c`: between -90 and 60. Outside that is a broken sensor, reject with 422.
- `timestamp`: must parse as ISO 8601 and must not be in the future. Bad clock, reject with 422.
- Duplicate `reading_id`: accept the request (200 or 201) but do not store it twice.
  Devices resend; punishing them for it would just make them retry again.

Every error response is JSON: `{"error": "<plain message>"}`.

## Storage

SQLite from the start, using Python's built-in `sqlite3`, behind a small `Storage`
class in `app/storage.py`. One table, `readings`, with `reading_id` as primary key.
Nothing outside that file knows how data is stored. Database path comes from config.

Why: persistence is real locally and across process restarts, and duplicate handling
falls out of the primary key. Trade-off: App Platform's disk is ephemeral, so the file
does not survive a redeploy, and two containers would not share it. Production answer
is managed Postgres, which is a swap inside storage.py. Stated in README.

## File layout

    app/main.py        routes only, no logic
    app/models.py      Pydantic schemas and validators
    app/storage.py     Storage class (SQLite via sqlite3)
    app/config.py      settings from environment variables with defaults
    tests/             pytest, using FastAPI's TestClient
    Dockerfile, requirements.txt, README.md

## Order of work

1. Skeleton: layout above, /healthz, Dockerfile. Run locally, push, deploy, green.
2. POST /v1/readings with validation, stored in SQLite. Duplicate reading_id is a no-op success.
3. GET readings for a device.
4. GET stats for a device.
5. Tests: happy path, three validation failures, 404 for unknown device.
6. README.
7. Then, in order: env-var config, request logging, batch endpoint.

## Rules for the AI

- One step at a time. Stop after each step so I can read and run it.
- Do not add auth, an ORM, background workers, or anything not listed.
- Small dependencies only: fastapi, uvicorn, pydantic, pytest, httpx.
