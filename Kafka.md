
좋아요. 지금까지 상황을 정리하고 해결책을 **"config 큰 변경 없이"**, 그리고 **버전 업데이트된 `confluentinc/cp-zookeeper:7.6.1`, `confluentinc/cp-kafka:7.6.1` 기준**으로 다시 구성해드릴게요. 또한 **ARM64 (M1/M2/M3, Apple Silicon)** 호환도 함께 고려하겠습니다.

---

## ✅ 문제 요약

* 에러: `Connection to node -1 (/127.0.0.1:9092) could not be established`
* 원인:

  1. Kafka 브로커 `advertised.listeners` 설정 문제
  2. Kafka 도커 이미지가 ARM64와 호환되지 않아 실행 실패 또는 이상동작
* 추가 메시지: `platform mismatch (amd64 vs arm64)` → M1/M2 Mac 환경

---

## ✅ 해결 방안 요약 (변경 최소화)

### 1. Docker Compose (Kafka/Zookeeper) 설정 예시 (ARM64 호환, 버전 7.6.1)

```yaml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.6.1
    platform: linux/amd64
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    ports:
      - "2181:2181"

  broker:
    image: confluentinc/cp-kafka:7.6.1
    platform: linux/amd64
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
```

🔧 **변경 사항 요약**

* `platform: linux/amd64` 추가하여 M1/M2 환경에서도 x86 이미지 실행 가능
* `KAFKA_ADVERTISED_LISTENERS` 를 `localhost:9092` 로 고정 → 클라이언트 접근 가능
* Kafka/Zookeeper 최신 버전 `7.6.1` 사용

---

## ✅ 실행 명령어

```bash
docker-compose up -d
```

### 확인:

```bash
docker-compose ps
# → zookeeper와 broker 둘 다 "Up" 상태여야 함

docker logs broker -f
# → "started (kafka.server.KafkaServer)" 로그가 보여야 정상
```

---

## ✅ 클라이언트 연결 확인 (예: kafka-console-producer)

```bash
docker exec -it <broker-container-id> kafka-topics \
  --bootstrap-server localhost:9092 \
  --create --topic test-topic --partitions 1 --replication-factor 1

docker exec -it <broker-container-id> kafka-console-producer \
  --broker-list localhost:9092 \
  --topic test-topic
```

---

## ✅ 추가 팁 (Kafka 클라이언트 연결 오류 방지)

만약 **외부 애플리케이션(Spring, Node.js, Python 등)** 이 Kafka에 접속할 경우, `localhost` 대신 `host.docker.internal` 혹은 실제 IP로 변경 필요:

```yaml
# 예시 (Mac에서 외부 접속 허용)
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://host.docker.internal:9092
```

---

필요하면 `docker-compose.override.yml` 로 ARM64 전용 설정 따로 분리해도 돼요.

---

## ✅ 마무리

Daniel의 기존 구성에서 **크게 변경하지 않으면서** 문제를 해결할 수 있는 방법을 적용했고, 최신 버전도 반영했어요. 다음과 같이 정리 가능합니다:

| 항목                   | 기존    | 변경 사항                      |
| -------------------- | ----- | -------------------------- |
| Kafka/Zookeeper 이미지  | 구버전   | `7.6.1` 최신                 |
| 플랫폼 호환성              | 오류    | `platform: linux/amd64` 추가 |
| advertised.listeners | 일부 누락 | 명시적으로 `localhost:9092` 고정  |

---

필요하면 Spring 쪽 설정도 예제로 알려드릴 수 있어요.
다시 연결이 안 되거나 에러 메시지가 조금이라도 달라지면 전체 로그랑 함께 공유해주세요.


----

좋습니다! 원래 사용 중인 `confluentinc/cp-server:6.2.1` 이미지를 그대로 유지하면서, **Mac M3(ARM64) 환경에서도 문제없이 실행되도록 최소한의 수정**만 적용한 `docker-compose.yml`을 아래에 제공합니다.

---

## ✅ 수정 포인트 요약

| 항목                      | 설명                                    |
| ----------------------- | ------------------------------------- |
| `platform: linux/amd64` | M3에서 x86 이미지 실행 가능하도록 지정 (QEMU 에뮬레이션) |
| `KAFKA_LISTENERS` 추가    | `localhost:9092` 실제 바인딩 안 되던 문제 해결    |
| 나머지 환경 변수               | 그대로 유지                                |

