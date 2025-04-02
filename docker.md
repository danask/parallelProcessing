Kafka를 Docker로 실행하고 Spring Boot 애플리케이션에서 이를 사용하려고 할 때, 처음에는 정상 작동하지만 이후 `org.apache.kafka.common.errors.TimeoutException: Timed out waiting to send the call. Call: fetchMetadata` 오류가 발생하는 경우, 몇 가지 원인이 있을 수 있습니다.  

---

## 🔍 **원인 분석 및 해결 방법**
### 1️⃣ **Kafka 컨테이너가 종료되거나 재시작됨**  
#### ✅ **확인 방법**  
Kafka 컨테이너의 상태를 확인하세요.
```bash
docker ps -a | grep kafka
```
Kafka 컨테이너가 실행 중인지 확인하고, 재시작된 경우 `docker logs kafka_container_name`을 확인해 보세요.  

#### 🔹 **해결 방법**  
- 컨테이너가 종료되는 경우, `docker-compose`를 사용하여 `restart: always` 설정을 추가하세요.
- 컨테이너를 **백그라운드 실행 (`-d`)** 하도록 설정하세요.

---

### 2️⃣ **Kafka와 Zookeeper가 제대로 연결되지 않음**
Kafka는 Zookeeper에 의존하므로, Zookeeper가 제대로 실행되지 않으면 Kafka도 정상적으로 동작하지 않습니다.  

#### ✅ **확인 방법**  
```bash
docker logs zookeeper_container_name
```
Zookeeper 로그에서 오류가 발생하는지 확인하세요.  

#### 🔹 **해결 방법**  
- `docker-compose.yml`에서 Kafka가 Zookeeper를 의존하도록 설정하세요.
- 또는 **Kafka 컨테이너 실행 전에 Zookeeper가 완전히 실행되었는지 확인**하세요.

---

### 3️⃣ **Kafka 브로커의 `advertised.listeners` 설정 문제**
Kafka의 `advertised.listeners` 설정이 잘못된 경우, Spring Boot 애플리케이션이 Kafka 브로커를 찾지 못할 수 있습니다.  

#### ✅ **확인 방법**  
Kafka 컨테이너 내부에서 `advertised.listeners` 설정을 확인하세요.
```bash
docker exec -it kafka_container_name cat /etc/kafka/server.properties | grep advertised.listeners
```

#### 🔹 **해결 방법**  
- `advertised.listeners`를 **올바르게 설정**해야 합니다.  
  - Docker 내부에서 실행 시:
    ```bash
    KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092
    ```
  - 외부에서 접근할 경우:
    ```bash
    KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://localhost:9092
    ```

**`docker-compose.yml` 예시:**
```yaml
services:
  zookeeper:
    image: wurstmeister/zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"

  kafka:
    image: wurstmeister/kafka
    container_name: kafka
    ports:
      - "9092:9092"
    environment:
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:9092
      KAFKA_LISTENERS: PLAINTEXT://0.0.0.0:9092
```

---

### 4️⃣ **Kafka 클러스터 상태 확인**
Kafka 토픽과 클러스터의 상태를 확인해야 합니다.

#### ✅ **토픽 확인 방법**
Kafka가 정상적으로 실행되고 있는지 확인하려면, Kafka 컨테이너 내부에서 토픽 목록을 확인해 보세요.
```bash
docker exec -it kafka_container_name kafka-topics.sh --list --bootstrap-server kafka:9092
```
토픽이 존재하는지 확인하세요.

#### 🔹 **해결 방법**
토픽이 없다면, Spring Boot 애플리케이션이 실행되기 전에 토픽을 미리 생성하세요.
```bash
docker exec -it kafka_container_name kafka-topics.sh --create --topic my-topic --bootstrap-server kafka:9092 --partitions 3 --replication-factor 1
```

---

