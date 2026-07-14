# Traefik v3 Migration Guide

Migrate this Talos homelab from ingress-nginx to Traefik without allowing Helm
or Kustomize to restore old routing configuration later.

This guide is written for the versions currently pinned in this repository:

- Traefik chart `41.0.1` (Traefik `v3.7.5`)
- ingress-nginx chart `4.14.0`
- ExternalDNS chart `9.0.3` using the Pi-hole provider
- MetalLB pool `192.168.50.201-192.168.50.235`

The migration uses standard Kubernetes `Ingress` resources. Gateway API is not
required and remains disabled. Traefik's own `IngressRoute` CRDs can be adopted
later, after the controller migration is stable.

## Migration model

Traefik `v3.7.5` includes the Kubernetes Ingress NGINX provider. During the
transition, that provider reads `nginx-internal` Ingresses and translates the
supported nginx annotations. This lets both controllers serve the same routes
while Traefik is tested on a temporary MetalLB IP.

```text
Initial:      DNS -> 192.168.50.201 -> ingress-nginx -> Services
Validation:   curl -> temporary IP  -> Traefik       -> Services
Cutover:      DNS -> 192.168.50.201 -> Traefik       -> Services
Final:        DNS -> 192.168.50.201 -> Traefik       -> Services
```

MetalLB cannot assign `192.168.50.201:80/443` to both controllers at once.
Therefore, retaining `.201` requires a brief handoff after Traefik has been
validated on another free address. Do not describe this single-IP handoff as
zero downtime. Schedule a short maintenance window.

## Route inventory

The repository currently declares seven Ingress routes:

| Host | Namespace | Declarative owner |
|---|---|---|
| `pihole.home` | `pihole-system` | `helm/values/pihole.values.yaml` |
| `grafana.home` | `grafana-system` | `helm/values/grafana.values.yaml` |
| `loki.home` | `grafana-system` | `helm/values/loki.values.yaml` |
| `longhorn.home` | `longhorn-system` | `helm/values/longhorn.values.yaml` |
| `prometheus.home` | `prometheus-system` | `helm/values/prometheus.values.yaml` |
| `homepage.core` | `homepage` | `base/homepage/ingress.yaml` |
| `jellyfin.home` | `jellyfin` | `base/jellyfin/ingress.yaml` |

Do not use `kubectl patch` as the migration mechanism. Helm and Kustomize own
these resources and would restore the old values on their next reconciliation.

## 1. Preflight

Run these checks with the cluster kubeconfig loaded:

```bash
helmfile --version
helm version
kubectl version

kubectl get pods -n nginx-system
kubectl get svc -n nginx-system ingress-nginx-internal-controller
kubectl get ingressclass
kubectl get ingress -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CLASS:.spec.ingressClassName,ADDRESS:.status.loadBalancer.ingress[*].ip'
kubectl get ipaddresspool -n metallb-system pool -o yaml
```

Confirm that:

- ingress-nginx is healthy in `nginx-system`.
- `192.168.50.201` belongs to its LoadBalancer Service.
- an unused temporary address exists in the MetalLB pool. At the time this
  guide was verified, `192.168.50.204` was free.
- all current hostnames resolve to `192.168.50.201` through Pi-hole.

Check ExternalDNS with the actual release label and namespace:

```bash
kubectl logs -n pihole-system \
  -l app.kubernetes.io/instance=externaldns-pihole \
  --tail=50
```

## 2. Prepare the declarative configuration

### 2.1 Make the Pi-hole Ingress class explicit

Pi-hole currently relies on ingress-nginx being the default class. Before
running two controllers, add this to `helm/values/pihole.values.yaml`:

```yaml
ingress:
  enabled: true
  ingressClassName: nginx-internal
  hosts:
    - pihole.home
```

Apply and verify it:

```bash
helmfile -f helm/helmfile.yaml apply --selector name=pihole
kubectl get ingress -n pihole-system pihole \
  -o jsonpath='{.spec.ingressClassName}{"\n"}'
```

Do not continue until every Ingress has an explicit class.

### 2.2 Preserve the old IngressClass during compatibility mode

Traefik's compatibility provider needs the `nginx-internal` IngressClass after
the ingress-nginx release is removed. Add the keep policy to
`helm/values/nginx-internal.values.yaml` and apply it before uninstalling:

```yaml
controller:
  ingressClassResource:
    annotations:
      helm.sh/resource-policy: keep
```

```bash
helmfile -f helm/helmfile.yaml apply \
  --selector name=ingress-nginx-internal
```

### 2.3 Configure Traefik for parallel validation

The Traefik repository and release are already declared in
`helm/helmfile.yaml`. For the parallel phase, temporarily adjust
`helm/values/traefik.values.yaml` as follows. Replace `.204` if that address is
already allocated.

```yaml
ingressClass:
  enabled: true
  isDefaultClass: false
  name: traefik

providers:
  kubernetesCRD:
    enabled: true

  kubernetesIngress:
    enabled: true
    publishedService:
      enabled: false

  kubernetesIngressNGINX:
    enabled: true
    controllerClass: k8s.io/ingress-nginx
    ingressClass: nginx-internal
    ingressClassByName: false
    watchIngressWithoutClass: false
    publishService:
      enabled: false

  kubernetesGateway:
    enabled: false

gateway:
  enabled: false

gatewayClass:
  enabled: false

service:
  enabled: true
  spec:
    type: LoadBalancer
    loadBalancerIP: "192.168.50.204"
  annotations:
    metallb.universe.tf/address-pool: pool
```

Keeping both Traefik status publishers disabled prevents Traefik and nginx from
continually overwriting `Ingress.status`. ingress-nginx remains the only status
writer while DNS still points to it.

Validate the exact pinned chart before touching the cluster:

```bash
helmfile -f helm/helmfile.yaml template --selector name=traefik
```

## 3. Deploy Traefik in parallel

The Helmfile does not currently set `createNamespace`, so create the namespace
if it does not exist:

```bash
kubectl create namespace traefik-system
```

If the namespace already exists, the command reports that and no change is
needed.

Deploy only Traefik:

```bash
helmfile -f helm/helmfile.yaml apply --selector name=traefik
```

Verify the rollout and temporary address:

```bash
kubectl rollout status deployment/traefik -n traefik-system
kubectl get pods -n traefik-system
kubectl get svc -n traefik-system traefik
kubectl logs -n traefik-system deployment/traefik --tail=100
```

The Traefik Service must show the temporary IP, not `.201`.

## 4. Validate every route through Traefik

Test Traefik directly without changing Pi-hole DNS:

```bash
TRAEFIK_TEST_IP=192.168.50.204

for host in \
  pihole.home \
  grafana.home \
  loki.home \
  longhorn.home \
  prometheus.home \
  homepage.core \
  jellyfin.home
do
  curl --resolve "${host}:80:${TRAEFIK_TEST_IP}" \
    --output /dev/null \
    --silent \
    --show-error \
    --write-out "${host} %{http_code}\n" \
    "http://${host}/"
done
```

Compare these responses with the same requests sent to the current controller
at `192.168.50.201`. The verified baseline for this cluster is:

| Host | Expected root response |
| --- | ---: |
| `grafana.home` | `302` |
| `loki.home` | `404` |
| `homepage.core` | `200` |
| `jellyfin.home` | `302` |
| `longhorn.home` | `401` |
| `pihole.home` | `403` |
| `prometheus.home` | `302` |

Investigate any difference from that baseline before proceeding. Also test
Jellyfin playback and Longhorn authentication because those routes use
NGINX-specific behavior during compatibility mode.

## 5. Transfer `192.168.50.201` to Traefik

This is the only interruption-prone phase. Existing Ingress resources still
use `nginx-internal`, so Traefik continues serving them through its compatibility
provider during the handoff.

1. Confirm the `nginx-internal` IngressClass now has the Helm keep annotation:

   ```bash
   kubectl get ingressclass nginx-internal -o yaml
   ```

2. Uninstall ingress-nginx to release `.201`:

   ```bash
   helmfile -f helm/helmfile.yaml destroy \
     --selector name=ingress-nginx-internal
   ```

3. Remove or comment the `ingress-nginx-internal` release block in
   `helm/helmfile.yaml` immediately. Otherwise a later unscoped Helmfile apply
   will reinstall it.

4. Change only `service.spec.loadBalancerIP` in
   `helm/values/traefik.values.yaml` from the temporary address to
   `192.168.50.201`, then apply Traefik:

   ```bash
   helmfile -f helm/helmfile.yaml apply --selector name=traefik
   ```

