# PostgreSQL Dump from OCP Pod to Dev Spaces

Complete documentation for extracting PostgreSQL database dumps from Redshift pods running in OpenShift Container Platform (OCP) and copying them to your Dev Spaces environment.

## Overview

This documentation covers the complete workflow for cluster administrators to:

1. **Connect to OCP clusters** from Dev Spaces using login tokens
2. **Create PostgreSQL dumps** inside Redshift pods
3. **Copy dump files** from pods to Dev Spaces environment

## Architecture

- **ClusterA & ClusterB**: Two separate OCP clusters with independent storage
- **Dev Spaces**: Development environment with OCP CLI tools
- **Redshift Pod**: PostgreSQL database running in OCP namespaces
- **Workflow**: Extract data from either cluster to Dev Spaces

## Prerequisites

- Cluster admin access to both ClusterA and ClusterB
- Dev Spaces instance with OCP CLI tools installed
- Access to namespaces containing Redshift pods

## Documentation Structure

### [Step 1: OCP Login and Connection](./01-ocp-login-connection.md)
- Obtaining login tokens from OCP web console
- Connecting to ClusterA or ClusterB from Dev Spaces
- Switching between clusters
- Verifying connections and permissions

### [Step 2: Create PostgreSQL Dump](./02-create-postgres-dump.md)
- Finding and connecting to Redshift pods
- Creating PostgreSQL dumps in pod `/tmp/` directory
- Various dump options (compressed, schema-only, etc.)
- Verification and troubleshooting

### [Step 3: Copy Dump to Dev Spaces](./03-copy-dump-to-devspaces.md)
- Copying files from pods to Dev Spaces
- File integrity verification
- Multi-cluster organization
- Cleanup and best practices

## Quick Start

For experienced users, here's the complete workflow:

```bash
# 1. Get login token from OCP console and connect
oc login --token=sha256~your-token --server=https://api.cluster-a.example.com:6443
oc project your-namespace

# 2. Create dump in pod
POD_NAME=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
oc exec $POD_NAME -- pg_dump -h localhost -U postgres -d your_database_name > /tmp/database_dump_$(date +%Y%m%d_%H%M%S).sql

# 3. Copy to Dev Spaces
DUMP_FILE=$(oc exec $POD_NAME -- ls -t /tmp/*.sql | head -1)
oc cp $POD_NAME:$DUMP_FILE ./$(basename $DUMP_FILE)

# 4. Verify and cleanup
du -h ./$(basename $DUMP_FILE)
oc exec $POD_NAME -- rm $DUMP_FILE
```

## Multi-Cluster Workflow

### Working with Both Clusters

```bash
# Connect to ClusterA
oc login --token=sha256~cluster-a-token --server=https://api.cluster-a.example.com:6443
# ... perform dump operations ...

# Switch to ClusterB  
oc login --token=sha256~cluster-b-token --server=https://api.cluster-b.example.com:6443
# ... perform dump operations ...
```

### Organized File Storage

```bash
# Create cluster-specific directories
mkdir -p ~/database-dumps/{cluster-a,cluster-b}/$(date +%Y-%m)

# Store dumps with cluster identification
oc cp $POD_NAME:$DUMP_FILE ~/database-dumps/cluster-a/$(date +%Y-%m)/cluster-a_$(basename $DUMP_FILE)
```

## Complete Automation Script

```bash
#!/bin/bash
# automated-dump-workflow.sh

# Configuration
CLUSTER_URL="https://api.cluster-a.example.com:6443"
LOGIN_TOKEN="your-token-here"
NAMESPACE="your-namespace"
DATABASE_NAME="your_database_name"

echo "=== Automated PostgreSQL Dump Workflow ==="

# Step 1: Connect
echo "Connecting to OCP cluster..."
oc login --token=$LOGIN_TOKEN --server=$CLUSTER_URL
oc project $NAMESPACE

# Step 2: Create dump
echo "Creating database dump..."
POD_NAME=$(oc get pods -l app=redshift -o jsonpath='{.items[0].metadata.name}')
DUMP_FILENAME="database_dump_$(date +%Y%m%d_%H%M%S).sql"
oc exec $POD_NAME -- pg_dump -h localhost -U postgres -d $DATABASE_NAME > /tmp/$DUMP_FILENAME

# Step 3: Copy to Dev Spaces
echo "Copying dump to Dev Spaces..."
oc cp $POD_NAME:/tmp/$DUMP_FILENAME ./$DUMP_FILENAME

# Verification
echo "Verifying dump..."
if [ -f "./$DUMP_FILENAME" ]; then
    echo "✅ Success! Dump size: $(du -h ./$DUMP_FILENAME | cut -f1)"
    
    # Cleanup
    oc exec $POD_NAME -- rm /tmp/$DUMP_FILENAME
    echo "✅ Pod cleanup completed"
else
    echo "❌ Dump failed"
    exit 1
fi

echo "=== Workflow Complete ==="
```

## Troubleshooting Guide

### Common Issues

| Issue | Solution | Reference |
|-------|----------|-----------|
| Cannot connect to cluster | Get fresh login token from web console | [Step 1](./01-ocp-login-connection.md#troubleshooting-login-issues) |
| Pod not found | Verify namespace and pod labels | [Step 2](./02-create-postgres-dump.md#find-and-connect-to-redshift-pod) |
| Permission denied in /tmp | Try `/var/tmp` or check pod permissions | [Step 2](./02-create-postgres-dump.md#troubleshooting) |
| Copy operation fails | Check network connectivity and file size | [Step 3](./03-copy-dump-to-devspaces.md#troubleshooting-copy-issues) |
| Large file timeouts | Use compression and increase timeouts | [Step 2](./02-create-postgres-dump.md#large-database-timeouts) |

### Getting Help

For detailed troubleshooting steps, refer to the individual step documentation:
- **Connection issues**: See Step 1 troubleshooting section
- **Dump creation issues**: See Step 2 troubleshooting section  
- **File copy issues**: See Step 3 troubleshooting section

## Security Considerations

- **Login tokens** are temporary and should not be stored in scripts
- **Database dumps** may contain sensitive data - handle appropriately
- **Clean up** dump files from pods after copying to prevent disk space issues
- **Verify file integrity** after each copy operation
- **Use compression** for large dumps to reduce transfer time and storage

## File Organization Best Practices

```
~/database-dumps/
├── cluster-a/
│   ├── 2025-01/
│   │   ├── cluster-a_database_dump_20250115_143022.sql
│   │   └── cluster-a_database_dump_20250120_091500.sql
│   └── 2025-02/
└── cluster-b/
    ├── 2025-01/
    └── 2025-02/
```

## Support

For questions or issues with this documentation:

1. **Check the troubleshooting sections** in each step
2. **Verify prerequisites** are met for your specific environment
3. **Test with small databases** first before working with large production data
4. **Keep backups** of important dumps

## Contributing

To update this documentation:
1. Modify the relevant step file
2. Update this main README if workflow changes
3. Test all commands in a development environment
4. Update troubleshooting sections based on new issues discovered

---

**Next**: Start with [Step 1: OCP Login and Connection](./01-ocp-login-connection.md)
