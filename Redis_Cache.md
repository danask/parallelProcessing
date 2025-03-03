네! `CacheInitializer` 클래스는 **Spring Bean**으로 등록되므로, 따로 트리거(trigger)할 필요 없이 **애플리케이션이 시작될 때 자동으로 실행**됩니다.  

Spring Boot에서 `@EventListener(ApplicationReadyEvent.class)`는 **애플리케이션이 모든 빈을 초기화한 후 자동으로 실행**되기 때문입니다.

---

### 🔹 실행 과정 요약
1. `CacheInitializer`가 Spring Bean으로 등록됨 (`@Component` 사용)
2. Spring Boot 애플리케이션이 **모든 빈을 초기화한 후** `ApplicationReadyEvent`를 발생시킴
3. `@EventListener(ApplicationReadyEvent.class)`가 자동으로 실행됨
4. 캐시를 초기화 (`cacheManager.getCacheNames().forEach(...)`)
5. **애플리케이션이 정상적으로 시작됨** 🚀

---

### ✅ `CacheInitializer`는 자동 실행됨 (트리거 불필요)
따라서, 별도의 메서드 호출 없이 아래처럼 `@Component`만 선언해두면 **Spring이 자동으로 실행**해 줍니다.

```java
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.cache.CacheManager;
import org.springframework.context.event.EventListener;
import org.springframework.stereotype.Component;

@Component
public class CacheInitializer {

    private final CacheManager cacheManager;

    public CacheInitializer(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @EventListener(ApplicationReadyEvent.class)
    public void clearAllCachesOnStartup() {
        System.out.println("Clearing all caches at startup...");
        cacheManager.getCacheNames().forEach(cacheName -> {
            System.out.println("Clearing cache: " + cacheName);
            cacheManager.getCache(cacheName).clear();
        });
        System.out.println("All caches cleared.");
    }
}
```
**이 코드는 자동 실행되므로 `main()`에서 직접 호출할 필요 없음!**  

---

### 🔹 실행 로그 예시
애플리케이션을 실행하면 콘솔에서 다음과 같은 로그가 출력될 것입니다.

```
Clearing all caches at startup...
Clearing cache: managedApps
Clearing cache: userSessions
Clearing cache: productData
All caches cleared.
```

---

### 🔹 만약 특정 캐시만 초기화하고 싶다면?
모든 캐시가 아닌 특정 캐시만 초기화하려면 아래처럼 수정하면 됩니다.

```java
@EventListener(ApplicationReadyEvent.class)
public void clearSpecificCachesOnStartup() {
    String[] cachesToClear = {"userSessions", "productData"};
    
    System.out.println("Clearing specific caches at startup...");
    for (String cacheName : cachesToClear) {
        if (cacheManager.getCache(cacheName) != null) {
            System.out.println("Clearing cache: " + cacheName);
            cacheManager.getCache(cacheName).clear();
        }
    }
    System.out.println("Selected caches cleared.");
}
```

---

### 결론 🎯  
✅ `CacheInitializer`는 **Spring Boot가 자동으로 실행**해 주므로, 따로 트리거할 필요 없음!  
✅ `@EventListener(ApplicationReadyEvent.class)` 덕분에 **빈 초기화가 끝난 후 안전하게 실행**됨  
✅ 특정 캐시만 초기화하고 싶다면 캐시 이름을 지정하여 `clear()` 호출하면 됨  

이제 `@Cacheable`로 저장된 캐시가 애플리케이션 시작 시 자동으로 초기화됩니다! 🚀
------

Redis에 데이터를 `redisTemplate.opsForValue().set()`을 사용해 저장하려고 했는데, 저장이 되지 않는 이유는 여러 가지가 있을 수 있습니다. 여기서는 그 원인들을 하나씩 점검하는 방법을 알려드리겠습니다.

### 1. Redis 서버가 제대로 동작하는지 확인

- Redis 서버가 제대로 동작하고 있는지 확인하는 것이 우선입니다.
- Redis 서버가 실행 중인지, 정상적으로 연결이 되어 있는지 확인하세요. 예를 들어, Redis CLI를 사용해 직접 Redis에 접근해 키를 조회해볼 수 있습니다.

```bash
redis-cli
> keys *
```

만약 Redis 서버가 제대로 동작하지 않거나, 연결 문제로 인해 데이터를 저장하지 못하는 경우가 있을 수 있습니다.

### 2. `RedisTemplate` 설정 문제

- `RedisTemplate`이 제대로 설정되지 않으면 데이터가 Redis에 저장되지 않을 수 있습니다. 예를 들어, `keySerializer`, `valueSerializer`가 잘못 설정되면 직렬화 문제가 발생할 수 있습니다.
- 아래와 같이 `RedisTemplate`에 대한 설정을 다시 확인해 보세요.

```java
@Bean
public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
    RedisTemplate<String, Object> template = new RedisTemplate<>();
    template.setConnectionFactory(redisConnectionFactory);
    template.setKeySerializer(new StringRedisSerializer());
    template.setValueSerializer(new GenericJackson2JsonRedisSerializer()); // value 직렬화 방식
    return template;
}
```

이 설정을 통해, 키는 문자열로, 값은 JSON 형식으로 직렬화되며 Redis에 저장됩니다.

### 3. TTL 설정 확인

- TTL(Time to Live)을 설정했는데 데이터가 바로 사라지는 경우가 있을 수 있습니다. `redisTemplate.opsForValue().set()` 메서드에 설정한 TTL 값이 의도한 대로 설정되었는지 확인하세요.

```java
redisTemplate.opsForValue().set(FILE_A_STATUS_KEY, fileASaved, 1, TimeUnit.HOURS); // TTL 1시간 
```

- 만약 `1시간` 설정 후 즉시 데이터를 조회하면 값이 없는지 확인해야 합니다. 만약 즉시 사라지거나 Redis TTL이 너무 짧게 설정되었으면 문제가 발생할 수 있습니다.

### 4. Key Collision 문제

- Redis에서 `FILE_A_STATUS_KEY`와 같은 키로 이미 다른 데이터가 저장되어 있는지 확인해 보세요. 키가 충돌하고 다른 데이터가 덮어씌워지지 않는 상황일 수 있습니다.

```java
String existingValue = redisTemplate.opsForValue().get(FILE_A_STATUS_KEY);
System.out.println("Existing Value: " + existingValue);
```

### 5. 다른 캐시 메서드는 동작하지만 `redisTemplate` 메서드는 동작하지 않는 경우

- Redis에 데이터를 캐싱하기 위해 `@Cacheable`, `@CachePut`, `@CacheEvict` 같은 캐시 어노테이션을 사용한 경우에, `RedisTemplate`을 사용하는 메서드와 충돌하거나 잘못된 설정이 있을 수 있습니다.
- 캐시 메커니즘과 RedisTemplate이 동시에 사용되면 의도하지 않은 상황이 발생할 수 있으므로 캐시 전략과 `redisTemplate`의 사용 범위를 구분하는 것이 좋습니다.

### 6. 로그 확인 및 디버그

- Spring 로그를 확인하여 어떤 이유로 저장이 실패했는지 확인할 수 있습니다.
- `logging.level.org.springframework=DEBUG` 설정을 통해 자세한 로그를 남길 수 있으며, Redis와 관련된 문제가 있는지 확인해 보세요.

```properties
logging.level.org.springframework.data.redis=DEBUG
```

이 설정을 통해 Redis 관련 모든 동작을 디버깅할 수 있는 로그가 남겨집니다.

### 7. Redis Persistence 설정 확인

- 만약 Redis에서 특정 조건에서 데이터를 잃어버리는 경우가 발생하면 Redis의 데이터 지속성(persistence) 설정이 잘못되어 있을 수 있습니다. Redis의 설정 파일 `redis.conf`에서 `appendonly` 설정이 되어 있는지 확인해보세요.
  
```bash
appendonly yes
```

### 8. Redis 권한 설정 확인

- Redis에 데이터를 저장할 때, Redis의 권한 설정으로 인해 데이터가 저장되지 않는 경우도 있을 수 있습니다. Redis에 접근하는 유저나 인증 정보를 확인하세요.

### 9. Redis에 값이 정상적으로 저장되었는지 직접 확인

Redis CLI를 이용해 키가 잘 저장되었는지 확인할 수 있습니다.

```bash
redis-cli
> get FILE_A_STATUS_KEY
```

이 명령을 통해 Redis에 값이 정상적으로 저장되었는지 확인할 수 있습니다.

### 결론

위에 나열된 항목들을 하나씩 점검해보시고, 특히 `RedisTemplate`의 설정 문제 및 Redis 서버 상태를 우선적으로 확인하는 것이 좋습니다. 각 단계에서 디버깅을 하면서 원인을 추적할 수 있습니다.

--------------------------

해당 오류는 `RedisTemplate<String, Boolean>` 빈이 정의되지 않았기 때문에 발생한 문제입니다. 해결하기 위해서는 해당 타입의 `RedisTemplate` 빈을 명시적으로 정의해야 합니다.

### 해결 방법
Redis 설정 클래스에 `RedisTemplate<String, Boolean>` 빈을 정의하세요. 예를 들어 아래와 같이 작성할 수 있습니다.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericToStringSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Boolean> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Boolean> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());  // key 직렬화
        template.setValueSerializer(new GenericToStringSerializer<>(Boolean.class));  // value 직렬화
        return template;
    }
}
```

위 코드는 `String` 타입의 키와 `Boolean` 타입의 값을 가지는 `RedisTemplate`을 정의하는 설정입니다.

### 추가로 확인할 사항
1. **RedisConnectionFactory가 올바르게 설정되었는지 확인**: 만약 `RedisConnectionFactory`도 설정되지 않았다면, 이를 설정하는 코드도 필요합니다. 예를 들어 `LettuceConnectionFactory`를 사용하고 있다면 아래처럼 추가합니다.

    ```java
    import org.springframework.context.annotation.Bean;
    import org.springframework.data.redis.connection.RedisConnectionFactory;
    import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory();
    }
    ```

2. **의존성 확인**: `spring-boot-starter-data-redis` 의존성이 제대로 추가되어 있는지 확인하세요.

    **Gradle**:
    ```groovy
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    ```

    **Maven**:
    ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    ```

### 추가 로그 확인 방법
`application.properties`에 다음을 추가하여 더 자세한 디버그 정보를 확인할 수 있습니다.

```properties
logging.level.org.springframework=DEBUG
```

위 설정을 통해 스프링 부트가 `RedisTemplate` 관련 빈을 로드하는 과정에서 어떤 문제가 발생하는지 더 명확하게 확인할 수 있습니다.

위 내용을 적용해도 문제가 지속된다면, 추가적인 로그나 코드를 제공해 주시면 더 구체적인 문제를 파악할 수 있도록 도와드리겠습니다.

---------------------------

오류 메시지 "Consider defining a bean of type 'org.springframework.data.redis.core.RedisTemplate' in your configuration"는 스프링에서 `RedisTemplate` 타입의 빈을 찾을 수 없다는 의미입니다. 이는 `RedisTemplate`을 자동으로 주입하기 위해 스프링이 필요한 설정을 찾지 못했기 때문에 발생합니다.

이 문제를 해결하려면 스프링 설정 파일에 `RedisTemplate` 빈을 정의해야 합니다.

### 1. `RedisTemplate` 빈을 설정 파일에 추가

아래와 같이 `RedisTemplate` 빈을 스프링 설정 클래스에 정의할 수 있습니다:

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;

@Configuration
public class RedisConfig {

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);

        // 키에 String 직렬화 사용
        template.setKeySerializer(new StringRedisSerializer());

        // 값에 JSON 직렬화 사용
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());

        // 해시 키와 해시 값에 동일한 직렬화 사용
        template.setHashKeySerializer(new StringRedisSerializer());
        template.setHashValueSerializer(new GenericJackson2JsonRedisSerializer());

        return template;
    }
}
```

### 설명:

- **RedisConnectionFactory**: Redis와의 연결을 관리하는 `RedisConnectionFactory`는 보통 이미 설정되어 있어야 하며, 이를 `RedisTemplate`이 사용합니다.
- **RedisTemplate**: Redis에 데이터를 쉽게 저장하고 불러오기 위한 템플릿 클래스입니다.
- **StringRedisSerializer**: 키에 문자열을 직렬화할 때 사용됩니다.
- **GenericJackson2JsonRedisSerializer**: 값을 JSON 형식으로 직렬화할 때 사용됩니다.

이와 같이 설정을 하면 스프링에서 `RedisTemplate` 빈을 찾을 수 있게 되어 오류가 해결됩니다.
-----------------------

Redis를 활용하여 파일 `a`, `b`, `c`의 데이터 임시 저장 상태를 관리하고, 세 파일 모두 저장된 경우 해당 상태를 Redis에서 삭제하는 로직을 구현할 수 있습니다. 이를 위해 RedisTemplate을 사용하여 Redis에 임시 데이터를 저장하고, 조건이 충족되면 데이터를 삭제하는 방식으로 처리합니다.

### 구현 계획

1. **파일 저장 상태를 Redis에 임시 저장**:
   각 파일의 저장 상태를 키로 저장하여, 파일이 성공적으로 처리되면 상태 값을 `true`로 설정합니다.

2. **모든 파일이 저장되었는지 확인**:
   모든 파일의 상태가 `true`인 경우에만, Redis에서 해당 키들을 삭제합니다.

3. **Redis Template을 활용한 로직**:
   - `SET` 명령으로 파일 상태를 저장
   - `GET` 명령으로 파일 상태 확인
   - `DEL` 명령으로 모든 파일이 저장되면 상태값 삭제

### 예시 코드 (Spring RedisTemplate 사용)

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

import java.util.concurrent.TimeUnit;

@Service
public class FileImportService {

    @Autowired
    private RedisTemplate<String, Boolean> redisTemplate;

    private static final String FILE_A_STATUS_KEY = "file:a:status";
    private static final String FILE_B_STATUS_KEY = "file:b:status";
    private static final String FILE_C_STATUS_KEY = "file:c:status";
    
    // 파일 A 처리
    public void saveFileAData() {
        // 파일 A 데이터 저장 로직 (예: DB 저장 후)
        boolean fileASaved = true; // 예시: 파일 저장 성공 여부
        
        // Redis에 파일 A 저장 상태 임시 저장
        redisTemplate.opsForValue().set(FILE_A_STATUS_KEY, fileASaved, 1, TimeUnit.HOURS); // TTL 1시간
        checkAllFilesAndClear();
    }

    // 파일 B 처리
    public void saveFileBData() {
        // 파일 B 데이터 저장 로직
        boolean fileBSaved = true; // 예시: 파일 저장 성공 여부
        
        // Redis에 파일 B 저장 상태 임시 저장
        redisTemplate.opsForValue().set(FILE_B_STATUS_KEY, fileBSaved, 1, TimeUnit.HOURS);
        checkAllFilesAndClear();
    }

    // 파일 C 처리
    public void saveFileCData() {
        // 파일 C 데이터 저장 로직
        boolean fileCSaved = true; // 예시: 파일 저장 성공 여부
        
        // Redis에 파일 C 저장 상태 임시 저장
        redisTemplate.opsForValue().set(FILE_C_STATUS_KEY, fileCSaved, 1, TimeUnit.HOURS);
        checkAllFilesAndClear();
    }

    // 모든 파일이 저장되었는지 확인하고, 완료된 경우 Redis 상태값 삭제
    private void checkAllFilesAndClear() {
        Boolean fileAStatus = redisTemplate.opsForValue().get(FILE_A_STATUS_KEY);
        Boolean fileBStatus = redisTemplate.opsForValue().get(FILE_B_STATUS_KEY);
        Boolean fileCStatus = redisTemplate.opsForValue().get(FILE_C_STATUS_KEY);

        // 모든 파일이 저장되었는지 확인 (fileAStatus, fileBStatus, fileCStatus 모두 true 인지 확인)
        if (Boolean.TRUE.equals(fileAStatus) && Boolean.TRUE.equals(fileBStatus) && Boolean.TRUE.equals(fileCStatus)) {
            // 세 가지 상태가 모두 true 이면 Redis에서 상태값 삭제
            redisTemplate.delete(FILE_A_STATUS_KEY);
            redisTemplate.delete(FILE_B_STATUS_KEY);
            redisTemplate.delete(FILE_C_STATUS_KEY);
            
            // 추가 작업: 파일이 모두 저장되었을 때 수행할 작업
            System.out.println("All files are saved, clearing temporary statuses.");
        }
    }
}
```

