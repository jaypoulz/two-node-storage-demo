# Test Results - LINSTOR Storage on OpenShift

This document tracks test results for the two-node storage demo.

## Test Environment

- **Storage Backend**: DRBD
- **Test Date**: October 31, 2025

---

## RWO Mode: Node Failover Tests

### Test 1: Cordon/Drain Mode

**Status**: ✅ PASS

**Description**: Kubernetes native graceful drain of node, pods migrate cleanly.

**Expected Behavior**:
- Node is cordoned (marked unschedulable)
- Pods are gracefully evicted
- LINSTOR releases volume on old node
- Pod reschedules on new node
- Volume mounts successfully on new node
- Writer continues writing with new node name

**Result**:
```
Test passes successfully. Pod migrates cleanly, volume remounts on new node,
and writes continue without interruption.
```

---

### Test 2: Graceful Shutdown Mode

**Status**: ❌ FAIL

**Description**: Graceful VM shutdown without draining workloads first.

**Expected Behavior**:
- Node powers down gracefully
- LINSTOR detects node failure
- LINSTOR promotes replica on surviving node
- Pod reschedules and mounts volume on new node

**Actual Behavior**:
Volume fails to mount on new node. This error occurred on the first test run but did not reproduce on a second attempt, suggesting a race condition:

```
Events:
  Normal   Scheduled               2m43s               default-scheduler        Successfully assigned storage-test/storage-demo-rwo-756bc6c68b-5nzfs to master-1
  Warning  FailedAttachVolume      2m44s               attachdetach-controller  Multi-Attach error for volume "pvc-4affbfd7-9e7b-4fa0-97db-57d21f6874cc" Volume is already exclusively attached to one node and can't be attached to another
  Normal   SuccessfulAttachVolume  104s                attachdetach-controller  AttachVolume.Attach succeeded for volume "pvc-4affbfd7-9e7b-4fa0-97db-57d21f6874cc"
  Warning  FailedMount             37s (x8 over 104s)  kubelet                  MountVolume.SetUp failed for volume "pvc-4affbfd7-9e7b-4fa0-97db-57d21f6874cc" : rpc error: code = Internal desc = NodePublishVolume failed for pvc-4affbfd7-9e7b-4fa0-97db-57d21f6874cc: mount failed: exit status 32
Mounting command: mount
Mounting arguments: -w-w -t xfs -o _netdev,nouuid /dev/drbd1000 /var/lib/kubelet/pods/44ea0d50-38d1-4d97-8c29-ee78dedd1061/volumes/kubernetes.io~csi/pvc-4affbfd7-9e7b-4fa0-97db-57d21f6874cc/mount
Output: mount: /var/lib/kubelet/pods/44ea0d50-38d1-4d97-8c29-ee78dedd1061/volumes/kubernetes.io~csi/pvc-4affbfd7-9e7b-4fa0-97db-57d21f6874cc/mount: /dev/drbd1000 is write-protected but explicit read-write mode requested.
```

**Root Cause**:
When the node is gracefully shutdown without draining first, the old pod doesn't terminate cleanly. LINSTOR/DRBD doesn't release the volume mount from the old node quickly enough, causing a timing issue where the new pod attempts to mount before the old mount is fully released. This results in a write-protected state preventing read-write access. The race condition nature (failed first test, passed second test) suggests the success depends on timing between node shutdown, volume release, and pod rescheduling.

**What Happens**:
When the node shuts down, LINSTOR initially detects the connection loss and puts the storage pool in a Warning state:

```
WARNING:
Description:
    No active connection to satellite 'master-0'

$ kubectl-linstor storage-pool list
╭─────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ StoragePool  ┊ Node     ┊ Driver   ┊ PoolName     ┊ FreeCapacity ┊ TotalCapacity ┊ State   ┊
╞═════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ vg1-thin     ┊ master-0 ┊ LVM_THIN ┊ vg1/vg1-thin ┊              ┊               ┊ Warning ┊
┊ vg1-thin     ┊ master-1 ┊ LVM_THIN ┊ vg1/vg1-thin ┊    59.88 GiB ┊     59.88 GiB ┊ Ok      ┊
╰─────────────────────────────────────────────────────────────────────────────────────────────────╯
```

**Post-Test Impact**:
This impact is **inconsistent** across test runs. Sometimes the storage pool recovers normally, other times it transitions to an Error state preventing further PVC provisioning.

When the error occurs:

