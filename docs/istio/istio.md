# Istio & Envoy
* Istio는 Service Mesh로, Envoy 프록시를 통한 트래픽 제어, 보안, 관찰을 담당함
* Istio `IngressGateway`는 외부 트래픽(클라이언트 → k8s 클러스터)을 Mesh로 진입시키는 gateway 역할을 함
  * 클러스터 진입점을 제어 - Gateway 리소스
* 내부 서비스 간 트래픽은 사이드카 `Envoy`가 직접 처리(서비스 간 통신에 JWT 검증/인가 포함)
  * 트래픽 라우팅 - virtualservice 리소스
  * 인증/인가 - RequestAuthentication / AuthorizationPolicy 리소스