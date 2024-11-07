# Section 5
* [Controller 공식문서](https://kubernetes.io/ko/docs/concepts/architecture/controller/)

## Controller란
* 공식 문서 출처 - k8s의 Controller란 클러스터의 상태를 관찰 한 다음, 필요한 경우에 생성 또는 변경을 요청하는 **컨트롤 루프**이다.
  * 컨트롤 루프: 시스템 상태를 조절하는 종료되지 않는 루프 (ex. 실내 온도 조절기)
  * 각 컨트롤러는 현재 클러스터 상태를 모니터링하며 의도한 상태와 차이가 생겼을 경우,
  * 생성 또는 변경을 요청을 하여 현재 상태를 의도한 상태에 가깝게 이동시킨다.
* Controller는 워크로드 리소스에 정의된 상태를 지속적으로 모니터링하고, 클러스터 상태가 변할 경우 이를 자동으로 조정하여 안정성을 유지한다.
  * 워크로드? - 워크로드 리소스는 실행하고자 하는 애플리케이션의 원하는 상태를 정의하는 설정
  * 컨트롤러? - 컨트롤러는 워크로드 리소스 설정에 따라 실제 상태가 원하는 상태와 일치하도록 관리하는 자동화된 메커니즘
* 서비스를 관리하고 운영하는데 도움을 준다.
  * Auto Healing
    * Controller가 장애가 생긴 Pod를 감지하고 재생성
    * ex. Pod가 down되거나 Node가 down될 경우 다른 노드에 Pod를 새로 할당
  * Auto Scaling
    * Pod의 자원이 limit 상태에 도달한 경우 Controller가 이를 감지하고 Replicas를 증가시켜 Pod의 수를 늘림
  * Software Update
    * deployment 워크로드 - ReCreate, RollingUpdate, Rollback
  * Job
    * 일시적인 작업 필요한 경우, Controller가 Pod를 생성하여 해당 작업을 수행하고 Pod삭제
    * 일시적으로 자원을 사용하고 반환하기 때문에 효율적임
    * cf. CronJob 컨트롤러는 주기적으로 실행되어야 하는 작업을 예약하고 관리
  * Replication Management (복제 관리)
    * replicas에 지정된 수의 Pod를 유지
    * ex. ReplicaSet
  * Configuration Drift Management (구성 드리프트 관리)
    * Controller는 사용자가 정의한 원하는 상태와 일치하도록 클러스터의 구성과 상태를 지속적으로 조정
    * 예상치 못한 구성 변화를 방지

## Controller 역할 그림 설명
* ![](2024-11-05-21-26-48.png)

