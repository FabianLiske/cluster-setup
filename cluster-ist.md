# Cluster – Ist-Zustand & Rahmen für neue Dienste

## Überblick

* **Distro/Flavor:** K3s v1.33.4+k3s1 (HA Control Plane & Worker Nodes).
* **API VIP:** `192.168.100.100` via kube-vip → `https://api.cluster.rohrbom.be`.
* **Storage:** Longhorn, Default-SC `longhorn-2r` (2 Repliken), Worker sind als Storage-Nodes gelabelt.
* **Zertifikate:** cert-manager mit Cloudflare DNS-01 (`ClusterIssuer: letsencrypt-dns`).
* **LoadBalancer:** MetalLB, je VLAN genau **eine** feste LB-IP `.151`.
* **Ingress:** Vier getrennte ingress-nginx Instanzen (pro VLAN eine) mit eigener IngressClass.

## Netzwerk & Ingress

| VLAN | Zweck               | Namespace (Ingress)       | IngressClassName | Controller-ID (Ist)             | LB-IP             | DNS/Erreichbarkeit |
| ---: | ------------------- | ------------------------- | ---------------- | ------------------------------- | ----------------- | ------------------ |
|  100 | Cluster-Admin       | `ingress-vlan100-admin`   | `nginx-admin`    | `k8s.io/ingress-nginx-admin`    | `192.168.100.151` | intern (Admin)     |
|   10 | Management-Tools    | `ingress-vlan10-mgmt`     | `nginx-mgmt`     | `k8s.io/ingress-nginx-mgmt`     | `192.168.10.151`  | intern (Mgmt)      |
|   20 | Interne Dienste     | `ingress-vlan20-internal` | `nginx-internal` | `k8s.io/ingress-nginx-internal` | `192.168.20.151`  | intern (Service)   |
|   30 | Öffentliche Dienste | `ingress-vlan30-public`   | `nginx`          | `k8s.io/ingress-nginx`          | `192.168.30.151`  | extern (Public)    |

**Router/DNS:**

* Port-Forward 80/443 **nur** → `192.168.30.151` (VLAN 30).
* Interne FQDNs via Split-DNS (Pi-hole etc.) → jeweilige `.151` im passenden VLAN.
* Cloudflare Records **DNS only** (kein Proxy) für Public-Hosts.

**MetalLB:**

* Pools: `pool-vlan100-admin`, `pool-vlan10-mgmt`, `pool-vlan20-internal`, `pool-vlan30-public` (jeweils exakt `…151-…151`).
* L2Advertisements binden an `eth0` (VLAN100), `vlan10`, `vlan20`, `vlan30`.

## Zertifikate

* **Issuer:** `ClusterIssuer/letsencrypt-dns`.
* **Secret:** `certmanager/cf-secret.yaml` (Token hat Zone.DNS-Edit).
* Referenz & How-To: `certmanager/certmanager-guide.md`.

## Storage

* **Default:** `longhorn-2r` (siehe `longhorn/longhorn.yaml`).
* Longhorn-UI: `https://longhorn.cluster.rohrbom.be` (Ingress ohne Whitelist-Annotation), Ingressdatei: `longhorn/longhorn-ui.yaml`.

## Repo – relevante Pfade

```
certmanager/        # Issuer & CF Secret (Beispiele + Guide)
kube-vip/           # kube-vip.yaml + Guide
metallb/            # metallb-multi-vlan.yaml (Produktiv)
ingress/            # je VLAN eigene values-*.yaml für ingress-nginx
longhorn/           # StorageClass + Longhorn UI Ingress
```

---

# Onboarding neuer Dienste (praktische Leitplanken)

## 1) Namespace & Labels

* Pro Anwendung **eigener Namespace** (z. B. `svc-<app>`).
* Labels: `app=<name>`, optional `tier=frontend|backend`, `team=<x>`.

```bash
kubectl create namespace svc-example
```

## 2) Service-Typ & Ports

* **Services immer `ClusterIP`** (Ingress übernimmt externe Erreichbarkeit).
* Containerports klar benennen (`http`, `metrics`…), Liveness/Readiness nutzen.

## 3) Storage (falls nötig)

* Default SC: `longhorn-2r`.

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: data
  namespace: svc-example