### 로직 설명

1. **파일 상태 저장**:
   - `saveFileAData()`, `saveFileBData()`, `saveFileCData()` 메서드는 각각 파일을 처리한 후 그 파일의 상태를 Redis에 저장합니다.
   - `opsForValue().set(key, value, timeout, timeUnit)`를 사용하여 Redis에 파일 상태를 저장하고 TTL(Time-to-Live)을 설정합니다 (1시간).

2. **모든 파일이 저장되었는지 확인**:
   - `checkAllFilesAndClear()` 메서드는 Redis에서 각 파일의 상태 값을 읽어와서(`opsForValue().get(key)`), 모든 파일이 저장되었는지 확인합니다.
   - 세 파일 모두 `true`일 경우, Redis에 저장된 파일 상태 데이터를 삭제합니다 (`redisTemplate.delete()`).

3. **상태 초기화**:
   - 세 파일이 모두 저장되면 Redis에서 상태값이 삭제되므로, 나중에 다시 파일을 처리할 때 상태가 남아 있지 않도록 합니다.

### 예외 처리 및 확장 가능성

1. **TTL(Time-to-Live)**:
   - 임시 저장된 파일 상태는 일정 시간이 지나면 자동으로 삭제됩니다. 여기서는 1시간으로 설정했지만, 필요에 따라 변경할 수 있습니다.

2. **확장 가능성**:
   - 필요하다면 더 많은 파일을 처리하거나, 상태값을 추가하는 방식으로 확장할 수 있습니다. 예를 들어, 파일 외에 다른 데이터를 임시 저장하고 처리하는 방식으로 적용 가능합니다.

3. **에러 처리**:
   - 파일 저장 중 예외 상황이 발생하면 `false` 값을 Redis에 저장하여 실패한 상태를 추적할 수 있습니다.

### Redis 구조

- **Key**: `file:a:status`, `file:b:status`, `file:c:status` (각 파일의 저장 상태)
- **Value**: 파일 저장 성공 여부 (`true` 또는 `false`)

이렇게 하면 각 파일의 상태를 Redis에서 관리하고, 조건이 충족되면 해당 상태를 자동으로 삭제하여 효율적으로 데이터를 처리할 수 있습니다.

-----------------------------

Spring Boot에서 `CacheManager`가 잘 로드되었는지 확인하기 위해 디버그 로그를 설정하고, `CacheManager`가 생성될 때 로그를 찍어 확인할 수 있습니다. Spring 자체 로깅을 활용하거나, `SLF4J` 또는 `Log4j2` 같은 로깅 프레임워크를 사용해 캐시 설정 및 로딩 과정을 디버깅할 수 있습니다.

아래는 디버그 로그를 출력할 수 있는 몇 가지 방법과 코드 예시입니다.

### 1. **로깅 설정 추가 (application.properties)**

Spring의 캐시 관련 로깅을 활성화하려면 `application.properties` 파일에 아래 설정을 추가하여 디버그 레벨의 로그를 출력하도록 설정할 수 있습니다.

```properties
# Cache 관련 로깅을 디버그 레벨로 설정
logging.level.org.springframework.cache=DEBUG
```

위 설정을 추가하면 Spring에서 캐시 설정 및 `CacheManager` 빈이 로드될 때 발생하는 디버그 로그가 출력됩니다.

### 2. **CacheManager 생성 시 커스텀 로그 추가**

`CacheConfig` 클래스에서 `CacheManager`를 생성할 때 커스텀 로그를 추가하여 `CacheManager`가 잘 로드되었는지 확인할 수 있습니다. 

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.cache.CacheManager;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.cache.redis.RedisCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;

@Configuration
public class CacheConfig {
    
    private static final Logger logger = LoggerFactory.getLogger(CacheConfig.class);

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        try {
            RedisCacheManager cacheManager = RedisCacheManager.builder(redisConnectionFactory)
                    .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig())
                    .build();
            logger.debug("RedisCacheManager has been successfully loaded: {}", cacheManager);
            return cacheManager;
        } catch (Exception e) {
            logger.error("Failed to load RedisCacheManager, falling back to ConcurrentMapCacheManager", e);
            CacheManager fallbackCacheManager = new ConcurrentMapCacheManager();
            logger.debug("Fallback ConcurrentMapCacheManager has been loaded: {}", fallbackCacheManager);
            return fallbackCacheManager;
        }
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        try {
            template.setConnectionFactory(redisConnectionFactory);
            logger.debug("RedisTemplate has been successfully created with connection factory: {}", redisConnectionFactory);
        } catch (Exception e) {
            logger.error("Failed to create RedisTemplate", e);
        }
        return template;
    }
}
```

### 3. **디버그 로그 출력 확인**

위처럼 `logger.debug()` 메서드를 사용해 디버그 로그를 찍어두면, `CacheManager`가 잘 로드되었는지 애플리케이션 시작 시점에서 확인할 수 있습니다. RedisCacheManager 또는 ConcurrentMapCacheManager가 어떤 빈으로 로드되었는지 로그로 확인할 수 있습니다.

**로그 출력 예시 (Redis가 성공적으로 로드된 경우):**

```
DEBUG CacheConfig: RedisCacheManager has been successfully loaded: RedisCacheManager@12345
DEBUG CacheConfig: RedisTemplate has been successfully created with connection factory: LettuceConnectionFactory@6789
```

**로그 출력 예시 (Redis 실패, Fallback CacheManager 사용):**

```
ERROR CacheConfig: Failed to load RedisCacheManager, falling back to ConcurrentMapCacheManager
DEBUG CacheConfig: Fallback ConcurrentMapCacheManager has been loaded: ConcurrentMapCacheManager@abcde
```

### 4. **Spring Context에서 CacheManager 확인**

또한 `Spring Boot`의 `ApplicationContext`에서 `CacheManager` 빈이 잘 등록되었는지 확인할 수도 있습니다. `ApplicationListener`를 활용해서 스프링 부트 애플리케이션이 시작할 때 `CacheManager` 빈을 확인하고 로그를 출력할 수 있습니다.

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.cache.CacheManager;
import org.springframework.context.ApplicationListener;
import org.springframework.stereotype.Component;

@Component
public class CacheManagerChecker implements ApplicationListener<ApplicationReadyEvent> {

    private static final Logger logger = LoggerFactory.getLogger(CacheManagerChecker.class);

    private final CacheManager cacheManager;

    public CacheManagerChecker(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    @Override
    public void onApplicationEvent(ApplicationReadyEvent event) {
        logger.debug("CacheManager has been initialized: {}", cacheManager);
    }
}
```

이 클래스는 애플리케이션이 완전히 시작된 후(`ApplicationReadyEvent` 발생 후)에 현재 애플리케이션에서 사용되는 `CacheManager`를 로그로 출력합니다.

### 5. **JMX를 통한 모니터링**

추가로 JMX(Java Management Extensions)를 활성화하여 애플리케이션이 사용하는 `CacheManager` 및 캐시 상태를 모니터링할 수도 있습니다. 이를 위해 `application.properties`에 다음 설정을 추가하세요:

```properties
spring.jmx.enabled=true
```

그 후 JConsole 또는 VisualVM 등의 JMX 모니터링 도구를 사용하여 캐시 상태를 실시간으로 확인할 수 있습니다.

---

이렇게 설정하면 `CacheManager`가 잘 로드되었는지 디버그 로그를 통해 확인할 수 있으며, 문제 발생 시 적절히 디버깅할 수 있습니다.

----------------------------

스프링에서 Redis CacheManager가 실패하거나 캐시 설정이 실패하더라도 애플리케이션이 계속 동작하게 하고, 캐시 없이 처리되도록 하려면 `Fallback CacheManager`를 설정할 수 있습니다. 이를 통해 Redis가 동작하지 않더라도 기본적으로 캐싱이 없는 모드로 돌아갈 수 있습니다.

여기서는 Redis CacheManager가 실패할 때, 기본 `ConcurrentMapCacheManager` 같은 인메모리 캐시로 대체하는 방법을 사용해볼 수 있습니다.

### 해결 방법

1. **Fallback CacheManager 설정**: RedisCacheManager가 실패하면 기본적인 `ConcurrentMapCacheManager`를 사용할 수 있도록 설정합니다. 이로 인해 캐시가 실패하더라도 애플리케이션은 정상적으로 동작하게 됩니다.

2. **캐시 설정에 예외 처리를 추가**: `RedisConnectionFactory`나 `RedisCacheManager`를 생성할 때 예외를 처리하고, 예외가 발생하면 다른 CacheManager로 대체하는 방식으로 구현할 수 있습니다.

### 예시 코드

```java
import org.springframework.cache.CacheManager;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.cache.redis.RedisCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.StringRedisSerializer;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;

@Configuration
public class CacheConfig {
    private static final Logger logger = LoggerFactory.getLogger(CacheConfig.class);

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        try {
            RedisCacheManager cacheManager = RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
                    .entryTtl(Duration.ofHours(1))
                    .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                    .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()))
                    .disableCachingNullValues())
                .build();

            logger.info("RedisCacheManager is configured and active.");
            return cacheManager;
        } catch (Exception e) {
            logger.error("Failed to configure RedisCacheManager. Falling back to ConcurrentMapCacheManager.", e);
            return new ConcurrentMapCacheManager(); // Fallback CacheManager
        }
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        try {
            template.setConnectionFactory(redisConnectionFactory);
            template.setKeySerializer(new StringRedisSerializer());
            template.setValueSerializer(new GenericJackson2JsonRedisSerializer());

            logger.info("RedisTemplate is configured.");
        } catch (Exception e) {
            logger.error("Failed to configure RedisTemplate. It will not be used.", e);
        }
        return template;
    }
}
```

### 설명
1. **RedisCacheManager 예외 처리**: `cacheManager` 메서드에서 `RedisCacheManager`를 생성할 때 예외 처리를 추가했습니다. Redis 관련 설정에 문제가 생기면 `ConcurrentMapCacheManager`로 대체하여 애플리케이션이 중단되지 않도록 합니다.

2. **RedisTemplate 예외 처리**: `RedisTemplate`도 마찬가지로 Redis 연결이나 설정에 실패할 경우 로그를 남기고 넘어가며, Redis가 제대로 작동하지 않더라도 기본적인 애플리케이션 로직이 동작할 수 있도록 했습니다.

### 추가 사항
- **로깅**: RedisCacheManager 또는 RedisTemplate가 실패했을 때, 로깅을 통해 문제의 원인을 확인할 수 있습니다.
- **프로파일 활용**: Redis 환경과 인메모리 환경을 프로파일로 나누어 관리할 수도 있습니다.

### 테스트
이를 유닛 테스트에서 검증하기 위해서는 Redis 서버가 실행되지 않은 상태에서도 테스트가 가능하도록 Redis 관련 빈들을 `Mock`으로 대체할 수 있습니다. Redis가 없을 때에도 애플리케이션이 정상적으로 동작하는지 테스트할 수 있습니다.

```java
@RunWith(SpringRunner.class)
@SpringBootTest
@ActiveProfiles("test") // test 프로파일에서는 Redis를 사용하지 않도록 설정
public class CacheConfigTest {

    @Autowired
    private CacheManager cacheManager;

    @Test
    public void whenRedisFails_thenFallbackToConcurrentMapCacheManager() {
        assertTrue(cacheManager instanceof ConcurrentMapCacheManager);
    }
}
```

이 코드를 사용하면 캐시 설정 실패 시에도 기본적인 `ConcurrentMapCacheManager`로 애플리케이션이 계속 동작할 수 있습니다.

----------------------

`NoUniqueBeanDefinitionException` 에러는 `CacheManager` 타입의 빈이 두 개 이상 정의되어 있을 때 발생합니다. 이를 해결하려면, `CacheManager` 빈 정의를 명확하게 설정하고, 하나의 `CacheManager`만을 사용하도록 해야 합니다.

먼저, `CacheConfig` 클래스에서 각 `CacheManager` 빈을 명시적으로 정의합니다. 이렇게 하면 특정 프로파일이 활성화되었을 때에만 해당 `CacheManager`가 빈으로 등록되도록 설정할 수 있습니다.

### CacheConfig 수정
다음과 같이 `CacheConfig`를 수정하여 `redisCacheManager`를 기본 `CacheManager`로 사용하도록 합니다.

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.cache.interceptor.CacheResolver;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

@Configuration
@EnableCaching
public class CacheConfig {

    @Bean(name = "defaultCacheManager")
    @Profile("!redis")
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager();
    }

    @Bean(name = "redisCacheManager")
    @Profile("redis")
    @Primary
    @ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis", matchIfMissing = true)
    public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)) // 캐시 만료 시간 설정
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("managedApps", cacheConfiguration);

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .withInitialCacheConfigurations(cacheConfigurations)
                .build();
    }

    @Bean
    @ConditionalOnMissingBean(RedisTemplate.class)
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public CacheResolver cacheResolver(@Qualifier("redisCacheManager") CacheManager cacheManager) {
        return new CustomCacheResolver(cacheManager);
    }
}
```

이 설정에서는:

- `defaultCacheManager` 빈은 `redis` 프로파일이 활성화되지 않은 경우에만 생성됩니다.
- `redisCacheManager` 빈은 `redis` 프로파일이 활성화된 경우에 생성되며, `Primary` 어노테이션을 사용하여 기본 `CacheManager`로 설정됩니다.
- `cacheResolver`는 `redisCacheManager` 빈을 사용하도록 명시적으로 지정됩니다.

### Spring Profile 설정
프로파일을 활성화하려면, `application.properties` 파일에서 프로파일을 설정합니다.

```properties
# application.properties (default 프로파일)
spring.profiles.active=default

# application-redis.properties (redis 프로파일)
spring.cache.type=redis
```

또는, 테스트 코드나 애플리케이션 시작 시점에서 프로파일을 활성화할 수 있습니다.

### Spring Boot 애플리케이션에서 프로파일 설정
프로파일을 명시적으로 설정하는 방법:

```java
@SpringBootApplication
public class MyApplication {

