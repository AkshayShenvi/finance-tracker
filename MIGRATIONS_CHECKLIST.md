# Database Migrations Checklist

## Overview

This document lists all SQL migrations required for the Finance Tracker application. When using Docker Compose, **these run automatically** - no manual action needed!

## Migration Files (In Order)

### 1. Base Tables (Automatic via SQLAlchemy)
Located in: `finance-tracker-backend/app/models/`

These tables are created automatically when the backend starts:
- ✅ `users` - User authentication
- ✅ `accounts` - Financial accounts
- ✅ `transactions` - Income/expense/transfer records
- ✅ `categories` - Transaction categories

**Action Required**: None - handled by `app/main.py:8` via `Base.metadata.create_all()`

---

### 2. Migration 001: Import Tracking System
**File**: `finance-tracker-backend/migrations/001_add_import_tracking.sql`
**Date**: 2026-02-17

**Creates**:
- `import_sessions` table - Tracks CSV import history
- `category_rules` table - Auto-categorization rules
- `bank_parser_templates` table - Custom parser configurations

**Modifies**:
- `transactions` table: Adds `import_session_id`, `raw_data` (JSONB)
- `accounts` table: Adds `bank_name`, `account_number_last4`

**Purpose**: Enables CSV import tracking and intelligent categorization

---

### 3. Migration 002: Default Parser
**File**: `finance-tracker-backend/migrations/002_add_default_parser_to_accounts.sql`
**Date**: 2026-02-18

**Modifies**:
- `accounts` table: Adds `default_parser` (VARCHAR 50)

**Purpose**: Links accounts to their preferred CSV parser for easier imports

---

### 4. Migration 003: Balance Snapshots
**File**: `finance-tracker-backend/migrations/003_add_balance_snapshots.sql`
**Date**: 2026-02-19

**Creates**:
- `account_balance_snapshots` table - Historical balance tracking

**Indexes**:
- `user_id`, `account_id`, `snapshot_date`, `period` indexes
- Unique constraint on `(account_id, snapshot_date, snapshot_type)`

**Purpose**: Net worth tracking and historical balance analysis

---

## Automated Migration System

### How It Works

When you run `docker-compose up -d`:

1. **PostgreSQL starts** and becomes healthy
2. **Backend container starts** and runs `scripts/entrypoint.sh`
3. **Migration script runs** (`scripts/run_migrations.sh`):
   - Waits for database to be ready
   - Creates `schema_migrations` tracking table
   - Checks which migrations have been applied
   - Runs pending migrations in alphabetical order
   - Records successful migrations
   - Starts FastAPI application

### Migration Tracking Table

```sql
CREATE TABLE schema_migrations (
  id SERIAL PRIMARY KEY,
  version VARCHAR(255) UNIQUE NOT NULL,
  applied_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);
```

Example contents after all migrations:
```
 id |              version              |         applied_at
----+-----------------------------------+----------------------------
  1 | 001_add_import_tracking           | 2026-02-20 10:30:15.123456
  2 | 002_add_default_parser_to_accounts| 2026-02-20 10:30:15.456789
  3 | 003_add_balance_snapshots         | 2026-02-20 10:30:15.789012
```

---

## Verification Commands

### Check Migration Status
```bash
docker exec -it finance-tracker-db psql -U financeuser -d financedb -c "SELECT * FROM schema_migrations ORDER BY id;"
```

### View All Tables
```bash
docker exec -it finance-tracker-db psql -U financeuser -d financedb -c "\dt"
```

### View Specific Table Schema
```bash
docker exec -it finance-tracker-db psql -U financeuser -d financedb -c "\d transactions"
```

### Check Migration Logs
```bash
docker logs finance-tracker-backend | grep -i migration
```

---

## Manual Migration (If Needed)

If automated migrations fail, run manually in order:

```bash
# After docker-compose up -d and database is ready

# Migration 1
docker exec -i finance-tracker-db psql -U financeuser -d financedb < finance-tracker-backend/migrations/001_add_import_tracking.sql

# Migration 2
docker exec -i finance-tracker-db psql -U financeuser -d financedb < finance-tracker-backend/migrations/002_add_default_parser_to_accounts.sql

# Migration 3
docker exec -i finance-tracker-db psql -U financeuser -d financedb < finance-tracker-backend/migrations/003_add_balance_snapshots.sql
```

---

## Adding New Migrations

1. **Create new file**: `finance-tracker-backend/migrations/004_description.sql`
2. **Use idempotent syntax**:
   ```sql
   CREATE TABLE IF NOT EXISTS ...
   ALTER TABLE ... ADD COLUMN IF NOT EXISTS ...
   CREATE INDEX IF NOT EXISTS ...
   ```
3. **Test locally**:
   ```bash
   docker-compose down -v  # Reset database
   docker-compose up -d    # Should apply all migrations
   ```
4. **Verify**:
   ```bash
   docker exec -it finance-tracker-db psql -U financeuser -d financedb -c "SELECT * FROM schema_migrations;"
   ```

---

## Troubleshooting

### Migrations Not Running
**Symptom**: Backend starts but schema is missing columns/tables

**Fix**:
1. Check logs: `docker logs finance-tracker-backend`
2. Verify database is ready: `docker exec finance-tracker-db pg_isready -U financeuser`
3. Restart backend: `docker-compose restart backend`
4. Manual run if needed (see above)

### "Table Already Exists" Errors
**Symptom**: Migration script fails with duplicate table error

**Fix**:
- If migrations were run manually before automation, manually insert records:
  ```bash
  docker exec -it finance-tracker-db psql -U financeuser -d financedb
  INSERT INTO schema_migrations (version) VALUES ('001_add_import_tracking');
  INSERT INTO schema_migrations (version) VALUES ('002_add_default_parser_to_accounts');
  INSERT INTO schema_migrations (version) VALUES ('003_add_balance_snapshots');
  ```

### Migration Script Fails Mid-Way
**Symptom**: Some migrations applied, some didn't

**Fix**:
1. Check what's applied: `SELECT * FROM schema_migrations;`
2. Manually run failed migration
3. System will skip already-applied ones

### Clean Slate Reset
```bash
docker-compose down -v  # Deletes all data and volumes
docker-compose up -d    # Fresh start with all migrations
```

---

## Production Considerations

1. **Backup before migrations**: Always backup production data
2. **Test migrations**: Run on staging environment first
3. **Downtime planning**: Some migrations may require brief downtime
4. **Rollback plan**: Know how to rollback each migration
5. **Monitor logs**: Watch migration progress in production

---

## Summary

✅ **3 SQL migrations** total
✅ **Automated execution** on container startup
✅ **Idempotent** - safe to re-run
✅ **Tracked** via `schema_migrations` table
✅ **Zero manual steps** required with Docker Compose

For detailed migration documentation, see:
- `finance-tracker-backend/migrations/README.md` - Migration runner details
- `SETUP.md` - Quick start guide
- `CLAUDE.md` - Developer reference
