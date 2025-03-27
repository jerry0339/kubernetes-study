# Redis Cluster 구축
* [참고 링크](https://jeongchul.tistory.com/725)

<br>

## 1. Redis Cluster helm repo 설치
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update
helm fetch bitnami/redis-cluster --untar
cd redis-cluster
ls # 설치 점검
```

<br><br>

## 2. Redis Cluster 설정 파일 수정
* values.yaml 파일 수정
  ```sh
  cd redis-cluster
  vi values.yaml
  ```
  ```yaml
  # redis-values.yaml 주요 설정
  global:
    storageClass: "rook-ceph-block"  # 생성한 PVC의 StorageClass 이름
    redis:
      password: "your_secure_password"

  cluster:
    nodes: 6  # 최소 3 Masters + 3 Replicas
    replicas: 1

  persistence:
    enabled: true
    size: 8Gi  # PVC 크기와 일치
    storageClass: "rook-ceph-block"

  service:
    type: ClusterIP # default는 Loadbalancer 로 되어 있음 - ingress 추가시 변경해야 할듯
    ports:
      redis: 6379
  ```
* configmap.yaml 파일 수정 (선택)
  ```sh
  vi templates/configmap.yaml

  # configmap.yaml
  # 용량이 8000M 넘어갈시 volatile-lr 정책에 의해 데이터 삭제 옵션, default설정은 용량 넘어갈시 에러발생임
  maxmemory 8000M
  maxmemory-policy volatile-lr
  ```

<br><br>

## 3. Helm으로 Redis Cluster 설치
```sh
helm install redis-cluster bitnami/redis-cluster \
  --namespace redis \
  --create-namespace \
  -f values.yaml
```

<br><br>

## 4. Redis cluster 상태 및 Pods 상태 점검
```sh
kubectl get pods -n redis # 파드 상태 점검
kubectl exec -it redis-cluster-0 -n redis -- \
  redis-cli --cluster check 127.0.0.1:6379 -a {redis_password}
```
* ![](2025-03-27-20-26-58.png)

<br><br>

## 5. Redis Cluster Headless 서비스 접근 하기, Headless Service 개념
* 다른 Service Pod에서 Redis Cluster 스테이트풀 셋으로 생성된 Redis Pod에 접근하는 방법?
  * Headless Service를 통하여 Pod에서 Pod로 접근이 가능
  * ![](2025-03-27-23-48-53.png)

<br>

### 1. Headless Service 생성 (개념 확인용)
* 위의 4번까지의 내용을 잘 수행했다면 `redis-cluster-headless`라는 Headless 서비스가 생성되었을 것이므로 **Headless Service를 추가적으로 생성할 필요가 없음!**
* 먼저 redis-cluster 스테이트풀 셋의 `spec.serviceName`과 `spec.template.metadata.labels`중 하나(instance로)를 확인
* 위에서 확인한, 같은 이름의 serviceName과 확인한 label을 selector로 지정하여 Headless 서비스를 생성해 주어야 한다.
  * ![](2025-03-27-23-14-16.png)
  * ![](2025-03-27-23-25-10.png)
* 위 내용 확인하여 아래 예시와 같은 설정 정보로 Headless service 생성
  ```yaml
  apiVersion: v1
  kind: Service
  metadata:
    name: redis-cluster-headless # Redis Cluster StatefulSet에 설정된 serviceName와 일치해야 함
    namespace: redis  # Redis Cluster가 배포된 네임스페이스
    labels:
      app.kubernetes.io/name: redis-cluster
  spec:
    clusterIP: None  # Headless Service로 설정
    selector:
      app.kubernetes.io/instance: redis-cluster  # StatefulSet의 Label과 일치해야 함
    ports:
      - name: redis
        port: 6379
        targetPort: 6379
  ```

### 2. Headless Service로 Redis Pod에 접근하기
* Headless Service가 등록되어 있다면 FQDN(Fully Qualified Domain Name)에 따라 아래와 같은 도메인으로 등록됨
  * `{Headless-Service-name}.{namespace}.svc.cluster.local`
  * 따라서, redis 네임스페이스에 redis-cluster-headless 이름으로 생성했다면, 
    * redis-cluster-headless.redis.svc.cluster.local으로 coreDNS(k8s service discovery)에 service에 대한 도메인이 등록됨
  * CoreDNS는 Pod로 생성되어 클러스터내에 위치해 있기 때문에, 클러스터 네트워크 대역에 해당하는 출발지(즉, 같은 대역폭)에서 `nslookup` 명령어로 확인 가능함
  * ![](2025-03-27-23-53-15.png)
* StatefulSet으로 생성된 Pod에도 도메인을 통해 바로 접근이 가능함
  * `{Pod-Name},{Headless-Service-Name}.{Namespace}.svc.cluster.local` - ex. redis-cluster-0.redis-cluster-headless.redis.svc.cluster.local
* Pod의 도메인을 이용하여 Spring 애플리케이션

<br><br>

## 6. Redis CLI 접속
* redis-tool을 설치하면 접속 가능
  ```sh
  sudo apt update
  sudo apt install redis-tools # CLI 사용을 위한 redis-tools 설치
  kubectl exec -it {pod-name} -n redis -- redis-cli -a {redis-password}
  # ex. kubectl exec -it redis-cluster-0 -n redis -- redis-cli -a 1234
  ```
* `MOVED 오류`
  * redis cluster라서 저장하려는 키가 현재 연결된 노드가 아닌 다른 노드에 할당된 슬롯일 수 있음
  * Redis Cluster는 데이터를 16384개의 해시 슬롯으로 분산 저장하며, 각 슬롯은 특정 마스터 노드에서 관리하기 때문
  * ![](2025-03-28-02-33-26.png)
  * 따라서 해당하는 redis 노드의 pod로 다시 이동후 명령어 수행 가능함
  * ![](2025-03-28-02-37-18.png)
