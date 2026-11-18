Bare Metal Kubernetes Cluster with 2 nodes in [Talos Linux 1.11](https://docs.siderolabs.com/kubernetes-guides/overview/kubernetes-guides-overview)

The current components for the base infrastructure are:
- Longhorn
- Cilium (without kube proxy)
- Pihole
- Traefik
- MetalLb
- ExternalDNS
- [Prometheus](https://artifacthub.io/packages/helm/prometheus-community/prometheus)
- [Grafana](https://artifacthub.io/packages/helm/grafana/grafana)
- [Loki](https://artifacthub.io/packages/helm/grafana/loki?modal=values&path=deploymentMode)
- Promtail

Helmfile was used to declare and manage most of these workloads and can be found in helm/helmfile.yaml
