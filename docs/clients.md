# Client configuration

Each client gets an independent Helm release in its own namespace. PostgreSQL,
application storage and Vault access are separate; the chart and ingress are shared.

| Client | Namespace / Vault role | Database / database user | Vault location |
| --- | --- | --- | --- |
| client-a | odoo-client-a | client_a | ODOO/CLIENT-A |
| client-b | odoo-client-b | client_b | ODOO/CLIENT-B |
| client-c | odoo-client-c | client_c | ODOO/CLIENT-C |

All releases are named `odoo`. Example hostnames follow `client-a.odoo.example.com`;
replace them with real domains before installation. Use the [README Helm commands](../README.md#install-a-client-with-helm).

## Values and overrides

```text
Chart defaults -> Common -> Client -> Optional proxy -> Local overrides
```

Helm applies the files in that order; later values override earlier ones.

- `clients/common.values.yaml`: Vault connection, shared ingress, HPA, Longhorn and quotas.
- `clients/<client>/values.yaml`: database/user, hostname, Vault role and paths; optional sizing overrides.
- `clients/<client>/traefik.values.yaml`: enables a dedicated proxy instead of shared ingress.
- An ignored `cluster.local.yaml`: environment-specific settings, repeated on every install/upgrade.

For shared ingress, match `sharedIngress.controller.namespace` and `podLabels` to
Traefik. Common values assume namespace `traefik`; packaged k3s Traefik normally uses
`kube-system`. Configure its HTTPS entrypoint `websecure`, redirects, timeouts and
access to client ports 8069/8072. The chart does not install that controller.

For the dedicated proxy, add the client's `traefik.values.yaml` after its normal values
on install and upgrade. Set its actual hostname, DNS destination and Vault certificate.
Both proxy modes cannot be enabled together.

## Autoscaling and capacity

HPA defaults are minimum 1, maximum 4 replicas and a 70% CPU target. To change a client,
add this under its existing `odoo` block, retaining database and user settings:

```yaml
odoo:
  database: client_a
  databaseUser: client_a
  autoscaling:
    minReplicas: 1
    maxReplicas: 3
    targetCPUUtilizationPercentage: 70
```

Apply the normal Helm upgrade. HPA then manages replica counts automatically;
`odoo.replicas` is used only when autoscaling is disabled.

The client profiles use Longhorn RWX storage for sessions and attachments. PostgreSQL
uses a separate Longhorn RWO volume for its database files. Client connection
pools are 4 per process, including a separate gevent pool of 4. The chart checks the
maximum pool budget against PostgreSQL's 100 connections, reserving 20 for operations.
Review pools, database resources and quotas together before raising the replica limit.

| Odoo replicas | CPU requests per client | Memory requests per client | Logical storage |
| --- | --- | --- | --- |
| 1 | 0.75 CPU | 1.25 GiB | 30 GiB |
| 4 | 2.25 CPU | 4.25 GiB | 30 GiB |

These totals include PostgreSQL but exclude the optional proxy, Longhorn replication,
backups and shared platform services. Quotas cap consumption; they do not reserve nodes.
Verify actual scaling and storage behavior using [the cluster checks](verification.md).

## Add another client

Copy an existing client directory, change the database/user, hostname and Vault
role/paths, and update the optional proxy hostname. Provision its Vault binding,
secrets, DNS and certificate. Run Helm lint/render, then install into a new namespace
with that client's values. Adding a directory alone does not deploy it.

## Existing data and upgrades

Fresh installs need no migration. If importing an existing RWO application volume,
stop all Odoo writers during maintenance, back up the database and filestore, and
copy the complete Odoo data directory to a new RWX claim with matching permissions.
Select it through `odoo.storage.existingClaim` and verify before reopening traffic.
The chart does not copy files; keep the old claim until recovery is confirmed.

HPA must be suspended while stopping writers; disabling it alone does not stop Odoo.
The chart does not provide a scale-to-zero maintenance mode. Existing PostgreSQL
volumes do not rerun initialization SQL. Major Odoo version changes require database
migration, and an existing web login is never reset from the initial Vault credentials.
