# Service
* Service를 더욱 심도있게 이해하기 위해서 Endpoint, CoreDNS, 그리고 kube-proxy의 개념에 대해 먼저 확인이 필요함

<br>

## Endpoint
* Endpoint는 Kubernetes의 **Service와 연결된 Pod들의 IP 주소 및 포트**를 나타냄
* Endpoint는 Service가 트래픽을 라우팅할 수 있도록 하는 핵심 정보를 가지고 있음
* ex. 특정 서비스에 여러 파드가 연결된 경우
  * 서비스가 여러 Pod로 트래픽을 분산시키기 위해, 해당 서비스의 Endpoint 객체는 연결된 Pod들의 IP 주소 목록을 포함
  * ![](2024-12-29-00-59-58.png)

<br><br>

## CoreDNS
* CoreDNS는 Kubernetes 클러스터 내에서 **서비스 디스커버리**를 담당
* 서비스와 파드에 대한 Domain을 생성
* 서비스와 파드의 도메인과 해당하는 ip가 저장되어 있음
* 생성된 Domain 통해 클러스터 내부의 오브젝트(서비스, 파드)가 IP 주소 대신 Domain 이름으로 통신할 수 있도록 함
* 왜 도메인으로 접근이 필요할까?
  * k8s환경에서 Pod는 빈번하게 동적으로 할당, 변경, 제거됨
  * DNS를 이용하면 이러한 변화에 대응하여 안정적인 네트워크 연결을 제공할 수 있음
* `nslookup` 명령어로 직접 도메인 레코드에 접근하는 경우, 클러스터내의 네트워크 대역에서만 호출이 가능하다.
  * CoreDNS가 pod로 구성되어 있어 **Cluster내에 위치**하기 때문
  * ![](2024-12-31-23-57-37.png)
  * 따라서 `kubectl exec` 명령어로 파드에 접속이 가능하므로, 일반적으로 Pod내에서만 CoreDNS에 접근이 가능함

<br><br>

## 클러스터 네트워크 대역
* 보통 `클러스터 내` 라고 표현하는 경우 클러스터 네트워크 대역에 위치한 Pod들과 Service들을 나타내는 것
  * 따라서, **master노드와 worker노드들은 클러스터 내에 있다고 표현할 수 없음**
* 클러스터에서 네트워크 대역은 Pod 네트워크 CIDR와 서비스 네트워크 CIDR로 나뉨
* Pod 네트워크 CIDR
  *  Pod에 할당되는 IP 주소 범위를 정의, 클러스터 초기화시 설정함
  * `Calico` 사용시: `192.168.0.0/16`
  * `Flannel` 사용시: `10.244.0.0/16`
* 서비스 네트워크 CIDR
  * 서비스(ClusterIP 등)에 할당되는 IP 주소 범위를 정의

<br><br>

## CoreDNS와 Endpoint 통신
* CoreDNS는 Kubernetes API 서버를 통해 **Endpoint 정보를 조회**하여, Service에 대한 Domain 요청을 해당하는 Pod로 라우팅할 수 있도록 함
* Headless Service의 경우, CoreDNS는 ClusterIP Service를 거치지 않고 직접 Pod들의 Endpoint IP 주소를 반환
  * Headless Service는 Service의 IP가 없고, Service도메인에 파드들의 IP가 매칭되기 때문
* ex. Pod가 다른 Pod와 통신하고자 하는 경우 
  * Cluster IP Service 이용(아래 그림의 왼쪽) - Service 도메인을 통해 Pod1, Pod2와 통신
  * Headless Service 이용(아래 그림의 오른쪽) - Pod 도메인 이용하여 바로 Pod4, Pod5와 통신
  * ![](2024-12-29-01-02-13.png)

<br><br>

## kube-proxy
* kube-proxy는 클러스터의 각 노드에서 실행되는 네트워크 프록시로, 쿠버네티스의 **서비스 개념의 구현부**이다.
* kube-proxy는 **노드 레벨에서 트래픽을 처리**하고 라우팅 규칙을 설정하는 역할을 함
* kube-proxy는 보통 데몬셋 형태로 노드마다 하나씩 실행됨 (리눅스 프로세스 형태로 노드에 직접 설치도 가능하긴 함)
  * 아래의 대시보드 이미지를 보면 데몬셋으로 생성된 kube-proxy 파드 3개를 확인 가능함 (master노드, worker노드1, worker노드2 구성)
  * ![](2024-12-30-21-40-27.png)
