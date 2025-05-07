# Ingress Controller 배포하기
* GCP 환경에서 k8s cluster를 구성했기 때문에 GCP의 service를 활용하여 External IP를 할당

## 1. External IP를 어떻게 할당해야 할까?
1. `GCP Static IP 예약` 이용하여 `NodePort 타입의 Ingress Controller`를 배포하는 방식
   * 비용에 대한 부담이 적고 설정하기 간편
   * 하지만 낮은 가용성 - External IP에 해당하는 노드에 장애가 생기면 서비스 전체에 영향
   * 따라서 dev 환경에 사용하기 좋을듯
2. `GCP LoadBalancer`를 이용하여 `LoadBalancer 타입의 Ingress Controller`를 배포하는 방식
   * 비용 부담이 큼 - 트래픽이 많을수록 비용도 커짐
   * 고가용성 - GCP 헬스 체크로 5초 내 자동 트래픽 전환?
   * 프로덕션 환경에 사용하기 적합
3. Keepalived VIP 이용하여 NodePort `NodePort 타입의 Ingress Controller`를 배포하는 방식
   * 적절한 비용과 가용성이라고 함
   * 구현 난이도가 높고 관리 부담이 있다고 함
   * 비용에 민감할 경우 사용 고려해보면 좋을듯
   * 필요한 경우 생기면 직접 설정해 봐야 할듯

## 2. NodePort 타입의 Ingress Controller 배포하기 - Static IP 이용
* 브라우저는 기본적으로 http://도메인 요청 시 80번 포트를 사용하지만
* NodePort 서비스는 30000-32767 포트 범위에서만 작동하기 때문에
* NodePort 타입의 Ingress Controller를 사용하면 주소에 해당하는 NodePort가 들어가야 함
* 간편한 설정으로 dev 환경에 적합한 방법임

<br>

### 2.1. 외부 고정 IP 주소 예약하기
* 고정 IP를 사용하지 않고 **기본으로 할당된 외부 IP를 사용해도 상관없음**
* vm이 재실행되어도 변경되지 않는 고정 IP 주소를 이용하여 External IP로 할당
* GCP에서 Ingress Controller Service의 External IP로 할당할 Static IP 예약 (NodePort 설정임)
  * `VPC 네트워크 - IP 주소 - 외부 고정 IP 주소 예약`에서 아래의 내용들 입력하여 IP 예약
    * 연결 대상 vm은 worker노드로 지정 - Master 노드에 추가 부하를 주는 것은 클러스터 안정성에 영향을 줄 수 있기 때문
  * ![](2025-04-05-02-02-40.png)

<br>

### 2.2. Ingress Controller 배포
* k8s 1.30 버전과 호환되는 Ingress Controller 4.9.1 버전으로 배포
* `k8s-worker-03`노드에 연결한 고정 IP `34.64.121.40`
* `--set controller.config.proxy-set-headers=chatflow-headers` 옵션으로 chatflow-headers ConfigMap 설정 세팅 (3 항목의 Ingress 배포 확인)
  ```sh
  helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx

  helm repo update

  helm upgrade --install ingress-nginx ingress-nginx/ingress-nginx \
  --version 4.9.1 \
  --namespace ingress-nginx --create-namespace \
  --set controller.service.externalIPs[0]=34.64.54.101 \
  --set controller.service.type=NodePort \
  --set controller.nodeSelector."kubernetes\.io/hostname"=k8s-worker-03 \
  --set controller.config.proxy-set-headers=chatflow-headers \ 
  --set controller.config.use-forwarded-headers="true"

  # External IP 할당 확인
  kubectl -n ingress-nginx get svc -o wide
  ```
* ![](2025-04-05-02-59-15.png)

<br><br>

## 3. Ingress 배포 - (미완 설정)
* 헤더 설정을 위한 ConfigMap 배포
  ```yaml
  apiVersion: v1
  kind: ConfigMap
  metadata:
    name: chatflow-headers
    namespace: chatflow
  data:
    Authorization: "$http_authorization" # authorization 헤더
    Content-Type: "$http_content_type" # content-type 헤더
  ```
* Ingress 설정 정보
  ```yaml
  apiVersion: networking.k8s.io/v1
  kind: Ingress
  metadata:
    annotations:
      nginx.ingress.kubernetes.io/use-regex: "true" # regex 사용할 수 있도록 설정
      nginx.ingress.kubernetes.io/rewrite-target: /$2 # rewrite 설정 (/test 참고) 
      nginx.ingress.kubernetes.io/proxy-body-size: "10m"  # 요청 데이터 10MB 제한
      # CORS 설정 - GET과 POST에 대한 모든 Origin 요청 허용 
      nginx.ingress.kubernetes.io/enable-cors: "true"
      nginx.ingress.kubernetes.io/cors-allow-methods: "GET, POST"
      # 헤더 설정 - `chatflow-headers` ConfigMap 참고
      # proxy_set_header 설정 - Authorization와 Content-Type 헤더가 확실히 백엔드 서비스로 전달되도록 보장
      nginx.ingress.kubernetes.io/proxy-set-headers: "chatflow-headers"
    name: chatflow-ingress
    namespace: chatflow
  spec:
    ingressClassName: nginx
    rules:
    - host: flowchat.shop # 루트 도메인으로 설정, 모든 하위 도메인으로 적용하려면 "*.flowchat.shop" 와 같이 작성
      http:
        paths:
        # 정규식 패턴을 사용한 경로 그룹화 (ex. /(sign-up|sign-in|members|friendships))
        # /test 경로 rewrite 설정 (접두어 제거) (ex. /test/send/123 -> /send/123 으로 라우팅)
        # 그 외 경로는 기본적으로 svc-frontend로 라우팅 (/ 경로에 해당)
        - path: /
          pathType: Prefix
          backend:
            service:
              name: svc-frontend
              port:
                number: 3000
        - path: /(sign-up|sign-in|members|friendships)
          pathType: ImplementationSpecific
          backend:
            service:
              name: svc-member
              port:
                number: 8080
        - path: /(teams|images)
          pathType: ImplementationSpecific
          backend:
            service:
              name: svc-team
              port:
                number: 8080
        - path: /test(/|$)(.*)
          pathType: ImplementationSpecific
          backend:
            service:
              name: svc-test
              port:
                number: 8080
  ```



