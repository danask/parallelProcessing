좋은 질문이에요. `@Scheduled`로 dummy ping을 보내는 건 일정 간격으로 커넥션을 "활용"하려는 의도인데, **WebClient가 내부적으로 커넥션 풀을 어떻게 관리하느냐에 따라** 실제로 idle 상태에서 커넥션이 유지될지는 다를 수 있어요. 아래에 보장 여부와 대안을 정리해 드릴게요:

---

### 🔍 이 코드만으로 idle 방지 보장될까?

```java
@Scheduled(fixedDelay = 30_000)
public void keepConnectionAlive() {
    webClient.get()
        .uri("/ping")
        .retrieve()
        .bodyToMono(Void.class)
        .subscribe();
}
```

* 이 방식은 일정 주기로 WebClient가 API 호출을 하도록 설정했기 때문에 **`/ping` 호출이 성공하면 커넥션은 재사용(keep-alive) 상태가 됩니다.**
* 하지만 중요한 점은:

  * `WebClient`가 **커넥션 풀을 재사용 중인지 여부** (Connection: keep-alive인지, 서버가 idle timeout으로 끊었는지 등)
  * `subscribe()` 방식이 **비동기라서 실행 시 예외가 내부에서만 터지고 로그로 안 나올 수 있음**

---

### 🧠 보장이 안 되는 이유

1. **서버가 일정 시간 후 idle 커넥션을 끊을 수 있음**

   * 서버(Nginx, API Gateway, Target 서버)가 60초, 90초 idle 커넥션을 끊는 설정이면 30초 간격이어도 race condition으로 끊길 수 있어요.

2. **연결 재사용이 안 되는 경우**

   * `Connection: close` 헤더가 있거나, 서버가 keep-alive를 지원 안 하면 항상 새로운 커넥션을 만들고, ping은 의미가 없음.

3. **응답 처리 안 하고 subscribe만 하면 커넥션 풀에 반환 안 될 수도 있음**

   * `bodyToMono(Void.class).subscribe()`로 응답을 무시하면 Netty 레벨에서 커넥션이 풀에 잘 반환되지 않거나 상태가 꼬일 수 있음

---

### ✅ 보장하려면 이렇게

1. **정상적인 응답 소비 + 에러 처리**

   ```java
   @Scheduled(fixedDelay = 30_000)
   public void keepConnectionAlive() {
       webClient.get()
           .uri("/ping")
           .retrieve()
           .toBodilessEntity()
           .doOnError(e -> log.warn("Ping failed: {}", e.getMessage()))
           .subscribe(response -> log.debug("Ping status: {}", response.getStatusCode()));
   }
   ```

2. **TCP-level keep-alive도 설정하는 것이 안전**

   * Netty 기반 WebClient에서 TCP Keep-Alive 설정:

     ```java
     HttpClient httpClient = HttpClient.create()
         .tcpConfiguration(client ->
             client.option(ChannelOption.SO_KEEPALIVE, true)
         );
     ```

3. **Connection pool 설정 확인**

   ```java
   HttpClient httpClient = HttpClient.create()
       .responseTimeout(Duration.ofSeconds(5))
       .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
       .doOnConnected(conn ->
           conn.addHandlerLast(new ReadTimeoutHandler(5))
               .addHandlerLast(new WriteTimeoutHandler(5)))
       .keepAlive(true); // 실제는 TCP KeepAlive가 아니라 Connection: keep-alive 헤더 의미

   WebClient webClient = WebClient.builder()
       .clientConnector(new ReactorClientHttpConnector(httpClient))
       .baseUrl("http://your-api")
       .build();
   ```

---

### 💡추가 팁

* **Wireshark / TCPDump**로 실제 커넥션이 재사용되는지 확인 가능
* 서버가 커넥션을 끊으면, 다음 호출에서 `Connection prematurely closed` 같은 에러 발생
* LoadBalancer 또는 API Gateway(Nginx 등)가 중간에 idle timeout 설정을 가지고 있으면 **클라이언트 측 keepalive만으론 방지 불가**

