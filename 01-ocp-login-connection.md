# Step 1: OCP Login and Connection

This document covers how to obtain login tokens from OpenShift Container Platform (OCP) and connect to your clusters from Dev Spaces.

## Prerequisites

- Access to OCP web console for both ClusterA and ClusterB
- Dev Spaces instance with OCP CLI tools installed

## Getting Login Token from OCP Web Console

### Method 1: Using OCP Web Console

1. **Access the OCP Web Console** for your target cluster (ClusterA or ClusterB)
2. **Click on your username** in the top-right corner of the console
3. **Select "Copy login command"** from the dropdown menu
4. **Click "Display Token"** on the new page that opens
5. **Copy the complete `oc login` command** - it will look like:
   ```bash
   oc login --token=sha256~abcd1234... --server=https://api.cluster-a.example.com:6443
   ```

### From Dev Spaces Terminal

Paste and run the login command you copied from the OCP console:

```bash
# Paste the complete oc login command from the web console
oc login --token=sha256~your-actual-token --server=https://api.cluster-a.example.com:6443

# Verify connection
oc whoami
oc project
```

## Switching Between Clusters

### Connect to ClusterA
```bash
# Get login command from ClusterA web console and paste here
oc login --token=sha256~cluster-a-token --server=https://api.cluster-a.example.com:6443
```

### Connect to ClusterB
```bash
# Get login command from ClusterB web console and paste here
oc login --token=sha256~cluster-b-token --server=https://api.cluster-b.example.com:6443
```

## Verification Commands

After connecting to any cluster, verify your access:

```bash
# Check current user
oc whoami

# List available projects/namespaces
oc get projects

# Check current project
oc project

# View cluster info
oc cluster-info
```

## Switch to Target Namespace

```bash
# List available namespaces (optional)
oc get namespaces

# Switch to the namespace containing your Redshift pod
oc project <your-namespace>

# Verify the Redshift pod is running
oc get pods | grep redshift
```

## Troubleshooting Login Issues

### Token Expired
If you get authentication errors:
1. Return to the OCP web console
2. Get a fresh login command following the steps above
3. Run the new login command

### Wrong Cluster
```bash
# Check which cluster you're connected to
oc cluster-info

# If wrong cluster, get login command for correct cluster
```

### Permission Issues
```bash
# Check your permissions in current project
oc auth can-i --list

# Verify cluster admin access
oc auth can-i "*" "*" --all-namespaces
```

## Security Notes

- Login tokens are temporary and expire
- Don't store tokens in scripts or version control
- Always log out when switching between sensitive environments:
  ```bash
  oc logout
  ```

## Next Step

Once connected and in the correct namespace, proceed to [Step 2: Create PostgreSQL Dump](./02-create-postgres-dump.md)