---

## ✅ 수정된 Kafka Compose 설정 (기존 구조 유지)

```yaml
services:
  zookeeper:
    image: confluentinc/cp-zookeeper:6.2.1
    platform: linux/amd64  # ✅ 추가
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
      ZOOKEEPER_SYNC_LIMIT: 2

  broker:
    image: confluentinc/cp-server:6.2.1
    platform: linux/amd64  # ✅ 추가
    hostname: broker
    container_name: broker
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
      - "9101:9101"
    expose:
      - "9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'

      # ✅ 실제 포트를 바인딩하게 명시 (없으면 localhost:9092 못 씀)
      KAFKA_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://0.0.0.0:9092

      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092

      KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_CONFLUENT_LICENSE_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_CONFLUENT_BALANCER_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_JMX_PORT: 9101
      KAFKA_CONFLUENT_SCHEMA_REGISTRY_URL: http://schema-registry:8081

      CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: broker:29092
      CONFLUENT_METRICS_REPORTER_ZOOKEEPER_CONNECT: zookeeper:2181
      CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
      CONFLUENT_METRICS_ENABLE: 'true'
      CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'
```

---

## ✅ 클라이언트 설정 예시 (Spring, CLI 등)

```properties
bootstrap.servers=localhost:9092
```

> `localhost:9092` 로 접속해야 하며, 위 설정에서 `PLAINTEXT_HOST://localhost:9092`가 브로커가 클라이언트에게 알려주는 주소로 광고됩니다.

---

## ✅ 실행 후 테스트

```bash
# Kafka topic list
docker exec -it broker kafka-topics --bootstrap-server localhost:9092 --list

# 새 토픽 생성
docker exec -it broker kafka-topics --create --bootstrap-server localhost:9092 --topic test-topic --partitions 1 --replication-factor 1
```

---

필요하면 `schema-registry`, `kafka-rest`, `kafka-ui`, `connect` 등도 이 구조에 맞춰 확장해드릴 수 있습니다.


----------------

아, 이 메시지는 \*\*MacBook M3 (ARM64 아키텍처)\*\*에서 `confluentinc/cp-server:6.2.1`과 같은 **x86\_64 전용 이미지**를 돌릴 때 흔히 뜨는 경고입니다:

---

## ⚠️ 메시지 해석:

```
The requested image's platform (linux/amd64) does not match the detected host platform (linux/arm64/v8)
```

이 말은:

* 너의 Mac은 `ARM64` (M1, M2, M3) 아키텍처인데
* Kafka 이미지(`cp-server`)는 `x86_64` 용이기 때문에
* Docker가 내부적으로 **QEMU 에뮬레이션**을 사용해서 돌릴 수는 있지만, 느리거나 불안정할 수 있다

> 즉, 이건 **실행은 되지만 성능 저하나 비정상적인 동작 가능성이 있는 상태**라는 경고입니다.

---

## ✅ 해결 방법

### 🔧 방법 1: `platform`을 명시하여 명확히 돌리기 (에뮬레이션 허용)

```yaml
broker:
  image: confluentinc/cp-server:6.2.1
  platform: linux/amd64
```

또한 `zookeeper`에도 추가:

```yaml
zookeeper:
  image: confluentinc/cp-zookeeper:6.2.1
  platform: linux/amd64
```

> 이렇게 하면 Docker가 에뮬레이션을 확실히 적용해서 "에러"는 안 나고, "성능은 조금 느릴 수 있음" 정도로 사용할 수 있습니다.

---

### 🔧 방법 2: ARM64 지원 Kafka 이미지로 교체 (Bitnami 추천)

Confluent 이미지 대신 **Bitnami Kafka**를 쓰면 M1/M2/M3에서 네이티브로 빠르게 동작합니다.

```yaml
image: bitnami/kafka:3.7
```

Bitnami는 `linux/arm64`와 `linux/amd64` 둘 다 지원하는 멀티 아키텍처 이미지를 제공합니다.

> 단점: Confluent 기능 (`cp-server`, Schema Registry, Control Center 등)를 쓰려면 추가 설정 필요

---

## ✅ 결론

