# TODO: HCP CLI Cleanup Reliability Issue

**Date**: 2026-01-17
**Priority**: High
**Status**: Open

## Summary

The `hcp destroy cluster aws` command fails to reliably clean up AWS resources when the HyperShift operator or guest cluster control plane becomes unavailable. This leaves orphaned AWS resources (VPCs, subnets, IAM roles, Route53 zones, OIDC providers) that must be manually cleaned up.

## Problem Description

When destroying a hosted cluster, the following issues occur:

1. **CloudResourcesDestroyed condition stays Unknown**: The condition never transitions to True, causing the finalizer to never be removed automatically.

2. **Missing command parameters**: The `hcp destroy cluster aws` command works better when `--infra-id` and `--base-domain` are provided (matching the create command), but these are not well-documented as required for destroy.

3. **Orphaned AWS resources**: Even after the HostedCluster CR is deleted, the following resources remain:
   - VPCs and subnets
   - NAT Gateways and Elastic IPs
   - IAM roles and instance profiles
   - OIDC providers
   - Route53 hosted zones

4. **Heartbeat mechanism not working**: The CPO should detect when HCCO stops updating the CloudResourcesDestroyed condition (heartbeat timeout), but this doesn't seem to trigger cleanup in all cases.

## Observed Behavior

```
kubectl get hostedcluster -n clusters <name> -o jsonpath='{.status.conditions[?(@.type=="CloudResourcesDestroyed")]}'

{
  "lastTransitionTime": "2026-01-17T16:36:00Z",
  "message": "",
  "observedGeneration": 4,
  "reason": "StatusUnknown",
  "status": "Unknown",
  "type": "CloudResourcesDestroyed"
}
```

The condition stays at "Unknown" indefinitely, even after:
- Control plane pods are deleted
- Control plane namespace is deleted
- hcp destroy command completes

## Expected Behavior

1. The `hcp destroy cluster aws` command should clean up ALL AWS resources it created
2. If cleanup fails, it should report which resources remain
3. The CloudResourcesDestroyed condition should transition to True (or provide clear error) within a reasonable timeout
4. The finalizer should be removed after cleanup completes (or times out with clear status)

## Workaround (Implemented in hypershift-automation)

We've implemented a comprehensive tag-based AWS cleanup fallback in the ansible playbook that:
1. Detects stuck finalizers (CloudResourcesDestroyed = True OR Unknown)
2. Removes the finalizer to allow Kubernetes deletion
3. Cleans up orphaned AWS resources using tag-based filtering:
   - EC2 instances, EBS volumes, security groups
   - VPCs, subnets, route tables
   - NAT gateways, internet gateways, elastic IPs
   - IAM roles, instance profiles, OIDC providers
   - Route53 hosted zones

All cleanup uses the `kubernetes.io/cluster/<cluster-name>=owned` tag for safety.

## Related Issues/PRs

1. [OCPBUGS-62949: Fix cloud resource cleanup blocking](https://github.com/openshift/hypershift/pull/7024)
2. [OCPBUGS-23126: Fix deletion hang with HCCO heartbeat](https://github.com/openshift/hypershift/pull/3234)
3. [Heartbeat mechanism for CloudResourcesDestroyed](https://github.com/openshift/hypershift/pull/1947)

## Action Items

- [ ] Create issue in openshift/hypershift repository describing this behavior
- [ ] Include reproduction steps and logs
- [ ] Reference related fixed issues to check if regression occurred
- [ ] Verify HyperShift operator version (we're using `:latest` which is not recommended)
- [ ] Consider pinning to a specific version

## Files Modified (Workaround)

- `hcp/tasks/hcp-destroy-cluster.yml` - Added comprehensive AWS cleanup
- `hcp/defaults/main.yml` - Added configurable timeouts and cleanup flags

## Research Document

See `docs/hypershift-cleanup-research.md` in kagenti_hypershift_ci for detailed analysis.
