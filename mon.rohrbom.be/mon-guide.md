# Namespace

```bash
kubectl apply -f mon.rohrbom.be/ns.yaml
```

# InfluxDB

```bash
kubectl apply -f mon.rohrbom.be/influxdb/influxdb-statefulset.yaml
kubectl apply -f mon.rohrbom.be/influxdb/influxdb-service.yaml
kubectl apply -f mon.rohrbom.be/influxdb/influxdb-ingress.yaml
```

# Grafana

```bash
kubectl apply -f mon.rohrbom.be/grafana/grafana.yaml
```

# Prometheus & Exporter

```bash
kubectl apply -f mon.rohrbom.be/node-exporter/node-exporter.yaml
kubectl apply -f mon.rohrbom.be/kube-state-metrics/ksm.yaml
kubectl apply -f kube-vip/kube-vip-metrics.yaml
kubectl apply -f metallb/metallb-metrics.yaml
helm upgrade --install ingress-vlan100-admin ingress-nginx/ingress-nginx \
  -n ingress-vlan100-admin --create-namespace \
  -f ingress/values-ingress-vlan100-admin.yaml
helm upgrade --install ingress-vlan10-mgmt ingress-nginx/ingress-nginx \
  -n ingress-vlan10-mgmt --create-namespace \
  -f ingress/values-ingress-vlan10-mgmt.yaml
helm upgrade --install ingress-vlan20-internal ingress-nginx/ingress-nginx \
  -n ingress-vlan20-internal --create-namespace \
  -f ingress/values-ingress-vlan20-internal.yaml
helm upgrade --install ingress-vlan30-public ingress-nginx/ingress-nginx \
  -n ingress-vlan30-public --create-namespace \
  -f ingress/values-ingress-vlan30-public.yaml
kubectl apply --server-side -f longhorn/longhorn-metrics.yaml
kubectl apply --server-side -f certmanager/certmanager-metrics.yaml
kubectl apply -f mon.rohrbom.be/grafana/grafana.yaml
kubectl apply -f mon.rohrbom.be/influxdb/influxdb.yaml
kubectl apply -f mon.rohrbom.be/prometheus/prometheus.yaml
```