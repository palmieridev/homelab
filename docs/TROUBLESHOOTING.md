# Homelab Troubleshooting Manual

## Table of Contents
1. [Cluster Health](#cluster-health)
2. [Cilium CNI](#cilium-cni)
3. [MetalLB Load Balancer](#metallb-load-balancer)
4. [Longhorn Storage](#longhorn-storage)
5. [Pihole DNS](#pihole-dns)
6. [Nginx Ingress](#nginx-ingress)
7. [Common Issues](#common-issues)
8. [Monitoring Commands](#monitoring-commands)

---

## Cluster Health

### Basic Cluster Status
```bash
# Check node status and resources
kubectl get nodes -o wide
kubectl describe nodes

# Check cluster-wide pod status
kubectl get pods -A
kubectl get pods -A --field-selector='status.phase!=Running'

# Check system components
kubectl get componentstatus
kubectl cluster-info
```

### Node Resource Usage
```bash
# Check node resource consumption
kubectl top nodes
kubectl top pods -A --sort-by=cpu
kubectl top pods -A --sort-by=memory

# Check disk usage on nodes
kubectl debug node/<node-name> -it --image=nicolaka/netshoot -- df -h
```

### Events and Logs
```bash
# Cluster-wide events
kubectl get events -A --sort-by='.lastTimestamp' | tail -20

# Kubelet logs on specific node
kubectl debug node/<node-name> -it --image=nicolaka/netshoot -- journalctl -u kubelet -f
```

---

## Cilium CNI

### Status and Health
```bash
# Overall Cilium status
cilium-cli status

# Detailed connectivity test
cilium-cli connectivity test

# Check Cilium pods
kubectl get pods -n kube-system -l k8s-app=cilium
kubectl logs -n kube-system -l k8s-app=cilium --tail=50
```

### Network Troubleshooting
```bash
# Check Cilium endpoints
kubectl get ciliumnodes
kubectl get ciliumendpoints -A

# Network policies
kubectl get networkpolicies -A
cilium-cli config view

# Pod-to-pod connectivity
kubectl exec -it <pod-name> -- ping <target-ip>
```

### Common Cilium Issues
```bash
# Restart Cilium (if needed)
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout restart deployment/cilium-operator -n kube-system

# Check for CNI errors
kubectl describe pod <failing-pod>
kubectl logs -n kube-system -l k8s-app=cilium | grep -i error
```

---

## MetalLB Load Balancer

### Status Checks
```bash
# Check MetalLB components
kubectl get pods -n metallb-system
kubectl get ipaddresspool -n metallb-system
kubectl get l2advertisement -n metallb-system

# Check LoadBalancer services
kubectl get svc -A --field-selector='spec.type=LoadBalancer'
```

### Speaker Logs and Debugging
```bash
# Check speaker logs on all nodes
kubectl logs -n metallb-system -l component=speaker --tail=50

# Check specific speaker
kubectl logs -n metallb-system speaker-<pod-suffix>

# Check for IP announcements
kubectl logs -n metallb-system -l component=speaker | grep -E "(announce|error|warn)"
```

### IP Pool Management
```bash
# View current IP pool configuration
kubectl get ipaddresspool pool -n metallb-system -o yaml

# Check IP assignments
kubectl get svc -n pihole-system -o wide
kubectl describe svc <service-name> -n <namespace>

# Test connectivity to LoadBalancer IPs
ping <loadbalancer-ip>
curl -v http://<loadbalancer-ip>
```

### Common MetalLB Issues

#### Issue: Service stuck in `<pending>`
```bash
# Check events
kubectl describe svc <service-name>

# Verify IP pool has available IPs
kubectl get ipaddresspool -n metallb-system -o yaml

# Check speaker logs for errors
kubectl logs -n metallb-system -l component=speaker | grep -i error
```

#### Issue: External IP not reachable
```bash
# Check L2Advertisement configuration
kubectl get l2advertisement -n metallb-system -o yaml

# Verify speaker pods on correct nodes
kubectl get pods -n metallb-system -o wide

# Check ARP responses (from external machine)
arping <loadbalancer-ip>
```

---

## Longhorn Storage

### Cluster Status
```bash
# Check Longhorn system pods
kubectl get pods -n longhorn-system
kubectl get nodes,pv,pvc

# Longhorn node status
kubectl get nodes.longhorn.io -n longhorn-system
```

### Storage Health
```bash
# Check volumes and replicas
kubectl get volumes.longhorn.io -n longhorn-system
kubectl get replicas.longhorn.io -n longhorn-system

# Check storage classes
kubectl get storageclass

# Volume status
kubectl describe pvc <pvc-name>
```

### Performance and Issues
```bash
# Check engine logs
kubectl logs -n longhorn-system <instance-manager-pod>

# CSI driver logs
kubectl logs -n longhorn-system -l app=longhorn-csi-plugin

# Storage usage
kubectl exec -it <pod-using-longhorn-volume> -- df -h
```

### Common Longhorn Issues

#### Issue: Volume attachment fails
```bash
# Check node conditions
kubectl get nodes.longhorn.io -n longhorn-system -o wide

# Check engine manager
kubectl describe instances.longhorn.io -n longhorn-system

# Restart Longhorn manager if needed
kubectl rollout restart daemonset/longhorn-manager -n longhorn-system
```

---

## Pihole DNS

### Service Status
```bash
# Check Pihole pod and services
kubectl get pods,svc -n pihole-system
kubectl describe svc pihole-web -n pihole-system

# Check LoadBalancer IP assignment
kubectl get svc -n pihole-system -o wide
```

### DNS Functionality
```bash
# Test DNS resolution
nslookup google.com 192.168.50.235
dig @192.168.50.235 example.com

# Check from within cluster
kubectl run test-dns --rm -it --image=busybox -- nslookup google.com 192.168.50.235
```

### Web Interface Access
```bash
# Test HTTP access
curl -v http://192.168.50.235/admin
curl -L http://192.168.50.235/admin | head -20

# Check service endpoints
kubectl get endpoints -n pihole-system
```

### Pihole Logs
```bash
# Application logs
kubectl logs -n pihole-system -l app=pihole

# Configuration check
kubectl exec -it -n pihole-system <pihole-pod> -- cat /etc/pihole/setupVars.conf
```

### Common Pihole Issues

#### Issue: Web interface not accessible
```bash
# Check service configuration
kubectl describe svc pihole-web -n pihole-system

# Verify MetalLB announcement
kubectl logs -n metallb-system -l component=speaker | grep "192.168.50.235"

# Test from pod network
kubectl run test-pod --rm -it --image=curl:latest -- curl http://pihole-web.pihole-system.svc.cluster.local/admin
```

#### Issue: DNS not working
```bash
# Check DNS service status
kubectl get svc pihole-dns-tcp pihole-dns-udp -n pihole-system

# Test DNS ports
nc -zv 192.168.50.235 53
kubectl run test-dns --rm -it --image=busybox -- nc -zv 192.168.50.235 53
```

---

## Nginx Ingress

### Controller Status
```bash
# Check ingress controller
kubectl get pods,svc -n nginx-system
kubectl logs -n nginx-system -l app.kubernetes.io/name=ingress-nginx

# Check ingress resources
kubectl get ingress -A
```

### Configuration and Routing
```bash
# Check nginx configuration
kubectl exec -it -n nginx-system <nginx-controller-pod> -- cat /etc/nginx/nginx.conf

# Test ingress connectivity
curl -v http://192.168.50.201
curl -H "Host: your-domain.com" http://192.168.50.201
```

### Common Nginx Issues

#### Issue: 404 or routing problems
```bash
# Check ingress rules
kubectl describe ingress <ingress-name>

# Check service endpoints
kubectl get endpoints <backend-service-name>

# Nginx controller logs
kubectl logs -n nginx-system -l app.kubernetes.io/name=ingress-nginx --tail=100
```

---

## Common Issues

### Pod Stuck in Pending
```bash
# Check node resources
kubectl describe pod <pod-name>
kubectl top nodes

# Check taints and tolerations
kubectl describe nodes | grep -A5 Taints
```

### Network Connectivity Issues
```bash
# Test pod-to-pod communication
kubectl exec -it <pod1> -- ping <pod2-ip>

# Check DNS resolution
kubectl exec -it <pod> -- nslookup kubernetes.default.svc.cluster.local

# Verify CNI
kubectl get pods -n kube-system -l k8s-app=cilium
```

### Storage Issues
```bash
# Check PVC status
kubectl get pvc -A
kubectl describe pvc <pvc-name>

# Check storage class
kubectl get storageclass
kubectl describe storageclass longhorn
```

---

## Monitoring Commands

### Real-time Monitoring
```bash
# Watch pods across all namespaces
watch kubectl get pods -A

# Monitor specific namespace
watch kubectl get pods,svc,pvc -n <namespace>

# Resource usage monitoring
watch kubectl top nodes
watch kubectl top pods -A
```

### Event Monitoring
```bash
# Follow events in real-time
kubectl get events -A --watch

# Filter events by type
kubectl get events -A --field-selector type=Warning
kubectl get events -A --field-selector involvedObject.kind=Pod
```

### Log Monitoring
```bash
# Follow logs from multiple pods
kubectl logs -f -l app=<app-name> --all-containers=true

# Stream logs from multiple namespaces
kubectl logs -f -l component=speaker -n metallb-system
```

### Health Check Scripts
```bash
#!/bin/bash
# Quick health check script

echo "=== Cluster Nodes ==="
kubectl get nodes

echo "=== System Pods ==="
kubectl get pods -n kube-system | grep -v Running || echo "All system pods running"

echo "=== MetalLB Status ==="
kubectl get pods -n metallb-system
kubectl get svc -A --field-selector='spec.type=LoadBalancer' -o custom-columns=NAMESPACE:.metadata.namespace,NAME:.metadata.name,EXTERNAL-IP:.status.loadBalancer.ingress[0].ip

echo "=== Cilium Status ==="
cilium-cli status --brief

echo "=== Storage Status ==="
kubectl get pvc -A | grep -v Bound || echo "All PVCs bound"

echo "=== Pihole Status ==="
curl -s -o /dev/null -w "%{http_code}" http://192.168.50.235/admin && echo " - Pihole web accessible" || echo " - Pihole web not accessible"
```

---

## Emergency Recovery

### Restart Core Services
```bash
# Restart Cilium
kubectl rollout restart daemonset/cilium -n kube-system
kubectl rollout restart deployment/cilium-operator -n kube-system

# Restart MetalLB
kubectl rollout restart daemonset/speaker -n metallb-system
kubectl rollout restart deployment/controller -n metallb-system

# Restart Ingress
kubectl rollout restart deployment/ingress-nginx-internal-controller -n nginx-system
```

### Force Pod Recreation
```bash
# Delete and recreate stuck pods
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force

# Scale down and up deployments
kubectl scale deployment <deployment-name> --replicas=0 -n <namespace>
kubectl scale deployment <deployment-name> --replicas=1 -n <namespace>
```

---

**Note**: Replace `<placeholders>` with actual values from your environment. Keep this manual updated as your infrastructure evolves.