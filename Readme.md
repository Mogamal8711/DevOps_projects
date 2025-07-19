# 🧪 Kubernetes TODO App Deployment (NFS + Ingress + MetalLB)

This guide walks you through deploying a full-stack TODO application with Apache, MySQL, Ingress, and NFS-based dynamic storage in Kubernetes.

## 🛠️ Prerequisites

- Kubernetes cluster (Minikube or kubeadm)
- `kubectl`, `helm` installed
- NFS server available
- Host and nodes on same LAN or bridged networking

---

## 🚀 Full Deployment Steps (Consolidated)

### 1. Install NFS Server on NFS Host

```bash
sudo apt update && sudo apt install nfs-kernel-server -y
sudo mkdir -p /srv/nfs/kubedata
sudo chmod 777 /srv/nfs/kubedata

echo "/srv/nfs/kubedata *(rw,sync,no_subtree_check,no_root_squash)" | sudo tee -a /etc/exports
sudo exportfs -rav
sudo systemctl restart nfs-kernel-server
```

### 2. Install NFS Client on All Kubernetes Nodes

```bash
sudo apt install nfs-common -y
```

### 3. Deploy NFS Subdir External Provisioner

```bash
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-client nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --set nfs.server=<NFS_SERVER_IP> \
  --set nfs.path=/srv/nfs/kubedata \
  --set storageClass.name=nfs-client
```

### 4. Deploy Ingress-NGINX Controller

```bash
helm upgrade --install ingress-nginx ingress-nginx \
  --repo https://kubernetes.github.io/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace
```

### 5. Enable strictARP in kube-proxy (Required for MetalLB)

```bash
kubectl edit configmap -n kube-system kube-proxy
```

Update the config to include:

```yaml
strictARP: true
```

### 6. Deploy MetalLB

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.13.10/config/manifests/metallb-frr.yaml
```

Wait for all MetalLB pods to be ready:

```bash
kubectl get pods -n metallb-system
```

Then apply your MetalLB IP Pool config `Full_metallb.yaml`:

```yaml
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata:
  name: my-pool
  namespace: metallb-system
spec:
  addresses:
    - 192.168.1.200-192.168.1.210
```

```bash
kubectl apply -f Full_metallb.yaml
kubectl rollout restart deployment controller -n metallb-system
```

### 7. Add Host Entry for Ingress Testing

Edit `/etc/hosts` on your local machine:

```text
192.168.1.200   todoapp.example.internal
```

### 8. Apply Secrets and ConfigMap

```bash
kubectl apply -f Full_Secrts.yaml
kubectl apply -f Full_ConfigMap.yaml
```

### 9. Deploy App (Apache + PHP)

```bash
kubectl apply -f Full_deplyment.yaml
```

### 10. Deploy App Service (NodePort)

Ensure `Full_service.yaml` defines port 80:

```yaml
ports:
  - port: 80
    targetPort: 80
    nodePort: 31000
```

Then:

```bash
kubectl apply -f Full_service.yaml
```

### 11. Deploy MySQL Database

Create DB init config:

```bash
kubectl create configmap mysql-init-config --from-file=init.sql
```

Create PVC:

```bash
kubectl apply -f Full_todo-app-pvc.yaml
```

Deploy DB:

```bash
kubectl apply -f Full_deplyment_db.yaml
kubectl apply -f Full_service_db.yaml
```

### 12. Deploy Ingress Resource

Ensure Ingress backend service port is **80**, not 8080:

```yaml
rules:
- host: todoapp.example.internal
  http:
    paths:
    - path: /
      pathType: Prefix
      backend:
        service:
          name: todo-app-service
          port:
            number: 80
```

Then:

```bash
kubectl apply -f Full_ingress.yaml
```

---

## ✅ Access the App

### Option A: Access via Ingress Domain (Preferred)

```
http://todoapp.example.internal
```

Ensure the IP 192.168.1.200 is reachable.

### Option B: Use Port Forwarding

```bash
kubectl port-forward svc/todo-app-service 8080:80
```

Access via:

```
http://localhost:8080
```

### Option C: Use NodePort

Find Minikube or VM IP:

```bash
minikube ip
```

Then open:

```
http://<NODE-IP>:31000
```

---

## 🔍 Troubleshooting

| Issue                    | Fix                                         |
| ------------------------ | ------------------------------------------- |
| Cannot connect to domain | Check `/etc/hosts` and Ingress IP           |
| Service has no endpoints | Check pod labels and service selectors      |
| curl timeout             | Verify MetalLB IP is reachable              |
| Ingress returns 404      | Check Ingress backend port and path         |
| Apache error in logs     | Add `ServerName localhost` in Apache config |

---

## 📓 Notes

- Works with Minikube or kubeadm using bridged networking
- NFS used for persistent storage via dynamic provisioning
- Ingress + MetalLB provides load-balanced HTTP access

---

Built and tested July 2025

