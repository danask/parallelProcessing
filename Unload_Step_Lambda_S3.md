
좋습니다. 아래는 당신의 목표를 완전하게 지원하는 **Spring Boot + Step Functions + Lambda + Kafka + Terraform 인프라 구성**의 전체 흐름 및 구현 예시입니다.

---

# ✅ 목표 정리

1. **Spring Boot**로 사용자 요청 수신 및 **Step Function 시작**
2. Step Function:

   * Redshift UNLOAD 실행 (Lambda1)
   * S3 확인 (Lambda2)
   * Kafka로 알림 전송 (Lambda3)
3. Kafka 이벤트를 **Spring Boot에서 수신 및 처리**
4. **Terraform으로 모든 AWS 자원 배포**

---

## 🧱 1. Maven 설정

### `pom.xml` 주요 의존성:

```xml
<dependencies>
  <!-- AWS SDK -->
  <dependency>
    <groupId>com.amazonaws</groupId>
    <artifactId>aws-java-sdk-stepfunctions</artifactId>
    <version>1.12.686</version>
  </dependency>

  <!-- Kafka -->
  <dependency>
    <groupId>org.springframework.kafka</groupId>
    <artifactId>spring-kafka</artifactId>
  </dependency>

  <!-- JSON -->
  <dependency>
    <groupId>com.fasterxml.jackson.core</groupId>
    <artifactId>jackson-databind</artifactId>
  </dependency>
</dependencies>
```

---

## 🔁 2. Kafka 수신 – Spring Boot Consumer

### Kafka 설정 (`application.yml`):

```yaml
spring:
  kafka:
    bootstrap-servers: localhost:9092
    consumer:
      group-id: unload-notifier
      auto-offset-reset: earliest
      key-deserializer: org.apache.kafka.common.serialization.StringDeserializer
      value-deserializer: org.apache.kafka.common.serialization.StringDeserializer
```

### Kafka Consumer 클래스:

```java
@Slf4j
@Component
public class UnloadKafkaListener {

    private final UnloadResultRepository repository;

    public UnloadKafkaListener(UnloadResultRepository repository) {
        this.repository = repository;
    }

    @KafkaListener(topics = "unload-complete", groupId = "unload-notifier")
    public void handleUnloadResult(String message) {
        try {
            ObjectMapper mapper = new ObjectMapper();
            JsonNode json = mapper.readTree(message);
            String requestId = json.get("requestId").asText();
            String s3Url = json.get("s3FileUrl").asText();

            UnloadResult result = repository.findById(requestId).orElseThrow();
            result.setStatus(UnloadResult.Status.DONE);
            result.setS3FileUrl(s3Url);
            repository.save(result);

            log.info("Updated result for requestId {} with link {}", requestId, s3Url);
        } catch (Exception e) {
            log.error("Failed to process Kafka message: {}", message, e);
        }
    }
}
```

---

## 🏗️ 3. Terraform 구성

Terraform으로 다음 자원을 생성합니다:

* Step Function
* 3개의 Lambda
* IAM Roles
* S3 Bucket (optional)
* Kafka 설정은 MSK 또는 별도 구성 가정

---

### `main.tf` 예시 (핵심)

```hcl
provider "aws" {
  region = "us-east-1"
}

resource "aws_iam_role" "lambda_exec_role" {
  name = "lambda_exec_role"
  assume_role_policy = jsonencode({
    Version = "2012-10-17",
    Statement = [{
      Action    = "sts:AssumeRole"
      Principal = { Service = "lambda.amazonaws.com" }
      Effect    = "Allow"
    }]
  })
}

resource "aws_lambda_function" "unload_lambda" {
  filename         = "unloadHandler.zip"
  function_name    = "StartUnloadFunction"
  handler          = "unloadHandler.lambda_handler"
  runtime          = "python3.9"
  role             = aws_iam_role.lambda_exec_role.arn
  source_code_hash = filebase64sha256("unloadHandler.zip")
}

resource "aws_lambda_function" "check_lambda" {
  filename         = "checkS3Handler.zip"
  function_name    = "CheckS3FileFunction"
  handler          = "checkS3Handler.lambda_handler"
  runtime          = "python3.9"
  role             = aws_iam_role.lambda_exec_role.arn
  source_code_hash = filebase64sha256("checkS3Handler.zip")
}

resource "aws_lambda_function" "kafka_lambda" {
  filename         = "sendKafkaHandler.zip"
  function_name    = "SendKafkaFunction"
  handler          = "sendKafkaHandler.lambda_handler"
  runtime          = "python3.9"
  role             = aws_iam_role.lambda_exec_role.arn
  source_code_hash = filebase64sha256("sendKafkaHandler.zip")
}

resource "aws_sfn_state_machine" "unload_state_machine" {
  name     = "RedshiftUnloadStateMachine"
  role_arn = aws_iam_role.lambda_exec_role.arn

  definition = file("unloadWorkflow.asl.json")
}
```

