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
kubectl apply -f mon.rohrbom.be/prometheus/prometheus.yaml
kubectl apply -f mon.rohrbom.be/node-exporter/node-exporter.yaml
kubectl apply -f mon.rohrbom.be/kube-state-metrics/ksm.yaml
```