# Step 3: Copy Dump to Dev Spaces

This document covers how to copy the PostgreSQL dump file from the Redshift pod to your Dev Spaces environment.

## Prerequisites

- Successfully completed [Step 2: Create PostgreSQL Dump](./02-create-postgres-dump.md)
- PostgreSQL dump file created in pod's `/tmp/` directory
- Still connected to the correct OCP cluster and namespace

## Overview

This step involves:
1. Locating the dump file in the pod
2. Copying the file from pod to Dev Spaces
3. Verifying file integrity after copy
4. Organizing files for multiple clusters

## Locate Dump File in Pod

### List Available Dump Files

```bash
# Set pod name variable (if not already set)
POD_NAME=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')

# List all SQL dump files in /tmp
oc exec $POD_NAME -- ls -la /tmp/*.sql

# If using compressed dumps
oc exec $POD_NAME -- ls -la /tmp/*.sql.gz

# Get specific file details
oc exec $POD_NAME -- ls -lh /tmp/database_dump_*.sql
```

### Store Dump Filename

```bash
# Get the most recent dump file
DUMP_FILE=$(oc exec $POD_NAME -- ls -t /tmp/*.sql | head -1)
echo "Dump file: $DUMP_FILE"

# Alternative: if you know the exact filename
DUMP_FILE="/tmp/database_dump_20250725_143022.sql"
```

## Copy File from Pod to Dev Spaces

### Basic Copy Operation

```bash
# Copy the dump file to current directory in Dev Spaces
oc cp $POD_NAME:$DUMP_FILE ./$(basename $DUMP_FILE)

# Verify the file was copied
ls -la *.sql
```

### Copy with Custom Local Name

```bash
# Copy with a more descriptive local filename
LOCAL_FILENAME="cluster-a-$(date +%Y%m%d)-database-dump.sql"
oc cp $POD_NAME:$DUMP_FILE ./$LOCAL_FILENAME

# Verify copy
ls -la $LOCAL_FILENAME
```

### Copy to Specific Directory

```bash
# Create directory structure for organized storage
mkdir -p ~/database-dumps/$(date +%Y-%m)

# Copy to organized location
oc cp $POD_NAME:$DUMP_FILE ~/database-dumps/$(date +%Y-%m)/$(basename $DUMP_FILE)
```

## Verify File Integrity

### Compare File Sizes

```bash
# Check file size in pod
echo "File size in pod:"
oc exec $POD_NAME -- du -h $DUMP_FILE

# Check file size in Dev Spaces
echo "File size in Dev Spaces:"
du -h ./$(basename $DUMP_FILE)
```

### Verify File Contents

```bash
# Check first few lines of copied file
head -20 ./$(basename $DUMP_FILE)

# Verify it's a valid SQL dump
grep -i "PostgreSQL database dump" ./$(basename $DUMP_FILE)

# Count tables and data
echo "Tables found:"
grep -c "CREATE TABLE" ./$(basename $DUMP_FILE)

echo "Insert statements found:"
grep -c "INSERT INTO" ./$(basename $DUMP_FILE)
```

### File Hash Verification (Advanced)

```bash
# Generate hash in pod
POD_HASH=$(oc exec $POD_NAME -- md5sum $DUMP_FILE | cut -d' ' -f1)

# Generate hash in Dev Spaces
LOCAL_HASH=$(md5sum ./$(basename $DUMP_FILE) | cut -d' ' -f1)

# Compare hashes
if [ "$POD_HASH" = "$LOCAL_HASH" ]; then
    echo "✓ File integrity verified - hashes match"
else
    echo "✗ Warning: File hashes do not match"
fi
```

## Multi-Cluster Organization

### Directory Structure for Multiple Clusters

```bash
# Create organized directory structure
mkdir -p ~/database-dumps/{cluster-a,cluster-b}/{$(date +%Y-%m)}

# Copy to cluster-specific directory
CLUSTER_NAME="cluster-a"  # or cluster-b
oc cp $POD_NAME:$DUMP_FILE ~/database-dumps/$CLUSTER_NAME/$(date +%Y-%m)/$(basename $DUMP_FILE)
```

### Automated Multi-Cluster Workflow

```bash
#!/bin/bash
# Script to organize dumps by cluster

# Determine current cluster
CURRENT_CLUSTER=$(oc cluster-info | grep -oP 'https://api\.\K[^.]+')
DATE_DIR=$(date +%Y-%m)

# Create directory structure
mkdir -p ~/database-dumps/$CURRENT_CLUSTER/$DATE_DIR

# Copy file with cluster prefix
CLUSTER_FILENAME="${CURRENT_CLUSTER}_$(basename $DUMP_FILE)"
oc cp $POD_NAME:$DUMP_FILE ~/database-dumps/$CURRENT_CLUSTER/$DATE_DIR/$CLUSTER_FILENAME

echo "Dump saved to: ~/database-dumps/$CURRENT_CLUSTER/$DATE_DIR/$CLUSTER_FILENAME"
```

## Batch Operations for Multiple Files

### Copy Multiple Dump Files

