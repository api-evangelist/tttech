---
name: Snapshot, back up and restore a Nerve VM workload
description: >-
  Take a snapshot of a virtual-machine workload on a Nerve node, schedule snapshots, create and list
  backups, and restore from either. This is the reversal surface of the Nerve Node API — the operations
  you set up BEFORE you make a change you might need to take back.
api: openapi/tttech-nerve-node-openapi.yml
operations:
  - get_system_status
  - get_vm_snapshots
  - create_vm_snapshot
  - revert_vm_snapshot
  - delete_vm_snapshot
  - create_vm_schedule_snapshot_config
  - delete_vm_schedule_snapshot_config
  - get_vm_backups_history
  - create_vm_backup
  - restart_vm_backup_creation
  - deploy_vm_backups
generated: '2026-09-01'
method: generated
source: openapi/tttech-nerve-node-openapi.yml + https://docs.nerve.cloud/user_guide/management_system/workload-control/
---

# Snapshot, back up and restore a VM workload

These operations live on the **Nerve Node API**, served by the Local UI on the device itself — not on the
Management System. The published address is `http://172.20.2.1:3333` on a Nerve Device's host-access
interface, or `<wanip>:3333` on devices without a dedicated host-access port. The harvested
specification declares no `servers[]` at all; the address comes from the device guide.

Authenticate with `login` (`POST /api/auth/login`); the Node API's global security requirement is the
`cookie` header, with HTTP basic accepted on some operations.

## 1. Know the node state

`get_system_status` (`GET /api/system/status`) before you touch anything.

## 2. Snapshot before a risky change

`create_vm_snapshot` (`POST /api/workloads/{deviceId}/snapshots`) captures the VM. `get_vm_snapshots`
lists what exists.

`create_vm_schedule_snapshot_config` (`POST .../snapshots/schedule`) sets a recurring schedule;
`delete_vm_schedule_snapshot_config` removes it.

## 3. Restore

`revert_vm_snapshot` (`PUT /api/workloads/{deviceId}/snapshots`) restores the VM to a snapshot. This is
the reversal path.

**TTTech states no retention window for snapshots.** Retention is whatever the operator configured, so
do not promise a caller that a snapshot from N days ago will still be there — check `get_vm_snapshots`
and act on what is actually listed. Do not assume a window that the documentation does not state.

`delete_vm_snapshot` (`DELETE .../snapshots`) removes one, and there is no undelete.

## 4. Backups

`create_vm_backup` (`POST /api/workloads/{deviceId}/backups`) writes a backup;
`get_vm_backups_history` lists them; `restart_vm_backup_creation` retries a failed run;
`deploy_vm_backups` (`POST /api/workloads/backups/deploy`) redeploys a backup from the local repository.

VM backups need an NFS repository — the operator sets up an NFS v3 or v4 server and defines it in the
Local UI first. Backups therefore live on customer infrastructure, and their retention is entirely the
customer's.

## 5. What cannot be taken back

Nothing in this contract reverses `node_offboarding` (`POST /api/system/offboard`) or the secure factory
reset introduced in Nerve 3.1.0, which removes customer data and destroys encryption keys. Both are
terminal by design. Confirm out of band before invoking either.

## Errors and retries

Node API errors are `{status, msg}` JSON, or `{status, message, errors[]}` for validation failures. There
is no idempotency key and no `Retry-After` header on the single 429 (brute-force login blocking). On a
timeout, re-read with `get_vm_snapshots` or `get_vm_backups_history` before resending.
