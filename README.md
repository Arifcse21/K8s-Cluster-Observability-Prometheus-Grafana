# Kubernetes Cluster Monitoring Setup
> Prometheus + Grafana + OpenEBS Hostpath + Traefik + cert-manager
> Tested on: kubeadm v1.30, 3-node KVM cluster (1 master + 2 workers)

---

## Prerequisites

- kubeadm cluster running (`kubectl get nodes` shows all Ready)
- Helm installed
- Workers running before installing any storage/monitoring stack

```bash
# Verify cluster health
kubectl get nodes
kubectl auth can-i '*' '*'   # should return: yes
```

If `auth can-i` returns `no`, fix RBAC first:
```bash
# Check cert group
grep client-certificate-data ~/.kube/config \
  | awk '{print $2}' | base64 -d \
  | openssl x509 -noout -subject
# If O=kubeadm:cluster-admins (not system:masters), bind manually:
kubectl create clusterrolebinding kubeadm-cluster-admins-binding \
  --clusterrole=cluster-admin \
  --group="kubeadm:cluster-admins"
```

---

## Step 1 — Add Helm Repos

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add openebs              https://openebs.github.io/openebs
helm repo add traefik              https://traefik.github.io/charts
helm repo add jetstack             https://charts.jetstack.io
helm repo update
```

---

## Step 2 — Install cert-manager

```bash
helm install cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --create-namespace \
  --set crds.enabled=true \
  --wait --timeout 5m

# Verify
kubectl get pods -n cert-manager
kubectl get crd | grep cert-manager
```

### Apply ClusterIssuer

```bash
# clusterissuer.yaml
cat <<EOF | kubectl apply -f -
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: selfsigned-issuer
spec:
  selfSigned: {}
EOF
```

---

## Step 3 — Install OpenEBS (Hostpath only)

```bash
helm install openebs openebs/openebs \
  --namespace openebs \
  --create-namespace \
  --set engines.replicated.mayastor.enabled=false \
  --set engines.local.lvm.enabled=false \
  --set engines.local.zfs.enabled=false \
  --wait --timeout 5m

# Verify StorageClass created
kubectl get storageclass
# Should show: openebs-hostpath
```

> **Note:** OpenEBS DaemonSet deploys to all nodes automatically — no per-node setup needed.

---

## Step 4 — Install Traefik

```bash
helm install traefik traefik/traefik \
  --namespace traefik \
  --create-namespace \
  --wait --timeout 5m
```

### Configure HostPort + Pin to Master Node

This ensures Traefik always runs on the master (known IP) so `/etc/hosts` entry stays stable:

```bash
cat <<EOF | helm upgrade traefik traefik/traefik \
  --namespace traefik -f -
deployment:
  replicas: 1
ports:
  web:
    hostPort: 80
  websecure:
    hostPort: 443
hostNetwork: false
service:
  type: ClusterIP
nodeSelector:
  node-role.kubernetes.io/control-plane: ""
tolerations:
  - key: node-role.kubernetes.io/control-plane
    operator: Exists
    effect: NoSchedule
EOF
```

### Verify

```bash
kubectl get pods -n traefik -o wide
# Should show 1 pod on k8s-master

ss -tlnp | grep -E ':80|:443'
# Should show ports bound on master
```

---

## Step 5 — Create Namespace & Secrets

```bash
kubectl create namespace monitoring
```

### Grafana Admin Credentials Secret

```bash
# secrets.yaml
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Secret
metadata:
  name: grafana-admin-credentials
  namespace: monitoring
type: Opaque
stringData:
  admin-user: admin
  admin-password: YOUR_SECURE_PASSWORD_HERE
EOF
```

---

## Step 6 — Install kube-prometheus-stack

### Values File

Save as `kube-prometheus-stack-custom-values.yaml`:

```yaml
alertmanager:
  alertmanagerSpec:
    retention: 120h
    resources:
      requests:
        cpu: 50m
        memory: 64Mi
      limits:
        cpu: 200m
        memory: 128Mi
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: openebs-hostpath
          resources:
            requests:
              storage: 5Gi
  ingress:
    enabled: false

prometheus:
  prometheusSpec:
    retention: 15d
    storageSpec:
      volumeClaimTemplate:
        spec:
          storageClassName: openebs-hostpath
          resources:
            requests:
              storage: 5Gi
    resources:
      requests:
        cpu: 200m
        memory: 512Mi
      limits:
        cpu: 1000m
        memory: 1Gi
  ingress:
    enabled: false

