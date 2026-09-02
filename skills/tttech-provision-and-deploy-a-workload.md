---
name: Provision a Nerve workload and deploy it to nodes
description: >-
  Create a workload in the Nerve Management System, add a version and its files, find the nodes eligible
  for that version, and deploy it. Covers the docker-compose pre-flight validator, which is the only
  rehearsal step this API offers.
api: openapi/tttech-nerve-management-system-openapi.yml
operations:
  - v3_process_docker_compose_file
  - create_workload_v3
  - create_version_v3
  - add_file_v3
  - define_all_files_v3
  - get_files_v3
  - list_versions_v3
  - get_nodes_for_deploy
  - managing_deployed_workload
  - list_deployed_workloads
generated: '2026-09-01'
method: generated
source: openapi/tttech-nerve-management-system-openapi.yml + https://docs.nerve.cloud/user_guide/management_system/
---

# Provision a workload and deploy it

Authenticate first — see the authenticate-and-inventory skill. Every call below carries the `sessionId`
header.

## 0. Validate before you create (docker-compose only)

`v3_process_docker_compose_file` (`POST /nerve/v3/workloads/compose`) uploads and **validates** a
docker-compose file without creating anything. Use it as a pre-flight check; this API has no `dry_run`
parameter, and this validator plus `validate_remote_connection_file` are the only two rehearsal
affordances it exposes. (The first-party CLI adds a global `--dry-run` flag — see `cli/tttech-cli.yml`.)

## 1. Create the workload

`create_workload_v3` (`POST /nerve/v3/workloads`) creates the workload container object. Note that v2 and
v3 workload endpoints are both live in the 3.1.0 contract; **use v3** for new integrations. `WORKLOAD_v2`
operations exist for compatibility with older Management Systems.

## 2. Add a version

`create_version_v3` (`POST /nerve/v3/workloads/{workloadId}/versions`) creates the deployable version.
A workload with no version cannot be deployed.

## 3. Attach the files

`add_file_v3` (`POST .../versions/{versionId}/files`) uploads each file. Then call `define_all_files_v3`
(`POST .../define-all-files`) to declare the version's file set complete, and `get_files_v3` to confirm
what landed and what state each file's download is in.

File uploads are **cancellable while in progress**: `cancel_file_v3` cancels one,
`cancel_all_files_v3` cancels every in-flight download. Once complete, the only reversal is
`delete_file_v3`.

## 4. Find deployable nodes

`get_nodes_for_deploy` (`GET /nerve/nodes/deploy/{workloadId}/{versionId}`) returns the nodes on which
that specific version can be deployed. Do not deploy against your own filtered node list — compatibility
between Management System and node versions is real and matters (registry-backed workloads, for
instance, require node 2.10 or later).

## 5. Deploy and control

`managing_deployed_workload` (`POST /nerve/workload/controller`) is the single lifecycle endpoint. The
body requires `deviceId`, `serialNumber` (12 uppercase alphanumerics), `sessionToken` and `command`, one
of `START`, `STOP`, `SUSPEND`, `RESUME`, `RESTART`, `UNDEPLOY`.

**This changes what is running on a machine on a plant floor.** Treat it as a safety-critical action:

- Confirm the node and `deviceId` with `list_deployed_workloads` immediately before you act.
- Every command has an inverse in the same enum except `UNDEPLOY`.
- `UNDEPLOY` accepts `removeImages`. With `removeImages: false` the images stay on the node, so a
  redeployment is fast. With `true` the images are deleted and redeploying means a full transfer. Choose
  deliberately; the default is not a safe assumption.
- There is **no idempotency key**. If a call times out, re-read state with `list_deployed_workloads`
  before resending — do not blind-retry a `RESTART` or an `UNDEPLOY`.

## 6. Verify

`list_deployed_workloads` (`GET /nerve/workload/node/{serialNumber}/devices`) lists what is actually on
the node. Verify after every control command; the deployment is asynchronous and the API returns 202 on
several paths.
