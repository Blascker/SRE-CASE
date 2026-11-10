# Three clients, one reusable chart

The repository defines three independent installations. Each owns an Odoo Deployment,
a PostgreSQL StatefulSet, two retained PVCs, credentials, TLS Secret, NetworkPolicies
and a ResourceQuota. PostgreSQL is never shared between clients. Only the existing
ingress controller, Vault/VSO and cluster infrastructure are shared.

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
also uses kubectl for local database setup access.

Values merge in this order: `charts/odoo/values.yaml`, `clients/common.values.yaml`,
`clients/<client>/values.yaml`, then explicit `-f` overrides. Bootstrap settings are
applied last during initial installation. Shared settings belong in the common file;
domains, Vault roles/paths and client-specific sizing or image versions belong in
the client file. Never put passwords or private keys in these files.

```sh
helm lint --strict charts/odoo --namespace odoo-client-a \
  -f clients/common.values.yaml -f clients/client-a/values.yaml
helm template odoo charts/odoo --namespace odoo-client-a \
  -f clients/common.values.yaml -f clients/client-a/values.yaml
helm template odoo charts/odoo --namespace odoo-client-a \
  -f clients/common.values.yaml -f clients/client-a/values.yaml \
  -f examples/bootstrap.values.yaml
```

Repeat with the other client names and namespaces. These commands do not access
Kubernetes. Rendered output is for review; install through Helm with the correct
namespace. Local rendering does not verify network enforcement or Vault policies.

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
3. Run `helm install` in the client's namespace with common values, client values,
   any local overrides and finally `examples/bootstrap.values.yaml`. Both proxy
   modes remain disabled. Create the database through a local port-forward, using
   the exact database name in the inventory. Verify login and attachments, then
   revoke the application's `CREATEDB` permission through the database administrator.
4. Run `helm upgrade --reset-values` with the same normal values and local overrides,
   omitting the bootstrap file. Confirm TLS synchronization, HTTPS login, WebSockets
   and attachment access. Repeat independently for the other clients.

Database initialization remains manual. Helm does not create Vault policies, issue
certificates, install the shared controller or implement backups. Configuration
rollback does not reverse database migrations. Retained PVCs do not protect against
namespace deletion.

If an installation already exists, keep its original release, namespace, database
and PVC identity using direct Helm commands until an explicit migration is planned.
The new fleet names do not adopt or migrate an existing `challenge` release.

## Add the fourth or hundredth client

Add `clients/<client-id>/values.yaml` with unique database/role names, hostname,
Vault role and KV paths, following the three examples. Provision the corresponding
Vault binding and secrets, run validation, then initialize/deploy that client.
Use the same chart with the new values path and namespace in the Helm commands.
Adding a directory does not deploy it automatically. Validate hostname and Vault
path uniqueness during review; Helm lint cannot prove the Vault policies or DNS
records are correct.

Common-value or chart edits affect every client at their next deployment. Validate
the whole fleet, then deploy to a test client and small batches with health checks
between batches. Pin images by digest and promote reviewed Git revisions. For 100
clients, a GitOps controller can consume this same directory layout; Argo CD is not
installed or configured by this milestone.

Current application/database requests total 0.75 CPU, 1.25 GiB RAM and 30 GiB of
storage per client: three clients request 2.25 CPU, 3.75 GiB RAM and 90 GiB storage;
100 request 75 CPU, 125 GiB RAM and 3,000 GiB storage. These exclude shared services,
backup capacity, storage replication and operating headroom, and are not load-test
results. Limits allow higher usage. ResourceQuota caps each namespace (including
temporary jobs) and does not reserve node capacity. Revisit quotas when increasing
client sizing. See [Kubernetes quotas](https://kubernetes.io/docs/concepts/policy/resource-quotas/).

Before production or growth toward 100: implement coordinated database/filestore
backups and restore drills, per-client monitoring, upgrade/migration automation and
capacity measurements. Both Odoo and PostgreSQL remain single replicas with downtime
during some failures/upgrades. A future PostgreSQL operator may add replication and
backup automation while preserving a dedicated PostgreSQL cluster for every client.
Distribute clients across bounded Kubernetes clusters as workload and recovery needs
require; client count alone does not determine the number of clusters.
