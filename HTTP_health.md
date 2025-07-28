
Spring Boot에서 위 `WebClient` 설정을 한 번만 설정하고 재사용하려면 \*\*Bean으로 등록해서 DI(의존성 주입)\*\*하면 됩니다. 아래와 같이 설정 클래스를 만들면 됩니다.

---

### ✅ `@Configuration` 클래스에 `@Bean` 등록

```java
import io.netty.channel.ChannelOption;
import io.netty.handler.timeout.ReadTimeoutHandler;
import io.netty.handler.timeout.WriteTimeoutHandler;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.http.client.reactive.ReactorClientHttpConnector;
import org.springframework.web.reactive.function.client.WebClient;
import reactor.netty.http.client.HttpClient;

import java.time.Duration;
import java.util.concurrent.TimeUnit;

@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        HttpClient httpClient = HttpClient.create()
                .responseTimeout(Duration.ofSeconds(10))
                .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
                .doOnConnected(conn -> conn
                        .addHandlerLast(new ReadTimeoutHandler(10, TimeUnit.SECONDS))
                        .addHandlerLast(new WriteTimeoutHandler(10, TimeUnit.SECONDS)));

        return WebClient.builder()
                .clientConnector(new ReactorClientHttpConnector(httpClient))
                .build();
    }
}
```

---

### ✅ 사용하는 곳에서는 주입만 하면 됨

```java
@Service
public class ApiService {

    private final WebClient webClient;

    public ApiService(WebClient webClient) {
        this.webClient = webClient;
    }

    public void callApi() {
        webClient.get()
                 .uri("https://example.com/api")
                 .retrieve()
                 .bodyToMono(String.class)
                 .subscribe(System.out::println);
    }
}
```

---

이렇게 하면 **WebClient 인스턴스 하나만 생성해서** 재사용하므로 효율적이고, `WebClient.builder()`를 반복적으로 호출할 필요도 없습니다.

필요하면 `@Qualifier("yourWebClient")`로 여러 개 구성도 가능합니다.


---

좋은 질문이에요.
`WebClient`는 **매번 요청마다 새로 만드는 것보다, 재사용하는 방식이 훨씬 효율적**입니다.

당신이 올린 코드:

```java
HttpClient httpClient = HttpClient.create()
    .responseTimeout(Duration.ofSeconds(10))
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
    .doOnConnected(conn -> conn
        .addHandlerLast(new ReadTimeoutHandler(10))
        .addHandlerLast(new WriteTimeoutHandler(10)));

WebClient webClient = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

이건 보통 **초기화 시점에 한 번만 구성해놓고, 이후에 계속 재사용**합니다.

---

### ✅ 추천 구조

보통 Spring Boot 환경에서는 다음처럼 `@Bean`으로 등록해서 **싱글톤으로 사용**합니다:

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient webClient() {
        HttpClient httpClient = HttpClient.create()
            .responseTimeout(Duration.ofSeconds(10))
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10))
                .addHandlerLast(new WriteTimeoutHandler(10)));

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

그리고 주입해서 사용:

```java
@Service
public class SomeService {
    private final WebClient webClient;

    public SomeService(WebClient webClient) {
        this.webClient = webClient;
    }

    public void callSomething() {
        webClient.get()
            .uri("https://example.com")
            .retrieve()
            .bodyToMono(String.class)
            .subscribe();
    }
}
```

---

### ❌ 매번 생성하면 안 좋은 이유

* 커넥션 풀을 재사용 못 함 (성능 저하)
* GC 부하 증가
* 타임아웃 등 설정 누락 위험

---

### ✅ 요약

* 위 코드는 **초기 한 번만 설정하고 재사용하는 것이 맞음**.
* Spring 환경이면 `@Bean` 으로 등록해서 DI로 주입받아 사용하세요.

필요하면 `baseUrl`이나 `defaultHeader` 등도 같이 세팅해줄 수 있어요. 원하시면 템플릿 코드도 만들어 드릴게요.


---

좋은 질문입니다. 결론부터 말씀드리면:

### ❗아니요, **매번 요청할 때마다 `WebClient`를 새로 생성하면 안 됩니다.**

---

### ✅ 올바른 방식

`WebClient`는 재사용 가능한 객체이므로 **한 번만 생성**해서 재사용하는 것이 **권장되는 방식**입니다. 위의 코드처럼 커넥션 타임아웃, 응답 타임아웃 등을 설정한 `WebClient`는 보통 다음과 같이 설정합니다:

```java
@Configuration
public class WebClientConfig {

