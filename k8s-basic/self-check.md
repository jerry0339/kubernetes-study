# Self Check
* 공부하면서 헷갈렸던 내용들

## 질문 모음
1. Pod에 접근하려고 할때 Cluster내에서 Pod의 IP를 통해서 접근이 가능한데 Service의 IP를 통해서 Pod에 접근하는 이유가 무엇?
   * 답: Pod를 지웠다 다시 생성하거나 장애로 인해 Pod가 재생성되는 경우 IP가 달라지기 때문에 Service를 통해 접근하면 항상 접근 가능
2. OX: 같은 pod에 있는 다른 container에는 localhost로 접근이 가능한가?
   * 답: O 가능함
   * cf. 다른 container여도 같은 pod 안에서 같은 port는 사용할 수 없다. 
3. OX: `HostPath Volume`은 주로 다른 호스트(node)에 배치된 Pod 및 Container에게 공유 디렉토리를 제공할 목적으로 사용한다.
   * 답: X, 어떤 노드에 배치된 Pod가 다른 노드의 volume을 접근하기 위해서는 개발자가 직접 노드끼리의 mount설정을 해주어야 한다.
   * cf. Pod 재생성시 기존과 다른 노드에 배치될 수 있기 때문에 HostPath Volume을 Pod의 데이터를 저장하기 위한 공간으로 사용하는 것은 적절하지 않다.
4. EmptyDir Volume과 HostPath Volume의 차이
   * 노드 의존적, 데이터 지속성 <-> Pod 내에서만 데이터 공유, 데이터 지속성 없음
   * c.f. EmptyDir / HostPath Volume 정의 참고
     * Pod가 생성될 때 빈 디렉토리를 제공하는 Volume 타입, Pod내의 컨테이너들끼리 데이터를 공유하기 위한 Volume
     * Node의 파일 시스템 path를 Pod에 mount하여 사용하는 volume 타입, 같은 호스트에 배치된 Pod 및 컨테이너들끼리 데이터를 공유하기 위한 Volume
5. OX: secret은 생성시 쿠버네티스 클러스터 DB(etcd)에 암호화되지 않은 상태로 저장된다.
   * 답: O, secret은 기본적으로 etcd에 평문 상태로 저장된다.
   * c.f. Base64 인코딩은 난독화된 것이지 암호화된 것이 아니다.
