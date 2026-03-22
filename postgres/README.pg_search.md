# PostgreSQL pg_search Setup

This Laradock workspace installs ParadeDB's `pg_search` extension into the
`postgres` image during `docker compose build postgres`.

The repository keeps Laradock's legacy data mount layout and explicitly sets
`PGDATA=/var/lib/postgresql/data` for PostgreSQL 18. This avoids the startup
failure caused by PostgreSQL 18's new default data directory when used with the
existing Laradock volume mapping.

## What Changed

- Base image moved from floating `postgres:alpine` to pinned `postgres:18-bookworm`
- `PG_SEARCH_VERSION` was added as a build argument so the extension version can
  be upgraded from `.env`
- `PGDATA` is explicitly set to `/var/lib/postgresql/data` in
  `docker-compose.yml` to stay compatible with Laradock's current bind mount
- `init_pg_search.sql` creates the extension automatically on first database
  initialization

## Rebuild

```powershell
docker compose build --no-cache postgres
docker compose up -d postgres
```

## Existing Data Volumes

The `postgres/docker-entrypoint-initdb.d/init_pg_search.sql` script only runs
when PostgreSQL initializes a new data directory.

If your `${DATA_PATH_HOST}/postgres` volume already exists, connect manually and
create the extension yourself:

```sql
CREATE EXTENSION IF NOT EXISTS pg_search;
```

## Important Compatibility Note

Switching to `postgres:18-bookworm` still requires your current data directory
to already be a PostgreSQL 18 cluster. Setting `PGDATA` fixes the mount-path
problem only; it does not perform a PostgreSQL major-version upgrade.

If the existing volume was initialized by PostgreSQL 17 or below, migrate it
with `pg_upgrade` or restore from dump before starting the rebuilt container.