### 5️⃣ **Spring Boot 설정 문제 (`bootstrap.servers` 설정 확인)**
Spring Boot에서 Kafka의 `bootstrap.servers` 설정이 올바르게 되어 있는지 확인하세요.

#### ✅ **application.yml / application.properties 확인**
```yaml
spring:
  kafka:
    bootstrap-servers: kafka:9092
```
**🚨 주의:**  
- `bootstrap-servers: localhost:9092`로 되어 있으면 Docker 컨테이너 내부에서는 접근할 수 없습니다.  
- Docker 내부에서 실행할 경우 `kafka:9092`와 같이 컨테이너 이름을 사용해야 합니다.

---

### 6️⃣ **Docker 네트워크 문제**
Kafka와 Spring Boot가 서로 다른 네트워크에 있으면, `fetchMetadata` 호출이 타임아웃될 수 있습니다.

#### ✅ **확인 방법**
```bash
docker network ls
```
Kafka와 Spring Boot가 같은 네트워크에 속해 있는지 확인하세요.

#### 🔹 **해결 방법**
Kafka와 Spring Boot를 **같은 Docker 네트워크에서 실행**하도록 설정하세요.
```yaml
networks:
  kafka_network:
    driver: bridge

services:
  kafka:
    networks:
      - kafka_network

  springboot-app:
    networks:
      - kafka_network
```

---

## 🏁 **결론**
✅ 처음에는 Kafka가 정상 동작하지만, 이후 타임아웃이 발생하는 경우 주된 원인은:
1. Kafka 컨테이너가 종료되거나 재시작됨 → `docker logs kafka` 확인  
2. Zookeeper와 연결 문제 → `docker logs zookeeper` 확인  
3. `advertised.listeners` 설정 문제 → `KAFKA_ADVERTISED_LISTENERS=PLAINTEXT://kafka:9092`  
4. Kafka 토픽이 생성되지 않음 → `kafka-topics.sh --list` 확인  
5. Spring Boot의 `bootstrap.servers` 설정 오류 → `bootstrap-servers: kafka:9092`  
6. Docker 네트워크 문제 → Kafka와 Spring Boot를 같은 네트워크에서 실행  

위 해결 방법을 하나씩 시도해보면 문제를 해결할 수 있을 거예요. 🚀

--------------------
Redis Docker 컨테이너에 외부에서 접근할 수 있도록 방화벽 및 네트워크 설정을 확인하고 필요한 경우 포트 6379를 허용하는 방법을 아래에 설명하겠습니다.

### macOS에서 방화벽 설정 확인 및 변경

macOS에서는 기본적으로 내장된 애플리케이션 방화벽을 사용합니다. 아래 단계에 따라 방화벽 설정을 확인하고 필요한 경우 변경할 수 있습니다.

1. **시스템 환경설정 열기**:
   - Apple 메뉴에서 `시스템 환경설정`을 클릭합니다.

2. **보안 및 개인 정보 보호**:
   - `보안 및 개인 정보 보호`를 클릭합니다.

3. **방화벽 탭**:
   - `방화벽` 탭을 선택합니다.
   - 방화벽이 켜져 있는지 확인합니다.

4. **방화벽 옵션**:
   - `방화벽 옵션...` 버튼을 클릭합니다.
   - 필요한 경우 관리자로 인증합니다.

5. **포트 열기**:
   - macOS 방화벽에서는 개별 포트를 열 수 없습니다. 대신 애플리케이션을 통해 포트를 열 수 있습니다.
   - Docker 및 Redis가 네트워크 접근을 허용하도록 설정되어 있는지 확인합니다.
   - 방화벽 옵션에서 Docker 관련 항목이 `허용`으로 설정되어 있는지 확인합니다.

### macOS 터미널을 이용한 포트 상태 확인

macOS에서 특정 포트가 열려 있는지 확인하려면 `nc`(netcat) 또는 `telnet` 명령어를 사용할 수 있습니다.