| 옵션                         | 설명                               | 추천 상황                                  |
| -------------------------- | -------------------------------- | -------------------------------------- |
| `platform: linux/amd64` 추가 | 기존 Confluent 이미지 유지 가능, 에뮬레이션 사용 | Confluent Platform 사용 꼭 필요할 때          |
| Bitnami 이미지로 변경            | M3에서 네이티브 실행, 빠름                 | Schema Registry 등 필요 없거나, 직접 붙일 수 있을 때 |

---

Daniel처럼 **Kafka 테스트/로컬 개발 환경을 빠르고 안정적으로 만들고 싶다면** → `bitnami/kafka` + `bitnami/zookeeper`가 좋고,
**Confluent 기능이 필요하면** → `platform: linux/amd64`를 붙여서 에뮬레이션으로 실행하세요.

필요하시면 둘 중 하나로 전체 Compose 파일 구성 도와드릴게요.


----
좋습니다! 기존의 `confluentinc/cp-server:6.2.1` 이미지 형식을 유지하면서 **Kafka 클라이언트가 `localhost:9092`로 접속 가능하게 만들기 위한 설정만 최소 변경**한 버전을 아래에 제공합니다.

---

## ✅ 문제 핵심 요약 (재확인)

현재 설정의 문제는 `KAFKA_ADVERTISED_LISTENERS`에 \*\*`PLAINTEXT_HOST://localhost:9092`\*\*가 있지만,
Kafka 내부 설정상 **해당 리스너가 `LISTENERS`에 없어서 실제 바인딩되지 않음**입니다.

---

## ✅ 수정된 설정 (`LISTENERS` 추가 포함)

```yaml
broker:
  image: confluentinc/cp-server:6.2.1
  hostname: broker
  container_name: broker
  depends_on:
    - zookeeper
  ports:
    - "9092:9092"
    - "9101:9101"
  expose:
    - "9092"
  environment:
    KAFKA_BROKER_ID: 1
    KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
    # 👇 LISTENERS 추가
    KAFKA_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://0.0.0.0:9092
    KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
    KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
    KAFKA_METRIC_REPORTERS: io.confluent.metrics.reporter.ConfluentMetricsReporter
    KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
    KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
    KAFKA_CONFLUENT_LICENSE_TOPIC_REPLICATION_FACTOR: 1
    KAFKA_CONFLUENT_BALANCER_TOPIC_REPLICATION_FACTOR: 1
    KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
    KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
    KAFKA_JMX_PORT: 9101
    KAFKA_CONFLUENT_SCHEMA_REGISTRY_URL: http://schema-registry:8081
    CONFLUENT_METRICS_REPORTER_BOOTSTRAP_SERVERS: broker:29092
    CONFLUENT_METRICS_REPORTER_ZOOKEEPER_CONNECT: zookeeper:2181
    CONFLUENT_METRICS_REPORTER_TOPIC_REPLICAS: 1
    CONFLUENT_METRICS_ENABLE: 'true'
    CONFLUENT_SUPPORT_CUSTOMER_ID: 'anonymous'
```

---

## ✅ 핵심 변경 포인트 요약

| 항목                           | 원래 값   | 변경 후                                                       |
| ---------------------------- | ------ | ---------------------------------------------------------- |
| `KAFKA_LISTENERS`            | ❌ 없음   | ✅ `PLAINTEXT://broker:29092,PLAINTEXT_HOST://0.0.0.0:9092` |
| `KAFKA_ADVERTISED_LISTENERS` | 그대로 유지 | ✅ `localhost:9092` 는 이제 실제 바인딩됨                            |
| 포트                           | 그대로 유지 | ✅ 외부 클라이언트가 `localhost:9092` 로 접속 가능해짐                     |

---

## ✅ 클라이언트 설정 예

Kafka 클라이언트(Java 등)에서 다음과 같이 설정합니다:

```properties
bootstrap.servers=localhost:9092
```

---

## ✅ 테스트 확인 방법

Kafka가 제대로 리스닝하고 광고(advertise)하고 있는지 확인하려면:

```bash
docker exec -it broker bash
kafka-topics --bootstrap-server localhost:9092 --list
```

또는

```bash
nc -vz localhost 9092
```

로 포트 오픈 여부 확인.

---

필요하시면 여기에 Schema Registry, Kafka REST Proxy, Kafka UI, Connect 등을 붙인 `confluent` 풀스택 구성도 드릴 수 있습니다.