---

### Step Function 정의 파일 – `unloadWorkflow.asl.json`

이 파일은 위에서 만든 `.asl.json` 내용 그대로 사용합니다.

```bash
terraform apply
```

> 💡 Lambda zip 파일은 `unloadHandler.zip` 등으로 미리 빌드하고 S3로 배포하거나 `file()` 로 로컬 참조 가능

---

## 🎯 전체 흐름 요약

```
[Spring Boot API 호출]
     ↓
StepFunction 실행 (StartExecution)
     ↓
Lambda #1: Redshift UNLOAD 수행
     ↓
Lambda #2: S3 파일 존재 확인 (Wait 반복)
     ↓
Lambda #3: Kafka 이벤트 발행
     ↓
Spring Kafka Listener 수신 → DB 업데이트
```

---

## ✅ 그 외 확장 포인트

| 기능                | 설명                                               |
| ----------------- | ------------------------------------------------ |
| Pre-signed URL 발급 | 사용자가 다운로드 시 `s3Client.generatePresignedUrl()` 사용 |
| 실패 시 재시도          | Step Function의 `Retry` 속성 사용 가능                  |
| 이메일 알림 연동         | Lambda에서 SES나 Slack Webhook 호출 가능                |

---

## ✋ 다음 단계로 도와드릴 수 있는 것

* Lambda 코드를 SAM 또는 CDK로 배포
* Kafka를 MSK (Amazon Managed Streaming for Kafka)로 연동
* `CloudWatch Logs`, `DLQ`, `S3 Versioning` 등 운영 안정화
* Terraform에 output/export 추가

필요한 세부 영역을 지정해 주시면 해당 부분부터 확장해드릴게요!

----

좋습니다! 아래는 **Spring Boot 기반에서 Redshift UNLOAD → S3 → Kafka 알림까지 전체 흐름을 Step Functions로 구성하고 실행하는 구조**입니다.

---

## ✅ 구성 요소 요약

| 구성 요소              | 설명                                   |
| ------------------ | ------------------------------------ |
| Spring Boot API    | Step Function 실행 요청                  |
| AWS Step Functions | 전체 흐름 제어 (UNLOAD → 상태 확인 → Kafka 발행) |
| Lambda #1          | Redshift에 UNLOAD 실행                  |
| Lambda #2          | S3에서 파일 존재 여부 확인                     |
| Lambda #3          | Kafka 이벤트 발행                         |
| S3                 | UNLOAD 결과 저장                         |
| Kafka              | 사용자에게 알림 전파                          |

---

## ✅ 최종 아웃풋

아래 파일들을 제공합니다:

1. \[`unloadWorkflow.asl.json`] – Step Function 정의
2. \[`lambda/unloadHandler.py`] – Redshift UNLOAD 실행 Lambda
3. \[`lambda/checkS3Handler.py`] – S3 결과 파일 확인 Lambda
4. \[`lambda/sendKafkaHandler.py`] – Kafka로 결과 발송 Lambda
5. \[`application.yml`] – Spring 설정
6. \[`UnloadStepFunctionService.java`] – Spring Boot에서 Step Function 시작 코드

---

## 📄 `unloadWorkflow.asl.json` (Step Function 정의)

