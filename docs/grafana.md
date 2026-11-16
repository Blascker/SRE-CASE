# Grafana dashboards

Grafana runs as a StatefulSet with a 5 GiB Longhorn RWO volume. Its ConfigMaps
[provision](https://grafana.com/docs/grafana/latest/administration/provisioning/)
the Prometheus data source and a Kubernetes overview dashboard.
Install [Prometheus](prometheus.md) first; both use namespace `monitoring`.

```text
Browser --> Grafana --queries--> Prometheus
```

## Install and open

Create the administrator Secret from private files containing the exact values
without trailing newlines. Keep these files outside Git.

```sh
kubectl -n monitoring create secret generic grafana-admin \
  --from-file=admin-user=/secure/path/grafana-user \
  --from-file=admin-password=/secure/path/grafana-password
kubectl apply -f services/grafana/
kubectl -n monitoring rollout status statefulset/grafana --timeout=10m
kubectl -n monitoring port-forward service/grafana 3000:3000
```

Open `http://localhost:3000`, sign in with the supplied credentials and select
**Kubernetes overview**. Filter by namespace to inspect a client, Vault or monitoring.
The data source uses the internal Service `http://prometheus:9090`.

## What the dashboard shows

- Node readiness, unreachable scrape targets and pod readiness/phases.
- CPU/memory by pod and container restarts.
- PVC binding and filesystem usage, desired HPA replicas and raw Longhorn health.
- Deployment available replicas, PV phases and Kubernetes Service metadata.

Node, scrape-target, PV and Longhorn panels remain cluster-wide. More node metrics
are available through Grafana Explore. Missing data is **unknown**, not zero;
PVC usage depends on kubelet/CSI support. Vault seal status and PostgreSQL internals
need separate application exporters. See [collection checks](prometheus.md#check-collection-and-alerts)
if panels show no data.

Service metadata is an inventory of Kubernetes Services, not a test of whether
the applications behind them respond successfully.

## Maintenance

The Service stays internal; use the local port-forward for access. One replica means
brief downtime during replacement. Back up Grafana state; retained PVCs do not protect
against namespace deletion. Resources and storage are configured in
[`services/grafana/grafana-sts.yaml`](../services/grafana/grafana-sts.yaml).

After changing the data source ConfigMap or administrator Secret, apply `services/grafana/`
and restart with `kubectl -n monitoring rollout restart statefulset/grafana`.
Changing the Secret does not reset an existing Grafana database password; manage
later password changes through Grafana. Provisioned dashboard changes are scanned
periodically after applying their ConfigMap; edit the source rather than the UI.
