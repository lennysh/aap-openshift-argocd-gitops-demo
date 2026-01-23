# DevSpaces Deployment Lifecycle

## Requirements

1. **Initial Creation**: All resources must be created in the correct order
2. **Updates**: Resources must update cleanly without breaking
3. **Deletion**: When Application is deleted, ALL resources must be deleted
4. **Recreation**: If Application is recreated, everything must install cleanly

## Resource Order (by sync-wave)

- **sync-wave: "0"**: Role and RoleBinding (MUST exist before anything else)
- **sync-wave: "1"**: CSV cleanup Job (PreSync hook), ServiceAccount, RBAC for cleanup
- **sync-wave: "2"**: Subscription (creates operator)
- **sync-wave: "1"**: CheCluster (after operator is ready)

## Critical Rules

1. **Role and RoleBinding**:
   - MUST have sync-wave: "0" (created first)
   - MUST NOT be hooks (no hook finalizer)
   - MUST have all permissions needed (checlusters, jobs, serviceaccounts)
   - MUST be bound to ArgoCD service account

2. **CSV Cleanup Job**:
   - MUST be a PreSync hook (runs before main sync)
   - MUST have hook-delete-policy: HookSucceeded
   - MUST run AFTER Role/RoleBinding exist (sync-wave: "1")
   - MUST have permissions to delete CSVs and Subscriptions

3. **Finalizer**:
   - MUST be present on Application for cascade deletion
   - MUST delete all resources when Application is deleted
   - MUST NOT get stuck (Role/RoleBinding must not have hook finalizers)

4. **Labels**:
   - All resources MUST have labels for tracking
   - Helps with cleanup and identification

## Common Issues and Fixes

### Issue: "cannot patch resource jobs"
**Cause**: Role/RoleBinding don't exist or don't have job permissions
**Fix**: Ensure Role has batch/jobs permissions and RoleBinding exists

### Issue: "cannot delete checlusters"
**Cause**: Role/RoleBinding deleted before finalizer runs
**Fix**: Ensure Role/RoleBinding are NOT hooks (no hook finalizer)

### Issue: Resources not deleted on Application deletion
**Cause**: Finalizer not working or resources have blocking finalizers
**Fix**: Ensure finalizer is present, Role/RoleBinding are not hooks

### Issue: ResolutionFailed on Subscription
**Cause**: Orphaned CSV from previous deployment
**Fix**: CSV cleanup job deletes orphaned CSVs before Subscription is created
