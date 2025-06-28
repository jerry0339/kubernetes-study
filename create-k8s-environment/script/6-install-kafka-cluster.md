## 사전 작업
* storageClass 정의 필요 (retain 설정 필수)

<br>

## 스크립트
```sh
#!/bin/bash
set -e

echo "🔧 Kafka Cluster 설치 중..."
helm repo add bitnami https://charts.bitnami.com/bitnami
helm repo update

cat > kafka-values.yaml <<EOF
global:
  storageClass: "stateful-rook-ceph-block"

kafka:
  replicaCount: 3  # Worker Node 3개에 대응
  kraft:
    enabled: true  # KRaft 모드 활성화
  listeners:
    client:
      protocol: PLAINTEXT  # SASL_PLAINTEXT → PLAINTEXT로 변경, 설치할때 --set옵션으로 주긴 하지만 그래도 추가하는 것이 좋음
  persistence:
    size: 5Gi     # PVC 크기 지정 - 20G 권장, allowVolumeExpansion: true설정이 storageclass에 있으므로 확장도 가능
    # existingClaim: ""  # 기존 PVC 사용할 경우, 이름 지정
  # KRaft모드에서는 pod가 controller와 broker 역할을 모두 수행하기 때문에 KAFKA_HEAP_OPTS을 아래와 같이 controller,broker 모두 설정해 주어야 함
  controller:
    extraEnvVars:
      - name: KAFKA_HEAP_OPTS
        value: "-Xmx768m -Xms768m" # default 설정이 1Gi임 -> memory resource 설정을 낮게 하면 1Gi를 요청하여 OOM이 발생하므로 Memory Limits 설정에 맞추어 768로 낮춘것임
  broker:
    extraEnvVars:
      - name: KAFKA_HEAP_OPTS
        value: "-Xmx768m -Xms768m"
  configurationOverrides:
    - "process.roles=broker,controller" # 단일 노드가 브로커(데이터 처리)와 컨트롤러(메타데이터 관리) 역할을 동시에 수행
    - "node.id=${HOSTNAME##*-}" # 각 노드별 고유 ID 설정 - 노드 이름이 k8s-worker-01~03일 경우 StatefulSet 인덱스(0,1,2) 자동 할당
    - "controller.quorum.voters=0@kafka-controller-0.kafka-controller-headless.kafka.svc.cluster.local:9093,1@kafka-controller-1.kafka-controller-headless.kafka.svc.cluster.local:9093,2@kafka-controller-2.kafka-controller-headless.kafka.svc.cluster.local:9093" # FQDN 직접 명시
    - "advertised.listeners=CLIENT://kafka-controller-${HOSTNAME##*-}.kafka-controller-headless.kafka.svc.cluster.local:9092"
    - "listeners=CLIENT://:9092,CONTROLLER://:9093"
    - "listener.security.protocol.map=CLIENT:PLAINTEXT"
    - "transaction.state.log.min.isr=2" # 기본값 1 -> Transaction 사용시 2이상 필요 (트랜잭션 로그 ISR)
    - "num.partitions=3"       # 자동 생성되는 토픽의 파티션 개수 설정
    - "num.network.threads=3"  # 기본값 5 -> 3으로 감소 설정 (test 환경 용도)
    - "num.io.threads=5"       # 기본값 8 -> 5로 감소 설정 (test 환경 용도)
    - "min.insync.replicas=2"  # 기본값 1 -> acks: "all" 사용시, ISR 파티션 개수 설정, 데이터 처리 속도에 큰 영향을 미치므로 신뢰성과의 trade-off 고려
  resources: # 아래 설정은 test환경 최소 사양 세팅
    requests:
      cpu: "300m"
      memory: "512Mi"
    limits:
      cpu: "500m"
      memory: "768Mi"
  monitoring:
    jmx:
      enabled: true  # 모니터링 활성화 - Prometheus와 연결 가능
EOF

helm install kafka bitnami/kafka \
  --namespace kafka \
  --create-namespace \
  -f kafka-values.yaml \
  --set listeners.client.protocol=PLAINTEXT \
  --set 'controller.extraEnvVars[0].name=KAFKA_HEAP_OPTS' \
  --set 'controller.extraEnvVars[0].value=-Xmx768m -Xms768m' \
  --set 'broker.extraEnvVars[0].name=KAFKA_HEAP_OPTS' \
  --set 'broker.extraEnvVars[0].value=-Xmx768m -Xms768m' \
  --set 'controller.resources.requests.cpu=300m' \
  --set 'controller.resources.limits.cpu=500m' \
  --set 'broker.resources.requests.cpu=300m' \
  --set 'broker.resources.limits.cpu=500m' \
  --set 'controller.resources.requests.memory=512Mi' \
  --set 'controller.resources.limits.memory=768Mi' \
  --set 'broker.resources.requests.memory=512Mi' \
  --set 'broker.resources.limits.memory=768Mi'

sudo rm -f kafka-values.yaml

echo "✅ Kafka Cluster 설치 완료"

```