```json
{
  "Comment": "UNLOAD from Redshift and notify via Kafka",
  "StartAt": "StartUnload",
  "States": {
    "StartUnload": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:StartUnloadFunction",
      "Next": "WaitForFile"
    },
    "WaitForFile": {
      "Type": "Wait",
      "Seconds": 30,
      "Next": "CheckFileExists"
    },
    "CheckFileExists": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:CheckS3FileFunction",
      "Next": "FileExists?"
    },
    "FileExists?": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.fileExists",
          "BooleanEquals": true,
          "Next": "SendKafkaEvent"
        }
      ],
      "Default": "WaitForFile"
    },
    "SendKafkaEvent": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:SendKafkaFunction",
      "End": true
    }
  }
}
```

---

## 🐍 `lambda/unloadHandler.py` (Lambda #1 - UNLOAD)

```python
import psycopg2

def lambda_handler(event, context):
    request_id = event['requestId']
    sql = event['sql']
    s3_path = f"s3://your-bucket/unload-results/{request_id}/output"
    creds = "aws_access_key_id=XXX;aws_secret_access_key=YYY"

    unload_sql = f"""
        UNLOAD ('{sql}')
        TO '{s3_path}'
        CREDENTIALS '{creds}'
        DELIMITER ',' ADDQUOTES GZIP ALLOWOVERWRITE PARALLEL OFF;
    """

    conn = psycopg2.connect(...)
    with conn.cursor() as cur:
        cur.execute(unload_sql)
    conn.close()
    
    return { "requestId": request_id, "s3Path": s3_path }
```

---

## 🐍 `lambda/checkS3Handler.py` (Lambda #2 - S3 확인)

```python
import boto3

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    request_id = event['requestId']
    bucket = "your-bucket"
    key = f"unload-results/{request_id}/output000.gz"

    exists = False
    try:
        s3.head_object(Bucket=bucket, Key=key)
        exists = True
    except:
        exists = False

    event['fileExists'] = exists
    return event
```

---

## 🐍 `lambda/sendKafkaHandler.py` (Lambda #3 - Kafka 알림)

```python
from kafka import KafkaProducer
import json

def lambda_handler(event, context):
    producer = KafkaProducer(bootstrap_servers='kafka:9092')
    producer.send("unload-complete", json.dumps({
        "requestId": event['requestId'],
        "s3FileUrl": f"https://{event['s3Path'].replace('s3://', '').replace('/', '.s3.amazonaws.com/', 1)}.gz"
    }).encode('utf-8'))
    return { "status": "sent" }
```

---

## ⚙️ `application.yml` (Spring 설정)

```yaml
aws:
  stepfunction:
    arn: arn:aws:states:REGION:ACCOUNT_ID:stateMachine:RedshiftUnloadStateMachine
```

---

## ☕ `UnloadStepFunctionService.java` (Spring에서 Step Function 시작)

```java
@Service
@RequiredArgsConstructor
public class UnloadStepFunctionService {

    private final AWSStepFunctions stepFunctions;

    @Value("${aws.stepfunction.arn}")
    private String stateMachineArn;

    public void startUnload(String requestId, String sql) {
        Map<String, Object> input = Map.of(
            "requestId", requestId,
            "sql", sql
        );

        StartExecutionRequest request = new StartExecutionRequest()
            .withStateMachineArn(stateMachineArn)
            .withName("unload-" + requestId)
            .withInput(new ObjectMapper().writeValueAsString(input));

        stepFunctions.startExecution(request);
    }
}
```

> `AWSStepFunctions`는 `com.amazonaws.services.stepfunctions.AWSStepFunctionsClientBuilder.defaultClient()`로 생성 가능

---

## ✅ 호출 예시

```java
@PostMapping("/api/unload")
public ResponseEntity<?> startUnload(@RequestBody UnloadRequest req) {
    stepFunctionService.startUnload(req.getRequestId(), req.getSql());
    return ResponseEntity.ok("UNLOAD started via StepFunction");
}
```

---

## ✅ 그 다음에 할 수 있는 것

