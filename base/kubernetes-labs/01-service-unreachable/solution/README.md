# Solution: Service selector mismatch

The Service selects `app.kubernetes.io/name: lab-backend`, while the Deployment's Pods
have `app.kubernetes.io/name: lab-api`. No Pods match, so the Service has no usable backend
endpoints. The process is healthy, and Deployment port-forwarding bypasses this mismatch.

In `../manifests/service.yaml`, set:

```yaml
spec:
  selector:
    app.kubernetes.io/name: lab-api
```

Apply your edited base, or apply this overlay to perform the repair without editing the base:

```bash
kubectl apply -k base/kubernetes-labs/01-service-unreachable/solution
```

The EndpointSlice controller should then publish the Ready API Pod's IP and port 8080.
Service `port: 80` is the client-facing port. Its `targetPort: http` resolves to the named
container port 8080. These port numbers do not need to be equal.

Verify through the actual Service path:

```bash
kubectl -n lab-service delete job lab-check --ignore-not-found
kubectl apply -f base/kubernetes-labs/01-service-unreachable/checks/job.yaml
kubectl -n lab-service wait --for=condition=complete job/lab-check --timeout=120s
kubectl -n lab-service logs job/lab-check
kubectl -n lab-service get endpointslices -l kubernetes.io/service-name=lab-api -o yaml
```

Expected: a completed Job, `PASS` in its logs, and a Ready backend on port 8080.

Readiness controls whether a selected Pod normally receives Service traffic. It cannot
compensate for a selector that never selects that Pod. Restarting a healthy Pod would not
repair this incident either.

The normal request path is client Pod → Service ClusterIP:80 → Cilium service forwarding
→ selected Ready Pod:8080. This exercise excludes ingress and DNS to isolate Service routing.

Use the main [exercise instructions](../README.md#reset-and-cleanup) to reset or clean up.