```
$ kubectl-linstor storage-pool list
╭─────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ StoragePool  ┊ Node     ┊ Driver   ┊ PoolName     ┊ FreeCapacity ┊ TotalCapacity ┊ State ┊
╞═════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ vg1-thin     ┊ master-0 ┊ LVM_THIN ┊ vg1/vg1-thin ┊        0 KiB ┊         0 KiB ┊ Error ┊
┊ vg1-thin     ┊ master-1 ┊ LVM_THIN ┊ vg1/vg1-thin ┊    59.88 GiB ┊     59.88 GiB ┊ Ok    ┊
╰─────────────────────────────────────────────────────────────────────────────────────────────────╯

ERROR:
Description:
    Node: 'master-0', storage pool: 'vg1-thin' - Failed to query free space from storage pool
```

When storage recovers successfully (inconsistent behavior):

```
$ kubectl-linstor storage-pool list
╭─────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ StoragePool  ┊ Node     ┊ Driver   ┊ PoolName     ┊ FreeCapacity ┊ TotalCapacity ┊ State ┊
╞═════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ vg1-thin     ┊ master-0 ┊ LVM_THIN ┊ vg1/vg1-thin ┊    59.87 GiB ┊     59.88 GiB ┊ Ok    ┊
┊ vg1-thin     ┊ master-1 ┊ LVM_THIN ┊ vg1/vg1-thin ┊    59.87 GiB ┊     59.88 GiB ┊ Ok    ┊
╰─────────────────────────────────────────────────────────────────────────────────────────────────╯
```

**Workaround**:
- Use "Cordon/Drain" mode instead, which properly migrates workloads before shutdown

---

### Test 3: Destroy Mode (Hard Power-Off)

**Status**: ❌ FAIL

**Description**: Hard VM power-off simulating catastrophic hardware failure.

**Expected Behavior**:
- VM is hard powered-off (like pulling power cable)
- Autostart is disabled to prevent automatic recovery
- Node becomes NotReady
- LINSTOR detects node failure and promotes replica
- Pod reschedules on surviving node
- Volume mounts successfully on new node

**Actual Behavior**:
Pod migration succeeds and storage remounts on surviving node during the test.

**Post-Test Impact**:
This impact is **inconsistent** across test runs, same as Test 2 (Graceful Shutdown). Sometimes the storage pool recovers normally after node restart, other times it remains in an error state preventing new PVC provisioning.

When the error occurs:

```
$ kubectl-linstor storage-pool list
╭─────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ StoragePool          ┊ Node     ┊ Driver   ┊ PoolName     ┊ FreeCapacity ┊ TotalCapacity ┊ State ┊
╞═════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ vg1-thin             ┊ master-0 ┊ LVM_THIN ┊ vg1/vg1-thin ┊        0 KiB ┊         0 KiB ┊ Error ┊
┊ vg1-thin             ┊ master-1 ┊ LVM_THIN ┊ vg1/vg1-thin ┊    59.88 GiB ┊     59.88 GiB ┊ Ok    ┊
╰─────────────────────────────────────────────────────────────────────────────────────────────────╯

ERROR:
Description:
    Node: 'master-0', storage pool: 'vg1-thin' - Failed to query free space from storage pool
```

This prevents new PVC provisioning due to insufficient available nodes for `autoPlace: 2`.

When storage recovers successfully (inconsistent behavior):

```
$ kubectl-linstor storage-pool list
╭─────────────────────────────────────────────────────────────────────────────────────────────────╮
┊ StoragePool  ┊ Node     ┊ Driver   ┊ PoolName     ┊ FreeCapacity ┊ TotalCapacity ┊ State ┊
╞═════════════════════════════════════════════════════════════════════════════════════════════════╡
┊ vg1-thin     ┊ master-0 ┊ LVM_THIN ┊ vg1/vg1-thin ┊    59.87 GiB ┊     59.88 GiB ┊ Ok    ┊
┊ vg1-thin     ┊ master-1 ┊ LVM_THIN ┊ vg1/vg1-thin ┊    59.87 GiB ┊     59.88 GiB ┊ Ok    ┊
╰─────────────────────────────────────────────────────────────────────────────────────────────────╯
```

**Notes**:
- The inconsistent behavior suggests a race condition in LINSTOR's node recovery process
- Similar timing issues observed in Test 2 may affect this scenario
- When recovery fails, manual intervention is required

**Workaround**:
- Restore the VM and wait for node to rejoin cluster
- Allow time for LINSTOR to detect node recovery and resync storage pools

---

## RWX Mode: Multi-Writer Tests

> ⚠️ **WARNING**: RWX mode is an UNTESTED PROTOTYPE. This configuration has not been validated.

### Test 4: Concurrent Writes

**Status**: ⏳ NOT TESTED

**Description**: Two pods on different nodes writing concurrently via NFS.

**Expected Behavior**:
- NFS server runs on one node with LINSTOR-backed storage
- Two application pods spread across different nodes
- Both pods write to shared NFS mount concurrently
- File shows entries from both pods interleaved
- No corruption or lock issues

**Result**:
```
Not yet tested
```
