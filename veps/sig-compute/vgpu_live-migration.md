# VEP #109: Implement vGPU Enabled Live Migration

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [X] (R) Enhancement issue created, which links to VEP dir in [kubevirt/enhancements] (not the initial VEP PR)
- [] (R) Target version is explicitly mentioned and approved
- [X] (R) Graduation criteria filled

## Overview

This is a proposal to allow live migrations in KubeVirt to work for VMs with a single NVIDIA vGPU, exposed by mdev, between two nodes in the same cluster with identical GPUs, matching GPU ECC configuration, and GPU drivers that satisfy NVIDIA’s requirements.

## Motivation

GPU usage is increasing with more and more companies running AI workloads, so companies are now requesting live migration to support GPU enabled VMs.

## Goals

* Address a common live migration problem where the target needs to update the destination Libvirt XML. In the case of mdevs, it needs to update the mdev UUID in the XML.
* Support single vGPU enabled live migrations for both nodes that are using the Nvidia GPU Operator and clusters that are using KubeVirt’s generic device plugin for mdev.
* Support single vGPU enabled live migrations with minimal data lost due to high dirty rates.

## Non Goals

* Do not want to change the live migration workflow for non vGPU enabled VMs.
* Do not support live migration for passthrough or SRIOV vGPU
* Do not support cross-cluster live migrations
* Do not support live migrations for VMs with multiple vGPUs

## Definition of Users

* **KubeVirt Administrators:** Users who have cluster wide privileges to trigger APIs to manage a cluster.
* **KubeVirt Owner:** VM workload owners who want high availability for their VMs.

## User Stories

As a KubeVirt admin/owner, I want to be able to live migrate my VMs that have an NVIDIA vGPU.

## Repos

https://github.com/kubevirt/kubevirt

## Design

### NVIDIA vGPU migration and driver / manager version requirements