---

필요하다면 idle 커넥션이 생기지 않게 서버와 클라이언트 모두에서 커넥션 유지 전략을 맞춰야 해요. Daniel의 환경(서버 종류, gateway 유무, cloud infra 등)을 알려주시면 더 맞춤으로 도와드릴게요.

---

"keep-alive"를 계속 보내는 접근은 경우에 따라 효과적일 수 있지만, 모든 상황에 적절한 해결책은 아닙니다. 아래에 이 문제를 분석하고 해결하기 위한 방향을 정리해드릴게요.

---

## 💡 상황 요약

* `/v1/query`를 호출할 때 간헐적으로 `/v1/query/v1/query`가 되는 문제 발생
* 오랫동안 접속이 없거나 연속적으로 호출할 경우 500 에러 발생
* 일정 시간이 지나면 connection 이 끊기거나 문제가 생기는 정황
* WebClient/RestTemplate 사용

---

## 🧠 가능한 원인

### 1. **Connection Pool에서 Idle Connection이 재사용되다 끊긴 경우**

* 서버나 프록시(Nginx, LB 등)가 idle connection을 일정 시간 후 끊는데,
* 클라이언트는 그 끊긴 connection을 다시 쓰려다 에러 발생

✅ 해결책:

* connection idle timeout 을 짧게 설정하거나
* connection validation (예: stale connection check)을 사용하거나
* 주기적으로 ping/keep-alive 요청 보내기

---

### 2. **Client 측의 URL 조립 버그**

* WebClient에서 base URL과 요청 URL을 잘못 결합해 `v1/query/v1/query`가 되는 경우

```java
// baseUrl: http://api.example.com/v1/query
webClient.get()
    .uri("/v1/query") // -> http://api.example.com/v1/query/v1/query
```

✅ 해결책:

* `.baseUrl("http://host")` 로 설정하고 `.uri("/v1/query")`
* 또는 `.baseUrl("http://host/v1")` + `.uri("/query")` 로 일관성 유지

---

### 3. **서버 측 Rate Limit / Thread Pool 고갈**

* 연속 호출 시 서버의 리소스(스레드, DB connection 등)가 고갈되어 500 에러
* 특히 health check 가 많으면 서버 자원을 낭비할 수 있음

✅ 해결책:

* 서버에 적절한 rate-limit, circuit breaker 적용
* health check endpoint는 최대한 lightweight하게 유지
* 500 발생 시 재시도(Retry) 설정 고려

---

## ✅ Keep-Alive 주기적 전송? (Ping Strategy)

### 🔹 효과 있을 수 있는 경우

* 연결이 자주 끊기는 경우, 커넥션 풀 내에서 keep-alive를 주기적으로 유지하면 연결 안정성 향상

### 🔹 한계 및 주의점

* 너무 자주 보내면 리소스 낭비
* LB나 프록시가 idle connection을 60초 이내로 kill 할 수 있으니 주기 맞추기
* 실제 API call이 아닌 dummy/ping call이어야 함

```java
// 예: 30초마다 dummy ping
@Scheduled(fixedDelay = 30_000)
public void keepConnectionAlive() {
    webClient.get()
        .uri("/ping")
        .retrieve()
        .bodyToMono(Void.class)
        .subscribe();
}
```

---

## 🛠 함께 쓰면 좋은 설정들

### 🔹 WebClient Timeout + Retry + Connection Pool

```java
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5_000)
        .responseTimeout(Duration.ofSeconds(5))
        .doOnConnected(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(5))
            .addHandlerLast(new WriteTimeoutHandler(5)))
        .keepAlive(true)
        .tcpConfiguration(client ->
            client.option(ChannelOption.SO_KEEPALIVE, true)
        )
        .metrics(true);

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .baseUrl("http://your-api.com")
        .build();
}
```

---

## 🔚 결론

