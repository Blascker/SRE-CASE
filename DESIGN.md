# Design

## Scope and current state

This chart deploys one customer installation into a dedicated namespace: one Odoo pod, one PostgreSQL pod, separate persistent volumes, an optional dedicated Traefik proxy, and VSO resources that synchronize secrets from the existing Vault in `vault-namespace`. It follows the service layout of the [OEC Docker Compose guide](https://oec.sh/guides/odoo-docker-compose) while using Kubernetes workloads, Secrets, probes and NetworkPolicies.

The current milestone is a readable, locally renderable chart. Deployment, Vault authentication/synchronization, database bootstrap, storage permissions, restore behavior and application functionality remain unverified on the target cluster. This is not yet a claim of production readiness.

```text
Browser -- HTTPS --> Traefik Service (the only external endpoint)
                            |
                     Traefik Deployment (1)
                            |
                    internal Odoo Service FQDN
                    :8069 HTTP / :8072 WebSockets
                            |
                     Odoo Deployment (1) ---- Odoo data PVC
                            |
                   internal PostgreSQL Service FQDN :5432
                            |
                 PostgreSQL StatefulSet (1) ---- PostgreSQL data PVC
```

## How the Compose design maps to this chart

| Compose concept | Helm/Kubernetes implementation |
| --- | --- |
| Odoo service | `templates/odoo/deployment.yaml` and `service.yaml` |
| PostgreSQL service | `templates/postgresql/statefulset.yaml` and headless Service |
| Named volumes | Two retained PVCs |
| Mounted application configuration | ConfigMap plus private runtime config volume |
| Environment credentials | VSO-created local Secrets referenced by the pods |
| Health checks and restart policy | Startup/readiness/liveness probes and workload controllers |
| Reverse proxy and certificates | Dedicated Traefik Deployment, Service and mounted TLS Secret |

The chart runs Traefik with its file provider, not as a cluster-wide ingress controller. It needs no Kubernetes API credentials, IngressClass, Ingress resource or Traefik CRDs. Certificate issuance is external. No custom application image, Python code, chart dependencies or shared pod-template framework is introduced.

Templates are grouped under `odoo/`, `postgresql/`, `traefik/` and `vault/`. Shared render-time checks live in `common/validate.yaml`: Helm evaluates them to reject incompatible values, but the file emits no Kubernetes resource. Config checksum references point to the new service folders.

## Vault Secrets Operator

Vault already exists in Kubernetes namespace `vault-namespace`. This chart creates a local `VaultConnection`, `VaultAuth`, authentication ServiceAccount, and `VaultStaticSecret` resources. It installs neither Vault nor VSO. The operator and its CRDs must already exist and watch the application namespace.

The supplied role is `vault-role`; secrets use KV v2 mount `ODOO`, path `CLIENT-A`. The configured address is `https://vault.vault-namespace.svc.cluster.local:8200`, built from the service, namespace and cluster-domain values. Service name `vault`, Kubernetes auth mount `kubernetes`, audience `vault`, and KV engine version are assumptions to confirm. A private Vault CA must be supplied in a separate local trust Secret; TLS verification stays enabled. `vault.auth.namespace` is an optional Vault Enterprise namespace and is distinct from the Kubernetes namespace.

VSO authenticates using a short-lived token for `<release>-vault-auth` in the application namespace. The existing Vault role must bind that exact ServiceAccount/namespace and grant read access to the configured KV path. No Vault token is stored in chart values or mounted into application pods. Workload service-account token automounting remains disabled.

The [VSO secret resources](https://developer.hashicorp.com/vault/docs/deploy/kubernetes/vso/sources/vault) reconcile Vault data into Kubernetes Secrets. One sync selects `admin-password`, `db-password` and `postgres-password` into `existingSecret`; another selects `tls.crt` and `tls.key` into `traefik.tlsSecret` when Traefik is enabled. Raw Vault payloads and unrelated fields are excluded. Destination Secrets are created locally, with overwrite of unmanaged existing Secrets disabled. Secret values remain in Vault and the destination Kubernetes Secrets, so Kubernetes RBAC and encryption at rest still matter.

VSO polls every minute and uses data-change detection plus rollout targets to restart Odoo after credential changes and Traefik after TLS changes. It deliberately does not restart PostgreSQL: a restart cannot change passwords stored in SQL. Coordinate database password updates and KV updates in a maintenance window; details are in [the Vault runbook](docs/vault.md). TLS data synchronization distributes an existing certificate; issuance and renewal still happen outside this chart.

Only the operator contacts Vault. The existing application NetworkPolicies continue to permit Traefik → Odoo → PostgreSQL and DNS, without application-to-Vault access. The operator's own namespace policies must allow Vault/API/DNS access; Vault's namespace policies must admit the operator. Those existing infrastructure policies are not modified here. Pods can wait for initial Secret synchronization; Helm rendering does not verify Vault connectivity or the first sync.

## Required services

PostgreSQL is the only required separate application service in this baseline; `psql` is a database client, not another server. The [official Odoo image](https://hub.docker.com/_/odoo) contains the HTTP, WebSocket and scheduled-job processes. Redis and a separate queue service are not required. Persistent filestore storage remains necessary alongside the database.

Traefik provides HTTPS and reverse proxying. SMTP is optional for email. Monitoring and backup systems are operational dependencies to add later. Any new dependency requires its own internal address and an explicit network rule.

## Traefik and internal FQDN routing

`templates/traefik/configmap.yaml` contains the listeners and routes. The [Traefik file provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/others/file/) lets us specify the upstream URLs directly and keep the configuration local to this release.

Every inter-service upstream is an internal FQDN built from the release, namespace and configurable `clusterDomain`:

- Traefik HTTP: `http://<release>-odoo.<namespace>.svc.<clusterDomain>:8069`.
- Traefik WebSockets: `http://<release>-odoo.<namespace>.svc.<clusterDomain>:8072`.
- Odoo database: `<release>-db.<namespace>.svc.<clusterDomain>:5432`, in both `odoo.conf` and the official entrypoint's `HOST` variable.

Traefik gives `/websocket` higher routing priority than the general application route. It listens on unprivileged container ports 8000/8443, exposed as 80/443 by its Service. HTTP redirects explicitly to external port 443. TLS uses a VSO-synchronized mounted certificate Secret, with TLS 1.2 as the minimum. The dashboard and API are not enabled; the separate ping port is only for pod probes. VSO restarts Traefik when certificate data changes. `tlsRevision` remains a manual restart option if VSO is disabled.

Odoo remains ClusterIP-only; PostgreSQL remains headless/internal. Only Traefik may use a LoadBalancer Service. Set its Service to ClusterIP if the browser-facing endpoint must also remain internal. No application upstream uses public DNS or a hard-coded pod IP.

NetworkPolicies restrict Traefik egress to this release's Odoo pods and cluster DNS; Odoo egress to its PostgreSQL pod and DNS; and PostgreSQL ingress to Odoo. PostgreSQL initiates no outbound connections. Odoo accepts application traffic only from this release's Traefik pods. Rules use same-namespace pod selectors and release labels, so other releases are not automatically trusted. Node probes and authorized administrative port-forwarding are separate control-plane paths.

DNS names provide service discovery; NetworkPolicies enforce the permitted destinations and ports. Standard Kubernetes NetworkPolicy cannot prove that a caller used a hostname instead of its resolved IP. Confirm CNI enforcement and DNS selectors, especially if the cluster uses NodeLocal DNS. Additional policies are additive, so review existing namespace policies as well.

## Application configuration

We retain Odoo 19 from the initial plan and use PostgreSQL 16. Odoo configuration stays visible in `templates/odoo/configmap.yaml`; it includes the database filter, worker limits, proxy mode and logging. Defaults use two HTTP workers and one cron worker with a 4 GiB pod limit. Memory limits in Odoo apply per worker, so they must be budgeted against the whole pod. PostgreSQL settings are starting estimates, not benchmark results.

The [official Odoo entrypoint](https://github.com/odoo/docker/blob/master/19.0/entrypoint.sh) supports `PASSWORD_FILE` and waits for PostgreSQL. We keep that entrypoint. The only application-side shell code copies the public configuration and appends the master password from a Secret to a memory-backed volume. The main container mounts the resulting file read-only. The stock entrypoint can pass the database password internally as a process argument; pod access remains privileged access.

No custom addons are mounted yet. Future addons should be delivered in a versioned image derived from the official image, so code changes are reproducible.

## Security and initial setup

All three workloads run as non-root with a read-only root filesystem, dropped capabilities and no service-account token. Writable data, configuration, temporary files and PostgreSQL sockets have explicit volumes. Odoo uses UID/GID 1000 with volume group permissions; PostgreSQL uses the Debian image's UID/GID 999; Traefik uses UID/GID 65532. The target storage driver must support these permissions.

PostgreSQL uses a separate administrator password. Its initialization SQL creates an Odoo role without superuser or role-management privileges. Temporary `CREATEDB` permission enables the database-manager bootstrap flow in the README; revoke it afterwards. The application never receives the PostgreSQL administrator password.

Bootstrap disables Traefik, leaving only internal Services and administrative port-forward access. Normal configuration selects one database and disables database listing. Enabling Traefik while bootstrap settings are active, or with Odoo proxy mode disabled, is rejected by the chart.

Public TLS terminates at Traefik. Internal HTTP and PostgreSQL connections stay inside the cluster and are network-restricted, but are not encrypted in this baseline. Add internal TLS if required by the deployment's threat model. Kubernetes Secrets also require suitable RBAC and encryption at rest. External APIs, SMTP and internet access are denied by default.

## Persistence, availability and upgrades

The Odoo volume holds filestore and sessions. PostgreSQL has its own volume and a `PGDATA` subdirectory. A single replica and `Recreate` avoid overlapping application pods using the same volume during upgrades. They imply downtime; neither workload is highly available.

PVC retention protects against Helm uninstall, not namespace deletion, storage loss or operator mistakes. PostgreSQL belongs to this release for a self-contained challenge demonstration. At fleet scale, move it to a separately managed database lifecycle with replication and tested failover.

The image settings accept full digest references. Resolve and pin all three image digests before deployment; the current tags are human-readable starting versions. Test application/module upgrades against a restored database. A Helm rollback only restores Kubernetes configuration; schema rollback requires a compatible recovery plan. Traefik is also single-replica and does not make the application highly available.

## Backup and recovery plan — not implemented yet

Treat the database and filestore as one recovery set. For the first implementation, stop all Odoo writers, take a PostgreSQL logical backup and filestore archive, record a shared backup identifier and image version, and resume service after both captures complete. Store encrypted copies outside the cluster with checksums and retention.

Restore into a separate database and volume, start the matching Odoo image, then verify login, representative records and attachment contents. Measure actual recovery time. Agree recovery-point and recovery-time targets before choosing the final schedule; scheduled backups and successful restore evidence are required before production acceptance.

## Observability and silent failures

Logs go to container stdout/stderr for the cluster logging system. Traefik emits JSON access logs without request headers; its ping probe checks proxy health, not end-to-end application behavior. Odoo liveness checks the HTTP process; readiness uses the [official health endpoint's database-server check](https://github.com/odoo/odoo/blob/19.0/addons/web/controllers/home.py). That check connects to the maintenance database and does **not** prove the customer database or business workflows are healthy.

Next, integrate alerts for unavailable pods, repeated restarts, latency/errors, database connections, disk capacity and backup age. Add an authenticated synthetic workflow that reads a record and retrieves an attachment, plus freshness checks for scheduled jobs and outgoing mail. Alert delivery and restore drills need independent verification. These integrations are planned, not resources currently installed by the chart.

## Growth from 50 to 500+ customers

Reuse this chart with version-controlled customer values and isolated database credentials. Roll updates out to a small customer group before expanding. Distribute customers across bounded clusters/database groups to limit incident impact. Capacity planning must include worker memory and database connections per customer.

Adding replicas is a separate design change: shared filestore, session behavior, cron execution, storage performance and failover must be tested together. CPU autoscaling alone does not establish an Odoo HA architecture.

## Next milestones

1. Confirm namespace, storage, cluster domain, DNS selectors, load-balancer support, Vault/VSO installation, auth binding and TLS trust.
2. Pin image digests and run Vault sync/rotation, bootstrap, login, HTTPS redirect, WebSocket, FQDN resolution, network-isolation and persistence checks once deployment is authorized.
3. Implement backup/restore automation and collect restore evidence.
4. Integrate monitoring, synthetic checks and alerts.
5. Exercise failure and upgrade scenarios, then prepare the interview presentation from verified results.
