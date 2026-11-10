# Odoo on Kubernetes with Helm

One reusable Helm chart, three independent clients. Each client has its own Odoo,
PostgreSQL instance, namespace, two persistent volumes, Vault identity and resource
quota. The client configurations use an existing shared Traefik ingress controller.

This is a Helm adaptation of the [OEC Docker Compose guide](https://oec.sh/guides/odoo-docker-compose).
Nothing has been applied to a cluster. [DESIGN.md](DESIGN.md) describes the architecture
and remaining production work; [docs/clients.md](docs/clients.md) explains configuration
and growth toward 100 clients.

## Client inventory

| Client | Helm release | Namespace | Database / database role | Vault role | Vault KV location | Example hostname |
| --- | --- | --- | --- | --- | --- | --- |
| client-a | odoo | odoo-client-a | client_a | odoo-client-a | ODOO/CLIENT-A | client-a.odoo.example.com |
| client-b | odoo | odoo-client-b | client_b | odoo-client-b | ODOO/CLIENT-B | client-b.odoo.example.com |
| client-c | odoo | odoo-client-c | client_c | odoo-client-c | ODOO/CLIENT-C | client-c.odoo.example.com |

The release name `odoo` repeats in separate namespaces. Each release owns a separate
PostgreSQL StatefulSet and database volume; there is no shared database server.

## Files

```text
charts/odoo/
  Chart.yaml
  values.yaml                    # Common chart defaults
  templates/
    odoo/                        # Application, config, Service, PVC, network policy
    postgresql/                  # Dedicated database, initialization, PVC, network policy
    traefik/                     # Optional dedicated proxy and TLS secret sync
    ingress/                     # Routes through an existing shared controller
    vault/                       # VSO connection, identity and credentials sync
    common/
      validate.yaml              # Helm configuration checks; no Kubernetes resource
      resourcequota.yaml         # Optional namespace consumption limits
clients/
  common.values.yaml             # Shared ingress and quotas for all clients
  client-a/values.yaml           # Client A database, hostname and Vault identity
  client-b/values.yaml
  client-c/values.yaml
examples/
  bootstrap.values.yaml          # Temporary database setup; both proxy modes disabled
  traefik.values.yaml            # Standalone dedicated-proxy example
  vault.values.yaml              # Standalone Vault example
docs/
  vault.md                       # Vault prerequisites, client bindings and rotation
  clients.md                     # Configuration, ingress prerequisites and scaling
```

## Prerequisites

- Helm and kubectl, with an explicitly selected Kubernetes context.
- A StorageClass supporting the configured volume permissions, and a CNI enforcing
  NetworkPolicy. Check `clusterDomain` and `networkPolicy.dnsTo` against the cluster.
- VSO and its `secrets.hashicorp.com/v1beta1` CRDs, watching all client namespaces
  and authorized to synchronize Secrets and restart the client Odoo Deployments.
- Existing Vault Kubernetes-auth roles bound to `odoo-vault-auth` in each client's
  exact namespace. Create the separate policies and KV entries in
  [docs/vault.md](docs/vault.md#client-fleet-bindings). The chart does not create them.
- Vault TLS trust. If using a private CA, pre-create each namespace and its CA Secret
  with key `ca.crt`, then configure `vault.server.caCertSecret` before installing.
- An existing shared Traefik controller with IngressClass `traefik`, HTTPS entrypoint
  `websecure`, HTTP-to-HTTPS redirect and access to the client namespaces. Match
  `sharedIngress.controller.namespace` and `podLabels` to its actual deployment.
  See [the ingress prerequisites](docs/clients.md#shared-ingress-prerequisites).
- Real hostnames replacing `*.odoo.example.com`, DNS pointing to the shared controller,
  and matching certificates/private keys in each client's Vault entry.

Values are applied in this order: chart defaults → `clients/common.values.yaml` →
`clients/<client>/values.yaml`. For environment-specific settings, add an ignored
`-f cluster.local.yaml` after the client values on every command. During initial
setup, keep `-f examples/bootstrap.values.yaml` **last**. Never put secrets in values.

## Validate locally

These commands require only Helm and do not contact Kubernetes. Substitute the
client name and namespace to check client-b or client-c.

```sh
helm lint --strict charts/odoo --namespace odoo-client-a \
  -f clients/common.values.yaml -f clients/client-a/values.yaml

helm template odoo charts/odoo --namespace odoo-client-a \
  -f clients/common.values.yaml -f clients/client-a/values.yaml

helm template odoo charts/odoo --namespace odoo-client-a \
  -f clients/common.values.yaml -f clients/client-a/values.yaml \
  -f examples/bootstrap.values.yaml
```

`templates/common/validate.yaml` rejects incompatible settings, including enabling
both proxy modes or exposing the database-manager bootstrap configuration. It emits
no Kubernetes resource. Rendering does not verify Vault, DNS or cluster behavior.

## Install a client with Helm

Run from the repository root after completing the prerequisites. These commands
change the cluster; they have not been executed here. Set `TARGET_CONTEXT` to the
intended kubeconfig context before running them.

### 1. Install with database setup enabled

For **client-a**:

```sh
helm install odoo charts/odoo \
  --kube-context "${TARGET_CONTEXT:?Set TARGET_CONTEXT first}" \
  --namespace odoo-client-a --create-namespace \
  -f clients/common.values.yaml \
  -f clients/client-a/values.yaml \
  -f examples/bootstrap.values.yaml \
  --wait --timeout 10m
```

For **client-b** and **client-c**, the namespace and client values change:

```sh
helm install odoo charts/odoo \
  --kube-context "${TARGET_CONTEXT:?Set TARGET_CONTEXT first}" \
  --namespace odoo-client-b --create-namespace \
  -f clients/common.values.yaml \
  -f clients/client-b/values.yaml \
  -f examples/bootstrap.values.yaml \
  --wait --timeout 10m

helm install odoo charts/odoo \
  --kube-context "${TARGET_CONTEXT:?Set TARGET_CONTEXT first}" \
  --namespace odoo-client-c --create-namespace \
  -f clients/common.values.yaml \
  -f clients/client-c/values.yaml \
  -f examples/bootstrap.values.yaml \
  --wait --timeout 10m
```

Bootstrap disables both shared ingress and the dedicated proxy. VSO creates the
credentials Secret; pods can wait until the first synchronization succeeds. If
installation times out, inspect `VaultStaticSecret/odoo-credentials` status/events
in the affected namespace and fix the cause before using `helm upgrade` to retry.
Do not delete retained volumes to resolve an installation error.

### 2. Create the client's database

For client-a, open a local-only port-forward:

```sh
kubectl --context "${TARGET_CONTEXT:?Set TARGET_CONTEXT first}" \
  --namespace odoo-client-a port-forward service/odoo-odoo 8069:8069
```

Open `http://localhost:8069/web/database/manager`. Use that client's Vault
`admin-password` as the master password, create database **`client_a`**, choose a
separate administrator login/password and leave demo data disabled. Verify login
and attachment upload/download, then stop the port-forward.

Revoke the application's database-creation permission through the PostgreSQL
administrator: `ALTER ROLE client_a NOCREATEDB;`.

Repeat for the other clients using their namespace and matching database/role from
the inventory: `client_b` and `client_c`. Initialize each database before enabling
its normal configuration. Helm readiness alone does not prove initialization.

### 3. Enable normal operation

For client-a, remove the bootstrap override and reset the saved release values:

```sh
helm upgrade odoo charts/odoo \
  --kube-context "${TARGET_CONTEXT:?Set TARGET_CONTEXT first}" \
  --namespace odoo-client-a --reset-values \
  -f clients/common.values.yaml \
  -f clients/client-a/values.yaml \
  --wait --timeout 10m
```

Use the same command with `odoo-client-b` / `clients/client-b/values.yaml` and
`odoo-client-c` / `clients/client-c/values.yaml` after initializing each client.
Repeat any environment-specific `-f` overrides used at installation. Do not use
`--reuse-values`: it can preserve the temporary bootstrap settings.

Normal values select the client's database, disable database listing and create
its shared Ingress and TLS synchronization resource. Verify TLS synchronization,
HTTPS login, WebSockets and attachments at the configured hostname. The shared
controller must provide HTTP-to-HTTPS redirection.

Use this normal `helm upgrade` command for subsequent configuration changes.
Application/module migrations require a separate backup and maintenance procedure.
Existing installations with different release, namespace, database or PVC names
need an explicit migration; these commands do not adopt a previous `challenge` release.

## Internal services and secrets

For client-a, Odoo uses `odoo-db.odoo-client-a.svc.cluster.local:5432`. Its HTTP and
WebSocket ports are exposed through the internal Service `odoo-odoo` on 8069/8072.
The shared controller uses Kubernetes Service references to route `/` and `/websocket`.
Neither Odoo nor PostgreSQL creates a public load balancer.

NetworkPolicies allow the configured ingress controller → Odoo → that client's
PostgreSQL, plus DNS. Ingress-controller access matches both its namespace and pod
labels; database access remains within the client namespace. Bootstrap allows no
application ingress peers. Administrative port-forwarding is a separate control-plane path.

VSO creates these namespace-local Secrets from each client's KV entry:

| Vault fields | Destination Secret | Consumer |
| --- | --- | --- |
| `admin-password`, `db-password`, `postgres-password` | `odoo-credentials` | Odoo/PostgreSQL, with only the required keys mounted per workload |
| `tls.crt`, `tls.key` | `odoo-tls` | Shared ingress controller, or dedicated proxy |

The TLS sync exists only when a proxy mode is enabled. Vault remains existing
infrastructure in `vault-namespace`; its endpoint, auth mount, audience and CA trust
must match the environment. See [the Vault runbook](docs/vault.md).

## Optional dedicated proxy

The standalone chart also supports `traefik.enabled: true` instead of shared ingress.
To use it with client values, explicitly set `sharedIngress.enabled: false`, configure
`traefik.host` and `traefik.service.type`, and keep Odoo proxy mode enabled. The
dedicated proxy uses internal FQDN upstreams and provides its own HTTPS redirect.
It needs no IngressClass or shared controller. Both proxy modes cannot be enabled
together; obsolete `ingress.*` settings are rejected.

`examples/traefik.values.yaml` demonstrates this mode for standalone chart rendering.
The three client files use shared ingress by default.

## Day-two behavior

- Odoo configuration changes and VSO credential changes restart Odoo.
- Shared ingress watches the client's TLS Secret. Dedicated Traefik instead restarts
  when VSO updates its certificate. Certificate issuance and renewal are external.
- PostgreSQL initialization runs only on an empty data directory. Secret updates do
  not change existing SQL passwords; coordinate database and Vault rotation.
- Both PVCs are retained on Helm uninstall, but not protected from namespace deletion.
  Retention is not a backup. Database and filestore must be recovered together.
- Odoo uses `Recreate`; application updates interrupt service. Both Odoo and PostgreSQL
  are single replicas. Helm rollback cannot reverse database/module migrations.
- SMTP and external integrations are blocked by default and need deliberate egress rules.
- With `vault.enabled: false`, supply Secrets separately and restart Odoo after credential
  changes. For dedicated Traefik, use `traefik.tlsRevision` to trigger a certificate reload.

Adding another client means adding its values file, provisioning its Vault identity
and secrets, then repeating the Helm install/bootstrap/upgrade steps with a new namespace.
Backups, monitoring, automated onboarding and GitOps remain future work, as described
in [DESIGN.md](DESIGN.md#three-clients-with-a-path-to-100).
