```bash
sudo kubectl apply -f kube-vip/kube-vip.yaml
sudo kubectl -n kube-system rollout status ds/kube-vip-ds
sudo kubectl -n kube-system get pods -l app=kube-vip -o wide

# Funktionstest
curl -sk https://192.168.100.100:6443/healthz   # 401/ok ist gut

```