    @Bean
    public WebClient customWebClient() {
        HttpClient httpClient = HttpClient.create()
            .responseTimeout(Duration.ofSeconds(10))
            .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
            .doOnConnected(conn -> conn
                .addHandlerLast(new ReadTimeoutHandler(10))
                .addHandlerLast(new WriteTimeoutHandler(10)));

        return WebClient.builder()
            .clientConnector(new ReactorClientHttpConnector(httpClient))
            .build();
    }
}
```

그리고 나서 필요한 곳에서 DI 받아서 사용:

```java
@Service
public class ApiService {

    private final WebClient webClient;

    public ApiService(WebClient customWebClient) {
        this.webClient = customWebClient;
    }

    public Mono<String> callServer() {
        return webClient.get()
            .uri("https://example.com/api/test")
            .retrieve()
            .bodyToMono(String.class);
    }
}
```

---

### 🔍 왜 매번 만들면 안 되는가?

1. **리소스 낭비**: `WebClient`는 내부적으로 커넥션 풀(`ConnectionPool`)을 사용합니다. 매번 만들면 커넥션 풀도 계속 새로 생성되어 성능 저하.
2. **버그 가능성**: 타임아웃 설정을 매번 다르게 쓰면 예측하기 어려운 문제 발생.
3. **불필요한 GC 부담**: 불필요하게 많은 인스턴스가 생기고, GC 성능 저하 유발 가능.

---

### 🤔 참고: 왜 ping은 되고 api는 500일까?

* `ping`은 단순한 요청이기 때문에 토큰 없이도 또는 인증 없이도 동작할 수 있지만,
* **실제 API**는 `Authorization`, `Content-Type`, `Body`, `Headers` 등이 더 많아지고, 서버에서 이를 처리하는 중 **JWT 만료**, **세션 관리**, **프록시 레이어 문제**, **서버 쪽 상태 이상** 등으로 500이 나올 수 있습니다.

이 문제는 `WebClient` 설정 외에도 서버 상태, 인증 흐름 등을 함께 봐야 하는 이슈입니다.

---

필요하시면 WebClient에 로깅 필터도 같이 붙여드릴게요.


---
좋아요, 정리해서 문제 상황과 가능성, 해결 방안 중심으로 아래와 같이 안내드릴게요.

---

## ✅ 현재 상황 요약

* **Spring Client → 다른 Spring 서버 API 호출**
* 서버 재시작 직후엔 정상 동작 → 일정 시간 후 **간헐적으로 500 오류 발생**
* `/ping` 엔드포인트로 **30초마다 keep-alive 호출 시 401 OK로 응답** 받음 (즉, 연결 자체는 살아 있음)
* 그런데 **실제 API 호출 시 500 오류 발생**

---

## ✅ 원인 가능성 정리 (Client/Server 분리)

### 🌐 1. 서버 측 문제 (API 서버)

| 가능성                     | 설명                                                                | 확인 방법                                                                                           |
| ----------------------- | ----------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| A. 인증 토큰 상태 이상          | `/ping`은 인증 불필요하거나 Refresh 없이 작동, 반면 실 API는 인증 필요 → 만료되었거나 비정상 처리 | `/api` 호출 시 Authorization 헤더 확인, 서버 로그에서 인증 관련 에러 (`Invalid token`, `Missing credentials` 등) 확인 |
| B. 세션/캐시/ThreadLocal 누수 | 서버가 오래 동작하다 보면 특정 상태나 캐시가 오염되거나 세션 timeout 발생                     | 서버 로그에서 `NullPointerException`, `IllegalStateException` 또는 `cache`, `session` 키워드 검색            |
| C. 서버 내부 상태 불안정         | 연결 풀 고갈 (DB 커넥션, HTTP Pool 등)                                     | DB 커넥션 풀, thread pool, queue 상태 모니터링 필요 (Actuator / Metrics / 로그)                               |
| D. 특정 요청 패턴에서 예외        | 요청 바디, 파라미터, 헤더가 일부 요청에서 달라지는 경우                                  | 클라이언트 로그와 서버 로그를 `traceId` 로 묶어서 비교 분석 필요                                                       |

---

### ☁️ 2. 클라이언트(Spring Client) 측 문제

| 가능성                        | 설명                                                | 확인 방법                                                             |
| -------------------------- | ------------------------------------------------- | ----------------------------------------------------------------- |
| A. 인증 헤더 누락                | `/ping`은 헤더 없이도 되는데, 실제 API는 인증 필요 → 토큰 갱신/설정이 누락 | 요청 로그 찍어서 헤더 내용 확인                                                |
| B. HTTP Connection Pool 고갈 | WebClient가 커넥션을 재사용하지 못하고 끊긴 연결을 계속 시도 → 500 가능   | `Connection reset`, `Premature close`, `ReactorNetty` 관련 에러 로그 확인 |
| C. WebClient 상태 오류         | 오래된 인스턴스를 재사용하거나 Reactor context 누락 시 오류 발생       | WebClient 재구성 혹은 `.clone()` → 요청 마다 fresh 상태로 만들기                 |
| D. 잘못된 요청 (URI, 바디 등)      | uri 조립 문제는 해결했다고 했지만, 바디 또는 파라미터 구조는 여전히 변할 수 있음  | 실제 500 날리는 요청 전체 로그 기록 (Body 포함해서)                                |

---

## 🔧 해결책 정리

### 🔹 클라이언트 쪽에서 가능한 조치

1. **요청 로그 전량 기록 (debug level)**

   * URI, 헤더, 바디, 응답 코드까지 **30초 주기로 전량 출력**
   * 요청 실패 구간에서 어떤 차이가 있는지 명확히 비교

2. **WebClient 커넥션 풀 설정 명확히 하기**

   ```java
   HttpClient httpClient = HttpClient.create()
       .responseTimeout(Duration.ofSeconds(10))
       .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
       .doOnConnected(conn -> conn
           .addHandlerLast(new ReadTimeoutHandler(10))
           .addHandlerLast(new WriteTimeoutHandler(10)));

   WebClient webClient = WebClient.builder()
       .clientConnector(new ReactorClientHttpConnector(httpClient))
       .build();
   ```

3. **WebClient를 요청마다 새로 clone 해서 사용**

   ```java
   webClient.mutate().build().get().uri(...).retrieve()...
   ```

4. **Ping이 아닌 실제 API 샘플로 keep-alive**
   → 실제 사용하는 엔드포인트 (e.g. `/api/status`)로 dummy call
   → 이걸로도 500이 나면 서버 측 문제 확실

---

### 🔹 서버 측에서 가능한 조치

1. **API 응답에 대한 `stacktrace` 로그 출력**

   * Spring Boot 3.x 기준 `error.includeStacktrace=always` 설정하고 actuator 통해 확인

2. **요청 `traceId` or `requestId` 로 전량 추적**

   * 클라이언트 → 서버 → 로그까지 동일한 ID로 추적하여 context 유지

3. **Actuator + Prometheus 설정 후 힙, 커넥션 풀, GC 상태 모니터링**
   → 특정 시간대의 pool 고갈 여부 확인

---

## ✅ 핵심 진단 포인트

* `/ping`은 통과하지만 `실제 API 호출`은 500 → **인증/세션/상태 문제일 가능성 높음**
* 일정 시간 후 발생 → **풀 고갈 / context 오염 / 세션 만료 / 인증 갱신 실패** 유형이 의심됨
* 서버 로그에 500 에러 원인이 반드시 남으므로 **서버 측 분석도 반드시 필요**
* **클라이언트에서 해결만으로 한계** 있음 → 서버와 함께 병렬 분석 필요

---

필요하면 `WebClient 설정`, `Actuator metrics`, `요청 log 템플릿`, `TraceID 삽입 방법`도 정리해 드릴게요.

-----

좋습니다, 상황을 정리해 보면:

---

### ✅ 현재 상황 요약

* **Spring WebClient 사용 중**
* `/ping`을 30초 간격으로 호출하여 세션이 **401 Unauthorized**로 유지되는 것을 확인함 → 세션이 **유효**함.
* 그런데 정작 **진짜 API 요청**을 보낼 때는 **간헐적으로 500 Internal Server Error** 발생.
* Spring 서버를 재시작하면 한동안 정상 작동하다가 시간이 지나면 500이 다시 발생.

---

## 🔍 가능한 원인 정리

### 1. **서버 세션/토큰 문제는 아님**

* 이미 ping 으로 **세션이 만료되지 않았음이 확인**됐음 → `401` 안 나오고 유지됨.

### 2. **서버 쪽 리소스 누수 / 누적 상태 문제 (가장 유력)**

* 일정 시간 이후에만 발생 → 메모리 누수, connection pool 고갈, 버퍼 overflow 가능성 있음.
* 특히 서버가 로드를 쌓아가다가 일정 시점 이후부터 500 응답 시작되는 현상은 **GC 미비, thread 누락, database connection 누락** 등을 강하게 의심함.

### 3. **HTTP Keep-Alive or Connection Pool 문제**

* 클라이언트(WebClient)나 서버 측에서 **keep-alive 연결이 재사용**되는 중간에 문제가 생김 → 재사용된 connection 이 **유효하지 않음**에도 불구하고 계속 쓰려다가 오류 발생.
* 재시작하면 커넥션이 초기화되어 잠시 정상 동작함.

### 4. **Reverse Proxy (Nginx 등) Timeout / Buffer 제한**

* 클라이언트는 유효한 요청을 보내지만 중간 프록시(예: Nginx) 또는 gateway 가 **buffer overflow**나 timeout 등의 이유로 500 반환.
* 서버 로그에 기록이 잘 안 남을 수도 있음.

### 5. **Server-side Internal Exception**

* 서버 API 내부에서 상태가 꼬인 상태에서 예외 발생 (예: DB 연결 timeout, NullPointerException, etc.)
* 재시작하면 내부 상태 초기화되어 잠시 정상화.

---

## 🔧 해결책 제안

### 🔹 클라이언트 측 대응 (Spring WebClient)

| 조치                       | 설명                                                                          |
| ------------------------ | --------------------------------------------------------------------------- |
| ✅ Connection pool 재사용 제한 | `WebClient`가 오래된 커넥션을 계속 재사용하면 문제가 될 수 있음.<br>→ 다음 설정으로 연결을 일정 시간 지나면 끊게 설정 |

```java
HttpClient httpClient = HttpClient.create()
    .responseTimeout(Duration.ofSeconds(10))
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
    .doOnConnected(conn -> conn
        .addHandlerLast(new ReadTimeoutHandler(10))
        .addHandlerLast(new WriteTimeoutHandler(10)))
    .keepAlive(false);  // <== 중요: keep-alive 끔