spec:
  accessModes: [ReadWriteOnce]
  storageClassName: longhorn-2r
  resources:
    requests:
      storage: 5Gi
```

## 4) TLS & DNS

* Ingress mit `cert-manager.io/cluster-issuer: letsencrypt-dns`.
* **DNS-A-Record** auf die **richtige VLAN-LB-IP** zeigen lassen:

  * Public → `192.168.30.151`
  * Intern/Mgmt/Admin → jeweilige `.151` im Ziel-VLAN (Split-DNS)

## 5) Ingress-Klasse wählen

* Admin-Zugriff (Cluster-Tools) → `nginx-admin` (VLAN 100)
* Management-Tools → `nginx-mgmt` (VLAN 10)
* Interne Business-Dienste → `nginx-internal` (VLAN 20)
* Öffentlich erreichbare Dienste → `nginx` (VLAN 30)

> **Wichtig:** In jedem Ingress **`spec.ingressClassName`** setzen.

## 6) Minimal-Blueprint (YAML)

> Passe `host`, `namespace`, `ingressClassName` und Ressourcengrößen an.
> Für **öffentliche** Dienste nimm `ingressClassName: nginx` und setze den Public-DNS auf `192.168.30.151`.

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: example
  namespace: svc-example
  labels: { app: example }
spec:
  replicas: 2
  selector: { matchLabels: { app: example } }
  template:
    metadata: { labels: { app: example } }
    spec:
      containers:
      - name: web
        image: ghcr.io/yourorg/example:latest
        ports: [{ containerPort: 8080, name: http }]
        resources:
          requests: { cpu: 100m, memory: 128Mi }
          limits:   { cpu: 500m, memory: 512Mi }
        readinessProbe:
          httpGet: { path: /healthz, port: http }
          initialDelaySeconds: 10
          periodSeconds: 10
        livenessProbe:
          httpGet: { path: /livez, port: http }
          initialDelaySeconds: 20
          periodSeconds: 20
---
apiVersion: v1
kind: Service
metadata:
  name: example
  namespace: svc-example
spec:
  selector: { app: example }
  ports:
  - name: http
    port: 80
    targetPort: http
  type: ClusterIP
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: example
  namespace: svc-example
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-dns
spec:
  ingressClassName: nginx-internal     # oder nginx/nginx-mgmt/nginx-admin
  rules:
  - host: example.internal.tld         # DNS → passende .151 IP
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: example
            port: { number: 80 }
  tls:
  - hosts: [example.internal.tld]
    secretName: example-tls
```

## 7) Sicherheitsrahmen

* Inter-VLAN-Zugriffe durch **Firewall** steuern (keine „whitelist-source-range“ mehr in Ingressen nötig).
* Optional **NetworkPolicies** (Default-deny + explizite Freigaben), z. B. Traffic nur vom passenden Ingress-Namespace erlauben.
* Keine NodePorts/hostNetwork für App-Workloads, außer es ist bewusst gewollt.

## 8) Betriebschecks (kurz)

```bash
kubectl get pods -A -o wide
kubectl get ingressclass
kubectl -n metallb-system get ipaddresspools,l2advertisements
for ns in ingress-vlan100-admin ingress-vlan10-mgmt ingress-vlan20-internal ingress-vlan30-public; do
  kubectl -n $ns get svc | grep ingress-nginx-controller
done
```

## 9) Troubleshooting (kurz)

* **403 über Ingress:** Prüfe, ob Cloudflare **DNS only** ist und keine alte Whitelist-Annotation mehr gesetzt ist.
* **EXTERNAL-IP `<pending>`:** Interface-Namen in L2Advertisements korrekt (`eth0`, `vlan10/20/30`) und IFs **UP**?
* **TLS fehlt:** FQDN → korrekte `.151`? cert-manager Logs prüfen (`kubectl -n cert-manager logs deploy/cert-manager`).

---

## Verweise im Repo

* Kube-VIP: `kube-vip/kube-vip.yaml`, Doku: `kube-vip/kubevip-guide.md`
* MetalLB: `metallb/metallb-multi-vlan.yaml`
* Ingress Values: `ingress/values-ingress-*.yaml`
* Cert-Manager: `certmanager/issuer.yaml`, `certmanager/certmanager-guide.md`
* Longhorn: `longhorn/longhorn.yaml`, `longhorn/longhorn-ui.yaml`
