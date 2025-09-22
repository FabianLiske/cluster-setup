# 1) Für alle Nodes

## 1.1) Init

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y kitty-terminfo
sudo timedatectl set-timezone Europe/Berlin
```

## 1.2) Swap deaktivieren

```bash
sudo swapoff -a
sudo sed -i 's/^\([^#].*swap\)/#\1/' /etc/fstab
```

## 1.3) K8s Module & Sysctls

```bash
echo -e "overlay\nbr_netfilter" | sudo tee /etc/modules-load.d/k8s.conf
sudo modprobe overlay && sudo modprobe br_netfilter
```

```bash
sudo rm /etc/sysctl.d/99-kubernetes.conf
sudo nano /etc/sysctl.d/100-multivlan.conf
```

```yaml
net.bridge.bridge-nf-call-iptables=1
net.ipv4.ip_forward=1
net.ipv6.conf.all.forwarding=1

net.ipv4.conf.all.rp_filter=0
net.ipv4.conf.default.rp_filter=0
net.ipv4.conf.vlan10.rp_filter=0
net.ipv4.conf.vlan20.rp_filter=0
net.ipv4.conf.vlan30.rp_filter=0
net.ipv4.conf.eth0.rp_filter=0
```

```bash
sudo sysctl --system
```

## 1.4) Longhorn Vorbereitung

```bash
sudo apt install -y open-iscsi nfs-common smartmontools
sudo systemctl enable --now iscsid
```

## 1.5) VLAN Subinterfaces

```bash
sudo nano /etc/netplan/90-vlans.yaml
```

```yaml
network:
  version: 2
  ethernets:
    eth0:
      dhcp4: true
  vlans:
    vlan10:
      id: 10
      link: eth0
      addresses: []
    vlan20:
      id: 20
      link: eth0
      addresses: []
    vlan30:
      id: 30
      link: eth0
      addresses: []
```

Apply und Test:

```bash
sudo netplan apply
# Wie ein guter Hobby-Admin ignorieren wir die Warnung zu den File Permissions
ip -br link | egrep '^vlan(10|20|30)\b'   # sollten UP sein
```

---

# 2) Control Plane Nodes

## 2.1) K3s auf cp-1

```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_EXEC="server \
    --cluster-init \
    --disable traefik \
    --disable servicelb \
    --tls-san 192.168.100.100 \
    --tls-san api.cluster.rohrbom.be" \
  sh -

sudo cat /var/lib/rancher/k3s/server/node-token
```

## 2.2) Alle weiteren Control Plane Nodes

```bash
curl -sfL https://get.k3s.io | \
  K3S_TOKEN="<TOKEN>" \
  INSTALL_K3S_EXEC="server \
    --server https://192.168.100.11:6443 \
    --disable traefik \
    --disable servicelb \
    --tls-san 192.168.100.100 \
    --tls-san api.cluster.rohrbom.be" \
  sh -
```

---

# 3) KubeVIP für externen Zugriff auf Cluster

## 3.1) DNS für API

api.cluster.rohrbom.be -> 192.168.100.100 in Cloudflare ohne Proxy

## 3.2) KubeVIP über cp-1 auf Cluster bringen

```bash
sudo kubectl apply -f kube-vip/kube-vip.yaml
```

Steps zum prüfen [hier](kube-vip/kubevip-guide.md)

---

# 4) Vorbereitung auf Admin-PC

## 4.1) Installation

```bash
sudo pacman -S kubectl
yay -S helm
```

## 4.2) Kubeconfig holen & auf VIP/FQDN umstellen

```bash
scp faba@cp-1:/etc/rancher/k3s/k3s.yaml ~/.kube/config
chmod 600 ~/.kube/config
# VIP/FQDN eintragen (eine von beiden Varianten):
# sed -i 's#server: https://.*:6443#server: https://192.168.100.100:6443#' ~/.kube/config
sed -i 's#server: https://.*:6443#server: https://api.cluster.rohrbom.be:6443#' ~/.kube/config
kubectl get nodes
```

---

# 5) Worker dem Cluster joinen

## 5.1) Token von `cp-1` holen:

```bash
# auf cp-1:
sudo cat /var/lib/rancher/k3s/server/node-token
```

## 5.2) Auf **jedem Worker** (Token einsetzen):

```bash
curl -sfL https://get.k3s.io | \
  K3S_TOKEN="<TOKEN>" \
  INSTALL_K3S_EXEC="agent --server https://192.168.100.100:6443" \
  sh -