* kube-proxy는 Service와 연결된 여러 Pod들의 **엔드포인트 목록을 관리**하는 역할을 함
* Service에 트래픽이 유입되면, 각 노드마다 존재하는 kube-proxy에 의해 **트래픽이 분산**됨
* 새로운 서비스나 엔드포인트가 추가되거나 삭제된다면 API 서버는 이러한 변화를 kube-proxy에게 전달함
* 전달해야 하는 요청의 엔드포인트가 해당하는 kube-proxy가 존재하는 노드에 없는 경우도 있는데
  * 이때는 CNI(ex.calico)를 통해 다른 노드로 패킷이 전달됨

<br><br>

## Service 관련 추가 내용
* Service의 설정은 Kubernetes API 서버에 저장되어 있음
* Service 자체는 물리적으로 특정 노드에 배치되지 않으며, **클러스터 전역에서 논리적으로 동작**
* Service는 클러스터 전역에서 동작하지만, 네임스페이스 기반으로 분리되어 있어 이름이 동일한 서비스라도 네임스페이스가 다르면 생성될 수 있음
* Service는 클러스터 내부에서 엔드포인트(ClusterIP 등)를 제공하는 역할을 함
* Service는 워크로드에 포함되지 않음 
  * 워크로드가 아니기 때문에 Deployment Controller 삭제시 함께 삭제되지 않음
  * ![](2024-12-29-02-13-15.png)
* 즉, Service는 Pod들의 로드 밸런싱을 지원하는 추상화된 개념이라고 볼 수 있음

<br><br>

## 트래픽 분산 과정 - Service와 kube-proxy
* `kube-proxy`는 각각의 노드마다 실행되며, Service로부터 들어오는 트래픽을 관련된 Pod(Endpoint)로 라우팅 되도록 함
* `NodePort` 타입의 Service의 경우, 클라이언트가 직접 `특정 노드`를 지정하거나 외부 로드 밸런서를 통해 `여러 노드`로 분산할 수 있음
* `LoadBalancer` 타입의 Service의 경우, Plugin으로 설치된 로드밸런서가 트래픽을 `여러 노드`로 분산시켜줌
* `ClusterIP` 타입의 Service의 경우에는, Service에 연결된 Pod들이 각각 `다른 노드`에 위치할 수 있음
  * 해당 요청은 각 노드마다 존재하는 kube-proxy에 의해 처리되는데,
  * ClusterIP Service에 들어오는 요청들의 자세한 처리 과정은 아래와 같음
    1. ClusterIP Service로 들어오는 요청은 클러스터 내의 어느 노드로든 전달될 수 있음
    2. 도달한 노드의 kube-proxy가 iptables 또는 IPVS 규칙을 사용해 적절한 Pod로 트래픽을 라우팅
    3. kube-proxy는 Service의 Endpoint 목록 중 하나를 선택하여 로드 밸런싱하며,
    4. **선택된 Pod가 다른 노드에 있다면**, CNI(ex.calico)를 통해 패킷이 전달됨

<br><br>

## Service - 중급편 내용
* [이전(section 4)](/k8s-basic/section-04/2.Service/service.md)에는 service를 통해 파드에 접근하는 방법을 다뤘음
* section 7 에서는 **파드 입장**에서 원하는 서비스(파드)에 접근하거나 외부 서비스에 접근하는 방법을 다룸
  * 파드에서 서비스로 접근 - `Headless` Service와 Cluster내의 DNS Server 이용
  * 파드에서 외부 서비스로 접근 - `ExternalName` Service 이용

<br>

### Headless Service
* Service 설정 정보에 `clusterIP:None` 옵션을 추가하면 Headless Service 생성 가능
  * Service의 ip를 생성하지 않겠다는 옵션
  * ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: headless1
    spec:
      selector:
        svc: headless
      ports:
        - port: 80
          targetPort: 8080    
      clusterIP: None
    ```
* EndPoint에 해당하는 파드의 설정에는 아래의 두 옵션을 추가해 주어야 함
  * `hostname: pod의 도메인`
  * `subdomain: service의 이름`
  * ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
      name: pod4
      labels:
        svc: headless
    spec:
      hostname: pod-a
      subdomain: headless1
      containers:
      - name: container
        image: kubetm/app
    ```
