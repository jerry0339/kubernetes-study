# Istio & Envoy
* Istio는 Service Mesh로, Envoy 프록시를 통한 트래픽 제어, 보안, 관찰을 담당함
* Istio `IngressGateway`는 외부 트래픽(클라이언트 → k8s 클러스터)을 Mesh로 진입시키는 gateway 역할을 함
  * 클러스터 진입점을 제어 - Gateway 리소스
* 내부 서비스 간 트래픽은 사이드카 `Envoy`가 직접 처리(서비스 간 통신에 JWT 검증/인가 포함)
  * 트래픽 라우팅 - virtualservice 리소스
  * 인증/인가 - RequestAuthentication / AuthorizationPolicy 리소스

<br>

## Istio 리소스 설정
* `Gateway`: 클러스터 외부로부터의 트래픽을 수신하는 진입점
* `VirtualService`: 트래픽 라우팅 규칙 정의
* `DestinationRule`: VirtualService가 트래픽을 전달할 대상 서비스의 세부 설정 정의
* `RequestAuthentication`: JWT 검증 정의
* `AuthorizationPolicy`: 검증된 JWT를 바탕으로 접근 허용/차단

<br>

### 1. Gateway 설정
* Gateway: Istio Ingress Gateway에 어떤 포트(HTTP, HTTPS 등)를 열고 어떤 호스트명에 대해 수신할지를 정의
* credentialName은 TLS 인증서를 담은 Kubernetes TLS Secret의 이름
  * istio-system 네임스페이스에 존재해야 함
  * **테스트용 인증서** 생성(tls.crt, tls.key)
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

<br>

### 2. VirtualService 설정
* VirtualService: URI, 헤더, 쿠키 등의 조건에 따라 어떤 서비스로 트래픽을 보낼지 지정
* 예시 설정 정보의 첫번째 match 설정에서, `/member`, `/v2/member`와 같은 경로들은 모두 prefix 유지 상태로 라우팅됨
* 예시 설정 정보의 두번째 match 설정에서, `/test` 경로는 `/` 경로로 rewrite 되어 요청되도록 설정
* 예시 설정 정보의 마지막 match 설정에서, **라우팅 우선순위는 위에서 아래로 평가**되므로 catch-all 룰(`/`)은 항상 마지막에 추가
* subset 설정과 name 설정은 Rollout CRD에서 strategy 설정시 사용
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
      name: http.0
      route:
        - destination:
            host: member-prod.chatflow.svc.cluster.local
            port:
              number: 8080
            subset: stable
          weight: 100
        - destination:
            host: member-prod.chatflow.svc.cluster.local
            port:
              number: 8080
            subset: canary
          weight: 0
    - match:
      - uri:
          prefix: /teams
      - uri:
          prefix: /channels
      name: http.1
      route:
        - destination:
            host: team-prod.chatflow.svc.cluster.local
            port:
              number: 8080
            subset: stable
          weight: 100
        - destination:
            host: team-prod.chatflow.svc.cluster.local
            port:
              number: 8080
            subset: canary
          weight: 0
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

<br>

### 3. DestinationRule 설정
* DestinationRule
  * Canary/Blue-Green 배포시 stable, canary 서브셋 정의
  * 서비스 간 mTLS 제어?
  * Circuit Breaking 설정
* 예시 설정 정보
  ```yaml
  apiVersion: networking.istio.io/v1beta1
  kind: DestinationRule
  metadata:
    name: member-destinationrule
    namespace: chatflow
  spec:
    host: member-prod.chatflow.svc.cluster.local
    subsets:
      - labels:
          app.kubernetes.io/instance: member-prod
          app.kubernetes.io/name: member
        name: stable
      - labels:
          app.kubernetes.io/instance: member-prod
          app.kubernetes.io/name: member
        name: canary

  ```

<br>

