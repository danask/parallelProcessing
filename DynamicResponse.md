좋습니다! 아래에 요청하신 전체 구성 요소들을 모두 예제로 제공합니다:

---

## ✅ 1. Step Function `.asl.json` 전체 예제

```json
{
  "Comment": "Redshift UNLOAD to S3 Step Function",
  "StartAt": "UnloadRedshiftData",
  "States": {
    "UnloadRedshiftData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:lambda-unload-redshift",
      "Next": "CheckS3Output"
    },
    "CheckS3Output": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:lambda-check-s3",
      "Next": "GeneratePresignedUrl"
    },
    "GeneratePresignedUrl": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:REGION:ACCOUNT_ID:function:lambda-generate-url",
      "End": true
    }
  }
}
```

> 🛠 각 Lambda 함수는 아래의 핸들러 예제 참고.

---

## ✅ 2. Redshift JDBC 기반 Lambda Handler 예시 (`lambda-unload-redshift`)

```java
public class UnloadHandler implements RequestHandler<Map<String, Object>, Map<String, Object>> {

    @Override
    public Map<String, Object> handleRequest(Map<String, Object> input, Context context) {
        String jdbcUrl = System.getenv("REDSHIFT_JDBC_URL");
        String user = System.getenv("REDSHIFT_USER");
        String password = System.getenv("REDSHIFT_PASSWORD");
        String s3Output = (String) input.get("s3Path");
        String sql = (String) input.get("unloadSql");

        try (Connection conn = DriverManager.getConnection(jdbcUrl, user, password);
             Statement stmt = conn.createStatement()) {

            stmt.execute(sql.replace(":s3path", s3Output));

            return Map.of("status", "success", "s3Path", s3Output);
        } catch (SQLException e) {
            throw new RuntimeException("UNLOAD failed: " + e.getMessage(), e);
        }
    }
}
```

환경 변수로:

* `REDSHIFT_JDBC_URL`
* `REDSHIFT_USER`, `REDSHIFT_PASSWORD`

---

## ✅ 3. S3 Lifecycle 정책 템플릿 (7일 후 삭제)

```json
{
  "Rules": [
    {
      "ID": "Delete export files after 7 days",
      "Prefix": "exports/",
      "Status": "Enabled",
      "Expiration": {
        "Days": 7
      }
    }
  ]
}
```

적용 방법:

* AWS S3 Console → 버킷 설정 → 수명 주기 규칙 → 추가

---

## ✅ 4. `unloadService` 전체 구현 클래스 예시 (Spring Boot)

```java
@Service
@RequiredArgsConstructor
public class UnloadService {

    private final StepFunctionsClient stepFunctionsClient;
    private final AmazonS3 amazonS3;

    private final String s3Bucket = "your-bucket";
    private final String sfnArn = "arn:aws:states:REGION:ACCOUNT_ID:stateMachine:RedshiftUnloadStepFunction";
    private final Duration presignedUrlDuration = Duration.ofMinutes(30);

    private final Map<String, String> jobStatusStore = new ConcurrentHashMap<>();
    private final Map<String, String> jobUrlStore = new ConcurrentHashMap<>();

    public String submitUnloadJob(DynamicQueryRequest request) {
        String jobId = UUID.randomUUID().toString();
        String s3Prefix = String.format("exports/%s/result_", jobId);

        String unloadSql = generateUnloadSql(request, s3Prefix);

        Map<String, Object> input = Map.of(
            "unloadSql", unloadSql,
            "s3Path", String.format("s3://%s/%s", s3Bucket, s3Prefix)
        );

        StartExecutionRequest executionRequest = StartExecutionRequest.builder()
            .stateMachineArn(sfnArn)
            .name("job-" + jobId)
            .input(new ObjectMapper().writeValueAsString(input))
            .build();

        stepFunctionsClient.startExecution(executionRequest);
        jobStatusStore.put(jobId, "processing");

        return jobId;
    }

    public Optional<String> getPresignedUrl(String jobId) {
        if (!jobStatusStore.containsKey(jobId)) return Optional.empty();

        if ("ready".equals(jobStatusStore.get(jobId))) {
            return Optional.of(jobUrlStore.get(jobId));
        }

        // Optional: Poll S3 to detect readiness here
        return Optional.empty();
    }

    public void markJobAsReady(String jobId, String s3Key) {
        String url = generatePresignedUrl(s3Key);
        jobStatusStore.put(jobId, "ready");
        jobUrlStore.put(jobId, url);
    }

    private String generateUnloadSql(DynamicQueryRequest request, String s3Prefix) {
        String baseQuery = request.toSql(); // 예시: SELECT ... FROM ... WHERE ...
        return String.format("""
            UNLOAD ('%s')
            TO 's3://%s/%s'
            IAM_ROLE 'arn:aws:iam::ACCOUNT_ID:role/RedshiftUnloadRole'
            FORMAT AS PARQUET
            ALLOWOVERWRITE
            PARALLEL ON
        """, baseQuery, s3Bucket, s3Prefix);
    }

    private String generatePresignedUrl(String key) {
        Date expiration = Date.from(Instant.now().plus(presignedUrlDuration));
        return amazonS3.generatePresignedUrl(
            new GeneratePresignedUrlRequest(s3Bucket, key).withExpiration(expiration)
        ).toString();
    }
}
```

