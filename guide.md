Perfekt—ab hier die **fortgeführte Anleitung** (aktueller Stand: **3× CP Ready, kube-vip\@192.168.100.100 läuft**). Fokus wie gewünscht: **kubectl auf deinem PC**, dann **Worker** und **Longhorn (einfach: Verzeichnis auf SSD)**. Zum Schluss **MetalLB** + **Ingress** und kurze Smoke-Tests.

---

# 1) kubectl auf deinem Admin-PC

**Install:**

* Arch/EndeavourOS: `sudo pacman -S kubectl`
* Ubuntu/Debian: `sudo apt install -y kubectl` (oder `snap install kubectl --classic`)

**Kubeconfig holen & auf VIP/FQDN umstellen:**

```bash
scp faba@cp-1:/etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config
# VIP/FQDN eintragen (eine von beiden Varianten):
sed -i 's#server: https://.*:6443#server: https://192.168.100.100:6443#' ~/.kube/config
# sed -i 's#server: https://.*:6443#server: https://api.deine-domain.tld:6443#' ~/.kube/config
kubectl get nodes
```

---

# 2) Worker-Nodes vorbereiten (Ubuntu 24.04 auf jedem Worker)

Auf **jedem Worker (wk-1..3)**:

```bash
# Basis
sudo apt update && sudo apt full-upgrade -y
sudo timedatectl set-timezone Europe/Berlin

# Swap aus
sudo swapoff -a
sudo sed -i 's/^\([^#].*swap\)/#\1/' /etc/fstab

# Kernel-Module & Sysctls
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter
sudo tee /etc/sysctl.d/99-kubernetes.conf >/dev/null <<'EOF'
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1
# bleibt harmlos, hilft später bei Multi-VLAN:
net.ipv4.conf.all.rp_filter=2
net.ipv4.conf.default.rp_filter=2
EOF
sudo sysctl --system

# Longhorn-Prereqs
sudo apt install -y open-iscsi nfs-common smartmontools
sudo systemctl enable --now iscsid
```

---

# 3) Worker dem Cluster joinen

Token von `cp-1` holen:

```bash
# auf cp-1:
sudo cat /var/lib/rancher/k3s/server/node-token
```

Auf **jedem Worker** (Token einsetzen):

```bash
curl -sfL https://get.k3s.io | \
  K3S_TOKEN="<KOMPLETTER_NODE_TOKEN>" \
  INSTALL_K3S_EXEC="agent --server https://192.168.100.100:6443" \
  sh -
```

Check:

```bash
kubectl get nodes -o wide
```

---

# 4) Longhorn installieren (einfach: Verzeichnis auf SSD, **keine** Extra-Partition/Loop)

> Longhorn nutzt per Default `/var/lib/longhorn` auf jedem **Worker**. Stelle nur sicher, dass **genug Platz** auf dem Root-FS ist.

**Control-Planes für Workloads sperren (falls noch nicht):**

```bash
kubectl taint nodes cp-1 node-role.kubernetes.io/control-plane=:NoSchedule --overwrite
kubectl taint nodes cp-2 node-role.kubernetes.io/control-plane=:NoSchedule --overwrite
kubectl taint nodes cp-3 node-role.kubernetes.io/control-plane=:NoSchedule --overwrite
```

**Worker für Storage labeln:**

```bash
for n in wk-1 wk-2 wk-3; do
  kubectl label nodes $n node-role.kubernetes.io/worker= longhorn=storage --overwrite
done
```

**Helm-Install:**

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn -n longhorn-system --create-namespace
```

**StorageClass (2 Repliken, Default):**

```bash
kubectl apply -f - <<'YAML'
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: longhorn-2r
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
provisioner: driver.longhorn.io
allowVolumeExpansion: true
parameters:
  numberOfReplicas: "2"
  dataLocality: "best-effort"
  replicaAutoBalance: "best-effort"
  nodeSelector: "longhorn=storage"
volumeBindingMode: WaitForFirstConsumer
YAML
```

---

# 5) MetalLB (Service-IPs) – **Pool ohne DHCP-Kollision**

Dein DHCP vergibt `.200–.250`, Reservierungen liegen bei `.11–.13` & `.21–.23`, VIP ist `.100`.
→ Nimm z. B. **`.151–.170`** als LB-Pool.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.14.5/config/manifests/metallb-native.yaml

kubectl apply -f - <<'YAML'
apiVersion: metallb.io/v1beta1
kind: IPAddressPool
metadata: { name: default-pool, namespace: metallb-system }
spec:
  addresses: ["192.168.100.151-192.168.100.170"]
---
apiVersion: metallb.io/v1beta1
kind: L2Advertisement
metadata: { name: default, namespace: metallb-system }
spec: {}
YAML
```

---

# 6) Ingress Controller (NGINX) – für Web-Apps

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress ingress-nginx/ingress-nginx \
  -n ingress-nginx --create-namespace \
  --set controller.service.type=LoadBalancer
```

→ Erhält eine IP aus `192.168.100.151–170`.
Test: `kubectl -n ingress-nginx get svc ingress-ingress-nginx-controller -o wide`

*(TLS/Let’s Encrypt via cert-manager kannst du später ergänzen.)*

---

# 7) Smoke-Tests

```bash
# Clusterbasis
kubectl get nodes -o wide

# kube-vip / API
curl -sk https://192.168.100.100:6443/healthz   # 401/ok = gut

# Longhorn
kubectl -n longhorn-system get pods
# Optional: Longhorn-UI via NodePort/LB freigeben (später)

# MetalLB
kubectl -n ingress-nginx get svc | grep LoadBalancer

# PVC-Test (schnelles BusyBox-Deployment mit PVC)
kubectl apply -f - <<'YAML'
apiVersion: v1
kind: PersistentVolumeClaim
metadata: { name: test-pvc }
spec:
  accessModes: ["ReadWriteOnce"]
  resources: { requests: { storage: 2Gi } }
  storageClassName: longhorn-2r
---
apiVersion: apps/v1
kind: Deployment
metadata: { name: bb }
spec:
  replicas: 1
  selector: { matchLabels: { app: bb } }
  template:
    metadata: { labels: { app: bb } }
    spec:
      containers:
      - name: bb
        image: busybox
        command: ["sh","-c","sleep 1d"]
        volumeMounts:
        - name: data
          mountPath: /data
      volumes:
      - name: data
        persistentVolumeClaim:
          claimName: test-pvc
YAML
kubectl get pvc,pods
```

---

# 8) Nächste Schritte (kurz)

* **cert-manager + DNS-01 (Cloudflare)** für echte Zertifikate.
* **kube-prometheus-stack** für Monitoring/Alerts.
* **Backups** (Longhorn BackupTarget auf NFS/S3).
* **PDBs & topologySpreadConstraints** für kritische Apps.

Wenn du willst, pack ich dir das als **README.md** ins passende Repo-Format (inkl. der YAMLs).