WebClient webClient = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

| 조치                       | 설명                                                                                                                                                      |
| ------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------- |
| ✅ ping 대신 실제 샘플 API call | `/ping`이 실제로 아무 리소스도 안 잡는 경우, 내부 세션이나 커넥션 사용 상태를 테스트하기 어렵다.<br>→ 실제 비즈니스 API 와 **유사한 구조**를 가진 `/sampleQuery` 나 `/fakeSearch` 같은 API 를 주기적으로 호출해보는 게 좋음. |

---

### 🔹 서버 측 대응

| 조치                                   | 설명                                                                                 |
| ------------------------------------ | ---------------------------------------------------------------------------------- |
| ✅ 로그 수준 올리기                          | `/ping`은 되는데, 실제 요청이 500 이 나면, 서버 **내부 API 로그**나 **예외 로그**를 확인해야 root cause 분석 가능. |
| ✅ Connection pool, DB pool 상태 확인     | DB 커넥션을 제대로 반납하지 않거나, 특정 상태에서 계속 누수되면 누적되고 결국 500 에러 유발.                           |
| ✅ API에 대한 `@Transactional`, 에러 캐치 확인 | 내부 API에서 Exception 이나 NullPointerException 등의 서버 에러가 날 경우 catch 안하면 500 으로 응답됨.    |