---

## ✳️ 후속 개선 가능 포인트

| 항목          | 설명                                         |
| ----------- | ------------------------------------------ |
| Redis 연동    | `jobStatusStore` → Redis 저장소로 교체 가능        |
| Webhook 처리  | Step Function 마지막 Lambda에서 Spring Boot로 콜백 |
| CSV/JSON 옵션 | UNLOAD 포맷 다르게 설정 가능                        |
| jobId 보안    | JWT 토큰 또는 UserId와 연결해서 인증 강화               |

---

필요하다면 다음도 도와드릴 수 있어요:

* 각 Lambda 함수 코드를 AWS SAM/CDK로 배포하는 방법
* Spring Boot + Redis 기반 `JobStatus` 관리 구조
* 전체 YAML 구성 (Lambda IAM, S3 정책 포함)

원하시면 말씀해주세요!


----

좋습니다! Redshift 기반 DWH에서 대용량 쿼리 결과를 프론트에 제공하기 위한 **Best Practice**를 설계 중심으로 **실제 예제 코드와 함께 정리**해드릴게요.
기술 스택: **Spring Boot + AWS Redshift + S3 + Step Functions**

---

## ✅ Best Practice: Redshift UNLOAD + S3 + Spring Boot 응답 구조

---

### 📌 목표 요약

| 항목      | 내용                                            |
| ------- | --------------------------------------------- |
| 사용 시나리오 | 쿼리 결과가 `table` 타입이고 수백 MB 이상                  |
| 처리 방식   | Redshift `UNLOAD` → S3 저장 → Pre-signed URL 생성 |
| 응답 구조   | 프론트엔드는 `jobId`로 요청 후 다운로드 URL 확인              |
| 아키텍처 핵심 | 비동기 Step Function으로 안정적인 작업 수행                |

---

## ✅ 아키텍처 흐름

```
[User Request] → Spring Boot
        ↓
[Generate SQL + Submit Step Function]
        ↓
[AWS Step Function]
    ├─ [Lambda: UNLOAD SQL 실행]
    ├─ [Lambda: S3 파일 확인]
    └─ [Lambda: Pre-signed URL 생성]
        ↓
[Result in S3] ← [프론트는 jobId로 Polling]
```

---

## ✅ 구성 요소별 Best Practice

---

### ✅ 1. UNLOAD SQL 생성 예제

```sql
UNLOAD ('SELECT d.device_id, a.app_name, a.usage_time FROM device d JOIN app_usage a ON d.id = a.device_id WHERE a.usage_time > 100')
TO 's3://your-bucket/exports/${jobId}/result_'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftS3AccessRole'
FORMAT AS PARQUET
ALLOWOVERWRITE
PARALLEL ON;
```

🔹 `FORMAT AS PARQUET` → 효율적 저장
🔹 `jobId` → S3 prefix로 사용하여 충돌 방지
🔹 `PARALLEL ON` → 분산 쓰기 성능 향상

---

### ✅ 2. Step Function 구성 예시

```json
{
  "Comment": "Redshift UNLOAD to S3",
  "StartAt": "UnloadData",
  "States": {
    "UnloadData": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:unloadHandler",
      "Next": "CheckS3"
    },
    "CheckS3": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:checkS3Handler",
      "Next": "GenerateUrl"
    },
    "GenerateUrl": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:...:generatePresignedUrl",
      "End": true
    }
  }
}
```

