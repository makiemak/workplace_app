# Running the app in Docker

The image builds cleanly with `docker build -t workplace_app .`, but running
it with plain `docker run -p 8080:8080 workplace_app` fails. This document
covers why, and the two steps needed to get a working container.

## Problem 1: the app crashes on startup with an empty `DATABASE_URI`

**Symptom:** every gunicorn worker fails to boot with:

```
sqlalchemy.exc.ArgumentError: Could not parse SQLAlchemy URL from string ''
```

**Cause:** [app.py](../app.py) reads the database URI like this:

```python
app.config["SQLALCHEMY_DATABASE_URI"] = os.environ.get("DATABASE_URI", "sqlite:///test.db")
```

`os.environ.get(..., default)` only returns the default when the key is
**completely absent** from the environment. The [Dockerfile](../Dockerfile)
declares `ARG DATABASE_URI` (no default value) and then always sets
`ENV DATABASE_URI=${DATABASE_URI}`. Since nothing supplies a build arg,
`DATABASE_URI` is baked into the image as an *empty string* — which counts
as "set" — so the app's sqlite fallback never kicks in and SQLAlchemy is
handed `''` as the connection URL.

**Fix:** pass `DATABASE_URI` (and `SECRET_KEY`, which has the same issue)
as a runtime environment variable when starting the container:

```bash
docker run -d --name workplace_app -p 8080:8080 \
  -e DATABASE_URI="sqlite:///test.db" \
  -e SECRET_KEY="dev-secret" \
  workplace_app
```

## Problem 2: routes 500 with "no such table"

**Symptom:** once the app boots, hitting any resource route
(e.g. `GET /team_members/`) returns a 500 with:

```
sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) no such table: team_members
```

**Cause:** the sqlite database is created fresh inside the container on
each run, but nothing creates the schema. Table creation/seeding is a
manual Flask CLI command ([controllers/cli_controller.py](../controllers/cli_controller.py)),
not something that runs automatically on boot.

**Fix:** run the CLI commands once against the running container:

```bash
docker exec workplace_app flask db create
docker exec workplace_app flask db seed   # optional, adds sample data
```

## Full working sequence

```bash
docker build -t workplace_app .
docker run -d --name workplace_app -p 8080:8080 \
  -e DATABASE_URI="sqlite:///test.db" \
  -e SECRET_KEY="dev-secret" \
  workplace_app
docker exec workplace_app flask db create
docker exec workplace_app flask db seed
```

Verified all seven resource routes (`/team_members/`, `/rosters/`,
`/projects/`, `/performance_reviews/`, `/departments/`, `/clients/`,
`/client_feedbacks/`) return `200` after these steps.

## Not fixed here, worth revisiting

The container's sqlite database lives inside the container's writable
layer, so data (and the schema itself) is lost every time the container is
recreated, requiring `flask db create`/`seed` to be re-run each time. If
this needs to persist across restarts, mount a volume for the sqlite file
or point `DATABASE_URI` at a real external database (e.g. Postgres via
`psycopg2-binary`, which is already a dependency).