6. Secret오브젝트가 ConfigMap보다 보안에 유리한 이유가 무엇일까?
   * 해당 노드의 파드가 필요로 하는 경우에만 시크릿이 노드로 전송된다.
   * 시크릿을 파드 내부로 마운트할 때, 기밀 데이터가 보존적인(durable) 저장소에 기록되지 않도록 하기 위해 kubelet이 데이터 복제본을 tmpfs에 저장한다.
   * 시크릿을 사용하는 파드가 삭제되면, kubelet은 시크릿에 있던 기밀 데이터의 로컬 복사본을 삭제
   * [시크릿을 위한 정보 보안](https://kubernetes.io/ko/docs/concepts/configuration/secret/) 참고
7. OX: secret을 생성할때 value데이터는 반드시 Base64로 인코딩된 값을 입력해 주어야 한다.
   * 답: X, base64 인코딩 수행을 원하지 않는다면, 그 대신 stringData 필드를 사용할 수 있다.
   * <details>
     <summary><b>예시</b></summary>
     <div markdown="1">

     ~~~yaml
     apiVersion: v1
     kind: Secret
     metadata:
         name: bootstrap-token-5emitj
         namespace: kube-system
     type: bootstrap.kubernetes.io/token
     data: # value를 Base64인코딩할 경우
         auth-extra-groups: c3lzdGVtOmJvb3RzdHJhcHBlcnM6a3ViZWFkbTpkZWZhdWx0LW5vZGUtdG9rZW4=
         expiration: MjAyMC0wOS0xM1QwNDozOToxMFo=
         token-id: NWVtaXRq
         token-secret: a3E0Z2lodnN6emduMXAwcg==
         usage-bootstrap-authentication: dHJ1ZQ==
         usage-bootstrap-signing: dHJ1ZQ==
     ~~~
     ~~~yaml
     apiVersion: v1
     kind: Secret
     metadata:
         name: bootstrap-token-5emitj
         namespace: kube-system
     type: bootstrap.kubernetes.io/token
     stringData: # value로 stringData 사용시
         auth-extra-groups: "system:bootstrappers:kubeadm:default-node-token"
         expiration: "2020-09-13T04:39:10Z"
         token-id: "5emitj"
         token-secret: "kq4gihvszzgn1p0r"
         usage-bootstrap-authentication: "true"
         usage-bootstrap-signing: "true"
     ~~~

     </div>
     </details>
8. Pod에서 환경 변수 설정으로 ConfigMap을 컨테이너로 가져왔을때와 mount로 가져올때의 차이점은?
9. OX: Namespace를 삭제하면 그 안의 오브젝트들(service, pod등)은 모두 삭제된다.
   * O: 모두 삭제됨
10. ResourceQuota를 설정한 Namespace에 Spec설정(Compute Resource, Object Count)을 하지 않은 Pod를 생성할 수 있을까 ?
    * O: Namespace에 default 설정을 해주면 가능, 없으면 안됨
11. namespace에 resource quota 를 적용했을 때 어떤 동작에 제한이 있고, namespace에 limit range를 적용했을 때 어떤 동작에 제한이 있음?
    * Namespace에 Resource Quota 를 설정해 한 Namespace 내 할당할 수 있는 자원의 양을 정할 수 있다
    * Namespace에 LimitRange를 설정해 해당 namespace에 할당될 수 있는 한 pod의 최대 자원량을 정할 수 있다
12. OX: Namespace로 NodePort Service를 구분할 수 있다 (같은 port의 NodePort Service를 Namespace별로 만들 수 있다)
    * X: Namespace와는 별개로 Cluster내에 같은 port를 사용하는 NodePort Service는 생성할 수 없음
13. Replication Controller과 ReplicaSet 차이점?
    * [replicaSet문서의 Replication Controller과 ReplicaSet 차이점 참고](/kubernetes-study/k8s-basic/section-05/1.ReplicaSet/replicaSet.md)
14. OX: Blue/Green 배포 방식에 Deployment 워크로드는 필요없다.
    * O: 새로운 template이 담긴 ReplicaSet을 추가하고 Service의 Label만 바꿔주면 된다.
15. OX: Deployment Controller의 image를 변경하면 항상 새로운 ReplicaSet이 생성된다.
    * X: Revision History에 image가 같은 Revision ReplicaSet이 있다면, Deployment는 해당 ReplicaSet과 연결된다.
16. OX: Deployment Controller를 삭제하면 해당 Deployment와 연결된 모든 ReplicaSet, Pod, Service가 삭제된다.
    * X: 워크로드들만 삭제된다. (Pod, ReplicaSet, Job 등)
17. Rolling Update시 새로운 ReplicaSet이 생성될때 기존의 Pod와 연결되지 않는 이유?
    * [deployment문서 참고](/kubernetes-study/k8s-basic/section-05/2.Deployment/deployment.md)
18. Readiness probe와 Liveness probe가 실패시 동작 차이점
    * Liveness Probe는 probe 핸들러 조건 아래 fail이 나면 pod를 재실행 시키지만,
    * Readiness Probe는 probe 핸들러 조건 아래 fail이 나면 pod를 서비스로부터 제외
      * 서비스들의 엔드포인트 목록에서 해당 Pod의 IP가 제거됨
19. podAffinity 설정한 Pod의 matchExpressions에 해당되는 파드가 없다면 어떻게 될까?
    * 파드가 생성되지 않음 (pending 상태)
    * podAffinity 설정한 Pod가 생성된 이후에, matchExpressions에 해당되는 파드가 생성된다면 어떻게 될까? (예시 참고)
    * <details>
      <summary><b>예시</b></summary>
      <div markdown="1">

      k8s-worker-01에 `a-team = 1` 라벨, <br>
      k8s-worker-02에 `a-team = 2` 라벨이 할당된 상황 <br>

      아래의 설정 정보로 server2 파드를 생성하면, <br>
      `a-team`을 key로 가지는 노드에 할당된 <br>
      `key=web2`라벨을 가진 Pod가 없기 때문에 server2 파드는 pending상태로 생성되지 않음 

      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: server2
      spec:
        affinity:
          podAffinity:
            requiredDuringSchedulingIgnoredDuringExecution:   
            - topologyKey: a-team
              labelSelector:
                matchExpressions:
                -  {key: type, operator: In, values: [web2]}
        containers:
        - name: container
          image: kubetm/app
        terminationGracePeriodSeconds: 0
      ```

      위의 server2파드가 생성된 상황에서, <br>
      아래의 설정 정보를 가진 web2 파드를 생성하면, <br>
      `a-team=2`라벨을 가진 k8s-worker-02 노드에 web2 파드가 스케줄링되고 <br>
      server2 파드가 `type: web2`라벨을 podAffinity matchExpressions로 설정했기 때문에 <br>
      pending 상태에 있던 server2파드 또한 k8s-worker-02 노드에 스케줄링된다.

      ```yaml
      apiVersion: v1
      kind: Pod
      metadata:
        name: web2
        labels:
          type: web2
      spec:
        nodeSelector:
          a-team: '2'
        containers:
        - name: container
          image: kubetm/app
        terminationGracePeriodSeconds: 0
      ```

      </div>
      </details>

20. 노드 Taint 설정시 `NoSchedule`과 `NoExecute` effect의 차이점
    * 이미 스케줄링 되어 있는 파드들의 동작
      * NoSchedule - 해당 노드에 생성되어 있는 Pod들은 삭제되지 않음
      * NoExecute - 해당 노드에 생성되어 있는 Pod들은 모두 삭제됨
21. master노드에서는 CoreDNS에 접근이 불가능하다.
    * O: CoreDNS는 Pod로 실행되어 클러스터내에 위치해 있기 때문에, 클러스터 네트워크 대역(Pod 네트워크 CIDR와 서비스 네트워크 CIDR)에서만 접근이 가능하다.
22. 