* `KafkaListener`로 결과 수신
* DB에 결과 저장 (파일 링크 포함)
* 사용자 polling API 또는 WebSocket 알림

---

필요하시면 전체 Maven 의존성, Lambda 배포 방법, Step Function 배포 스크립트 (`SAM`, `Terraform`, `CDK`) 도 도와드릴 수 있습니다.

진행하시려는 방식이 `Lambda + StepFunction 기반 운영환경`이면 다음 단계도 도와드릴게요!

---

좋은 질문입니다! Redshift의 UNLOAD 작업은 데이터 크기가 크고, 작업 자체가 **비동기처럼 느릴 수 있으며**, 실제로는 JDBC 연결에서 **쿼리가 끝날 때까지 blocking** 되기 때문에 **모니터링과 안정성 확보**가 중요합니다.

### ✅ 요약: UNLOAD 처리 흐름의 모니터링 필요 여부

| 항목                              | 필요 여부        | 이유                                                    |
| ------------------------------- | ------------ | ----------------------------------------------------- |
| **UNLOAD 실행 자체의 모니터링**          | ⚠️ **선택적**   | JDBC로 실행하면 blocking이지만, 작업이 오래 걸리거나 실패 시 catch로 처리 가능 |
| **후속 처리 (S3 완료 → Kafka 이벤트 등)** | ✅ **강력히 권장** | 이벤트가 누락되거나 실패하면 결과 전달 안 됨                             |
| **전체 workflow 추적 (end-to-end)** | ✅ **권장**     | 사용자가 `requestId`로 진행 상태를 볼 수 있어야 함                    |

---

## ✅ 선택지 1: Step Functions 도입 (권장 상황)

Step Functions는 아래와 같은 조건에 적합합니다:

* UNLOAD 작업이 길고 불안정할 수 있음
* UNLOAD 이후 Lambda, Kafka, S3 등을 엮은 **비동기 흐름**이 존재함
* 전체 워크플로우 성공/실패를 추적하고 싶음

### 예시 Step Function Workflow:

```
[Start]
   ↓
[StartUnload Lambda]   → UNLOAD 실행 (JDBC or boto3)
   ↓
[Wait / Check Status]  → Redshift query 상태 or polling
   ↓
[S3 파일 존재 확인]    → S3에 결과 파일 생성됐는지 확인
   ↓
[SendKafkaEvent Lambda]
   ↓
[Done]
```

이 구조는:

* 실패 시 재시도 설정 가능
* 작업 이력 시각화 가능 (CloudWatch)
* 상태 전이 상태 및 로그 추적 가능

---

## ✅ 선택지 2: Spring Scheduler + 상태 테이블

보다 단순한 경우에는 **Spring Boot 내부에서 상태 기반 Polling + Retry** 방식도 쓸 수 있습니다.

### 구조 예시:

1. `unload_result` 테이블에 상태값 기록 (PENDING → DONE/FAILED)
2. UNLOAD 실패 시 `status=FAILED`로 기록 후 retry 대상 선정
3. S3에 파일 존재하는지 주기적으로 확인 (`AmazonS3Client.doesObjectExist`)
4. 조건 만족 시 Kafka event 발행

### Scheduler 예시:

```java
@Scheduled(fixedDelay = 60000)
public void checkUnloadFiles() {
    List<UnloadResult> pending = repository.findByStatus(Status.PENDING);
    for (UnloadResult result : pending) {
        String key = "unload-results/" + result.getRequestId() + "/output.gz";
        if (s3Client.doesObjectExist(bucketName, key)) {
            result.setStatus(Status.DONE);
            result.setS3FileUrl(generateUrl(key));
            repository.save(result);
            kafkaTemplate.send("unload-complete", result.getRequestId());
        }
    }
}
```

---

## ✅ 선택지 3: Lambda + Kafka Event → 결과 Polling

* S3 → Lambda → Kafka → Spring Boot로 상태 갱신
* 프론트엔드는 `GET /api/unload/{id}`로 polling (직접 상태 확인)

이 방식은 Step Functions보다는 간단하지만, 중간 이벤트 누락에 대한 방어 로직이 필요합니다.