```bash
# List all dump files in pod
DUMP_FILES=$(oc exec $POD_NAME -- ls /tmp/*.sql)

# Copy all dump files
for dump_file in $DUMP_FILES; do
    echo "Copying $dump_file..."
    oc cp $POD_NAME:$dump_file ./$(basename $dump_file)
done

# Verify all copies
ls -la *.sql
```

### Compressed File Handling

```bash
# If working with compressed dumps
oc cp $POD_NAME:/tmp/dump.sql.gz ./dump.sql.gz

# Decompress in Dev Spaces
gunzip ./dump.sql.gz

# Verify decompressed file
ls -la ./dump.sql
```

## Clean Up Pod Files

### Remove Dump Files from Pod

After successful copy, clean up the pod:

```bash
# List files to be removed
oc exec $POD_NAME -- ls -la /tmp/*.sql

# Remove specific dump file
oc exec $POD_NAME -- rm $DUMP_FILE

# Remove all dump files (use with caution)
oc exec $POD_NAME -- rm /tmp/*.sql

# Verify cleanup
oc exec $POD_NAME -- ls -la /tmp/*.sql
```

## Complete Copy Script

Here's a complete script that handles the entire copy process:

```bash
#!/bin/bash
# complete-copy-script.sh

set -e  # Exit on any error

# Configuration
POD_NAME=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
CURRENT_CLUSTER=$(oc cluster-info | grep -oP 'https://api\.\K[^.]+' || echo "unknown-cluster")
DATE_DIR=$(date +%Y-%m)
TIMESTAMP=$(date +%Y%m%d_%H%M%S)

echo "=== PostgreSQL Dump Copy Process ==="
echo "Pod: $POD_NAME"
echo "Cluster: $CURRENT_CLUSTER"
echo "Timestamp: $TIMESTAMP"

# Find the most recent dump file
echo "Looking for dump files in pod..."
DUMP_FILE=$(oc exec $POD_NAME -- ls -t /tmp/*.sql 2>/dev/null | head -1 || echo "")

if [ -z "$DUMP_FILE" ]; then
    echo "❌ No dump files found in pod /tmp/ directory"
    echo "Please run Step 2 to create a dump first"
    exit 1
fi

echo "Found dump file: $DUMP_FILE"

# Check file size in pod
POD_SIZE=$(oc exec $POD_NAME -- du -h $DUMP_FILE | cut -f1)
echo "File size in pod: $POD_SIZE"

# Create directory structure
LOCAL_DIR="~/database-dumps/$CURRENT_CLUSTER/$DATE_DIR"
mkdir -p "$LOCAL_DIR"

# Generate local filename
LOCAL_FILENAME="${CURRENT_CLUSTER}_$(basename $DUMP_FILE)"
LOCAL_PATH="$LOCAL_DIR/$LOCAL_FILENAME"

# Copy file
echo "Copying to: $LOCAL_PATH"
oc cp $POD_NAME:$DUMP_FILE "$LOCAL_PATH"

# Verify copy
if [ -f "$LOCAL_PATH" ]; then
    LOCAL_SIZE=$(du -h "$LOCAL_PATH" | cut -f1)
    echo "✅ Copy successful!"
    echo "Local file size: $LOCAL_SIZE"
    
    # Quick integrity check
    if grep -q "PostgreSQL database dump" "$LOCAL_PATH"; then
        echo "✅ File appears to be a valid PostgreSQL dump"
    else
        echo "⚠️  Warning: File may not be a valid PostgreSQL dump"
    fi
else
    echo "❌ Copy failed - file not found locally"
    exit 1
fi

# Ask about cleanup
read -p "Remove dump file from pod? (y/N): " -n 1 -r
echo
if [[ $REPLY =~ ^[Yy]$ ]]; then
    oc exec $POD_NAME -- rm $DUMP_FILE
    echo "✅ Cleanup completed"
fi

echo "=== Process Complete ==="
echo "Dump file available at: $LOCAL_PATH"
```

## Troubleshooting Copy Issues

### Permission Errors
```bash
# Check if you have read permissions on the pod file
oc exec $POD_NAME -- ls -la $DUMP_FILE

# Check write permissions in Dev Spaces
touch test-write.txt && rm test-write.txt
```

### Network/Connection Issues
```bash
# Test pod connectivity
oc exec $POD_NAME -- echo "Connection test"

# Check if oc cp command is available
oc cp --help
```

### Large File Issues
```bash
# For very large files, use compression during copy
oc exec $POD_NAME -- gzip $DUMP_FILE
oc cp $POD_NAME:${DUMP_FILE}.gz ./$(basename ${DUMP_FILE}).gz
gunzip ./$(basename ${DUMP_FILE}).gz
```

## Security and Best Practices

- **Verify file integrity** after every copy operation
- **Clean up pod files** to prevent disk space issues
- **Organize files** by cluster and date for easy management
- **Backup important dumps** to external storage
- **Use compression** for large files to save space and transfer time

## Next Steps

After successfully copying your database dump to Dev Spaces, you can:

- Import the dump into another PostgreSQL instance
- Archive the file for backup purposes
- Analyze the data structure
- Transfer between development environments
- Compare dumps from different clusters

Your PostgreSQL dump is now safely stored in your Dev Spaces environment and ready for use!