* **일반 Service**는 CoreDNS에 Service도메인에 Service IP가 매칭되지만
  * cf. 도메인 풀네임(FQDN - Fully Qualified Domain Name)으로 호출할 필요 없이 Service의 이름으로 Service 호출 가능함 - 아래 그림의 경우 `Service1`
  * ![](2024-12-30-22-21-08.png)
  * ![](2024-12-30-23-33-05.png)
* **Headless Service**는 Service의 IP가 없기 때문에 Service도메인에 Pod들의 IP가 매칭됨
  * cf. 파드의 도메인 풀네임으로 호출할 필요 없이 (파드 이름 + 서비스 이름)으로 파드 호출이 가능함 - 아래 그림의 경우 `pod4.headless1`, `pod5.headless1`
  * ![](2024-12-30-22-21-42.png)
  * ![](2024-12-30-23-44-20.png)

<br>

### Endpoint
* Endpoint는 Service와 연결된 Pod들의 IP주소 및 포트 또는 외부 IP주소를 나타냄
* Label을 통해 Service와 Pod를 연결시키면 Endpoint는 자동으로 생성됨
* Endpoint를 직접 생성할 수도 있음
  * Label없이 Endpoints를 직접 생성
  * 설정 정보에서 Endpoint의 이름은 Service와 동일하고 연결될 Pod의 IP주소를 적어주어야 함
  * Endpoint 생성하고 나서, 대시보드에서 해당하는 Service를 확인해 보면 Endpoint가 생성된 것을 확인 가능, 또는 `k get endpoints {이름}`으로 검색
    * Service 상세 정보에 연결된 파드는 따로 표시되지 않음
    * ex. endpoint2라는 이름의 clusterIP service 생성하고 라벨링 없이 Pod와 Endpoint 생성했을때, 대시보드에서 endpoint2 서비스의 상세정보 이미지
      * ![](2024-12-31-16-30-37.png)
    * cf. 위의 ex의 설정 정보 - Service, Pod, Endpoints 순서
    * ```yaml
      apiVersion: v1
      kind: Service
      metadata:
        name: endpoint2
      spec:
        ports:
        - port: 8080
      ```
    * ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: pod9
      spec:
        containers:
        - name: container
          image: kubetm/app
      ```
    * ```yaml
      apiVersion: v1
      kind: Endpoints
      metadata:
        name: endpoint2 # 서비스 이름과 동일하게게
      subsets:
      - addresses:
        - ip: 20.109.5.12 # Pod의 IP
        ports:
        - port: 8080
      ```
* 아래 그림에서 윗 부분은 Label을 통해 Endpoint가 생성된 모습을 표현, 아래 부분은 Endpoint를 직접 생성하여 Service와 Pod IP주소 또는 외부 IP주소와 연결시킨 모습
  * ![](2024-12-30-22-31-54.png)

<br>

### ExternalName
* Endpoint설명의 그림에서 Endpoint안에 Service와 연결된 IP의 정보가 들어가 있지만, Service의 설정에 ExternalName 속성을 추가하면 도메인 이름을 추가할 수 있음
* ExternalName 속성을 추가하면 Endpoint를 따로 생성해 주지 않아도 ExternalName에 도메인만 명시해 주면 Service를 통해 해당 도메인에 접근이 가능함
  * ExternalName속성에 추가한 도메인을 DNS cache가 내부와 외부 DNS server와 통신하여 IP를 조회해 줌
* 특정 Pod에서 ExternalName 속성이 추가된 Service를 이용하여 **클러스터 외부 도메인**에 접근할 수 있음
* Service에서 필요시마다 도메인 주소를 변경하여, Pod가 통신하고자 하는 목적지를 변경할 수 있음
  * Pod를 수정하고 재배포하지 않아도 됨
  * ex. externalname1 라는 서비스로 깃허브에 저장된 파일에 접근
  * `curl -O externalname1/kubetm/kubetm.github.io/blob/master/documents/intermediate/Service-Headless_ExternalName.pdf`
  * ```yaml
    apiVersion: v1
    kind: Service
    metadata:
      name: externalname1
    spec:
      type: ExternalName
      externalName: github.github.io
    ```
* ![](2024-12-30-22-41-41.png)

<br><br>

## 그림 설명 정리
* ![](2024-12-29-02-22-09.png)