---

### ✅ 3. Spring Boot API 예제

#### ⬛️ 3.1. UNLOAD 요청 API

```java
@PostMapping("/api/query/table")
public ResponseEntity<?> requestUnload(@RequestBody DynamicQueryRequest request) {
    String jobId = unloadService.submitUnloadJob(request);
    return ResponseEntity.accepted().body(Map.of("jobId", jobId));
}
```

#### ⬛️ 3.2. 결과 조회 API

```java
@GetMapping("/api/query/table/result/{jobId}")
public ResponseEntity<?> getDownloadUrl(@PathVariable String jobId) {
    Optional<String> presignedUrl = unloadService.getPresignedUrl(jobId);
    return presignedUrl
            .map(url -> ResponseEntity.ok(Map.of("status", "ready", "downloadUrl", url)))
            .orElse(ResponseEntity.status(HttpStatus.ACCEPTED).body(Map.of("status", "processing")));
}
```

---

### ✅ 4. S3 Pre-signed URL 생성 예시

```java
public String generatePresignedUrl(String bucket, String key, Duration duration) {
    Date expiration = Date.from(Instant.now().plus(duration));
    GeneratePresignedUrlRequest generatePresignedUrlRequest = new GeneratePresignedUrlRequest(bucket, key)
            .withMethod(HttpMethod.GET)
            .withExpiration(expiration);
    return amazonS3.generatePresignedUrl(generatePresignedUrlRequest).toString();
}
```

---

### ✅ 5. 상태 관리 전략

| 상태                 | 처리 방법                        |
| ------------------ | ---------------------------- |
| job 생성됨            | Redis 또는 DB에 `jobId` + 상태 저장 |
| Step Function 완료 시 | 상태를 `ready`로 업데이트, URL 저장    |
| 실패 시               | `error` 상태 + 로그 저장           |

---

## ✅ 추가 팁

| 항목            | 추천                                                    |
| ------------- | ----------------------------------------------------- |
| 파일 포맷         | `PARQUET` 권장 (CSV보다 압축률 및 처리 효율 ↑)                    |
| S3 구조         | `s3://bucket/exports/{userId}/{jobId}/result.parquet` |
| 유효시간          | Pre-signed URL은 15분\~1시간 설정                           |
| 파일 다운로드 권장 방식 | 브라우저 다운로드 또는 직접 저장 (파일 사이즈 큼)                         |
| 리소스 정리        | 주기적으로 S3 export 디렉터리 삭제 (LifeCycle 설정)                |

---

## ✅ 결론

| 항목     | 요약                                     |
| ------ | -------------------------------------- |
| 성능     | Redshift `UNLOAD`가 수백만 건 이상 처리에 적합     |
| 확장성    | Step Function + S3 구조는 병렬, 확장, 자동화에 유리 |
| 사용자 경험 | Pre-signed URL은 보안적이고 직접 다운로드에 적합      |
| 관리성    | `jobId`로 상태 관리하면 실패/재시도 등 확장 가능        |

---

필요하시면 아래 항목도 제공해드릴 수 있어요:

* Step Function `.asl.json` 전체 파일
* Redshift JDBC 기반 Lambda Handler 예시
* S3 LifeCycle 정책 템플릿
* `unloadService` 전체 구현 클래스

원하시는 부분 알려주세요!


-----

Redshift DWH의 대용량 쿼리 결과를 처리하는 데 있어 **`UNLOAD` → S3 저장 → 다운로드 링크 생성**은 성능/비용/확장성 측면에서 매우 적절한 접근입니다.
Spring Boot와 AWS Step Functions를 사용하는 현재 구조를 바탕으로 **모듈화된 설계안 + 대안**을 다음과 같이 정리할 수 있습니다.

---

## ✅ 1. 요구사항 정리

| 항목        | 내용                                 |
| --------- | ---------------------------------- |
| 쿼리 유형     | 조인 그래프 기반 동적 쿼리 (Spring Boot에서 생성) |
| 데이터량      | 수백만 건 이상일 수 있음 (JSON 응답 비효율적)      |
| 결과 저장     | Redshift `UNLOAD`로 S3에 저장          |
| 사용자 제공 방식 | 다운로드 가능한 pre-signed URL            |
| 실행 방식     | 비동기, 안정적 처리를 위해 Step Function 사용 중 |