| 문제                | 원인 가능성             | 조치                                   |
| ----------------- | ------------------ | ------------------------------------ |
| URL 중복            | URI 설정 오류          | `baseUrl`, `uri` 확인                  |
| 간헐적 500           | 끊긴 커넥션 재사용, 서버 과부하 | 커넥션 풀, keep-alive, retry, timeout 설정 |
| 연속 호출 시 500       | 서버 리소스 부족          | Rate limit / 백오프 / 서킷브레이커            |
| keep-alive 주기적 호출 | Idle 방지 가능         | 주의해서 적용 (30\~60초)                    |

---

필요하면 WebClient 설정, 서킷브레이커, 또는 상태 추적 모니터링 설정도 도와드릴게요.


----

그렇다면 문제는 단순한 코드 버그가 아니라 **동적 환경에서 URL이 중복되거나 잘못 조립되는 시점이 존재한다**는 뜻입니다. 즉, 다음 중 하나 혹은 복합적 원인이 있을 수 있습니다:

---

## ✅ 흔히 발생하는 상황별 원인 (헬스체크 URL이 `됐다 안됐다` 또는 `/v1/query/v1/query`가 가끔 나올 때)

### 1. **동적으로 URL 조립할 때 슬래시 유무에 따라 중복 경로 발생**

```java
String baseUrl = "https://api.com/v1/query"; // 슬래시 없음
String path = "/v1/query";                   // 슬래시 있음

String fullUrl = baseUrl + path; // 결과: https://api.com/v1/query/v1/query (❌)
```

✅ 해결: `UriComponentsBuilder`를 사용해서 안전하게 조립

```java
String fullUrl = UriComponentsBuilder.fromHttpUrl(baseUrl)
    .path(path)
    .build()
    .toUriString();
```

---

### 2. **Spring `WebClient`의 `baseUrl()` 설정과 `.uri()`가 충돌**

| 설정                                                           | 현상                           |
| ------------------------------------------------------------ | ---------------------------- |
| baseUrl: `https://api.com/v1/query` <br> `.uri("/v1/query")` | 최종 URL이 `/v1/query/v1/query` |
| baseUrl: `https://api.com` <br> `.uri("/v1/query")`          | 올바르게 작동                      |

> 그런데 `.uri("v1/query")` (슬래시 없음)으로 쓸 경우, baseUrl에 path가 있다면 그대로 붙습니다 → 이게 "됐다 안됐다" 하는 원인이 될 수 있음.

✅ 해결: `.uri()`에 인자를 명확하게 주거나, baseUrl을 고정된 `/` 없는 주소로 설정

---

### 3. **헬스체크 로직에서 URL 캐시나 조건 분기 버그**

* URL을 캐시에 저장하거나 동적으로 생성한 후 reuse하는 구조에서, 간혹 중복된 URL이 누적됨
* 예:

  ```java
  // 조건 따라 baseUrl이 설정됨
  String baseUrl = isInternal ? "http://internal/api" : "http://external/api/v1/query";
  ```

  이후 동일하게 `.uri("/v1/query")`를 붙여버림 → 경우에 따라 중복 발생

✅ 해결:

* 모든 URL 조합부에 로그를 심어, 실제 URL 조립 전후 값을 확인
* 조건 분기를 없애고, baseUrl + path를 일관성 있게 처리

---

### 4. **로드 밸런서(LB) 또는 프록시가 경로를 변경**

* 예: AWS ALB, Istio, Nginx 등이 `/v1/query`를 프록시 내부로 넘기면서 다시 붙이는 설정이 존재
* 또는 특정 타겟 인스턴스만 rewrite 룰이 적용되어 URL이 변형될 수도 있음

✅ 해결:

* LB, 프록시, Istio ingress 등의 설정 확인
* LB 대상 그룹에 따라 경로 변경이 발생하는지 비교

---

### 5. **WebClient 커스텀 필터나 공통 interceptor가 중복 조작**

* 모든 요청에 prefix를 자동으로 붙이는 필터가 존재하고,
* 코드에서도 수동으로 `/v1/query`를 붙이면 이중으로 생김

