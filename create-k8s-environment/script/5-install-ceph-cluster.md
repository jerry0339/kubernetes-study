## 사전 작업
* storageClass 정의 필요 (retain 설정 필수)

<br>

## 스크립트
```sh
#!/bin/bash
set -e

echo "🔧 Ceph Cluster 설치중..."
git clone --single-branch --branch v1.16.7 https://github.com/rook/rook.git
cd rook/deploy/examples

# CRD 및 Operator 설치
kubectl apply -f common.yaml
kubectl apply -f crds.yaml
kubectl apply -f operator.yaml

echo "⏳ rook-ceph-operator 준비 대기 중..."
kubectl wait --for=condition=Available -n rook-ceph deploy/rook-ceph-operator --timeout=180s

kubectl apply -f - <<EOF
apiVersion: ceph.rook.io/v1
kind: CephCluster
metadata:
  name: rook-ceph
  namespace: rook-ceph
spec:
  cephVersion:
    image: quay.io/ceph/ceph:v19.2.2
  dataDirHostPath: /var/lib/rook
  storage:
    useAllNodes: false
    useAllDevices: false
    nodes:
    - name: "k8s-worker-01.us-central1-c.c.nice-theater-462514-k6.internal"
      devices:
      - name: "sdb"
    - name: "k8s-worker-02.us-central1-c.c.nice-theater-462514-k6.internal"
      devices:
      - name: "sdb"
    - name: "k8s-worker-03.us-central1-c.c.nice-theater-462514-k6.internal"
      devices:
      - name: "sdb"
EOF

kubectl apply -f toolbox.yaml

echo "⏳ rook-ceph-tools Pod 준비 중..."
kubectl wait --for=condition=Ready pod -l app=rook-ceph-tools -n rook-ceph --timeout=180s

kubectl apply -f - <<EOF
# SSD 전용 풀 (StatefulSet용)
apiVersion: ceph.rook.io/v1
kind: CephBlockPool
metadata:
  name: ssd-pool
  namespace: rook-ceph
spec:
  replicated:
    size: 3
  failureDomain: host
  deviceClass: ssd
---
# Stateful 전용 StorageClass
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: stateful-rook-ceph-block
provisioner: rook-ceph.rbd.csi.ceph.com
parameters:
  clusterID: rook-ceph
  pool: ssd-pool
  imageFormat: "2"
  imageFeatures: layering,exclusive-lock
  csi.storage.k8s.io/fstype: xfs
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Retain
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
---
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
  csi.storage.k8s.io/fstype: xfs
  csi.storage.k8s.io/provisioner-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/provisioner-secret-namespace: rook-ceph
  csi.storage.k8s.io/controller-expand-secret-name: rook-csi-rbd-provisioner
  csi.storage.k8s.io/controller-expand-secret-namespace: rook-ceph
  csi.storage.k8s.io/node-stage-secret-name: rook-csi-rbd-node
  csi.storage.k8s.io/node-stage-secret-namespace: rook-ceph
reclaimPolicy: Delete
allowVolumeExpansion: true
volumeBindingMode: WaitForFirstConsumer
EOF

# default StorageClass를 general-rook-ceph-block로 설정
kubectl patch storageclass general-rook-ceph-block -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"true"}}}'

kubectl -n rook-ceph exec -it deploy/rook-ceph-tools -- ceph status

echo "✅ Ceph Cluster 설치 완료"
```