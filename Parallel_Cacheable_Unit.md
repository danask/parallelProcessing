Spring에서 `DeferredResult`를 사용하여 비동기 응답을 처리하고, `@Cacheable` 어노테이션을 사용하여 캐시를 활용하는 방법을 설명하겠습니다. 먼저 `DeferredResult`를 사용하여 비동기 응답을 처리하고, 그 다음 캐시와 서비스 메소드 호출을 조합하는 방법을 보여줄 것입니다.

1. `DeferredResult`를 사용한 비동기 컨트롤러:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import org.springframework.web.context.request.async.DeferredResult;

import java.util.concurrent.CompletableFuture;
import java.util.concurrent.ExecutorService;
import java.util.concurrent.Executors;

@RestController
@RequestMapping("/example")
public class ExampleController {

    private final ExampleService exampleService;
    private final ExecutorService executorService = Executors.newFixedThreadPool(10);

    @Autowired
    public ExampleController(ExampleService exampleService) {
        this.exampleService = exampleService;
    }

    @GetMapping("/data/{id}")
    public DeferredResult<String> getData(@PathVariable String id) {
        DeferredResult<String> deferredResult = new DeferredResult<>();
        
        // 먼저 캐시에서 데이터를 찾아보고, 캐시가 있으면 즉시 응답
        String cachedData = exampleService.getCachedData(id);
        if (cachedData != null) {
            deferredResult.setResult(cachedData);
        } else {
            // 캐시가 없으면 비동기로 서비스 메소드 호출하고 결과를 반환
            CompletableFuture.supplyAsync(() -> exampleService.getData(id), executorService)
                    .whenCompleteAsync((result, throwable) -> {
                        if (throwable != null) {
                            deferredResult.setErrorResult(throwable);
                        } else {
                            deferredResult.setResult(result);
                        }
                    });
        }
        return deferredResult;
    }
}
```

2. `@Cacheable`을 사용한 서비스:

```java
import org.springframework.cache.annotation.CacheEvict;
import org.springframework.cache.annotation.Cacheable;
import org.springframework.stereotype.Service;

@Service
public class ExampleService {

    @Cacheable(cacheNames = "dataCache", key = "#id")
    public String getCachedData(String id) {
        // 이 메소드는 캐시가 존재하는 경우 캐시에서 데이터를 반환합니다.
        return null;
    }

    @CacheEvict(cacheNames = "dataCache", key = "#id")
    public void evictCachedData(String id) {
        // 이 메소드는 캐시에서 데이터를 제거합니다.
    }

    public String getData(String id) {
        // 실제 데이터를 가져오는 로직
        String data = fetchDataFromDatabase(id);
        // 데이터를 가져온 후에 캐시에 저장
        storeDataInCache(id, data);
        return data;
    }

    private String fetchDataFromDatabase(String id) {
        // 데이터베이스에서 데이터를 가져오는 로직
    }

    private void storeDataInCache(String id, String data) {
        // 데이터를 캐시에 저장하는 로직
    }
}
```

위의 코드에서는 `/example/data/{id}` 엔드포인트를 통해 데이터를 비동기로 반환하는 예제를 제시하고 있습니다. 이때 `DeferredResult`를 사용하여 비동기 응답을 처리하며, `@Cacheable` 어노테이션을 사용하여 데이터를 캐시에서 검색합니다. 만약 캐시에 데이터가 없는 경우에는 서비스 메소드를 호출하여 데이터를 가져오고, 가져온 데이터를 캐시에 저장합니다.

--------------------------

`@Cacheable` 어노테이션 대신에 함수로만 캐시된 데이터를 가져오는 예제를 보여드리겠습니다. 이를 위해서는 Spring의 `CacheManager`를 직접 사용하여 캐시에 접근하고 데이터를 가져올 수 있습니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Service;

@Service
public class ExampleService {

    private final CacheManager cacheManager;

    @Autowired
    public ExampleService(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    public String getCachedData(String cacheName, String key) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            Cache.ValueWrapper wrapper = cache.get(key);
            if (wrapper != null) {
                return (String) wrapper.get();
            }
        }
        return null; // 캐시에 데이터가 없는 경우
    }

    public void storeDataInCache(String cacheName, String key, String data) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.put(key, data);
        }
    }

    public void evictCachedData(String cacheName, String key) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.evict(key);
        }
    }
    
    public String fetchDataFromDatabase(String id) {
        // 데이터베이스에서 데이터를 가져오는 로직
    }

    public String getData(String id) {
        String cacheName = "dataCache";
        String key = id;

        // 캐시에서 데이터 검색
        String cachedData = getCachedData(cacheName, key);
        if (cachedData != null) {
            return cachedData;
        }

        // 캐시에 데이터가 없는 경우 데이터베이스에서 데이터를 가져옴
        String data = fetchDataFromDatabase(id);

        // 가져온 데이터를 캐시에 저장
        storeDataInCache(cacheName, key, data);

        return data;
    }
}
```

위의 코드에서 `ExampleService` 클래스는 `CacheManager`를 사용하여 캐시에 직접 접근합니다. `getCachedData()` 함수는 캐시에서 데이터를 가져오고, `storeDataInCache()` 함수는 데이터를 캐시에 저장하며, `evictCachedData()` 함수는 캐시에서 데이터를 제거합니다. 이 함수들을 이용하여 캐시를 다룰 수 있으며, `getData()` 함수에서는 캐시에 데이터가 없는 경우 데이터베이스에서 데이터를 가져와 캐시에 저장합니다.
