# Secrets für GHCR deployen

```bash
kubectl create namespace svc-kochen
kubectl apply -f kochen.mitschen.de/ghcr-cred.yaml
kubectl apply -f kochen.mitschen.de/kochen.yaml
```