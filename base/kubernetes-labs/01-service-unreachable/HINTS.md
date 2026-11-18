# Progressive hints

Read one hint at a time, then return to the investigation.

<details>
<summary>Hint 1: compare the two request paths</summary>

Deployment port-forwarding reaches a Pod without going through the Service ClusterIP.
Inspect the Service and the backend information Kubernetes has generated for it:

```bash
kubectl -n lab-service describe service lab-api
kubectl -n lab-service get endpointslices -l kubernetes.io/service-name=lab-api -o yaml
```

Are there any Ready backend addresses?

</details>

<details>
<summary>Hint 2: inspect how the Service finds its Pods</summary>

```bash
kubectl -n lab-service get service lab-api -o yaml
kubectl -n lab-service get pods --show-labels
```

Compare `spec.selector` on the Service with the labels on the running API Pod.
A Service's own metadata labels do not select its backend Pods.

</details>

<details>
<summary>Hint 3: make the smallest declarative repair</summary>

The Service selects a label value that the API Pod does not have.
Fix the Service selector in `manifests/service.yaml`, apply the lab again, and rerun the client Job.
The Service port 80 correctly targets the container's named port `http`, which is 8080.

</details>