-----

네, **Docker Desktop에서도 동일한 문제가 발생한다면 거의 100% `KAFKA_ADVERTISED_LISTENERS` 설정 이슈**입니다. 특히 다음과 같은 상황일 가능성이 높습니다:

---

## 🔍 핵심 요약

| 환경                       | 접속 실패 원인 가능성                                      |
| ------------------------ | ------------------------------------------------- |
| Docker Desktop (x86/ARM) | ❗ Kafka 브로커가 `localhost:9092`로 광고(advertise)하지 않음 |
| Colima                   | ❗ 위 + 네트워크 격리 이슈                                  |

---

## ✅ Kafka의 핵심 개념 다시 보기

Kafka는 **"클라이언트가 어디로 접속하라고 알려줄지"를 `advertised.listeners`로 설정**합니다.

### 예시:

```yaml
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
```

이렇게 해야 외부에서 클라이언트가 `localhost:9092`로 접속할 수 있습니다.
반대로 **`broker:29092`** 같은 내부 주소만 설정해 두면 외부에서는 접속 불가합니다.

---

## 🔥 잘못된 설정의 전형적 사례

```yaml
KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://broker:29092
```

* 이건 Docker 네트워크 내부에서만 접근 가능
* **Docker 밖의 호스트(개발자 Macbook 등)** 에선 `broker`라는 이름은 알 수 없음 → 접속 실패

---

## ✅ Docker Desktop에서도 되는 확실한 설정

```yaml
services:
  kafka:
    image: bitnami/kafka:3.7
    ports:
      - "9092:9092"
    environment:
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
      - ALLOW_PLAINTEXT_LISTENER=yes
```

* `advertised.listeners`에 `localhost:9092` 명시
* 클라이언트에서도 `bootstrap.servers=localhost:9092` 사용 가능

---

## 🧪 확인 방법

Kafka 컨테이너에서 아래 명령으로 확인할 수 있습니다:

```bash
docker exec -it kafka bash
kafka-configs.sh --bootstrap-server localhost:9092 --entity-type brokers --entity-name 1 --describe
```

여기서 `advertised.listeners` 가 `localhost:9092`로 잡혀 있어야 외부에서 접근됩니다.

---

## ✅ 결론

* ✅ Docker Desktop이든 Colima든 **Kafka 접속 안 되는 문제는 대부분 `advertised.listeners` 설정 때문**
* ✅ 외부 클라이언트에서 `localhost:9092` 접속하려면, Kafka도 그걸 advertise해야 함
* 🧠 Docker 안의 이름(`broker`, `kafka`)은 외부에서는 모름
* ✅ 해결: `PLAINTEXT://localhost:9092` 설정 or 실제 호스트 IP 사용

---

필요하시면 Docker Desktop 기준으로 `zookeeper + kafka + ui` 전체 예제도 드릴 수 있어요.


----

좋아요! 그럼 MacBook M3 + Colima 환경에서 잘 작동하는 Kafka + Zookeeper용 `docker-compose.yml` 예제를 아래에 제공합니다.

---

## ✅ Mac M1/M2/M3 + Colima 환경용 Kafka `docker-compose.yml`

```yaml
version: '3.8'

services:
  zookeeper:
    image: bitnami/zookeeper:3.8
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes

  kafka:
    image: bitnami/kafka:3.7
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      - KAFKA_BROKER_ID=1
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_CFG_LISTENER_SECURITY_PROTOCOL_MAP=PLAINTEXT:PLAINTEXT
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://host.docker.internal:9092
      - KAFKA_CFG_AUTO_CREATE_TOPICS_ENABLE=true
      - ALLOW_PLAINTEXT_LISTENER=yes
    depends_on:
      - zookeeper
    extra_hosts:
      - "host.docker.internal:host-gateway"
```

---

## 💡 주요 설정 설명

| 항목                          | 설정                     | 설명                                                |
| --------------------------- | ---------------------- | ------------------------------------------------- |
| `bitnami/kafka:3.7`         | ARM64 지원               | M3에서도 native 실행 가능                                |
| `host.docker.internal:9092` | `advertised.listeners` | Kafka 클라이언트가 실제 접속할 주소. macOS에선 Colima가 이 주소로 연결됨 |
| `extra_hosts`               | `host-gateway` 설정      | Colima에서는 이 설정이 있어야 `host.docker.internal`이 동작함   |

