---
title: Skip Volume Ownership Change
authors:
  - "@gnuified"
owning-sig: sig-storage
participating-sigs:
  - sig-auth
reviewers:
  - "@msau42"
  - "@liggit"
  - "@tallclair"
approvers:
  - "@saad-ali"
editor: TBD
creation-date: 2020-01-20
last-updated: 2020-01-20
status: implementable
see-also:
replaces:
superseded-by:

---

# Skip Volume Ownership Change

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Summary](#summary)
* [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
* [Proposal](#proposal)
    * [Implementation Details/Notes/Constraints [optional]](#implementation-detailsnotesconstraints-optional)
    * [Risks and Mitigations](#risks-and-mitigations)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)
* [Drawbacks [optional]](#drawbacks-optional)
* [Alternatives [optional]](#alternatives-optional)

## Summary

Currently before a volume is bind-mounted inside a container the permissions on
that volume are changed recursively to the provided fsGroup value.  This change
in ownership can take an excessively long time to complete, especially for very
large volumes (>=1TB) as well as a few other reasons detailed in [Motivation].
To solve this issue we will add a new field called `pod.Spec.SecurityContext.VolumePermissionChangePolicy` and
allow the user to specify whether they want the ownership change to occur.

## Motivation

When a volume is mounted on the node, we recursively change permissions of volume
before bind mounting the volume inside container. The reason of doing this is to ensure
that volumes are readable/writable by provided fsGroup.

But this presents following problems:
 - An application(many popular databases) which is sensitive to permission bits changing
   underneath may refuse to start whenever volume being used inside pod gets mounted on
   different node.
 - If volume has a large number of files, performing recursive `chown` and `chmod`
   could be slow and could cause timeout while starting the pod.

### Goals

 - Allow volume ownership and permission to be skipped during mount

### Non-Goals

 - In some cases if user brings in a large enough volume from outside, the first time ownership and permission change still could take lot of time.
 - On SELinux enabled distributions we will still do recursive chcon whenever applicable and handling that is outside the scope.

## Proposal

We propose that an user can optionally opt-in to skip recursive ownership(and permission) change on the volume if volume already has right permissions.

### Implementation Details/Notes/Constraints [optional]

When creating a pod, we propose that `pod.Spec.SecurityContext` field expanded to include a new field called `VolumePermissionChangePolicy` which can have following possible values:

 - `NoChange` --> Don't change permissions and ownership. This could be useful to users when security policy of a cluster prevents users from removing `fsGroup` from their pod's `SecurityContext`.
 - `Always` --> Always change the permissions and ownership to match fsGroup. This is the current behavior and it will be the default one when this proposal is implemented. Algorithm that performs permission change however will be changed to perform permission change of top level directory last.
 - `OnDemand` --> Only perform permission change if permissions of top level directory does not match with expected permissions.

```go
type PodVolumePermissionChangePolicy string

const(
    NeverChangeVolumePermission PodVolumePermissionChangePolicy = "NoChange"
    OnDemandChangeVolumePermission PodVolumePermissionChangePolicy = "OnDemand"
    AlwaysChangeVolumePermission PodVolumePermissionChangePolicy = "Always"
)

type PodSecurityContext struct {
    // VolumePermissionChangePolicy ← new field
    // + optional
    VolumePermissionChangePolicy *PodVolumePermissionChangePolicy
}
```

### Risks and Mitigations

- One of the risks is if user volume's permission was previously changed using old algorithm(which changes permission of top level directory first) and user opts in for `OnDemand` `VolumePermissionChangePolicy` then we can't distinguish if the volume was previously only partially recursively chown'd.


## Graduation Criteria

* Alpha in 1.18 provided all tests are passing and gated by the feature Gate
   ConfigurableVolumeFilePermissions and set to a default of `False`

* Beta in 1.19 with design validated by at least two customer deployments
  (non-production), with discussions in SIG-Storage regarding success of
  deployments.  A metric will be added to report time taken to perform a
  volume ownership change.
* GA in 1.20, with Node E2E tests in place tagged with feature Storage


[umbrella issues]: https://github.com/kubernetes/kubernetes/issues/69699

### Test Plan

A test plan will consist of the following tests

* Basic tests including a permutation of the following values
  - PodSecurityContext.VolumePermissionChangePolicy (Never/OnDemand/Always)
  - PersistentVolumeClaimStatus.FSGroup (matching, non-matching)
  - Volume Filesystem existing permissions (none, matching, non-matching, partial-matching?)
* E2E tests


### Monitoring

We will add a metric that measures the volume ownership change times.

## Implementation History

- 2020-01-20 Initial KEP pull request submitted

## Drawbacks [optional]


## Alternatives [optional]

## Infrastructure Needed [optional]
