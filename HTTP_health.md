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


----