5. Wait for MetalLB and test the unchanged DNS names:

   ```bash
   kubectl get svc -n traefik-system traefik --watch
   dig +short pihole.home
   curl --fail --head http://grafana.home/
   ```

DNS remains `.201`, so ExternalDNS does not need to change during this IP
handoff.

## 6. Migrate the declarative Ingress classes once

This section replaces the old guide's duplicated class and ExternalDNS steps.

### 6.1 Allow ExternalDNS to observe both classes

Temporarily change `helm/values/externaldns.values.yaml` to:

```yaml
ingressClassFilters:
  - nginx-internal
  - traefik
```

Use the correct Helmfile release name:

```bash
helmfile -f helm/helmfile.yaml apply \
  --selector name=externaldns-pihole
```

### 6.2 Enable status publishing for native Traefik Ingresses

Change this setting in `helm/values/traefik.values.yaml`:

```yaml
providers:
  kubernetesIngress:
    publishedService:
      enabled: true
```

Keep `kubernetesIngressNGINX` enabled for now, but leave its
`publishService.enabled` value `false`.

```bash
helmfile -f helm/helmfile.yaml apply --selector name=traefik
```

### 6.3 Change every declarative owner

Change `nginx-internal` to `traefik` in these files:

- `helm/values/pihole.values.yaml`
- `helm/values/grafana.values.yaml`
- `helm/values/loki.values.yaml`
- `helm/values/longhorn.values.yaml`
- `helm/values/prometheus.values.yaml`
- `base/homepage/ingress.yaml`
- `base/jellyfin/ingress.yaml`

Remove nginx-only annotations when changing the class:

- Prometheus: remove `nginx.ingress.kubernetes.io/backend-protocol`.
- Jellyfin: remove the `nginx.ingress.kubernetes.io/*` annotations. Traefik does
  not impose nginx's body-size behavior, and its streaming behavior must be
  validated directly.
- Longhorn: do not remove nginx BasicAuth annotations until equivalent Traefik
  authentication exists. The compatibility provider translates them, but the
  standard Kubernetes Ingress provider does not.

The native Longhorn `Middleware` is declared in `base/longhorn`. Traefik expects
htpasswd entries in a Secret key named `users`, while the existing
`longhorn-system/basic-auth` Secret uses NGINX's `auth` key. Copy that value into
a Traefik-specific Secret without decoding or printing it:

```bash
kubectl get secret basic-auth -n longhorn-system -o json |
  jq '{
    apiVersion: "v1",
    kind: "Secret",
    metadata: {
      name: "longhorn-basic-auth",
      namespace: "longhorn-system",
      labels: {
        "app.kubernetes.io/name": "longhorn-basic-auth",
        "app.kubernetes.io/part-of": "homelab"
      }
    },
    type: "Opaque",
    data: {users: .data.auth}
  }' |
  kubectl apply -f -

kubectl apply -k base/longhorn
```

Then replace the nginx annotations in the Longhorn values with:

```yaml
ingress:
  annotations:
    traefik.ingress.kubernetes.io/router.middlewares: longhorn-system-longhorn-basic-auth@kubernetescrd
```

Do not commit the password hash unless it is encrypted with the repository's
secret-management workflow.

Apply the Helm-owned routes:

```bash
helmfile -f helm/helmfile.yaml apply --selector name=pihole
helmfile -f helm/helmfile.yaml apply --selector name=grafana
helmfile -f helm/helmfile.yaml apply --selector name=loki
helmfile -f helm/helmfile.yaml apply --selector name=longhorn
helmfile -f helm/helmfile.yaml apply --selector name=prometheus
```

Apply the Kustomize-owned routes:

```bash
kubectl apply -k base/homepage
kubectl apply -k base/jellyfin
```

Verify all seven classes and addresses:

```bash
kubectl get ingress -A \
  -o custom-columns='NAMESPACE:.metadata.namespace,NAME:.metadata.name,CLASS:.spec.ingressClassName,ADDRESS:.status.loadBalancer.ingress[*].ip'
```

Every row in the inventory must show class `traefik` and address
`192.168.50.201` before cleanup.

### 6.4 Finish ExternalDNS and provider cleanup

Change ExternalDNS to its final filter:

```yaml
ingressClassFilters:
  - traefik
```

First disable the compatibility provider while leaving Traefik non-default:

```yaml
ingressClass:
  enabled: true
  isDefaultClass: false
  name: traefik

providers:
  kubernetesIngress:
    enabled: true
    publishedService:
      enabled: true

  kubernetesIngressNGINX:
    enabled: false
```

Apply and verify:

```bash
helmfile -f helm/helmfile.yaml apply --selector name=traefik
helmfile -f helm/helmfile.yaml apply \
  --selector name=externaldns-pihole

kubectl delete ingressclass nginx-internal
kubectl get ingressclass
```

After the old default class is gone, set Traefik as the only default and apply
it once more:

```yaml
ingressClass:
  enabled: true
  isDefaultClass: true
  name: traefik
```

```bash
helmfile -f helm/helmfile.yaml apply --selector name=traefik
kubectl logs -n pihole-system \
  -l app.kubernetes.io/instance=externaldns-pihole \
  --tail=50
```

Only delete `nginx-internal` after no Ingress references it.

## 7. Rollback

Before the `.201` handoff, rollback is simply:

```bash
helmfile -f helm/helmfile.yaml destroy --selector name=traefik
```

DNS and ingress-nginx remain unchanged.

After the handoff but before native class cleanup:

1. Change Traefik back to the temporary IP and apply it.
2. Restore the ingress-nginx release block in `helm/helmfile.yaml`.
3. Apply `ingress-nginx-internal` and wait for it to claim `.201`.
4. Verify routes through `.201` before removing Traefik.

After native class cleanup, revert the seven declarative owners and
ExternalDNS filter in Git first, then apply them. Avoid ad-hoc cluster patches
that disagree with the repository.

## 8. TLS certificates and a local CA

Traefik terminates TLS and can act as an ACME client, but it is not a
certificate authority. It cannot create a trusted local root CA by itself.

There are three practical choices:

### Publicly trusted certificates with a real domain

Use a domain you own, such as `grafana.home.example.com`, and an ACME DNS-01
challenge. Pi-hole can resolve those names to the LAN address while the public
DNS provider hosts only the ACME challenge records. This provides certificates
trusted by browsers without opening ports 80 or 443 to the Internet.

This is usually the simplest client experience. Public CAs will not issue
certificates for private names such as `grafana.home`.

### Private CA with cert-manager

Run a private CA such as `step-ca`, or use a cert-manager CA Issuer if you are
prepared to manage PKI lifecycle yourself. cert-manager writes a normal TLS
Secret in each Ingress namespace and Traefik reads `spec.tls[].secretName`.

This separates certificate lifecycle from the ingress controller and scales
better if Traefik later has multiple replicas. Every client device must trust
the private root CA.

### Private ACME server used directly by Traefik

Traefik's `certificatesResolvers` can point `acme.caServer` at a private ACME
directory such as `step-ca`. Traefik then requests and renews the leaf
certificates. The ACME state must be persisted, and Traefik must trust the CA
certificate used by the private ACME endpoint.

The shape in chart `41.0.1` is:

```yaml
persistence:
  enabled: true
  storageClass: longhorn
  size: 128Mi

certificatesResolvers:
  homelab:
    acme:
      email: admin@example.com
      storage: /data/acme.json
      caServer: https://ca.example.internal/acme/acme/directory
      caCertificates:
        - /etc/step-ca/root_ca.crt
      httpChallenge:
        entryPoint: web
```

The CA root still has to be mounted into the Traefik pod and installed in every
client trust store. The example is not deployable until the private CA,
challenge routing, root distribution, and backup strategy are designed.

For this homelab, prefer either:

1. a real domain plus DNS-01 for publicly trusted certificates, or
2. `step-ca` plus cert-manager for a fully private PKI.

Do not use a collection of unrelated self-signed leaf certificates. They do
not provide manageable trust or renewal.

## References

- [Traefik ingress-nginx migration](https://doc.traefik.io/traefik/migrate/nginx-to-traefik/)
- [Traefik Kubernetes Ingress NGINX provider](https://doc.traefik.io/traefik/reference/install-configuration/providers/kubernetes/kubernetes-ingress-nginx/)
- [Traefik ACME certificate resolver](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/)
- [cert-manager CA Issuer](https://cert-manager.io/docs/configuration/ca/)
- [Smallstep private ACME integration](https://smallstep.com/docs/tutorials/acme-protocol-acme-clients/)
