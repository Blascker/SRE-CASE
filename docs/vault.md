# Vault Secrets Operator setup

Vault is existing infrastructure in `vault-namespace`. This chart creates VSO resources in the Odoo release namespace and synchronizes existing KV data; it does not install Vault/VSO, configure the Vault server, or issue certificates.

## Connection and identity

The client-a values configure role `odoo-client-a`, KV mount `ODOO`, and path `CLIENT-A`. The chart assumes KV v2 and Service `vault` on HTTPS port 8200. Confirm those details, the Kubernetes auth mount `kubernetes`, and token audience `vault` before deployment. Configure overrides in `values.local.yaml`.

For release `odoo` in namespace `odoo-client-a`:

| Resource | Name / value |
| --- | --- |
| VaultConnection | `odoo-vault` |
| Server address | `https://vault.vault-namespace.svc.cluster.local:8200` |
| VaultAuth and authentication ServiceAccount | `odoo-vault-auth` |
| Vault auth role | `odoo-client-a` |
| Bound Kubernetes ServiceAccount | `odoo-vault-auth` |
| Bound Kubernetes namespace | `odoo-client-a` |
| Expected token audience | `vault` |

The Vault administrator must configure the Kubernetes auth backend and bind the existing role to the exact application namespace and generated ServiceAccount. Changing Helm release/namespace changes that identity. The operator needs TokenRequest access for the ServiceAccount, permission to manage local Secrets and restart the Odoo/Traefik Deployments, and a watch scope covering this namespace.

The role's policy needs read access to the configured KV v2 data endpoint. With client-a's mount/path, the policy fragment is:

```hcl
path "ODOO/data/CLIENT-A" {
  capabilities = ["read"]
}
```

This is configuration for the existing Vault administrator to review, not a policy installed by the chart. `vault.kvMount` is `ODOO` and the VSO `path` is `CLIENT-A`; do not add `data/` to either chart value.

## Client fleet bindings

The [three-client configuration](clients.md) overrides the standalone `vault-role`.
Create a separate policy and Kubernetes auth role for each client; do not grant one
shared role wildcard access to all client secrets.

| Vault role | Bound Kubernetes namespace | Bound ServiceAccount | Read-only KV v2 policy path |
| --- | --- | --- | --- |
| odoo-client-a | odoo-client-a | odoo-vault-auth | ODOO/data/CLIENT-A |
| odoo-client-b | odoo-client-b | odoo-vault-auth | ODOO/data/CLIENT-B |
| odoo-client-c | odoo-client-c | odoo-vault-auth | ODOO/data/CLIENT-C |

Each policy uses the exact path above and `capabilities = ["read"]`. Bind each role
to its exact namespace and ServiceAccount with the configured `vault` audience.
Populate every KV entry with its own database/master passwords, initial web credentials and matching certificate/key;
shared Secret names such as `odoo-credentials` are local to each namespace. The
chart creates VSO resources, not these server-side Vault roles or policies.
VSO must watch all three namespaces. Test that each identity can read its own path
and is denied the other two paths before accepting the isolation setup.

If Vault uses a private CA, place its public CA chain in a Kubernetes Secret with key `ca.crt` in the application namespace, then set `vault.server.caCertSecret`. That initial trust Secret must be supplied independently. Set `tlsServerName` only if needed to match Vault's certificate name. TLS verification remains enabled. `vault.auth.namespace` refers to Vault Enterprise namespaces, not Kubernetes namespaces.

## KV data contract

Populate these keys in each client's KV entry (`ODOO/CLIENT-A`, `ODOO/CLIENT-B` or `ODOO/CLIENT-C`) through the existing Vault administration process:

| Key | Purpose |
| --- | --- |
| `admin-password` | Odoo database-manager master password |
| `db-password` | Odoo's PostgreSQL role password |
| `postgres-password` | PostgreSQL administrator password |
| `odoo-admin-login` | Initial Odoo web administrator login; mounted only in the init container |
| `odoo-admin-password` | Initial Odoo web administrator password; distinct from the master password |
| `tls.crt` | PEM certificate/full chain for the enabled proxy hostname |
| `tls.key` | Matching PEM private key |

Passwords must be strong, distinct, single-line values without leading/trailing whitespace. The init container sets the initial Odoo web administrator from odoo-admin-login and odoo-admin-password only for a new installation. Existing initialized databases retain their current logins; later changes to these Vault fields do not reset web passwords. Keep both fields populated for existing deployments because the init-only Secret volume still requires them.

`vault/credentials.yaml` selects the five credential keys into the local `odoo-credentials` Secret. `traefik/vault-static-secret.yaml` selects the two TLS keys into a local Secret of type `kubernetes.io/tls`. The TLS resource exists when either dedicated Traefik or shared ingress is enabled. The two syncs can use separate KV paths by changing `credentialsPath` and `tlsPath` later.

No values are committed to Git or rendered into Helm manifests. VSO creates the destination Secrets with `overwrite: false`; it will not silently take over an unmanaged existing Secret. Review any existing destination Secret ownership and plan its migration before enabling synchronization.

## First synchronization and access

Install VSO and its compatible CRDs through the cluster's infrastructure process first. The operator must reach Vault, Kubernetes API and DNS; Vault must allow the operator's incoming connection. Application pods do not contact Vault, so their network policies need no Vault exception.

Follow the [README installation steps](../README.md#install-a-client-with-helm). Pods may wait until VSO has produced the destination Secret. Inspect the `VaultStaticSecret` status, conditions and events; never print Secret payloads merely to check synchronization. The init container initializes the database before Odoo starts; verify its completion, HTTPS login and attachments after readiness.

Helm can render the resources without an installed operator; rendering does not demonstrate successful authentication or secret synchronization. A Vault outage preserves already synchronized Secrets, but prevents updates and can block first-time deployment. Existing pods do not depend on contacting Vault per request.

## Rotation

VSO polls at `refreshAfter` (default one minute). Changed credentials trigger an Odoo rollout, and changed TLS data triggers a Traefik rollout. PostgreSQL is deliberately absent from automatic restart targets.

For database password rotation, stop application writers during maintenance, change the database role password through SQL, publish the identical value to Vault, wait for synchronization, and resume Odoo. Coordinate administrator password changes the same way. A Kubernetes Secret update or PostgreSQL restart alone cannot change stored database passwords. Test this procedure before using it in production.

For certificate renewal, publish the renewed matching certificate/key pair together to Vault. VSO syncs it and restarts dedicated Traefik when enabled. In shared ingress mode, the controller watches the namespace's TLS Secret; VSO does not restart the shared controller. This chart does not renew certificates through ACME or Vault PKI; their issuance process must exist separately.

With `vault.enabled: false`, no VSO resources are rendered. Another process must provide the destination Secrets, Odoo restarts become manual, and `traefik.tlsRevision` can trigger certificate reloads.

API fields follow the [HashiCorp VSO reference](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/vso/api-reference). Confirm compatibility with the operator version installed in your cluster before applying.
