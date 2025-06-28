
## 스크립트
```sh
#!/bin/bash
set -e  # 에러 발생 시 스크립트 중단

echo "🔧 k8s-dashboard 설치 중..."
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml

kubectl patch svc kubernetes-dashboard \
  -n kubernetes-dashboard \
  -p '{"spec": {"type": "NodePort", "ports": [{"port": 443, "targetPort": 8443, "nodePort": 30000}]}}'

kubectl patch deployment kubernetes-dashboard \
  -n kubernetes-dashboard \
  --type='json' \
  -p='[{"op": "add", "path": "/spec/template/spec/containers/0/args/-", "value": "--enable-skip-login"}]'

sudo mkdir -p /etc/kubernetes/dashboard

sudo tee /etc/kubernetes/dashboard/kubernetes-dashboard-admin.yaml > /dev/null <<EOF
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubernetes-dashboard-admin
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: cluster-admin
subjects:
- kind: ServiceAccount
  name: kubernetes-dashboard
  namespace: kubernetes-dashboard
EOF

kubectl apply -f /etc/kubernetes/dashboard/kubernetes-dashboard-admin.yaml
echo "✅ k8s-dashboard 설치 완료"

echo "🔧 metrics server 설치 중..."
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.7.0/components.yaml

kubectl -n kube-system patch deployment metrics-server \
  --type='json' \
  -p='[{"op":"add","path":"/spec/template/spec/containers/0/args/-","value":"--kubelet-insecure-tls"}]'

echo "✅ metrics server 설치 완료"
```