---

## 🧪 Kafka 테스트 명령

컨테이너 안에 들어가서 직접 topic을 만들고 확인해볼 수 있습니다.

```bash
docker exec -it kafka bash
# 토픽 생성
kafka-topics.sh --create --topic test-topic --bootstrap-server localhost:9092 --partitions 1 --replication-factor 1

# 토픽 목록 확인
kafka-topics.sh --list --bootstrap-server localhost:9092
```

---

## 💬 Kafka 클라이언트 설정 예 (Java 등)

```properties
bootstrap.servers=host.docker.internal:9092
```

Colima에서는 **절대 `localhost:9092` 쓰지 마세요**, 동작 안 합니다. 반드시 `host.docker.internal:9092` 또는 실제 호스트 IP를 사용해야 합니다.

---

필요하시면 Schema Registry, Kafka UI, Kafka Connect 등 추가된 구성도 도와드릴게요.


----

좋은 질문입니다. MacBook M3 칩셋(Apple Silicon, ARM64 아키텍처) 환경에서 Kafka Docker 이미지 및 Kafka 클라이언트가 잘 작동하는지 확인하려는 것이죠. 이 문제는 최근 Kafka나 관련 도구를 M1/M2/M3 맥에서 돌릴 때 꽤 자주 나오는 이슈입니다.

---

## ✅ 1. **Kafka Docker 이미지와 M3 (ARM64) 호환성**

### 🔍 현재 사용 중인 이미지:

```yaml
image: confluentinc/cp-server:6.2.1
```

이 이미지(`cp-server:6.2.1`)는 **x86-64 전용 이미지**입니다.
즉, **Apple M3 (ARM64)** 환경에서는 **에뮬레이션 (Rosetta/Colima QEMU 등)** 으로 돌아가야 합니다. 이 과정에서 네트워크 관련 문제나 성능 저하가 생길 수 있습니다.

---

## ✅ 2. Kafka 클라이언트와 M3 호환성

Kafka Java 클라이언트 라이브러리 (`kafka-clients`)는 Pure Java이므로 **M1/M2/M3 맥북에서 기본적으로 호환됩니다.**
단, Kafka 클라이언트가 연결하는 주소(`localhost:9092` 등)가 **Docker 네트워크 바깥으로 잘 노출되어 있어야 함**은 여전히 중요합니다.

---

## ✅ 3. 해결 방법 (실제 사용 가능한 방법)

### 🟡 방법 1: ARM64 대응 Kafka 이미지 사용

M3 맥에서 Kafka를 네이티브하게 잘 돌리고 싶다면 **ARM64를 지원하는 이미지**를 사용하는 것을 추천합니다.

```yaml
image: bitnami/kafka:3.7.0
```

`bitnami/kafka` 이미지는 `multi-arch (ARM64/x86)` 지원되며, M1\~M3에서 잘 동작합니다.

### 🛠 docker-compose 예시:

```yaml
version: '3'

services:
  zookeeper:
    image: bitnami/zookeeper:3.8
    ports:
      - "2181:2181"
    environment:
      - ALLOW_ANONYMOUS_LOGIN=yes

  kafka:
    image: bitnami/kafka:3.7
    ports:
      - "9092:9092"
    environment:
      - KAFKA_BROKER_ID=1
      - KAFKA_CFG_ZOOKEEPER_CONNECT=zookeeper:2181
      - KAFKA_CFG_LISTENERS=PLAINTEXT://:9092
      - KAFKA_CFG_ADVERTISED_LISTENERS=PLAINTEXT://host.docker.internal:9092
      - ALLOW_PLAINTEXT_LISTENER=yes
    depends_on:
      - zookeeper
```

> ✅ `host.docker.internal`이 Colima에서 동작하지 않는다면 실제 호스트 IP를 넣어주세요.

---

## ✅ 4. 결론: M3 맥에서 Kafka 문제 요약

