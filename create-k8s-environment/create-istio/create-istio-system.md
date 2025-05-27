# Istio(+Envoy)와 Kiali 설치 방법

## 1. Istioctl & Istio & Envoy 설치
* Istioctl 설치
  ```sh
  curl -L https://istio.io/downloadIstio | sh -

  cd istio-*

  export PATH=$PWD/bin:$PATH

  istioctl version
  ```

* Istio 설치 (기본 프로필 사용 - default)
  * 설치후 istio-ingressgateway 서비스를 NodePort 타입으로 변경 (external-ip 없는 경우)
  ```sh
  istioctl install --set profile=default -y
  ```

* Envoy Proxy 설치
  * 원하는 네임스페이스에 사이드카 자동주입 활성화
  ```sh
  kubectl label namespace {namespace} istio-injection=enabled
  ```
  * 네임스페이스를 지정하여 설치하여도, 기존에 생성되어 있던 Pod에는 사이드카가 생성되지 않으므로 재시작해 주어야 함
  * Envoy Proxy 사이드카가 잘 설치되었다면 아래와 같이 Pod에 이미지가 추가된 모습을 확인 가능
  * ![](2025-05-23-01-49-49.png)
  
### Envoy Proxy 사이드카 추가시 주의점
* Gateway와 VirtualService로 라우팅 설정을 해주지 않으면 API에 접근이 불가능함
  * Gateway 및 VirtualService 설정은 4번 항목 참고...
* cf. 정적 리소스는 접근이 가능함 (ex. swagger-ui/index.html)


<br><br>

## 2. Kiali 설치
* Helm repo 등록
  ```sh
  helm repo add kiali https://kiali.org/helm-charts
  helm repo update
  kubectl create namespace kiali-operator
  ```
* Kiali Operator 설치
  ```sh
  helm install kiali-operator kiali/kiali-operator -n kiali-operator
  ```
* Kiali Custom Resource 생성
  * `external_services.prometheus.url`에는 prometheus datasource 주소 입력하면 됨
  ```sh
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
  ```

<br><br>

## 3. NodePort 타입 서비스 변경 및 설치 점검
* svc중 istio-ingressgateway 서비스를 NodePort타입으로 변경 (external-ip 할당 안되는 경우)
* svc중 kiali 서비스를 NodePort타입으로 변경
  ```sh
  kubectl get pods -n istio-system
  kubectl get pods -n kiali-operator
  kubectl get svc -n istio-system
  ```
* ![](2025-05-23-01-56-43.png)
* ![](2025-05-23-01-51-10.png)

<br>

* 이후 nodePort를 이용하여 Kiali 서비스 접근시 Kiali UI 확인 가능함
* ![](2025-05-23-01-58-19.png)

<br><br>

## 4. Gateway 및 VirtualService 설정 추가

### 4.1. Gateway 설정
* credentialName은 TLS 인증서를 담은 Kubernetes TLS Secret의 이름
* istio-system 네임스페이스에 존재해야 함
* 테스트용 인증서 생성(tls.crt, tls.key)
  ```sh
  openssl req -x509 -nodes -days 365 \
    -newkey rsa:2048 \
    -keyout tls.key \
    -out tls.crt \
    -subj "/CN=flowchat.shop/O=Flowchat"

  # gateway에 적용할 secret 생성
  kubectl create -n istio-system secret tls flowchat-cert --key tls.key --cert tls.crt
  ```
* 예시 설정 정보
  ```yaml
  apiVersion: networking.istio.io/v1beta1
  kind: Gateway
  metadata:
    name: chatflow-gateway
    namespace: istio-system
  spec:
    selector:
      istio: ingressgateway
    servers:
    - port:
        number: 80       # 내부 Envoy가 수신할 HTTP 포트
        name: http
        protocol: HTTP
      hosts:
      - "flowchat.shop"
    - port:
        number: 443      # 내부 Envoy가 수신할 HTTPS 포트
        name: https
        protocol: HTTPS
      hosts:
      - "flowchat.shop"
      tls:
        mode: SIMPLE
        credentialName: flowchat-cert  # istio-system 네임스페이스의 Secret에 해당
  ```

### 4.2. VirtualService 설정
* 예시 설정 정보의 첫번째 match 설정에서, `/member`, `/v2/member`와 같은 경로들은 모두 prefix 유지 상태로 라우팅됨
* 예시 설정 정보의 두번째 match 설정에서, `/test` 경로는 `/` 경로로 rewrite 되어 요청되도록 설정
* 예시 설정 정보의 마지막 match 설정에서, **라우팅 우선순위는 위에서 아래로 평가**되므로 catch-all 룰(`/`)은 항상 마지막에 추가
* 예시 설정 정보
  ```yaml
  apiVersion: networking.istio.io/v1
  kind: VirtualService
  metadata:
    name: chatflow-routing
    namespace: chatflow
  spec:
    gateways:
    - istio-system/chatflow-gateway
    hosts:
    - flowchat.shop
    http:
    - match:
      - uri:
          prefix: /members
      - uri:
          prefix: /friendships
      - uri:
          prefix: /sign-up
      - uri:
          prefix: /sign-in
      route:
      - destination:
          host: member.chatflow.svc.cluster.local
          port:
            number: 8080

    - match:
      - uri:
          prefix: /teams
      - uri:
          prefix: /v2/teams
      route:
      - destination:
          host: team.chatflow.svc.cluster.local
          port:
            number: 8080

    - match:
      - uri:
          prefix: /test
      rewrite:
        uri: /  # test 경로는 제거하여 요청
      route:
      - destination:
          host: test.chatflow.svc.cluster.local
          port:
            number: 8080

    - match:
      - uri:
          prefix: /
      route:
      - destination:
          host: frontend.chatflow.svc.cluster.local
          port:
            number: 3000
  ```