✅ 확인:

* `WebClient.builder().filter(...)` 내 로직 점검
* interceptor나 logging filter 등 공통 모듈에서 URL을 수정하지 않는지

---

## ✅ 체크리스트 요약

| 체크 포인트                         | 설명                                           |
| ------------------------------ | -------------------------------------------- |
| `baseUrl`과 `uri()`의 path 충돌 여부 | 둘 다 경로 포함 시 중복 가능성                           |
| 슬래시(`/`) 유무에 따른 concat 버그      | `"a/" + "/b"` → `a//b`, `"a" + "/b"` → `a/b` |
| 조건 분기 또는 캐시된 URL의 오류           | 동적으로 조립한 URL이 상태 따라 잘못됨                      |
| 프록시 / LB 설정                    | 프록시가 내부에서 path 재작성 가능                        |
| WebClient filter/interceptor   | 공통 필터가 URL을 수정하지 않는지                         |

---

## ✅ 대응 전략

1. **URL 조립 시 로그 남기기** (경로/슬래시 포함 여부 명확히)
2. `UriComponentsBuilder` 등 안전한 방식 사용
3. 공통 baseUrl은 `/` 없이 설정, `.uri()`는 항상 `/v1/query`처럼 절대 경로
4. 가능하면 baseUrl에 path 붙이지 말 것 (그럼 `.uri("v1/query")`이 의도대로 작동 안함)
5. WebClient 필터와 proxy 설정 리뷰

---

필요하시면 다음도 도와드릴 수 있어요:

* `WebClient` + `Scheduler`로 주기적 헬스 체크 구성 코드
* Istio/Nginx 등 ingress 설정 확인 예제
* 로그 필터를 통해 URL 추적하는 공통 코드

어느 쪽 코드에서 조립 중인지 (RestTemplate/WebClient 등) 알려주시면 정확히 진단해드릴 수 있습니다.

---

"keep-alive"를 계속 보내는 접근은 경우에 따라 효과적일 수 있지만, 모든 상황에 적절한 해결책은 아닙니다. 아래에 이 문제를 분석하고 해결하기 위한 방향을 정리해드릴게요.


---

💡 상황 요약

/v1/query를 호출할 때 간헐적으로 /v1/query/v1/query가 되는 문제 발생

오랫동안 접속이 없거나 연속적으로 호출할 경우 500 에러 발생

일정 시간이 지나면 connection 이 끊기거나 문제가 생기는 정황

WebClient/RestTemplate 사용



---

🧠 가능한 원인

1. Connection Pool에서 Idle Connection이 재사용되다 끊긴 경우

서버나 프록시(Nginx, LB 등)가 idle connection을 일정 시간 후 끊는데,

클라이언트는 그 끊긴 connection을 다시 쓰려다 에러 발생


✅ 해결책:

connection idle timeout 을 짧게 설정하거나

connection validation (예: stale connection check)을 사용하거나

주기적으로 ping/keep-alive 요청 보내기



---

2. Client 측의 URL 조립 버그

WebClient에서 base URL과 요청 URL을 잘못 결합해 v1/query/v1/query가 되는 경우


// baseUrl: http://api.example.com/v1/query
webClient.get()
    .uri("/v1/query") // -> http://api.example.com/v1/query/v1/query

✅ 해결책:

.baseUrl("http://host") 로 설정하고 .uri("/v1/query")

또는 .baseUrl("http://host/v1") + .uri("/query") 로 일관성 유지



---

3. 서버 측 Rate Limit / Thread Pool 고갈

연속 호출 시 서버의 리소스(스레드, DB connection 등)가 고갈되어 500 에러

특히 health check 가 많으면 서버 자원을 낭비할 수 있음


✅ 해결책:

서버에 적절한 rate-limit, circuit breaker 적용

health check endpoint는 최대한 lightweight하게 유지

500 발생 시 재시도(Retry) 설정 고려



---