grafana:
  admin:
    existingSecret: "grafana-admin-credentials"
    userKey: "admin-user"
    passwordKey: "admin-password"
  grafana.ini:
    plugins:
      enable_alpha: false
  plugins: []
  livenessProbe:
    initialDelaySeconds: 120
    timeoutSeconds: 30
    failureThreshold: 15
  readinessProbe:
    initialDelaySeconds: 60
  securityContext:
    runAsUser: 472
    runAsGroup: 472
    fsGroup: 472
  initChownData:
    enabled: false
  ingress:
    enabled: true
    ingressClassName: traefik
    annotations:
      cert-manager.io/cluster-issuer: selfsigned-issuer
    hosts:
      - grafana.k8s.local
    tls:
      - secretName: grafana-tls
        hosts:
          - grafana.k8s.local
  persistence:
    enabled: true
    storageClassName: openebs-hostpath
    size: 5Gi
  resources:
    requests:
      cpu: 100m
      memory: 256Mi
    limits:
      cpu: 500m
      memory: 512Mi

defaultRules:
  create: true

kubeApiServer:
  enabled: true
kubelet:
  enabled: true
kubeProxy:
  enabled: true
kubeScheduler:
  enabled: true
nodeExporter:
  enabled: true
```

### Install

```bash
helm install prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f kube-prometheus-stack-custom-values.yaml \
  --timeout 10m

# Watch rollout
kubectl get pods -n monitoring -w
```

### Upgrade (after config changes)

```bash
helm upgrade prometheus prometheus-community/kube-prometheus-stack \
  --namespace monitoring \
  -f kube-prometheus-stack-custom-values.yaml \
  --timeout 10m
```

---

## Step 7 — Host Machine Access

Add to `/etc/hosts` on your **local machine** (not k8s master):

```bash
echo "192.168.100.10  grafana.k8s.local" | sudo tee -a /etc/hosts
```

Then open browser: **https://grafana.k8s.local**

> Accept the self-signed cert warning. Login with credentials from your secret.

### Quick Access via Port-Forward (no DNS needed)

```bash
kubectl port-forward -n monitoring svc/prometheus-grafana 3000:80 --address=0.0.0.0
# Access: http://192.168.100.10:3000
```

---

## Verification Commands

```bash
# All monitoring pods
kubectl get pods -n monitoring -o wide

# PVCs bound
kubectl get pvc -n monitoring

# Ingress
kubectl get ingress -n monitoring

# Traefik routing
kubectl get svc -n traefik

# Grafana logs
kubectl logs -n monitoring deployment/prometheus-grafana -c grafana --tail=50

# Cert issued
kubectl get certificate -n monitoring
```

---

## Grafana — Finding Node Dashboards

1. Left sidebar → **Dashboards**
2. Search for:
   - `Node Exporter Full` — raw hardware metrics (CPU, RAM, disk, network per node)
   - `Kubernetes / Nodes` — kubelet-level node metrics
   - `Kubernetes / Compute Resources / Node (Pods)` — per-node pod resource usage
   - `USE Method / Node` — Utilization, Saturation, Errors

---

## Troubleshooting

### RBAC Forbidden errors
```bash
kubectl create clusterrolebinding kubeadm-cluster-admins-binding \
  --clusterrole=cluster-admin \
  --group="kubeadm:cluster-admins"
```

### Namespace stuck in Terminating
```bash
kubectl get namespace <ns> -o json \
  | python3 -c "
import json,sys
ns=json.load(sys.stdin)
ns['spec']['finalizers']=[]
print(json.dumps(ns))
" | kubectl replace --raw /api/v1/namespaces/<ns>/finalize -f -
```

### Webhook blocking Longhorn/resource deletion
```bash
kubectl delete mutatingwebhookconfiguration longhorn-webhook-mutator
kubectl delete validatingwebhookconfiguration longhorn-webhook-validator
```

### Force delete stuck CRDs
```bash
for crd in $(kubectl get crd | grep <name> | awk '{print $1}'); do
  kubectl patch crd $crd -p '{"metadata":{"finalizers":[]}}' --type=merge
  kubectl delete crd $crd --force --grace-period=0
done
```

### Grafana init-chown-data CrashLoop
```bash
# Undo last rollout restart
kubectl rollout undo deployment prometheus-grafana -n monitoring
# The initChownData: enabled: false in values prevents this on fresh installs
```

### Helm TLS handshake timeout
```bash
# Usually RBAC issue — verify
kubectl auth can-i '*' '*'
# Fix with kubeadm group binding above
```

---

## Resource Summary

| Component | CPU Request | RAM Request | Storage |
|---|---|---|---|
| Prometheus | 200m | 512Mi | 5Gi |
| Grafana | 100m | 256Mi | 5Gi |
| Alertmanager | 50m | 64Mi | 5Gi |
| node-exporter (×3) | ~30m | ~30Mi | — |
| kube-state-metrics | ~10m | ~50Mi | — |
| OpenEBS | ~10m | ~15Mi | — |
| Traefik | ~50m | ~50Mi | — |
| cert-manager | ~20m | ~50Mi | — |
| **Total** | **~600m** | **~1.2Gi** | **15Gi** |

> Designed for 3-node cluster: 1 master + 2 workers, 2 CPU / 4GB RAM each.
> All monitoring workloads schedule on workers, leaving master for control plane.