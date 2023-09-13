아래는 Spring WebFlux와 Mono를 사용하여 Entity를 이용해서 응답을 먼저 Front-end로 보내고 동시에 비동기로 최종 응답과는 관계 없는 다른 별도의 작업을 하는 예제입니다.

```java
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RestController;
import reactor.core.publisher.Mono;
import reactor.core.scheduler.Schedulers;

@RestController
public class MyController {

    @GetMapping("/api/some-endpoint")
    public Mono<MyEntity> getSomeData() {
        // Entity를 생성하여 초기 응답을 만듭니다.
        MyEntity initialResponse = new MyEntity("Initial response");

        // 초기 응답을 Mono로 래핑하여 Front-end로 보냅니다.
        Mono<MyEntity> responseMono = Mono.just(initialResponse);

        // 별도의 비동기 작업을 수행합니다.
        Mono<Void> asyncTask = Mono.fromRunnable(() -> {
            // 비동기 작업 시뮬레이션 (예: 로깅 또는 데이터베이스 저장)
            try {
                Thread.sleep(2000); // 2초 대기
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
            System.out.println("Async task completed.");
        }).subscribeOn(Schedulers.elastic());

        // 비동기 작업과 응답을 결합하여 최종 응답을 만듭니다.
        return responseMono.then(asyncTask).then(Mono.just(initialResponse));
    }
}
```

이 코드에서는 `/api/some-endpoint`로 요청이 오면 Entity를 생성하여 초기 응답을 만들고, Mono를 사용하여 Front-end로 보냅니다. 그런 다음 별도의 비동기 작업을 Mono로 래핑하여 수행하고, 이 작업과 초기 응답을 결합하여 최종 응답을 반환합니다.

`subscribeOn(Schedulers.elastic())`를 사용하여 비동기 작업을 백그라운드 스레드에서 실행하도록 설정하였습니다. 이렇게 하면 응답을 먼저 보내고 나서도 비동기 작업이 백그라운드에서 병렬로 수행됩니다.

Front-end에서는 Mono를 구독하여 최종 응답을 처리할 수 있습니다. Mono가 완료되면 최종 응답과 동시에 비동기 작업이 수행된 것을 확인할 수 있습니다.


Reactor 3.4부터는 `Schedulers.elastic()` 대신에 `Schedulers.boundedElastic()`를 사용하는 것이 권장됩니다. `boundedElastic()`는 Elastic Scheduler의 개선된 버전으로, 새로운 프로젝트 리액터 스케줄러 모델에 더 잘 통합되어 있습니다.

따라서 최신 버전의 Reactor에서는 다음과 같이 `Schedulers.boundedElastic()`을 사용하는 것이 좋습니다.

```java
Mono<Void> asyncTask = Mono.fromRunnable(() -> {
    // 비동기 작업 시뮬레이션 (예: 로깅 또는 데이터베이스 저장)
    try {
        Thread.sleep(2000); // 2초 대기
    } catch (InterruptedException e) {
        Thread.currentThread().interrupt();
    }
    System.out.println("Async task completed.");
}).subscribeOn(Schedulers.boundedElastic());
```

`boundedElastic()`은 작업 스레드 풀을 보다 효율적으로 관리하고 리소스를 더 효과적으로 활용할 수 있도록 설계되었습니다. 따라서 비동기 작업을 스케줄링할 때 이 스케줄러를 사용하면 성능 및 확장성을 향상시킬 수 있습니다.


`boundedElastic`와 `parallel`은 Reactor에서 사용되는 스케줄러(Scheduler)입니다. 이 두 가지 스케줄러는 다음과 같은 차이점이 있습니다.

1. **Thread 개수 및 관리**: 

    - `boundedElastic`: `boundedElastic` 스케줄러는 스레드 풀의 크기를 제한하고, 작업 큐의 크기를 조절하여 리소스를 더 효율적으로 관리합니다. 이것은 CPU 바운드 및 I/O 바운드 작업 모두에 적합합니다. 스레드 개수가 제한되어 있기 때문에 너무 많은 작업이 동시에 실행되지 않도록 합니다.

    - `parallel`: `parallel` 스케줄러는 CPU 코어 수와 같은 수의 스레드를 생성합니다. 이 스케줄러는 주로 CPU 바운드 작업에 사용됩니다. 스레드 개수는 주어진 CPU 코어 수에 따라 자동으로 조절되므로 개발자가 스레드 수를 직접 관리할 필요가 없습니다.

2. **용도**:

    - `boundedElastic`: I/O 작업 또는 네트워크 작업과 같은 비동기 I/O 작업에 적합합니다. 스레드 풀의 크기를 제한하고 리소스 사용을 최적화하여 여러 비동기 작업을 동시에 처리할 때 유용합니다.

    - `parallel`: CPU 집약적인 작업에 적합합니다. CPU 코어 수에 따라 스레드를 자동으로 조절하므로 CPU 바운드 작업을 병렬로 처리하는 데 적합합니다.

3. **스레드 풀 크기 제한**:

    - `boundedElastic`: 스레드 풀 크기를 개발자가 제한할 수 있습니다. 따라서 사용 가능한 리소스에 따라 스레드 풀 크기를 조절할 수 있습니다.

    - `parallel`: CPU 코어 수에 따라 자동으로 스레드 풀 크기가 조절됩니다. 개발자는 일반적으로 스레드 풀 크기를 직접 제어하지 않습니다.

어떤 스케줄러를 선택할지는 작업의 성격과 사용 사례에 따라 다릅니다. CPU 바운드 작업을 수행할 때는 `parallel` 스케줄러가 적합하며, I/O 바운드 작업을 수행할 때는 `boundedElastic` 스케줄러를 고려해보는 것이 좋습니다. 이러한 스케줄러를 적절하게 선택하면 리액티브 애플리케이션의 성능을 최적화할 수 있습니다.

