# Dataspace cluster setup on AWS: issues identified and solutions applied

## Initial problem

When pods were scheduled on the worker node, they could not reach pods/services running on the master node.

This caused:
- timeouts
- DNS resolution issues

When everything ran on the master node, it worked fine.

---

## Likely root cause (OpenStack networking)

OpenStack may have **anti-spoofing / port security** enabled on the VMs’ network ports.

When a pod sends traffic using its **pod IP** (e.g., `192.168.x.x`), OpenStack drops it because that IP does not match the node IP (e.g., `100.100.0.x`).

---

## Proper long-term fix (infra)

Add **allowed address pairs** to the network ports of both nodes to allow traffic from the pod CIDR (`192.168.0.0/16`).

### Reference commands

```bash
# Identify VM port IDs
openstack port list --server deploytour-master
openstack port list --server deploytour-worker

# Add allowed address pairs to each port
openstack port set --allowed-address ip-address=192.168.0.0/16 <port-id-master>
openstack port set --allowed-address ip-address=192.168.0.0/16 <port-id-worker>
```

---

## Applied workaround (working solution)

Calico was switched to **VXLAN** with:

- `vxlanMode: Always`

This encapsulates pod traffic over VXLAN (UDP 4789), bypassing OpenStack filtering.

**Outcome:** all pod traffic goes through VXLAN and connectivity is restored.

**Operational note:** it may be necessary to restart previously affected pods so they recover cleanly after the change.

---

## “db-init.sh” problem

In AWS, the ecosystem did not recover properly after partial restarts:

- `*-db-init` Jobs failing with:
    - `BackoffLimitExceeded`
    - `job ... is not in complete state`
- Authority / Consumer / Provider pods went into `CrashLoopBackOff`
- Terraform kept failing repeatedly even though the cluster looked “almost” up

Locally it worked by deleting the whole cluster, but in AWS that is not viable.

---

## Root cause (db-init Jobs)

The jobs were using:

- Image: `postgres:15.3-alpine`
- Script: expected `bash`, but Alpine does not include bash by default:
    - `/usr/bin/env bash` does not exist
    - container exited with an error

---

## Solution

Make the script Alpine-compatible (`/bin/sh`, POSIX-friendly).

Example adjustment (conceptual):

```diff
- #!/usr/bin/env bash
- set -euo pipefail
+ #!/bin/sh
+ set -eu
```

---

## Image version management

In Kubernetes / Terraform, images were using `:latest`.

In GHCR, not all images had the `latest` tag, so some pods failed with:

- `ImagePullBackOff`
- `not found: ghcr.io/...:latest`

### Solution

- Pin a stable working version (e.g., `0.1`)
- Avoid relying on `latest` in AWS

---

## OAuth2 (kafka-proxy-provider-oauth2)

The `kafka-proxy-provider-oauth2` deployment shows as `Running`.
However, it has been temporarily omitted (not functionally active).
It does not interfere with the current flow.

---

## `restart.sh` utility

A `restart.sh` script was added as a utility to clean the `default` namespace and start from scratch:

- Uses GHCR
- Respects pinned versions
- Recreates secrets if needed
- Runs Terraform
- Waits for critical jobs

This script does not create custom pods manually.
Everything is created by Kubernetes via Terraform / Helm.