    public static void main(String[] args) {
        SpringApplication application = new SpringApplication(MyApplication.class);
        application.setAdditionalProfiles("redis"); // 'redis' 프로파일을 활성화합니다.
        application.run(args);
    }
}
```

이 설정으로 인해 `redis` 프로파일이 활성화되면 `redisCacheManager`가 기본 `CacheManager`로 사용되고, `defaultCacheManager`는 생성되지 않습니다. 따라서 `NoUniqueBeanDefinitionException` 에러를 방지할 수 있습니다.

------------------------------

에러 메시지를 보면 'managedApps'라는 이름의 캐시를 찾을 수 없다는 것을 나타내고 있습니다. 이는 `RedisCacheManager`가 초기화될 때 해당 캐시가 정의되지 않았기 때문일 수 있습니다. 

이 문제를 해결하려면 `RedisCacheManager` 초기화 시점에 특정 캐시를 미리 정의해주어야 합니다.

### 캐시 이름 사전 정의

`RedisCacheManager`를 설정할 때, 특정 캐시 이름을 사전에 정의할 수 있습니다. 이를 통해 스프링 부트가 시작될 때 해당 캐시들이 존재하도록 보장할 수 있습니다.

아래와 같이 `RedisCacheManager` 설정에 캐시 이름을 미리 정의할 수 있습니다:

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.cache.CacheManager;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.HashMap;
import java.util.Map;

@Configuration
public class CacheConfig {

    @Bean(name = "defaultCacheManager")
    @Primary
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager();
    }

    @Bean(name = "redisCacheManager")
    @Profile("redis")
    @ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis", matchIfMissing = true)
    public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        Map<String, RedisCacheConfiguration> cacheConfigurations = new HashMap<>();
        cacheConfigurations.put("managedApps", cacheConfiguration);

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .withInitialCacheConfigurations(cacheConfigurations)
                .build();
    }

    @Bean
    @ConditionalOnMissingBean(RedisTemplate.class)
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

### 주의 사항
- `redisCacheManager` 빈 설정에서 `withInitialCacheConfigurations` 메서드를 통해 `managedApps` 캐시를 사전에 정의합니다.
- 이렇게 하면 스프링 부트가 시작될 때 `managedApps`라는 이름의 캐시가 존재하도록 보장됩니다.

### 테스트 설정 확인

테스트 설정에서도 `redis` 프로파일이 활성화되도록 하고, 올바른 `CacheManager`를 사용하도록 설정합니다:

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cache.CacheManager;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.junit.jupiter.SpringExtension;

@ExtendWith(SpringExtension.class)
@SpringBootTest
@ActiveProfiles("redis")
public class MyServiceTest {

    @Autowired
    @Qualifier("redisCacheManager")
    private CacheManager cacheManager;

    @Test
    public void testServiceMethod() {
        // 테스트 코드 작성
    }
}
```

위의 설정을 통해 캐시 이름을 사전에 정의하고, 이를 통해 캐시를 찾지 못하는 문제를 해결할 수 있습니다.
----------------------------------

`NoUniqueBeanDefinitionException` 에러를 해결하려면 스프링이 어떤 `CacheManager`를 사용할지 명확히 지정해줘야 합니다. 이전에 시도한 방법들이 작동하지 않는다면, 모든 `CacheManager` 빈의 이름을 명확히 지정하고 테스트 환경에서 해당 빈을 사용하도록 강제하는 방법을 고려할 수 있습니다.

### 해결 방법

1. 모든 `CacheManager` 빈에 명확한 이름을 지정합니다.
2. 테스트 환경에서 특정 `CacheManager`를 명확히 지정합니다.

아래는 이를 적용한 코드 예제입니다:

#### Cache Configuration Class

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.autoconfigure.condition.ConditionalOnMissingBean;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.cache.CacheManager;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

@Configuration
public class CacheConfig {

    @Bean(name = "defaultCacheManager")
    @Primary
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager();
    }

    @Bean(name = "redisCacheManager")
    @Profile("redis")
    @ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis", matchIfMissing = true)
    public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1))
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();
    }

    @Bean
    @ConditionalOnMissingBean(RedisTemplate.class)
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }
}
```

#### 테스트 클래스에서 특정 빈 사용

테스트 클래스에서 명확히 `redisCacheManager`를 사용하도록 설정합니다.

```java
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.cache.CacheManager;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.test.context.junit.jupiter.SpringExtension;

@ExtendWith(SpringExtension.class)
@SpringBootTest
@ActiveProfiles("redis")
public class MyServiceTest {

    @Autowired
    @Qualifier("redisCacheManager")
    private CacheManager cacheManager;

    @Test
    public void testServiceMethod() {
        // 테스트 코드 작성
    }
}
```

이렇게 하면 `redisCacheManager` 빈이 명확히 지정되어 사용됩니다. `@Qualifier`를 사용하여 `CacheManager` 빈을 특정하면, 스프링이 어떤 빈을 사용할지 명확히 지정할 수 있습니다.

### 추가 디버깅 팁

- **빈 정의 확인**: 애플리케이션 컨텍스트가 시작될 때, 등록된 모든 빈의 이름과 타입을 로그로 확인할 수 있습니다.
    ```java
    @Bean
    public CommandLineRunner commandLineRunner(ApplicationContext ctx) {
        return args -> {
            System.out.println("Let's inspect the beans provided by Spring Boot:");
            String[] beanNames = ctx.getBeanDefinitionNames();
            Arrays.sort(beanNames);
            for (String beanName : beanNames) {
                System.out.println(beanName);
            }
        };
    }
    ```

- **테스트 클래스에 프로파일 명확히 지정**: `@ActiveProfiles`를 사용하여 테스트 클래스가 올바른 프로파일을 사용하도록 합니다.
    ```java
    @ActiveProfiles("redis")
    @SpringBootTest
    public class MyServiceTest {
        // 테스트 코드 작성
    }
    ```

이 방법들을 통해 동일 타입의 두 개 이상의 `CacheManager` 빈으로 인한 `NoUniqueBeanDefinitionException` 오류를 해결할 수 있을 것입니다.

----------------------------

테스트 실행 시 `CacheManager` 빈이 두 개 이상 등록되어 `NoUniqueBeanDefinitionException` 예외가 발생하는 문제를 해결하기 위해 몇 가지 방법을 시도할 수 있습니다.

### 1. `@Primary` 애너테이션 사용

만약 `redisCacheManager`를 기본으로 사용하려면 `@Primary` 애너테이션을 추가하여 기본 `CacheManager`로 지정할 수 있습니다.

```java
@Bean
@Primary
@Profile("redis")
@ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis", matchIfMissing = true)
public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
    RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
            .entryTtl(Duration.ofHours(1))
            .disableCachingNullValues()
            .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
            .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

    return RedisCacheManager.builder(redisConnectionFactory)
            .cacheDefaults(cacheConfiguration)
            .build();
}
```

이 경우 `redisCacheManager`가 `@Primary` 애너테이션을 갖기 때문에 기본 `CacheManager`로 사용됩니다.

### 2. 테스트에서 특정 프로필 활성화

테스트 실행 시 특정 프로필을 활성화하여, `redisCacheManager` 또는 `defaultCacheManager` 중 하나만 활성화되도록 할 수 있습니다.

```java
import org.springframework.test.context.ActiveProfiles;

@ActiveProfiles("redis") // 'redis' 프로필을 활성화하여 redisCacheManager를 사용
@SpringBootTest
public class MyServiceTest {
    // 테스트 코드
}
```

### 3. 특정 빈 이름을 명시적으로 사용

테스트에서 명시적으로 특정 `CacheManager` 빈을 사용하도록 설정할 수도 있습니다.

```java
import org.springframework.beans.factory.annotation.Qualifier;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cache.CacheManager;
import org.springframework.test.context.junit.jupiter.SpringExtension;
import org.junit.jupiter.api.Test;
import org.junit.jupiter.api.extension.ExtendWith;

@ExtendWith(SpringExtension.class)
@SpringBootTest
public class MyServiceTest {

    @MockBean
    @Qualifier("redisCacheManager")
    private CacheManager cacheManager;

    @Test
    public void testServiceMethod() {
        // 테스트 코드 작성
    }
}
```

이렇게 하면 테스트 시 `redisCacheManager` 빈만을 사용하도록 강제할 수 있습니다.

### 4. 테스트 컨피그 클래스 정의

테스트 전용 설정 클래스를 만들어 테스트 실행 시 원하는 `CacheManager` 빈만 로드하도록 할 수도 있습니다.

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Primary;
import org.springframework.cache.CacheManager;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;

@TestConfiguration
public class TestCacheConfig {

    @Bean
    @Primary
    public CacheManager testCacheManager() {
        return new ConcurrentMapCacheManager();
    }
}
```

그리고 테스트 클래스에서 이 설정 클래스를 사용하도록 합니다.

```java
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ContextConfiguration;

@ContextConfiguration(classes = TestCacheConfig.class)
@SpringBootTest
public class MyServiceTest {
    // 테스트 코드
}
```

위 방법 중 하나를 선택하여 적용하면 `CacheManager` 빈 충돌 문제를 해결할 수 있습니다.

-----------------------------

테스트에서 일부 요청이 500 에러로 응답하는 문제를 해결하려면 다음 사항들을 점검해야 합니다. 일부 테스트만 실패하는 이유는 특정 조건이나 설정에서 문제가 발생하고 있을 가능성이 큽니다.

### 1. 에러 로그 확인
에러 로그를 통해 500 에러의 정확한 원인을 파악합니다. 500 에러의 경우 대부분 애플리케이션 내부에서 발생한 예외입니다.

### 2. Mocking 및 설정 확인
Mocking이 제대로 되어 있는지, 테스트 환경 설정이 올바른지 확인합니다.

### 3. 애너테이션 및 프로파일 확인
테스트 클래스와 메서드에 필요한 애너테이션과 프로파일이 올바르게 설정되어 있는지 확인합니다.

### 4. Embedded Redis 설정 확인
Embedded Redis가 올바르게 설정되어 실행되고 있는지 확인합니다.

### 체크리스트

1. **에러 로그 확인**:
   ```java
   @Test
   public void testMyController() throws Exception {
       // ...
   }
   ```

2. **Mocking 및 설정 확인**:
   ```java
   @MockBean
   private CacheManager cacheManager;

   @MockBean
   private RedisTemplate<String, Object> redisTemplate;
   ```

3. **애너테이션 및 프로파일 확인**:
   ```java
   @ExtendWith(SpringExtension.class)
   @SpringBootTest
   @AutoConfigureMockMvc
   @ActiveProfiles("test")
   public class MyControllerTest {
       // ...
   }
   ```

4. **Embedded Redis 설정 확인**:
   Embedded Redis가 테스트 환경에서 올바르게 설정되어 있는지 확인합니다.

### Embedded Redis 예제

```java
import org.springframework.boot.test.context.TestConfiguration;
import org.springframework.context.annotation.Bean;
import redis.embedded.RedisServer;

@TestConfiguration
public class EmbeddedRedisConfig {

    private RedisServer redisServer;

    @PostConstruct
    public void startRedis() {
        redisServer = new RedisServer(6379);
        redisServer.start();
    }

    @PreDestroy
    public void stopRedis() {
        redisServer.stop();
    }
}
```

그리고 테스트 클래스에서 해당 설정을 사용합니다.

```java
@Import(EmbeddedRedisConfig.class)
public class MyControllerTest {
    // ...
}
```

### 예제: 구체적인 문제 해결

만약 특정 테스트에서만 문제가 발생한다면, 해당 테스트를 분리하여 개별적으로 실행해보고 문제가 발생하는 지점을 정확히 파악합니다.

```java
@Test
public void testSpecificScenario() throws Exception {
    // 특정 시나리오를 테스트
}
```

### 500 에러 해결
에러 로그를 통해 어떤 예외가 발생했는지 확인합니다. 발생한 예외와 관련된 코드 부분을 검토하여 문제를 해결합니다. 예를 들어, NullPointerException이 발생했다면 해당 객체가 제대로 초기화되었는지 확인합니다.

```java
public class MyControllerTest {

    @Test
    public void testMyController() throws Exception {
        mockMvc.perform(get("/api/resource/nonexistent-id"))
                .andExpect(status().isNotFound());
    }
}
```

### 종합
모든 설정을 다시 한 번 점검하고, 특히 테스트 환경에서의 설정이 프로덕션 환경과 다르지 않은지 확인합니다. 500 에러는 내부적인 문제를 의미하므로, 에러 로그를 통해 정확한 원인을 파악하고 문제를 해결해야 합니다.

만약 여전히 문제가 해결되지 않는다면, 테스트 코드와 애플리케이션 코드를 자세히 검토하여 문제가 발생하는 지점을 정확히 파악하는 것이 중요합니다. 문제가 지속된다면, 추가적인 로그 출력을 통해 더 많은 정보를 얻는 것도 도움이 될 수 있습니다.



404 에러 대신 500 에러가 발생하는 원인을 찾아 해결하려면 다음 단계를 따라야 합니다:

1. **에러 로그 확인**: 500 에러의 정확한 원인을 확인하기 위해 에러 로그를 철저히 분석합니다. 이를 통해 어떤 예외가 발생했는지, 어느 부분에서 문제가 생겼는지 알 수 있습니다.

2. **핸들러 검토**: 404 에러가 발생해야 하는 상황에서 500 에러가 발생하는 원인을 검토합니다. 이는 잘못된 컨트롤러 매핑이나 예외 처리 설정이 원인일 수 있습니다.

3. **테스트 코드 검토**: 유닛 테스트 코드에서 잘못된 설정이나 예외 처리가 원인일 수 있습니다.

### 에러 로그 확인
먼저, 테스트 실행 시 콘솔에 출력되는 에러 로그를 확인하고, 어떤 예외가 발생했는지 확인합니다. 예외의 상세 내용을 알아야 적절한 해결책을 찾을 수 있습니다.

### 핸들러 검토
컨트롤러나 예외 처리 핸들러에서 404 에러를 발생시키도록 설정되어 있는지 확인합니다.

예를 들어, 컨트롤러에서 특정 조건을 만족하지 않는 경우 404 에러를 발생시키려면 `ResponseStatusException`을 사용할 수 있습니다.

```java
import org.springframework.http.HttpStatus;
import org.springframework.web.server.ResponseStatusException;

@RestController
@RequestMapping("/api")
public class MyController {

    @GetMapping("/resource/{id}")
    public MyObject getResource(@PathVariable String id) {
        MyObject resource = myService.getResourceById(id);
        if (resource == null) {
            throw new ResponseStatusException(HttpStatus.NOT_FOUND, "Resource not found");
        }
        return resource;
    }
}
```

### 테스트 코드 검토
테스트 코드에서 404 에러를 검증하는지 확인합니다. 예를 들어, `MockMvc`를 사용하여 테스트할 수 있습니다.

```java
import static org.springframework.test.web.servlet.request.MockMvcRequestBuilders.get;
import static org.springframework.test.web.servlet.result.MockMvcResultMatchers.status;

import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.web.servlet.MockMvc;

@SpringBootTest
@AutoConfigureMockMvc
public class MyControllerTest {

    @Autowired
    private MockMvc mockMvc;

    @Test
    public void testGetResourceNotFound() throws Exception {
        mockMvc.perform(get("/api/resource/nonexistent-id"))
                .andExpect(status().isNotFound());
    }
}
```

위의 예제는 `MockMvc`를 사용하여 존재하지 않는 리소스를 요청하고, 404 상태 코드를 기대하는 테스트입니다.

### 500 에러 해결
500 에러가 발생하는 원인은 다양할 수 있습니다. 에러 로그를 확인하고, 해당 예외가 발생하는 위치와 원인을 파악한 후, 적절한 예외 처리 로직을 추가합니다.

```java
import org.springframework.web.bind.annotation.ExceptionHandler;
import org.springframework.web.bind.annotation.RestControllerAdvice;

@RestControllerAdvice
public class GlobalExceptionHandler {

    @ExceptionHandler(Exception.class)
    public ResponseEntity<String> handleAllExceptions(Exception ex) {
        // 예외 처리 로직 추가
        return new ResponseEntity<>("Internal Server Error", HttpStatus.INTERNAL_SERVER_ERROR);
    }

    @ExceptionHandler(ResourceNotFoundException.class)
    public ResponseEntity<String> handleResourceNotFoundException(ResourceNotFoundException ex) {
        return new ResponseEntity<>("Resource Not Found", HttpStatus.NOT_FOUND);
    }
}
```

위의 예제는 모든 예외를 처리하는 글로벌 예외 처리 핸들러입니다. `ResourceNotFoundException`이 발생하면 404 상태 코드를 반환하도록 설정했습니다.

위 단계를 통해 500 에러 대신 404 에러가 발생하도록 설정할 수 있습니다. 문제가 계속 발생할 경우, 에러 로그를 통해 구체적인 원인을 파악하고 해결책을 찾아야 합니다.


