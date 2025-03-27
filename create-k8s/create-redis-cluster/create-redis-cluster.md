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