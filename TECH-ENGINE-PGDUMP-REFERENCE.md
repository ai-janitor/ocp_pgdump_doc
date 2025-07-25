# PostgreSQL Dump and Restore - Technical Reference

## Quick Commands

### Backup (pg_dump)
```bash
# Basic dump
pg_dump -h localhost -U postgres -d database_name > backup.sql

# Compressed dump
pg_dump -h localhost -U postgres -d database_name | gzip > backup.sql.gz

# Custom format (faster restore)
pg_dump -h localhost -U postgres -d database_name -Fc > backup.dump

# Schema only
pg_dump -h localhost -U postgres -d database_name --schema-only > schema.sql

# Data only
pg_dump -h localhost -U postgres -d database_name --data-only > data.sql
```

### Restore (psql/pg_restore)
```bash
# From SQL dump
psql -h localhost -U postgres -d target_database < backup.sql

# From compressed SQL
gunzip -c backup.sql.gz | psql -h localhost -U postgres -d target_database

# From custom format
pg_restore -h localhost -U postgres -d target_database backup.dump

# Clean restore (drop/recreate)
pg_restore -h localhost -U postgres -d target_database --clean --create backup.dump
```

## OCP Pod Context

### In Pod Commands
```bash
# Connect to pod
oc exec -it pod-name -- bash

# Inside pod: Create dump
pg_dump -h localhost -U postgres -d dbname > /tmp/backup.sql

# Inside pod: Restore
psql -h localhost -U postgres -d dbname < /tmp/backup.sql

# Exit pod
exit
```

### Pod File Operations
```bash
# Copy dump from pod to local
oc cp pod-name:/tmp/backup.sql ./backup.sql

# Copy dump from local to pod
oc cp ./backup.sql pod-name:/tmp/backup.sql

# Run dump command directly (no shell)
oc exec pod-name -- pg_dump -h localhost -U postgres -d dbname > /tmp/backup.sql
```

## Complete Dump Options

### Connection Options
```bash
-h, --host=HOSTNAME      database server host (default: localhost)
-p, --port=PORT          database server port (default: 5432)
-U, --username=NAME      connect as specified database user
-d, --dbname=DBNAME      database to dump
-W, --password           force password prompt
```

### Output Options
```bash
-f, --file=FILENAME      output file name
-F, --format=c|d|t|p     output file format (custom, directory, tar, plain text)
-c, --clean              clean (drop) database objects before recreating
-C, --create             include commands to create database
-n, --schema=SCHEMA      dump only the named schema(s)
-t, --table=TABLE        dump only the named table(s)
-T, --exclude-table=TABLE  do NOT dump the named table(s)
```

### Data Options
```bash
-a, --data-only          dump only the data, not the schema
-s, --schema-only        dump only the schema, no data
--inserts                dump data as INSERT commands, rather than COPY
--column-inserts         dump data as INSERT commands with column names
```

## Complete Restore Options

### pg_restore (for custom/tar/directory formats)
```bash
-d, --dbname=NAME        connect to database name
-c, --clean              clean (drop) database objects before recreating
-C, --create             create the target database
-1, --single-transaction restore as a single transaction
-v, --verbose            verbose mode
-j, --jobs=NUM           use this many parallel jobs to restore
```

### psql (for plain text SQL dumps)
```bash
-d, --dbname=DBNAME      database name
-f, --file=FILENAME      execute commands from file
-1, --single-transaction execute as a single transaction
-v, --set=, --variable=NAME=VALUE  set psql variable NAME to VALUE
```

## Performance Options

### Large Database Dumps
```bash
# Parallel dump (custom format only)
pg_dump -h localhost -U postgres -d dbname -j 4 -Fc > backup.dump

# Parallel restore
pg_restore -h localhost -U postgres -d dbname -j 4 backup.dump

# Exclude large log tables
pg_dump -h localhost -U postgres -d dbname -T logs -T audit_trail > backup.sql
```

### Compression
```bash
# Gzip compression
pg_dump -h localhost -U postgres -d dbname | gzip -9 > backup.sql.gz

# Custom format (built-in compression)
pg_dump -h localhost -U postgres -d dbname -Fc -Z9 > backup.dump
```

## Database Recreation

### Drop and Recreate Database
```bash
# Drop existing database
psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS dbname;"

# Create new database
psql -h localhost -U postgres -c "CREATE DATABASE dbname;"

# Restore data  
psql -h localhost -U postgres -d dbname < backup.sql
```

### With Owner and Permissions
```bash
# Create database with owner
psql -h localhost -U postgres -c "CREATE DATABASE dbname OWNER username;"

# Dump with no owner/privileges (portable)
pg_dump -h localhost -U postgres -d dbname --no-owner --no-privileges > backup.sql
```

## Verification

### Pre-Dump Checks
```bash
# Check database size
psql -h localhost -U postgres -d dbname -c "SELECT pg_size_pretty(pg_database_size('dbname'));"

# List tables
psql -h localhost -U postgres -d dbname -c "\dt"

# Count records in main tables
psql -h localhost -U postgres -d dbname -c "SELECT COUNT(*) FROM tablename;"
```

### Post-Restore Checks
```bash
# Verify table count
psql -h localhost -U postgres -d dbname -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';"

# Check data integrity
psql -h localhost -U postgres -d dbname -c "SELECT COUNT(*) FROM main_table;"

# Check for errors
psql -h localhost -U postgres -d dbname -c "SELECT version();"
```

## Error Handling

### Common Issues
```bash
# Permission denied
# Fix: Check user permissions and database ownership

# Database does not exist
psql -h localhost -U postgres -c "CREATE DATABASE dbname;"

# Connection refused
# Fix: Check PostgreSQL is running
pg_isready -h localhost -U postgres

# Out of disk space
# Fix: Check available space
df -h /tmp/
```

### Debugging
```bash
# Verbose output
pg_dump -h localhost -U postgres -d dbname --verbose > backup.sql

# Test connection
psql -h localhost -U postgres -d dbname -c "SELECT 1;"

# Check dump file
head -20 backup.sql
tail -10 backup.sql
grep -c "INSERT INTO" backup.sql
```

## Examples

### Basic Backup and Restore
```bash
# Backup
pg_dump -h localhost -U postgres -d production_db > prod_backup.sql

# Restore to new database
psql -h localhost -U postgres -c "CREATE DATABASE dev_db;"
psql -h localhost -U postgres -d dev_db < prod_backup.sql
```

### Cross-Environment Copy
```bash
# From production pod
oc exec prod-pod -- pg_dump -h localhost -U postgres -d proddb > /tmp/backup.sql
oc cp prod-pod:/tmp/backup.sql ./backup.sql

# To development pod  
oc cp ./backup.sql dev-pod:/tmp/backup.sql
oc exec dev-pod -- psql -h localhost -U postgres -d devdb < /tmp/backup.sql
```

### Large Database with Compression
```bash
# Backup with compression
oc exec pod-name -- bash -c "pg_dump -h localhost -U postgres -d bigdb | gzip" > /tmp/backup.sql.gz

# Restore from compression
oc exec pod-name -- bash -c "gunzip -c /tmp/backup.sql.gz | psql -h localhost -U postgres -d bigdb"
```

That's it. Pure commands, no fluff.