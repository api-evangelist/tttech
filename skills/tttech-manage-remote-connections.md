---
name: Open, audit and terminate Nerve remote connections
description: >-
  Validate a remote-connection definition, open a tunnel to a node, workload or CODESYS runtime, list
  what is currently open, and terminate it. Remote connections are the highest-privilege surface in
  Nerve — this skill treats them accordingly.
api: openapi/tttech-nerve-management-system-openapi.yml
operations:
  - validate_remote_connection_file
  - connect_remote_connections
  - get_list_of_active_remote_connections
  - terminate_remote_connections
  - cancel_rc_approval
  - export_rc_on_node
  - import_rc_on_node
generated: '2026-09-01'
method: generated
source: openapi/tttech-nerve-management-system-openapi.yml + https://docs.nerve.cloud/user_guide/management_system/remote/
---

# Manage remote connections

A Nerve remote connection is an interactive tunnel to a node, a Docker or compose workload, a VM or a
CODESYS runtime. The specification's named schemas — `rc_node`, `rc_docker`, `rc_compose`, `rc_vm`,
`rc_codesys` — are the five kinds.

## 1. Validate the definition before you import it

`validate_remote_connection_file` (`POST /nerve/v2/remote-connections-file/{target}/validate`) validates
a remote-connection YAML file against a node or a workload without applying it. Along with the
docker-compose validator this is one of only two pre-flight checks in the whole API. Use it.

## 2. Move definitions between nodes

`export_rc_on_node` (`GET /nerve/v2/node/{serial_number}/export-remote-connections`) and
`import_rc_on_node` (`PUT .../import-remote-connections`) move remote-connection sets between nodes.
Export before you import — the import is the write, and there is no undo for it beyond re-importing the
exported file.

## 3. Open a connection

`connect_remote_connections` (`POST /nerve/remote-connections/connect`) opens the tunnel. From Nerve
2.9.0 an operator can require approval **on the node** before a connection is established; if that is on,
your request sits pending until someone approves it. Do not treat a pending request as a failure and
retry it — you will queue duplicates.

## 4. Audit what is open

`get_list_of_active_remote_connections` (`GET /nerve/active-remote-connections`) is the live inventory.
An agent operating this API should list before and after every connect.

## 5. Close it

- `terminate_remote_connections` (`POST /nerve/active-remote-connections/terminate-connections`) ends
  active sessions. This is the reversal path and it works at any time — no window.
- `cancel_rc_approval` (`DELETE /nerve/active-remote-connections/cancel-connection-approval/{connectionRequestUid}`)
  withdraws a request that is still waiting for approval.

## Operating notes

- Concurrent remote connections per user became operator-configurable in Nerve 3.1.0; a request can be
  refused because of a limit rather than a fault.
- A fixed 2.9.1 issue had orphaned remote connections tripping brute-force protection. If you are seeing
  429s from an automation that opens tunnels, look for connections you never terminated.
- Always terminate what you open. An abandoned tunnel into an industrial node is a security finding, and
  TTTech's own IEC 62443 mapping cites remote session termination (CR2.6) as a control this API provides.
