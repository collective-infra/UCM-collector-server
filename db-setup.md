# UCM Sensor Database Setup (PostgreSQL + TimescaleDB)

How to rebuild the database that stores sensor readings.

Data flow: `sensors → MQTT → Telegraf → PostgreSQL/TimescaleDB → Grafana`

| Component | Role in this setup |
|-----------|--------------------|
| Database `ucm` | Holds all tables below |
| `environment_readings` | TimescaleDB hypertable with one row per sensor reading |
| `sensors` | Metadata per sensor (display name, location, contact) |
| Role `telegraf` | Writes readings. `INSERT` on `environment_readings` only |
| Role `grafana` | Reads data. `SELECT` on `environment_readings` and `sensors` only |
| Role `postgres` | Admin and management only |

## 1. Prerequisites

- PostgreSQL and the TimescaleDB extension packages are installed.
- `shared_preload_libraries = 'timescaledb'` is set in `postgresql.conf` (`timescaledb-tune` does this), and PostgreSQL was restarted afterwards.

All commands below run in a `psql` session as the admin user:

```bash
sudo -u postgres psql
```

## 2. Create the database and enable TimescaleDB

```sql
CREATE DATABASE ucm;
\c ucm

CREATE EXTENSION IF NOT EXISTS timescaledb;
```

## 3. Create the tables

Run these while connected to `ucm` as `postgres`, so `postgres` owns the tables.

### `environment_readings`

```sql
CREATE TABLE public.environment_readings (
    "time"            timestamptz NOT NULL,
    node_id           text        NOT NULL,
    temperature_c     float8,
    humidity_percent  float8,
    pm1               float8,
    pm25              float8,
    pm10              float8,
    voc               float8,
    received_at       timestamptz NOT NULL DEFAULT now(),
    pm4               float8
);

-- Also creates the default index environment_readings_time_idx ("time" DESC)
SELECT create_hypertable('public.environment_readings', 'time');
```

Notes:

- `time` is the timestamp reported by the sensor. `received_at` is filled in by the database on insert.
- There is deliberately no primary key. `time` alone is not unique, and hypertables require unique indexes to include the partitioning column.

### `sensors`

```sql
CREATE TABLE public.sensors (
    node_id       text PRIMARY KEY,
    display_name  text NOT NULL,
    lat           double precision,
    lon           double precision,
    contact_point text,
    note          text,
    created_at    timestamptz NOT NULL DEFAULT now()
);
```

Register sensors (replace the placeholders; `lat`, `lon` and `contact_point` may be `NULL`):

```sql
INSERT INTO public.sensors (node_id, display_name, lat, lon, contact_point) VALUES
    ('UCM-XXXXXX', 'Human readable name', NULL, NULL, NULL);
```

Update a sensor later:

```sql
UPDATE public.sensors SET display_name = 'New Name' WHERE node_id = 'UCM-XXXXXX';
```

### Optional: foreign key from readings to sensors

Skip this if sensors may start publishing before they are registered in `sensors`.

With the foreign key, Postgres rejects readings from any `node_id` that is not in `sensors`. Telegraf logs a permanent error and drops those rows, so the data is lost. Without it, the Grafana queries still work, because they use a `LEFT JOIN` and fall back to the raw `node_id` as the legend label.

```sql
ALTER TABLE public.environment_readings
    ADD CONSTRAINT environment_readings_node_id_fkey
    FOREIGN KEY (node_id) REFERENCES public.sensors (node_id);
```

The `telegraf` role does not need any privileges on `sensors` for the check to work.

## 4. Create roles and permissions

Generate strong passwords first, for example `openssl rand -base64 24`, and replace the placeholders. Do not commit real passwords to git.

```sql
CREATE ROLE telegraf LOGIN PASSWORD 'CHANGE_ME_TELEGRAF';
CREATE ROLE grafana  LOGIN PASSWORD 'CHANGE_ME_GRAFANA';

-- Only the two service roles (and superusers) may connect to ucm
REVOKE ALL ON DATABASE ucm FROM PUBLIC;
GRANT CONNECT ON DATABASE ucm TO telegraf, grafana;

-- Nobody but the owner creates objects in public (already the default on PostgreSQL 15+)
REVOKE CREATE ON SCHEMA public FROM PUBLIC;
GRANT USAGE ON SCHEMA public TO telegraf, grafana;

-- Telegraf: insert only
GRANT INSERT ON public.environment_readings TO telegraf;

-- Grafana: read only
GRANT SELECT ON public.environment_readings, public.sensors TO grafana;
```

Grants on the hypertable apply to its chunks automatically. If you add new tables later, grant access explicitly, because privileges are per table.

## 5. Restrict where users can connect from (`pg_hba.conf`)

Copy the ACL settings from `postgres/pg_hba.conf` into `/etc/postgresql/18/main/pg_hba.conf`

Apply the changes. `pg_hba.conf` only needs a reload, but a change to `listen_addresses` needs a restart:

```bash
sudo systemctl reload postgresql     # pg_hba.conf changes
sudo systemctl restart postgresql    # if listen_addresses changed
```

Check that PostgreSQL parsed the rules and found no errors:

```sql
SELECT line_number, type, database, user_name, address, auth_method, error
FROM pg_hba_file_rules;
```

## 6. Verify permissions

From the `postgres` session, switch roles to test without needing passwords:

```sql
\c ucm

-- telegraf: INSERT works, SELECT / DELETE fail
SET ROLE telegraf;
INSERT INTO public.environment_readings ("time", node_id, pm25)
    VALUES (now(), 'UCM-TEST', 1.0);                 -- works
SELECT * FROM public.environment_readings LIMIT 1;   -- permission denied
RESET ROLE;

-- grafana: SELECT works, INSERT / DELETE fail
SET ROLE grafana;
SELECT * FROM public.environment_readings LIMIT 1;   -- works
SELECT * FROM public.sensors LIMIT 1;                -- works
INSERT INTO public.sensors (node_id, display_name)
    VALUES ('UCM-TEST', 'x');                        -- permission denied
RESET ROLE;

-- Remove the test row (as postgres)
DELETE FROM public.environment_readings WHERE node_id = 'UCM-TEST';
```

Summary of the granted privileges:

```sql
SELECT grantee, table_name, privilege_type
FROM information_schema.role_table_grants
WHERE grantee IN ('telegraf', 'grafana')
ORDER BY grantee, table_name, privilege_type;
```

Expected result:

| grantee | table | privileges |
|---------|-------|------------|
| telegraf | environment_readings | INSERT |
| grafana | environment_readings | SELECT |
| grafana | sensors | SELECT |

Also test from the real client hosts, so the `pg_hba.conf` rules are exercised too:

```bash
psql "host=DB_HOST dbname=ucm user=telegraf" -c "SELECT 1"
psql "host=DB_HOST dbname=ucm user=grafana"  -c "SELECT 1"
```

## 7. Client configuration

**Telegraf** (`[[outputs.postgresql]]`):
Update the password to match the one set in the DB for the telegraf user

```toml
[[outputs.postgresql]]
  connection = "postgres://telegraf:<password>@localhost:5432/ucm?sslmode=disable"
  schema     = "public"
```

The `telegraf` role cannot create or alter tables. The existing table must already match the metric's tags and fields. If a new field is ever added to the MQTT payload, add the column manually as `postgres`:

```sql
ALTER TABLE public.environment_readings ADD COLUMN new_field float8;
```

If Telegraf logs `permission denied` on a `SELECT` against `environment_readings` after the migration, check that first. Granting `SELECT` to `telegraf` is the fallback, at the cost of the strict insert-only rule.

