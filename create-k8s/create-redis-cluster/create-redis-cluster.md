# Redis Cluster 구축
* [참고 링크](https://jeongchul.tistory.com/725)

<br>

## 1. Redis Cluster helm repo 설치
```sh
helm repo add bitnami https://charts.bitnami.com/bitnami
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

    # values.yaml
    global:
      # ...
      defaultStorageClass: rook-ceph-block # storage solution 후 생성한 Storage Class의 이름
      storageClass: rook-ceph-block
      redis:
        password: "{your-redis-password}"
    # ...
    persistence:
      # ...
      storageClass: rook-ceph-block
      size: "5Gi"
    ```
* configmap.yaml 파일 수정
    ```sh
    vi templates/configmap.yaml

    # configmap.yaml
    # 용량이 8000M 넘어갈시 volatile-lr 정책에 의해 데이터 삭제 옵션, default설정은 용량 넘어갈시 에러발생임
    maxmemory 8000M
    maxmemory-policy volatile-lr
    ```

<br><br>

## 3. Redis Cluster 설치
```sh
helm install redis-cluster . --namespace redis --create-namespace
```

<br><br>

## 4. Todo: PVC 생성