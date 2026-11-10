# Vault Secrets Operator setup

Vault is existing infrastructure in `vault-namespace`. This chart creates VSO resources in the Odoo release namespace and synchronizes existing KV data; it does not install Vault/VSO, configure the Vault server, or issue certificates.

## Connection and identity

Supplied values are role `vault-role`, KV mount `ODOO`, and path `CLIENT-A`. The chart assumes KV v2 and Service `vault` on HTTPS port 8200. Confirm those details, the Kubernetes auth mount `kubernetes`, and token audience `vault` before deployment. Configure overrides in `values.local.yaml`.

For release `challenge` in namespace `odoo-challenge`:

| Resource | Name / value |
| --- | --- |
| VaultConnection | `challenge-vault` |
| Server address | `https://vault.vault-namespace.svc.cluster.local:8200` |
| VaultAuth and authentication ServiceAccount | `challenge-vault-auth` |
| Vault auth role | `vault-role` |
| Bound Kubernetes ServiceAccount | `challenge-vault-auth` |
| Bound Kubernetes namespace | `odoo-challenge` |
| Expected token audience | `vault` |

The Vault administrator must configure the Kubernetes auth backend and bind the existing role to the exact application namespace and generated ServiceAccount. Changing Helm release/namespace changes that identity. The operator needs TokenRequest access for the ServiceAccount, permission to manage local Secrets and restart the Odoo/Traefik Deployments, and a watch scope covering this namespace.

The role's policy needs read access to the configured KV v2 data endpoint. With the supplied mount/path, the policy fragment is:

```hcl
path "ODOO/data/CLIENT-A" {
  capabilities = ["read"]
}
```

This is configuration for the existing Vault administrator to review, not a policy installed by the chart. `vault.kvMount` is `ODOO` and the VSO `path` is `CLIENT-A`; do not add `data/` to either chart value.

If Vault uses a private CA, place its public CA chain in a Kubernetes Secret with key `ca.crt` in the application namespace, then set `vault.server.caCertSecret`. That initial trust Secret must be supplied independently. Set `tlsServerName` only if needed to match Vault's certificate name. TLS verification remains enabled. `vault.auth.namespace` refers to Vault Enterprise namespaces, not Kubernetes namespaces.

## KV data contract

Populate these keys in `ODOO/CLIENT-A` through the existing Vault administration process:

| Key | Purpose |
| --- | --- |
| `admin-password` | Odoo database-manager master password |
| `db-password` | Odoo's PostgreSQL role password |
| `postgres-password` | PostgreSQL administrator password |
| `tls.crt` | PEM certificate/full chain for `traefik.host` |
| `tls.key` | Matching PEM private key |

Passwords must be strong, distinct, single-line values without leading/trailing whitespace. The Odoo web administrator login/password is chosen during database bootstrap and is separate from the master password.

`vault/credentials.yaml` selects the three password keys into the local `odoo-credentials` Secret. `traefik/vault-static-secret.yaml` selects the two TLS keys into a local Secret of type `kubernetes.io/tls`. The TLS resource exists only when Traefik is enabled. The two syncs can use separate KV paths by changing `credentialsPath` and `tlsPath` later.

No values are committed to Git or rendered into Helm manifests. VSO creates the destination Secrets with `overwrite: false`; it will not silently take over an unmanaged existing Secret. Review any existing destination Secret ownership and plan its migration before enabling synchronization.

## First synchronization and access

Install VSO and its compatible CRDs through the cluster's infrastructure process first. The operator must reach Vault, Kubernetes API and DNS; Vault must allow the operator's incoming connection. Application pods do not contact Vault, so their network policies need no Vault exception.

Once future deployment is authorized, follow README bootstrap steps. Pods may wait until VSO has produced the destination Secret. Inspect the `VaultStaticSecret` status, conditions and events; never print Secret payloads merely to check synchronization. Then verify database bootstrap and login before enabling Traefik.

Helm can render the resources without an installed operator; rendering does not demonstrate successful authentication or secret synchronization. A Vault outage preserves already synchronized Secrets, but prevents updates and can block first-time deployment. Existing pods do not depend on contacting Vault per request.

## Rotation

VSO polls at `refreshAfter` (default one minute). Changed credentials trigger an Odoo rollout, and changed TLS data triggers a Traefik rollout. PostgreSQL is deliberately absent from automatic restart targets.

For database password rotation, stop application writers during maintenance, change the database role password through SQL, publish the identical value to Vault, wait for synchronization, and resume Odoo. Coordinate administrator password changes the same way. A Kubernetes Secret update or PostgreSQL restart alone cannot change stored database passwords. Test this procedure before using it in production.

For certificate renewal, publish the renewed matching certificate/key pair together to Vault. VSO syncs it and restarts Traefik. This chart does not renew certificates through ACME or Vault PKI; their issuance process must exist separately.

With `vault.enabled: false`, no VSO resources are rendered. Another process must provide the destination Secrets, Odoo restarts become manual, and `traefik.tlsRevision` can trigger certificate reloads.

API fields follow the [HashiCorp VSO reference](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/vso/api-reference). Confirm compatibility with the operator version installed in your cluster before applying.