---

## 🟨 비교 정리

| 방식                           | 적합 상황                | 장점              | 단점                  |
| ---------------------------- | -------------------- | --------------- | ------------------- |
| **Step Functions**           | 전체 워크플로우 통제, 안정성 필요  | 시각화, 재시도, 상태 추적 | 복잡도 증가, 비용 발생       |
| **Spring Scheduler Polling** | 간단한 구조, UNLOAD 상태 추적 | 구현 쉬움, 비용 없음    | polling 주기/성능 조절 필요 |
| **Lambda + Kafka only**      | 이벤트 기반, 빠른 응답 원할 때   | 이벤트 기반 알림       | 누락되면 polling 병행 필요  |

---

## ✅ 결론 (추천 조합)

* **단기 MVP**:

  * `Spring Boot + JDBC UNLOAD + 상태 테이블 + Kafka + polling`
  * 간단하고 유지관리 쉬움

* **장기 운영 또는 안정성 강화 필요 시**:

  * `Step Functions + Lambda + Kafka` 조합으로 전환
  * 상태/실패 추적 및 리커버리 가능

---

필요하시면 Step Function `.asl.json` 템플릿이나 Spring Scheduler 샘플 전체 코드도 만들어드릴 수 있어요.
어떤 방식으로 갈지 결정하셨다면 알려주세요!


-----
좋습니다! Spring Boot에서 **Redshift에 UNLOAD 명령을 실행하고, S3 파일 링크를 생성하여 DB에 저장하는 전체 흐름**을 구현해드리겠습니다.

---

## ✅ 목표

1. UNLOAD 쿼리를 JDBC로 Redshift에 실행
2. S3 경로로 파일이 저장됨
3. S3 파일 URL을 생성해서 DB에 저장
4. 추후 사용자에게 다운로드 링크 제공 가능

---

## ✅ 전체 구조

* **API**: `/api/unload`
* **입력**: SQL 쿼리, 요청 ID
* **출력**: 요청 ID로 DB에 저장된 링크
* **구성 클래스**:

  * `UnloadRequest` DTO
  * `UnloadResult` Entity
  * `UnloadService`
  * `UnloadController`

---

## ✅ 1. DTO – `UnloadRequest`

```java
public class UnloadRequest {
    private String requestId;
    private String sql;
}
```

---

## ✅ 2. Entity – `UnloadResult`

```java
@Entity
public class UnloadResult {
    @Id
    private String requestId;

    private String s3FileUrl;

    @Enumerated(EnumType.STRING)
    private Status status;

    public enum Status {
        PENDING, DONE, FAILED
    }
}
```

---

## ✅ 3. Repository – `UnloadResultRepository`

```java
public interface UnloadResultRepository extends JpaRepository<UnloadResult, String> {
}
```

---

## ✅ 4. Service – `UnloadService`

```java
@Service
@RequiredArgsConstructor
public class UnloadService {

    private final JdbcTemplate jdbcTemplate;
    private final UnloadResultRepository repository;

    @Value("${s3.bucket}")
    private String s3Bucket;

    @Value("${s3.accessKey}")
    private String accessKey;

    @Value("${s3.secretKey}")
    private String secretKey;

    public void unloadToS3(UnloadRequest request) {
        String s3KeyPrefix = String.format("unload-results/%s/output", request.getRequestId());
        String s3Path = String.format("s3://%s/%s", s3Bucket, s3KeyPrefix);

        String unloadQuery = String.format(
            "UNLOAD ('%s') TO '%s' " +
            "CREDENTIALS 'aws_access_key_id=%s;aws_secret_access_key=%s' " +
            "DELIMITER ',' ADDQUOTES GZIP ALLOWOVERWRITE PARALLEL OFF;",
            request.getSql(), s3Path, accessKey, secretKey
        );

        UnloadResult result = new UnloadResult();
        result.setRequestId(request.getRequestId());
        result.setStatus(UnloadResult.Status.PENDING);
        repository.save(result);

        try {
            jdbcTemplate.execute(unloadQuery);

            String s3FileUrl = String.format("https://%s.s3.amazonaws.com/%s.gz", s3Bucket, s3KeyPrefix);

            result.setStatus(UnloadResult.Status.DONE);
            result.setS3FileUrl(s3FileUrl);
            repository.save(result);

        } catch (Exception e) {
            result.setStatus(UnloadResult.Status.FAILED);
            repository.save(result);
            throw new RuntimeException("UNLOAD failed", e);
        }
    }

    public Optional<UnloadResult> getResult(String requestId) {
        return repository.findById(requestId);
    }
}
```

