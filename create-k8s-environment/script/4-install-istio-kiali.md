## 사전 작업
* envoy 적용할 namespace 확인 - 아래 스크립트에는 chatflow로 되어 있음
* Kiali 설치시 Prometheus 설치되어 있어야 함

<br>

## 사후 작업
* external-ip가 없는 경우, istio-ingressgateway 서비스를 NodePort 타입으로 변경

## 스크립트
```sh
#!/bin/bash
set -e

echo "🔧 Istio 설치중..."
# Istioctl 설치
curl -L https://istio.io/downloadIstio | sh -
cd istio-*
export PATH=$PWD/bin:$PATH
istioctl version

# Istio 기본 프로파일 설치
istioctl install --set profile=default -y

# chatflow 네임스페이스 생성 및 사이드카 주입 활성화
# create는 오류 발생 가능하므로 apply로 선언적 구성
kubectl create namespace chatflow --dry-run=client -o yaml | kubectl apply -f -
kubectl label namespace chatflow istio-injection=enabled --overwrite

echo "✅ Istio 설치 완료"

echo "🔧 Kiali 설치중..."
helm repo add kiali https://kiali.org/helm-charts
helm repo update
kubectl create namespace kiali-operator

helm install kiali-operator kiali/kiali-operator -n kiali-operator

cat <<EOF | kubectl apply -f -
apiVersion: kiali.io/v1alpha1
kind: Kiali
metadata:
  name: kiali
  namespace: istio-system
spec:
  auth:
    strategy: anonymous
  external_services:
    prometheus:
      url: "http://nps.flowchat.shop:32090"
EOF

echo "✅ Kiali 설치 완료"
```