---

## ❓ Client만으로 해결 가능한가?

* 완전히는 **불가능합니다**.
* 클라이언트 쪽에서 커넥션 재설정, timeout 관리, ping 등으로 **간접적으로 증상 완화는 가능**.
* 그러나 진짜 원인이 서버 내부 자원 고갈이라면 **서버 수정 없이는 근본 해결은 어려움**.

---

## 🔎 추천 디버깅 흐름

1. **WebClient 로그 레벨**을 `DEBUG`로 올려서 500 응답 나올 때 request/response 추적
2. 서버측 access log와 error log를 500 코드 중심으로 확인
3. 문제가 발생한 시간대의 서버 thread dump (heap dump도 가능하면 좋음)
4. DB 쪽 connection 누수 확인 (HikariCP 등 pool 상태)
5. 커넥션 pool / thread pool / file descriptor 누수 확인 (`lsof`, `netstat`, `jstack`, `jmap` 활용)

---

원하시면 `WebClient`를 안정적으로 구성하는 샘플 코드 전체도 드릴 수 있습니다.


----
정리해드리겠습니다.

---

## ✅ 현상 요약

* 클라이언트(Spring) 서버에서 다른 Spring 서버에 API 요청.
* 처음엔 정상 → 일정 시간 후부터 **간헐적 500 에러 발생.**
* 클라이언트 서버를 재시작하면 다시 정상 → 일정 시간 후 재발.
* 최근에는 URI 조립 문제는 해결되었고, 이제 **500 오류만 해결 필요.**

