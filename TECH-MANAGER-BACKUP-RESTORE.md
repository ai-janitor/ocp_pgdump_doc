# Database Backup and Restore Process - Technical Manager Guide

## Process Overview

Complete workflow for backing up PostgreSQL databases from OpenShift pods and restoring them to target pods across ClusterA and ClusterB environments.

## Technical Architecture

```
Dev Spaces (CLI Environment)
    ↓ oc login (token from web console)
ClusterA/ClusterB OCP Environment
    ↓ oc exec (bash into pod)
Redshift Pod (/tmp/ directory)
    ↓ pg_dump (create SQL dump)
Pod Filesystem → Dev Spaces
    ↓ oc cp (file transfer)
Local Storage → Target Pod
    ↓ psql (restore database)
Target Database
```

## Implementation Steps

### 1. Source Environment Access
```bash
# Get login token from OCP web console
# Username dropdown → "Copy login command" → "Display Token"
oc login --token=sha256~actual-token --server=https://api.cluster-a.example.com:6443
oc project target-namespace
```

### 2. Database Backup Creation
```bash
# Locate and access pod
POD_NAME=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
oc exec -it $POD_NAME -- /bin/bash

# Inside pod: Create dump
cd /tmp
pg_dump -h localhost -U postgres -d database_name > database_dump_$(date +%Y%m%d_%H%M%S).sql
exit
```

### 3. File Transfer to Dev Spaces
```bash
# Copy dump file from pod to local environment
DUMP_FILE=$(oc exec $POD_NAME -- ls -t /tmp/*.sql | head -1)
oc cp $POD_NAME:$DUMP_FILE ./$(basename $DUMP_FILE)

# Verify transfer
du -h ./$(basename $DUMP_FILE)
```

### 4. Target Environment Restoration
```bash
# Connect to target cluster (if different)
oc login --token=sha256~target-cluster-token --server=https://api.cluster-b.example.com:6443
oc project target-namespace

# Get target pod
TARGET_POD=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')

# Copy dump to target pod
oc cp ./$(basename $DUMP_FILE) $TARGET_POD:/tmp/

# Restore database
oc exec $TARGET_POD -- psql -h localhost -U postgres -d target_database < /tmp/$(basename $DUMP_FILE)
```

## Command Reference

### Essential Commands
```bash
# Pod operations
oc get pods -l app=redshift
oc exec -it $POD_NAME -- /bin/bash
oc exec $POD_NAME -- command

# File operations  
oc cp source:path destination:path
oc exec $POD_NAME -- ls -la /tmp/
oc exec $POD_NAME -- du -h /tmp/file.sql

# Database operations
pg_dump -h localhost -U postgres -d dbname > dump.sql
psql -h localhost -U postgres -d dbname < dump.sql
pg_dump --help  # for additional options
```

### Cluster Switching
```bash
# Check current cluster
oc cluster-info
oc whoami

# Switch clusters
oc login --token=new-token --server=new-cluster-api
oc project new-namespace
```

## Automation Script

```bash
#!/bin/bash
# backup-restore.sh - Complete automation

set -e

# Configuration
SOURCE_CLUSTER_TOKEN="sha256~source-token"
SOURCE_CLUSTER_URL="https://api.cluster-a.example.com:6443"
TARGET_CLUSTER_TOKEN="sha256~target-token"  
TARGET_CLUSTER_URL="https://api.cluster-b.example.com:6443"
NAMESPACE="database-namespace"
DATABASE_NAME="production_db"

echo "=== Database Backup and Restore Process ==="

# 1. Connect to source cluster
echo "Connecting to source cluster..."
oc login --token=$SOURCE_CLUSTER_TOKEN --server=$SOURCE_CLUSTER_URL
oc project $NAMESPACE

# 2. Create backup
echo "Creating database backup..."
SOURCE_POD=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
DUMP_FILE="backup_$(date +%Y%m%d_%H%M%S).sql"

oc exec $SOURCE_POD -- pg_dump -h localhost -U postgres -d $DATABASE_NAME > /tmp/$DUMP_FILE
echo "Backup created: $DUMP_FILE"

# 3. Copy to local
echo "Transferring backup file..."
oc cp $SOURCE_POD:/tmp/$DUMP_FILE ./$DUMP_FILE
FILE_SIZE=$(du -h ./$DUMP_FILE | cut -f1)
echo "Transfer complete. Size: $FILE_SIZE"

# 4. Connect to target cluster
echo "Connecting to target cluster..."
oc login --token=$TARGET_CLUSTER_TOKEN --server=$TARGET_CLUSTER_URL
oc project $NAMESPACE

# 5. Restore to target
echo "Restoring to target database..."
TARGET_POD=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
oc cp ./$DUMP_FILE $TARGET_POD:/tmp/$DUMP_FILE

# Optional: Drop and recreate database for clean restore
oc exec $TARGET_POD -- psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS $DATABASE_NAME;"
oc exec $TARGET_POD -- psql -h localhost -U postgres -c "CREATE DATABASE $DATABASE_NAME;"

# Restore data
oc exec $TARGET_POD -- psql -h localhost -U postgres -d $DATABASE_NAME < /tmp/$DUMP_FILE

# 6. Cleanup
echo "Cleaning up temporary files..."
oc exec $SOURCE_POD -- rm /tmp/$DUMP_FILE
oc exec $TARGET_POD -- rm /tmp/$DUMP_FILE
rm ./$DUMP_FILE

echo "=== Process Complete ==="
echo "Database successfully backed up and restored"
```

