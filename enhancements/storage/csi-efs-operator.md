---
title: AWS EFS CSI driver operator via OLM
authors:
  - "@jsafrane"
reviewers:
  - 2uasimojo # from OCP Dedicated team (i.e. "the customer")
  - bertinatto # from OCP storage
approvers:
  - bbennett # As pillar lead?

creation-date: 2021-03-09
last-updated: 2021-09-22
status: implemented
see-also:
  - "/enhancements/storage/csi-driver-install.md"
replaces:
superseded-by:
---

# AWS EFS CSI driver operator via OLM

## Release Signoff Checklist

- [x] Enhancement is `implementable`
- [x] Design details are appropriately documented from clear requirements
- [x] Test plan is defined
- [x] Operational readiness criteria is defined
- [x] Graduation criteria for dev preview, tech preview, GA
- [x] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

AWS EFS CSI driver (+ corresponding operator) is an optional OCP component. It should be installed through OLM, when
users opts-in.

This document describes existing (unsupported) solution and how to turn it into a supported one, both from code (and
build and shipment), and from user perspective.

## Motivation

Some of our customers want to use Amazon Elastic FileSystem in their clusters. At the same time, not all customers want
to use it, so it should not be installed by default as our AWS EBS CSI driver.

Despite written by Red Hat employees, the existing CSI driver operator
[aws-efs-operator](https://github.com/openshift/aws-efs-operator) uses upstream CSI driver, upstream CSI sidecars and
[the operator itself](https://quay.io/repository/app-sre/aws-efs-operator?tab=tags) is not released through
registry.redhat.com and therefore cannot be supported by us.

### Goals

* Allow users to use AWS EFS volumes in a supported way.
* Allow existing users of community-supported aws-efs-operator to update to Red Hat supported solution (mostly manually).

### Non-Goals

* Change the EFS CSI driver in any way. All missing functionality there must go through RFE process.
* Keep existing community-supported aws-efs-operator features (namely, `SharedVolume` in
  `aws-efs.managed.openshift.io` API group will not be preserved). See below.

## Existing situation

### [aws-efs-operator](https://github.com/openshift/aws-efs-operator)

Summary of current [aws-efs-operator](https://github.com/openshift/aws-efs-operator) (no judging, just stating the facts):

* It's written using `controller-runtime`.
* Once the operator is installed by OLM, the operator automatically installs the CSI driver, without any CRD/CR.
  * As consequence, user cannot un-install the CSI driver easily.
* By default, it is installed in `openshift-operators` namespace and the CSI driver runs there too.
* It does not create EFS volumes in AWS! It's up to the cluster admin to create EFS volume in AWS, figure in which VPC
  it should be available and set correct security group. See [this KB article](https://access.redhat.com/articles/5025181)
  for suggested procedure.
* It offers `SharedVolume` CRD in `aws-efs.managed.openshift.io/v1alpha1` API group to create PV + PVC for an EFS
  volume. The EFS volume still needs to be created manually, see the previous bullet!
  * All that `SharedVolume` really does is that it creates PV with proper settings + a PVC. This allows unprivileged
    users to create PVs. Cluster admins need to be careful to give permissions to use this CRD only to trusted users.
  * PVCs created by `SharedVolume` have predetermined name, i.e. it may be hard to use them as a part of a StatefulSet
    or a Helm chart, which may require their own PVC name.
  * Using `SharedVolume` is purely optional! Cluster admin may create PVs for EFS volumes manually, it's not harder
    than using `SharedVolume`.

### AWS EFS CSI driver

AWS EFS CSI driver is different from the other CSI drivers we ship. Listing the main differences here for completeness.

* It implements dynamic provisioning in non-standard way. Cluster admin must manually create an EFS share in the cloud
  (and figure out related networking and firewalls). The EFS CSI driver will provision volumes out of it as
  *subdirectories* of it. The CSI driver cannot create EFS shares on its own.
  * The driver creates an [AccessPoint](https://docs.aws.amazon.com/efs/latest/ug/efs-access-points.html) for each
    subdirectory for easier usage.
  * AWS has a limit of 120 Access Points per EFS volume, i.e. only 120 PVs can be created for a single StorageClass
    (= single EFS volume). [Upstream issue](https://github.com/kubernetes-sigs/aws-efs-csi-driver/issues/310).
  * Proper dynamic provisioning can come in the future, dealing with the networking and security groups is the hardest
    part there.
* AWS EFS volumes do not have any size enforcement. They can hold as much data as apps put there and someone will have to
  pay for the storage. OCP offers standard metrics for amount of data on PVCs, it's up to the cluster admin to create
  alerts if the consumption gets too high.
* It uses [efs-utils](https://github.com/aws/efs-utils) to provide encryption in transport, which is not included in
  RHEL. We will need to package it, either as an image or a RPM package.

## Proposal
High level only, see design details below.

1. Write a new aws-efs-csi-driver-operator from scratch using [CSI driver installation functions of library-go](https://github.com/openshift/library-go/tree/master/pkg/operator/csi/csicontrollerset).
   * This allows us to share well tested code to install CSI drivers and implement bugfixes / new features at a single
     place.
   * It will add common features of CSI driver operators which are missing in the current community operator, such as
     proxy injection and restarting the drivers when cloud credentials change.

2. Let aws-efs-csi-driver-operator to watch `ClusterCSIDriver` CR named `efs.csi.aws.com`.
   * `ClusterCSIDriver` CRD is already provided by OCP and is used to install all other CSI drivers that OCP ships.
     * Every CSI driver operator watches only its CR, with [a defined name](https://github.com/openshift/api/blob/4b79815405ec40f1d72c3a74bae0ae7da543e435/operator/v1/0000_90_cluster_csi_driver_01_config.crd.yaml#L39).
       They don't act on or modify CRs of other CSI drivers.
   * Users must explicitly create the CR to install the CSI driver to allow users to uninstall the driver if they don't
     like it.

3. Remove `SharedVolume` CRD.
   * The CRD may be insecure - it allows anyone who has permission to create CRs to "steal" data from any
     EFS share by crafting `SharedVolume` pointing to it.
   * The CR just creates a PV+PVC, cluster admin may grant permissions to create PVs to trusted people, which is more
     explicit.
   * Automatic update from the community operator to the supported operator will not be possible. Cluster admin has
     to stop all pods that use EFS volumes, remove the community operator + CSI driver and install the new one. PVs/PVCs
     should not require any changes.

4. Ship the operator + CSI driver through ART pipeline.
   * Most of the operator just installs / removes CSI driver, i.e. manages driver's RBAC rules, CredentialsRequest,
     DaemonSet, Deployment, CSIDriver and StorageClasses. See existing
     [AWS EBS operator](https://github.com/openshift/aws-ebs-csi-driver-operator/tree/master/assets) for objects it
     manages, EFS should be very similar in this regard.

5. Ship [efs-utils](https://github.com/aws/efs-utils) as a base image for the EFS CSI driver image.

6. Update installer to delete Access Points and EFS volumes that have `owner` tag when destroying a cluster.
   * Existence of EFS volume using a VPC / Subnets actively prevents installer to delete them.
     `openshift-install destroy cluster` gets stuck at VPC (or subnet?) deletion, because AWS does not allow to delete
     a network that's used.
   * This way, cluster admins can tag the volumes to be deleted when the cluster is deleted (this is opt in!).
     At least CI should use this approach to ensure there are no leftovers.

### User Stories

#### Story 1: Installation

1. Cluster-admin installs EFS CSI driver operator via OLM.
2. Cluster-admin creates ClusterCSIDriver instance for the driver (`efs.csi.aws.com`).
3. Cluster-admin creates EFS volumes in AWS.
   1. Including mount targets creation.
   2. Including firewall configuration to allow access to the EFS volume from all cluster nodes.
4. Cluster-admin makes the EFS volume available to OCP users.
   1. Either the volume EFS volume will be one PV.
   2. Or the cluster-admin creates a StorageClass that will provision PVs as subdirectories of the EFS volume.
5. User (or StatefulSet or Template etc) can create PVC that is either bound to the pre-provisioned PV or a new EFS subdir + PV is dynamically provisioned.

#### Story 2: Un-installation
1. Cluster-admin ensures that nothing uses EFS PVs, e.g. by stopping all application pods that use EFS.
2. Clusted-admin deletes the PVs. This is not required, but it will help the pods to be stopped.
3. Cluster-admin can optionally back-up EFS volumes.
4. Cluster-admin deletes ClusterCSIDriver `efs.csi.aws.com`.
   1. The operator will remove the CSI driver and allow ClusterCSIDriver to be deleted.
5. Cluster-admin deletes EFS volumes in AWS.
6. Cluster admin un-installs the EFS CSI driver operator. This should be done only after the CSI driver itself was already removed.
   Removing the operator does not automatically remove the driver!

### Implementation Details/Notes/Constraints [optional]

### Risks and Mitigations

Currently, all cloud-based CSI drivers shipped by OCP are installed when a cluster is installed on the particular cloud.
AWS EFS will not be installed by default. We will include a note about opt-in installation in docs.

After installation of the new operators, either as a fresh install or upgrading the existing one, users must create the
CR when the operator is installed on a new cluster
([example of CR for AWS EBS](https://github.com/openshift/cluster-storage-operator/blob/master/assets/csidriveroperators/aws-ebs/10_cr.yaml),
EFS will be very similar). This is something that was not necessary in the previous releases of
the operator. We will add a release note and introduce an (info) alert to remind admins to create the CR.

## Design Details

### Rewrite to library-go & ClusterCSIDriver CRD

This should be pretty straightforward. We already ship number of CSI drivers that use library-go. Example usage:
[AWS EBS CSI driver](https://github.com/openshift/aws-ebs-csi-driver-operator/blob/383a7638b37a4b9a9831c7747e8c499eedcf030f/pkg/operator/starter.go#L67).
EFS CSI driver operator should not be much more complicated.

Library-go does not support un-installation of CSI driver (and static resources created by `StaticResourcesController`),
this must be implemented there.

Note: When the *operator* is un-installed, nothing happens to the CRD, CR, the installed CSI driver, existing PVs and
PVCs. Everything is working, just not managed by the operator. When the *operand* is un-instaled by removing the
operator CR, the driver will be removed. Existing PVs and PVCs will be still present in the API server,
however, they won't be really usable. Nobody can mount / unmount EFS volumes without the driver.

### Upgrade from the community operator

Prerequisites:
* The community aws-efs-operator is installed in OCP 4.8 cluster in `openshift-operators` namespace. The CSI driver runs
in the same namespace too.
  * The operator `Subscription` contains the community catalog as the operator source.

Expected workflow (confirmed with OLM team):

1. User upgrades to 4.9. Nothing interesting happens at this point, because the operator `Subscription` still points
   to the community source. The "old" operator is still running.
2. User must stop all Pods that use EFS CSI volumes. The next step will make all volume mounts unavailable!
3. User un-installs the operator + the CSI driver via OLM.
4. User installs the new operator via OLM and creates `ClusterCSIDriver` CR for it.
5. The new CSI driver "adopts" PVs + PVCs of the old operator - no changes are needed there.
6. User can start Pods that use EFS volumes.

### [efs-utils](https://github.com/aws/efs-utils)

We need to get efs-utils into the container with the EFS CSI driver to enable data encryption in transport. efs-utils
are just two Python scripts, which create / monitor stunnel connections between OCP nodes and NFS server somewhere in
AWS cloud.

We've chosen to use base image approach. There are two reasons: 1) we know the drill, and 2) we don't want
to support a generic RPM that can be used outside of OCP.

1. Fork `github.com/aws/efs-utils` into `github.com/openshift/aws-efs-utils`.
2. Create a new base image with the utilities, say `ose-aws-efs-utils-base`:
   ```Dockerfile
   FROM: ocp-4.9:base
   RUN yum install stunnel python <and any other deps>
   COPY <the utilities> /usr/bin
   ```
3. The EFS CSI driver then uses it as the base image:
   ```Dockerfile
   FROM registry.ci.openshift.org/openshift/release:golang-1.15 AS builder
   ... build the image ...

   FROM ose-aws-efs-utils-base
   COPY --from=builder <the driver binary> /usr/bin/
   ```

It would be nice if we did not ship `ose-aws-efs-utils-base` anywhere, so people can consume the EFS utils only through
the EFS operator. **We do not want to maintain it for everyone.**

### Code organization

* Create new  `github.com/openshift/aws-efs-csi-driver-operator` repository for the operator code and start from
  scratch there.
* Leave existing `github.com/openshift/aws-efs-operator` untouched, for the community version that may be maintained
  for older OCP releases.
* Fork the CSI driver into `github.com/openshift/aws-efs-csi-driver`.
* Fork `github.com/aws/efs-utils` into `github.com/openshift/aws-efs-utils`.

### Open Questions

**How to actually hide the community operator in 4.9 and newer operator catalogs?** We don't want to have two operators
in the catalog there. We still want to offer the community operator in 4.8 and earlier though.

* Right now it seems there will be two operators available in OLM and it's not that bad.

### Test Plan

* Automatic jobs:
  * Run CSI certification tests with the CSI driver:
     1. Install OCP on AWS.
     2. Install the operator.
     3. Create AWS EFS share (this may require extra privileges in CI and figure out the networking!)
     4. Set up dynamic provisioning in the driver (by creating a StorageClass).
     5. Run CSI certification tests.
  * Run upgrade tests:
     1. Install OCP on AWS.
     2. Install the operator.
     3. Create AWS EFS share (this may require extra privileges in CI and figure out the networking!)
     4. Create an dummy app that uses EFS share.
     5. Upgrade the EFS operator (via OLM).
     6. Check the app survived.

* Manual tests:
  * Upgrade from the community CSI driver. With some application pods using volumes provided by the community driver.
    There should be no (long) disruption of the application.

### Graduation Criteria

#### Dev Preview -> Tech Preview

* The functionality described above is implemented.
* At least manual testing passed (CI may need dynamic provisioning in the driver).
* End user documentation exists.

#### Tech Preview -> GA

* Test plan is implemented in our CI.
* All bus with urgent + high severity are fixed.
* The tech preview operator was available in at lest one OCP release.

#### Removing a deprecated feature

We remove support of `SharedVolume` CR. It was never officially supported by Red Hat, only by community on OperatorHub.
Although the community were Red Hat employees. On Jun 28 2021, there are 41 clusters with a SharedVolume CR. These will
need to use the community CSI driver operator + driver.

### Upgrade / Downgrade Strategy

Nothing special here, the operator will follow generic OLM upgrade path. We do not support downgrade of the operator.

### Version Skew Strategy

N/A, it's managed by OLM and allows to run on any OCP within the boundaries set by the operator metadata.

## Implementation History

## Drawbacks

## Alternatives (Not Implemented)

## Infrastructure Needed [optional]

We need to create / delete EFS shares in CI.

* This may require new permissions in CI.