---

## 🔍 가능한 원인 (정리)

| 원인                              | 설명                                                         | 해결 주체            |
| ------------------------------- | ---------------------------------------------------------- | ---------------- |
| 1. **Connection Pool 고갈 / 누수**  | `WebClient` 또는 `RestTemplate`에서 커넥션이 닫히지 않아 Pool이 고갈될 수 있음 | Client 측         |
| 2. **Keep-Alive 연결 Idle Close** | 서버가 idle 커넥션을 일정 시간 후 강제로 close. 그 후 다시 재사용하려다 500 발생      | Client or Server |
| 3. **서버 쪽 리소스 문제**              | 서버 측에서 요청이 많아지며 GC, 스레드, DB 연결 등 문제 발생 → 500 반환            | Server           |
| 4. **웹서버 프록시(Nginx 등)**         | 프록시에서 idle 커넥션 타임아웃 설정이 짧으면 끊김                             | 주로 Server        |
| 5. **에러 응답을 500으로 반환하는 코드**     | 실제 에러는 400, 401, 503 등인데 일괄적으로 500으로 보내는 경우                | Server           |

---

## ✅ KeepAlive 대책: `ping` 대신 **샘플 API 반복 호출하면 어떨까?**

> ✔️ 결론: 매우 좋은 방법일 수 있음. 이유는 다음과 같습니다.