## Performance Considerations

### File Size Impact
| Database Size | Dump Time | Transfer Time | Restore Time |
|---------------|-----------|---------------|--------------|
| < 100MB | 30 seconds | 10 seconds | 45 seconds |
| 100MB - 1GB | 2-5 minutes | 30-60 seconds | 3-8 minutes |
| 1GB - 10GB | 10-30 minutes | 2-10 minutes | 15-45 minutes |
| > 10GB | 30+ minutes | 10+ minutes | 45+ minutes |

### Optimization Options
```bash
# Compressed dumps for large databases
pg_dump -h localhost -U postgres -d dbname | gzip > dump.sql.gz

# Schema-only backup (structure without data)
pg_dump -h localhost -U postgres -d dbname --schema-only > schema.sql

# Parallel dump for large databases
pg_dump -h localhost -U postgres -d dbname -j 4 -F d -f dump_directory/

# Exclude specific tables
pg_dump -h localhost -U postgres -d dbname --exclude-table=logs > dump.sql
```

## Error Handling and Troubleshooting

### Common Issues and Fixes

**Pod Not Found**
```bash
# Debug pod discovery
oc get pods --all-namespaces | grep redshift
oc get pods -o wide
oc describe pod $POD_NAME
```

**Permission Denied**
```bash
# Check pod permissions
oc exec $POD_NAME -- ls -la /tmp/
oc exec $POD_NAME -- whoami
oc auth can-i "*" "*" --as=system:serviceaccount:namespace:podname
```

**Database Connection Failed**
```bash
# Test PostgreSQL connectivity
oc exec $POD_NAME -- pg_isready -h localhost -U postgres
oc exec $POD_NAME -- psql -h localhost -U postgres -c "SELECT version();"
```

**File Transfer Failed**
```bash
# Check file exists and permissions
oc exec $POD_NAME -- ls -la /tmp/backup.sql
oc exec $POD_NAME -- file /tmp/backup.sql

# Alternative transfer method
oc exec $POD_NAME -- cat /tmp/backup.sql > local_backup.sql
```

**Large File Issues**
```bash
# Check disk space in pod
oc exec $POD_NAME -- df -h
oc exec $POD_NAME -- du -h /tmp/

# Use streaming for large files
oc exec $POD_NAME -- pg_dump -h localhost -U postgres -d dbname | cat > local_dump.sql
```

## Verification Steps

### Post-Backup Verification
```bash
# Verify dump file integrity
head -20 backup.sql | grep "PostgreSQL database dump"
tail -10 backup.sql | grep "PostgreSQL database dump complete"

# Check record counts
grep -c "INSERT INTO" backup.sql
grep -c "CREATE TABLE" backup.sql
```

### Post-Restore Verification
```bash
# Compare table counts
oc exec $SOURCE_POD -- psql -h localhost -U postgres -d $DATABASE_NAME -c "
  SELECT schemaname,tablename FROM pg_tables WHERE schemaname NOT IN ('information_schema','pg_catalog');"

oc exec $TARGET_POD -- psql -h localhost -U postgres -d $DATABASE_NAME -c "
  SELECT schemaname,tablename FROM pg_tables WHERE schemaname NOT IN ('information_schema','pg_catalog');"

# Compare row counts
oc exec $TARGET_POD -- psql -h localhost -U postgres -d $DATABASE_NAME -c "
  SELECT COUNT(*) FROM your_main_table;"
```

## Multi-Environment Patterns

### Development Refresh Pattern
```bash
# Weekly production → development refresh
SOURCE="production-cluster"
TARGET="development-cluster"
# Run backup-restore script with dev-specific settings
```

### Cross-Cluster Migration Pattern
```bash
# One-time data migration between clusters
# Include schema modifications if needed
pg_dump --schema-only > schema.sql
pg_dump --data-only > data.sql
# Apply schema changes, then restore data
```

### Disaster Recovery Pattern
```bash
# Emergency restore from backup
# Skip normal validation steps for speed
# Full verification after restore complete
```

## Security Requirements

### Access Control
- Cluster admin privileges required on both clusters
- Database credentials must be available in target pods
- Network connectivity between Dev Spaces and both clusters

### Data Handling
- Backup files contain sensitive data - handle appropriately
- Clean up temporary files after operations
- Use compressed transfers for large datasets

### Audit Trail
```bash
# Log all operations
echo "$(date): Backup started from $SOURCE_POD" >> backup.log
echo "$(date): Restore completed to $TARGET_POD" >> backup.log
```

## Monitoring and Status

### Progress Tracking
```bash
# Monitor dump progress
oc exec $POD_NAME -- ls -la /tmp/*.sql
watch "oc exec $POD_NAME -- du -h /tmp/backup.sql"

# Monitor restore progress  
oc logs $TARGET_POD -f | grep -i "restore\|error"
```

### Resource Monitoring
```bash
# Check pod resource usage during operations
oc top pod $POD_NAME
oc exec $POD_NAME -- top
oc exec $POD_NAME -- iostat 5
```

This process provides complete database backup and restore capability across OpenShift environments with full error handling and verification steps.