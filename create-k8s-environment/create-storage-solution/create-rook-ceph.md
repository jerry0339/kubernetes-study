# Ceph

## 1. 사전 준비 사항
1. Rook-Ceph를 배포할 Kubernetes 클러스터
   * 최소 노드 3개 필요 - master 제외 worker 3개
2. 클러스터에 접근할 수 있는 kubectl 설치
   * k8s cluster 구축시 해결
3. Ceph의 데이터 저장을 위해 각 노드에 별도의 디스크 필요
   * 최소 노드 3개에 별도의 디스크 세팅으로 총 `3개 이상의 여분의 디스크` 필요
   * 여분의 디스크는 `사용 가능한 남은 용량을 나타내는 것이 아님`
   * `lsblk`, `sudo fdisk -l` 명령어를 통해 확인할 수 있는 여분의 disk임
   * 아래의 그림을 보면 `lsblk` 명령어 결과로 MOUNTPOINT가 빈 디스크가 여분의 디스크인데 sda의 경우 파티션이 존재하므로 여분이 아님.
   * `sudo fdisk -l` 명령어 결과를 보면 연결된 모든 디스크와 파티션에 대한 자세한 정보를 확인 가능함. 아래 이미지 확인 결과 여분의 디스크가 없음
   * ![](2025-03-25-03-02-23.png)

<br><br>

## 2. 추가 디스크 생성
* GCP의 vm으로 k8s cluster를 구축한 상황
* 여분의 디스크가 필요하므로 디스크를 추가
* 인스턴스 수정에서 디스크 추가가 가능함
* 모든 노드에 빈 디스크를 추가 (ceph-disk-00 ~ 03, 용량 10GB)
  * ![](2025-03-25-03-12-59.png)
  * 또는 명령어로 특정 디스크 데이터 초기화
    ```sh
    sudo wipefs -a /dev/sdb # 초기화할 디스크 경로 입력
    sudo sgdisk --zap-all /dev/sdb # 초기화할 디스크 경로 입력
    ```
* 추가로 인스턴스 삭제시 디스크도 삭제되도록 설정
  * ![](2025-03-25-03-21-33.png)
* `lsblk`과 `sudo fdisk -l` 명령어로 확인한 결과 빈 디스크인 sdb가 생성된 것을 확인 가능함
  * ![](2025-03-25-03-28-21.png)
  * ![](2025-03-25-03-28-49.png)

<br><br>