#### `nc` 명령어 사용:
터미널을 열고 다음 명령어를 실행하여 포트가 열려 있는지 확인합니다:

```sh
nc -zv localhost 6379
```

정상적으로 열려 있다면 다음과 같은 메시지가 표시됩니다:

```
Connection to localhost port 6379 [tcp/redis] succeeded!
```

#### `telnet` 명령어 사용:
`telnet` 명령어를 사용하여 포트가 열려 있는지 확인합니다:

```sh
telnet localhost 6379
```

정상적으로 열려 있다면 Redis 서버와의 연결 메시지가 표시됩니다.

### macOS에서 ufw (Uncomplicated Firewall) 설정 (필요한 경우)

기본적으로 macOS는 ufw를 사용하지 않지만, ufw를 설치하여 관리할 수도 있습니다. Homebrew를 통해 ufw를 설치하고 설정할 수 있습니다.

1. **Homebrew로 ufw 설치**:
   ```sh
   brew install ufw
   ```

2. **ufw 활성화**:
   ```sh
   sudo ufw enable
   ```

3. **포트 6379 허용**:
   ```sh
   sudo ufw allow 6379
   ```

4. **ufw 상태 확인**:
   ```sh
   sudo ufw status
   ```

### 요약
1. **시스템 환경설정**에서 방화벽 설정을 확인하고 Docker 관련 항목이 허용되도록 설정합니다.
2. **nc** 또는 **telnet** 명령어를 사용하여 포트 6379가 열려 있는지 확인합니다.
3. 필요한 경우 **ufw**를 설치하고 포트 6379를 허용합니다.

위 단계를 따르면 방화벽 및 네트워크 설정을 통해 Redis Docker 컨테이너에 외부에서 접근할 수 있는지 확인할 수 있습니다.
------------------------------

Colima는 macOS에서 로컬 Kubernetes 및 Docker를 실행하기 위한 경량의 VM 관리 도구입니다. Colima를 업그레이드하고 모든 VM 이미지를 삭제하려면 다음 단계를 따르세요.

### 1. Colima 및 기존 VM 이미지 삭제
먼저, 현재 Colima와 모든 VM 이미지를 삭제합니다.

#### Colima와 모든 VM 삭제
터미널에서 다음 명령어를 실행합니다:

```sh
colima stop
colima delete --force
```

### 2. Colima 업그레이드
Colima를 업그레이드하려면 Homebrew를 사용합니다. 터미널에서 다음 명령어를 실행하세요:

```sh
brew update
brew upgrade colima
```

### 3. Colima 설정 및 시작
Colima를 업그레이드한 후 새로 설정하고 시작합니다.

#### Colima 설치 확인
Colima가 업그레이드되었는지 확인합니다:

```sh
colima version
```

#### Colima 시작
Colima를 새로 시작합니다. 기본 Docker 환경을 설정하려면 다음 명령어를 실행하세요:

```sh
colima start
```

기본적으로 Colima는 Docker를 지원합니다. Kubernetes 환경을 설정하려면 다음 명령어를 실행하세요:

```sh
colima start --kubernetes
```

### 4. Docker 및 Kubernetes 확인
Colima가 제대로 동작하는지 확인합니다.

#### Docker 확인
Docker가 실행 중인지 확인합니다:

```sh
docker version
docker ps
```

#### Kubernetes 확인
Kubernetes가 실행 중인지 확인합니다:

```sh
kubectl version
kubectl get nodes
```

### 문제 해결
Colima 업그레이드 및 VM 이미지 삭제 후 문제가 발생할 경우 다음 명령어를 사용하여 로그를 확인하고 문제를 해결할 수 있습니다:

```sh
colima status
colima logs
```

위 단계를 따르면 Colima를 성공적으로 업그레이드하고 모든 VM 이미지를 삭제할 수 있습니다. Colima가 정상적으로 동작하는지 확인한 후 Docker 및 Kubernetes 환경을 다시 설정하여 사용하면 됩니다.
