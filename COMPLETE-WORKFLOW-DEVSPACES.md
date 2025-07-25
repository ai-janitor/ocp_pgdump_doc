# Complete PostgreSQL Dump and Restore Workflow

## Prerequisites
- Access to Dev Spaces environment
- Cluster admin access to OCP clusters
- Knowledge of target namespace and database name

## Step 1: Access Dev Spaces

### Open Dev Spaces Terminal
1. Navigate to your Dev Spaces URL
2. Open a terminal window
3. Verify oc CLI is available:
```bash
oc version
```

## Step 2: Get OCP Login Token

### From OCP Web Console
1. **Open target cluster web console** (ClusterA or ClusterB)
2. **Click your username** (top-right corner)
3. **Select "Copy login command"**
4. **Click "Display Token"** on the new page
5. **Copy the complete command** - looks like:
```bash
oc login --token=sha256~abcd1234efgh5678... --server=https://api.cluster-a.example.com:6443
```

## Step 3: Connect to OCP Cluster

### In Dev Spaces Terminal
```bash
# Paste the complete login command from step 2
oc login --token=sha256~your-actual-token --server=https://api.cluster-a.example.com:6443

# Verify connection
oc whoami

# Switch to target namespace
oc project your-namespace

# Verify you're in correct namespace
oc project
```

## Step 4: Locate Database Pod

### Find Redshift Pod
```bash
# List all pods
oc get pods

# Find Redshift pod specifically  
oc get pods | grep redshift

# Store pod name in variable
POD_NAME=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
echo "Pod: $POD_NAME"

# Verify pod is running
oc get pod $POD_NAME
```

## Step 5: Create Database Dump

### Method A: Interactive Shell
```bash
# Enter the pod
oc exec -it $POD_NAME -- bash

# Inside pod - create dump
cd /tmp
pg_dump -h localhost -U postgres -d your_database_name > database_dump_$(date +%Y%m%d_%H%M%S).sql

# Verify dump created
ls -la *.sql
du -h *.sql

# Exit pod
exit
```

### Method B: Direct Command
```bash
# Create dump without entering pod
DUMP_FILE="database_dump_$(date +%Y%m%d_%H%M%S).sql"
oc exec $POD_NAME -- pg_dump -h localhost -U postgres -d your_database_name > /tmp/$DUMP_FILE

# Verify dump
oc exec $POD_NAME -- ls -la /tmp/*.sql
oc exec $POD_NAME -- du -h /tmp/$DUMP_FILE
```

## Step 6: Copy Dump to Dev Spaces

### Transfer File
```bash
# Get dump filename
DUMP_FILE=$(oc exec $POD_NAME -- ls -t /tmp/*.sql | head -1)
echo "Dump file: $DUMP_FILE"

# Copy to Dev Spaces
oc cp $POD_NAME:$DUMP_FILE ./$(basename $DUMP_FILE)

# Verify copy
ls -la *.sql
du -h *.sql
```

## Step 7: Connect to Target Cluster (if different)

### Switch Clusters
```bash
# If restoring to different cluster, repeat steps 2-4 for target cluster
# Get new login token from target cluster web console
oc login --token=sha256~target-cluster-token --server=https://api.cluster-b.example.com:6443

# Switch to target namespace
oc project target-namespace

# Get target pod
TARGET_POD=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
echo "Target pod: $TARGET_POD"
```

## Step 8: Restore Database

### Copy Dump to Target Pod
```bash
# Copy dump file to target pod
oc cp ./$(basename $DUMP_FILE) $TARGET_POD:/tmp/

# Verify file copied
oc exec $TARGET_POD -- ls -la /tmp/*.sql
```

### Restore Data

#### Option A: Clean Restore (Drop/Recreate Database)
```bash
# Drop existing database
oc exec $TARGET_POD -- psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS your_database_name;"

# Create new database
oc exec $TARGET_POD -- psql -h localhost -U postgres -c "CREATE DATABASE your_database_name;"

# Restore data
oc exec $TARGET_POD -- psql -h localhost -U postgres -d your_database_name < /tmp/$(basename $DUMP_FILE)
```

