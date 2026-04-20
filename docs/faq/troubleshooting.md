---
title: Troubleshooting
sidebar_position: 4
---

## Troubleshooting

### I am getting a "server-side apply failed" error

Helm 4 uses [server-side apply](https://kubernetes.io/docs/reference/using-api/server-side-apply/) (SSA) by default when installing and upgrading releases.
When a server-side apply operation fails, you see an error like:

```
Error: INSTALLATION FAILED: server-side apply failed for object default/my-deployment apps/v1, Kind=Deployment: <error details>
```

The most common cause is **field ownership conflicts**: another controller or process manages the same fields that your chart is trying to update.
SSA tracks which manager owns each field, and conflicts occur when multiple managers try to set the same field.

To resolve server-side apply errors:

1. **Use `--force-conflicts`**: If another manager owns fields that your chart needs to update, add `--force-conflicts` to override the conflict and take ownership:

   ```console
   $ helm upgrade my-release my-chart --force-conflicts
   ```

2. **Disable server-side apply**: As a workaround, you can disable SSA for the operation:

   ```console
   $ helm install my-release my-chart --server-side=false
   ```

   This reverts to client-side apply, which does not enforce field ownership.
   However, you lose the benefits of server-side field management.

For more information about server-side apply, see [HIP-0023](/community/hips/hip-0023) and the [Kubernetes SSA documentation](https://kubernetes.io/docs/reference/using-api/server-side-apply/).

### I am seeing a warning about duplicate list-map entries

When Helm applies your chart with server-side apply, you may see a warning like:

```
level=WARN msg="deduplicated list-map entries in manifest; please remove duplicates from the chart" name=myapp namespace=default gvk=apps/v1, Kind=Deployment
```

This warning means your chart rendered a manifest with duplicate entries in a list field (such as environment variables, volumes, or containers).
Helm automatically deduplicates these entries before sending the manifest to Kubernetes, keeping the **last occurrence** of each duplicate name.
This matches Kubernetes client-side apply behavior, which also used the last value when duplicates were present.

Common list types affected:

- `spec.containers[].env` (environment variables)
- `spec.initContainers[].env`
- `spec.volumes`
- `spec.containers[].volumeMounts`
- `spec.containers` (container names)

**To fix this warning**, update your chart templates to remove duplicate entries.
For example, if your chart defines the same environment variable twice with different values, restructure the template logic so each variable name appears only once.
A dictionary pattern in your templates can help prevent accidental duplicates when merging values from multiple sources.