---

## ✅ 2. 추천 설계 흐름

### 🌐 A. 프론트 요청 흐름

1. 사용자가 `table` 타입으로 요청
2. 백엔드가 쿼리 구조 파싱
3. 쿼리를 Redshift UNLOAD로 변환
4. Step Function 실행 → Redshift `UNLOAD` 실행 + S3에 파일 생성
5. 성공 시 pre-signed URL 반환
6. 프론트에서 다운로드 링크 제공

---

### ⚙️ B. 구성 요소

#### 1) `Spring Boot`에서 처리할 로직

* UNLOAD SQL 생성 (`DynamicQueryRequest` → SQL 변환)
* Step Function 실행 (AWS SDK 통해)
* 작업 상태 Polling (또는 SNS/Lambda 콜백)
* 성공 시 S3 pre-signed URL 반환

#### 2) `Step Function` 구성

| Step     | 역할                                        |
| -------- | ----------------------------------------- |
| Lambda-1 | UNLOAD SQL 실행 (Redshift Data API 또는 JDBC) |
| Lambda-2 | UNLOAD 결과 확인 (S3 존재 여부 확인)                |
| Lambda-3 | 성공 시 S3 Pre-signed URL 생성 (또는 Return URL) |

*선택적으로 SQS, SNS, EventBridge 등을 활용 가능*

---

## 🧠 예시: UNLOAD 쿼리

```sql
UNLOAD ('SELECT d.device_id, a.app_name FROM device d JOIN app_usage a ON d.id = a.device_id WHERE a.usage_time > 100')
TO 's3://your-bucket/export/device_app_usage_'
IAM_ROLE 'arn:aws:iam::123456789012:role/RedshiftS3AccessRole'
FORMAT AS PARQUET;
```

* `TO`는 S3의 prefix (Redshift는 여러 파일 생성함)
* 형식은 `PARQUET`, `CSV`, `JSON` 선택 가능
* 분산 환경에서 병렬 처리되어 빠름

---

## ✅ 3. 결과 전달 방식

* Redshift UNLOAD 후 `S3`에 저장된 파일을 다음 방식으로 제공:

  * **Pre-signed URL** 생성 (`AmazonS3.generatePresignedUrl`)
  * URL 유효 기간 제한 (예: 15분, 1시간)
  * 프론트에 JSON으로 응답:

```json
{
  "status": "ready",
  "downloadUrl": "https://your-bucket.s3.amazonaws.com/export/abc.parquet?..."
}
```

---

## ✅ 4. 대안 설계 (비교)

| 방식                            | 설명             | 장점            | 단점           |
| ----------------------------- | -------------- | ------------- | ------------ |
| ✅ UNLOAD + S3 + Step Function | 현재 구조          | 확장성, 대량 처리 최적 | 복잡도 있음       |
| Redshift Data API 직접 호출       | 단일 쿼리 응답       | 간단            | 대량 데이터 응답 제한 |
| Redshift Spectrum 활용          | S3 → 외부 테이블 등록 | 유연성 ↑         | 초기 구성 복잡     |
| Redshift → SQS → Lambda       | 비동기 처리 단순화     | 모듈화 쉬움        | 관리 필요        |

---

## ✅ 추천 Best Practice 정리

* UNLOAD는 \*\*`PARQUET`\*\*로 저장 (CSV보다 효율적)
* S3는 쿼리 기준 `prefix` 구분 (예: `s3://bucket/exports/{userId}/{jobId}/`)
* Spring Boot는 `JobId` 기반 Polling 제공 or `WebSocket/SNS` 알림
* `UNLOAD` 실패 시 로그 저장 (S3, CloudWatch)
* 다운로드 만료 URL은 Redis 캐시 등에 저장해 관리 가능

---

## ✳️ 보너스: Spring Boot 비동기 응답 설계