✅ Keep-Alive 주기적 전송? (Ping Strategy)

🔹 효과 있을 수 있는 경우

연결이 자주 끊기는 경우, 커넥션 풀 내에서 keep-alive를 주기적으로 유지하면 연결 안정성 향상


🔹 한계 및 주의점

너무 자주 보내면 리소스 낭비

LB나 프록시가 idle connection을 60초 이내로 kill 할 수 있으니 주기 맞추기

실제 API call이 아닌 dummy/ping call이어야 함

```java
// 예: 30초마다 dummy ping
@Scheduled(fixedDelay = 30_000)
public void keepConnectionAlive() {
    webClient.get()
        .uri("/ping")
        .retrieve()
        .bodyToMono(Void.class)
        .subscribe();
}
```

---

🛠 함께 쓰면 좋은 설정들

🔹 WebClient Timeout + Retry + Connection Pool

```java
@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5_000)
        .responseTimeout(Duration.ofSeconds(5))
        .doOnConnected(conn -> conn
            .addHandlerLast(new ReadTimeoutHandler(5))
            .addHandlerLast(new WriteTimeoutHandler(5)))
        .keepAlive(true)
        .tcpConfiguration(client ->
            client.option(ChannelOption.SO_KEEPALIVE, true)
        )
        .metrics(true);

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .baseUrl("http://your-api.com")
        .build();
}
```

---

🔚 결론

문제	원인 가능성	조치

URL 중복	URI 설정 오류	baseUrl, uri 확인
간헐적 500	끊긴 커넥션 재사용, 서버 과부하	커넥션 풀, keep-alive, retry, timeout 설정
연속 호출 시 500	서버 리소스 부족	Rate limit / 백오프 / 서킷브레이커
keep-alive 주기적 호출	Idle 방지 가능	주의해서 적용 (30~60초)



---

필요하면 WebClient 설정, 서킷브레이커, 또는 상태 추적 모니터링 설정도 도와드릴게요.




그렇다면 문제는 단순한 코드 버그가 아니라 **동적 환경에서 URL이 중복되거나 잘못 조립되는 시점이 존재한다**는 뜻입니다. 즉, 다음 중 하나 혹은 복합적 원인이 있을 수 있습니다:

---

## ✅ 흔히 발생하는 상황별 원인 (헬스체크 URL이 `됐다 안됐다` 또는 `/v1/query/v1/query`가 가끔 나올 때)

### 1. **동적으로 URL 조립할 때 슬래시 유무에 따라 중복 경로 발생**

```java
String baseUrl = "https://api.com/v1/query"; // 슬래시 없음
String path = "/v1/query";                   // 슬래시 있음

String fullUrl = baseUrl + path; // 결과: https://api.com/v1/query/v1/query (❌)
```

✅ 해결: `UriComponentsBuilder`를 사용해서 안전하게 조립

```java
String fullUrl = UriComponentsBuilder.fromHttpUrl(baseUrl)
    .path(path)
    .build()
    .toUriString();
```

---

### 2. **Spring `WebClient`의 `baseUrl()` 설정과 `.uri()`가 충돌**

| 설정                                                           | 현상                           |
| ------------------------------------------------------------ | ---------------------------- |
| baseUrl: `https://api.com/v1/query` <br> `.uri("/v1/query")` | 최종 URL이 `/v1/query/v1/query` |
| baseUrl: `https://api.com` <br> `.uri("/v1/query")`          | 올바르게 작동                      |

> 그런데 `.uri("v1/query")` (슬래시 없음)으로 쓸 경우, baseUrl에 path가 있다면 그대로 붙습니다 → 이게 "됐다 안됐다" 하는 원인이 될 수 있음.

✅ 해결: `.uri()`에 인자를 명확하게 주거나, baseUrl을 고정된 `/` 없는 주소로 설정

---

### 3. **헬스체크 로직에서 URL 캐시나 조건 분기 버그**

