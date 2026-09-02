---
name: Apply a declarative node configuration with Nerve DNA
description: >-
  Read a node's current configuration, apply a target configuration, watch it converge, and cancel or
  re-apply it. Nerve DNA is the declarative, GitOps-shaped half of the Nerve API and the safest way to
  change a node.
api: openapi/tttech-nerve-management-system-openapi.yml
operations:
  - dna_current_configuration
  - dna_target_configuration
  - apply_dna_configuration
  - dna_configuration_status
  - cancel_dna_configuration
  - re_apply_dna_configuration
  - apply_service_os_dna_configuration
  - cancel_service_os_dna_configuration
generated: '2026-09-01'
method: generated
source: openapi/tttech-nerve-management-system-openapi.yml + https://docs.nerve.cloud/developer_guide/dna/
---

# Apply a node configuration with Nerve DNA

Nerve DNA is a declarative target/current model: you write a target configuration for a node and the node
converges to it. It is the closest thing Nerve has to a transactional change, and it is the mechanism to
prefer over imperative calls whenever both are available.

All operations take `{serialNumber}` — **uppercase, 12 characters**.

## 1. Read the current state first

`dna_current_configuration` (`GET /nerve/dna/{serialNumber}/current`) returns what the node is running
now. Capture it. This is your rollback artifact — DNA has no undo operation, and applying the previously
captured target is how a rollback is expressed.

`dna_target_configuration` (`GET /nerve/dna/{serialNumber}/target`) returns what has been asked for. If
current and target differ, a change is already in flight; do not stack another one on top.

## 2. Know what a DNA file controls

From Nerve 3.1.0 the node DNA covers network interfaces (DHCP or static), proxy settings, timezone,
session controls (`sessionTimeout`, `maxSessionsPerUser`, `maxSSHConnections`), brute-force policy
(`maxFailedLoginAttemptsBeforeLockout`, `loginAttemptWindowMinutes`, `lockoutDurationMinutes`),
credential policy and the workload set. A wrong network block can take the node off the network.

The docs also require that **DNA files contain no credentials** — that is in TTTech's own security
recommendations checklist.

## 3. Apply

`apply_dna_configuration` (`PUT /nerve/dna/{serialNumber}/target`) sets the target. From 3.1.0 the
Management System signs workload DNA files and the node verifies the signature and the workload hashes
before applying them.

## 4. Watch it converge

`dna_configuration_status` (`GET /nerve/dna/{serialNumber}/status`) reports progress. Poll it; do not
assume the `PUT` means done.

## 5. Reversal — and its limits

- `cancel_dna_configuration` (`PATCH /nerve/dna/{serialNumber}/target/cancel`) aborts a configuration
  **that is in progress**. That is a state condition, not a time window: once the configuration is
  applied, cancel is no longer the reversal path.
- `re_apply_dna_configuration` (`PUT .../target/re-apply`) re-converges to the existing target. Use it
  when a node has drifted, or after a node update — a known 3.1.0 issue was DNA apply/re-apply failing
  after an update, fixed in 3.1.1.
- To roll back, apply the configuration you captured in step 1.

Known limitation worth planning around: if workloads were modified locally on the node, a DNA re-apply
may report `applied` without replacing them. TTTech's stated workaround is to remove the locally
modified workload and redeploy it.

## Service OS DNA

`apply_service_os_dna_configuration`, `dna_service_os_target_configuration`,
`dna_service_os_current_configuration`, `service_os_dna_configuration_status`,
`cancel_service_os_dna_configuration` and `re_apply_service_os_dna_configuration` mirror this whole flow
for the node's Service OS. Same discipline applies.
