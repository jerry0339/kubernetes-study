# Service 이해하기 위한 배경지식

## CoreDNS
* CoreDNS는 Kubernetes 클러스터 내에서 서비스 디스커버리를 담당
* 서비스와 파드에 대한 Domain을 생성
* 서비스와 파드의 도메인과 해당하는 ip가 저장되어 있음
* 생성된 Domain 통해 클러스터 내부의 오브젝트가 IP 주소 대신 Domain 이름으로 통신할 수 있도록 함

## kube-proxy
* Service에는 여러 파드가 연결될 수 있음
* Service와 연결된 Pod들의 **엔드포인트 목록을 관리**하는 역할
* Service에 트래픽이 유입되면, 각 노드마다 존재하는 kube-proxy에 의해 **분산됨**

# Service




* [이전(section 4)](/k8s-basic/section-04/2.Service/service.md)에는 service를 통해 파드에 접근하는 방법을 다뤘음
* section 7 에서는 **파드 입장**에서 원하는 서비스(파드)에 접근하거나 외부 서비스에 접근하는 방법을 다룸
  * 파드에서 서비스로 접근 - `Headless` Service와 Cluster내의 DNS Server 이용
  * 파드에서 외부 서비스 - `ExternalName` Service 이용
* Service를 통해 분산된 트래픽이 Pod에 전달됨
  * 트래픽 분산은 k8s의 kube-proxy에 의해 이루어 짐
  * kube-proxy는 각 노드에서 실행되며, 서비스와 연결된 Pod들의 **엔드포인트 목록을 관리**