---

## ✅ 5. Controller – `UnloadController`

```java
@RestController
@RequiredArgsConstructor
@RequestMapping("/api/unload")
public class UnloadController {

    private final UnloadService unloadService;

    @PostMapping
    public ResponseEntity<?> unload(@RequestBody UnloadRequest request) {
        unloadService.unloadToS3(request);
        return ResponseEntity.ok("UNLOAD started for: " + request.getRequestId());
    }

    @GetMapping("/{requestId}")
    public ResponseEntity<?> getResult(@PathVariable String requestId) {
        return unloadService.getResult(requestId)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
}
```

---

## ✅ 6. application.yml 예시

```yaml
s3:
  bucket: your-bucket-name
  accessKey: YOUR_AWS_ACCESS_KEY
  secretKey: YOUR_AWS_SECRET_KEY
```

---

## ✅ 예시 사용 흐름

### 🔹 POST `/api/unload`

```json
{
  "requestId": "abc123",
  "sql": "SELECT * FROM large_table"
}
```

### 🔹 GET `/api/unload/abc123`

```json
{
  "requestId": "abc123",
  "s3FileUrl": "https://your-bucket-name.s3.amazonaws.com/unload-results/abc123/output.gz",
  "status": "DONE"
}
```

---

## ✅ 확장: Kafka 연동 또는 Polling

* 현재는 `UNLOAD → JDBC 완료 → 즉시 저장` 방식입니다.
* 추후 S3 Lambda 이벤트 수신을 통한 Kafka → `status=DONE` 업데이트 로직도 추가 가능

---

## ✅ 보안 팁

* JDBC UNLOAD 시 `aws_iam_role='arn:aws:iam::xxx:role/RedshiftUnloadRole'` 방식 사용 추천 (accessKey, secretKey 대신)
* S3 버킷에 PreSigned URL을 발급해서 다운로드 제한 가능

---

필요 시 다음 단계로 이어서 드릴 수 있어요:

* Kafka 연동
* S3 PreSigned URL로 보안 설정
* Step Functions/Lambda 연계로 비동기 처리

원하시는 흐름이 더 있으면 말씀 주세요.

--------------

당신이 원하시는 기능은 다음과 같은 **Redshift → UNLOAD → S3 → 알림 수신 → 파일 링크 다운로드 제공** 흐름입니다. 각 단계를 아래처럼 구성하면 됩니다:

---

### ✅ 전체 아키텍처 요약

```
[Sprint Boot API] ─┬─> [Redshift UNLOAD Query 실행 (JDBC)]
                  │
                  └─> [UNLOAD 대상 쿼리 생성 + S3 대상 지정]
                       ↓
       Redshift → UNLOAD → S3 (1-week retention policy)
                       ↓
              [S3 ObjectCreated Lambda Trigger]
                       ↓
          [Kafka Producer → Kafka Event 발행]
                       ↓
            [Spring Boot Listener → 클라이언트 전송]
```

---

### ✅ 구성 요소별 상세 가이드

---

#### 1. **Spring Boot API에서 UNLOAD 요청 받기**

* 사용자로부터 쿼리, ID 등을 받아 API로 요청
* 요청 예시:

```json
POST /api/unload
{
  "query": "SELECT * FROM your_large_table",
  "requestId": "12345"
}
```

---

#### 2. **Redshift UNLOAD 실행 (JDBC 사용)**

