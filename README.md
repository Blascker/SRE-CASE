# Odoo on Kubernetes with Helm

A readable Helm adaptation of the [OEC Docker Compose guide](https://oec.sh/guides/odoo-docker-compose): official Odoo and PostgreSQL images, persistent storage, health checks, a dedicated Traefik HTTPS proxy, and secrets synchronized from the existing Vault through Vault Secrets Operator (VSO). All application upstreams use internal Kubernetes Service FQDNs.

This is the initial implementation for the SRE challenge. Nothing has been applied to a cluster. See [DESIGN.md](DESIGN.md) for the decisions and remaining production work.

## Files

```text
charts/odoo/
  Chart.yaml
  values.yaml                    # Images, resources, DNS, Traefik and Vault settings
  templates/
    odoo/
      configmap.yaml            # Readable odoo.conf
      deployment.yaml           # Application, config preparation and probes
      service.yaml
      pvc.yaml
      networkpolicy.yaml
    postgresql/
      configmap.yaml            # One-time application role creation
      statefulset.yaml
      service.yaml
      pvc.yaml
      networkpolicy.yaml
    traefik/
      configmap.yaml            # HTTPS, WebSockets and explicit FQDN upstreams
      deployment.yaml
      service.yaml              # The only potentially external-facing Service
      networkpolicy.yaml
      vault-static-secret.yaml  # TLS certificate synchronization
    vault/
      connection.yaml           # Existing Vault's internal service address
      auth.yaml                 # Kubernetes auth method and existing Vault role
      serviceaccount.yaml       # Identity VSO uses to authenticate
      credentials.yaml          # Shared Odoo/PostgreSQL credentials synchronization
    common/
      validate.yaml             # Helm checks; produces no Kubernetes resource
examples/
  bootstrap.values.yaml          # Temporary first-install configuration
  traefik.values.yaml            # Example HTTPS proxy configuration
  vault.values.yaml              # Vault namespace, role and paths
docs/
  vault.md                       # Existing Vault prerequisites and secret contract
```

## Validate locally

Only Helm is required. These commands do not contact Kubernetes:

```sh
helm lint --strict charts/odoo
helm template challenge charts/odoo --namespace odoo-challenge
helm template challenge charts/odoo --namespace odoo-challenge \
  -f examples/bootstrap.values.yaml
helm template challenge charts/odoo --namespace odoo-challenge \
  -f examples/vault.values.yaml -f examples/traefik.values.yaml
```

The last command checks Traefik rendering. Use an ignored `values.local.yaml` for actual cluster settings. Previous `ingress.*` settings have been replaced by `traefik.*`; the chart rejects obsolete ingress settings.

`templates/common/validate.yaml` is a Helm template containing `fail` checks. Helm evaluates it during linting/rendering and stops with a readable error for incompatible settings, such as exposing the database-manager setup through Traefik. It emits no Kubernetes object and runs no service.

## Vault secret synchronization

VSO is enabled by default. Vault remains in `vault-namespace`; the connection assumes Service `vault` on HTTPS port 8200. The existing Kubernetes-auth role is `vault-role`, and the KV v2 location is mount `ODOO`, path `CLIENT-A`. Confirm the Service name, auth mount (`kubernetes`), token audience (`vault`) and Vault CA trust before deployment.

VSO reads these fields from `ODOO/CLIENT-A`:

| Vault field | Destination Secret | Consumer |
| --- | --- | --- |
| `admin-password` | `odoo-credentials` | Odoo master password |
| `db-password` | `odoo-credentials` | Odoo connection and PostgreSQL role initialization |
| `postgres-password` | `odoo-credentials` | PostgreSQL administrator |
| `tls.crt`, `tls.key` | `odoo-tls` | Traefik certificate and key |

The TLS sync resource is created only when Traefik is enabled. Both destination names remain configurable. VSO creates the Secrets in the release namespace, not the Vault namespace. Workloads continue reading the same local Secret keys; no passwords or private keys are added to Helm values. See [Vault setup](docs/vault.md) for role/policy binding, sync checks and rotation behavior.

## Services and internal addresses

PostgreSQL is the required external application dependency. `psql` is its command-line client. Odoo does not require Redis in this configuration; HTTP, WebSockets and scheduled jobs run in the Odoo container. SMTP is optional if sending email is required, and is currently blocked by egress policy.

For release `challenge`, namespace `odoo-challenge` and `clusterDomain: cluster.local`:

| Connection | Address |
| --- | --- |
| Traefik to Odoo HTTP | `http://challenge-odoo.odoo-challenge.svc.cluster.local:8069` |
| Traefik to Odoo WebSockets | `http://challenge-odoo.odoo-challenge.svc.cluster.local:8072` |
| Odoo to PostgreSQL | `challenge-db.odoo-challenge.svc.cluster.local:5432` |

Odoo uses a ClusterIP Service; PostgreSQL uses a headless internal Service. Neither exposes a node port or external load balancer. Traefik is the only entry point and redirects HTTP to HTTPS. For a completely internal frontend as well, set `traefik.service.type: ClusterIP`.

NetworkPolicies permit only the expected application peers and DNS. They enforce allowed destinations, not whether a client used DNS or the resulting IP address. Kubernetes probes and PostgreSQL's local checks are not inter-service upstreams.

## Prerequisites for a future deployment

- An explicitly chosen kubeconfig context and an existing dedicated namespace.
- A StorageClass supporting writable volumes and the configured `fsGroup` permissions.
- A CNI that enforces NetworkPolicy, the correct `clusterDomain`, and `networkPolicy.dnsTo` matching your DNS implementation. The default assumes CoreDNS/kube-dns pods in `kube-system`; NodeLocal DNS needs its specific destination configured.
- VSO and its `secrets.hashicorp.com/v1beta1` CRDs installed, watching the target namespace and authorized to create Secrets and restart Deployments there.
- Existing Vault configured for Kubernetes authentication, with role `vault-role` bound to this release's auth ServiceAccount and namespace. Populate the KV entry with three different strong single-line passwords without leading/trailing whitespace. See [Vault setup](docs/vault.md).
- Trust for the Vault server certificate. For a private CA, supply a separate CA Secret with `ca.crt` in the release namespace and set `vault.server.caCertSecret`; this trust bootstrap cannot depend on the same Vault sync.
- For public access: load-balancer support, a DNS record pointing to Traefik, and a certificate/private key in Vault matching `traefik.host`. VSO creates the TLS Secret. This chart runs Traefik itself; no ingress controller, IngressClass or Traefik CRDs are required.

## First database setup — later, when deployment is authorized

The commands in this section change the cluster. They are documented for later use and have not been executed. Set `TARGET_CONTEXT` and `TARGET_NAMESPACE` explicitly first.

1. Complete the existing Vault/VSO prerequisites, populate `ODOO/CLIENT-A`, and install with the temporary bootstrap settings, which keep Traefik disabled:

   ```sh
   helm upgrade --install challenge charts/odoo \
     --kube-context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
     -f values.local.yaml -f examples/bootstrap.values.yaml
   ```

   VSO will create the local credentials Secret. Pods can remain pending until the first sync succeeds. Inspect `VaultStaticSecret/challenge-credentials` status and events without printing Secret data before proceeding.

2. Access the database manager through a local-only port-forward:

   ```sh
   kubectl --context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
     port-forward service/challenge-odoo 8069:8069
   ```

   Open `http://localhost:8069/web/database/manager`. Use the master password to create the database named in your normal `odoo.database` value, choose an administrator login and a separate strong login password, and leave demo data disabled. Verify login and attachment upload/download.

3. Upgrade with the normal values, omitting the bootstrap file and avoiding `--reuse-values`:

   ```sh
   helm upgrade challenge charts/odoo \
     --kube-context "$TARGET_CONTEXT" --namespace "$TARGET_NAMESPACE" \
     -f values.local.yaml
   ```

   Normal settings restrict Odoo to one database and disable database listing. Revoke the application's `CREATEDB` permission after setup with `ALTER ROLE odoo NOCREATEDB;` through the database administrator; substitute the configured role name. Enable `traefik.enabled` only after setup and TLS provisioning have been verified, keeping `odoo.proxyMode: true`.

## Day-two behavior

- ConfigMap changes restart Odoo through its checksum annotation. VSO restarts Odoo when synchronized credentials change so the runtime config and database password are reloaded.
- Traefik configuration changes trigger a rollout. VSO also restarts Traefik when its synchronized TLS data changes. Certificate issuance/renewal is external: publish renewed certificate/key data to Vault, then VSO distributes it. No ACME or Vault PKI issuance is configured by this chart.
- PostgreSQL initialization SQL runs only on an empty data directory. Editing the Secret does **not** change passwords already stored in PostgreSQL; rotate database credentials and the Secret together.
- Both PVCs are retained on Helm uninstall. Retention is not a backup. Reinstall with the same release name only after checking ownership of retained resources; do not delete volumes to resolve installation errors.
- Application updates use `Recreate` and interrupt service. Database/module migrations require a separate maintenance procedure and a verified backup. Helm rollback does not reverse database changes.
- Outbound SMTP, external APIs and other integrations are denied. Add a deliberate NetworkPolicy rule and configure an internal service FQDN when introducing another dependency.
- Set `vault.enabled: false` only when credentials/TLS Secrets are supplied by another process. In that mode, restart Odoo manually after credential changes and use `traefik.tlsRevision` for certificate reloads.
