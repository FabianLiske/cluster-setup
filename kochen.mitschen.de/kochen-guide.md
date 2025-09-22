# Secrets für GHCR deployen

```bash
kubectl apply -f kochen.mitschen.de/ns.yaml
kubectl apply -f kochen.mitschen.de/ghcr-cred.yaml
kubectl apply -f kochen.mitschen.de/kochen.yaml
```