* URL을 캐시에 저장하거나 동적으로 생성한 후 reuse하는 구조에서, 간혹 중복된 URL이 누적됨
* 예:

  ```java
  // 조건 따라 baseUrl이 설정됨
  String baseUrl = isInternal ? "http://internal/api" : "http://external/api/v1/query";
  ```

  이후 동일하게 `.uri("/v1/query")`를 붙여버림 → 경우에 따라 중복 발생

✅ 해결:

* 모든 URL 조합부에 로그를 심어, 실제 URL 조립 전후 값을 확인
* 조건 분기를 없애고, baseUrl + path를 일관성 있게 처리

---

### 4. **로드 밸런서(LB) 또는 프록시가 경로를 변경**

* 예: AWS ALB, Istio, Nginx 등이 `/v1/query`를 프록시 내부로 넘기면서 다시 붙이는 설정이 존재
* 또는 특정 타겟 인스턴스만 rewrite 룰이 적용되어 URL이 변형될 수도 있음

✅ 해결:

* LB, 프록시, Istio ingress 등의 설정 확인
* LB 대상 그룹에 따라 경로 변경이 발생하는지 비교

---

### 5. **WebClient 커스텀 필터나 공통 interceptor가 중복 조작**

* 모든 요청에 prefix를 자동으로 붙이는 필터가 존재하고,
* 코드에서도 수동으로 `/v1/query`를 붙이면 이중으로 생김

✅ 확인:

* `WebClient.builder().filter(...)` 내 로직 점검
* interceptor나 logging filter 등 공통 모듈에서 URL을 수정하지 않는지

---

## ✅ 체크리스트 요약

| 체크 포인트                         | 설명                                           |
| ------------------------------ | -------------------------------------------- |
| `baseUrl`과 `uri()`의 path 충돌 여부 | 둘 다 경로 포함 시 중복 가능성                           |
| 슬래시(`/`) 유무에 따른 concat 버그      | `"a/" + "/b"` → `a//b`, `"a" + "/b"` → `a/b` |
| 조건 분기 또는 캐시된 URL의 오류           | 동적으로 조립한 URL이 상태 따라 잘못됨                      |
| 프록시 / LB 설정                    | 프록시가 내부에서 path 재작성 가능                        |
| WebClient filter/interceptor   | 공통 필터가 URL을 수정하지 않는지                         |

---

## ✅ 대응 전략

1. **URL 조립 시 로그 남기기** (경로/슬래시 포함 여부 명확히)
2. `UriComponentsBuilder` 등 안전한 방식 사용
3. 공통 baseUrl은 `/` 없이 설정, `.uri()`는 항상 `/v1/query`처럼 절대 경로
4. 가능하면 baseUrl에 path 붙이지 말 것 (그럼 `.uri("v1/query")`이 의도대로 작동 안함)
5. WebClient 필터와 proxy 설정 리뷰

---

필요하시면 다음도 도와드릴 수 있어요:

* `WebClient` + `Scheduler`로 주기적 헬스 체크 구성 코드
* Istio/Nginx 등 ingress 설정 확인 예제
* 로그 필터를 통해 URL 추적하는 공통 코드

어느 쪽 코드에서 조립 중인지 (RestTemplate/WebClient 등) 알려주시면 정확히 진단해드릴 수 있습니다.

----
`/v1/query/v1/query`처럼 **헬스 체크용 API 경로가 중복**된다면, 그건 거의 100% **잘못된 base URL 또는 URI 결합 문제**입니다. 정확히 어떤 부분에서 문제가 생겼는지 아래처럼 분석해볼 수 있어요.

---

## ✅ 문제 상황 요약

예상했던 URL:

```
https://api.service.com/v1/query
```

실제 호출 URL:

```
https://api.service.com/v1/query/v1/query
```

---

## ✅ 원인 1: **base URL 잘못 설정**

### 예: `WebClient`에서 `baseUrl`에 path를 포함했는데, `uri()` 호출 시 중복됨

