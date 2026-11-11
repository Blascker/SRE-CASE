# Three clients, one reusable chart

The repository defines three independent Odoo 20 installations. Each owns an Odoo Deployment,
a PostgreSQL StatefulSet, two retained PVCs, credentials, TLS Secret, NetworkPolicies,
a per-client HPA and a ResourceQuota. Its Odoo replicas share one Longhorn RWX
volume for sessions and attachments. PostgreSQL is never shared between clients. Only the existing
ingress controller, Vault/VSO and cluster infrastructure are shared.

The current project has no deployed clients or existing data. First installation
creates new RWX application volumes and new RWO database volumes; no migration is
required. Follow the [README installation commands](../README.md#install-a-client-with-helm),
then [verify the deployment on k3s](verification.md).

| Client | Helm release | Kubernetes namespace | Database / role | Vault role | KV path | Example hostname |
| --- | --- | --- | --- | --- | --- | --- |
| client-a | odoo | odoo-client-a | client_a | odoo-client-a | ODOO/CLIENT-A | client-a.odoo.example.com |
| client-b | odoo | odoo-client-b | client_b | odoo-client-b | ODOO/CLIENT-B | client-b.odoo.example.com |
| client-c | odoo | odoo-client-c | client_c | odoo-client-c | ODOO/CLIENT-C | client-c.odoo.example.com |

Release names and Secret names can repeat because namespaces differ. Do not install
two clients in the same namespace. The same-namespace selectors keep each Odoo
connected only to its own PostgreSQL. A namespace is not a complete security boundary:
use cluster RBAC to restrict access and a CNI that enforces NetworkPolicy.

## Configuration and local verification

Use Helm directly. The [README](../README.md#install-a-client-with-helm) is the
installation runbook, with commands for client-a, client-b and client-c. Deployment
also uses kubectl to inspect initialization and application status.

Values merge in this order: chart defaults, `clients/common.values.yaml`, the
client's `values.yaml`, its optional `traefik.values.yaml`, then local overrides.
Shared settings belong in the common file; database names, domains, Vault roles/paths
and client-specific sizing or image versions belong in the client files. Never put
passwords or private keys in them. Each client has two files; there is no temporary
initialization override.

```sh
helm lint --strict charts/odoo --namespace odoo-client-a \
  -f clients/common.values.yaml -f clients/client-a/values.yaml
helm template odoo charts/odoo --namespace odoo-client-a \
  -f clients/common.values.yaml -f clients/client-a/values.yaml
```

Repeat with the other client names and namespaces. These commands do not access
Kubernetes. Local rendering does not verify network enforcement or Vault policies.

## Shared ingress prerequisites

The fleet sets `sharedIngress.enabled: true` and `traefik.enabled: false`.
Install/manage one shared Traefik ingress controller separately, with:

- IngressClass `traefik`, the Kubernetes Ingress provider and access to all client namespaces.
- Namespace `traefik` and controller pod label `app.kubernetes.io/name: traefik`, or
  update `sharedIngress.controller` to match the actual installation. Both the
  namespace and pod labels must match the same NetworkPolicy peer.
- HTTPS entrypoint `websecure`, HTTP-to-HTTPS redirect configured at the controller,
  suitable request timeouts, and a shared load balancer/DNS destination.
- Network access to client pods on TCP 8069 and 8072 if the controller has egress restrictions.

Each client Ingress routes `/websocket` to 8072 and `/` to 8069. VSO synchronizes
that client's TLS certificate from Vault into its own namespace; the controller
watches Kubernetes Secrets for changes. There is no restart of a shared controller
when one client's certificate changes. Certificate issuance/renewal remains external.
The obsolete `ingress.*` key is still rejected; use `sharedIngress.*`.

The shared controller uses Kubernetes Service references and endpoint discovery;
the dedicated proxy mode still uses explicit internal FQDN URLs. Odoo-to-PostgreSQL
uses an internal FQDN in both modes. See the [Traefik Ingress reference](https://doc.traefik.io/traefik/reference/routing-configuration/kubernetes/ingress/).

## Prepare and initialize each client

Follow the [README Helm commands](../README.md#install-a-client-with-helm).
They are instructions for deployment, not evidence of a running installation.
Use an explicitly selected context, valid domains, suitable StorageClasses and
verified Vault CA trust. The controller and VSO must already be installed.

1. Configure the three Vault roles/policies described in [vault.md](vault.md#client-fleet-bindings).
   Generate distinct passwords and supply each client's matching certificate/key.
   Pre-create each namespace if needed to supply its Vault CA Secret before install.
2. Replace the example hostname in each client's values. Put environment-specific
   settings in an ignored `cluster.local.yaml`; use an additional client-specific
   override where needed. Explicit `-f` arguments must be repeated on every command.
3. Run `helm install` with common values, client values, any optional dedicated-proxy
   file and local overrides. PostgreSQL provisions an empty database and restricted
   owner on a fresh volume. Odoo's `initialize-database` init container installs
   `base` and `web` without demo data, then sets the first web administrator from
   Vault keys `odoo-admin-login` and `odoo-admin-password`.
4. Wait for readiness, then verify HTTPS login, WebSockets and attachments. Subsequent
   changes use `helm upgrade --reset-values` with the same normal values files.

The init container skips initialized databases from the same Odoo major series and
never resets existing users or runs module upgrades. It blocks mismatched versions;
an Odoo 19 database must be migrated and validated before deployment with Odoo 20. A durable database marker records initial credential setup.
Incomplete module initialization blocks startup and requires manual recovery;
interrupted credential finalization can resume on the next attempt. PostgreSQL
initialization SQL runs only on an empty volume, so existing database/role provisioning
and legacy privilege revocation require an administrator. Supply the two initial
web credential keys before upgrading existing installations, even when they will
be skipped. Initial credential changes in Vault do not rotate an existing web login.

Helm does not create Vault policies, issue certificates, install the shared controller
or implement backups. Configuration rollback does not reverse database migrations.
Retained PVCs do not protect against namespace deletion.

If an installation already exists, keep its original release, namespace, database
and PVC identity using direct Helm commands until an explicit migration is planned.
The new fleet names do not adopt or migrate an existing `challenge` release.

## Automatic Odoo scaling

Each client inherits an enabled CPU HPA from clients/common.values.yaml. The default
range is one to four Odoo pods, targeting 70% average CPU utilization. A working k3s
Metrics Server is required. Tune minReplicas, maxReplicas and the CPU target in that
client's existing odoo.autoscaling block, then apply its normal Helm upgrade; HPA
handles subsequent scaling. See [the README commands](../README.md#automatic-scaling-per-client).

Shared sessions and attachments require the client's Longhorn RWX volume. Install
NFSv4 client support on eligible nodes and verify the longhorn StorageClass and volume
permissions. Fresh installations create the RWX claim automatically. Only an already
deployed RWO application claim requires the migration described below. PostgreSQL keeps its
existing dedicated single-instance storage; this work adds no database replication.

Client defaults use dbMaxconn=4 and dbMaxconnGevent=4. The chart validates their
conservative maximum process-pool budget against PostgreSQL maxConnections, including
20 connections reserved for operations. Four replicas fit the default budget of 100;
raising the maximum requires reviewing pools, database resources and namespace quotas.
At four replicas, requests per client are 2.25 CPU and 4.25 GiB RAM, with the same
30 GiB of logical PVC capacity. Optional proxy, Longhorn replication and platform
resources are additional. Quotas bound consumption rather than reserve it.

Longhorn share-manager/storage failures, the single PostgreSQL instance and Recreate
rollouts can interrupt service across replicas. This is automatic application scaling,
not end-to-end HA. Validate actual CPU-driven scale-up/down and cross-node RWX behavior
on the target k3s cluster before treating the configuration as production-ready.

Local container verification confirmed concurrent initialization, a login session
created on one replica being accepted on the other, attachment upload on one and
download on the other, and a WebSocket upgrade using the shared session. This used
a shared local volume and does not verify Longhorn NFS behavior or the Kubernetes HPA.
See [Longhorn RWX requirements](https://longhorn.io/docs/latest/nodes-and-volumes/volumes/rwx-volumes/)
and [Kubernetes autoscaling](https://kubernetes.io/docs/tasks/run-application/horizontal-pod-autoscale/).

For exact commands, expected results and troubleshooting, use the
[k3s verification runbook](verification.md). Verify all three clients independently;
the HPA in one namespace does not scale another client's Odoo.

## Only for existing RWO application volumes

This is not needed for the project's fresh installations. Use it only when bringing
an existing application PVC with data into the new RWX configuration.

1. Schedule maintenance and stop incoming traffic and every Odoo writer, including
   cron workers. Suspend autoscaling through the release configuration while stopping
   the application; an enabled HPA would otherwise restore its minimum replicas.
2. Take a coordinated database/filestore backup. Provision a new Longhorn RWX PVC
   in the same namespace. Do not attempt to patch an RWO claim's access mode in place.
3. Copy the complete `/var/lib/odoo` data directory, including filestore and sessions,
   preserving UID/GID and permissions. Keep the original claim until recovery is verified.
4. Set `odoo.storage.existingClaim` to the new claim and keep
   `odoo.storage.accessMode: ReadWriteMany`. Restore the release's autoscaling settings
   and upgrade with the same client values and local overrides.
5. Verify login, attachments and cross-replica access before reopening traffic.

Stopping an existing release for a data copy requires a separate maintenance procedure;
the chart deliberately requires a positive fixed replica count and does not provide a
scale-to-zero maintenance mode. Disabling HPA alone does not stop Odoo. The chart
references the supplied claim but never copies data or removes the old retained PVC.

## Add the fourth or hundredth client

Add `clients/<client-id>/values.yaml` with unique database/role names, hostname,
Vault role and KV paths, following the three existing clients. Copy its optional Traefik
override too, updating comments and proxy hostname for the new client. Provision the corresponding
Vault binding and secrets, run validation, then deploy that client; its init container handles database setup.
Use the same chart with the new values path and namespace in the Helm commands.
Adding a directory does not deploy it automatically. Validate hostname and Vault
path uniqueness during review; Helm lint cannot prove the Vault policies or DNS
records are correct.

Common-value or chart edits affect every client at their next deployment. Validate
the whole fleet, then deploy to a test client and small batches with health checks
between batches. Pin images by digest and promote reviewed Git revisions. For 100
clients, a GitOps controller can consume this same directory layout; Argo CD is not
installed or configured by this milestone.

At the default HPA minimum of one Odoo replica, application/database requests total 0.75 CPU, 1.25 GiB RAM and 30 GiB of
storage per client: three clients request 2.25 CPU, 3.75 GiB RAM and 90 GiB storage;
100 request 75 CPU, 125 GiB RAM and 3,000 GiB storage. These exclude shared services,
backup capacity, storage replication and operating headroom, and are not load-test
results. Limits allow higher usage. ResourceQuota caps each namespace (including
temporary jobs) and does not reserve node capacity. Revisit quotas when increasing
client sizing. See [Kubernetes quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/).

Before production or growth toward 100: implement coordinated database/filestore
backups and restore drills, per-client monitoring, upgrade/migration automation and
capacity measurements. Odoo automatically scales from one to four replicas; PostgreSQL remains a single
instance and Recreate application rollouts still cause downtime. A future PostgreSQL operator may add replication and
backup automation while preserving a dedicated PostgreSQL cluster for every client.
Distribute clients across bounded Kubernetes clusters as workload and recovery needs
require; client count alone does not determine the number of clusters.
