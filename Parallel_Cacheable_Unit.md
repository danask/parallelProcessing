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


---------------------------

클래스 형태의 객체를 사용하여 캐시된 데이터를 가져오고, cacheName과 key를 지정하는 예제를 아래에 제시합니다. 이 예제에서는 Spring의 `CacheManager`를 직접 사용하여 캐시 작업을 수행합니다.

1. 데이터 객체 클래스:

```java
public class CachedData {
    private String id;
    private String value;

    // 생성자, getter, setter 메소드 생략
}
```

2. 캐시 관리를 위한 서비스 클래스:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Service;

@Service
public class CacheService {

    private final CacheManager cacheManager;

    @Autowired
    public CacheService(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    // 캐시에 데이터를 저장하는 메소드
    public void putData(String cacheName, String key, CachedData data) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.put(key, data);
        }
    }

    // 캐시에서 데이터를 가져오는 메소드
    public CachedData getData(String cacheName, String key) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            Cache.ValueWrapper valueWrapper = cache.get(key);
            if (valueWrapper != null) {
                return (CachedData) valueWrapper.get();
            }
        }
        return null; // 캐시에 데이터가 없는 경우
    }
}
```

3. 컨트롤러에서 캐시 사용 예제:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/example")
public class ExampleController {

    private final CacheService cacheService;

    @Autowired
    public ExampleController(CacheService cacheService) {
        this.cacheService = cacheService;
    }

    @GetMapping("/data/{id}")
    public CachedData getData(@PathVariable String id) {
        String cacheName = "dataCache";
        CachedData cachedData = cacheService.getData(cacheName, id);

        if (cachedData == null) {
            // 캐시에서 데이터를 찾지 못한 경우, 실제 데이터를 가져와서 캐시에 저장
            cachedData = fetchDataFromDatabase(id);
            cacheService.putData(cacheName, id, cachedData);
        }

        return cachedData;
    }

    // 실제 데이터를 가져오는 메소드 (예시)
    private CachedData fetchDataFromDatabase(String id) {
        // 데이터베이스에서 데이터를 가져오는 로직
        // 적절한 CachedData 객체를 생성하여 반환
    }
}
```

이 예제에서 `CacheService` 클래스를 사용하여 캐시에 데이터를 저장하고 가져오는 메소드를 구현합니다. 컨트롤러에서는 캐시에 데이터를 검색하고, 데이터가 존재하지 않는 경우 데이터베이스에서 가져온 데이터를 캐시에 저장합니다.


---------------------------

예, 캐시 키가 존재하는지 확인할 수 있습니다. Spring의 `Cache` 인터페이스를 사용하면 캐시에서 특정 키가 존재하는지 확인할 수 있습니다. 아래에 확인하는 방법을 보여주는 예제를 제시하겠습니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.cache.Cache;
import org.springframework.cache.CacheManager;
import org.springframework.stereotype.Service;

@Service
public class CacheService {

    private final CacheManager cacheManager;

    @Autowired
    public CacheService(CacheManager cacheManager) {
        this.cacheManager = cacheManager;
    }

    // 캐시에 데이터를 저장하는 메소드
    public void putData(String cacheName, String key, CachedData data) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            cache.put(key, data);
        }
    }

    // 캐시에서 데이터를 가져오는 메소드
    public CachedData getData(String cacheName, String key) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            Cache.ValueWrapper valueWrapper = cache.get(key);
            if (valueWrapper != null) {
                return (CachedData) valueWrapper.get();
            }
        }
        return null; // 캐시에 데이터가 없는 경우
    }

    // 캐시에서 특정 키가 존재하는지 확인하는 메소드
    public boolean isKeyPresent(String cacheName, String key) {
        Cache cache = cacheManager.getCache(cacheName);
        if (cache != null) {
            Cache.ValueWrapper valueWrapper = cache.get(key);
            return valueWrapper != null;
        }
        return false; // 캐시 자체가 없는 경우
    }
}
```

위의 `isKeyPresent` 메소드를 사용하여 특정 캐시에서 키가 존재하는지 확인할 수 있습니다. 이 메소드는 `Cache.ValueWrapper`를 사용하여 해당 키가 캐시에 있는지 확인하고, 캐시가 존재하지 않는 경우 `false`를 반환합니다.

캐시 키의 존재 여부를 확인할 때, `isKeyPresent` 메소드를 호출하여 캐시 키가 존재하는지 여부를 판단하고, 이에 따라 추가적인 로직을 수행할 수 있습니다.