```java
@PostMapping("/api/query/table")
public ResponseEntity<?> queryToS3(@RequestBody DynamicQueryRequest request) {
    String jobId = unloadService.submitUnloadJob(request);
    return ResponseEntity.accepted().body(Map.of("jobId", jobId));
}

@GetMapping("/api/query/table/result/{jobId}")
public ResponseEntity<?> getDownloadUrl(@PathVariable String jobId) {
    Optional<String> url = unloadService.getDownloadUrl(jobId);
    return url.map(u -> ResponseEntity.ok(Map.of("url", u)))
              .orElse(ResponseEntity.status(HttpStatus.ACCEPTED).build());
}
```

---

원하시면 다음 항목도 구체적으로 제공해 드릴 수 있어요:

* UNLOAD SQL Generator (`DynamicQueryRequest` → SQL)
* Step Function 정의 예시 (`.asl.json`)
* S3 pre-signed URL 생성 코드
* 비동기 Job 관리 구조 (`JobId`, 상태 관리용 Redis or DB)

필요한 부분 알려주세요.


----

조인 그래프 기반의 **동적 쿼리**를 생성하고 그 결과를 **프론트엔드에 응답**하는 과정은 다음과 같은 단계로 나눌 수 있습니다. 이 흐름은 Spring Boot 기반 시스템과 JPA(QueryDSL 등)를 사용하는 구조를 기준으로 설명할게요:

---

### ✅ 1. 조인 그래프 기반의 동적 쿼리 생성

1. **프론트 요청 예시**:

   ```json
   {
     "group": "device",
     "fields": ["deviceId", "appUsage.appName", "appUsage.usageTime"],
     "filters": {
       "appUsage.usageTime": { "gt": 100 }
     }
   }
   ```

2. **백엔드 처리 흐름**:

   * `group`에 따라 메인 테이블 선택 (예: `device`)
   * `fields`와 `filters`를 기반으로 조인 그래프 탐색
   * 필요한 조인을 자동 구성 (`JoinGraphUtil` 또는 `metadata`)
   * QueryDSL로 동적 쿼리 생성

   예시 QueryDSL 코드:

   ```java
   QDevice device = QDevice.device;
   QAppUsage appUsage = QAppUsage.appUsage;

   JPQLQuery<?> query = queryFactory.select(Projections.fields(
       Map.class,
       device.deviceId.as("deviceId"),
       appUsage.appName.as("appUsage_appName"),
       appUsage.usageTime.as("appUsage_usageTime")
   ))
   .from(device)
   .leftJoin(appUsage).on(appUsage.deviceId.eq(device.id))
   .where(appUsage.usageTime.gt(100));
   ```

---

### ✅ 2. 결과 변환: `List<Map<String, Object>>`

QueryDSL 결과를 `Map<String, Object>` 구조로 만들면, 프론트에서 처리하기 쉬운 JSON으로 자동 직렬화됩니다.

* **Service 메소드**:

  ```java
  public List<Map<String, Object>> getDynamicResult(DynamicQueryRequest request) {
      // JoinGraphUtil을 통한 조인 설정 및 QueryDSL 동적 쿼리 생성
      JPQLQuery<?> query = buildDynamicQuery(request);
      return query.fetch();
  }
  ```

---

### ✅ 3. Controller에서 응답

```java
@GetMapping("/api/query")
public ResponseEntity<?> query(@RequestBody DynamicQueryRequest request) {
    List<Map<String, Object>> result = queryService.getDynamicResult(request);
    return ResponseEntity.ok(result);
}
```

프론트에는 다음과 같은 JSON 응답이 내려감:

```json
[
  {
    "deviceId": "abc123",
    "appUsage_appName": "YouTube",
    "appUsage_usageTime": 300
  },
  {
    "deviceId": "def456",
    "appUsage_appName": "KakaoTalk",
    "appUsage_usageTime": 200
  }
]
```

---

### ✅ 보완할 수 있는 점

* `Map<String, Object>` → DTO 변환 (optional)
* 응답 필드명 포맷을 `appUsage.appName` 스타일로 변경
* 캐싱 활용 (Spring Cache 등)
* 쿼리 로깅 및 성능 모니터링

---

### 📦 핵심 유틸 구조 예시

```java
public List<Map<String, Object>> getCategoryAndFields(String group) {
    // metadata를 기반으로 각 그룹에 대한 필드 목록 반환
}

public List<Map<String, Object>> getDynamicResult(DynamicQueryRequest request) {
    // join 그래프 기반 쿼리 생성 및 결과 반환
}
```