#### Option B: Restore to Existing Database
```bash
# Restore directly (may cause conflicts with existing data)
oc exec $TARGET_POD -- psql -h localhost -U postgres -d your_database_name < /tmp/$(basename $DUMP_FILE)
```

## Step 9: Verify Restore

### Check Data
```bash
# Verify database exists
oc exec $TARGET_POD -- psql -h localhost -U postgres -c "\l" | grep your_database_name

# Check table count
oc exec $TARGET_POD -- psql -h localhost -U postgres -d your_database_name -c "\dt" | wc -l

# Count records in main table
oc exec $TARGET_POD -- psql -h localhost -U postgres -d your_database_name -c "SELECT COUNT(*) FROM your_main_table;"

# Check database size
oc exec $TARGET_POD -- psql -h localhost -U postgres -d your_database_name -c "SELECT pg_size_pretty(pg_database_size('your_database_name'));"
```

## Step 10: Cleanup

### Remove Temporary Files
```bash
# Clean up source pod
oc exec $POD_NAME -- rm /tmp/*.sql

# Clean up target pod  
oc exec $TARGET_POD -- rm /tmp/*.sql

# Clean up Dev Spaces
rm *.sql

# Verify cleanup
oc exec $POD_NAME -- ls /tmp/*.sql 2>/dev/null || echo "Source cleaned"
oc exec $TARGET_POD -- ls /tmp/*.sql 2>/dev/null || echo "Target cleaned"
ls *.sql 2>/dev/null || echo "Dev Spaces cleaned"
```

## Complete Automation Script

```bash
#!/bin/bash
# complete-workflow.sh - Full automation from login to cleanup

set -e

# Configuration - UPDATE THESE VALUES
SOURCE_CLUSTER_TOKEN="sha256~your-source-token"
SOURCE_CLUSTER_URL="https://api.cluster-a.example.com:6443"
SOURCE_NAMESPACE="your-source-namespace"
SOURCE_DATABASE="your_source_database"

TARGET_CLUSTER_TOKEN="sha256~your-target-token"  
TARGET_CLUSTER_URL="https://api.cluster-b.example.com:6443"
TARGET_NAMESPACE="your-target-namespace"
TARGET_DATABASE="your_target_database"

echo "=== PostgreSQL Dump and Restore Complete Workflow ==="
echo "Source: $SOURCE_CLUSTER_URL/$SOURCE_NAMESPACE"
echo "Target: $TARGET_CLUSTER_URL/$TARGET_NAMESPACE"

# Step 1: Connect to source cluster
echo "1. Connecting to source cluster..."
oc login --token=$SOURCE_CLUSTER_TOKEN --server=$SOURCE_CLUSTER_URL
oc project $SOURCE_NAMESPACE

# Step 2: Get source pod
echo "2. Finding source pod..."
SOURCE_POD=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
echo "Source pod: $SOURCE_POD"

# Step 3: Create dump
echo "3. Creating database dump..."
DUMP_FILE="dump_$(date +%Y%m%d_%H%M%S).sql"
oc exec $SOURCE_POD -- pg_dump -h localhost -U postgres -d $SOURCE_DATABASE > /tmp/$DUMP_FILE
echo "Dump created: $DUMP_FILE"

# Step 4: Copy to Dev Spaces
echo "4. Copying dump to Dev Spaces..."
oc cp $SOURCE_POD:/tmp/$DUMP_FILE ./$DUMP_FILE
SOURCE_SIZE=$(du -h ./$DUMP_FILE | cut -f1)
echo "Copied to Dev Spaces: $SOURCE_SIZE"

# Step 5: Connect to target cluster
echo "5. Connecting to target cluster..."
oc login --token=$TARGET_CLUSTER_TOKEN --server=$TARGET_CLUSTER_URL
oc project $TARGET_NAMESPACE

# Step 6: Get target pod
echo "6. Finding target pod..."
TARGET_POD=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
echo "Target pod: $TARGET_POD"

# Step 7: Copy dump to target
echo "7. Copying dump to target pod..."
oc cp ./$DUMP_FILE $TARGET_POD:/tmp/$DUMP_FILE

# Step 8: Restore database
echo "8. Restoring database..."
echo "   Dropping existing database..."
oc exec $TARGET_POD -- psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS $TARGET_DATABASE;" || true

echo "   Creating new database..."
oc exec $TARGET_POD -- psql -h localhost -U postgres -c "CREATE DATABASE $TARGET_DATABASE;"

echo "   Restoring data..."
oc exec $TARGET_POD -- psql -h localhost -U postgres -d $TARGET_DATABASE < /tmp/$DUMP_FILE

# Step 9: Verify restore
echo "9. Verifying restore..."
RECORD_COUNT=$(oc exec $TARGET_POD -- psql -h localhost -U postgres -d $TARGET_DATABASE -t -c "SELECT COUNT(*) FROM information_schema.tables WHERE table_schema = 'public';" | tr -d ' ')
echo "Tables restored: $RECORD_COUNT"

DB_SIZE=$(oc exec $TARGET_POD -- psql -h localhost -U postgres -d $TARGET_DATABASE -t -c "SELECT pg_size_pretty(pg_database_size('$TARGET_DATABASE'));" | tr -d ' ')
echo "Database size: $DB_SIZE"

# Step 10: Cleanup
echo "10. Cleaning up..."
oc login --token=$SOURCE_CLUSTER_TOKEN --server=$SOURCE_CLUSTER_URL
oc exec $SOURCE_POD -- rm /tmp/$DUMP_FILE || true

oc login --token=$TARGET_CLUSTER_TOKEN --server=$TARGET_CLUSTER_URL  
oc exec $TARGET_POD -- rm /tmp/$DUMP_FILE || true

rm ./$DUMP_FILE

echo "=== Workflow Complete ==="
echo "✅ Database successfully copied from source to target"
echo "Source: $SOURCE_CLUSTER_URL/$SOURCE_NAMESPACE/$SOURCE_DATABASE"
echo "Target: $TARGET_CLUSTER_URL/$TARGET_NAMESPACE/$TARGET_DATABASE"
```