테스트 중에 발생하는 500 에러는 다양한 원인으로 인해 발생할 수 있습니다. 테스트 코드와 설정을 면밀히 검토하여 문제를 해결할 필요가 있습니다. 기본적으로 다음과 같은 단계를 통해 문제를 해결할 수 있습니다.

1. **에러 로그 확인**:
   테스트 중에 발생하는 500 에러의 구체적인 원인을 확인하기 위해 에러 로그를 면밀히 살펴보세요. 에러 로그에서 발생한 예외와 해당 예외가 발생한 위치를 확인할 수 있습니다.

2. **테스트 환경 설정 확인**:
   테스트 환경이 프로덕션 환경과 다를 수 있습니다. 테스트 환경에서의 설정이 올바른지 확인해야 합니다. 특히, 테스트에서 Redis를 사용하고 있는지, 그리고 Redis 설정이 올바른지 확인하세요.

3. **Mocking**:
   유닛 테스트에서는 종속성을 Mocking하는 것이 좋습니다. Spring Boot에서는 `@MockBean` 어노테이션을 사용하여 특정 빈을 Mock할 수 있습니다.

4. **테스트 프로파일**:
   테스트 시 사용할 별도의 프로파일을 설정하여 테스트 환경에 맞는 설정을 할 수 있습니다.

다음은 예제입니다:

### 1. 에러 로그 확인
먼저, 테스트 실행 시 콘솔에 출력되는 에러 로그를 확인하고, 어떤 예외가 발생했는지 확인합니다. 여기서 예외의 상세 내용을 알아야 적절한 해결책을 찾을 수 있습니다.

### 2. 테스트 환경 설정 확인
테스트 환경 설정이 올바르게 되어 있는지 확인합니다. 예를 들어, 테스트에서 Redis가 제대로 설정되어 있는지 확인합니다.

### 3. Mocking
테스트에서 Redis나 CacheManager를 Mocking합니다. 예제는 다음과 같습니다:

```java
import static org.mockito.BDDMockito.given;

import org.junit.jupiter.api.Test;
import org.mockito.InjectMocks;
import org.mockito.Mock;
import org.mockito.junit.jupiter.MockitoExtension;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.mock.mockito.MockBean;
import org.springframework.cache.CacheManager;
import org.springframework.test.context.junit.jupiter.SpringExtension;

import java.util.Optional;

@ExtendWith(MockitoExtension.class)
@ExtendWith(SpringExtension.class)
@SpringBootTest
public class MyControllerTest {

    @MockBean
    private CacheManager cacheManager;

    @MockBean
    private RedisTemplate<String, Object> redisTemplate;

    @Autowired
    private MyController myController;

    @Test
    public void testMyController() {
        // Mocking behavior
        Cache cache = mock(Cache.class);
        given(cacheManager.getCache("myCacheName")).willReturn(cache);
        given(redisTemplate.opsForValue().get("myKey")).willReturn(new MyObject());

        // Call the controller method and assert the results
        // ...
    }
}
```

### 4. 테스트 프로파일 설정
테스트 환경에 맞는 프로파일을 설정하여 테스트 시 특정 설정이 적용되도록 합니다. 예를 들어, `application-test.properties` 파일을 만들어 설정을 추가할 수 있습니다.

```properties
# application-test.properties
spring.profiles.active=test

# Redis 설정
spring.redis.host=localhost
spring.redis.port=6379
```

이제 테스트 클래스에서 `@ActiveProfiles` 어노테이션을 사용하여 `test` 프로파일을 활성화할 수 있습니다.

```java
import org.springframework.test.context.ActiveProfiles;

@ActiveProfiles("test")
public class MyControllerTest {
    // 테스트 코드
}
```

위의 방법들을 통해 테스트 환경 설정을 올바르게 하고, 필요한 경우 Mocking을 사용하여 의존성을 해결하면 500 에러를 해결할 수 있을 것입니다. 정확한 해결책은 에러 로그를 확인하여 원인을 파악한 후 적용하는 것이 중요합니다.


--------------------------
`@Primary`로 설정된 `CacheManager` 빈이 잘못 참조되는 문제로 보입니다. 이 문제를 해결하려면 순환 의존성을 피하면서도 초기화 작업을 수행할 수 있도록 초기화 로직을 변경해야 합니다. `ApplicationListener` 인터페이스를 구현하여 애플리케이션 컨텍스트 초기화 이벤트를 처리하는 방법을 사용할 수 있습니다.

다음은 `ApplicationListener`를 사용하여 초기화 작업을 수행하는 예제입니다:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.CacheManager;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.context.event.ContextRefreshedEvent;
import org.springframework.context.event.EventListener;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

@Configuration
public class CacheConfig {

    @Bean
    @Primary
    public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)) // 캐시 만료 시간 설정
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @EventListener(ContextRefreshedEvent.class)
    public void onApplicationEvent(ContextRefreshedEvent event) {
        ApplicationContext applicationContext = event.getApplicationContext();

        CacheManager cacheManager = applicationContext.getBean(CacheManager.class);
        if (cacheManager.getCache("myCacheName") != null) {
            cacheManager.getCache("myCacheName").clear(); // 특정 캐시 클리어
        }

        RedisTemplate<String, Object> redisTemplate = applicationContext.getBean(RedisTemplate.class);
        redisTemplate.delete("myKey");
    }
}
```

### 설명

1. `redisCacheManager` 빈과 `redisTemplate` 빈을 각각 정의합니다.
2. `ContextRefreshedEvent` 이벤트를 리스닝하는 메소드를 정의합니다.
3. 애플리케이션 컨텍스트 초기화가 완료되면 `onApplicationEvent` 메소드가 호출되어 `CacheManager`와 `RedisTemplate`을 사용하여 초기화 작업을 수행합니다.

이 접근 방식은 애플리케이션이 완전히 초기화된 후에 캐시 초기화 작업을 수행하므로, 순환 의존성 문제를 피할 수 있습니다. `ApplicationListener` 인터페이스를 직접 구현하는 대신, `@EventListener` 어노테이션을 사용하는 방법으로 코드를 간결하게 유지했습니다.
----------

로그를 통해 `redisCacheManager`와 `defaultCacheManager`가 생성되는 순서를 확인한 경우, 이것이 꼭 문제를 의미하는 것은 아닙니다. Spring Boot는 컨텍스트 초기화 과정에서 모든 빈을 생성하지만, 어떤 빈을 실제로 사용할지는 빈의 우선순위와 의존성에 따라 결정됩니다.

`@Primary` 어노테이션을 통해 우선순위를 설정했기 때문에, `CacheManager`가 필요할 때 `redisCacheManager`가 사용됩니다. 생성 순서가 아니라, 실제로 어떤 빈이 사용되는지가 중요한 것입니다. 

### 확인 방법

1. **빈 생성 확인**: 로그를 통해 두 빈이 모두 생성되고 있다는 것을 확인합니다.
2. **의존성 주입 확인**: 실제로 `CacheManager` 빈이 필요한 클래스에 `@Autowired` 또는 생성자 주입을 통해 어떤 빈이 주입되는지 확인합니다.
3. **애플리케이션 동작 확인**: 애플리케이션이 정상적으로 동작하는지, 특히 캐시와 관련된 기능이 기대한 대로 동작하는지 확인합니다.

아래는 간단한 예제입니다. `CacheManager`를 주입받는 서비스 클래스와, 해당 클래스에서 실제로 어떤 빈이 주입되는지 확인하는 방법입니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Service;

@Service
public class MyService {

    private final CacheManager cacheManager;

    @Autowired
    public MyService(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    public void performCacheOperation() {
        System.out.println("Using CacheManager: " + cacheManager.getClass().getName());
        // 캐시 관련 작업 수행
    }
}
```

애플리케이션을 실행한 후, `MyService.performCacheOperation()` 메소드를 호출하면 실제로 어떤 `CacheManager`가 사용되고 있는지 확인할 수 있습니다.

### 결론

로그에서 빈 생성 순서가 `redisCacheManager`가 먼저, `defaultCacheManager`가 나중에 찍히는 것은 문제가 아닙니다. 중요한 것은 실제로 `CacheManager`를 필요로 하는 곳에서 `redisCacheManager`가 사용되는지 확인하는 것입니다. `@Primary` 어노테이션을 통해 우선순위가 설정되어 있기 때문에, `CacheManager`를 주입받는 곳에서는 `redisCacheManager`가 기본적으로 사용됩니다.

------------------------
캐시 초기화를 위해 `CacheManager`를 직접 참조하는 경우 무한 루프 문제가 발생할 수 있습니다. 이를 피하기 위해, `CacheManager` 빈을 정의할 때 함께 초기화를 수행할 수 있습니다. 

Spring Boot에서 `CacheManager`를 정의할 때 초기화 작업을 함께 수행하도록 설정할 수 있습니다. 다음은 `RedisCacheManager`를 정의할 때 특정 캐시 항목을 삭제하는 방법의 예입니다:

### 방법: CacheManager 빈 정의 시 초기화 로직 추가

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.boot.autoconfigure.cache.CacheProperties;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.repository.configuration.EnableRedisRepositories;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.Arrays;
import java.util.stream.Collectors;

@Configuration
@EnableCaching
@EnableRedisRepositories
public class CacheConfig {

    @Value("${spring.redis.host:localhost}")
    private String redisHost;

    @Value("${spring.redis.port:6379}")
    private int redisPort;

    @Value("${spring.redis.password:}")
    private String redisPassword;

    @Value("${spring.redis.database:0}")
    private int redisDatabase;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(redisHost);
        config.setPort(redisPort);
        if (!redisPassword.isEmpty()) {
            config.setPassword(redisPassword);
        }
        config.setDatabase(redisDatabase);
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)) // 캐시 만료 시간 설정
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        RedisCacheManager cacheManager = RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();

        // 초기화 로직
        if (cacheManager.getCache("myCacheName") != null) {
            cacheManager.getCache("myCacheName").clear(); // 특정 캐시 클리어
        }

        // RedisTemplate을 사용하여 특정 키 삭제
        redisTemplate.delete("myKey");

        return cacheManager;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public KeyGenerator customKeyGenerator() {
        return (target, method, params) -> {
            // 커스텀 키 생성 로직
            return method.getName() + "_" + Arrays.stream(params).map(String::valueOf).collect(Collectors.joining("_"));
        };
    }
}
```

위와 같이 `cacheManager` 빈을 정의할 때 초기화 로직을 추가하여 특정 캐시 항목을 삭제할 수 있습니다. 이 방법을 사용하면 별도의 `CacheInitializer` 빈을 만들 필요 없이 `CacheManager` 빈 생성 시 필요한 초기화를 수행할 수 있습니다.

이 설정을 통해 애플리케이션이 시작될 때 지정된 캐시 항목을 초기화하거나 삭제할 수 있습니다. `clear()` 메서드를 사용하여 특정 캐시 전체를 삭제할 수 있고, `evict()` 메서드를 사용하여 특정 키의 항목만 삭제할 수 있습니다.
-> 안됨

`RedisTemplate`와 `CacheManager`가 서로를 참조하여 순환 의존성(circular dependency) 문제가 발생하는 경우가 있습니다. 이를 해결하려면, 빈 정의를 나누거나, `ApplicationContext`를 사용하여 빈을 초기화하는 방식을 사용할 수 있습니다.

다음은 순환 의존성 문제를 피하면서 `CacheManager`와 `RedisTemplate`을 사용하여 초기화 작업을 수행하는 예제입니다:

1. `CacheManager`를 정의하는 빈과 `RedisTemplate`을 정의하는 빈을 각각 설정합니다.
2. `ApplicationContext`를 사용하여 초기화 작업을 분리합니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.ApplicationContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;

@Configuration
public class CacheConfig {

    @Autowired
    private ApplicationContext applicationContext;

    @Bean
    @Primary
    public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)) // 캐시 만료 시간 설정
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        RedisCacheManager cacheManager = RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();

        return cacheManager;
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public InitializingBean cacheInitializer() {
        return () -> {
            CacheManager cacheManager = applicationContext.getBean(CacheManager.class);
            if (cacheManager.getCache("myCacheName") != null) {
                cacheManager.getCache("myCacheName").clear(); // 특정 캐시 클리어
            }

            RedisTemplate<String, Object> redisTemplate = applicationContext.getBean(RedisTemplate.class);
            redisTemplate.delete("myKey");
        };
    }
}
```

### 설명

1. `redisCacheManager` 빈과 `redisTemplate` 빈을 각각 정의합니다.
2. `InitializingBean` 인터페이스를 사용하여 초기화 작업을 수행하는 별도의 빈을 정의합니다.
3. `cacheInitializer` 빈이 생성될 때 `ApplicationContext`를 통해 `CacheManager`와 `RedisTemplate`을 가져와 초기화 작업을 수행합니다.

이렇게 하면 순환 의존성 문제를 피하면서, `CacheManager`와 `RedisTemplate`를 초기화 작업에 사용할 수 있습니다.

--------------------
스프링부트 애플리케이션 시작 시 특정 캐시된 내용을 삭제하려면, 애플리케이션이 시작될 때 실행되는 메서드를 작성하여 원하는 캐시를 삭제할 수 있습니다. 이를 위해 `@PostConstruct` 어노테이션을 사용하거나, `ApplicationRunner` 또는 `CommandLineRunner` 인터페이스를 구현할 수 있습니다.

이방법 안됨 -> 무한 루프

### 방법 1: `@PostConstruct` 사용
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Component;

import javax.annotation.PostConstruct;

@Component
public class CacheInitializer {

    @Autowired
    private CacheManager cacheManager;

    @PostConstruct
    public void clearCacheOnStartup() {
        // 특정 캐시 이름으로 캐시 삭제
        cacheManager.getCache("myCacheName").clear();
    }
}
```

### 방법 2: `ApplicationRunner` 사용
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationRunner;
import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CacheConfig {

    @Autowired
    private CacheManager cacheManager;

    @Bean
    public ApplicationRunner clearCacheOnStartup() {
        return args -> {
            // 특정 캐시 이름으로 캐시 삭제
            cacheManager.getCache("myCacheName").clear();
        };
    }
}
```

### 방법 3: `CommandLineRunner` 사용
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.CommandLineRunner;
import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CacheConfig {

    @Autowired
    private CacheManager cacheManager;

    @Bean
    public CommandLineRunner clearCacheOnStartup() {
        return args -> {
            // 특정 캐시 이름으로 캐시 삭제
            cacheManager.getCache("myCacheName").clear();
        };
    }
}
```

### 특정 키만 삭제하는 방법
만약 특정 캐시의 모든 항목이 아니라 특정 키만 삭제하고 싶다면, 아래와 같이 `evict` 메서드를 사용할 수 있습니다:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.ApplicationRunner;
import org.springframework.cache.CacheManager;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class CacheConfig {

    @Autowired
    private CacheManager cacheManager;

    @Bean
    public ApplicationRunner clearSpecificCacheEntriesOnStartup() {
        return args -> {
            // 특정 캐시에서 특정 키만 삭제
            cacheManager.getCache("myCacheName").evict("myKey");
        };
    }
}
```

위와 같이 설정을 통해 스프링부트 애플리케이션이 시작될 때 원하는 캐시 항목을 삭제할 수 있습니다. `CacheManager`를 통해 캐시를 관리하고, `clear`, `evict` 등의 메서드를 사용하여 특정 캐시 또는 특정 키의 캐시를 삭제할 수 있습니다.


--------------------------
Spring Boot 애플리케이션이 Redis 연결 문제로 인해 시작되지 않는 경우, Redis 연결 실패 시에도 애플리케이션이 계속 시작되도록 설정할 수 있습니다. 이를 위해 다음과 같은 방법을 사용할 수 있습니다:

1. **Redis CacheManager가 선택적으로 빈 등록되도록 설정**:
   - Redis가 사용 가능하지 않을 때 기본 CacheManager를 등록하여 Redis 연결 실패 시에도 애플리케이션이 계속 실행되도록 합니다.

2. **ConditionalOnMissingBean**과 **ConditionalOnProperty** 어노테이션 사용**:
   - 이를 통해 Redis 설정이 존재하지 않거나 Redis 연결 실패 시 기본 CacheManager를 사용하도록 할 수 있습니다.

다음은 Redis 연결 실패 시 기본 CacheManager를 사용하도록 설정하는 예제입니다:

### `RedisConfig.java`

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.concurrent.ConcurrentMapCacheManager;
import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.context.annotation.Primary;
import org.springframework.context.annotation.Profile;
import org.springframework.context.annotation.ConditionalOnMissingBean;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.repository.configuration.EnableRedisRepositories;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.Arrays;
import java.util.stream.Collectors;

@Configuration
@EnableCaching
@EnableRedisRepositories
public class RedisConfig {

    @Value("${spring.redis.host:localhost}")
    private String redisHost;

    @Value("${spring.redis.port:6379}")
    private int redisPort;

    @Value("${spring.redis.password:}")
    private String redisPassword;

    @Value("${spring.redis.database:0}")
    private int redisDatabase;

    @Bean
    @ConditionalOnMissingBean(RedisConnectionFactory.class)
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(redisHost);
        config.setPort(redisPort);
        if (!redisPassword.isEmpty()) {
            config.setPassword(redisPassword);
        }
        config.setDatabase(redisDatabase);
        return new LettuceConnectionFactory(config);
    }

    @Bean
    @Primary
    @ConditionalOnMissingBean(CacheManager.class)
    public CacheManager defaultCacheManager() {
        return new ConcurrentMapCacheManager(); // 기본 CacheManager
    }

    @Bean
    @Profile("redis")
    @ConditionalOnProperty(name = "spring.cache.type", havingValue = "redis", matchIfMissing = true)
    public CacheManager redisCacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)) // 캐시 만료 시간 설정
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();
    }

    @Bean
    @ConditionalOnMissingBean(RedisTemplate.class)
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public KeyGenerator customKeyGenerator() {
        return (target, method, params) -> {
            // 커스텀 키 생성 로직
            return method.getName() + "_" + Arrays.stream(params).map(String::valueOf).collect(Collectors.joining("_"));
        };
    }
}
```

### `application.properties` 설정

```properties
spring.cache.type=redis
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=your_password
spring.redis.database=0
```

### 주의 사항

위의 설정에서는 `ConditionalOnMissingBean`과 `ConditionalOnProperty` 어노테이션을 사용하여 Redis CacheManager가 없거나 Redis 연결에 실패할 경우 기본 CacheManager(ConcurrentMapCacheManager)를 사용하도록 설정합니다. 이를 통해 Redis 연결 문제로 인해 애플리케이션이 시작되지 않는 문제를 회피할 수 있습니다.

이와 같이 설정을 변경한 후 애플리케이션을 다시 빌드하고 실행하여 문제가 해결되는지 확인해보세요.

--------------------------

에러 메시지를 보면 `java.lang.NoSuchMethodError: 'org.springframework.http.HttpStatus org.springframework.http.ResponseEntity.getStatusCode()'`와 관련된 문제가 발생하고 있습니다. 이는 사용 중인 Spring 라이브러리 버전이 호환되지 않는 경우에 발생할 수 있는 문제입니다.

이 문제를 해결하려면 다음 사항들을 점검하고 수정해보세요:

1. **Spring 라이브러리 버전 호환성 확인**: `ResponseEntity.getStatusCode()` 메서드는 Spring 5.3.x 버전에서는 `HttpStatus`를 반환하지만, 이후 버전에서는 `HttpStatusCode`를 반환하는 등 변경이 있을 수 있습니다. 사용 중인 Spring Boot 버전과 다른 Spring 라이브러리 버전 간의 호환성을 확인하십시오.

2. **Spring Boot와 의존성 정리**: `pom.xml` 또는 `build.gradle` 파일에서 사용 중인 Spring Boot 버전과 관련된 모든 의존성이 호환되는지 확인하십시오.

### Spring Boot 프로젝트 의존성 정리 예제 (Maven)

#### `pom.xml`

```xml
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>2.7.0</version> <!-- 사용 중인 Spring Boot 버전에 맞게 변경 -->
    <relativePath/> <!-- lookup parent from repository -->
</parent>

<dependencies>
    <!-- Spring Boot Starter Web -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>

    <!-- Spring Boot Starter Data Redis -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>

    <!-- Lettuce Core -->
    <dependency>
        <groupId>io.lettuce.core</groupId>
        <artifactId>lettuce-core</artifactId>
    </dependency>

    <!-- Other dependencies -->

    <!-- Test dependencies -->
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-test</artifactId>
        <scope>test</scope>
    </dependency>
</dependencies>

<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-dependencies</artifactId>
            <version>2.7.0</version> <!-- 사용 중인 Spring Boot 버전에 맞게 변경 -->
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>
```

### Spring Boot 프로젝트 의존성 정리 예제 (Gradle)

#### `build.gradle`

```groovy
plugins {
    id 'org.springframework.boot' version '2.7.0' // 사용 중인 Spring Boot 버전에 맞게 변경
    id 'io.spring.dependency-management' version '1.0.11.RELEASE'
    id 'java'
}

group = 'com.example'
version = '0.0.1-SNAPSHOT'
sourceCompatibility = '17'

repositories {
    mavenCentral()
}

dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-web'
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'io.lettuce.core:lettuce-core'

    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

dependencyManagement {
    imports {
        mavenBom "org.springframework.boot:spring-boot-dependencies:2.7.0" // 사용 중인 Spring Boot 버전에 맞게 변경
    }
}
```

3. **Spring Boot 및 Spring Framework 버전 일치**: `spring-boot-starter-parent` 또는 `spring-boot-dependencies`를 사용하는 경우 버전을 일치시켜 호환성을 유지하는 것이 좋습니다. 

위와 같이 설정을 점검한 후 프로젝트를 다시 빌드하고 실행하여 문제가 해결되는지 확인하십시오. 

필요한 경우, 특정 버전 호환성을 확인하는데 도움이 되는 [Spring Initializr](https://start.spring.io/)를 활용하여 기본 설정을 참고할 수도 있습니다.

----------------------------
현재 설정된 Redis 서버의 정보를 `application.properties` 또는 `application.yml` 파일에 추가하고, 이를 `LettuceConnectionFactory`에 반영하도록 설정 클래스를 변경하면 됩니다. 이 방법은 Spring Boot에서 제공하는 `RedisStandaloneConfiguration` 클래스를 사용하여 레디스 서버의 호스트, 포트, 비밀번호 등의 설정을 지정할 수 있습니다.

### 1. application.properties 또는 application.yml 설정

#### application.properties
```properties
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=your_password   # 필요한 경우
spring.redis.database=0
```

#### application.yml
```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: your_password   # 필요한 경우
    database: 0
```

### 2. Redis Configuration 클래스 변경

이제 설정 클래스를 변경하여 `RedisStandaloneConfiguration`을 사용하도록 합니다.

```java
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.RedisStandaloneConfiguration;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.repository.configuration.EnableRedisRepositories;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.Arrays;
import java.util.stream.Collectors;

@Configuration
@EnableCaching
@EnableRedisRepositories
public class RedisConfig {

    @Value("${spring.redis.host}")
    private String redisHost;

    @Value("${spring.redis.port}")
    private int redisPort;

    @Value("${spring.redis.password}")
    private String redisPassword;

    @Value("${spring.redis.database}")
    private int redisDatabase;

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        RedisStandaloneConfiguration config = new RedisStandaloneConfiguration();
        config.setHostName(redisHost);
        config.setPort(redisPort);
        config.setPassword(redisPassword);
        config.setDatabase(redisDatabase);
        return new LettuceConnectionFactory(config);
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)) // 캐시 만료 시간 설정
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public KeyGenerator customKeyGenerator() {
        return (target, method, params) -> {
            // 커스텀 키 생성 로직
            return method.getName() + "_" + Arrays.stream(params).map(String::valueOf).collect(Collectors.joining("_"));
        };
    }
}
```

이 설정은 Spring Boot의 `@Value` 어노테이션을 사용하여 `application.properties` 또는 `application.yml`에 설정된 값을 읽어와서 `RedisStandaloneConfiguration`에 적용합니다. 이를 통해 Redis 서버의 호스트, 포트, 비밀번호 등의 정보를 지정할 수 있습니다.

----------------------------

Spring Boot에서 JDK 17과 JDK 21을 사용하는 설정은 기본적으로 동일합니다. JDK 버전 간의 차이점은 주로 JVM 자체의 변경 사항에 있습니다. Spring Boot의 설정이나 Redis 설정에는 큰 차이가 없습니다. 

아래에 설명된 설정은 JDK 17과 JDK 21 모두에서 작동합니다. 이 설정에서는 Redis Cache Manager와 RedisTemplate을 사용하여 Redis 캐싱을 설정합니다.

### 1. Maven 또는 Gradle 의존성 추가

먼저, `pom.xml` 또는 `build.gradle` 파일에 필요한 의존성을 추가합니다.

#### Maven
```xml
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    <dependency>
        <groupId>io.lettuce.core</groupId>
        <artifactId>lettuce-core</artifactId>
    </dependency>
</dependencies>
```

#### Gradle
```groovy
dependencies {
    implementation 'org.springframework.boot:spring-boot-starter-data-redis'
    implementation 'io.lettuce.core:lettuce-core'
}
```

### 2. application.properties 또는 application.yml 설정

Redis 연결 설정을 `application.properties` 또는 `application.yml` 파일에 추가합니다.

#### application.properties
```properties
spring.redis.host=localhost
spring.redis.port=6379
spring.redis.password=your_password   # 필요한 경우
spring.redis.database=0
```

#### application.yml
```yaml
spring:
  redis:
    host: localhost
    port: 6379
    password: your_password   # 필요한 경우
    database: 0
```

### 3. Redis Configuration 클래스 작성

RedisCacheManager와 RedisTemplate을 설정하는 `@Configuration` 클래스를 작성합니다.

```java
import org.springframework.cache.CacheManager;
import org.springframework.cache.annotation.EnableCaching;
import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.data.redis.cache.RedisCacheConfiguration;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.connection.lettuce.LettuceConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.data.redis.repository.configuration.EnableRedisRepositories;
import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
import org.springframework.data.redis.serializer.RedisSerializationContext;
import org.springframework.data.redis.serializer.StringRedisSerializer;

import java.time.Duration;
import java.util.Arrays;
import java.util.stream.Collectors;

@Configuration
@EnableCaching
@EnableRedisRepositories
public class RedisConfig {

    @Bean
    public RedisConnectionFactory redisConnectionFactory() {
        return new LettuceConnectionFactory();
    }

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        RedisCacheConfiguration cacheConfiguration = RedisCacheConfiguration.defaultCacheConfig()
                .entryTtl(Duration.ofHours(1)) // 캐시 만료 시간 설정
                .disableCachingNullValues()
                .serializeKeysWith(RedisSerializationContext.SerializationPair.fromSerializer(new StringRedisSerializer()))
                .serializeValuesWith(RedisSerializationContext.SerializationPair.fromSerializer(new GenericJackson2JsonRedisSerializer()));

        return RedisCacheManager.builder(redisConnectionFactory)
                .cacheDefaults(cacheConfiguration)
                .build();
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> template = new RedisTemplate<>();
        template.setConnectionFactory(redisConnectionFactory);
        template.setKeySerializer(new StringRedisSerializer());
        template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
        return template;
    }

    @Bean
    public KeyGenerator customKeyGenerator() {
        return (target, method, params) -> {
            // 커스텀 키 생성 로직
            return method.getName() + "_" + Arrays.stream(params).map(String::valueOf).collect(Collectors.joining("_"));
        };
    }
}
```

### 4. 캐시 사용 및 RedisTemplate 사용

캐시를 적용하고자 하는 메소드에 `@Cacheable`, `@CachePut`, `@CacheEvict` 어노테이션을 사용하고, RedisTemplate을 이용한 예제도 함께 작성합니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.stereotype.Service;

@Service
public class MyService {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Cacheable(value = "myCache", keyGenerator = "customKeyGenerator")
    public String getData(String param) {
        // 실제 데이터 로드 로직
        return "Data for " + param;
    }

    public void saveDataToRedis(String key, Object value) {
        redisTemplate.opsForValue().set(key, value);
    }

    public Object getDataFromRedis(String key) {
        return redisTemplate.opsForValue().get(key);
    }
}
```

이렇게 하면 Spring Boot에서 Lettuce를 이용한 Redis Cache Manager 설정과 RedisTemplate을 이용한 Redis 상호작용이 완료됩니다. 이 설정은 JDK 17과 JDK 21 모두에서 작동합니다.

--------------------------------
맥북에 로컬 환경에서 docker 에 redis 7.2.5 가 있는데 intellij 앱에서 인식을 못하는 거 같아. 접근이 안되거나

로컬에 설치된 Docker 컨테이너에서 실행 중인 Redis 7.2.5에 접근하는 데 문제가 있는 경우, 몇 가지 기본적인 문제 해결 단계를 통해 원인을 파악하고 해결할 수 있습니다. 아래에 문제 해결을 위한 몇 가지 방법을 안내해 드리겠습니다.

### 1. Redis Docker 컨테이너 상태 확인
먼저 Redis Docker 컨테이너가 정상적으로 실행 중인지 확인해야 합니다. 터미널에서 다음 명령어를 사용해 컨테이너 상태를 확인하세요.

```sh
docker ps
```

이 명령어를 통해 Redis 컨테이너가 실행 중인지, 포트가 올바르게 매핑되었는지 확인할 수 있습니다. 예시 출력은 다음과 같습니다.

```
CONTAINER ID   IMAGE        COMMAND                  CREATED          STATUS          PORTS                    NAMES
abc123def456   redis:7.2.5  "docker-entrypoint.s…"   2 minutes ago    Up 2 minutes    0.0.0.0:6379->6379/tcp   redis-container
```

여기서 `PORTS` 항목이 `0.0.0.0:6379->6379/tcp`로 되어 있는지 확인합니다. 이는 호스트의 6379 포트가 컨테이너의 6379 포트로 매핑되었음을 의미합니다.

### 2. Docker 컨테이너 포트 매핑 확인
Docker 컨테이너를 실행할 때 포트 매핑이 올바르게 설정되었는지 확인하세요. 올바른 설정 예시는 다음과 같습니다.

```sh
docker run -d --name redis-container -p 6379:6379 redis:7.2.5
```

### 3. Redis 컨테이너 접근 테스트
터미널에서 `redis-cli`를 사용하여 Docker 컨테이너에 접근할 수 있는지 테스트합니다.

```sh
redis-cli -h localhost -p 6379
```

정상적으로 연결되면 `redis-cli` 프롬프트가 나타나야 합니다. 이 단계가 성공한다면 Docker와 Redis 설정에는 문제가 없을 가능성이 큽니다.

### 4. IntelliJ 설정 확인
IntelliJ에서 Redis에 접근하기 위해 Redis 클라이언트 플러그인 또는 DataGrip과 같은 도구를 사용합니다. IntelliJ 설정이 올바른지 확인하세요.

#### Redis 클라이언트 플러그인 사용법
1. **플러그인 설치**:
   - IntelliJ에서 `File > Settings > Plugins`로 이동합니다.
   - `Redis`를 검색하고 Redis 플러그인을 설치합니다.
   - IntelliJ를 재시작합니다.

2. **Redis 서버 설정**:
   - IntelliJ의 오른쪽 하단에 `Database` 탭을 클릭합니다.
   - `+` 버튼을 클릭하고 `Data Source > Redis`를 선택합니다.
   - `Host`를 `localhost`, `Port`를 `6379`로 설정합니다.
   - `Test Connection` 버튼을 클릭하여 연결이 정상적인지 확인합니다.

