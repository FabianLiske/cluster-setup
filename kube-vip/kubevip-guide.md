```bash
kubectl apply -f kube-vip.yaml
kubectl -n kube-system rollout status ds/kube-vip-ds
kubectl -n kube-system get pods -l app=kube-vip -o wide
curl -sk https://192.168.100.100:6443/healthz  # 401/ok erwartet
```
