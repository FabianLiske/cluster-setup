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