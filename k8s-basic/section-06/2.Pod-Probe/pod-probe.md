# Pod - Probe
* Probe는 컨테이너에서 kubelet에 의해 주기적으로 수행되는 진단
* kubelet은 컨테이너의 상태를 진단하기 위해 Handler를 호출
* Probe를 통해 컨테이너의 상태를 주기적으로 체크
  * 문제가 있는 컨테이너를 자동으로 재시작 하거나
  * 문제가 있는 컨테이너를 서비스에서 제외할 수 있음
* [공식 문서 참고](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/)

<br><br>

## Probe의 종류
* Readiness probe
  * 컨테이너가 요청을 처리할 준비가 되었는지 여부를 나타냄
  * Pod가 Running 상태여도 처음에 App이 로딩하는 시간이 있기 때문에 이 시간 동안은 App에 요청하려고 하면 오류가 발생
  * Readiness probe는 App이 구동되기 전까지 Service와 연결되지 않게 해줌
  * Probe가 실패하면 엔드포인트 컨트롤러는 파드에 연관된 모든 서비스들의 엔드포인트(서비스와 연결된 Pod) 목록에서 해당 파드의 IP를 제거함
  * App 구동 순간에 트래픽 실패를 없애줌
* Liveness probe
  * 컨테이너가 동작 중인지 여부를 나타냄
  * Pod은 정상적인 Running 상태이지만, 컨테이너의 App에 장애가 생겨서 응답이 안되는 경우를 감지함
  * Probe가 실패하면 kubelet은 컨테이너를 죽이고 [컨테이너를 재시작](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 함
  * App 장애시 지속적인 트래픽 실패를 없애줌
* Startup probe
  * 컨테이너 내의 App이 시작되었는지를 나타냄
  * startup probe가 주어진 경우, 성공할 때 까지 다른 나머지 probe는 활성화 되지 않는다.
    * 즉, startupProbe가 OK되기 전에는 readinessProbe와 livenessProbe가 동작하지 않음
  * App 기동이 오래 걸리는 상황일때 livenessProbe가 체크가되면 무한정 pod restart를 하게되고, 
    * readinessProbe역시 App기동이 되기 전에 활성화가 되면 요청이 실패함
    * 이러한 상황에서 Startup probe를 사용하면 좀 더 안정적으로 Probe 세팅이 가능
  * startup probe가 실패하면, kubelet이 컨테이너를 죽이고 [컨테이너를 재시작](https://kubernetes.io/ko/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy) 함
* ![](2024-11-20-23-21-57.png)

<br>

### Probe fail일때 Liveness probe vs Readiness probe
* Liveness Probe는 probe 핸들러 조건 아래 fail이 나면 pod를 재실행 시키지만,
* Readiness Probe는 probe 핸들러 조건 아래 fail이 나면 pod를 서비스로부터 제외
  * 서비스들의 엔드포인트 목록에서 해당 Pod의 IP가 제거됨

<br>

### ReadinessProbe와 LivenessProbe의 동작 그림 설명
* ![](2024-11-21-00-05-24.png)


<br><br>

## Probe의 Handler
* 컨테이너의 상태를 진단하기 위해 어떻게 진단할 것인지 명시한 것이 Handler
* Pod의 spec에 Probe추가시, 아래의 3가지 Handler중 하나를 반드시 포함해야 함
  * `Exec`
    * Command - 실행할 명령어 배열
  * `TcpSocket`
    * Port - 포트 번호
    * Host - 호스트명 (선택 사항)
  * `HttpGet`
    * Port - 포트 번호
    * Host - 호스트명 (선택 사항)
    * Path - HTTP 요청 경로
    * HttpHeader - 추가 HTTP 헤더 목록 (선택 사항)
    * Scheme - HTTP 또는 HTTPS 중 선택

<br>

### Probe Handler의 공통 옵션
* initialDelaySeconds
  * 첫 Probe 실행 전 대기 시간
  * default - 0초
* periodSeconds
  * Probe 실행 주기
  * default - 10초
* timeoutSeconds
  * Probe 응답 대기 시간
  * default - 1초
* successThreshold
  * 성공으로 간주되기 전 연속 성공 횟수
  * default - 1회
  * Readiness Probe에서 설정 가능
    * Liveness와 Startup Probe에서는 항상 1로 고정
* failureThreshold
  * 실패로 간주되기 전 연속 실패 횟수
  * default - 3회

<br>

### Probe 별 옵션 지원 여부
* **successThreshold**는 Readiness Probe에서만 유의미하게 작동하며, Liveness와 Startup Probe에서는 항상 1로 고정
* **Startup Probe**가 정의된 경우, 해당 Probe가 성공할 때까지 **Liveness Probe**는 비활성화
* **terminationGracePeriodSeconds**는 PodSpec에서 설정하는 옵션으로, Probe 자체의 구성 옵션은 아님
  * Graceful shutdown을 위한 옵션으로 컨테이너가 종료되기 전에 대기해야 하는 시간을 지정
  * default:30
  * `K8s 1.25` 버전부터는 Liveness Probe에 대해 개별적인 terminationGracePeriodSeconds를 설정할 수 있음
  * ```yaml
    spec:
      terminationGracePeriodSeconds: 3600  # pod-level
      containers:
      - name: test
        image: ...

        ports:
        - name: liveness-port
          containerPort: 8080

        livenessProbe:
          httpGet:
            path: /healthz
            port: liveness-port
          failureThreshold: 1
          periodSeconds: 60
          # Override pod-level terminationGracePeriodSeconds #
          terminationGracePeriodSeconds: 60
    ```

|         옵션 / Probe          | Readiness Probe | Liveness Probe | Startup Probe |
| :---------------------------: | :-------------: | :------------: | :-----------: |
|      initialDelaySeconds      |        ✅        |       ✅        |       ✅       |
|         periodSeconds         |        ✅        |       ✅        |       ✅       |
|        timeoutSeconds         |        ✅        |       ✅        |       ✅       |
|       successThreshold        |        ✅        |       ❌        |       ❌       |
|       failureThreshold        |        ✅        |       ✅        |       ✅       |
| terminationGracePeriodSeconds |        ❌        |       ✅        |       ❌       |

<br>

### Handler 그림 설명
* ![](2024-11-21-00-07-54.png)

