예외 처리 부분을 함수로 분리하고 `CompletableFuture`의 `get()` 메서드를 더 간단하게 표현할 수 있습니다. `Optional`을 이용한 방법은 함수형 스타일을 유지하면서도 예외 처리를 함수로 분리하여 코드의 가독성을 높일 수 있습니다.

### 1. 예외 처리 부분을 함수로 분리하기

```java
import java.util.Optional;
import java.util.concurrent.CompletableFuture;

public class Example {

    public static void main(String[] args) {
        CompletableFuture<String> completableFuture = ...;

        // get()을 예외 처리와 함께 분리한 함수 사용
        String result = Optional.ofNullable(completableFuture)
                                .map(Example::safeGet)
                                .orElse(null);
    }

    // 예외 처리를 포함한 CompletableFuture.get() 함수
    private static <T> T safeGet(CompletableFuture<T> future) {
        try {
            return future.get();
        } catch (Exception e) {
            throw new RuntimeException("Failed to get result from CompletableFuture", e);
        }
    }
}
```

### 2. 더 간단한 표현
예외 처리 로직을 함수로 분리한 덕분에 `Optional`을 사용하는 코드가 더 간결해졌습니다. 이 방식은 함수형 스타일을 유지하면서도 예외 처리를 재사용할 수 있어 가독성 및 유지 보수성이 향상됩니다.

- `safeGet` 메서드는 `CompletableFuture`의 `get()` 호출 시 발생할 수 있는 예외를 처리하는 역할을 합니다.
- `Optional.ofNullable`을 사용하여 `CompletableFuture`가 `null`인 경우에도 안전하게 처리할 수 있습니다.

이 방법은 코드의 간결성을 유지하면서도, 예외 처리를 명확하게 처리할 수 있게 도와줍니다.

-----------------------------

# Parallel Processing


Spring API에서 응답을 먼저 주고 후속 작업을 처리하는 방법이 있습니다. 이를 비동기적으로 처리하는 방식을 사용하면 됩니다. 비동기 처리를 사용하면 요청에 대한 응답을 먼저 클라이언트에게 보내고, 해당 요청에 대한 후속 작업은 다른 스레드나 프로세스에서 처리할 수 있습니다.

Spring에서는 비동기 처리를 위해 다음과 같은 방법들을 제공합니다:

1. DeferredResult 또는 CompletableFuture 사용:
DeferredResult는 Spring 3.2 이후에 도입된 클래스로, 비동기적으로 처리되는 작업의 결과를 나중에 제공할 수 있도록 합니다. CompletableFuture는 Java 8에서 추가된 기능으로, 비동기적으로 작업을 수행하고 완료되면 결과를 처리할 수 있도록 합니다.

```java
@RestController
public class MyController {

    @GetMapping("/api/some-endpoint")
    public DeferredResult<ResponseEntity<?>> getSomeData() {
        DeferredResult<ResponseEntity<?>> deferredResult = new DeferredResult<>();

        // 비동기 작업 실행
        CompletableFuture.supplyAsync(() -> {
            // 비동기적으로 처리할 작업 수행
            return "Some data from async process";
        }).whenComplete((result, throwable) -> {
            if (throwable != null) {
                // 에러 처리
                deferredResult.setErrorResult(ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("Error occurred"));
            } else {
                // 정상적인 결과 전달
                deferredResult.setResult(ResponseEntity.ok(result));
            }
        });

        return deferredResult;
    }
}
```

2. @Async 어노테이션 사용:
Spring에서는 @Async 어노테이션을 이용하여 메소드를 비동기적으로 실행할 수 있습니다.

```java
@RestController
public class MyController {

    @Autowired
    private MyService myService;

    @GetMapping("/api/some-endpoint")
    public ResponseEntity<String> getSomeData() {
        // 비동기적으로 처리할 작업 시작
        myService.doAsyncWork();

        // 응답을 바로 반환
        return ResponseEntity.ok("Response sent!");
    }
}

@Service
public class MyService {

    @Async
    public void doAsyncWork() {
        // 비동기적으로 처리할 작업 수행
        // 작업 결과는 이 메소드 외부에서 처리됨
    }
}
```

이렇게 하면 클라이언트에게 먼저 응답이 전달되고, MyService 클래스의 doAsyncWork() 메소드는 별도의 스레드에서 비동기적으로 실행되어 후속 작업을 수행합니다.

비동기 처리는 대규모 동시 요청 처리나 시간이 오래 걸리는 작업의 처리에 유용하며, 효율적인 리소스 사용과 높은 성능을 제공하는 데 도움이 됩니다. 하지만 적절한 상황에서 사용해야 하며, 작업의 특성과 환경을 고려하여 적절하게 적용하는 것이 중요합니다.


각 API가 응답 후에 후속 처리를 수행하려면 Spring의 DeferredResult 또는 CompletableFuture를 사용하고 응답 이후에 별도의 메소드로 처리하면 됩니다. 다음은 이를 어떻게 구현할 수 있는지 설명합니다:

1. 후속 처리를 담당할 별도의 메소드 정의:
컨트롤러 또는 서비스에 응답 이후의 후속 처리를 담당할 별도의 메소드를 생성합니다.