### 5. 방화벽 및 네트워크 설정 확인
로컬 방화벽 또는 네트워크 설정이 Docker 컨테이너에 접근하는 것을 차단하지 않는지 확인합니다. 이를 위해 로컬 방화벽 규칙을 검토하거나 일시적으로 비활성화하여 문제가 해결되는지 확인할 수 있습니다.

### 요약
1. Docker 컨테이너 상태와 포트 매핑을 확인합니다.
2. `redis-cli`를 사용하여 Redis 컨테이너에 접근할 수 있는지 테스트합니다.
3. IntelliJ에서 Redis 클라이언트 플러그인을 사용하여 설정이 올바른지 확인합니다.
4. 방화벽 및 네트워크 설정을 검토합니다.

위 단계를 통해 문제를 해결할 수 있기를 바랍니다. 그래도 문제가 지속된다면 추가적인 로그나 오류 메시지를 제공해 주시면 더 구체적으로 도와드릴 수 있습니다.


## IntelliJ에서 Docker 내 Redis 7.2.5 접근 문제 해결

맥북 로컬 환경에서 Docker로 실행한 Redis 7.2.5에 IntelliJ에서 접근하지 못하는 문제를 해결하는 방법을 알려드리겠습니다. 몇 가지 가능한 원인과 해결 방법을 단계별로 살펴보겠습니다.

**1. Docker 연결 확인:**

* **Docker Engine 실행 확인:**
    * 터미널을 열고 `docker ps` 명령어를 실행하여 Docker Engine이 실행 중인지 확인합니다. 실행 중이 아니면 `docker start` 명령어로 실행합니다.
* **Docker 내 Redis 컨테이너 확인:**
    * `docker ps` 명령어를 사용하여 Redis 컨테이너가 실행 중인지 확인합니다. 컨테이너 이름은 Docker 실행 시 지정한 이름을 사용합니다.
    * 컨테이너가 실행 중이지 않으면 `docker run` 명령어를 사용하여 Redis 컨테이너를 실행합니다. 예시:

    ```bash
    docker run -d --name my-redis -p 6379:6379 redis:7.2.5
    ```

* **네트워크 연결 확인:**
    * Docker 내 Redis 컨테이너가 `bridge` 네트워크 모드로 실행되었는지 확인합니다. `docker inspect` 명령어를 사용하여 컨테이너 정보를 확인하고, `NetworkMode` 값이 `bridge`인지 확인합니다.
    * 만약 `host` 모드로 실행된 경우, `docker network connect` 명령어를 사용하여 컨테이너를 브리지 네트워크에 연결해야 합니다.

**2. IntelliJ 설정 확인:**

* **IntelliJ IDEA 버전 확인:**
    * IntelliJ IDEA 버전이 최신인지 확인합니다. 오래된 버전에서는 Docker 연결 관련 기능이 제대로 작동하지 않을 수 있습니다.
* **IntelliJ IDEA에서 Docker 플러그인 설치:**
    * 아직 설치하지 않았다면, IntelliJ IDEA에 Docker 플러그인을 설치합니다.
    * `File` > `Settings` > `Plugins` 메뉴에서 `Docker` 플러그인을 검색하여 설치합니다.
* **IntelliJ IDEA에서 Docker 설정:**
    * `File` > `Settings` > `Build, Execution, Deployment` > `Docker` 메뉴에서 Docker 설정을 확인합니다.
    * Docker Engine 연결 정보 (Docker Host, Docker Machine)가 정확하게 설정되어 있는지 확인합니다.

**3. Redis 접근 방법 확인:**

* **IntelliJ IDEA에서 Redis 데이터베이스 탐색기 사용:**
    * IntelliJ IDEA에서 `Database` > `Data Sources` 메뉴로 이동하여 `+` 버튼을 클릭합니다.
    * `NoSQL` > `Redis`를 선택하고, Redis 서버 정보 (IP 주소, 포트, 인증 정보)를 입력합니다.
    * 연결 테스트를 수행하여 Redis 서버에 정상적으로 접근할 수 있는지 확인합니다.

* **코드에서 Redis 라이브러리 사용:**
    * 프로젝트에 Jedis, Lettuce와 같은 Redis Java 라이브러리를 추가합니다.
    * 코드에서 라이브러리를 사용하여 Redis 서버에 연결하고 데이터를 삽입/조회/삭제 등의 작업을 수행합니다.

**4. 추가 확인 사항:**

* **방화벽 설정:**
    * 방화벽 설정이 Redis 서버 포트 (기본적으로 6379)를 차단하지 않는지 확인합니다.
* **포트 충돌:**
    * 다른 응용 프로그램이 이미 6379 포트를 사용하지 않는지 확인합니다.
* **로그 확인:**
    * Docker 컨테이너 로그와 IntelliJ IDEA 로그를 확인하여 오류 메시지가 있는지 확인합니다.

위의 단계를 차근차근 확인하며 문제를 해결해나가세요. 

**참고:**

* Docker 버전, Redis 버전, IntelliJ IDEA 버전 등을 명시해주시면 문제 해결에 도움이 될 수 있습니다.
* 오류 메시지가 있다면 함께 공유해주세요.
* 맥북 버전, 운영 체제 버전 등 추가적인 환경 정보도 도움이 될 수 있습니다.



----------------------------
org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'appIssueController': Unsatisfied dependency expressed through field 'cacheManager'; nested exception is org.springframework.beans.factory.NoSuchBeanDefinitionException: No qualifying bean of type 'org.springframework.cache.CacheManager' available: expected at least 1 bean which qualifies as autowire candidate. Dependency annotations: {@org.springframework.beans.factory.annotation.Autowired(required=true)} 가 나오는데 redis server를 인식 못하는 거 같아. 어떡하지? 한글로

`org.springframework.beans.factory.UnsatisfiedDependencyException` 예외가 발생하고, `NoSuchBeanDefinitionException`이 발생하는 경우, 이는 Spring이 `CacheManager` 타입의 빈을 찾을 수 없기 때문입니다. 이 문제는 일반적으로 Redis 서버가 제대로 설정되지 않았거나, Spring Boot의 캐시 설정이 올바르게 구성되지 않았을 때 발생합니다.

이 문제를 해결하기 위해 다음 단계를 따라 해보세요:

1. **의존성 추가**:
   먼저, `spring-boot-starter-data-redis`와 관련된 의존성을 `pom.xml` 또는 `build.gradle`에 추가해야 합니다. `pom.xml` 예시는 다음과 같습니다:

   ```xml
   <dependency>
       <groupId>org.springframework.boot</groupId>
       <artifactId>spring-boot-starter-data-redis</artifactId>
   </dependency>
   <dependency>
       <groupId>io.lettuce.core</groupId>
       <artifactId>lettuce-core</artifactId>
   </dependency>
   ```

2. **Redis 설정**:
   `application.properties` 또는 `application.yml` 파일에 Redis 서버 연결 정보를 추가합니다. 예를 들어, `application.properties`는 다음과 같습니다:

   ```properties
   spring.redis.host=localhost
   spring.redis.port=6379
   spring.cache.type=redis
   ```

3. **CacheManager 설정**:
   `RedisCacheManager`를 수동으로 설정하거나, `@EnableCaching` 어노테이션을 추가하여 캐시를 활성화합니다. 예제 코드:

   ```java
   import org.springframework.cache.annotation.EnableCaching;
   import org.springframework.context.annotation.Bean;
   import org.springframework.context.annotation.Configuration;
   import org.springframework.data.redis.connection.RedisConnectionFactory;
   import org.springframework.data.redis.core.RedisTemplate;
   import org.springframework.data.redis.cache.RedisCacheManager;
   import org.springframework.data.redis.serializer.GenericJackson2JsonRedisSerializer;
   import org.springframework.data.redis.serializer.StringRedisSerializer;

   @Configuration
   @EnableCaching
   public class RedisConfig {

       @Bean
       public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory connectionFactory) {
           RedisTemplate<String, Object> template = new RedisTemplate<>();
           template.setConnectionFactory(connectionFactory);
           template.setKeySerializer(new StringRedisSerializer());
           template.setValueSerializer(new GenericJackson2JsonRedisSerializer());
           return template;
       }

       @Bean
       public RedisCacheManager cacheManager(RedisConnectionFactory connectionFactory) {
           RedisCacheManager.RedisCacheManagerBuilder builder = RedisCacheManager.RedisCacheManagerBuilder.fromConnectionFactory(connectionFactory);
           builder.cacheDefaults(RedisCacheConfiguration.defaultCacheConfig()
               .entryTtl(Duration.ofMinutes(10)) // 캐시 만료 시간 설정
               .disableCachingNullValues());
           return builder.build();
       }
   }
   ```

4. **테스트 환경 설정**:
   테스트에서 Embedded Redis를 사용하려면, `spring-boot-starter-data-redis`와 함께 `testcontainers`를 사용하여 Redis 인스턴스를 실행할 수 있습니다. `pom.xml`에 다음 의존성을 추가합니다:

   ```xml
   <dependency>
       <groupId>org.testcontainers</groupId>
       <artifactId>testcontainers</artifactId>
       <version>1.16.0</version>
       <scope>test</scope>
   </dependency>
   <dependency>
       <groupId>org.testcontainers</groupId>
       <artifactId>mysql</artifactId>
       <version>1.16.0</version>
       <scope>test</scope>
   </dependency>
   ```

   그리고 테스트 클래스에서 `Testcontainers`를 사용하여 Embedded Redis를 설정합니다:

   ```java
   import org.junit.jupiter.api.Test;
   import org.springframework.beans.factory.annotation.Autowired;
   import org.springframework.boot.test.context.SpringBootTest;
   import org.springframework.cache.CacheManager;
   import org.springframework.data.redis.core.RedisTemplate;
   import org.springframework.test.context.ActiveProfiles;
   import org.testcontainers.containers.GenericContainer;
   import org.testcontainers.junit.jupiter.Container;
   import org.testcontainers.junit.jupiter.Testcontainers;

   @SpringBootTest
   @ActiveProfiles("test")
   @Testcontainers
   public class YourRedisTest {

       @Container
       private static final GenericContainer<?> redisContainer = new GenericContainer<>("redis:latest")
           .withExposedPorts(6379);

       @Autowired
       private CacheManager cacheManager;

       @Autowired
       private RedisTemplate<String, Object> redisTemplate;

       @Test
       public void testRedisCache() {
           // 캐시 테스트 로직 작성
           String key = "testKey";
           String value = "testValue";

           redisTemplate.opsForValue().set(key, value);
           Object cachedValue = redisTemplate.opsForValue().get(key);

           assertEquals(value, cachedValue);
       }
   }
   ```

이 설정을 통해 Redis 서버가 제대로 인식되고, 캐시 관련 빈이 정상적으로 생성될 것입니다. 만약 여전히 문제가 발생한다면, 로그를 확인하고 Redis 연결 설정이 올바른지 재검토하시기 바랍니다.

## Spring Boot에서 `org.springframework.beans.factory.UnsatisfiedDependencyException: Error creating bean with name 'appIssueController'` 오류 해결

데모 내용을 바탕으로 분석해보면, `appIssueController` 클래스에서 `CacheManager` 타입의 의존성을 가지고 있지만, 스프링 컨테이너에서 해당 타입의 Bean을 찾지 못해 발생하는 오류 같습니다. 이는 Redis 서버 설정 문제일 가능성이 높습니다.

해결 방법은 다음과 같습니다.

**1. Redis 설정 확인**

* **application.yml 설정 확인:**
    * `application.yml` 파일에 Redis 서버 정보 (host, port)가 정확하게 설정되어 있는지 확인하세요.
    * 예시:
    ```yaml
    spring:
      cache:
        type: redis  # 캐시 타입을 Redis로 설정
        cache-names: myCache  # 캐싱 적용할 캐시 이름 설정 (필요에 따라 추가)
        redis:
          host: localhost  # Redis 서버 주소
          port: 6379  # Redis 서버 포트
    ```

* **Redis 서버 실행 확인:**
    * 로컬 Redis 서버가 실행 중인지 확인하세요.
    * 별도의 Redis 서버를 사용하는 경우, 연결 및 설정이 정확한지 확인하세요.

**2. Spring Cache 라이브러리 확인**

프로젝트에 Spring Cache 라이브러리가 의존 관계로 포함되어 있는지 확인하세요. 의존 관계 관리 도구 (Maven, Gradle)에 따라 설정 방법이 다를 수 있습니다.

**3. `@EnableCaching` 어노테이션 사용**

Spring Cache 기능을 활성화하기 위해 메인 클래스 (`@SpringBootApplication`이 선언된 클래스) 또는 설정 클래스에 `@EnableCaching` 어노테이션을 추가하세요.

**4. 캐시 관리자 Bean 직접 설정 (선택적)**

위 방법으로 해결되지 않을 경우, `@Configuration` 클래스를 만들어 `CacheManager` Bean을 직접 설정하는 방법도 있습니다. 하지만 보통 `application.yml` 설정만으로 충분합니다.

문제 해결 과정에서 추가적인 질문이나 도움이 필요하면 언제든지 말씀해주세요!



---------------------------
Spring의 `@Cacheable` 어노테이션에서 `value` 속성을 커스텀 캐시 키 생성기(`KeyGenerator`) 내에서 직접 받아올 수는 없습니다. `@Cacheable` 어노테이션에서 `value` 속성은 캐시 이름을 나타내며, 이 정보는 캐시 키 생성기 클래스에서 직접 참조할 수 없습니다.

대신, `@Cacheable` 어노테이션 내부에서는 `org.springframework.cache.interceptor.KeyGenerator` 타입의 객체가 사용되며, 이 객체를 통해 메서드의 캐시 키 생성을 관리합니다. 이 객체는 `KeyGenerator` 인터페이스의 `generate` 메서드를 호출할 때 캐시 이름을 넘기지 않습니다.

일반적으로 `@Cacheable` 어노테이션은 메서드에 대한 캐시 설정을 제어하고, 커스텀 캐시 키 생성기(`KeyGenerator`)는 캐시 키를 생성하는 역할을 수행합니다. 따라서 캐시 이름(`value`)을 `generate` 메서드로 직접 전달하는 것은 일반적으로 지원되지 않습니다.

그러나 캐시 이름을 동적으로 변경해야 하는 경우, 커스텀 로직을 구현할 수 있습니다. 이를 위해 아래와 같은 방법을 고려할 수 있습니다:

1. 메서드 파라미터로 캐시 이름을 전달: 메서드의 파라미터로 캐시 이름을 전달하고, 커스텀 캐시 키 생성기에서 이 파라미터를 활용하여 캐시 키를 생성합니다.

```java
@Service
public class ExampleService {

    @Cacheable(keyGenerator = "customKeyGenerator")
    public CachedData getCachedData(String cacheName, String id) {
        // 캐시 이름과 id를 이용하여 캐시 키 생성
    }
}
```

2. ThreadLocal 또는 RequestContext를 사용하여 캐시 이름 설정: 요청 스레드마다 다른 캐시 이름을 설정하고, 커스텀 캐시 키 생성기에서 이 정보를 참조합니다.

```java
@Service
public class ExampleService {

    @Cacheable(keyGenerator = "customKeyGenerator")
    public CachedData getCachedData(String id) {
        String cacheName = CacheContextHolder.getCurrentCacheName(); // ThreadLocal 또는 RequestContext를 통해 캐시 이름 가져오기
        // 캐시 이름과 id를 이용하여 캐시 키 생성
    }
}
```

위의 방법 중 하나를 선택하여 캐시 이름을 커스텀 캐시 키 생성기에 전달할 수 있습니다.

--------------------