NVIDIA documents vGPU live migration support and limitations for Red Hat Enterprise Linux with KVM in [vGPU Migration Support (RHEL with KVM release notes)](https://docs.nvidia.com/vgpu/latest/grid-vgpu-release-notes-red-hat-el-kvm/index.html#vgpu-migration-support). The following summarizes what is **allowed or required for migration at the NVIDIA layer** (independent of KubeVirt’s own scheduling rules):

| Host OS (RHEL + KVM) | NVIDIA Virtual GPU Manager across source and destination |
|----------------------|----------------------------------------------------------|
| **9.4** | Source and destination hosts **must** run the **same** Virtual GPU Manager **version**. Migration is **not** supported between hosts on **different** manager versions, **even within the same manager branch**. |
| **9.6 and later** (unless NVIDIA states otherwise for a specific release) | Migration **is** supported between hosts running **different** Virtual GPU Manager versions. |

Additional NVIDIA migration constraints from the same section:

* **VFIO stack:** On **RHEL KVM 9.4**, migration **to or from** a host that uses a **vendor-specific VFIO framework** is **not** supported. (NVIDIA notes that the 9.4 restrictions above generally **do not** apply from **9.6** onward.)
* **Guest vs host drivers:** The guest VM must still use an NVIDIA guest driver combination that is **compatible** with the Virtual GPU Manager on the hosts, per NVIDIA’s general [vGPU Manager / guest driver compatibility](https://docs.nvidia.com/vgpu/latest/grid-vgpu-release-notes-red-hat-el-kvm/index.html) rules (same branch, cross-branch rules, etc.). Migration documentation above is about **manager version alignment between hypervisor hosts**, not a substitute for guest/host compatibility.
* **CUDA / dev tooling:** vGPU migration is **disabled** on a VM if certain CUDA Toolkit features are enabled in the guest: **unified memory**, **debuggers**, or **profilers**.
* **ECC memory configuration:** Source and destination hosts must use the **same ECC memory setting** on the physical GPU (enabled vs disabled) used for the vGPU. NVIDIA documents that migration between hosts with **different** ECC configurations can **stop before completion**; treat matching ECC as a **requirement** for supported migration, not optional tuning.
* **Known issues (NVIDIA):** Migration **fails** for GPUs that include a **GPU System Processor (GSP)** (see NVIDIA known-issues list for current status).

For **KubeVirt Alpha**, we still require **identical** NVIDIA Virtual GPU Manager (host), **matching GPU ECC settings** across nodes used for vGPU migration, and aligned guest drivers on all worker nodes participating in vGPU migration. That satisfies NVIDIA’s strictest case (including RHEL 9.4) and avoids KubeVirt having to reason about OS minor version and NVIDIA’s per-release exceptions. **Beta** may relax this where NVIDIA allows (e.g. RHEL 9.6+ with different manager versions), by taking driver/manager version into account when scheduling migration targets—see Graduation Requirements.

[VEP 141](https://github.com/kubevirt/enhancements/issues/141) introduces a feature gate in KubeVirt, TargetSideMigrationHooks, to register and write QEMU hooks for the target `virt-launcher`. We will use this new infrastructure to mutate the domain XML with the updated mdev UUID, which will be the one assigned to the target `virt-launcher` by `gpu.CreateHostDevices()` in `manager.go`. VGPU live migration will only be available with the TargetSideMigrationHooks feature gate enabled. 

Once the destination XML contains the correct fields, the live migration can begin. Libvirt/QEMU already support vGPU live migration for mdev (since Libvirt 8.6.0 and QEMU 8.1.0) and will do the actual migration, so no further work is needed by KubeVirt to migrate the vGPU. Some migration configs at the Libvirt/QEMU level, such as the migration method or downtime limit, may be necessary however.

### Example 
XML snippet before hook:
```
<hostdev mode='subsystem' type='mdev' managed='no' model='vfio-pci' display='on' ramfb='on'>
      <source>
        <address uuid='bb4a98d8-60c1-40c6-b39b-866b1e82bd8c'/>
      </source>
      <alias name='ua-gpu-gpu1'/>
      <address type='pci' domain='0x0000' bus='0x09' slot='0x00' function='0x0'/>
    </hostdev>
```

XML snippet after hook (address uuid updated):
```
<hostdev mode='subsystem' type='mdev' managed='no' model='vfio-pci' display='on' ramfb='on'>
      <source>
        <address uuid='05b59010-d19c-47d2-9477-33b4579edc90'/>
      </source>
      <alias name='ua-gpu-gpu1'/>
      <address type='pci' domain='0x0000' bus='0x09' slot='0x00' function='0x0'/>
    </hostdev>
```

**Failed migrations:** Cleanup will be performed by existing code and by code introduced in [16212](https://github.com/kubevirt/kubevirt/pull/16212).

## API Examples

N/A

## Alternatives

Instead of relying on a QEMU hook, a Libvirt API could be introduced to allow KubeVirt to update the destination XML at the start of migration via callbacks. However, previous discussions asking for this API haven’t made progress.

## Scalability

The unix socket used will be `/var/run/kubevirt/migration-hook-socket` introduced in PR [16212](https://github.com/kubevirt/kubevirt/pull/16212). A target `virt-launcher` pod will have at most one of this socket open at a time, so it should be possible to live migrate a large number of VMs concurrently without significant performance issues. KubeVirt also imposes its own limitations on the number of live migrations on a node and cluster-wide level.

## Update/Rollback Compatibility

* Needs TargetSideMigrationHooks feature gate from PR [16212](https://github.com/kubevirt/kubevirt/pull/16212) to be enabled
* Will be safe during upgrades as long as the newer node's mdev uuids don't change unexpectedly.

## Functional Testing Approach

* Unit tests: Verify that the VM is able to live migrate with the vGPU given the proper conditions.
* [Optional] Also verify that this works with the NVIDIA GPU Operator.

## Implementation History

N/A

## Graduation Requirements

### Alpha

* Implement basic functionality and testing.
* Limitations
    * Users must ensure all worker nodes use the **same NVIDIA Virtual GPU Manager build/version** on the hypervisor and compatible guest drivers in the VM. This matches **NVIDIA’s requirement for RHEL KVM 9.4** (no cross-host manager version mismatch, even within the same branch) and is a safe default on **9.6+** where NVIDIA allows different manager versions—KubeVirt Alpha does not exploit that allowance yet.
    * Users must still comply with NVIDIA’s other migration rules (e.g. **identical ECC memory configuration** on the GPU between source and destination nodes; no GSP GPUs where migration is broken per NVIDIA; no enabled unified memory / CUDA debuggers / profilers in the guest if those disable migration).
    * KubeVirt is unable to estimate the maximum period for the migration. Use a hard limit that is equal to the existing   calculated values (which ignore gpu info)
* Figure out how to handle any data loss during the migration.

### Beta

* Where the platform meets NVIDIA’s criteria (e.g. **RHEL KVM 9.6+** and NVIDIA’s documented allowances), KubeVirt may schedule migration to targets with **different** Virtual GPU Manager versions than the source; on **RHEL KVM 9.4**-class environments, scheduling must still enforce **identical** manager versions per NVIDIA. KubeVirt should take **hypervisor OS version**, **VFIO implementation**, **NVIDIA manager version**, and **matching GPU ECC configuration** on source and destination into account when choosing migration targets.
* Find a way to estimate the maximum period for the migration 
* Needs [VEP 141](https://github.com/kubevirt/enhancements/issues/141) to be in Beta.

### GA

* Needs [VEP 141](https://github.com/kubevirt/enhancements/issues/141) to be in GA.