```java
WebClient client = WebClient.builder()
    .baseUrl("https://api.service.com/v1/query") // ❌ 잘못된 baseUrl
    .build();

client.get().uri("/v1/query").retrieve()  // 결과: https://api.service.com/v1/query/v1/query
```

### 🔧 해결:

baseUrl에는 **host만** 넣고 path는 `uri()`로 분리하세요.

```java
WebClient client = WebClient.builder()
    .baseUrl("https://api.service.com")  // ✅ 올바른 baseUrl
    .build();

client.get().uri("/v1/query").retrieve(); // 결과: https://api.service.com/v1/query
```

---

## ✅ 원인 2: **RestTemplate에서 URL 조합 시 슬래시 중복 또는 누락**

### 예:

```java
String baseUrl = "https://api.service.com/v1/query";  // ❌ 경로 포함된 baseUrl
String path = "/v1/query";

restTemplate.getForObject(baseUrl + path, String.class);  
// 결과: https://api.service.com/v1/query/v1/query
```

### 🔧 해결:

* baseUrl에는 path를 포함하지 말고
* 경로는 path 변수로 분리하세요

```java
String baseUrl = "https://api.service.com";
String path = "/v1/query";

restTemplate.getForObject(baseUrl + path, String.class);  
// 결과: https://api.service.com/v1/query
```

---

## ✅ 원인 3: **Spring Cloud Gateway나 proxy 설정에서 path rewriting 문제**

* `/v1/query`를 다시 내부 라우팅으로 붙여버리는 경우가 있음
* 예: `filters: - RewritePath=/v1/query,/v1/query/v1/query`

이 경우는 `application.yml`이나 gateway 설정 확인 필요.

---

## ✅ 디버깅 팁

* 로그에서 최종 요청 URL 확인 (`DEBUG` 레벨에서 WebClient 또는 RestTemplate 로그 확인)
* `WebClient`는 `.filter()`에 로깅 필터 추가해서 추적 가능
* Postman이나 cURL로 동일한 요청 보내서 실제 호출 URL 확인

---

## ✅ 정리

| 원인                                 | 해결 방법                                                 |
| ---------------------------------- | ----------------------------------------------------- |
| baseUrl에 path까지 포함됨                | baseUrl에는 호스트까지만 설정하고, path는 `uri()`에만 작성             |
| URI 결합 시 슬래시 중복 또는 누락              | `UriComponentsBuilder` 또는 `.join("/", ...)`으로 안전하게 조립 |
| Gateway나 proxy가 path를 중복으로 rewrite | Gateway 필터나 Nginx rewrite rule 확인                     |

---

필요하면 WebClient, RestTemplate, 또는 Spring Cloud Gateway에 맞는 구체적인 코드 예제도 바로 드릴게요. 어떤 방식으로 헬스 체크를 구성했는지 알려주시면 더 정확히 진단해드릴 수 있어요.


----

좋습니다. 아래에 `WebClient`와 `RestTemplate` 각각에 대해 **retry, timeout, connection pool 설정**을 모두 정리해서 예제와 함께 드릴게요. 프로덕션에 바로 적용 가능한 형태로 구성했습니다.

---

## ✅ WebClient 설정 (Spring Boot 2.x\~3.x)

### 1. **Timeout + Connection Pool + Retry 포함**

```java
import io.netty.channel.ChannelOption;
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.timeout.WriteTimeoutHandler;
import reactor.netty.http.client.HttpClient;
import reactor.util.retry.Retry;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Bean
public WebClient webClient() {
    HttpClient httpClient = HttpClient.create()
        .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
        .responseTimeout(Duration.ofSeconds(5))
        .doOnConnected(conn ->
            conn.addHandlerLast(new ReadTimeoutHandler(5, TimeUnit.SECONDS))
                .addHandlerLast(new WriteTimeoutHandler(5, TimeUnit.SECONDS))
        );

    return WebClient.builder()
        .clientConnector(new ReactorClientHttpConnector(httpClient))
        .baseUrl("https://target-api.com")
        .filter((request, next) -> next.exchange(request)
            .retryWhen(Retry.fixedDelay(3, Duration.ofMillis(500))) // 재시도 3회
        )
        .build();
}
```