## Quick Reference Commands

### Essential Commands by Step
```bash
# Connect to cluster
oc login --token=TOKEN --server=URL
oc project NAMESPACE

# Find pod
oc get pods | grep redshift
POD_NAME=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')

# Create dump
oc exec $POD_NAME -- pg_dump -h localhost -U postgres -d DATABASE > /tmp/dump.sql

# Copy files
oc cp $POD_NAME:/tmp/dump.sql ./dump.sql
oc cp ./dump.sql $POD_NAME:/tmp/dump.sql

# Restore database
oc exec $POD_NAME -- psql -h localhost -U postgres -c "DROP DATABASE IF EXISTS DATABASE;"
oc exec $POD_NAME -- psql -h localhost -U postgres -c "CREATE DATABASE DATABASE;"
oc exec $POD_NAME -- psql -h localhost -U postgres -d DATABASE < /tmp/dump.sql

# Verify
oc exec $POD_NAME -- psql -h localhost -U postgres -d DATABASE -c "SELECT COUNT(*) FROM main_table;"

# Cleanup
oc exec $POD_NAME -- rm /tmp/dump.sql
rm ./dump.sql
```

## Troubleshooting

### Login Issues
```bash
# Token expired - get new token from web console
# Wrong cluster - verify URL in browser
# Permission denied - verify cluster admin access
oc auth can-i "*" "*" --all-namespaces
```

### Pod Issues
```bash
# Pod not found
oc get pods --all-namespaces | grep redshift
oc describe pod $POD_NAME

# Pod not ready
oc get pod $POD_NAME -o wide
oc logs $POD_NAME
```

### Database Issues
```bash
# Connection failed
oc exec $POD_NAME -- pg_isready -h localhost -U postgres

# Database doesn't exist
oc exec $POD_NAME -- psql -h localhost -U postgres -c "\l"

# Permission denied
oc exec $POD_NAME -- psql -h localhost -U postgres -c "\du"
```

### File Transfer Issues
```bash
# File not found
oc exec $POD_NAME -- ls -la /tmp/

# Disk space full
oc exec $POD_NAME -- df -h

# Copy failed
oc exec $POD_NAME -- cat /tmp/dump.sql > local_dump.sql
```

Start from Dev Spaces, follow the steps, done.