| 방식                             | 장점                                          | 단점                                   |
| ------------------------------ | ------------------------------------------- | ------------------------------------ |
| `/ping` 같은 dummy 호출            | 단순하지만 서버에서 특별한 처리가 없어 효과가 제한적일 수 있음         | 실제 API 경로와 달라서 캐시 또는 연결에 실효성이 떨어질 수도 |
| **실제 API (ex. /users/1 등) 호출** | 커넥션 reuse, 인증, 프록시 경로 모두 실제 환경 그대로 동작 확인 가능 | 리소스를 일부 사용하므로 너무 자주 호출은 X            |

> 💡 즉, 실제 쓰는 API 중 부담 적은 read-only API 하나를 주기적으로 호출하는 게 가장 실용적입니다.

```java
@Scheduled(fixedDelay = 30_000)
public void keepConnectionAlive() {
    webClient.get()
        .uri("/health/readiness") // 또는 /users/1 처럼 read-only API
        .retrieve()
        .bodyToMono(Void.class)
        .subscribe();
}
```

---

## 🧪 해결 방안 요약 (Client + Server 관점)

### 1. **클라이언트(Spring) 측에서 할 수 있는 해결책**

* ✅ `WebClient` 재사용 (`WebClient.builder()` 사용하여 Bean 등록)
* ✅ `keep-alive` 유지 위한 실제 API 호출 (위 예시)
* ✅ TCP 커넥션 timeout, response timeout 명시 설정
* ✅ 예외 발생 시 재시도 로직 추가 (`retryWhen`, `Resilience4j`)

### 2. **서버(Spring) 측 확인 포인트**

* ✅ `/actuator/health` 등 readiness endpoint가 정상인지
* ✅ 서버 로그에서 500 발생 시 stacktrace 확인
* ✅ timeout 설정이 짧은지 확인 (application.yml, nginx 등)
* ✅ 서버가 보내는 `Connection: close` 헤더 확인

---

## ✅ 다음 액션 플랜 제안

1. 클라이언트 쪽 `keep-alive` 유지용으로 실제 API를 주기적으로 호출 (30\~60초 간격)
2. WebClient 인스턴스를 매번 만들지 말고 **Bean으로 싱글턴 재사용**
3. 서버 쪽 로그에서 500 에러 stacktrace 추적 → 원인 더 좁히기
4. 서버 측 actuator 활성화해서 `/actuator/metrics/http.server.errors` 확인
5. nginx, load balancer 쪽 timeout 설정 확인 (idle timeout 60초 이하인지)

---

필요하면 `tcpdump`, `actuator`, 또는 `Resilience4j` 설정 예시도 도와드릴 수 있습니다.
더 구체적인 로그나 설정 공유 주시면 디버깅도 함께 해드릴게요.


----

좋습니다. 현재 상황을 요약하고, 가능한 원인과 해결책을 **클라이언트(Spring WebClient)와 서버 양측 관점**에서 정리해드리겠습니다.

---

## ✅ 현상 요약

* Spring WebClient로 API 요청 시 간헐적으로 `500 Internal Server Error` 발생
* 클라이언트(Spring 서버)를 재시작하면 한동안 정상 동작 → 일정 시간 후 다시 `500`
* 일정 시간 요청이 없었다가 다시 호출하면 에러 발생 → *idle connection 관련 추정*

---

## 🔍 가능한 원인

### 1. **Idle Connection으로 인한 커넥션 종료**

* HTTP Keep-Alive 연결은 클라이언트에서 재사용을 시도하지만, 서버가 idle timeout 후 연결을 끊었을 경우 → 끊긴 connection을 WebClient가 재사용하려다 500 발생
* **특히 Netty 기반 WebClient는 커넥션 재시도 없이 예외로 터뜨리는 경우가 많음**

### 2. **Connection Pool에 Stale Connection 존재**

* WebClient는 커넥션 풀에서 stale(죽은) connection을 골라 쓸 수 있음
* 이로 인해 서버 입장에서 `socket closed` 또는 처리 오류 → 500 반환 가능성

### 3. **서버측 Timeout 또는 Reverse Proxy 문제**