```

Check:

```bash
kubectl get nodes -o wide
```

---

# 6) Longhorn installieren (ab hier wieder alles auf Admin-PC)

## 6.1) Control Plane tainten

```bash
kubectl taint nodes cp-X node-role.kubernetes.io/control-plane=:NoSchedule --overwrite
```

## 6.2) Worker tainten

```bash
kubectl label nodes wk-X node-role.kubernetes.io/worker= longhorn=storage --overwrite
```

## 6.3) Longhorn per Helm installieren

```bash
helm repo add longhorn https://charts.longhorn.io
helm repo update
helm install longhorn longhorn/longhorn -n longhorn-system --create-namespace
```

## 6.4) StorageClass (2 Repliken, Default)

```bash
kubectl apply -f longhorn/longhorn.yaml
```

---

# 7) MetalLB (Service-IPs)

Mein DHCP vergibt `.200–.250`, Reservierungen liegen bei `.11–.13` & `.21–.23`, VIP ist `.100`, daher **`.151–.199`** als LB-Pool.

```bash
kubectl apply -f https://raw.githubusercontent.com/metallb/metallb/v0.15.2/config/manifests/metallb-native.yaml
kubectl apply -f metallb/metallb-multi-vlan.yaml
kubectl -n metallb-system get ipaddresspools,l2advertisements
```

---

# 8) Ingress Controller (NGINX) - Multi VLAN

## 8.1) Helm Repo

```bash
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
```

## 8.2) VLAN 100

```bash
helm install ingress-vlan100-admin ingress-nginx/ingress-nginx \
  -n ingress-vlan100-admin --create-namespace \
  -f ingress/values-ingress-vlan100-admin.yaml
```

## 8.3) VLAN 10

```bash
helm install ingress-vlan10-mgmt ingress-nginx/ingress-nginx \
  -n ingress-vlan10-mgmt --create-namespace \
  -f ingress/values-ingress-vlan10-mgmt.yaml
```

## 8.4 VLAN 20

```bash
helm install ingress-vlan20-internal ingress-nginx/ingress-nginx \
  -n ingress-vlan20-internal --create-namespace \
  -f ingress/values-ingress-vlan20-internal.yaml
```

## 8.5 VLAN 30

```bash
helm install ingress-vlan30-public ingress-nginx/ingress-nginx \
  -n ingress-vlan30-public --create-namespace \
  -f ingress/values-ingress-vlan30-public.yaml
```

Erhält eine IP aus `192.168.100.151–199`.
Test:
```bash
# Jede Ingress-Instanz hat die richtige EXTERNAL-IP?
kubectl -n ingress-vlan100-admin   get svc | grep ingress-nginx-controller
kubectl -n ingress-vlan10-mgmt     get svc | grep ingress-nginx-controller
kubectl -n ingress-vlan20-internal get svc | grep ingress-nginx-controller
kubectl -n ingress-vlan30-public   get svc | grep ingress-nginx-controller

# IngressClasses registriert?
kubectl get ingressclass

# MetalLB gesund?
kubectl -n metallb-system get ipaddresspools,l2advertisements
kubectl -n metallb-system logs deploy/controller | tail -n 100
```

---

# 9) Longhorn Web UI

## 9.1) DNS für Longhorn

longhorn.cluster.rohrbom.be -> 192.168.100.151 in Cloudflare ohne Proxy

## 9.2) cert-manager via Helm

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.18.2/cert-manager.yaml
kubectl apply -f --server-side -f certmanager/certmanager-metrics.yaml
kubectl -n cert-manager get pods
```

## 9.3) Cloudflare Token als Secret

```bash
kubectl apply -f certmanager/cf-secret.yaml
```

## 9.4) Cert Issuer

```bash
kubectl apply -f certmanager/issuer.yaml
```

## 9.5) Longhorn Ingress

```bash
kubectl apply -f longhorn/longhorn-ui.yaml
```

## 9.6) Testen

```bash
kubectl -n longhorn-system get ingress longhorn-ui
kubectl -n longhorn-system get certificate longhorn-tls
```

---

# 10) Longhorn Nodes labeln

## 10.1) Longhorn UI abrufen

Für mich ist das [hier](https://longhorn.cluster.rohrbom.be)

## 10.2) Für jede Node

- Node selecten
- Unter Operation > Edit Node and Disks > New Tag
  - `general_storage`
- Save

---

# 11) Rancher auf VM

## 11.1) VM anlegen

- Ubuntu Server 24.04 LTS
- 2 CPUs
- 8GB RAM
- 64GB Disk
- NIC im 100er-VLAN

## 11.2) DNS für Rancher

rancher.cluster.rohrbom.be -> 192.168.100.2 in Cloudflare ohne Proxy

## 11.3) VM Init

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install kitty-terminfo -y
sudo timedatectl set-timezone Europe/Berlin

# Docker installieren
curl -fsSL https://get.docker.com | sudo sh
sudo systemctl enable --now docker

sudo usermod -aG docker $USER
```

## 11.4) Rancher Container starten

```bash
sudo docker run -d --restart=unless-stopped --privileged \
  -p 80:80 -p 443:443 \
  --name rancher rancher/rancher:stable
```

## 11.5) Rancher einrichten

- Bootstrap Passwort extrahieren und neues Passwort setzen
- Cluster importieren und erzeugten Befehl auf Admin PC ausführen (`curl ...`)