| 항목                                   | 문제 여부                    | 해결 방법                               |
| ------------------------------------ | ------------------------ | ----------------------------------- |
| Kafka 클라이언트 (Java 등)                 | ✅ 정상 작동                  | Pure Java이므로 M3 호환                  |
| Kafka Docker 이미지 (`cp-server:6.2.1`) | ⚠️ x86 only → 느리거나 실패 가능 | ARM64 지원 이미지(`bitnami/kafka`) 사용    |
| `localhost:9092` 접근                  | ⚠️ Colima에선 실패 가능        | `host.docker.internal` 또는 호스트 IP 사용 |
| Zookeeper                            | ✅ 문제 없음                  | `bitnami/zookeeper` 추천              |

---

필요하시면 Bitnami Kafka 기반으로 Colima에 최적화된 전체 docker-compose 예제를 구성해드릴게요.



----

주어진 Kafka Docker Compose 설정에서 **`localhost:9092` 접속 실패** 문제가 발생하는 주요 원인은 다음 두 가지 중 하나입니다:

---

### ✅ 1. **`KAFKA_ADVERTISED_LISTENERS` 설정 문제**

Kafka의 `advertised.listeners` 설정은 클라이언트가 접속할 때 사용하는 브로커의 주소를 결정합니다.
현재 설정은 다음과 같습니다:

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://localhost:9092
```

이 의미는 다음과 같습니다:

* **`broker:29092`** → Docker 내부 컨테이너 간 통신 (예: `schema-registry`가 Kafka와 통신할 때 사용)
* **`localhost:9092`** → Docker 밖(호스트)에서 Kafka 브로커에 접근할 때 사용

문제는 `localhost:9092` 포트를 **컨테이너에서 바인딩되지 않았거나**, **Colima와 Docker 네트워크 간의 호환성 문제**로 인해 실제로 브로커가 **`localhost:9092`를 바인딩하지 못한 경우**입니다.

---

### ✅ 2. **Colima의 네트워크 구조 차이**

Colima (특히 macOS 상에서 Lima 기반으로 동작)는 Docker Desktop과는 다르게 **네트워크 bridge** 방식이 제한됩니다. 일반적인 Docker Desktop에서는 `localhost:9092`가 곧바로 호스트와 매핑되지만, Colima에서는 네트워크 분리가 되어 있어 **호스트에서 `localhost:9092`로 접속하면 컨테이너 내부의 Kafka에 도달하지 못합니다.**

---

## 🔧 해결 방법

### 🔁 방법 1: **`advertised.listeners` 수정**

Kafka 브로커가 외부에서 올바르게 접근되도록 하려면, 다음과 같이 설정을 변경해 보세요:

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://host.docker.internal:9092
```

또는 Colima에서는 `host.docker.internal`이 동작하지 않을 수 있으니 **호스트 IP를 직접 쓰는 방법도 있습니다.**

```bash
HOST_IP=$(ipconfig getifaddr en0)  # macOS에서 로컬 IP 구하기
```

예:

```yaml
KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://broker:29092,PLAINTEXT_HOST://192.168.1.100:9092
```

이 IP는 **Kafka 클라이언트가 접속할 주소**이므로 실제 호스트에서 접속 가능한 IP여야 합니다.

---

### 🔁 방법 2: **Colima에 `host.docker.internal` 지원 추가**

Colima는 `host.docker.internal`을 기본으로 지원하지 않을 수 있습니다. 이 경우 다음 명령으로 `/etc/hosts`에 매핑하거나 host gateway를 명시할 수 있습니다.

```bash
colima start --network-address
```

그리고 `docker-compose.yml`에서 다음과 같이 설정:

```yaml
extra_hosts:
  - "host.docker.internal:host-gateway"
```

---

## 🧪 테스트

Kafka 클라이언트에서 `bootstrap.servers=localhost:9092` 대신, `192.168.1.100:9092` 또는 `host.docker.internal:9092`로 설정해 보세요.

또는 브로커 컨테이너 내부에서 접속해도 되면:

```bash
docker exec -it broker bash
kafka-topics --bootstrap-server broker:29092 --list
```

---

## ✅ 최종 권장 요약

1. **Colima에서 Kafka 접속하려면 `advertised.listeners`를 호스트 IP에 맞게 설정**
2. **가능하면 `host.docker.internal` 사용, 안되면 직접 IP 사용**
3. **클라이언트에서도 `localhost:9092` 말고 위에서 설정한 IP로 접속**

---

필요하시면 Colima + Kafka를 위한 `docker-compose.yml` 완성본도 예제로 드릴 수 있어요.