> 🔸 기본적으로 WebClient는 `reactor-netty`를 사용하므로, Netty 기반으로 설정합니다.

---

## ✅ RestTemplate 설정 (Apache HttpClient 기반)

### 1. **Timeout + Connection Pool 설정**

```java
@Bean
public RestTemplate restTemplate() {
    int timeout = 5000;

    RequestConfig config = RequestConfig.custom()
        .setConnectTimeout(timeout)
        .setConnectionRequestTimeout(timeout)
        .setSocketTimeout(timeout)
        .build();

    PoolingHttpClientConnectionManager connectionManager = new PoolingHttpClientConnectionManager();
    connectionManager.setMaxTotal(100); // 전체 커넥션 수
    connectionManager.setDefaultMaxPerRoute(20); // 목적지당 최대 커넥션 수

    CloseableHttpClient httpClient = HttpClients.custom()
        .setConnectionManager(connectionManager)
        .setDefaultRequestConfig(config)
        .build();

    HttpComponentsClientHttpRequestFactory factory = new HttpComponentsClientHttpRequestFactory(httpClient);

    return new RestTemplate(factory);
}
```

---

### 2. **Retry 처리 (Spring Retry + RestTemplate)**

#### 2-1. 의존성 추가 (Maven)

```xml
<dependency>
    <groupId>org.springframework.retry</groupId>
    <artifactId>spring-retry</artifactId>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-aop</artifactId> <!-- @Retryable 작동 위해 필요 -->
</dependency>
```

#### 2-2. 사용 예제

```java
@EnableRetry  // Spring Boot main class 또는 config class에 선언
@Configuration
public class RetryConfig {
}

@Service
public class MyService {

    private final RestTemplate restTemplate;

    public MyService(RestTemplate restTemplate) {
        this.restTemplate = restTemplate;
    }

    @Retryable(
        value = { ResourceAccessException.class, HttpServerErrorException.class },
        maxAttempts = 3,
        backoff = @Backoff(delay = 1000) // 1초 딜레이
    )
    public String callExternalApi() {
        ResponseEntity<String> response = restTemplate.getForEntity("https://target-api.com/resource", String.class);
        return response.getBody();
    }

    @Recover
    public String recover(Exception e) {
        return "Fallback result due to: " + e.getMessage();
    }
}
```

---

## ✅ 참고: 타임아웃 설명

| 설정 종류                             | 설명                        |
| --------------------------------- | ------------------------- |
| `connectTimeout`                  | 서버에 연결하는 데 걸리는 최대 시간      |
| `readTimeout` / `responseTimeout` | 서버가 응답하는 데까지 기다리는 시간      |
| `connectionRequestTimeout`        | 커넥션 풀에서 커넥션을 얻기까지 기다리는 시간 |

---

## ✅ 결론

| 항목       | WebClient                                              | RestTemplate                                            |
| -------- | ------------------------------------------------------ | ------------------------------------------------------- |
| 타임아웃 설정  | `HttpClient.create().option(...).responseTimeout(...)` | `RequestConfig`                                         |
| 커넥션 풀    | Netty는 자동 또는 `ConnectionProvider`로 설정 가능               | Apache HttpClient의 `PoolingHttpClientConnectionManager` |
| Retry 지원 | `retryWhen(...)` 직접 구현                                 | `@Retryable` (Spring Retry 사용)                          |

---

필요하다면 다음도 알려드릴 수 있어요:

* Circuit Breaker (Resilience4j) 설정
* WebClient 로깅 (`logging.level.reactor.netty` 등)
* WebClient에서 타임아웃, 실패 로직 별도 fallback 처리

어떤 환경 (예: ECS, EC2, 쿠버네티스)인지 말씀해 주시면 그에 맞게 커넥션 유지 전략도 함께 설명드릴게요.


