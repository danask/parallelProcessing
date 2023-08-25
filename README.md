# Parallel Processing

1Spring에서 `@Cacheable` 애노테이션을 사용하지 않고 캐시를 수동으로 실행하려면 `CacheManager`와 `Cache`를 직접 사용하여 캐싱 작업을 수행할 수 있습니다. 아래는 이를 수행하는 예제 코드입니다.

1. **의존성 설정**:

   Spring 캐시 관련 라이브러리에 대한 의존성을 추가해야 합니다. 아래는 Maven을 사용하는 경우의 예제입니다.

   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-cache</artifactId>
   </dependency>
   ```

2. **CacheManager 주입**:

   먼저, `CacheManager`를 주입합니다. 이것을 사용하여 캐시를 관리할 수 있습니다.

   ```java
   import org.springframework.cache.CacheManager;
   import org.springframework.stereotype.Service;
   import org.springframework.cache.Cache;
   
   @Service
   public class MyCacheService {
       private final CacheManager cacheManager;
   
       public MyCacheService(CacheManager cacheManager) {
           this.cacheManager = cacheManager;
       }
   }
   ```

3. **캐시에 데이터 추가 및 검색**:

   이제 `CacheManager`를 사용하여 캐시에 데이터를 추가하고 검색할 수 있습니다.

   ```java
   import org.springframework.cache.CacheManager;
   import org.springframework.stereotype.Service;
   import org.springframework.cache.Cache;
   
   @Service
   public class MyCacheService {
       private final CacheManager cacheManager;
   
       public MyCacheService(CacheManager cacheManager) {
           this.cacheManager = cacheManager;
       }
   
       public void addToCache(String cacheName, String key, Object value) {
           Cache cache = cacheManager.getCache(cacheName);
           if (cache != null) {
               cache.put(key, value);
           }
       }
   
       public Object getFromCache(String cacheName, String key) {
           Cache cache = cacheManager.getCache(cacheName);
           if (cache != null) {
               Cache.ValueWrapper valueWrapper = cache.get(key);
               if (valueWrapper != null) {
                   return valueWrapper.get();
               }
           }
           return null;
       }
   }
   ```

4. **캐시 사용**:

   이제 `MyCacheService`를 사용하여 캐시에 데이터를 추가하고 검색할 수 있습니다.

   ```java
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.web.bind.annotation.GetMapping;
   import org.springframework.web.bind.annotation.RestController;
   
   @RestController
   public class MyController {
       @Autowired
       private MyCacheService myCacheService;
   
       @GetMapping("/addToCache")
       public void addToCache() {
           myCacheService.addToCache("myCache", "key1", "value1");
       }
   
       @GetMapping("/getFromCache")
       public String getFromCache() {
           Object value = myCacheService.getFromCache("myCache", "key1");
           return value != null ? value.toString() : "Cache miss";
       }
   }
   ```

   위의 코드에서 `/addToCache` 엔드포인트를 호출하면 "myCache"라는 캐시에 "key1"과 "value1"이라는 데이터가 추가됩니다. 그 후 `/getFromCache` 엔드포인트를 호출하면 캐시에서 데이터를 가져올 수 있습니다.

이렇게 하면 `@Cacheable` 애노테이션을 사용하지 않고도 프로그래밍 방식으로 캐시를 다룰 수 있습니다. `CacheManager` 및 `Cache`를 사용하여 캐싱 작업을 직접 관리할 수 있습니다.


2.
Java의 `CompletableFuture.supplyAsync`를 사용하여 결과를 리턴하지 않도록 처리하려면, 비동기 작업에서 `null`을 반환하면 됩니다. `supplyAsync` 메서드는 `Supplier` 함수를 사용하며, `Supplier` 함수의 반환 유형이 `T`인 경우에는 작업의 결과를 반환하고, `null`을 반환하면 작업의 결과를 무시합니다.

예를 들어, 다음과 같이 `supplyAsync`를 사용하여 결과를 리턴하지 않는 비동기 작업을 만들 수 있습니다:

```java
import java.util.concurrent.CompletableFuture;

public class CompletableFutureExample {
    public static void main(String[] args) {
        CompletableFuture<Void> future = CompletableFuture.supplyAsync(() -> {
            // 비동기 작업 내용
            System.out.println("Async task is running.");
            return null; // 결과를 반환하지 않음
        });

        // 작업이 완료될 때까지 대기
        future.join(); // 또는 future.get()을 사용할 수도 있음

        System.out.println("Async task has completed.");
    }
}
```

위 코드에서는 `supplyAsync`로 생성된 `CompletableFuture`에 `null`을 반환하여 결과를 무시하고 있습니다. 이렇게 하면 작업이 비동기적으로 실행되고, 결과를 받을 필요가 없을 때 유용합니다. 결과를 받지 않는 경우에는 `CompletableFuture<Void>`를 사용할 수 있습니다.

3
`CompletableFuture`에 timeout을 설정하려면 Java의 `ScheduledExecutorService`를 사용하여 `CompletableFuture`를 wrapping하는 방법을 고려할 수 있습니다. 아래는 timeout을 설정한 `CompletableFuture`를 만드는 예제입니다:

```java
import java.util.concurrent.*;

