Spring Controller를 테스트할 때 `CompletableFuture`와 비동기 작업을 사용하는 경우, 일부 주의할 점이 있습니다. 특히 비동기 작업이 완료되기를 기다릴 필요가 있으며, 예외 처리 및 상태 코드 검증을 수행해야 할 것입니다. 아래는 Spring MVC 컨트롤러를 `CompletableFuture`와 함께 테스트하는 방법을 보여주는 간단한 예제입니다.

먼저, Maven 또는 Gradle과 같은 빌드 도구를 사용하여 Spring Test 및 관련 라이브러리를 의존성으로 추가해야 합니다. Spring Boot를 사용하고 있다면 일반적으로 아래와 같이 의존성을 추가할 수 있습니다:

```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

그런 다음, Spring의 `MockMvc`를 사용하여 컨트롤러를 테스트합니다. 아래는 테스트 코드의 예제입니다:

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RunWith(SpringRunner.class)
@WebMvcTest(YourController.class)
public class YourControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testYourController() throws Exception {
        // 모의 비동기 작업을 만듭니다.
        CompletableFuture<String> asyncTask = CompletableFuture.supplyAsync(() -> {
            // 여기에 비동기 작업을 정의합니다.
            return "Result of async task";
        });

        // 컨트롤러 메서드에서 CompletableFuture를 반환하도록 모의합니다.
        when(yourService.someAsyncMethod()).thenReturn(asyncTask);

        // 컨트롤러의 엔드포인트를 호출합니다.
        mockMvc.perform(get("/your-endpoint"))
            .andExpect(status().isOk()) // 예상되는 HTTP 상태 코드를 검증합니다.
            .andExpect(content().string("Result of async task")); // 응답 본문을 검증합니다.
    }
}
```

위의 예제에서는 `YourController` 클래스를 `@WebMvcTest`로 테스트하고, `CompletableFuture`를 사용하여 모의 비동기 작업을 만들어 컨트롤러에서 반환하도록 설정합니다. 그런 다음 `MockMvc`를 사용하여 컨트롤러를 호출하고 응답을 검증합니다.

또한, 테스트 시에 컨트롤러에서 예외를 던질 가능성이 있는 경우 해당 예외를 기대하고 처리하는 방법도 고려해야 합니다.

이제 테스트가 실패하고 500 에러가 발생하는 경우, 로그와 스택 트레이스를 검토하여 문제를 식별하고 해결할 수 있어야 합니다. 로깅 레벨을 조정하여 더 자세한 디버그 정보를 얻을 수도 있습니다.

--------------

`org.mockito.exceptions.misusing.UnfinishedStubbingException`은 Mockito에서 모의 객체(Mock)를 사용할 때 발생할 수 있는 예외입니다. 이 예외는 모의 객체에 대한 스텁(기대 동작)이 아직 완료되지 않은 경우 발생합니다.

이 예외를 해결하기 위해서는 다음과 같은 접근 방식을 고려해야 합니다:

1. **스텁 완료**: `when(...).thenReturn(...)` 또는 `when(...).thenThrow(...)`과 같은 Mockito 스텁을 사용할 때, 해당 스텁을 모두 완료해야 합니다. 예를 들어, `when(yourService.someAsyncMethod()).thenReturn(future);`에서 `future`를 완료하는 부분이 빠져 있는지 확인합니다.

   ```java
   CompletableFuture<String> future = new CompletableFuture<>();
   future.complete("Expected result");
   when(yourService.someAsyncMethod()).thenReturn(future);
   ```

2. **기대 호출 지정**: 실제로 테스트 메서드 내에서 모의 객체를 호출하는 코드를 포함해야 합니다. 모의 객체에 대한 스텁만 정의하고 실제로 호출하지 않으면 `UnfinishedStubbingException`이 발생합니다.

3. **실행 경로 검토**: 테스트 메서드의 실행 경로를 검토하여 스텁에 대한 호출과 실제 호출이 일치하는지 확인합니다. 예를 들어, 모의 객체에 대한 스텁을 정의한 후 실제 메서드를 호출하기 전에 스텁이 적용되도록 코드를 조정합니다.

다음은 예외가 발생하지 않도록 수정된 예제입니다:

```java
import static org.mockito.Mockito.*;

@RunWith(SpringRunner.class)
@SpringBootTest
public class YourServiceTest {

    @Autowired
    private YourService yourService;

    @Test(expected = YourCustomException.class)
    public void testYourService() throws YourCustomException {
        // 모의 CompletableFuture를 만듭니다.
        CompletableFuture<String> future = new CompletableFuture<>();
        future.completeExceptionally(new YourCustomException("Something went wrong"));

        // 서비스 함수를 모의하고 예외를 발생시키도록 설정합니다.
        when(yourService.someAsyncMethod()).thenReturn(future);

        // 서비스 함수 호출
        yourService.someAsyncMethod(); // 예외 발생을 기대합니다.
    }
}
```

이제 `when(yourService.someAsyncMethod()).thenReturn(future);`에서 스텁이 완료되었고 실제 호출이 있으므로 `UnfinishedStubbingException`이 발생하지 않아야 합니다.

-----------------------

Spring의 `MockMvc`를 사용하여 비동기 컨트롤러 엔드포인트를 테스트하려면 `CompletableFuture`가 아닌 동기적인 방식으로 테스트 코드를 작성해야 합니다. `MockMvc`는 동기적인 호출을 기대하므로 `CompletableFuture`를 직접 사용하는 것이 아니라 해당 결과를 기다리는 동기적인 코드로 테스트해야 합니다.

아래는 `CompletableFuture`를 사용하는 컨트롤러 엔드포인트를 테스트하는 예제입니다:

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.*;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.*;

@RunWith(SpringRunner.class)
@WebMvcTest(YourController.class)
public class YourControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testYourController() throws Exception {
        // 서비스 함수에서 CompletableFuture를 반환하도록 모의합니다.
        when(yourService.someAsyncMethod()).thenReturn(CompletableFuture.completedFuture("Expected Result"));

        // 컨트롤러의 엔드포인트를 호출하고 결과를 기다립니다.
        MvcResult result = mockMvc.perform(get("/your-endpoint"))
            .andExpect(status().isOk())
            .andReturn();

        // 응답 본문을 검증합니다.
        String responseBody = result.getResponse().getContentAsString();
        assertThat(responseBody).isEqualTo("Expected Result");
    }
}
```

이 테스트 코드에서는 `CompletableFuture.completedFuture("Expected Result")`를 사용하여 비동기 작업이 완료된 결과를 반환하도록 모의 설정하고, 컨트롤러를 호출한 후 `andReturn()`를 사용하여 결과를 기다립니다. 그런 다음 응답 본문을 검증하여 예상된 결과와 일치하는지 확인합니다.

중요한 점은 `MockMvc`를 사용하여 컨트롤러를 테스트할 때는 비동기적인 코드 대신 동기적인 방식으로 테스트 코드를 작성해야 한다는 것입니다. 컨트롤러가 비동기 작업을 수행하더라도 `MockMvc`는 동기적인 호출을 기대합니다.
