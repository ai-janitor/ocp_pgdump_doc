# Step 2: Create PostgreSQL Dump in Pod

This document covers how to create a PostgreSQL database dump inside the Redshift pod running in your OCP namespace.

## Prerequisites

- Successfully completed [Step 1: OCP Login and Connection](./01-ocp-login-connection.md)
- Connected to the correct OCP cluster and namespace
- Cluster admin privileges

## Overview

This step involves:
1. Identifying and connecting to the Redshift pod
2. Creating a PostgreSQL dump inside the pod's `/tmp/` directory
3. Verifying the dump was created successfully

## Find and Connect to Redshift Pod

### Get Pod Information

```bash
# List all pods in current namespace
oc get pods

# Find Redshift pod specifically
oc get pods | grep redshift

# Get detailed pod information
oc describe pod <redshift-pod-name>
```

### Store Pod Name for Easy Reference

```bash
# Get the exact pod name and store in variable
POD_NAME=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
echo "Pod name: $POD_NAME"

# Alternative: if you know the exact pod name
POD_NAME="redshift-deployment-12345-abcde"
```

## Create PostgreSQL Dump

### Method 1: Interactive Shell Access

Connect to the pod and create the dump interactively:

```bash
# Bash into the pod
oc exec -it $POD_NAME -- /bin/bash
```

Once inside the pod:

```bash
# Navigate to tmp directory
cd /tmp

# Check available space
df -h /tmp

# Create PostgreSQL dump with timestamp
DUMP_FILE="database_dump_$(date +%Y%m%d_%H%M%S).sql"
pg_dump -h localhost -U postgres -d your_database_name > $DUMP_FILE

# Verify the dump was created
ls -la /tmp/*.sql

# Check file size to ensure dump completed
du -h /tmp/$DUMP_FILE

# Quick verification - check first few lines
head -20 $DUMP_FILE

# Exit the pod
exit
```

### Method 2: Direct Command Execution

Create the dump without entering the pod interactively:

```bash
# Create dump with timestamp directly
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
DUMP_FILE="database_dump_$TIMESTAMP.sql"

oc exec $POD_NAME -- pg_dump -h localhost -U postgres -d your_database_name > /tmp/$DUMP_FILE

# Verify dump creation
oc exec $POD_NAME -- ls -la /tmp/*.sql

# Check file size
oc exec $POD_NAME -- du -h /tmp/$DUMP_FILE
```

## Database Connection Options

### Standard Connection
```bash
# Basic pg_dump command
pg_dump -h localhost -U postgres -d your_database_name > dump.sql
```

### With Specific Options
```bash
# Include additional options for better dumps
pg_dump -h localhost -U postgres -d your_database_name \
  --verbose \
  --no-owner \
  --no-privileges \
  --clean \
  --if-exists > dump.sql
```

### Compressed Dump (for large databases)
```bash
# Create compressed dump to save space
pg_dump -h localhost -U postgres -d your_database_name | gzip > dump.sql.gz
```

### Schema-only or Data-only Dumps
```bash
# Schema only (structure without data)
pg_dump -h localhost -U postgres -d your_database_name --schema-only > schema_dump.sql

# Data only (no structure)
pg_dump -h localhost -U postgres -d your_database_name --data-only > data_dump.sql
```

## Verify Dump Quality

### Check Dump Contents

```bash
# Inside the pod, verify the dump file
oc exec $POD_NAME -- bash -c "head -20 /tmp/$DUMP_FILE"

# Check for common SQL elements
oc exec $POD_NAME -- bash -c "grep -i 'CREATE TABLE' /tmp/$DUMP_FILE | wc -l"
oc exec $POD_NAME -- bash -c "grep -i 'INSERT INTO' /tmp/$DUMP_FILE | wc -l"
```

### Size Verification

```bash
# Compare original database size with dump
oc exec $POD_NAME -- psql -h localhost -U postgres -d your_database_name -c "
  SELECT 
    pg_database.datname,
    pg_size_pretty(pg_database_size(pg_database.datname)) AS size
  FROM pg_database
  WHERE datname = 'your_database_name';
"

# Check dump file size
oc exec $POD_NAME -- ls -lh /tmp/$DUMP_FILE
```

## Troubleshooting

### Common Issues and Solutions

#### PostgreSQL Connection Issues
```bash
# Test database connectivity first
oc exec $POD_NAME -- psql -h localhost -U postgres -d your_database_name -c "SELECT version();"

# Check if PostgreSQL is running
oc exec $POD_NAME -- ps aux | grep postgres

# Check PostgreSQL status
oc exec $POD_NAME -- pg_isready -h localhost -U postgres
```

#### Permission Issues in /tmp
```bash
# Check /tmp directory permissions
oc exec $POD_NAME -- ls -la /tmp

# Try alternative directory if /tmp is not writable
oc exec $POD_NAME -- ls -la /var/tmp

# Use alternative directory
pg_dump -h localhost -U postgres -d your_database_name > /var/tmp/dump.sql
```

#### Disk Space Issues
```bash
# Check available disk space
oc exec $POD_NAME -- df -h

# Check specific directory space
oc exec $POD_NAME -- du -h /tmp
```

#### Large Database Timeouts
```bash
# For very large databases, use compression and increase timeout
timeout 3600 oc exec $POD_NAME -- bash -c "pg_dump -h localhost -U postgres -d your_database_name | gzip > /tmp/dump.sql.gz"
```

## Security Considerations

- Dumps may contain sensitive data - handle appropriately
- Clean up dump files from pod after copying (covered in Step 3)
- Consider using compressed dumps for large datasets
- Verify dump integrity before relying on it for restoration

## File Naming Conventions

Use consistent naming patterns:

```bash
# Include date, time, and database name
database_dump_$(date +%Y%m%d_%H%M%S).sql

# Include cluster and environment info
${DATABASE_NAME}_${CLUSTER_NAME}_$(date +%Y%m%d_%H%M%S).sql

# For different dump types
${DATABASE_NAME}_schema_only_$(date +%Y%m%d_%H%M%S).sql
${DATABASE_NAME}_data_only_$(date +%Y%m%d_%H%M%S).sql
```

## Next Step

Once your PostgreSQL dump is created and verified, proceed to [Step 3: Copy Dump to Dev Spaces](./03-copy-dump-to-devspaces.md)
