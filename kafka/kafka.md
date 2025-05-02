## Kafka local 환경(window)에서 실행하기
* [참고링크](https://soonmin.tistory.com/22)
* 실행 스크립트 참고
  ```sh
  cd C:\kafka_2.13-3.8.1
  .\bin\windows\zookeeper-server-start.bat config\zookeeper.properties
  .\bin\windows\kafka-server-start.bat config\server.properties

  # c.f. port
  netstat -ano | find "LISTEN"
  netstat -ano | find "2181" # zookeeper 포트
  netstat -ano | find "9092" # 브로커 포트
  ```

<br><br>

## 1. 카프카 구조
* 프로듀서
  * 프로듀서는 메시지를 생성하여 메시지 큐에 전달하는 역할
  * 프로듀서는 동기식 또는 비동기식으로 메시지를 전송할 수 있음
* 컨슈머
  * 컨슈머는 메시지 큐에서 메시지를 가져와 처리하는 역할
  * 메시지 큐에서 메시지를 순차적으로 가져와 처리
    * 파티션에서 커밋된(컨슈머로 처리된) 메시지 이후의 메시지부터 처리
  * 컨슈머 그룹?
    * 컨슈머 그룹은 동일한 그룹 ID를 가진 하나 이상의 Kafka 컨슈머 인스턴스 집합
    * 이 그룹은 하나의 토픽을 여러 컨슈머가 나눠서 병렬로 메시지를 처리할 수 있게 해줌
    * 즉, 컨슈머 그룹 내의 각 컨슈머는 토픽의 파티션을 분배받아 독립적으로 메시지를 읽고 처리
    * 부하 분산, 고가용성을 위함
    * 일반적으로 MSA 환경에서는 컨슈머 그룹을 마이크로서비스 단위로 설정
* 토픽
  * 토픽은 메시지를 특정 주제나 테마에 따라 분류하는 메커니즘
  * 프로듀서가 메시지를 특정 주제로 발행하고, 컨슈머가 해당 주제에 관심이 있는 메시지만 수신할 수 있도록 함
  * 즉, 프로듀서는 메시지를 특정 토픽에 발행하고, 컨슈머는 해당 토픽을 구독하여 메시지를 받음 (pub-sub 패턴)
  * cf. Lag 이란?
    * lag = (토픽의 최신 오프셋) - (컨슈머 그룹이 읽은 오프셋)
    * lag가 크면 컨슈머가 메시지를 제때 처리하지 못하고 있다는 뜻
    * 시스템 병목, 장애, 컨슈머 다운, 처리 성능 저하, 파티션 불균형 등의 문제가 원인임
* 파티션
  * 파티션은 큐로 이루어져 있어 메시지의 순서를 유지
  * Kafka의 파티션은 데이터를 분산 저장하는 단위 (데이터의 복제가 아님)
  * 메시지는 특정 키에 따라 파티션으로 분배되며, 각 파티션의 메시지는 독립적으로 처리됨
  * 파티션은 메시지를 여러 개의 하위 그룹으로 나누어 저장하고 처리하는 방식
  * 예를들어, 토픽의 파티션 설정 개수가 10개라고 가정할때,
    * 주문 이벤트를 저장하는 "orders"라는 토픽이 있다고 가정하면, 이 토픽에 대한 10개의 파티션이 생성됨
    * 주문 ID같은 특정 키를 파티션 키로 사용할 수 있고, 같은 주문에 대한 이벤트(생성, 결제, 배송 등)는 항상 같은 파티션에 저장되어 순서가 보장됨
    * 주문 이벤트가 발생할때 Kafka 프로듀서는 주문 ID를 키로, 해시를 계산하거나 라운드로빈 방식 등으로 파티션을 선택함
    * 즉, 주문 A는 파티션 1, 주문 B는 파티션 2, 주문 C는 파티션 3에 저장될 수 있어 같은 키를 가진 메시지는 항상 같은 파티션에 저장되어 순서가 보장됨
    * 동시에 여러 Consumer가 각 파티션을 병렬로 읽어 주문 처리 속도가 크게 향상됨
  * 동일한 파티션의 복제본은 여러 브로커에 분산하여 저장됨 - 고가용성
    * 각 파티션은 하나의 리더(leader)와 여러 팔로워(follower) 복제본을 가짐 (복제 수는 브로커의 수를 초과할 수 없음)
    * 리더 파티션은 한 브로커에 있고, 팔로워 파티션들은 다른 브로커에 위치
    * 프로듀서와 컨슈머는 항상 리더 파티션을 통해 데이터를 주고받으며, 팔로워 파티션들은 리더의 데이터를 실시간으로 복제하는 방식
* 브로커
  * kafka 클러스터내에 존재하는 작업 단위
  * 각 브로커는 하나 이상의 토픽 파티션을 관리하고, 데이터의 복제본을 유지
  * 데이터는 모든 브로커에 복제되어 offset이 저장된 세그먼트 파일 시스템 형태로 저장된다.
    * offset: 어디까지 데이터를 읽었는지에 대한 위치 정보
  * 브로커의 그룹 코디네이터 역할
    * 토픽내의 파티션 하나에 컨슈머 하나가 1:1로 대응되지만, 특정 컨슈머에 문제가 생길 경우 해당하는 파티션을 다른 컨슈머에 재할당(리밸런스) 시켜줌
    * 컨슈머 그룹의 상태를 체크하고 토픽의 파티션을 컨슈머와 매칭되도록 분배하는 역할을 함
  * 컨슈머 오프셋 저장
    * 컨슈머 그룹이 파티션의 어느 레코드까지 가져갔는지 확인하기 위해 오프셋을 커밋함
  * 데이터를 삭제 할 수 있음. (컨슈머나 프로듀서를 통해서 데이터 삭제는 불가능)
  * 다수의 브로커중 하나는 컨트롤러 역할을 함.
    * 컨트롤러는 다른 브로커들의 상태를 체크한다.
    * 어떤 브로커의 상태가 비정상이라면 클러스터에서 해당 브로커를 제외시키고 해당 브로커의 파티션을 재분배 하는 역할을 한다.
* ![](2025-03-10-01-22-36.png)

<br><br>

## 2. Kafka 특징
* **높은 처리량**
  * 카프카는 프로듀서가 브로커로 데이터를 보낼 때와 컨슈머가 브로커로부터 데이터를 받을 때 모두 묶어서 전송
    * 네트워크 통신 횟수 줄어듦
  * 많은 양의 데이터를 묶음 단위로 처리하는 배치로 빠르게 처리할 수도 있음
  * 파티션 단위를 통해 동일 목적의 데이터를 여러 파티션에 분배하고 데이터를 병렬 처리할 수 있음
    * 파티션 개수만큼 컨슈머 개수를 늘려서 동일 시간당 데이터 처리량을 늘리는 것
* **확장성**
  * 메시지 양에 따라서 클러스터의 브로커 개수(파티션에 1:1대응)를 scale-out / scale-in 할 수 있음
    * scale-out: 브로커 늘림, in은 반대
  * 365일 24시간 데이터를 처리해야 하는 서비스에서 안정적으로 운영이 가능함
* **영속성**
  * 카프카에서 프로듀서로부터 전송받은 메시지 데이터들은 메모리에 저장하지 않고 파일 시스템에 저장됨
  * 따라서 브로커에 급작스럽게 장애가 발생하더라도, 브로커를 재시작하고 파일시스템에 저장된 데이터를 불러와 메시지를 다시 처리할 수 있음
* **고가용성**
  * 프로듀서로부터 전송받은 데이터의 복제를 통해 고가용성의 특징을 가짐
  * 즉, 프로듀서로부터 전송받은 데이터는 1대의 브로커에만 저장되는 것이 아니라 다른 브로커들 모두에도 저장되어 있으므로,
  * 저장된 데이터를 기준으로 지속적으로 데이터 처리가 가능한 것

<br><br>

## 3. Batch 데이터와 Stream 데이터 ?
* Batch 데이터
  * 한정된 데이터 세트를 일정 기간 동안 수집한 후, 이를 한꺼번에 처리
  * 대규모 데이터 처리에 적합
  * 예시
    * 급여 처리: 회사에서 직원의 근무 기록을 한 달 또는 2주 단위로 모아서 한 번에 급여를 계산하고 지급하는 작업
    * 카드 거래 정산: 하루 동안 발생한 신용카드 거래를 모아서 일정 시점에 한 번에 정산하는 작업
* Stream 데이터
  * 지속적으로 들어오는 데이터를 실시간으로 처리
  * 실시간 분석 및 빠른 응답이 필요할 때 적합
  * Batch 프로세싱과 비교하여 낮은 지연시간으로 생성되는 데이터
  * 예시
    * 실시간 위치 추적: 차량, 배달, 택시 서비스 등에서 GPS 센서가 지속적으로 위치 정보를 전송하여 실시간으로 위치를 추적하는 경우
    * 실시간 로그 분석: 서버나 애플리케이션에서 발생하는 로그 데이터를 실시간으로 수집하고 분석하여 장애를 즉시 탐지하는 경우

### 3.1. Kafka에서는 Batch데이터와 Stream데이터를 모두 처리할 수 있음
* 어떻게?
  * Kafka에서는 메시지 로그에 시간을 남기기 때문에, Stream으로 적재된 데이터를 Batch데이터로 처리할 수 있음
* ![](2025-03-10-03-34-02.png)

<br><br>

## 4. Kafka Producer
* kafka의 Producer는 데이터를 Kafka 클러스터로 전송하는 역할을 하는 클라이언트 애플리케이션
* 프로듀서는 데이터를 전송할 때, **리더 파티션**을 가지고 있는 카프카 브로커와 직접 통신한다.
* 프로듀서는 브로커로 데이터를 전송할 때 내부적으로 파티셔너, 배치 생성 단계(Accumulator)를 거침

<br>

### 4.1. Producer 내부 구조
* `ProducerRecord`: 프로듀서에서 생성하는 레코드. 오프셋은 포함되지 않음(클러스터에 저장될때 생성됨)
* `send()`: 레코드 전송 요청 메서드
* `Partitioner`: 어느 파티션으로 전송할지 지정하는 파티셔너. DefaultPartitioner가 기본 클래스
  * 파티셔닝 전략(파티셔너 클래스)에 따라 다르지만, 레코드는 기본적으로 키값에 의해 특정 파티션에 고정적으로 할당됨.
  * cf. 레코드의 복제는 다른 브로커에 이루어 짐. 즉, 해당 파티션의 레코드는 그 브로커 내에 하나만 존재하게 됨
* `Accumulator`: 배치로 묶어 전송할 데이터를 모으는 **버퍼**
  * 즉, send()할때마다 데이터를 보내는 것이 아니라 데이터를 묶어서 브로커로 전송하여 데이터 처리량을 높임
  * cf. producer.flush() 코드로 버퍼의 데이터를 바로 보내도록 할 수도 있음
* ![](2025-05-01-14-31-17.png)

<br>

### 4.2. ISR과 acks 옵션 (Producer 옵션)
* [참고한 링크](https://techietalks.tistory.com/entry/Apache-Kafka-%EB%B8%8C%EB%A1%9C%EC%BB%A4-%EB%B3%B5%EC%A0%9C-ISR-%EC%9D%B4%EB%9E%80)
* 카프카에서는 하나의 토픽 파티션은 여러 브로커에 걸쳐 복제된다.
* 이는 데이터의 안정성을 보장하며, 특정 브로커 장애 발생 시 빠른 복구를 가능하게 한다. (고가용성)
* 리더 파티션과 팔로워 파티션?
  * 여러 브로커에 복제되어 저장되는 파티션은 각각 리더 또는 팔로워의 역할을 가짐
  * 프로듀서와 컨슈머는 오직 **리더 파티션**과만 직접 통신한다.
  * 팔로워 파티션은 리더 파티션의 데이터를 복제하여 저장한다. 
  * 팔로워 파티션에서는, 새로운 메시지가 있으면 리더로부터 데이터를 받아 동기화하는 작업이 필요함
* `ISR(In-Sync-Replicas)`
  * 팔로워 파티션은 리더 파티션과의 동기화 작업이 필요하기 때문에, 리더 파티션과 offset차이가 발생할 수 있다.
  * 이때, 리더 파티션과 팔로워 파티션이 모두 싱크가 된 상태를 ISR이라고 함
  * 아래 그림의 파티션을 보면 리더와 팔로워 파티션 모두 offset이 3으로 동일하므로 ISR 형태인 것임
  * ![](2025-05-01-16-24-08.png)
* `asks`
  * 신뢰성과 처리 속도 사이의 trade off를 결정하는 설정
  * acks(acknowledgments)는 프로듀서가 메시지를 브로커에 전송한 뒤,
    * 브로커로부터 "정상적으로 메시지가 저장되었다"는 확인(응답)을 어떤 수준까지 받을지 결정하는 옵션
  * 0, 1, all(-1) 값을 가질 수 있다.
    * `0`: 확인하지 않음 - 최고 속도, 데이터 유실 위험 큼, 데이터 유실 발생하더라도 전송 속도가 중요한 경우 사용
    * `1`: 리더 파티션 저장만 확인 - 리더 파티션에 적재되지 않았다면, 적재될 때까지 재시도 할 수 있다. 하지만 데이터 유실 가능
      * 일반적인 경우, 해당(1) 옵션을 사용함
    * `all(-1)`: ISR 내 모든 복제본 저장 확인 - 최고 신뢰성, 상대적으로 느림
      * `min.insync.replicas`: ISR 파티션 개수, 1로 설정시 acks = 1 설정과 동일하다고 볼 수 있음. 즉, 1로 설정시 리더 파티션만 적재 확인함
    * kafka 3.0 이상부터 **all**이 default 설정이지만, **min.insync.replicas = 1**이 default 설정이므로 acks=1과 같은 설정임

<br>

### 4.3. 멱등성 프로듀서 (enable.idempotence 옵션)
* cf. 멱등성: 여러번 연산을 수행하더라도 동일한 결과를 나타내는 것
* 프로듀서가 보내는 데이터의 중복 적재를 막기 위해 프로듀서의 enable.idempotence 옵션을 true로 설정하여 멱등성 프로듀서로써 사용할 수 있다.
  * kafka 3.0 부터 `enable.idempotence = true`가 default
* 멱등성 프로듀서는 동일한 데이터를 여러번 전송하더라도 카프카 클러스터에 단 한번만 저장되도록 할 수 있다. (Exactly once delivery)

<br>

### 4.4 producer 코드 예시

#### 4.4.1. Producer 기본 사용법 예시 코드
  ```java
  KafkaProducer<String, String> producer = new KafkaProducer<>(configs);

  // Value만 전송하는 경우
  ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, "testMessage");
  producer.send(record);

  // (Key , Value) 전송하는 경우
  ProducerRecord<String, String> record2 = new ProducerRecord<>(TOPIC_NAME, "6", "Busan");
  producer.send(record2);

  // (partition번호 지정, key, Value) 전송하는 경우
  int partitionNo = 2;
  ProducerRecord<String, String> record3 = new ProducerRecord<>(TOPIC_NAME, partitionNo, "1", "Seoul");
  producer.send(record3);

  producer.flush(); // Accumulator(버퍼)의 데이터를 클러스터(토픽의 파티션)으로 전송하는 코드
  producer.close(); // Producer의 안전한 종료
  ```
#### 4.4.2. 커스텀 파티셔너를 가지는 Producer 예시 코드
* ex. "Pangyo"라는 value를 가진 메시지를 항상 0번 파티션으로 할당하는 CustomPartitioner 코드
  ```java
  public class CustomPartitioner  implements Partitioner {

      @Override
      public int partition(String topic, Object key, byte[] keyBytes, Object value, byte[] valueBytes, Cluster cluster) {
          if (keyBytes == null) {
              throw new InvalidRecordException("Need message key");
          }
          if (((String)value).equals("Pangyo")) // 값이 Pangyo일 경우
              return 0;

          // Pangyo아닌 경우 정상 처리
          List<PartitionInfo> partitions = cluster.partitionsForTopic(topic);
          int numPartitions = partitions.size();
          return Utils.toPositive(Utils.murmur2(keyBytes)) % numPartitions;
      }

      @Override
      public void configure(Map<String, ?> configs) {}

      @Override
      public void close() {}
  }
  ```
* CustomPartitioner를 Partitioner로 변경하는 코드
  ```java
  Properties configs = new Properties();
  // 위에서 정의한 CustomPartitioner를 파티셔너로 할당
  configs.put(ProducerConfig.PARTITIONER_CLASS_CONFIG, CustomPartitioner.class);

  KafkaProducer<String, String> producer = new KafkaProducer<>(configs);
  ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, "2", "Pangyo"); // 해당 레코드는 0번 파티션으로...

  producer.send(record);
  producer.flush();
  producer.close(); // 프로듀서의 안전한 종료
  ```

#### 4.4.3. Producer의 Metadata 출력하기
* producer.send(record).get() 으로 producer의 config 정보 받을 수 있음
  ```java
  // ...
  KafkaProducer<String, String> producer = new KafkaProducer<>(configs);
  ProducerRecord<String, String> record = new ProducerRecord<>(TOPIC_NAME, "2", "Pangyo");

  // ...
  try {
      RecordMetadata metadata = producer.send(record).get(); // producer의 config 정보 받기
      // logger.info(metadata.toString()); // 확인용 코드
  } catch (Exception e) {
      logger.error(e.getMessage(),e); // 문제가 생겼을때만 로그 남기는게 일반적임
  } finally {
      producer.flush();
      producer.close();
  }
  ```

<br><br>

## 5. Kafka Consumer
* Producer가 전송한 데이터는 Kafka 브로커에, 정확히는 토픽 내의 특정 파티션에 적재되는데,
* 컨슈머는 적재된 데이터를 사용하기 위해 브로커로부터 데이터를 가져와 처리한다.
* 컨슈머는 데이터를 가져올 때, **리더 파티션**을 가지고 있는 카프카 브로커와 직접 통신한다.

<br>

### 5.1. Consumer Group
* 컨슈머 그룹은 카프카의 컨슈머 인스턴스들을 하나로 묶는 논리적 그룹 단위 이다.
  * 컨슈머들은 토픽을 구독하여, 토픽의 1개 이상의 파티션들에 할당되어 데이터를 가져갈 수 있다.
  * 파티션과 컨슈머는 N:1 로 할당 가능하다. 따라서 컨슈머의 개수는 파티션의 개수와 같거나 적어야 한다.
    * 컨슈머 개수가 더 많아지는 경우, Idle 상태로 분류되고 스레드를 차지하므로 불필요한 리소스에 해당
    * 따라서 아래 그림과 같이 컨슈머를 할당하며, 파티션과 컨슈머가 1:1로 대응되는 경우 리소스를 더 많이 차지하지만 처리속도가 가장 빠른 형태에 해당
    * cf. **컨슈머와 파티션의 개수를 같게 하여 컨슈머와 파티션을 1:1로 할당하여 사용하는 것이 일반적인 방법**
  * ![](2025-04-29-19-18-00.png)

#### 5.1.1. Consumer Group 왜 필요할까?
* `fail over - Rebalancing`: 특정 Consumer에 장애가 발생해도 동일 그룹 내의 다른 Consumer가 남은 파티션을 자동으로 할당받아 데이터를 계속 소비할 수 있다.
  * 리밸런싱(rebalancing): 컨슈머가 추가되거나 장애 상황으로 컨슈머가 제외되는 상황에서, 파티션의 소유권을 새로운, 또는 다른 컨슈머에게 이관하는 조정 작업
* `병렬 처리`: 하나의 Consumer Group 내에 여러 Consumer Instance를 두면, 파티션을 여러 Consumer가 나눠서 읽을 수 있어서 데이터 처리 속도가 증가
  * ex. 아래 그림에서, 하둡 적재 컨슈머 그룹
* `Offset 관리 단일화`: Consumer Group마다 토픽에 대한 Offset을 별도로 저장하므로, 여러 Group이 동일한 Topic을 읽더라도 서로 영향을 주지 않고 독립적으로 데이터를 소비
* `다양한 용도별 데이터 소비`: 동일한 Topic에 여러 Consumer Group이 붙을 수 있고 필요에 따라 컨슈머의 개수를 다르게 적용 가능함
  * ex. 아래 그림에서와 같이, 그룹마다 다른 형태로 데이터가 적재되도록 소비할 수 있다.
  * 또한, 즉각적인 데이터 처리가 필요하다면 컨슈머 개수를 최대한으로 늘려 처리 속도를 올리거나 그렇지 않은 경우라면 컨슈머를 줄여 리소스를 절약할 수 있다.
  * **컨슈머와 파티션의 개수를 같게 하여 컨슈머와 파티션을 1:1로 할당하여 사용하는 것이 일반적인 방법**
* ![](2025-04-29-19-18-59.png)

<br>

### 5.2. 커밋(commit)과 오프셋(offset)
* 컨슈머가 poll()을 호출할 때마다 Consumer Group은 카프카에 저장되어 있는 아직 읽지 않은 메시지를 가져온다.
* 이렇게 동작할 수 있는 이유는, 컨슈머 그룹이 메시지를 어디까지 가져갔는지 알 수 있도록 offset을 기록하기 때문
* `offset`이란 어디까지 데이터를 읽었는지에 대한 위치 정보이다.
* 토픽의 리더 Partition에 있는 n번째 레코드를 Consumer Group이 `poll()`하여 처리후 `commit()`하게 되면,
* 토픽 내부의 `offset`에 기록되어 어디까지가 처리된 데이터인지 확인할 수 있다.
* ![](2025-04-30-01-34-41.png)

<br>

### 5.3. auto-offset-reset
* 컨슈머 그룹이 특정 파티션을 읽을 때, 커밋된 컨슈머 오프셋이 없는 경우 어디부터 데이터를 읽을지 선택하는 옵션
* `latest`: 가장 최근에 적재된 데이터부터 읽기 시작함
* `earliest`: 가장 오래된 오프셋부터 읽기 시작함
* 커밋 기록이 있는 경우 `auto-offset-reset`옵션은 무시된다.
* 따라서 특별한 경우가 아니라면 earliest로 설정하는 것이 좋다.
  * 예를들어 파티션을 늘리는 경우, 처음 데이터부터 처리되기를 기대하는 것이 일반적이다.
  * **기본 값이 latest이므로 earliest로 꼭 명시해 주는 것이 좋다.**

<br>

### 5.4. 컨슈머의 세션 타임아웃과 안전한 종료
* 컨슈머가 코드 상에서 명시적으로 종료되지 않거나, 장애가 생겨 **하트비트(heartbeat)**를 전송하지 못한다면
  * 세션 타임아웃(`session.timeout.ms`)이 발생할 때까지 컨슈머 그룹에 남아있게 된다.
* 컨슈머는 주기적으로 그룹 코디네이터(브로커)에게 하트비트를 전송하여 활성 상태를 유지
* 하트비트 주기(`heartbeat.interval.ms`)는 기본값 3초로 제어되며, 아래의 조건이 충족되지 않으면 컨슈머는 그룹에서 제거됨
  * session.timeout.ms(기본값 45초) 내에 최소 한 번의 하트비트 전송 - kafka 4.0.0 기준
  * 컨슈머 장애 시 최대 45초간 데이터 처리 중단 발생 가능하다는 것을 의미함
  * `session.timeout.ms`과 `heartbeat.interval.ms`의 시간은 설정 가능함 (ms기준 주의)
* 컨슈머를 코드 상에서 명시적으로 종료하려면, `KafkaConsumer.close()` 메서드를 호출해야 한다.
  * Shutdown Hook & Wakeup 이용하여 컨슈머 종료 시키기
  * `todo` - 실습 코드

<br>

### 5.5. 멀티 프로세스, 멀티 스레드 컨슈머
* 기본적으로 하나의 컨슈머는 하나의 스레드로 동작한다.
* k8s 환경에서는 아래 그림의 그룹 A와 같이 Pod를 여러개 띄워서 멀티 프로세서 환경으로 운영이 가능
  * 고가용성이지만 멀티 스레드(B그룹) 방식과 비교하여 자원 소모가 크다
* `kafka.listener.concurrency` 옵션을 이용하여 오른쪽 그림과 같이 멀티 스레드 환경으로도 운영이 가능하다.
  * 자원 효율성이 높지만 특정 컨슈머에 장애 발생시 다른 컨슈머에도 영향이 감
* ![](2025-05-01-22-32-55.png)

<br>

### 5.6. Consumer Lag
* 컨슈머 랙(Lag)은 파티션의 최신 오프셋(Log-end-offset)과 컨슈머 오프셋(current-offset)간의 차이이다.
* 컨슈머 랙의 개수?
  * 컨슈머 랙은 그룹과 토픽, 파티션별로 생성된다. 
  * 2개의 토픽에 각각 3개의 파티션이 있고 컨슈머 그룹이 2개씩 있다면?
  * 컨슈머 랙의 총 개수 = 2(토픽) x 2(그룹) × 3(파티션) = 12개
* ![](2025-04-30-02-06-09.png)
* 컨슈머 랙의 최솟값(개수 말고..)은 0으로 지연이 없음을 뜻한다.
* 프로듀서의 데이터양이 늘어날 경우, 컨슈머 랙이 늘어날 수 있다.
* 파티션 개수와 컨슈머 개수를 늘려 병렬 처리량을 높여 컨슈머 랙을 줄일 수 있다.
  * cf. 컨슈머 최대 개수는 파티션의 개수이므로, 컨슈머와 파티션 개수가 같다면 파티션을 늘려야 처리량을 높일 수 있다.
* 컨슈머의 장애로 컨슈머 랙이 증가할 수도 있다.
  * ex. 프로듀서가 보내는 데이터양은 일정한데, 파티션 2번의 컨슈머 랙이 늘어나는 상황이 발생한다면 2번 파티션에 할당된 컨슈머의 장애를 의심

#### 5.6.1. Consumer Lag Monitoring
* 명령어로 확인하기
  ```sh
  kafka-consumer-groups.sh --bootstrap-server <브로커주소>:9092 --group <그룹명> --describe
  ```
* Kafka Consumer Lag 모니터링 도구 사용하기
  * Burrow, Kafka Exporter, Kafka Lag Exporter와 같은 도구를 사용하여 모니터링 할 수 있음
    * `Burrow`: Consumer Lag에 특화된 모니터링
    * `Kafka Exporter`: 간단한 설정으로 Prometheus & Grafana와 통합, Kafka의 전반적인 상태와 함께 Consumer Lag도 모니터링
    * `Kafka Lag Exporter`: Lag의 시간(초단위) 측정과 Kubernetes 환경 통합, 단 다소 높은 리소스 사용량
  * Prometheus 및 Grafana와 통합하여 데이터 시각화도 가능함

<br><br>

## 6. Transaction Producer와 Consumer
* 다수의 파티션에 데이터를 저장해야 하는 경우, `Transaction Producer`를 사용하여 하나의 Transaction으로 묶어서 처리할 수 있다.
  * ![](2025-05-01-19-30-38.png)
* 이때, `Transaction Consumer`는 트랜잭션이 완료되어 Commit된 데이터를 확인하고 데이터를 가져간다.
  * 아래의 그림을 보면, 트랜잭션 컨슈머가 파티션의 트랜잭션 레코드 Commit 기록을 확인하고 데이터를 가져가는 모습
  * ![](2025-05-01-19-34-33.png)
* Transaction Producer 생성 코드 예시
  ```java
  // Transaction id 설정 -> UUID와 같이 고유한 값 사용
  configs.put(ProducerConfig.TRANSACTIONAL_ID_CONFIG, UUID.randomUUID());

  Producer<String, String> producer = new KafkaProducer<>(configs); 
  producer.initTransactions(); // 트랜잭션 초기화

  producer.beginTransaction(); // 트랜잭션 시작
  // 하나의 트랜잭션내에 여러 토픽의 데이터 전송 가능
  producer.send(new ProducerRecord<>(TOPIC_1, "전달하는 메시지 값 1"));
  producer.send(new ProducerRecord<>(TOPIC_2, "전달하는 메시지 값 2")); 
  producer.commitTransaction(); // 트랜잭션 완료(커밋)

  producer.close(); // 안전한 프로듀서 종료
  ```
* Transaction Consumer 생성 코드 예시
  ```java
  // Transaction Consumer 설정 - read_committed로 변경
  configs.put(ConsumerConfig.ISOLATION_LEVEL_CONFIG, "read_committed"); // 기본값 - read_uncommitted
  KafkaConsumer<String, String> consumer = new KafkaConsumer<>(configs);
  ```


<br><br>

## 7. Kafka Streams
* Kafka Streams는 kafka의 **topic의 데이터**를 읽고 **변환 처리**한 후 그 결과를 kafka의 **topic에 저장**하는 작업을 해줌
* 출처 - [confluent 공식 문서](https://docs.confluent.io/platform/current/streams/architecture.html)
* Kafka Streams API를 사용하는 애플리케이션의 구조
  * ![](2025-04-27-17-20-16.png)

<br>

### 7.1. Kafka Streams란 ?
  * Apache Kafka 기반의 실시간 데이터 처리를 위한 자바 라이브러리
  * Kafka 토픽에서 데이터를 읽고(Source Processor), 변환 or 가공 처리한 후(Stream Processor), 다시 Kafka 토픽으로 결과를 저장(Sink Processor)하는 작업을 수행함
  * Kafka Streams는 별도의 클러스터를 필요로 하지 않고, 라이브러리 형태로 일반 Java 애플리케이션을 통해 구현 가능

<br>

### 7.2. Kafka Streams 구조 ?
* Kafka Streams 는 Tree형 Topology 구조를 가짐, DAG임
* 노드를 `Processor`, 간선을 `Stream` 이라고 부름
* ![](2025-04-27-17-20-54.png)
* `Processor` ?
  * `소스 프로세서(Source Processor)`
    * 소스 프로세서는 Kafka 토픽에서 데이터를 읽어오는 시작점이다.
    * 하나 이상의 Kafka 토픽을 구독하고, 레코드를 스트림 처리 파이프라인으로 전달
  * `스트림 프로세서(Stream Processor)`
    * 스트림 프로세서는 데이터 변환, 필터링, 집계 등의 비즈니스 로직을 수행하는 핵심 컴포넌트
    * 강력한 Stateful, Stateless 프로세싱 기능이 들어 있어 카프카의 토픽 데이터 처리시 선택하여 처리 가능
      * stateless ex. map(레코드 변환), filter(레코드 조건 필터링) 등
      * stateful ex. windowed(시간 기반 윈도우 집계), join(스트림/테이블 조인) 등
  * `싱크 프로세서(Sink Processor)`
    * 싱크 프로세서는 처리된 데이터를 Kafka 토픽 또는 외부 시스템에 쓰는 종착점
    * 다운스트림 프로세서가 없으며, 최종 결과를 출력함
    * 즉, 최종 처리된 데이터를 어떤 토픽에 저장하는 역할을 함

<br>

### 7.3. kafka Producer와 Consumer를 이용하면 되는 내용인데 Kafka Streams를 쓰는 이유?
* `실시간 데이터 처리` - Kafka 공식 Java 라이브러리로, 실시간 데이터 처리(필터, 변환, 집계, 조인 등)를 손쉽게 구현 가능
  * 여러 Producer/Consumer 조합으로 처리하던 복잡한 구조를 Kafka Streams를 이용하여 단순화
* `부하 분산` - 내부적으로 파티션 단위로 작업을 분산 처리하고, 여러 인스턴스를 실행하면 자동으로 파티션을 나눠 병렬로 처리
* `failover 및 확장성` - 장애 발생 시 상태 저장소를 기반으로 자동 복구 및 재처리가 가능, 인스턴스를 추가하면 자동으로 파티션이 재분배되어 수평 확장이 용이
* `시간 기반 윈도우 연산` - 이벤트 타임, 처리 타임 등 다양한 시간 기반 윈도우 연산(예: 5분 집계, 슬라이딩 윈도우 등)을 지원
  * 단순 Consumer로는 구현이 어려움
* `상태 저장` - 로컬 RocksDB 등으로 상태를 직접 저장하며, Kafka 내부 토픽을 통해 장애 복구 및 정확성을 보장
  * 별도의 cluster 필요 없이 애플리케이션 단위로 내장 실행
* `Exactly-once 처리 보장` - 메시지의 중복 처리나 누락 없이 정확히 한 번만 처리하는 것을 보장

<br>

### 7.4. Streams DSL(Domain Specific Language)
* [참고 링크](https://goslim56.tistory.com/29)
* 미리 제공되는 함수들을 이용하여 토폴로지를 정의하는 방식
* 이벤트 기반 데이터 처리에 필요한 기능들을 제공하기 때문에 스트림즈를 구현하기 편함
* 대부분의 변환 로직을 어렵지 않게 개발할 수 있도록 스트림 프로세싱에 쓰일만한 다양한 기능들을 자체 API로 제공
  * 이벤트 기반 데이터 처리를 할 때 필요한 다양한 기능들(map, join, window 등)을 대부분 제공
* application id로 Streams App을 구분하여 데이터 작업을 처리함
  * `application.id`: consumer의 Group Id와 동일한 역할로 Streams App을 구분하기 위한 고유 Id에 해당

#### 7.4.1. Streams DSL의 stateful 프로세싱, stateless 프로세싱
* Streams DSL는 실시간 데이터 처리를 위해 Stateful과 Stateless 연산(API)을 제공
* cf. Streams DSL에서 제공하는 연산 이외의 커스텀 연산이 필요하다면 Processor API를 사용해야 함
* Stateful 데이터 처리에 강함 (Streams는 상태를 유지해야 하는 복잡한 연산에 적합)
  * count: 키별 레코드 수 집계 - `stream.groupByKey().count()`
  * reduce: 키별 값 누적 연산 - `stream.groupByKey().reduce((v1, v2) -> v1 + v2)`
  * aggregate: 사용자 정의 초기값 + 집계 - `stream.groupByKey().aggregate(() -> 0L, (k, v, agg) -> agg + v.length())`
  * windowed: 시간 기반 윈도우 집계 ex.5분 간격 매출 계산 - `stream.groupByKey().windowedBy(TimeWindows.of(Duration.ofMinutes(5))).count()`
  * join: 스트림/테이블 조인 - `stream.join(otherStream, (v1, v2) -> v1 + v2, JoinWindows.of(Duration.ofSeconds(10)))`
* stateless 데이터 처리도 가능 (단순 변환 작업도 지원)
  * filter:	조건에 맞는 레코드만 필터링	- `stream.filter((k, v) -> v > 100)`
  * map: 레코드 변환 - `stream.mapValues(v -> v.toUpperCase())`
  * flatMap: 레코드를 여러 개로 분할 - `stream.flatMapValues(v -> Arrays.asList(v.split(" ")))`
  * branch: 조건별로 스트림 분기 - `KStream<String, String>[] branches = stream.branch(pred1, pred2)`
  * merge: 여러 스트림을 하나로 병합 - `stream1.merge(stream2)`

#### 7.4.2. stateful processing으로 topic에 저장되는 데이터는 batch 데이터라고 볼 수 있을까?
* ex. 주문량에 대해 1시간 구간으로 windowed stateful 프로세싱을 사용한다면?
  * 해당 데이터는 1시간 단위의 실시간(Stream) 데이터에 해당
  * 1시간 사이에 증분 업데이트(새로 추가되거나 수정된 데이터만 처리)된 결과가, Kafka 토픽에 스트림 형태로 저장되는 것
  * [09:00 ~ 10:00 매출 100만원], [10:00 ~ 11:00 매출 150만원], [11:00 ~ 12:00 매출 80만원], ...
* 따라서 해당 데이터는 1시간 단위의 스트림 데이터로 볼 수 있음
* 해당 데이터를 집계하여 월별 매출 데이터를 뽑아낸다면? Batch데이터라고 할 수 있음
* 즉, 실시간으로 데이터 발생 즉시 처리되며 `지속적`으로 들어오는 데이터 형태가 Stream 데이터이며
* 월간 통계와 같이 일정 기간동안 수집된 데이터를 한번에 처리하여 `완결된` 데이터 형태가 Batch 데이터이다.

#### 7.4.3. KStream, KTable, GlobalKTable ?
* Streams DSL에서는 스트림 데이터 처리를 위해 KStream, KTable, GlobalKTable이라는 세 가지 핵심 추상화를 제공
* `KStream`
  * 이벤트 스트림을 연속적인 레코드 시퀀스로 모델링
  * 모든 레코드를 개별 처리하며 상태 비저장(stateless) 연산에 적합
* `KTable`
  * 키 기반의 최신 상태를 추적하는 변경 로그
  * 동일 키에 대한 최신 값만 유지(upsert 방식)
* `GlobalKTable`
  * 모든 인스턴스에 전체 데이터 복사본을 저장하는 전역 테이블
  * 조인 시 파티션 재배치 불필요
* 조인을 수행하려는 두 토픽은 코파티셔닝이 보장되어 있어야 한다. 그렇지 않은 경우에 Join을 수행했을시 TopologyException이 발생 가능하다.
* 코파티셔닝? 두 개 이상의 토픽이 아래의 조건을 만족할 때 코파티셔닝이 됨으로 간주
  * 동일 키 사용
  * 동일 파티션 수
  * 동일 파티셔닝 전략
    * ex. UniformStickyPartitioner또는 custom Partitioner 구현하여 사용
    * 파티셔너 - key에 따라서 어느 Partition으로 갈지 정해주는 역할

#### 7.4.4. Streams dsl 라이브러리 추가
* 버전 확인
  * 설치된 kafka 버전 확인
  * kafka 4.0.0 사용중이므로 streams dsl 4.0.0 이상 버전을 사용해야 함
  * streams dsl 4.0.0은 java 17 이상의 버전을 사용해야 함
* gradle에 추가시 아래와 같음
  ```
  dependencies {
    implementation 'org.apache.kafka:kafka-streams:4.0.0'
  }
  ```

<br><br>

## 8. Kafka Connect
* Kafka Connect는 데이터 파이프라인 생성시 반복 작업을 줄이고 효율적인 전송을 하기 위한 애플리케이션이다.
  * DB나 FS와 연결하는데 사용함
    * ex. DB의 데이터를 Topic에 저장하고자 할때, DB에서 select하고 Topic에 Produce하는 과정을 Template으로 만들어 반복적으로 사용할 수 있음
  * 특정 작업 형태를 템플릿으로 만들어 놓은 Connector를 실행함으로써 반복 작업을 줄일 수 있다.
* Kafka Connect 작동 방식
  * ![](2025-04-27-21-38-16.png)
  * Source Connector: Producer 역할
  * Sink Connector: Consumer 역할
  * Connector는 오픈 소스를 이용할수도 있고 직접 개발하여 사용할 수도 있음
    * 오픈 소스 커넥터 ex. HDFS 커넥터, AWS S3 커넥터, JDBC 커넥터, ElasticSearch 커넥터 등
* Kafka Connect 내부 구조
  * ![](2025-04-27-21-39-32.png)
  * Kafka Connect 생성시 Connector와 Task를 생성해야 함
    * Connector에서는 주로 설정 값에 대한 Validation 로직들이 들어가고 모니터링에도 활용됨
    * Task는 Thread하나가 할당되며, 실질적인 데이터 처리 로직(DB와 연결, 리소스와 커넥트 등)이 들어감

<br>

### 8.1. Kafka Connect를 실행하는 방법
* 단일 모드 커넥트 (Standalone Mode Kafka Connect)
  * 개발 환경 구성 용도
* 분산 모드 커넥트 (Distributed Mode Kafka Connect)
  * 고가용성 구성 용도, failover 가능
  * 분산모드에서는 데이터량이 많아져 Consumer Lag이 늘어나거나 지연이 발생하는 경우 커넥트를 scale-out할 수 있음, scale-in도 마찬가지.

<br><br>

## 9. Kafka Streams vs Kafka Connect
* `Kafka Streams`는 윈도우 연산, 조인(Join), 집계(Aggregation) 등 **상태를 유지**해야 하는 복잡한 연산에 적합하며,
  * filter, map, branch 등 단순 변환 작업에 해당하는 Stateless 처리도 가능하다.
  * 즉, Kafka Streams는 Stateful 처리에 최적화되어 있지만, Stateless 작업도 수행 가능
* `Kafka Connect`는 **데이터 이동에 특화**되어 있음
  * Kafka와 외부 시스템(DB, 클라우드, 파일 등) 간 데이터 이동을 담당
  * 복잡한 연산이나 상태 관리(stateful data 관리)는 불가능함
  * 사전 정의된 커넥터(JDBC, Elasticsearch 등)를 사용해 코드 없이 데이터 연동 가능하다는 장점이 큼
  * 즉, Kafka Connect는 Stateless 데이터 이동에 특화되어 있으며, 복잡한 처리보다는 **데이터 연동이 목적**
* 따라서, 두 도구는 목적이 다르며 용도에 따라 사용하는 것이 일반적
  * ex. Kafka Connect로 데이터 수집 -> Kafka Streams로 실시간 처리 -> Kafka Connect로 결과 저장

<br>

## 9.1. Kafka Streams와 Kafka Connect를 포함한 카프카 아키텍처 예시
* ![](2025-04-29-19-26-14.png)