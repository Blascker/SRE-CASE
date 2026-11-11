# Verify the deployment on k3s

Use this runbook after the [fresh Helm installation](../README.md#install-a-client-with-helm).
No migration is needed for a fresh installation. These checks have not been run on
the target k3s cluster; they describe how to verify its actual behavior.

The chart, local two-replica application checks and cluster checks cover different things:

| Check | Already verified locally | Still requires k3s |
| --- | --- | --- |
| Helm rendering, HPA configuration and storage/connection-budget validation | Yes, all three clients | Resource admission and reconciliation |
| Concurrent initialization, sessions/attachments shared by two Odoo replicas, WebSocket access | Yes, with a local shared Docker volume | Longhorn mounts and access from actual nodes |
| CPU-driven automatic scale-up and scale-down | No | Metrics Server, HPA controller and sufficient node capacity |
| Public DNS/TLS, shared ingress and network-policy enforcement | No | Actual cluster networking and certificates |

Run commands from the repository root. Set `TARGET_CONTEXT` to the intended kubeconfig
context and `TARGET_NAMESPACE` to the client being verified. For example:

```sh
export TARGET_CONTEXT='your-k3s-context'
export TARGET_NAMESPACE='odoo-client-a'
```

Replace the context placeholder before running. Repeat for `odoo-client-b` and
`odoo-client-c`. All three use release `odoo`; names below follow that release name.
The read-only inspection commands do not print Secret payloads. The bounded CPU load
and scratch-file checks below actively exercise the selected client; run them before
onboarding real users.

## 1. Check cluster prerequisites

```sh
kubectl --context "${TARGET_CONTEXT:?Set TARGET_CONTEXT first}" get nodes -o wide
kubectl --context "$TARGET_CONTEXT" get storageclass longhorn
kubectl --context "$TARGET_CONTEXT" get apiservice v1beta1.metrics.k8s.io
kubectl --context "$TARGET_CONTEXT" top nodes
kubectl --context "$TARGET_CONTEXT" get ingressclass traefik
```

Expected: eligible nodes are Ready, the `longhorn` StorageClass exists, the metrics
API reports `AVAILABLE=True`, CPU/memory values are numerical, and the configured
IngressClass exists. An RWX cross-node check needs at least two eligible nodes.
Longhorn must have enough healthy replicas and capacity for its configured policy.
NFSv4 client support and the required storage network access must be present on every
node that may run Odoo. A StorageClass name alone does not prove RWX mounts work.

k3s normally packages Metrics Server and Traefik, but either can be disabled or
configured differently. If using packaged Traefik in `kube-system`, set
`sharedIngress.controller.namespace: kube-system` in your local overrides. The common
values currently assume a separately managed controller in namespace `traefik`.
Always match its actual namespace and pod labels before installing clients.

## 2. Check the client installation

```sh
helm status odoo --kube-context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE"
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" get pods -o wide
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" get pvc
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" get vaultstaticsecret
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" get hpa odoo-odoo
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" top pods -l app=odoo
```

Expected: Helm reports deployed, Odoo and PostgreSQL become Ready, both PVCs are Bound,
VSO reports successful synchronization, and HPA has numerical CPU utilization with
minimum 1 and maximum 4 replicas unless the client overrides them. Allow initial
metrics collection time; persistent `<unknown>` is not a passing result.

The application claim `odoo-odoo-data` must show `RWX` and StorageClass `longhorn`.
The separate database claim `odoo-db-data` must show `RWO`; this is intentional.
If using `odoo.storage.existingClaim`, inspect that application's claim instead.
The application remains unavailable until its init container finishes successfully.

## 3. Verify automatic scale-up and scale-down

Keep HPA enabled; do not use `kubectl scale` for this check. In one terminal, watch:

```sh
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  get hpa odoo-odoo --watch
```

In another terminal, list Odoo pods and set `POD_A` to one Ready pod name:

```sh
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  get pods -l app=odoo -o wide
export POD_A='replace-with-a-ready-odoo-pod-name'
```

The following controlled load runs two CPU-bound processes inside that pod for five
minutes, then exits. It does not change the database or application files. It competes
with Odoo for the pod's CPU limit, so use it only during initial verification:

```sh
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  exec -i "${POD_A:?Set POD_A first}" -c odoo -- python3 - <<'PY'
from multiprocessing import Process
import time

def consume_cpu():
    deadline = time.monotonic() + 300
    while time.monotonic() < deadline:
        sum(number * number for number in range(10000))

workers = [Process(target=consume_cpu) for _ in range(2)]
for worker in workers:
    worker.start()
for worker in workers:
    worker.join()
PY
```

Expected: numerical CPU utilization rises above the configured target, HPA requests
more than one replica, and the new pods initialize and become Ready. The default
policy permits only one added pod per minute, up to four. Reaching exactly four is
not guaranteed by this load: observe actual utilization and HPA decisions. A CPU-bound
request workload against Odoo can additionally validate representative user traffic.

While extra replicas are running, perform the shared-storage/session checks below
in other terminals. After the load ends, CPU should drop and HPA should eventually
return toward its minimum. Scale-down stabilizes for five minutes and removes at
most one pod per minute, so allow additional time for metrics and stabilization.

Pass criteria: observed automatic growth above the initial replica count and later
automatic reduction without manually changing replicas. HPA desired replicas alone
are insufficient; the requested pods must actually become Ready.

## 4. Verify shared storage and sessions

While at least two Odoo pods are Ready, choose distinct pod names:

```sh
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  get pods -l app=odoo -o wide
export POD_A='replace-with-first-ready-odoo-pod-name'
export POD_B='replace-with-second-ready-odoo-pod-name'
```

Prefer pods on different nodes. The chart does not force replica placement. If they
are on the same node, this verifies shared access but leaves cross-node RWX behavior
unverified; repeat when two Ready replicas are scheduled on different nodes.

Create the runbook's scratch file from the first pod. Exclusive
creation prevents overwriting an existing file with that name:

```sh
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  exec "${POD_A:?Set POD_A first}" -c odoo -- python3 -c \
  'from pathlib import Path; p=Path("/var/lib/odoo/.odoo-rwx-verification"); f=p.open("x"); f.write("odoo-rwx-ok"); f.close()'
```

Read the same file through the second pod and remove it only after the content matches:

```sh
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  exec "${POD_B:?Set POD_B first}" -c odoo -- python3 -c \
  'from pathlib import Path; p=Path("/var/lib/odoo/.odoo-rwx-verification"); assert p.read_text()=="odoo-rwx-ok"; p.unlink(); print("Shared read/write verified")'
```

Expected: both commands succeed without mount or permission errors. If exclusive
creation reports an existing file, inspect the previous verification attempt before
reusing the name; do not overwrite unknown data.

For session behavior, open two local port-forwards in separate terminals with the
same context/namespace variables and the selected pod names set:

```sh
# Terminal A
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  port-forward "pod/${POD_A:?Set POD_A first}" 18069:8069
```

```sh
# Terminal B
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  port-forward "pod/${POD_B:?Set POD_B first}" 28069:8069
```

Log in at `http://localhost:18069` and open `http://localhost:28069` in the same
browser profile, consistently using `localhost` for both. Cookies are shared across
these ports. Expected: the second replica recognizes the authenticated session.
Use the deployed application's attachment UI to upload a small file through one
replica and download it through the other; confirm the contents match. Stop both
port-forwards after checking. Pod port-forwards isolate application behavior; they
do not validate ingress routing or NetworkPolicy enforcement.

## 5. Verify the actual hostname

Use the client's real configured hostname and certificate:

- HTTP redirects to HTTPS, and HTTPS presents the matching trusted certificate.
- Login and attachment upload/download work through ingress while multiple replicas run.
- Browser developer tools show the `/websocket` request upgrading with status 101.
- When HPA reduces replica count, subsequent browser requests and attachment access
  continue through the remaining Ready endpoints. WebSocket connections may reconnect.

Check HPA, pod status and application logs together. These checks demonstrate scaling;
they do not establish database, storage or ingress high availability.

## If a check fails

```sh
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" describe hpa odoo-odoo
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" describe pvc odoo-odoo-data
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" get events --sort-by=.metadata.creationTimestamp
kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
  logs deployment/odoo-odoo -c initialize-database
```

| Symptom | What to inspect |
| --- | --- |
| HPA CPU stays `<unknown>` | Metrics API availability, Metrics Server errors, pod readiness and CPU requests |
| HPA requests replicas but pods remain Pending | Scheduling capacity, namespace quota, PVC binding and mount events |
| RWX claim remains Pending | StorageClass, Longhorn volume/replica health and available storage |
| Mount or permission errors on one node | NFS client support, storage connectivity, Longhorn share-manager and UID/GID 1000 permissions |
| New pod remains in initialization | VSO synchronization, database reachability, init logs and initialization lock timeout |
| Repeated logout or missing attachments across pods | All pods reference the same application claim/data directory, matching versions and file permissions |
| No scale-down after load ends | Remaining CPU load, HPA stabilization/recommendations and any per-client minimum override |

Fix the underlying condition and repeat the failed check. Do not delete PVCs or
print credential payloads to diagnose these issues. Record the context, client,
date, observed replica changes, pod/node placement and results before marking the
cluster verification complete.
