# Exercise 01: healthy application, unreachable Service

**Incident:** A small HTTP API was deployed. Its Pod becomes Ready and direct access works,
but an in-cluster client cannot reach it through its Service. Find the cause and repair the
configuration without changing the application code.

Allow about 30–45 minutes. This exercise covers Deployments, Pod labels, Services,
EndpointSlices, named ports, and the difference between application health and connectivity.
There is one deliberate fault. The API returns JSON rather than using a database.

## Environment and scope

- Use your existing Kubernetes cluster with kubectl and Cilium enforcing NetworkPolicies.
- Everything is contained in the dedicated `lab-service` namespace. Reserve this name for this lab.
- No ingress, LAN DNS records, LoadBalancer address, persistent storage, or Helm changes are needed.
- The two containers request a combined 50m CPU / 64Mi memory and have combined limits of 400m CPU / 128Mi memory.
- Containers run without root, with read-only root filesystems and no Kubernetes API token.
- Both images must be downloadable from Docker Hub. Image pull failures are setup issues, not the intended incident.

## 1. Deploy

Check that kubectl targets your homelab, then apply the exercise:

```bash
kubectl config current-context
kubectl apply -k base/kubernetes-labs/01-service-unreachable
kubectl -n lab-service rollout status deployment/lab-api --timeout=120s
kubectl -n lab-service get pods,services
```

The Deployment should become available with one Ready Pod. If it does not, inspect
`kubectl -n lab-service describe pods` and resolve image pulls or scheduling before continuing.

## 2. Prove that the application works

In one terminal:

```bash
kubectl -n lab-service port-forward deployment/lab-api 18080:8080
```

In another terminal:

```bash
curl --fail http://127.0.0.1:18080/
```

Expected JSON:

```json
{"status": "ok", "application": "lab-api"}
```

Keep the port-forward running while testing, then stop it with Ctrl-C.
This path bypasses the Service and does not prove that normal Pod-to-Pod networking works.

## 3. Reproduce the incident

Start a finite client Job that makes requests through the Service's ClusterIP:

```bash
kubectl apply -f base/kubernetes-labs/01-service-unreachable/checks/job.yaml
kubectl -n lab-service wait --for=condition=Ready pod -l job-name=lab-check --timeout=90s
kubectl -n lab-service logs -f job/lab-check
kubectl -n lab-service get jobs
```

Expected: five unsuccessful attempts followed by `FAIL`. The exact network error may vary.
The Job exits with a failure status; this is intentional. It can take around 25 seconds after startup.
The client uses Kubernetes-injected Service IP environment variables, so this check does not
rely on DNS. NetworkPolicies permit this client to reach the API's TCP port 8080.

If the Ready wait misses a fast-completing Job, inspect its logs and Job status instead.
If the Pod never starts, inspect its events; that is a setup issue.

## 4. Investigate and repair

Before changing anything, write down:

1. What evidence shows the API process is healthy?
2. Which Pods should this Service send traffic to, and how does Kubernetes choose them?
3. What addresses and ports does Kubernetes currently publish for the Service?
4. Which manifest explains the difference between what you expect and what you observe?

Useful tools include `kubectl get`, `kubectl describe`, `kubectl logs`, and YAML output.
Investigate the `lab-service` namespace. Edit the faulty manifest in this directory,
then apply the lab again. Keep the ClusterIP Service, existing Pod labels, and isolation policies.
Do not solve the incident by using port-forwarding as the client path.

[Progressive hints](HINTS.md) · [Solution and explanation](solution/README.md)

## 5. Verify your repair

A completed or failed Job must be recreated to run again:

```bash
kubectl -n lab-service delete job lab-check --ignore-not-found
kubectl apply -f base/kubernetes-labs/01-service-unreachable/checks/job.yaml
kubectl -n lab-service wait --for=condition=complete job/lab-check --timeout=120s
kubectl -n lab-service logs job/lab-check
kubectl -n lab-service get endpointslices -l kubernetes.io/service-name=lab-api -o yaml
```

Success means the Job completes and prints `PASS`, the Service has a Ready endpoint
for the API Pod on port 8080, and you can explain why direct access worked before your fix.
A failed Job makes the completion wait time out; read its logs to investigate.

## Reset and cleanup

To replay the original fault, restore the `app.kubernetes.io/name` selector value in
`manifests/service.yaml` to `lab-backend` if you edited it, then run:

```bash
kubectl -n lab-service delete job lab-check --ignore-not-found
kubectl apply -k base/kubernetes-labs/01-service-unreachable
```

This also resets the Service after applying the supplied solution overlay, provided the
base file still has its original selector. Re-run the client from step 3.

To remove all exercise resources, including its namespace:

```bash
kubectl delete namespace lab-service
```

Only use this namespace for the exercise; cleanup deletes everything inside it.

## Debrief

Write a short incident note: symptom, evidence, root cause, repair, and verification.
Explain the roles of Service `port`, `targetPort`, Pod labels, and readiness.

Reference: [Kubernetes Service debugging](https://kubernetes.io/docs/tasks/debug/debug-application/debug-service/).
Your cluster uses Cilium rather than kube-proxy; kube-proxy-specific sections do not apply.