Spring Boot에서 Redis 캐시 구성 설정이 실패하는 경우, 어플리케이션의 시작을 방해할 수 있습니다. Spring Boot는 어플리케이션 컨텍스트가 올바르게 구성되었는지 확인하기 위해 시작 시 다양한 체크와 검증을 수행합니다. Redis 캐시와 같은 중요한 컴포넌트가 잘못 구성되거나 시작에 실패하면 어플리케이션 시작 실패로 이어질 수 있습니다.

하지만 Redis 캐시 설정 중 초기화 중에 발생할 수 있는 예외를 잡고 어플리케이션을 캐싱 없이 실행하도록 graceful하게 처리할 수 있습니다. 이를 위한 일반적인 접근 방식은 다음과 같습니다:

1. **Try-Catch 블록으로 캐시 설정 래핑:** Redis 캐시를 구성하는 Spring Boot 구성 클래스에서 Redis 캐시 설정 코드를 try-catch 블록으로 감싸서 초기화 중 발생할 수 있는 예외를 catch합니다.

```java
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.cache.CacheManager;
import org.springframework.data.redis.cache.RedisCacheManager;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;

@Configuration
public class CacheConfig {

    @Bean
    public CacheManager cacheManager(RedisConnectionFactory redisConnectionFactory) {
        try {
            RedisCacheManager cacheManager = new RedisCacheManager(redisTemplate());
            cacheManager.setTransactionAware(true);
            return cacheManager;
        } catch (Exception ex) {
            // 예외를 graceful하게 처리합니다 (로그 기록, 메시지 표시 등)
            // 예외를 무시하기로 선택할 수도 있습니다.
            return null; // 캐시 매니저를 null로 반환하여 캐싱을 비활성화합니다.
        }
    }

    @Bean
    public RedisTemplate<String, Object> redisTemplate() {
        // RedisTemplate을 구성하고 반환합니다.
    }
}
```

2. **예외 graceful하게 처리:** catch 블록 내에서 예외를 로그로 기록하거나 메시지를 표시하거나 어플리케이션에서 필요한 조치를 취합니다. 캐시가 사용 불가능한 경우에 대비하여 어플리케이션 로직에서 이 상황을 주의 깊게 처리해야 합니다. 사용 사례에 따라 캐싱이 사용 불가능한 경우 데이터 검색 또는 기타 작업에 대한 대체 메커니즘을 구현하는 것을 고려할 수 있습니다.

이 접근 방식을 따르면 Redis 캐시 설정 실패로 인해 어플리케이션이 시작하지 않더라도 적어도 어플리케이션은 시작되며 캐싱이 비활성화됩니다. 그러나 Redis 캐시 설정 실패의 근본적인 이슈를 해결하여 캐싱 시스템의 올바른 작동을 보장하는 것이 중요하다는 점을 염두에 두세요.


Spring에서 `@Cacheable` 어노테이션을 사용하면 캐시 설정이 실패하더라도 해당 어노테이션을 사용한 메서드 자체는 에러가 나지 않습니다. `@Cacheable` 어노테이션은 캐싱 관련 동작을 처리하기 위해 사용되며, 캐시 설정과는 독립적으로 작동합니다.

실제로 `@Cacheable` 어노테이션을 사용한 메서드는 다음과 같은 동작을 합니다:

1. 메서드 호출 시 캐시 키를 생성하고, 해당 키로 캐시에서 데이터를 검색합니다.
2. 캐시에 데이터가 존재하는 경우, 메서드 실행을 스킵하고 캐시된 데이터를 반환합니다.
3. 캐시에 데이터가 없는 경우, 메서드를 실행하고 그 결과를 캐시에 저장한 후 반환합니다.

따라서 캐시 설정이 실패하더라도 `@Cacheable` 어노테이션을 사용한 메서드는 그대로 실행되며, 캐싱 동작은 비활성화됩니다. 이는 어플리케이션에 영향을 미치지 않을 뿐만 아니라, 해당 메서드를 호출하는 코드에서도 변경이 필요하지 않습니다.

그러나 캐시 설정이 실패하면 캐시가 활성화되지 않으므로 메서드가 항상 실행되고 캐시에 저장된 데이터는 사용되지 않습니다. 따라서 캐시 설정에 문제가 있는 경우 캐싱 기능을 사용할 수 없게 되며, 이에 대한 대안을 고려해야 할 수 있습니다.


----------------

`CacheManager`가 실패하더라도 Spring Boot 어플리케이션은 기본적으로 시작할 수 있습니다. Spring Boot는 `CacheManager` 구성에 대한 오류를 처리하고 Graceful한 실패 처리를 지원합니다. 실패한 `CacheManager` 설정이 있더라도 어플리케이션은 캐싱을 사용하지 않는 상태로 시작됩니다.

실패한 `CacheManager` 설정은 캐싱 관련 동작을 비활성화하는데, 이는 `@Cacheable` 어노테이션이 사용된 메서드에 영향을 미칩니다. `@Cacheable` 어노테이션이 있는 메서드는 캐싱을 시도하겠지만, 실패한 `CacheManager`로 인해 캐싱은 비활성화되므로 해당 메서드는 항상 실행되고, 캐시된 데이터는 사용되지 않습니다.

그러나 `CacheManager` 구성이 실패하는 것은 일반적으로 원치 않는 상황이며, 적절한 캐시 설정을 해결하고 싶을 것입니다. `CacheManager` 설정이 실패하는 경우에는 다음과 같은 몇 가지 대응 방안을 고려할 수 있습니다:

1. **오류 복구:** Redis, EhCache 또는 다른 캐시 백엔드와의 연결 문제, 구성 오류 또는 종속성 문제 등을 해결하여 `CacheManager`를 올바르게 구성하십시오.

2. **다른 캐시 백엔드 사용:** 필요에 따라 다른 캐시 백엔드를 검토하고, 실패한 캐시 매니저 대신 다른 캐시 백엔드를 사용하십시오.

3. **기본 캐시 사용:** Spring의 기본 메모리 캐시인 ConcurrentMapCache를 사용하여 캐시를 설정하십시오. 이는 로컬 개발 및 테스트 환경에서 유용할 수 있습니다.

4. **테스트 및 디버깅:** `CacheManager` 구성을 테스트하고, 실패한 이유를 식별하고 디버깅하는 데 시간을 투자하십시오. 로그를 자세히 확인하여 문제를 찾을 수 있습니다.

`CacheManager`가 실패한 경우에는 캐싱 기능을 사용할 수 없게 되므로 어플리케이션의 성능에 영향을 미칠 수 있습니다. 따라서 `CacheManager` 설정에 대한 문제를 신속하게 해결하는 것이 좋습니다.


----------------------

Spring의 캐시 기능은 기본적으로 캐시 객체의 클래스 이름을 전체 패키지 경로와 함께 사용하여 캐시 키를 생성합니다. 이 동작은 캐시 키의 고유성을 보장하기 위한 것이지만 때로는 클래스 이름만을 사용하고자 할 수 있습니다.

클래스 이름만 사용하려면 다음과 같은 방법을 고려할 수 있습니다:

1. **Custom Cache Key Generator (커스텀 캐시 키 생성기):** Spring에서는 캐시 키를 생성하는데 사용되는 `KeyGenerator`를 커스터마이징할 수 있습니다. 이를 통해 클래스 이름만을 캐시 키로 사용할 수 있습니다.

2. **toString() 오버라이드:** 캐시 객체 클래스에서 `toString()` 메서드를 오버라이드하여 클래스 이름만 반환하도록 수정할 수 있습니다.

3. **클래스 이름 추출 후 사용:** 캐시 객체를 저장하기 전에 해당 객체의 클래스 이름을 추출하여 캐시 키로 사용할 수 있습니다.

아래에 각 방법을 더 자세히 설명하겠습니다:

**Custom Cache Key Generator (커스텀 캐시 키 생성기)를 사용한 방법:**

```java
import org.springframework.cache.interceptor.KeyGenerator;
import org.springframework.stereotype.Component;
import java.lang.reflect.Method;

@Component
public class CustomCacheKeyGenerator implements KeyGenerator {

    @Override
    public Object generate(Object target, Method method, Object... params) {
        String className = target.getClass().getName(); // 클래스 이름 추출
        // 다른 로직을 추가하여 캐시 키 생성
        return className;
    }
}
```

위의 `CustomCacheKeyGenerator` 클래스는 클래스 이름만을 캐시 키로 사용하도록 설정합니다. 이 커스텀 캐시 키 생성기를 사용하려면 적절한 메서드에 `keyGenerator` 속성을 설정하면 됩니다.

**toString() 메서드 오버라이드를 사용한 방법:**

```java
public class CachedData {
    private String id;
    private String value;

    // 기타 필드, 생성자, getter 및 setter 생략

    @Override
    public String toString() {
        return this.getClass().getName(); // 클래스 이름 반환
    }
}
```

위의 코드에서 `CachedData` 클래스의 `toString()` 메서드를 오버라이드하여 클래스 이름만 반환하도록 설정했습니다. 이렇게 하면 캐시 키로 클래스 이름만 사용됩니다.

**클래스 이름 추출 후 사용하는 방법:**

```java
public class CacheService {

    private final CacheManager cacheManager;

    @Autowired
    public CacheService(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    public void putData(String cacheName, String key, Object data) {
        String className = data.getClass().getName(); // 클래스 이름 추출
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.put(className, data); // 클래스 이름을 캐시 키로 사용
        }
    }

    // 기타 메서드
}
```

위의 코드에서 `putData` 메서드에서 클래스 이름을 직접 추출하여 캐시 키로 사용했습니다. 이 방식은 클래스 이름을 추출하고 나중에 캐시에서 데이터를 검색할 때 동일한 클래스 이름을 사용하여 데이터를 가져옵니다.


`Profile` 클래스 내부의 `Name` 클래스가 클래스 이름 대신 패키지 경로를 포함하여 캐시에 저장되는 문제를 해결하려면 `Name` 클래스에 대한 `toString()` 메서드를 오버라이드하여 클래스 이름만 반환하도록 만들면 됩니다.

여기서는 `Name` 클래스를 수정하고 `toString()` 메서드를 추가하는 방법을 제시합니다:

```java
public class Name {
    private String firstName;
    private String lastName;

    // 생성자, getter, setter 등의 코드

    @Override
    public String toString() {
        return getClass().getSimpleName(); // 클래스 이름만 반환
    }
}
```

위의 코드에서 `Name` 클래스의 `toString()` 메서드는 `getClass().getSimpleName()`를 사용하여 클래스의 단순 이름 (패키지 경로 없이)을 반환합니다. 이렇게 하면 `Name` 클래스의 객체를 캐시로 저장할 때 패키지 경로가 아닌 클래스 이름만 캐시 키로 사용됩니다.

이렇게 수정한 후 `putData` 메서드를 호출하면 `Name` 클래스의 객체가 클래스 이름만을 포함하여 캐시에 저장될 것입니다.


-------------------

Java의 `Object` 클래스를 사용하면 클래스 이름과 패키지 정보가 함께 저장될 수 있습니다. `Object` 클래스는 모든 클래스의 부모 클래스이며, `toString()` 메서드를 호출하면 클래스의 정규화된 이름이 반환됩니다.

만약 특정 클래스의 정보를 클래스 이름만으로 캐시하고 싶다면 해당 클래스에서 `toString()` 메서드를 오버라이드하고 클래스 이름만을 반환하도록 만들어야 합니다. 그러나 `Object` 클래스 자체를 수정할 수는 없으므로 다른 방법을 고려해야 합니다.

다음과 같은 옵션을 고려할 수 있습니다:

1. **특정 클래스의 `toString()` 오버라이드:** `Profile` 클래스 또는 해당 클래스의 내부 클래스(`Name`, `Property` 등)에서 `toString()` 메서드를 오버라이드하여 클래스 이름만 반환하도록 설정합니다. 그러나 이 방법은 클래스마다 수정이 필요하며, 모든 클래스에 대해 적용하기 어려울 수 있습니다.

2. **캐싱 전에 데이터 변환:** 데이터를 캐시에 넣기 전에 데이터를 변환하는 작업을 수행하여 클래스 정보를 숨길 수 있습니다. 예를 들어, `Profile` 객체를 캐시에 넣기 전에 `Profile` 객체를 `Map` 또는 다른 데이터 구조로 변환하여 캐시에 저장하면 클래스 정보가 저장되지 않습니다.

예를 들어, 다음과 같이 변환하는 방법이 있습니다:

```java
public void putData(String cacheName, String key, Profile profile) {
    Map<String, Object> dataMap = new HashMap<>();
    dataMap.put("name", profile.getName());
    dataMap.put("property", profile.getProperty());
    Cache cache = cacheManager.getCache(cacheName);
    if (cache != null) {
        cache.put(key, dataMap);
    }
}
```

이렇게 하면 `Profile` 객체 대신 `dataMap`을 캐시에 저장하게 됩니다. 이 방법은 클래스 정보를 캐시 키로 사용하지 않고 원하는 데이터 구조로 변환하여 저장하는 방법입니다.


---------------------

이러한 `SerializationException`과 "Could not read JSON"과 같은 에러는 Jackson 라이브러리를 사용하여 JSON 직렬화 및 역직렬화할 때 발생합니다. 이 에러는 Jackson이 역직렬화할 때 클래스에 기본 생성자가 없는 경우 발생합니다. "java.util.ArrayList$SubList"와 같은 클래스에는 기본 생성자가 없기 때문에 Jackson이 역직렬화를 처리하지 못하게 됩니다.

이 문제를 해결하기 위한 몇 가지 방법을 제시합니다:

1. **직렬화 클래스 수정:** `java.util.ArrayList$SubList`와 같은 클래스를 사용하는 데이터 구조를 수정하여 Jackson이 역직렬화를 수행할 수 있도록 만들 수 있습니다. 이를 위해 클래스에 기본 생성자를 추가하거나 Jackson에서 역직렬화할 때 무시하도록 설정할 수 있습니다.

    ```java
    @JsonIgnoreProperties(ignoreUnknown = true)
    public class MySubList extends ArrayList {
        // 기본 생성자 추가
        public MySubList() {
            super();
        }
    }
    ```

    이렇게 클래스를 수정하면 Jackson이 해당 클래스를 역직렬화할 때 문제를 해결할 수 있습니다.

2. **Mixin 클래스 사용:** Jackson에서 Mixin 클래스를 사용하여 역직렬화할 클래스를 수정하거나 무시하도록 설정할 수 있습니다. Mixin 클래스는 클래스의 역직렬화 방법을 변경하는 데 도움이 됩니다.

    ```java
    @JsonIgnoreProperties(ignoreUnknown = true)
    public abstract class ArrayListMixin extends ArrayList<Object> {
        // 믹스인 클래스
    }
    ```

    그런 다음 ObjectMapper에 Mixin 클래스를 등록합니다.

    ```java
    ObjectMapper mapper = new ObjectMapper();
    mapper.addMixIn(ArrayList.class, ArrayListMixin.class);
    ```

    이렇게 하면 Jackson이 `java.util.ArrayList$SubList`를 처리할 때 믹스인 클래스의 설정을 따르게 됩니다.

3. **Jackson 설정 수정:** Spring Boot에서 Jackson의 ObjectMapper 설정을 수정하여 역직렬화 중에 클래스 생성자를 호출하지 않도록 설정할 수 있습니다. 이렇게 하면 Jackson이 역직렬화를 시도할 때 클래스의 생성자를 호출하지 않습니다.

    ```java
    ObjectMapper objectMapper = new ObjectMapper();
    objectMapper.configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false);
    ```

    이 설정을 사용하면 Jackson이 무시할 수 없는 속성이나 생성자가 없는 클래스와 관련된 에러를 발생시키지 않습니다.

