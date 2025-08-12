# PostgreSQL Database Restore Guide

## Overview
This guide covers the complete process to restore a PostgreSQL database from an SQL dump file when the target database is actively being used by other connections.

## Prerequisites
- PostgreSQL superuser access (postgres user)
- SQL dump file location: `/tmp/designdb_master.sql`
- Target database: `designdb_master`

## Step 1: Check Current Database Status

### Check who's connected to the database:
```sql
psql -h localhost -U postgres -c "
SELECT pid, usename, application_name, client_addr, state
FROM pg_stat_activity 
WHERE datname = 'designdb_master';
"
```

### Verify current database owner:
```sql
psql -h localhost -U postgres -c "
SELECT datname, pg_catalog.pg_get_userbyid(datdba) as owner 
FROM pg_catalog.pg_database 
WHERE datname = 'designdb_master';
"
```

## Step 2: Terminate Active Connections

### Connect as postgres superuser and kill connections:
```bash
psql -h localhost -U postgres
```

### Inside psql, run:
```sql
-- Terminate all connections to the target database
SELECT pg_terminate_backend(pg_stat_activity.pid)
FROM pg_stat_activity
WHERE pg_stat_activity.datname = 'designdb_master'
  AND pid <> pg_backend_pid();

-- Exit psql
\q
```

### Alternative one-liner method:
```bash
psql -h localhost -U postgres -c "
SELECT pg_terminate_backend(pg_stat_activity.pid) 
FROM pg_stat_activity 
WHERE pg_stat_activity.datname = 'designdb_master' 
  AND pid <> pg_backend_pid();
"
```

## Step 3: Drop and Recreate Database

### Drop the existing database:
```bash
dropdb -h localhost -U postgres designdb_master
```

### Create a fresh database:
```bash
createdb -h localhost -U postgres designdb_master
```

## Step 4: Restore from SQL Dump

### Restore the database:
```bash
psql -h localhost -U postgres -d designdb_master < /tmp/designdb_master.sql
```

### Alternative with error checking:
```bash
psql -h localhost -U postgres -d designdb_master -v ON_ERROR_STOP=1 < /tmp/designdb_master.sql
```

### Alternative with progress output:
```bash
psql -h localhost -U postgres -d designdb_master -f /tmp/designdb_master.sql
```

## Step 5: Verify Restore

### Check that tables were restored:
```bash
psql -h localhost -U postgres -d designdb_master -c "\dt"
```

### Check database size:
```bash
psql -h localhost -U postgres -c "
SELECT datname, pg_size_pretty(pg_database_size(datname)) as size
FROM pg_database 
WHERE datname = 'designdb_master';
"
```

## Complete One-Liner Process

For a quick execution, you can combine steps 2-4:

```bash
psql -h localhost -U postgres -c "SELECT pg_terminate_backend(pg_stat_activity.pid) FROM pg_stat_activity WHERE pg_stat_activity.datname = 'designdb_master' AND pid <> pg_backend_pid();" && dropdb -h localhost -U postgres designdb_master && createdb -h localhost -U postgres designdb_master && psql -h localhost -U postgres -d designdb_master < /tmp/designdb_master.sql
```

## Alternative: Restore to New Database

If you prefer to keep the original database until restore is complete:

### Create new database with different name:
```bash
createdb -h localhost -U postgres designdb_master_new
```

### Restore to new database:
```bash
psql -h localhost -U postgres -d designdb_master_new < /tmp/designdb_master.sql
```

### Later, when ready to switch:
```sql
-- Terminate connections to old database
SELECT pg_terminate_backend(pg_stat_activity.pid)...

-- Rename databases
ALTER DATABASE designdb_master RENAME TO designdb_master_old;
ALTER DATABASE designdb_master_new RENAME TO designdb_master;
```

## Troubleshooting

### If connections persist:
- Check for persistent connections from applications
- Restart application services that connect to the database
- Use `pg_stat_activity` to identify stubborn connections

### If restore fails:
- Check the SQL dump file for syntax errors: `head -50 /tmp/designdb_master.sql`
- Verify file permissions: `ls -la /tmp/designdb_master.sql`
- Check disk space: `df -h /tmp`

### Permission errors:
- Ensure you're running as postgres superuser
- Check if the dump contains ownership commands that conflict

## Notes
- Always backup important data before performing database drops
- Consider scheduling this during maintenance windows
- Test the restore process in a development environment first
- Monitor application logs after restore for any issues