```java
@Service
public class MyService {

    public void handlePostProcessing(String data) {
        // 응답 데이터를 활용하여 후속 처리 작업을 수행합니다.
        // 이 메소드는 응답을 클라이언트에게 보낸 후에 비동기적으로 호출될 것입니다.
    }
}
```

2. DeferredResult 또는 CompletableFuture를 사용하여 비동기 처리:
API 메소드에서 DeferredResult 또는 CompletableFuture를 사용하여 비동기적으로 요청을 처리하고, 응답이 준비되면 후속 처리 메소드를 호출합니다.

DeferredResult를 사용한 예:

```java
@RestController
public class MyController {

    @Autowired
    private MyService myService;

    @GetMapping("/api/some-endpoint")
    public DeferredResult<ResponseEntity<?>> getSomeData() {
        DeferredResult<ResponseEntity<?>> deferredResult = new DeferredResult<>();

        // 비동기 처리
        CompletableFuture.supplyAsync(() -> {
            // 주요 처리를 수행하고 응답 데이터를 가져옵니다.
            String responseData = "비동기 처리로부터의 데이터";
            return responseData;
        }).whenComplete((result, throwable) -> {
            if (throwable != null) {
                // 에러가 발생한 경우 처리
                deferredResult.setErrorResult(ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body("에러가 발생했습니다."));
            } else {
                // 응답 결과 설정
                deferredResult.setResult(ResponseEntity.ok(result));
                // 응답 이후에 후속 처리 메소드를 비동기적으로 호출합니다.
                CompletableFuture.runAsync(() -> myService.handlePostProcessing(result));
            }
        });

        return deferredResult;
    }
}
```

CompletableFuture를 사용한 예:

```java
@RestController
public class MyController {

    @Autowired
    private MyService myService;

    @GetMapping("/api/some-endpoint")
    public ResponseEntity<String> getSomeData() {
        // 비동기 처리
        CompletableFuture.supplyAsync(() -> {
            // 주요 처리를 수행하고 응답 데이터를 가져옵니다.
            String responseData = "비동기 처리로부터의 데이터";
            return responseData;
        }).whenComplete((result, throwable) -> {
            if (throwable != null) {
                // 에러가 발생한 경우 처리
                // 필요한 경우 에러 응답 반환
            } else {
                // 응답 이후에 후속 처리 메소드를 비동기적으로 호출합니다.
                CompletableFuture.runAsync(() -> myService.handlePostProcessing(result));
            }
        });

        // 클라이언트에게 즉시 응답을 보냅니다.
        return ResponseEntity.ok("응답이 전송되었습니다!");
    }
}
```

DeferredResult 또는 CompletableFuture와 별도의 후속 처리 메소드를 함께 사용함으로써 각 API의 응답 후에 후속 처리를 수행할 수 있습니다. 후속 처리 메소드는 비동기적으로 호출되므로, 메인 처리 스레드가 클라이언트에게 즉시 응답을 반환하고, 추가 작업을 백그라운드에서 수행할 수 있습니다.


Spring에서 `@Cacheable` 애노테이션을 사용하지 않고 캐시를 수동으로 실행하려면 `CacheManager`와 `Cache`를 직접 사용하여 캐싱 작업을 수행할 수 있습니다. 아래는 이를 수행하는 예제 코드입니다.

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


----------------------------

리액티브 스프링의 `Mono`와 기존의 비동기 응답 방식(예: `CompletableFuture` 또는 스레드 기반의 비동기) 간에는 몇 가지 중요한 차이점이 있습니다.

1. **Reactive Programming**: 리액티브 스프링은 리액티브 프로그래밍 모델을 사용합니다. 이 모델은 데이터 스트림을 다루며, 데이터가 발생하는 시점에 대한 처리를 구성합니다. 이것은 비동기 처리와 데이터 흐름을 동시에 관리할 수 있게 해줍니다.

2. **Backpressure**: 리액티브 프로그래밍에서는 백프레셔(Backpressure)를 처리할 수 있습니다. 백프레셔는 데이터 소비자가 데이터 생산자의 속도를 조절할 수 있는 메커니즘입니다. 이것은 높은 처리량을 가진 스트림에서 데이터를 안정적으로 처리할 수 있게 도와줍니다.

3. **비동기성 관리**: 리액티브 프로그래밍에서는 스레드 관리를 개발자가 직접하지 않아도 됩니다. 스레드 풀 및 스레드 스케줄링에 대한 복잡한 관리가 필요하지 않으며, 리액티브 런타임이 이를 처리합니다.

4. **메모리 효율성**: 리액티브 프로그래밍은 메모리를 효율적으로 관리합니다. 작은 메모리 버퍼와 높은 처리량을 가진 스트림에서도 메모리 누수를 방지할 수 있습니다.

5. **콜백 지옥 방지**: 리액티브 프로그래밍은 비동기 코드를 작성할 때 콜백 지옥(Callback Hell)을 방지하는데 도움이 됩니다. `Mono`와 `Flux`는 각 단계의 비동기 작업을 연결하여 가독성이 뛰어난 코드를 작성할 수 있게 합니다.

