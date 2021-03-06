---
title: Cleanup
weight: 3900
indent: true
---

# Cleaning up a Cluster
If you want to tear down the cluster and bring up a new one, be aware of the following resources that will need to be cleaned up:
- `rook-ceph` namespace: The Rook operator and cluster created by `operator.yaml` and `cluster.yaml` (the cluster CRD)
- `/var/lib/rook`: Path on each host in the cluster where configuration is cached by the ceph mons and osds

Note that if you changed the default namespaces or paths in the sample yaml files, you will need to adjust these namespaces and paths throughout these instructions.

If you see issues tearing down the cluster, see the [Troubleshooting](#troubleshooting) section below.

If you are tearing down a cluster frequently for development purposes, it is instead recommended to use an environment such as Minikube that can easily be restarted without worrying about any of these steps.

## Delete the Block and File artifacts
First you will need to clean up the resources created on top of the Rook cluster.

These commands will clean up the resources from the [block](ceph-block.md#teardown) and [file](ceph-filesystem.md#teardown) walkthroughs (unmount volumes, delete volume claims, etc). If you did not complete those parts of the walkthrough, you can skip these instructions:
```console
kubectl delete -f ../wordpress.yaml
kubectl delete -f ../mysql.yaml
kubectl delete -n rook-ceph cephblockpool replicapool
kubectl delete storageclass rook-ceph-block
kubectl delete -f kube-registry.yaml
```

## Delete the Cluster CRD
After those block and file resources have been cleaned up, you can then delete your Rook cluster. This is important to delete **before removing the Rook operator and agent or else resources may not be cleaned up properly**.
```console
kubectl -n rook-ceph delete cephcluster rook-ceph
```

Verify that the cluster CRD has been deleted before continuing to the next step.
```
kubectl -n rook-ceph get cephcluster
```

## Delete the Operator
This will begin the process of all cluster resources being cleaned up, after which you can delete the operator and related resources such as the agent and discover daemonsets with the following:
```console
kubectl delete -f operator.yaml
```

Optionally remove the rook-ceph namespace if it is not in use by any other resources.
```
kubectl delete namespace rook-ceph
```

## Delete the data on hosts
IMPORTANT: The final cleanup step requires deleting files on each host in the cluster. All files under the `dataDirHostPath` property specified in the cluster CRD will need to be deleted. Otherwise, inconsistent state will remain when a new cluster is started.

Connect to each machine and delete `/var/lib/rook`, or the path specified by the `dataDirHostPath`.

In the future this step will not be necessary when we build on the K8s local storage feature.

If you modified the demo settings, additional cleanup is up to you for devices, host paths, etc.

Disks on nodes used by Rook for osds can be reset to a usable state with the following methods:
```sh
#!/usr/bin/env bash
DISK="/dev/sdb"
# Zap the disk to a fresh, usable state (zap-all is important, b/c MBR has to be clean)
# You will have to run this step for all disks.
sgdisk --zap-all $DISK

# These steps only have to be run once on each node
# If rook sets up osds using ceph-volume, teardown leaves some devices mapped that lock the disks.
ls /dev/mapper/ceph-* | xargs -I% -- dmsetup remove %
# ceph-volume setup can leave ceph-<UUID> directories in /dev (unnecessary clutter)
rm -rf /dev/ceph-*
```

## Troubleshooting
If the cleanup instructions are not executed in the order above, or you otherwise have difficulty cleaning up the cluster, here are a few things to try.

The most common issue cleaning up the cluster is that the `rook-ceph` namespace or the cluster CRD remain indefinitely in the `terminating` state. A namespace cannot be removed until all of its resources are removed, so look at which resources are pending termination.

Look at the pods:
```
kubectl -n rook-ceph get pod
```
If a pod is still terminating, you will need to wait or else attempt to forcefully terminate it (`kubectl delete pod <name>`).

Now look at the cluster CRD:
```
kubectl -n rook-ceph get cephcluster
```
If the cluster CRD still exists even though you have executed the delete command earlier, see the next section on removing the finalizer.

### Removing the Cluster CRD Finalizer
When a Cluster CRD is created, a [finalizer](https://kubernetes.io/docs/tasks/access-kubernetes-api/extend-api-custom-resource-definitions/#finalizers) is added automatically by the Rook operator. The finalizer will allow the operator to ensure that before the cluster CRD is deleted, all block and file mounts will be cleaned up. Without proper cleanup, pods consuming the storage will be hung indefinitely until a system reboot.

The operator is responsible for removing the finalizer after the mounts have been cleaned up.
If for some reason the operator is not able to remove the finalizer (ie. the operator is not running anymore), you can delete the finalizer manually with the following command:

```
kubectl -n rook-ceph patch cephclusters.ceph.rook.io rook-ceph -p '{"metadata":{"finalizers": []}}' --type=merge
```

Within a few seconds you should see that the cluster CRD has been deleted and will no longer block other cleanup such as deleting the `rook-ceph` namespace.
