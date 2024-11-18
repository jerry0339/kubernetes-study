# Section 6
* Pod 중급편
* Pod의 Lifecycle에 대해

## Pod의 `Status`
* `Phase` 5가지 - Pod의 전체 상태를 대표하는 속성
  * Pending
  * Running
  * Succeeded
  * Failed
  * Unknown
* `Conditions` 4가지 - Pod가 생성되면서 실행하는 단계와 속성
  * Initialized
  * ContainerReady
  * PodScheduled
  * Read
  * `Conditions의 Reason` 2가지 - Conditions의 세부 내용
    * ContainerNotReady
    * PodCompleted
    * cf. 아래의 이미지에서 status가 False인 이유로 ContainerNotReady가 표기됨

## Container의 `ContainerStatus`
* Pod의 컨테이너 마다 `ContainerStatus`가 있음
* ContainerStatus `State` 3가지 - 컨테이너의 상태
  * Waiting
  * Running
  * Terminated
  * `ContainerStatus의 Reason`4가지 - State에 대한 세부 내용
    * ContainerCreating
    * CrashLoopBackOff
    * Error
    * Completed
    * cf. 아래의 이미지에서 상태가 waiting인 이유로 ContainerCreating이 표기됨

## 이미지 설명
* ![](2024-11-18-23-39-42.png)