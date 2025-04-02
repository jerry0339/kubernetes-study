# 클라우드 플랫폼(GCP) 이용하여 k8s 클러스터 구축하기
* 구글 클라우드 플랫폼(`GCP`)를 이용하여 k8s 클러스터 구축
  * AWS는 free tier 사용시 12개월 무료이지만 ec2 성능이 제한적(1 cpu, 1GB mem)이라 무료로 사용하기에 어려움이 있음
  * GCP는 90일 기준 300달러만큼 사용 가능해서 VM의 성능을 원하는 대로 구성 가능함
* GCP VM 정보
  * OS - `Ubuntu` 20.04.6 LTS
  * master-node용 VM - 1대
    * e2-medium - cpu2개, core1개, 4GB-memory
  * worker-node용 VM - 3대
    * e2-small - cpu2개, core1개, 2GB-memory
* k8s 버전 - [`1.30.7-1.1`](https://kubernetes.io/ko/releases/#release-v1-30)
  * 1.30.7 - 2024-11-19에 릴리스됨
  * 1.30 지원 종료 - 2025-06-28
* 네트워크 솔루션 - `Calico`
* k8s cluster 노드 구성
  * Master Node 1개
  * Worker Node 3개

<br>

## 참고 링크
* [k8s 1.30 공식 문서](https://v1-30.docs.kubernetes.io/docs/setup/production-environment/tools/kubeadm/install-kubeadm/)
* [dashboard 배포 공식 문서](https://kubernetes.io/ko/docs/tasks/access-application-cluster/web-ui-dashboard/)
* https://jongsky.tistory.com/112
* https://jbground.tistory.com/107
* https://jongsky.tistory.com/113

<br>

## 클러스터 구축
* [명령어 모음 링크](/create-k8s-cluster/used-script.md)

<br>

## 대시보드 구축
* [명령어 모음 링크](/create-k8s-dashboard/used-script.md)