public class CompletableFutureWithTimeout {
    public static void main(String[] args) {
        // 기본 ExecutorService 생성
        ExecutorService executor = Executors.newCachedThreadPool();

        // Timeout 설정
        long timeoutMillis = 1000; // 1초
        ScheduledExecutorService timeoutExecutor = Executors.newScheduledThreadPool(1);

        // 비동기 작업 생성
        CompletableFuture<String> future = CompletableFuture.supplyAsync(() -> {
            // 여기서 오래 걸리는 작업을 수행
            try {
                Thread.sleep(2000); // 2초 대기
                return "Task completed.";
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
                return "Task interrupted.";
            }
        }, executor);

        // Timeout 설정
        CompletableFuture<String> result = future.orTimeout(timeoutMillis, TimeUnit.MILLISECONDS);

        // 결과 처리
        result.whenComplete((response, throwable) -> {
            if (throwable != null) {
                System.out.println("Timeout occurred.");
            } else {
                System.out.println("Result: " + response);
            }

            // Executor 종료
            executor.shutdown();
            timeoutExecutor.shutdown();
        });

        // Executor 종료 대기
        try {
            executor.awaitTermination(5, TimeUnit.SECONDS);
            timeoutExecutor.awaitTermination(5, TimeUnit.SECONDS);
        } catch (InterruptedException e) {
            Thread.currentThread().interrupt();
        }
    }
}
```

위 코드에서는 `CompletableFuture.supplyAsync`를 사용하여 비동기 작업을 생성하고, `orTimeout` 메서드를 사용하여 timeout을 설정합니다. 결과를 처리하기 위해 `whenComplete` 메서드를 사용합니다.

주의할 점은 `orTimeout`은 Java 9 이상에서 사용 가능하며, Java 8에서는 직접 timeout을 구현해야 합니다.

또한, `ExecutorService`를 생성하고 사용한 후에는 명시적으로 종료해야 하므로, `awaitTermination`을 사용하여 종료를 대기합니다.


4
Java에서 스레드 풀을 구성하려면 `ExecutorService`를 사용합니다. `ExecutorService`는 스레드 풀을 관리하고 스레드로 작업을 제출할 수 있는 인터페이스입니다. 스레드 풀을 구성하려면 다음 단계를 따릅니다:

1. **ExecutorService 생성**:

   `ExecutorService`를 생성하여 스레드 풀을 만듭니다. 일반적으로 `Executors` 클래스를 사용하여 스레드 풀을 생성합니다. 다음은 몇 가지 스레드 풀 생성 방법의 예입니다.

   - **고정 크기 스레드 풀**:
     ```java
     ExecutorService executorService = Executors.newFixedThreadPool(5); // 5개의 스레드를 가진 스레드 풀
     ```

   - **캐시 스레드 풀**:
     ```java
     ExecutorService executorService = Executors.newCachedThreadPool(); // 필요에 따라 스레드를 생성하는 동적 스레드 풀
     ```

   - **단일 스레드 풀**:
     ```java
     ExecutorService executorService = Executors.newSingleThreadExecutor(); // 하나의 스레드를 사용하는 스레드 풀
     ```

2. **작업 제출**:

   스레드 풀에 작업을 제출하려면 `execute()` 또는 `submit()` 메서드를 사용합니다. `execute()`는 `Runnable` 작업을, `submit()`은 `Callable` 또는 `Runnable` 작업을 제출할 수 있습니다.

   ```java
   executorService.execute(() -> {
       // 작업 내용
   });

   Future<String> future = executorService.submit(() -> {
       // 작업 내용
       return "결과";
   });
   ```

3. **작업 완료 대기**:

   `submit()`을 사용하여 작업을 제출하면 `Future` 객체를 반환합니다. 이를 통해 작업이 완료될 때까지 대기하고 결과를 가져올 수 있습니다.

   ```java
   try {
       String result = future.get(); // 작업이 완료될 때까지 대기하고 결과를 가져옴
       System.out.println("결과: " + result);
   } catch (InterruptedException | ExecutionException e) {
       // 예외 처리
   }
   ```

4. **스레드 풀 종료**:

   스레드 풀을 더 이상 사용하지 않을 때는 명시적으로 종료해야 합니다. 종료하지 않으면 프로그램이 종료되지 않을 수 있습니다.

   ```java
   executorService.shutdown(); // 스레드 풀 종료
   ```

   `shutdown()` 메서드는 현재 진행 중인 작업을 완료한 뒤 스레드 풀을 종료합니다. 만약 완료되지 않은 작업을 중단하고 싶다면 `shutdownNow()` 메서드를 사용합니다.

이것으로 스레드 풀을 구성하고 사용하는 기본적인 방법을 알아보았습니다. 스레드 풀을 조절하거나 다른 구성 옵션을 사용하려면 `ThreadPoolExecutor` 클래스를 직접 사용하거나 Spring의 `TaskExecutor` 인터페이스를 사용하는 등의 방법도 있습니다.