6. **Reactor 라이브러리**: 리액티브 스프링은 Reactor 라이브러리를 기반으로 합니다. Reactor는 리액티브 스트림(Reactive Streams) 스펙의 구현체로서, 다양한 연산자를 제공하여 데이터 스트림을 조작할 수 있게 합니다.

비동기 응답(예: `CompletableFuture` 또는 스레드 기반 비동기)와 리액티브 스프링의 `Mono` 간에 선택은 프로젝트 요구 사항과 목표에 따라 달라집니다. 리액티브 프로그래밍은 대용량 데이터 처리 및 동시성이 요구되는 시나리오에서 빛을 발합니다. 그러나 작은 규모의 프로젝트나 기존의 동기 코드와의 통합 시, 기존의 비동기 방식도 매우 유효할 수 있습니다. 선택은 프로젝트의 성격과 개발자의 선호도에 따라 다를 것입니다.

기존의 `CompletableFuture` 기반 코드를 `Mono.fromFuture`를 사용하여 리액티브 스프링의 `Mono`로 변환하는 좀 더 복잡한 예제를 제공하겠습니다. 이번 예제에서는 비동기적으로 여러 데이터를 가져와서 조합하고, 그 결과를 `Mono`로 반환하는 과정을 보여줄 것입니다.

먼저, 다음과 같이 의존성을 설정합니다.

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-webflux</artifactId>
</dependency>
```

그런 다음, 아래 코드는 `CompletableFuture`를 `Mono`로 변환하는 복잡한 예제입니다.

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

import java.util.concurrent.CompletableFuture;

@RestController
public class MyController {

    @GetMapping("/api/complex-endpoint")
    public Mono<String> getComplexData() {
        // 첫 번째 비동기 작업
        CompletableFuture<String> future1 = CompletableFuture.supplyAsync(() -> {
            return "Data from async task 1";
        });

        // 두 번째 비동기 작업
        CompletableFuture<String> future2 = CompletableFuture.supplyAsync(() -> {
            return "Data from async task 2";
        });

        // 두 개의 CompletableFuture를 조합
        CompletableFuture<String> combinedFuture = future1.thenCombineAsync(future2, (result1, result2) -> {
            return result1 + " " + result2;
        });

        // CompletableFuture를 Mono로 변환하여 스케줄러에서 실행
        return Mono.fromFuture(() -> combinedFuture, Schedulers.elastic());
    }
}
```

위 코드에서는 `CompletableFuture`를 사용하여 두 개의 비동기 작업을 수행하고, 그 결과를 조합합니다. 그런 다음, `Mono.fromFuture`를 사용하여 `CompletableFuture`를 `Mono`로 변환하고, `Schedulers.elastic()`를 사용하여 결과를 다른 스레드에서 실행합니다.

이렇게 하면 `CompletableFuture`를 `Mono`로 변환하고 리액티브 스프링의 비동기 모델을 활용할 수 있습니다. 이것은 복잡한 비동기 작업을 다룰 때 유용한 방법 중 하나입니다.


`CompletableFuture`를 `Mono` 또는 `Flux`로 대체하려면 리액티브 프로그래밍 모델에 따라 비동기 작업을 처리해야 합니다. 아래는 `Mono`와 `Flux`를 사용하여 `CompletableFuture`를 대체하는 예제를 제공합니다.

**1. 단일 결과를 `Mono`로 처리:**

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;

@RestController
public class MyController {

    @GetMapping("/api/replace-completable-future")
    public Mono<String> replaceCompletableFuture() {
        return Mono.defer(() -> {
            // 비동기 작업을 Mono로 감싸기
            return Mono.fromFuture(() -> {
                // 비동기 작업 수행
                return CompletableFuture.supplyAsync(() -> "Data from CompletableFuture");
            });
        });
    }
}
```

위의 코드에서는 `Mono.defer()`를 사용하여 비동기 작업을 `Mono`로 감쌉니다. 그리고 `Mono.fromFuture()`를 사용하여 `CompletableFuture`를 `Mono`로 변환합니다.

**2. 여러 결과를 `Flux`로 처리:**

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Flux;

@RestController
public class MyController {

    @GetMapping("/api/replace-completable-future-list")
    public Flux<String> replaceCompletableFutureList() {
        return Flux.defer(() -> {
            // 비동기 작업을 Flux로 감싸기
            return Flux.fromFuture(() -> {
                // 비동기 작업 수행
                return CompletableFuture.supplyAsync(() -> "Data 1")
                        .thenApplyAsync(result1 -> {
                            // 다른 비동기 작업
                            return "Data 2";
                        });
            });
        });
    }
}
```

위 코드에서는 여러 비동기 작업을 처리하고, 각 작업 결과를 `thenApplyAsync`를 사용하여 조합하고 있습니다. 이렇게 여러 작업의 결과를 `Flux`로 처리할 수 있습니다.

이러한 방식으로 `CompletableFuture`를 `Mono` 또는 `Flux`로 대체할 수 있으며, 리액티브 프로그래밍의 모델을 활용하여 비동기 작업을 처리할 수 있습니다.