## 3. Rook-Ceph 버전 체크 및 Rook 설치 파일 다운로드
* k8s 버전과 호환되는 [rook 버전 확인](https://rook.io/docs/rook/v1.16/Getting-Started/quickstart/)
* [rook 문서](https://rook.io/docs/rook/latest-release/Upgrade/ceph-upgrade/)에서 rook버전과 호환되는 ceph버전을 확인
* ceph의 이미지 tag는 [quay의 ceph tag](https://quay.io/repository/ceph/ceph?tab=tags)에서 확인 가능
* Rook git 저장소 clone
  * rook 1.16.7 버전
  ```sh
  git clone --single-branch --branch v1.16.7 https://github.com/rook/rook.git
  cd rook/deploy/examples
  ```

<br><br>

## 4. CRD 및 Operator 배포
* `CRD(Custom Resource Definition)`: k8s의 사용자 정의 리소스
  * 아래에서 적용하는 CRD들은 Rook이 Ceph 클러스터를 관리하는데 사용되는 리소스에 해당
  ```sh
  cd rook/deploy/examples

  # CRDs(Custom Resource Definitions) 적용
  kubectl apply -f crds.yaml

  # Rook Operator 배포
  kubectl apply -f common.yaml
  kubectl apply -f operator.yaml
  ```

<br><br>

## 5. Ceph Cluster 설정 및 생성
* cluster.yaml 파일에서 사용하려는 노드와 디스크 구성을 확인하거나 필요한 경우 수정해야 하지만 일단 기본 설정으로 적용함
  ```sh
  cd rook/deploy/examples
  kubectl apply -f cluster.yaml
  ```
* `특정 노드` 또는 `특정 디스크`를 사용하고자 한다면 아래와 같이 세팅 가능
  * cephVersion을 spec에 적어주는 이유 (필수)
    * Rook이 관리하는 새로운 Ceph 클러스터를 생성할 때는 초기 cluster.yaml 파일의 spec 섹션 안에 cephVersion.image를 명시적으로 포함해야 함
    * 누락하면 Rook Operator가 Ceph 버전을 감지하지 못하여 Rook을 통해 Ceph Cluster에 새로운 자원을 배포하지 못함
    * 따라서 rook 버전과 호환되는 ceph버전을 찾아 입력해 주어야 함
      * `quay.io/ceph/ceph:v19.2.2` 사용함. `quay.io/`생략시 docker.io로 요청되니 조심
  ```yaml
  # cluster.yaml 수정
  spec:
    cephVersion:
      image: quay.io/ceph/ceph:v19.2.2 # Rook v1.16와 호환되는 Ceph 이미지 버전 v19.2
    storage:
      useAllNodes: false # default: true - 사용할 노드 지정시 false
      useAllDevices: false # default: true - 사용할 디스크 지정시 false
      nodes: # 아래와 같이 노드와 디스크를 지정해 주지 않으면 알아서 빈 디스크로 설정
      - name: "k8s-worker-01"
        devices:
        - name: "sdb"
      - name: "k8s-worker-02"
        devices:
        - name: "sdb"
      - name: "k8s-worker-03"
        devices:
        - name: "sdb"
  ```
* cluster.yaml 적용 이후 Pod 상태 점검
  * 모든 Pod가 정상 실행될 때까지 대기
  * `kubectl -n rook-ceph get pod`
  * ![](2025-04-17-17-17-45.png)

<br><br>

## 6. ToolBox 설치 및 Ceph 상태 확인
* toolbox 설치하여 Ceph 상태 확인
  ```sh
  cd rook/deploy/examples
  kubectl create -f toolbox.yaml
  kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status
  ```
* health ok 체크 및 OSD 3up 3in 확인
  * `OSD(Object Storage Daemon)?`
    * Ceph 클러스터에서 실제 데이터가 저장되고 관리되는 스토리지 노드 또는 그 노드에서 실행되는 데몬 프로세스
  * ![](2025-03-27-19-39-52.png)
* `8.rook-ceph cluster 및 PVC 점검`에서 사용법 확인

<br><br>

## 7. Storage class 생성
* StorageClass을 생성하려면 CephBlockPool(Custom Resource)를 먼저 생성해야 함
* general-rook-ceph-block이랑 stateful-rook-ceph-block 나눠서 생성함
  ```yaml
  # SSD 전용 풀 (StatefulSet용)
  apiVersion: ceph.rook.io/v1
  kind: CephBlockPool
  metadata:
    name: ssd-pool
    namespace: rook-ceph  # Rook-Ceph 설치 네임스페이스
  spec:
    replicated:
      size: 3             # size:3 설정하려면, 최소 3개의 OSD가 동작 중이어야 함
    failureDomain: host   # 데이터 복제본을 노드(호스트) 단위로 분산(ceph는 노드 장애 대비하여 데이터를 분산 저장함)
    deviceClass: ssd      # OSD 장치 클래스 지정
  ---
  # Stateful 전용 StorageClass
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: stateful-rook-ceph-block
  provisioner: rook-ceph.rbd.csi.ceph.com  # 필수 필드
  parameters:
    clusterID: rook-ceph                   # Rook-Ceph 네임스페이스
    pool: ssd-pool                         # 위에서 생성한 풀 이름
    imageFormat: "2"                       # RBD 이미지 포맷
    imageFeatures: layering,exclusive-lock # exclusive-lock - 동시 쓰기 방지로 데이터 일관성 보장
    csi.storage.k8s.io/fstype: ext4        # 파일시스템 타입
  reclaimPolicy: Retain                    # Retain - PVC가 삭제되어도 PV와 데이터는 그대로 유지(상태저장 필요한 경우)
  allowVolumeExpansion: true               # 디스크 확장이 필요한 경우 또는 PVC가 더 큰 용량 요청 - PVC 크기 변경 가능
  volumeBindingMode: WaitForFirstConsumer  # volume을 특정 노드에 고정
  ```
  ```yaml
  # 일반 풀 (ReplicaSet용)
  apiVersion: ceph.rook.io/v1
  kind: CephBlockPool
  metadata:
    name: general-pool
    namespace: rook-ceph
  spec:
    replicated:
      size: 3
    failureDomain: host
  ---
  # 일반 용도 StorageClass
  apiVersion: storage.k8s.io/v1
  kind: StorageClass
  metadata:
    name: general-rook-ceph-block
  provisioner: rook-ceph.rbd.csi.ceph.com
  parameters:
    clusterID: rook-ceph
    pool: general-pool
    imageFormat: "2"
    imageFeatures: layering,exclusive-lock
    csi.storage.k8s.io/fstype: ext4
  reclaimPolicy: Delete
  allowVolumeExpansion: true
  volumeBindingMode: WaitForFirstConsumer
  ```

<br><br>

## 8. PVC 생성
* **PV가 필요한 Pod가 있을때** 해당하는 namespace에 pvc를 생성해 주면 됨
* 아래의 yaml내용에 대한 pvc 생성
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: stateful-rook-ceph-pvc
  spec:
    storageClassName: stateful-rook-ceph-block
    accessModes:
    - ReadWriteOnce # 단일 노드에서만 PV를 읽고 쓸 수 있도록 제한
    resources:
      requests:
        storage: 5Gi
  ```
  ```yaml
  apiVersion: v1
  kind: PersistentVolumeClaim
  metadata:
    name: general-rook-ceph-pvc
  spec:
    storageClassName: general-rook-ceph-block
    accessModes:
    - ReadWriteOnce
    resources:
      requests:
        storage: 5Gi
  ```
* PVC 적용 확인 - `kubectl get pvc`

<br><br>

## 9. Rook-Ceph Cluster 및 PVC 점검
* storageclass, pvc 및 파드 기본 점검
  * pod는 mon파드 확인
    * `mon:` Ceph 모니터(MON) 데몬, Ceph 클러스터를 관리하는 데몬셋 pod임
  ```sh
  kubectl get pods -n rook-ceph
  kubectl get cephcluster -n rook-ceph
  kubectl get storageclass
  
  kubectl get pv
  kubectl get pvc
  kubectl get sc rook-ceph-block -o yaml
  ```

<br>

### ceph 점검을 위한 명령어 사용법
* 위에서 설치한 toolbox(rook-ceph-tool)를 이용하여 ceph cluster를 점검할 수 있다.
  * 아래의 명령어 입력하여 ceph명령어를 사용할 수 있다.
    * `kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- bash`
    * ![](2025-04-14-02-54-32.png)
  * 또는 `kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- `명령어에 ceph 명령어를 추가하여 사용할 수 있음
    * ex. kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph df
    * ![](2025-04-14-02-54-47.png)
* ceph 명령어
  * `ceph health detail`: 클러스터의 상세 경고 메시지를 제공
  * `ceph -s`: 클러스터의 전체 상태를 요약
  * `ceph df`: 전체 스토리지 사용량 확인
  * `ceph osd df`: 각 OSD별 상세 사용량 확인
  * OSD 상태 점검
    * `ceph osd status`
    * `ceph osd df`

<br><br>

## 10. cf. Rook-Ceph 삭제 방법
```sh
kubectl -n rook-ceph patch cephcluster rook-ceph --type merge \
  -p '{"spec":{"cleanupPolicy":{"confirmation":"yes-really-destroy-data"}}}'

kubectl -n rook-ceph patch cephcluster/rook-ceph --type json --patch='[ { "op": "remove", "path": "/metadata/finalizers" } ]'

cd rook/deploy/examples
kubectl delete -f operator.yaml
kubectl delete -f cluster.yaml
kubectl delete -f common.yaml

for crd in $(kubectl get crd | grep ceph | awk '{print $1}'); do
  kubectl get crd $crd -o json | jq '.metadata.finalizers=[]' | kubectl replace --raw "/apis/apiextensions.k8s.io/v1/customresourcedefinitions/$crd" -f -
  kubectl delete crd $crd
done

kubectl -n rook-ceph delete cephcluster rook-ceph
kubectl delete crd cephclusters.ceph.rook.io cephblockpools.ceph.rook.io

kubectl delete namespace rook-ceph

# 삭제 확인
kubectl api-resources --verbs=list --namespaced -o name | xargs -n 1 kubectl get --show-kind --ignore-not-found -n rook-ceph

# 그래도 삭제 안된것이 있으면 해당 리소스 설정에서 finalizers: [] 로 변경해야 함
k edit configmap rook-ceph-mon -n rook-ceph
k edit secret rook-ceph-mon -n rook-ceph

# 이후 각 노드마다 수행
# 파일 생성해서 실행 - 노드마다 디스크 초기화(예: /dev/sdb 사용 시)
cat > initdisk.sh
DISK="/dev/sdb"
sudo sgdisk --zap-all $DISK
sudo dd if=/dev/zero of=$DISK bs=1M count=100 oflag=direct,dsync
sudo wipefs -a /dev/sdb
sudo rm -rf /var/lib/rook/*
sudo partprobe $DISK
# ctrl + d

chmod +x initdisk.sh

./initdisk.sh
```