어떤 방법을 선택할지는 프로젝트의 요구사항과 상황에 따라 다를 수 있습니다. 클래스를 수정하거나 Mixin 클래스를 사용하는 방법은 클래스의 수정이 가능한 경우 유용합니다. 그러나 Jackson의 ObjectMapper 설정을 변경하는 방법은 전역적으로 영향을 미칠 수 있으므로 주의해야 합니다.


---------------------------

Spring Boot의 `SpringApplication`을 사용하여 애플리케이션 시작 시 Redis 캐시를 플러시하는 것은 조금 더 복잡하지만 가능합니다. `SpringApplication`의 이벤트 리스너를 활용하여 이 작업을 수행할 수 있습니다.

Redis 캐시를 플러시하려면 다음과 같은 단계를 따를 수 있습니다:

1. `SpringApplication`에 이벤트 리스너를 등록하여 애플리케이션이 시작될 때 Redis 캐시를 플러시하도록 합니다.

2. `ApplicationRunner` 또는 `CommandLineRunner`를 사용하여 Redis 캐시를 플러시하는 로직을 정의합니다.

아래는 Spring Boot의 `SpringApplication`을 사용하여 Redis 캐시를 플러시하는 예제입니다:

```java
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.boot.context.event.ApplicationReadyEvent;
import org.springframework.context.ApplicationListener;
import org.springframework.context.annotation.Bean;
import org.springframework.data.redis.cache.RedisCacheManager;

@SpringBootApplication
public class YourApplication {

    public static void main(String[] args) {
        SpringApplication.run(YourApplication.class, args);
    }

    @Bean
    public ApplicationListener<ApplicationReadyEvent> cacheFlushApplicationListener(RedisCacheManager cacheManager) {
        return event -> {
            // Redis 캐시를 플러시
            cacheManager.getCacheNames().forEach(cacheName -> cacheManager.getCache(cacheName).clear());
            System.out.println("Redis 캐시가 플러시되었습니다.");
        };
    }
}
```

위 코드에서는 `ApplicationReadyEvent` 이벤트가 발생하면 Redis 캐시를 플러시하는 이벤트 리스너를 등록합니다. `cacheFlushApplicationListener` 메서드에서 Redis 캐시를 플러시하는 코드를 정의합니다. 이 방법을 사용하면 Spring Boot 애플리케이션이 시작될 때 Redis 캐시를 플러시할 수 있습니다.


----------------------------

Spring Boot의 스케줄러(Scheduler)를 사용하여 Redis의 `FLUSHALL` 명령을 실행하려면 다음과 같은 단계를 따를 수 있습니다:

1. **스케줄러 구성:** 먼저 Spring Boot 애플리케이션에서 스케줄러를 구성해야 합니다. 이를 위해 `@EnableScheduling` 어노테이션을 사용하여 스케줄링을 활성화하고, `@Scheduled` 어노테이션을 사용하여 주기적으로 실행할 메서드를 정의합니다.

2. **Redis 연결 설정:** Redis를 사용하려면 `RedisConnectionFactory`와 `RedisTemplate`을 구성해야 합니다.

3. **Flush 명령 실행 메서드 구현:** Redis의 `FLUSHALL` 명령을 실행할 메서드를 구현합니다.

4. **스케줄링 설정:** 스케줄러를 사용하여 Redis `FLUSHALL` 명령을 주기적으로 실행하도록 설정합니다.

아래는 이러한 단계를 구체적으로 설명하는 예제입니다:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.context.annotation.Bean;
import org.springframework.data.redis.connection.RedisConnectionFactory;
import org.springframework.data.redis.core.RedisTemplate;
import org.springframework.scheduling.annotation.EnableScheduling;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;

@SpringBootApplication
@EnableScheduling
public class YourApplication {

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    public static void main(String[] args) {
        SpringApplication.run(YourApplication.class, args);
    }

    @Component
    public class RedisFlushScheduler {

        @Autowired
        private RedisConnectionFactory redisConnectionFactory;

        @Scheduled(cron = "0 0 * * * ?") // 매 시간 실행
        public void flushRedisCache() {
            // Redis FLUSHALL 명령 실행
            redisConnectionFactory.getConnection().flushAll();
            System.out.println("Redis FLUSHALL 명령이 실행되었습니다.");
        }
    }

    // Redis 연결 설정
    @Bean
    public RedisTemplate<String, Object> redisTemplate(RedisConnectionFactory redisConnectionFactory) {
        RedisTemplate<String, Object> redisTemplate = new RedisTemplate<>();
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        return redisTemplate;
    }
}
```

위 코드에서는 다음과 같은 주요 단계를 수행합니다:

1. `@EnableScheduling` 어노테이션을 사용하여 스케줄링을 활성화합니다.

2. `RedisFlushScheduler` 클래스에서 Redis의 `FLUSHALL` 명령을 실행할 메서드 `flushRedisCache`를 정의하고, `@Scheduled` 어노테이션을 사용하여 매 시간 실행되도록 설정합니다.

3. Redis 연결을 설정하기 위해 `RedisTemplate`을 구성합니다.

4. `flushRedisCache` 메서드에서는 `RedisConnectionFactory`를 사용하여 Redis의 `FLUSHALL` 명령을 실행하고, 실행 결과를 출력합니다.

이제 Spring Boot 애플리케이션이 시작되면 스케줄러를 통해 주기적으로 Redis `FLUSHALL` 명령이 실행됩니다.

-----------------------------------

Redis 성능 튜닝을 위해서는 여러 가지 측면에서 접근할 수 있습니다. Redis의 성능을 최적화하려면 아래와 같은 방법을 고려할 수 있습니다:

1. **적절한 하드웨어 선택:**
   - Redis는 주로 메모리 기반으로 동작하기 때문에 빠른 메모리와 높은 I/O 처리 능력을 가진 서버를 선택하세요.
   - SSD(Solid State Drive)를 사용하여 디스크 I/O를 최적화할 수 있습니다.

2. **캐시 만료 전략:**
   - 데이터가 더 이상 필요하지 않은 경우, `EXPIRE` 또는 `TTL` 명령을 사용하여 데이터를 자동으로 만료시키세요.
   - 만료되지 않는 데이터는 메모리를 계속 사용하므로, 적절한 만료 전략을 사용하여 메모리를 최적화하세요.

3. **적절한 데이터 구조 선택:**
   - Redis는 여러 가지 데이터 구조를 제공합니다. 적절한 데이터 구조를 선택하여 데이터를 저장하세요.
   - 예를 들어, 정렬된 집합(`ZSET`)은 범위 쿼리를 위한 데이터 구조로 사용될 수 있습니다.

4. **Pipeline 사용:**
   - 여러 개의 명령을 하나의 요청으로 묶어 전송하는 Redis Pipeline을 사용하여 네트워크 오버헤드를 줄일 수 있습니다.

5. **비동기 처리:**
   - 데이터베이스와 같은 외부 시스템과의 통신은 비동기로 처리하여 응답 시간을 최적화하세요.

6. **Lua 스크립팅 활용:**
   - 복잡한 연산이나 여러 명령을 수행할 때 Lua 스크립팅을 활용하여 Redis 서버 측에서 처리할 수 있습니다.

7. **Connection Pooling 사용:**
   - Redis 클라이언트에서 연결을 재사용하도록 connection pooling을 설정하여 연결 오버헤드를 최소화하세요.

8. **분산 환경:**
   - Redis 클러스터를 사용하여 데이터의 분산 저장과 처리를 가능하게 하여 확장성을 확보하세요.

9. **스냅샷 제한:**
   - 주기적인 스냅샷을 만들어 데이터를 디스크에 저장하는 방식을 사용할 때 스냅샷 빈도와 크기를 제한하여 디스크 I/O 부하를 줄일 수 있습니다.

10. **레디스 설정 튜닝:**
    - Redis의 설정 파일(redis.conf)을 편집하여 네트워크 버퍼 크기, 백그라운드 스냅샷 빈도, 연결 수 등을 조절할 수 있습니다.
    - Redis 6부터는 스레드(Thread) 모델을 지원하므로 Redis의 내부 스레드 수를 조절하여 CPU를 최적화할 수 있습니다.

11. **실시간 모니터링:**
    - Redis의 상태를 실시간으로 모니터링하고 성능 이슈를 파악할 수 있는 도구를 사용하세요. 예를 들어, `redis-cli`의 `info` 명령이나 여러 모니터링 도구를 활용할 수 있습니다.

성능 튜닝은 어떤 케이스에서도 유일한 해결책이 없습니다. 실제 사용 사례와 환경에 맞게 적절한 조치를 취해야 합니다. 필요하다면 Redis의 공식 문서와 Redis 커뮤니티의 지원을 참고하세요.

-----------------

Redis의 실시간 모니터링을 위해 여러 도구와 명령어가 있습니다. 다음은 실시간으로 Redis를 모니터링하는 방법에 대한 몇 가지 예시입니다:

1. **redis-cli의 MONITOR 명령어:**
   `redis-cli`를 사용하여 Redis 서버의 모든 명령어를 실시간으로 모니터링할 수 있습니다.
   ```
   redis-cli MONITOR
   ```

2. **redis-cli의 INFO 명령어:**
   `INFO` 명령어는 Redis 서버의 다양한 정보를 제공합니다. 이 정보에는 메모리 사용량, 클라이언트 연결 정보, 키의 수 등이 포함될 수 있습니다.
   ```
   redis-cli INFO
   ```

3. **Redis Live:**
   [Redis Live](https://redis.io/commands/live)는 Redis 6부터 제공되는 기능으로, Redis 서버의 다양한 지표를 실시간으로 볼 수 있는 웹 기반 대시보드를 제공합니다.

4. **Redis Exporter와 Prometheus:**
   [Redis Exporter](https://github.com/oliver006/redis_exporter)를 사용하여 Redis의 메트릭을 수집하고, [Prometheus](https://prometheus.io/)를 사용하여 이러한 메트릭을 저장하고 쿼리할 수 있습니다. 이를 통해 실시간으로 Redis의 성능과 상태를 모니터링할 수 있습니다.

5. **Grafana와 Redis Dashboard:**
   [Grafana](https://grafana.com/)는 다양한 데이터 소스로부터 데이터를 시각적으로 표시하는 대시보드를 제공하는 오픈 소스 툴입니다. Redis와 Grafana를 연동하여 Redis의 메트릭을 시각적으로 모니터링할 수 있는 대시보드를 구성할 수 있습니다.

이러한 방법들을 사용하여 Redis 서버의 상태를 실시간으로 모니터링할 수 있습니다. 선택한 방법은 환경과 요구사항에 따라 달라질 수 있습니다.


-----------------------------


물론, 당신이 진행하고 있는 테스트를 통해 Redis를 Embedded 모드로 사용하면서 CacheManager와 RedisTemplate을 사용하는 방법에 대해 자세히 설명하겠습니다.

Embedded Redis를 사용하여 테스트를 수행할 때, 일반적으로 Embedded Redis를 설정한 후 테스트용 RedisConnectionFactory를 생성하고 이를 기반으로 CacheManager와 RedisTemplate을 생성합니다. 이를 위해 여러 클래스와 어노테이션을 사용할 것입니다.

1. **Embedded Redis 설정 클래스 생성:**

```java
@Configuration
@Profile("test")
public class EmbeddedRedisConfig {

    @Value("${spring.redis.port}") // application-test.properties에서 실제 포트 번호를 가져옴
    private int redisPort;

    @Bean
    public RedisServer redisServer() throws IOException {
        RedisServer redisServer = new RedisServer(redisPort);
        redisServer.start();
        return redisServer;
    }
}
```

위 설정은 `@Profile("test")` 어노테이션을 통해 `application-test.properties`에서 정의한 Embedded Redis의 포트 번호를 가져와 Embedded Redis를 시작합니다.

2. **테스트 클래스에서 CacheManager와 RedisTemplate 사용:**

```java
@SpringBootTest
@ActiveProfiles("test")
public class YourRedisTest {

    @Autowired
    private CacheManager cacheManager;

    @Autowired
    private RedisTemplate<String, Object> redisTemplate;

    @Test
    public void testRedisOperations() {
        // Embedded Redis를 사용하여 Redis 연산 수행
        String key = "testKey";
        String value = "testValue";

        // RedisTemplate을 사용하여 데이터 저장
        redisTemplate.opsForValue().set(key, value);

        // CacheManager를 사용하여 캐시에 데이터 저장
        Cache cache = cacheManager.getCache("yourCacheName");
        cache.put(key, value);

        // 테스트 로직 검증
        assertEquals(value, redisTemplate.opsForValue().get(key));
        assertEquals(value, cache.get(key).get());
    }
}
```

위의 코드에서 `@ActiveProfiles("test")` 어노테이션은 Embedded Redis 설정을 사용할 것임을 나타냅니다. 테스트 클래스에서 `CacheManager`와 `RedisTemplate`을 주입받아 Embedded Redis를 사용한 Redis 연산을 수행할 수 있습니다.

이러한 설정을 사용하면 Embedded Redis를 사용하여 테스트 중에도 `CacheManager`와 `RedisTemplate`을 정상적으로 사용할 수 있습니다. 이 방법은 Embedded Redis를 사용하면서도 실제 Redis와 동일한 방식으로 캐시 및 Redis 데이터 저장 작업을 수행할 수 있게 해줍니다.


------------------------

`redisServer.start()`가 동작하지 않는다면 대체로 다음 두 가지 이유가 있을 수 있습니다:

1. **의존성 문제:** Embedded Redis를 사용하기 위해 올바른 의존성을 프로젝트에 추가했는지 확인하십시오. 대부분의 경우 `spring-boot-starter-data-redis` 또는 `redis.embedded`와 같은 의존성을 추가해야 합니다.

    ```xml
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-redis</artifactId>
    </dependency>
    ```

2. **포트 충돌:** 이미 시스템에서 사용 중인 Redis 또는 다른 프로그램이 선택한 포트 번호를 사용하고 있을 수 있습니다. Embedded Redis를 시작할 때 사용할 포트 번호를 지정하여 충돌을 방지할 수 있습니다.

다른 방법으로 Embedded Redis를 시작하는 대체 방법이 있습니다. 예를 들어, 다음과 같이 `GenericContainer`를 사용하여 Docker를 통해 Embedded Redis를 시작할 수 있습니다. 이 방법은 더 많은 유연성을 제공하고 대부분의 환경에서 동작합니다.

1. **Docker를 사용한 Embedded Redis 시작:**

   ```java
   import org.testcontainers.containers.GenericContainer;

   public class EmbeddedRedisContainer extends GenericContainer<EmbeddedRedisContainer> {

       public EmbeddedRedisContainer() {
           super("redis:latest");
           addExposedPort(6379); // Redis 포트 번호
       }
   }
   ```

   이제 테스트 클래스에서 이 컨테이너를 사용할 수 있습니다.

   ```java
   import org.junit.jupiter.api.Test;
   import org.springframework.boot.test.context.SpringBootTest;
   import org.springframework.test.context.ActiveProfiles;
   import org.testcontainers.junit.jupiter.Container;
   import org.testcontainers.junit.jupiter.Testcontainers;

   @SpringBootTest
   @ActiveProfiles("test")
   @Testcontainers
   public class YourRedisTest {

       @Container
       private static final EmbeddedRedisContainer embeddedRedis = new EmbeddedRedisContainer();

       // 테스트 메소드
   }
   ```

이 예제에서는 Testcontainers 라이브러리를 사용하여 Docker 컨테이너를 생성하고 테스트에서 사용합니다. `@Testcontainers` 어노테이션을 사용하면 테스트를 실행할 때 Docker 컨테이너가 자동으로 시작됩니다. 이 방법을 사용하면 다른 환경에서도 테스트를 실행할 때 일관된 Embedded Redis 환경을 얻을 수 있습니다. Testcontainers 라이브러리에 대해 더 알아보려면 [Testcontainers 공식 문서](https://www.testcontainers.org/)를 참고하십시오.