* 서버가 오랫동안 idle이던 connection을 끊어버림 (e.g., Nginx `keepalive_timeout`, `idle_timeout`)
* 혹은 서버 내부 thread pool exhaustion 후 임의로 500 발생

---

## 🛠 해결 방법

### 📌 클라이언트(WebClient) 측에서 가능한 해결책

#### ✅ 1. **Connection TTL 및 Idle Timeout 설정**

```java
HttpClient httpClient = HttpClient.create()
    .option(ChannelOption.CONNECT_TIMEOUT_MILLIS, 5000)
    .responseTimeout(Duration.ofSeconds(10))
    .doOnConnected(conn ->
        conn.addHandlerLast(new ReadTimeoutHandler(10))
            .addHandlerLast(new WriteTimeoutHandler(10)))
    .keepAlive(true)
    .resolver(DefaultAddressResolverGroup.INSTANCE)
    .tcpConfiguration(tcp ->
        tcp.option(ChannelOption.SO_KEEPALIVE, true))
    .secure() // TLS 쓴다면

    // 중요: Idle timeout과 TTL 설정
    .connectionProvider(ConnectionProvider.builder("custom")
        .maxIdleTime(Duration.ofSeconds(60)) // 60초 이상 idle이면 닫음
        .maxLifeTime(Duration.ofMinutes(5))  // 커넥션은 최대 5분 후 강제 종료
        .evictInBackground(Duration.ofSeconds(30)) // 백그라운드 evict 주기
        .build());

WebClient webClient = WebClient.builder()
    .clientConnector(new ReactorClientHttpConnector(httpClient))
    .build();
```

#### ✅ 2. **Periodic Dummy Ping (Keep Alive)**

```java
@Scheduled(fixedDelay = 30_000)
public void keepAlive() {
    webClient.get()
        .uri("/ping")
        .retrieve()
        .bodyToMono(Void.class)
        .subscribe();
}
```

* *주의*: 이 방식은 idle 연결이 **운 좋게 살아있다면** 유지 가능하지만, 서버가 이미 끊은 경우엔 무의미
* 따라서 위 `connectionProvider` 설정과 **병행**해야 함

---

### 📌 서버 측에서 가능한 조치

#### ✅ 1. **Reverse Proxy 또는 Load Balancer 설정 확인**

* Nginx 예시:

```nginx
keepalive_timeout  75s;
proxy_read_timeout 90s;
```

* AWS ALB:

  * idle timeout 기본 60초 → 늘릴 수 있음

#### ✅ 2. **서버 자체의 Thread Pool, Connection 수 조정**

* Tomcat, Undertow, Netty 설정 확인
* 만약 request thread가 고갈되어 500이 발생한다면 성능 설정 문제

#### ✅ 3. **서버 로그 확인**

* 500 에러의 정확한 로그나 stack trace가 중요함
  → 서버에서 어떤 예외나 조건으로 500을 주는지 반드시 확인

---

## 🎯 결론: 어느 쪽에서 해결해야 하나?

| 구분                 | 해결 가능 여부  | 주요 대응                                          |
| ------------------ | --------- | ---------------------------------------------- |
| ✅ 클라이언트(WebClient) | **가능**    | connection pool TTL 설정, stale connection evict |
| ✅ 서버 (API 제공자)     | **가능**    | idle timeout 늘리기, reverse proxy 설정 확인          |
| 🔁 협력 필요           | **양쪽 조정** | idle 관리 정책을 서로 맞춰야 함                           |

---

## 🔚 추천 순서

1. 클라이언트 측 `ConnectionProvider` 설정으로 idle & TTL 관리
2. 서버측 `keepalive_timeout`, `proxy_read_timeout` 확인 및 로그 분석
3. 필요시 클라이언트 ping 보냄 (`/ping`)으로 보완

---

필요하면 Netty 또는 Apache HttpClient 기반 설정도 비교해 드릴 수 있습니다.
Daniel이 사용하는 정확한 WebClient 설정 코드 공유해주시면 더 맞춤 분석도 가능해요.


-----

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


