# Prometheus and cluster metrics

The manifests in `services/prometheus/` run Prometheus and Alertmanager as StatefulSets, with metric collectors
for the whole cluster. Each StatefulSet has its own Longhorn volume. Install this once,
then install [Grafana](grafana.md) for dashboards. Both use namespace `monitoring`.

```text
Kubernetes state + kubelet + node-exporter + Longhorn
                          |
                          v
                      Prometheus --> Grafana
                          |
                          v
                     Alertmanager
```

## Install

Select the intended kube-context first. Longhorn must already be available. Create
the monitoring namespace and apply the manifests:

```sh
kubectl apply -f services/prometheus/namespace.yaml
kubectl apply -f services/prometheus/
kubectl -n monitoring rollout status statefulset/prometheus --timeout=10m
kubectl -n monitoring rollout status statefulset/alertmanager --timeout=10m
kubectl -n monitoring get pods,pvc
```

The directory command does not recurse into `optional/`. It creates the monitoring
resources, including cluster RBAC.

| Component | Runs as | Purpose |
| --- | --- | --- |
| Prometheus | StatefulSet, 20 GiB RWO | Collects and stores metrics for up to 15 days or 16 GB |
| Alertmanager | StatefulSet, 2 GiB RWO | Groups alerts; external delivery is not configured |
| kube-state-metrics | Deployment | Node/pod state, Deployment replicas, PV/PVC state, Service metadata and HPA state |
| node-exporter | DaemonSet | Node CPU, memory, filesystem and network metrics |

Prometheus discovers kubelets and scrapes them through the Kubernetes API proxy with
TLS verification. This includes cAdvisor container metrics and kubelet volume usage.
Its ServiceAccount needs `nodes/proxy` access: treat this identity as privileged and
restrict who can modify or execute commands in monitoring pods. kube-state-metrics
has read-only access to the specific workload resource types it collects.

CPU and memory usage come from kubelet/cAdvisor and node-exporter. Object state
comes from kube-state-metrics, including [Service metadata](https://github.com/kubernetes/kube-state-metrics/blob/main/docs/metrics/service/service-metrics.md).
Service metadata describes Kubernetes configuration; it does not verify application
responses or network connectivity.

## Check collection and alerts

```sh
kubectl -n monitoring get pods,pvc
kubectl -n monitoring port-forward service/prometheus 9090:9090
```

Open `http://localhost:9090/targets`. Prometheus, Kubernetes API, kube-state-metrics,
kubelet, cAdvisor, node-exporter and Longhorn targets should be **UP**. Dynamic jobs
should have targets for all eligible nodes/managers. Verify the dashboard contains
actual client pods and Bound PVCs after deployment.

Prometheus rules cover unavailable nodes, pending/failed pods, repeated container
restarts, unbound/nearly-full PVCs and failed scrapes. They appear on its Alerts page.
Alertmanager is available through:

```sh
kubectl -n monitoring port-forward service/alertmanager 9093:9093
```

External notifications require a receiver and credentials in Alertmanager's
configuration. The supplied receiver keeps alerts visible locally without sending
messages. This stack collects metrics, not application logs or event history.

## Networking and maintenance

Longhorn discovery assumes namespace `longhorn-system`, pod label
`app: longhorn-manager` and metrics port name `manager`. No ServiceMonitor is used.
If Longhorn already has ingress-isolating NetworkPolicies, apply the supplemental
allowance explicitly:

```sh
kubectl apply -f services/prometheus/optional/longhorn-networkpolicy.yaml
```

Do not introduce this policy alone on an unrestricted Longhorn installation: a first
policy would block other traffic. Keep the existing Longhorn peer/API access rules.
Also allow Prometheus to reach the Kubernetes API, kube-state-metrics, node-exporter,
Longhorn managers on TCP 9500 and DNS. Node-exporter uses host networking, host PID
visibility and read-only host filesystem mounts, so it needs compatible admission
policies and an available host TCP port 9100. All UIs remain internal; local
port-forwards avoid public exposure.

Each StatefulSet has one replica, so monitoring is interrupted during replacement.
PVCs are retained when StatefulSets are removed; namespace deletion can delete them.
Increase resources/retention storage as ingestion grows. Prometheus and Alertmanager
request about 2.2 GiB memory, plus collectors on each node.
Longhorn replication requires extra physical disk capacity.

After editing ConfigMaps, apply the directory and restart the affected StatefulSet
with `kubectl -n monitoring rollout restart statefulset/<name>`. Keep credentials
out of ConfigMaps.

Metrics Server is still required for CPU HPA. k3s scheduler/controller-manager, etcd
and kube-proxy metrics are not scraped because their endpoints are not exposed by
default. Verify all collection on the actual cluster before relying on the dashboards.

References: [Prometheus Kubernetes discovery](https://prometheus.io/docs/prometheus/latest/configuration/configuration/#kubernetes_sd_config),
[Grafana provisioning](https://grafana.com/docs/grafana/latest/administration/provisioning/).
