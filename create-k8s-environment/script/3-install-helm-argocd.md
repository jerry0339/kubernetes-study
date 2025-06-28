## 사전 작업
* docker-registry user의 secret 생성
  * user 정보 - username, password

<br>

## 사후 작업
```
argocd-image-updater.argoproj.io/image-list
chatflow-fe=chatflow/fe

argocd-image-updater.argoproj.io/chatflow-fe.allow-tags
regexp:^[0-9].[0-9].[0-9]-[0-9]{6}.[0-9]{6}$

argocd-image-updater.argoproj.io/chatflow-fe.update-strategy
alphabetical
```

## 스크립트
```sh
#!/bin/bash
set -e  # 에러 발생 시 스크립트 중단


echo "🔧 Helm 설치 중..."
curl https://baltocdn.com/helm/signing.asc | gpg --dearmor | sudo tee /usr/share/keyrings/helm.gpg > /dev/null
sudo apt-get install apt-transport-https --yes
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/helm.gpg] https://baltocdn.com/helm/stable/debian/ all main" | sudo tee /etc/apt/sources.list.d/helm-stable-debian.list

sudo apt-get update
sudo apt-get install -y helm=3.15.4-1
helm version # 설치 잘 되었는지 확인
echo "✅ Helm 설치 완료"

echo "🔧 Argo CD 설치 중..."
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

cat > argocd-values.yaml <<EOF
server:
  service:
    type: NodePort
    nodePortHttps: 32000
EOF

helm upgrade --install argocd argo/argo-cd -f argocd-values.yaml -n argocd --create-namespace
echo "✅ Argo CD 설치 완료"

echo "🔧 ArgoCD Image Updater 설치중..."
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj-labs/argocd-image-updater/stable/manifests/install.yaml

kubectl create secret docker-registry regcred \
  --docker-server=https://index.docker.io/v1/ \
  --docker-username='chatflow' \
  --docker-password='password' \
  --docker-email='chatflowgcp@gmail.com' \
  -n argocd

# ConfigMap 생성까지 대기
echo "⏳ ConfigMap이 생성될 때까지 대기중..."
until kubectl get configmap argocd-image-updater-config -n argocd &> /dev/null; do
  echo "Waiting for argocd-image-updater-config to be created..."
  sleep 2
done

kubectl patch configmap argocd-image-updater-config \
  -n argocd \
  --type merge \
  -p '
data:
  registries.conf: |
    registries:
      - name: Docker Hub
        prefix: docker.io
        api_url: https://index.docker.io/v1/
        credentials: pullsecret:argocd/regcred
        defaultns: library
        default: true
'

echo "✅ ArgoCD Image Updater 설치 완료"

echo "🔧 Argo Rollouts 설치중..."
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

cat > rollouts-values.yaml <<EOF
controller:
  replicas: 1

dashboard:
  enabled: true
  service:
    type: NodePort
    portName: dashboard
    port: 3100
    targetPort: 3100
    nodePort: 32100
EOF

helm upgrade --install argo-rollouts argo/argo-rollouts \
  --namespace argo-rollouts \
  --create-namespace \
  -f rollouts-values.yaml
echo "✅ Argo Rollouts 설치 완료"

curl -LO https://github.com/argoproj/argo-rollouts/releases/download/v1.8.3/kubectl-argo-rollouts-linux-amd64
chmod +x kubectl-argo-rollouts-linux-amd64
sudo mv kubectl-argo-rollouts-linux-amd64 /usr/local/bin/kubectl-argo-rollouts

echo "Argo Rollouts 버전 확인"
kubectl argo rollouts version

sudo rm -f argocd-values.yaml rollouts-values.yaml

echo "ArgoCD 대시보드 초기 비밀번호 조회"
kubectl -n argocd get secret argocd-initial-admin-secret \
  -o jsonpath="{.data.password}" | base64 -d; echo
```