### 4. RequestAuthentication 설정
* HMAC 방식(HS512)으로 설정시 RequestAuthentication 생성 방법에 해당
  * cf. RS512 + JWKS URL 인증 방식 사용시 jwks대신 jwksUri 속성을 사용
    ```yaml
    # 예시
    jwtRules:
      - issuer: "https://auth.example.com"
        jwksUri: "https://auth.example.com/jwks.json"
    ```
* secret key값은 base64로 인코딩한 값을 jwks의 "k" 설정에 기입
* jwks의 key가 1개 이상인 경우, kid를 포함해야 함
  * kid는 secret key의 id임
* `forwardOriginalToken: true` 설정 - JWT 검증 후에 헤더에서 token 그대로 전달
* 예시 설정 정보
  ```yaml
  apiVersion: security.istio.io/v1beta1
  kind: RequestAuthentication
  metadata:
    name: member-authentication
    namespace: chatflow
  spec:
    selector:
      matchLabels:
        app.kubernetes.io/instance: member-prod
        app.kubernetes.io/name: member
    jwtRules:
      - forwardOriginalToken: true # false 설정시, Istio가 JWT 검증 후에 헤더에서 token 제거함
        issuer: "jerry0339" # JWT의 iss 필드 값
        # audiences:
        #  - "team"        # JWT의 aud 필드 값
        # jwks의 keys 값이 1개 이상이면 kid(secret key의 id) 반드시 포함
        jwks: |
          {
            "keys": [
              {
                "kty": "oct",
                "kid": "v1",
                "k": "dXNlcl90b2tlbl9mb3Jfc2lnbmF0dXJlX211c3RfYmVfYXRfbGVhc3RfMjU2X2JpdHNfaW5fSE1BQ19zaWduYXR1cmVfYWxnb3JpdGhtcw==",
                "alg": "HS512"
              }
            ]
          }
    selector:
      matchLabels:
        app.kubernetes.io/instance: member-prod
        app.kubernetes.io/name: member
  ```

<br>

### 5. AuthorizationPolicy 설정
* `action: DENY` 설정도 있는데, 설정하려면 아래와 같은 ALLOW 설정과 따로 작성해야 함
* AuthorizationPolicy 리소스 추가시, matchLabels에 해당하는 파드에 HTTP 요청시 인증 token을 헤더에 포함해야 접근 가능
  * 헤더가 필요없는 경로 설정 가능한데 따로 설정해야 함 - member-authorization-public 참고
* 예시 설정 정보1 (member-authorization-secured)
  ```yaml
  apiVersion: security.istio.io/v1beta1
  kind: AuthorizationPolicy
  metadata:
    name: member-authorization-secured
    namespace: chatflow
  spec:
    action: ALLOW
    rules:
      - to: # ADMIN role만 허용
          - operation:
              paths:
                - /admins/*
        when:
          - key: request.auth.claims[role]
            values:
              - ADMIN
      - to: # MEMBER role만 허용
          - operation:
              paths:
                - /members/*
                - /friendships/*
        when:
          - key: request.auth.claims[role]
            values:
              - MEMBER
      - from: # 그 외의 모든 경로는 모두(MEMBER, ADMIN) 접근 가능
          - source:
              requestPrincipals:
                - '*'
        when:
          - key: request.auth.claims[role]
            values:
              - MEMBER
              - ADMIN
    selector:
      matchLabels:
        app.kubernetes.io/instance: member-prod
        app.kubernetes.io/name: member
  ```

* 예시 설정 정보2 (member-authorization-public)
```yaml
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
  name: member-authorization-public
  namespace: chatflow
spec:
  action: ALLOW
  rules:
    - to:
        - operation:
            paths:
              - /sign-up
              - /sign-in
              - /member/swagger
              - /member/swagger-ui
              - /member/swagger-ui/*
              - /member/v3/api-docs
              - /member/v3/api-docs/*
      # `when:` 설정 빼서 인증 헤더 없이 접근 가능하도록 설정 가능
  selector:
    matchLabels:
      app.kubernetes.io/instance: member-prod
      app.kubernetes.io/name: member
```