# Energy Monitor Database Schema (PostgreSQL)

This database container uses PostgreSQL. The live database was initialized/updated by executing **one SQL statement at a time** via the CLI connection command in:

- `energy_monitor_database/db_connection.txt`

Connection (example):
- `psql postgresql://appuser:dbuser123@localhost:5000/myapp`

> Note: Per container rules, schema/seed were applied via CLI and **no .sql migration files** were created.

---

## Entities

### `app_users`
Authentication/users table.

Columns:
- `id uuid PK default gen_random_uuid()`
- `email text unique not null`
- `password_hash text not null`
- `full_name text null`
- `is_active boolean not null default true`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`
- `last_login_at timestamptz null`

Constraints/Indexes:
- `UNIQUE(email)`
- `updated_at` maintained by trigger `tr_app_users_updated_at`

---

### `auth_sessions`
Optional refresh-token/session persistence (useful if backend implements refresh tokens).

Columns:
- `id uuid PK default gen_random_uuid()`
- `user_id uuid not null -> app_users(id) on delete cascade`
- `refresh_token_hash text unique not null`
- `user_agent text null`
- `ip_address text null`
- `expires_at timestamptz not null`
- `revoked_at timestamptz null`
- `created_at timestamptz not null default now()`

Indexes:
- `idx_auth_sessions_user_id (user_id)`
- `idx_auth_sessions_expires_at (expires_at)`

---

### `devices`
User-owned devices/meters/plugs.

Columns:
- `id uuid PK default gen_random_uuid()`
- `user_id uuid not null -> app_users(id) on delete cascade`
- `name text not null`
- `location text null`
- `model text null`
- `manufacturer text null`
- `serial_number text null`
- `external_device_id text null`
- `timezone text not null default 'UTC'`
- `is_active boolean not null default true`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Constraints/Indexes:
- `UNIQUE(user_id, name)` (device names unique per user)
- `idx_devices_user_id (user_id)`
- `updated_at` maintained by trigger `tr_devices_updated_at`

---

### `energy_readings`
Time-series readings for devices.

Columns:
- `id bigserial PK`
- `user_id uuid not null -> app_users(id) on delete cascade`
- `device_id uuid not null -> devices(id) on delete cascade`
- `ts timestamptz not null`
- `power_w double precision null`
- `voltage_v double precision null`
- `current_a double precision null`
- `energy_wh double precision null`
- `source text not null default 'device'`
- `created_at timestamptz not null default now()`

Constraints/Indexes:
- `chk_non_negative_energy_wh` (energy_wh is null or >= 0)
- `chk_non_negative_power_w` (power_w is null or >= 0)
- `uq_energy_readings_device_ts UNIQUE(device_id, ts)` (prevents duplicates; supports time-range queries)
- `idx_energy_readings_user_ts (user_id, ts desc)` (per-user dashboards over time)
- `idx_energy_readings_device_ts_desc (device_id, ts desc)` (per-device history/latest)

> Note: `user_id` is stored redundantly (in addition to device ownership) to simplify/accelerate per-user filtering and enforce isolation patterns in the backend.

---

### `alerts`
Alert definitions/rules.

Columns:
- `id uuid PK default gen_random_uuid()`
- `user_id uuid not null -> app_users(id) on delete cascade`
- `device_id uuid null -> devices(id) on delete cascade`
- `name text not null`
- `alert_type text not null` in: `threshold | anomaly | offline`
- `metric text not null` (e.g. `power_w`)
- `comparison text not null` in: `gt | gte | lt | lte | eq | neq`
- `threshold double precision null`
- `window_seconds integer null` (optional evaluation window)
- `severity text not null default 'medium'` in: `low | medium | high | critical`
- `is_enabled boolean not null default true`
- `cooldown_seconds integer not null default 300`
- `created_at timestamptz not null default now()`
- `updated_at timestamptz not null default now()`

Constraints/Indexes:
- `UNIQUE(user_id, name)` (alert names unique per user)
- `idx_alerts_user_id (user_id)`
- `idx_alerts_device_id (device_id)`
- `updated_at` maintained by trigger `tr_alerts_updated_at`

---

### `alert_events`
Alert event history log (triggered/ack/resolved).

Columns:
- `id bigserial PK`
- `user_id uuid not null -> app_users(id) on delete cascade`
- `alert_id uuid not null -> alerts(id) on delete cascade`
- `device_id uuid null -> devices(id) on delete set null`
- `ts timestamptz not null default now()`
- `status text not null default 'triggered'` in: `triggered | acknowledged | resolved | suppressed`
- `message text null`
- `metric_value double precision null`
- `acknowledged_at timestamptz null`
- `resolved_at timestamptz null`
- `created_at timestamptz not null default now()`

Indexes:
- `idx_alert_events_user_ts (user_id, ts desc)`
- `idx_alert_events_alert_ts (alert_id, ts desc)`
- `idx_alert_events_device_ts (device_id, ts desc)`

---

## Utility objects

### Trigger function
- `set_updated_at()` used by triggers on `app_users`, `devices`, `alerts` to keep `updated_at` current.

### Debug view
- `v_devices_scoped` (joins `devices` with user email) for local inspection.

---

## Seed data (demo)

Seeded minimal demo data for local dev/demo:

User:
- `demo@example.com` (id `11111111-1111-1111-1111-111111111111`)

Devices:
- `Living Room Plug` (id `22222222-2222-2222-2222-222222222222`)
- `Office Meter` (id `33333333-3333-3333-3333-333333333333`)

Energy readings:
- a few sample readings across the last ~hour for chart/demo

Alerts:
- `High Power Living Room` threshold alert on `power_w > 180`

Alert events:
- one sample `triggered` event

All inserts are idempotent where appropriate via `ON CONFLICT DO NOTHING`.