---

필요하시면 위 코드들을 실제 클래스 구조로 만들어 드릴 수 있어요. 예: `JoinGraphUtil`, `DynamicQueryRequest`, `QueryService`, `MetadataProperties` 등.


-------
**조인 그래프 기반의 동적 쿼리를 생성하고**, 그 결과를 **프론트에 응답할 때** 문제는 바로 이거지:

> ❗ **결과의 구조가 고정되어 있지 않다 (Dynamic Schema)**

이럴 땐 응답 구조를 유연하게 가져가야 해. 아래 단계별로 정리해줄게.

---

## ✅ 1. 응답 방식의 2가지 전략

### ✅ A. **Flat Map 리스트로 응답 (`List<Map<String, Object>>`)**

```java
public List<Map<String, Object>> getDynamicResult(...);
```

**예시 응답 JSON:**

```json
[
  {
    "app_id": "com.kakao.talk",
    "device_model": "SM-A505",
    "foreground_time": 120
  },
  {
    "app_id": "com.youtube.app",
    "device_model": "SM-G960",
    "foreground_time": 300
  }
]
```

🔧 **장점:**

* 구조가 유동적이어서 어떤 필드 조합이 와도 문제 없음
* Mongo나 SQL 양쪽에서 쉽게 대응 가능

🧩 **단점:**

* 타입 정보 없음 (프론트에서 어떤 게 `int`, `string`인지 모르므로 별도 메타 필요)

---

### ✅ B. **별도 `meta + data` 구조로 응답**

```java
public class DynamicResponse {
    private List<ColumnMeta> meta;
    private List<Map<String, Object>> data;
}
```

```java
public class ColumnMeta {
    private String field;      // e.g. "app_id"
    private String label;      // e.g. "App ID"
    private String type;       // e.g. "string", "int", "float"
    private String group;      // measure/dimension/filter
    private String category;   // app_usage, device 등
}
```

**예시 응답 JSON:**

```json
{
  "meta": [
    { "field": "app_id", "label": "App ID", "type": "string", "group": "dimension", "category": "package" },
    { "field": "device_model", "label": "Device Model", "type": "string", "group": "dimension", "category": "device" }
  ],
  "data": [
    { "app_id": "com.kakao.talk", "device_model": "SM-A505" },
    { "app_id": "com.youtube.app", "device_model": "SM-G960" }
  ]
}
```

🔧 **장점:**

* **프론트에서 컬럼 이름, 타입, 분류를 보여주기 용이**
* 시각화용 그래프, 테이블 등에 적합

🧩 **단점:**

* 응답 크기 약간 증가

---

## ✅ 2. SQL / Mongo 대응 방식

* SQL: `JdbcTemplate.queryForList(...)` → 자동으로 `List<Map<String, Object>>` 로 변환됨
* Mongo: `MongoTemplate.aggregate(...)` → `List<Document>` → `List<Map<String, Object>>` 로 쉽게 매핑 가능

---

## ✅ 3. 확장 구조 예시

```java
public class QueryResult {
    private List<ColumnMeta> meta;                 // 각 필드 정보
    private List<Map<String, Object>> data;        // 결과 데이터
    private List<JoinFieldInfo> usedJoins;         // 사용된 조인 정보 (선택)
    private List<FilterInfo> appliedFilters;       // 필터 조건 (선택)
}
```

이 구조면 이후 아래도 다 대응 가능해:

* 컬럼 리네이밍
* 조인 트래킹
* 사용자가 선택한 필드 하이라이팅 등

---

## ✅ 결론

| 상황                    | 추천 응답 방식                         |
| --------------------- | -------------------------------- |
| 프론트가 단순 테이블만 보여줄 경우   | `List<Map<String, Object>>`      |
| 프론트에서 시각화/분류/메타데이터 필요 | `meta + data` 구조 (`QueryResult`) |
| 향후 기능 확장 고려 시         | `QueryResult` 형태로 설계 시작 추천       |

---

원한다면:

* `QueryResult`, `ColumnMeta` 클래스 완성 코드
* SQL 또는 Mongo 결과 → `QueryResult` 로 매핑하는 함수
  바로 만들어줄 수 있어.
  어느 방향으로 갈지 알려줘!