```java
public void unloadToS3(String query, String s3Path, String accessKey, String secretKey) {
    String unloadQuery = String.format(
        "UNLOAD ('%s') TO '%s' " +
        "CREDENTIALS 'aws_access_key_id=%s;aws_secret_access_key=%s' " +
        "DELIMITER ',' ADDQUOTES GZIP ALLOWOVERWRITE PARALLEL OFF;",
        query, s3Path, accessKey, secretKey
    );

    jdbcTemplate.execute(unloadQuery);
}
```

* `s3Path`: 예시 `"s3://your-bucket/unload-results/12345/output"`
* `PARALLEL OFF`: 하나의 gzip 파일로 받기 위함
* **주의**: UNLOAD 명령은 Redshift에서 직접 실행되며 S3 권한이 필요

---

#### 3. **S3에 업로드 완료되면 Lambda Trigger**

* S3 bucket에 `ObjectCreated` 이벤트 설정
* 해당 prefix (`unload-results/`) 아래 파일이 생기면 Lambda 실행
* Lambda 예시 코드 (Python):

```python
import json, boto3
from kafka import KafkaProducer

producer = KafkaProducer(bootstrap_servers='your-kafka:9092')

def lambda_handler(event, context):
    for record in event['Records']:
        s3_info = record['s3']
        bucket = s3_info['bucket']['name']
        key = s3_info['object']['key']
        
        file_url = f"https://{bucket}.s3.amazonaws.com/{key}"
        request_id = key.split('/')[1]

        payload = {
            "requestId": request_id,
            "s3FileUrl": file_url
        }

        producer.send("unload-complete-topic", json.dumps(payload).encode('utf-8'))
```

---

#### 4. **Spring Boot에서 Kafka Consumer 등록**

```java
@KafkaListener(topics = "unload-complete-topic", groupId = "unload-group")
public void handleUnloadComplete(String message) {
    ObjectMapper mapper = new ObjectMapper();
    Map<String, String> data = mapper.readValue(message, new TypeReference<>() {});
    
    String requestId = data.get("requestId");
    String fileUrl = data.get("s3FileUrl");

    // DB에 저장하거나 클라이언트에 전송 등 후처리
}
```

---

#### 5. **S3 파일 링크 사용자에게 제공**

* Kafka Event 수신 후 DB나 캐시(Redis 등)에 저장
* 사용자 요청 시 해당 ID로 파일 링크 조회

---

#### 6. **S3 Retention (1주일 설정)**

* S3 콘솔 > Lifecycle Policy 설정:

```json
{
  "ID": "TemporaryFiles",
  "Filter": {
    "Prefix": "unload-results/"
  },
  "Expiration": {
    "Days": 7
  },
  "Status": "Enabled"
}
```

---

### 🔄 진행상황 Polling

* Spring에서 UNLOAD 시작 후 상태를 확인하기 위해 DB에 상태 저장 (`PENDING`, `DONE`, `FAILED`)
* Kafka 수신 시 상태 `DONE`으로 변경
* 사용자: `/api/unload/status/{requestId}`로 polling

---

### ✅ 보안 권한 설정

* S3에 UNLOAD하려면 Redshift IAM Role 또는 Access Key 필요
* 최소 권한 정책:

```json
{
  "Effect": "Allow",
  "Action": [
    "s3:PutObject",
    "s3:GetObject",
    "s3:ListBucket"
  ],
  "Resource": [
    "arn:aws:s3:::your-bucket-name/unload-results/*",
    "arn:aws:s3:::your-bucket-name"
  ]
}
```

---

### ✅ 요약 API 스펙

| API                           | 설명                |
| ----------------------------- | ----------------- |
| `POST /api/unload`            | UNLOAD 시작 요청      |
| `GET /api/unload/status/{id}` | 현재 진행 상태 조회       |
| `GET /api/unload/file/{id}`   | 완료된 S3 다운로드 링크 제공 |

---

필요하시면 아래 전체 예제들도 구성해드릴 수 있습니다:

* Lambda 전체 코드
* Kafka 설정 YAML
* Spring Boot KafkaListener 구성
* Redshift UNLOAD 쿼리 builder

원하시는 구성 방향 있으시면 알려주세요.

