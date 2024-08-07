이해했습니다. 주어진 요구 사항에 따라 동적으로 쿼리를 생성하고 실행하려는 예제를 구현해 보겠습니다. 특히 `_keyword`가 NULL이 아닐 때 쿼리를 동적으로 생성하고, 그 쿼리를 사용하여 최종 쿼리를 구성하려는 목적입니다.

### 구현 예제

```sql
CREATE OR REPLACE FUNCTION getData(_keyword character varying)
RETURNS TABLE(appname character varying) AS
$$
DECLARE
    variable text;
    query text;
BEGIN
    -- 기본 쿼리 조건을 설정
    query := 'SELECT x FROM y WHERE count > ';

    -- 조건에 따라 variable 값을 설정
    IF _keyword IS NOT NULL THEN
        variable := 'SELECT a FROM b WHERE appname = ' || quote_literal(_keyword);
    ELSE
        variable := 'SELECT a FROM b'; -- _keyword가 NULL인 경우 기본 쿼리
    END IF;

    -- 동적으로 쿼리 생성
    query := query || ' (SELECT COUNT(*) FROM (' || variable || ') subquery)';

    -- 동적으로 쿼리 실행
    RETURN QUERY EXECUTE query;
END;
$$ LANGUAGE plpgsql;
```

### 설명

1. **변수 선언**:
   - `variable`과 `query` 변수를 선언합니다. `variable`은 조건에 따라 동적으로 생성될 쿼리를 저장하고, `query`는 최종적으로 실행할 쿼리를 구성합니다.

2. **조건에 따른 변수 설정**:
   - `_keyword`가 NULL이 아닌 경우, `variable`에 `appname`이 `_keyword`와 일치하는 데이터를 선택하는 쿼리를 설정합니다.
   - `_keyword`가 NULL인 경우, 기본 쿼리를 설정합니다.

3. **동적으로 쿼리 생성**:
   - `query` 변수에 `variable` 쿼리를 포함하여 최종 쿼리를 만듭니다. 여기서는 `count > (SELECT COUNT(*) FROM ...)` 형태로 `variable` 쿼리 결과의 카운트를 비교합니다.

4. **쿼리 실행 및 반환**:
   - `RETURN QUERY EXECUTE query;`를 사용하여 동적으로 생성된 쿼리를 실행하고 결과를 반환합니다.

### 사용 예

이 함수를 호출하여 데이터를 필터링하는 방법은 다음과 같습니다:

```sql
SELECT * FROM getData('example_keyword');
```

이 쿼리는 `_keyword`가 NULL이 아닐 경우 `_keyword`와 일치하는 `appname`을 가진 데이터를 포함하는 쿼리를 동적으로 생성하여 실행합니다.

이 접근 방법을 사용하면 `_keyword`의 값에 따라 쿼리를 동적으로 조정하고, 필요에 따라 기본 쿼리와 필터링된 쿼리를 유연하게 처리할 수 있습니다.
----------------------

만약 전체 쿼리 안에 조건부로 일부 쿼리를 동적으로 추가해야 한다면, 쿼리를 동적으로 생성하고 그에 맞춰서 조건을 처리할 수 있습니다. 이 경우, 쿼리의 일부를 `IF` 조건에 따라 추가하거나 제외할 수 있습니다. 다음은 이런 시나리오를 처리하는 방법을 보여주는 예입니다.

```sql
CREATE OR REPLACE FUNCTION get_filtered_data(_keyword character varying, _customer_id integer, _group_id integer)
RETURNS TABLE(appname character varying, customer_id integer, group_id integer) AS
$$
DECLARE
    base_query text;
BEGIN
    -- 기본 쿼리
    base_query := 'SELECT appname, customer_id, group_id FROM my_table WHERE 1=1';

    -- 조건에 따라 키워드 필터 추가
    IF _keyword IS NOT NULL AND _keyword <> '' THEN
        base_query := base_query || ' AND appname LIKE ''%' || _keyword || '%''';
    END IF;

    -- 조건에 따라 customer_id 필터 추가
    IF _customer_id IS NOT NULL AND _customer_id <> -1 THEN
        base_query := base_query || ' AND customer_id = ' || _customer_id;
    END IF;

    -- 조건에 따라 group_id 필터 추가
    IF _group_id IS NOT NULL AND _group_id <> -1 THEN
        base_query := base_query || ' AND group_id = ' || _group_id;
    END IF;

    -- 동적으로 생성된 쿼리 실행 및 결과 반환
    RETURN QUERY EXECUTE base_query;
END;
$$ LANGUAGE plpgsql;
```

### 설명:

1. **기본 쿼리 설정**:
   - `base_query := 'SELECT appname, customer_id, group_id FROM my_table WHERE 1=1';`:
     - 기본 쿼리 문자열을 설정합니다. `1=1`은 쿼리 생성을 단순화하기 위해 항상 참인 조건으로 사용됩니다.

2. **조건부 쿼리 추가**:
   - **키워드 조건**: `_keyword`가 `NULL`이 아니고 빈 문자열이 아닐 경우에만 `appname LIKE` 조건을 추가합니다.
   - **customer_id 조건**: `_customer_id`가 `NULL`이 아니고 -1이 아닐 경우에만 `customer_id` 필터를 추가합니다.
   - **group_id 조건**: `_group_id`가 `NULL`이 아니고 -1이 아닐 경우에만 `group_id` 필터를 추가합니다.

3. **쿼리 실행 및 결과 반환**:
   - 동적으로 생성된 `base_query`를 실행하고 결과를 반환합니다.

이 방식은 전체 쿼리 구조에 동적으로 조건을 추가하며, 조건에 따라 쿼리의 일부만 실행되도록 처리합니다.

----------------------

`_keyword`가 `NULL`이거나 빈 문자열(`''`)일 경우, 쿼리를 실행하지 않고 `NULL`을 반환하도록 하려면 PL/pgSQL에서 다음과 같이 구현할 수 있습니다.

```sql
CREATE OR REPLACE FUNCTION get_appname_by_keyword(_keyword character varying)
RETURNS character varying AS
$$
DECLARE
    result character varying;
    query text;
BEGIN
    IF _keyword IS NULL OR _keyword = '' THEN
        RETURN NULL;
    ELSE
        query := 'SELECT appname FROM my_table WHERE appname LIKE ''%' || _keyword || '%'' LIMIT 1';
        EXECUTE query INTO result;
        RETURN result;
    END IF;
END;
$$ LANGUAGE plpgsql;
```

### 설명:

1. **조건문**: 
   - `IF _keyword IS NULL OR _keyword = '' THEN`: `_keyword`가 `NULL`이거나 빈 문자열일 경우, 바로 `NULL`을 반환합니다.
   
2. **쿼리 생성 및 실행**: 
   - `query := 'SELECT appname FROM my_table WHERE appname LIKE ''%' || _keyword || '%'' LIMIT 1';`:
     - `_keyword`가 유효할 때만 쿼리를 생성합니다.
     - `LIMIT 1`을 추가해 한 개의 결과만 반환합니다.
   - `EXECUTE query INTO result;`: 동적으로 생성된 쿼리를 실행하고, 결과를 `result` 변수에 저장합니다.

3. **결과 반환**: 
   - `RETURN result;`: 결과를 반환합니다.

이 방식은 `_keyword`가 `NULL`이거나 빈 문자열일 경우 `NULL`을 반환하고, 그렇지 않을 경우 쿼리 결과를 반환합니다.

-----------------------

`_deviceCount`가 `-1`이 아닌 경우에만 `deviceCount > _deviceCount` 조건을 적용하려면, PL/pgSQL 함수에서 다음과 같은 방법으로 구현할 수 있습니다.

```sql
CREATE OR REPLACE FUNCTION get_filtered_data(
    _customer_id integer,
    _group_id integer,
    _appName character varying,
    _deviceCount integer
)
RETURNS TABLE(customer_id integer, group_id integer, appname character varying, devicecount integer, data_value text) AS
$$
BEGIN
    RETURN QUERY
    SELECT customer_id, group_id, appname, devicecount, data_value
    FROM my_table
    WHERE 
        (customer_id = _customer_id OR _customer_id = -1)
        AND 
        (group_id = _group_id OR _group_id = -1)
        AND 
        (appname = _appName OR _appName IS NULL)
        AND 
        (_deviceCount = -1 OR devicecount > _deviceCount);
END;
$$ LANGUAGE plpgsql;
```

### 설명:

- `(customer_id = _customer_id OR _customer_id = -1)`: `customer_id` 필터, `_customer_id`가 `-1`이면 조건을 무시.
- `(group_id = _group_id OR _group_id = -1)`: `group_id` 필터, `_group_id`가 `-1`이면 조건을 무시.
- `(appname = _appName OR _appName IS NULL)`: `appname` 필터, `_appName`이 `NULL`이면 조건을 무시.
- `(_deviceCount = -1 OR devicecount > _deviceCount)`: `_deviceCount`가 `-1`이 아니면 `devicecount > _deviceCount` 조건을 적용, 그렇지 않으면 조건을 무시.

이 방식은 `_deviceCount`가 `-1`일 경우 해당 조건을 무시하고, 그렇지 않을 경우 `deviceCount > _deviceCount` 조건을 적용합니다. 이는 `_deviceCount` 값이 특정 임계값 이상인지 확인하는데 유용합니다.

---------------------------------

PL/pgSQL에서는 `IF` 문을 사용하여 조건을 처리할 수 있습니다. 하지만 `IF` 문은 보통 절차적인 논리에서 사용되고, SQL 쿼리 내에서 조건을 동적으로 변경하려면 `CASE` 문이나 `COALESCE`를 사용하는 것이 더 일반적입니다.

지금 설명하신 조건을 처리하려면, WHERE 절 내에서 다음과 같은 조건을 사용하면 됩니다:

```sql
WHERE 
    (customer_id = _customer_id OR _customer_id = -1)
    AND 
    (group_id = _group_id OR _group_id = -1)
    AND 
    (appname = _appName OR _appName IS NULL);
```

이 코드의 의미는 `_appName`이 `NULL`이 아닌 경우에만 `appname` 컬럼이 `_appName`과 일치하는 값을 반환하도록 하고, `_appName`이 `NULL`일 경우에는 이 조건을 무시하도록 합니다.

### 전체 함수 코드 예시:

```sql
CREATE OR REPLACE FUNCTION get_filtered_data(
    _customer_id integer,
    _group_id integer,
    _appName character varying
)
RETURNS TABLE(customer_id integer, group_id integer, appname character varying, data_value text) AS
$$
BEGIN
    RETURN QUERY
    SELECT customer_id, group_id, appname, data_value
    FROM my_table
    WHERE 
        (customer_id = _customer_id OR _customer_id = -1)
        AND 
        (group_id = _group_id OR _group_id = -1)
        AND 
        (appname = _appName OR _appName IS NULL);
END;
$$ LANGUAGE plpgsql;
```

### 설명:

- `(appname = _appName OR _appName IS NULL)` 조건은 `_appName`이 `NULL`이 아닌 경우 `appname`이 `_appName`과 일치하는 데이터를 필터링하며, `_appName`이 `NULL`이면 해당 조건을 무시하게 됩니다.
- 이렇게 하면 `_appName`이 `NULL`일 때 `appname` 필터를 무시하고, 그렇지 않으면 해당 필터를 적용할 수 있습니다. 

이 방식을 사용하면 입력된 값에 따라 동적으로 조건을 적용할 수 있어 더 유연한 쿼리를 작성할 수 있습니다.
-------------------------

`WHERE` 절에서 `LIKE` 연산자를 사용하여 `keyword` 변수를 포함한 검색을 수행하려면, 변수의 값을 적절히 포함시켜야 합니다. 단순히 `'%'keyword'%'`로 작성하면 변수로 인식되지 않고 문자 그대로 취급됩니다.

### PostgreSQL에서 `LIKE`를 사용하는 방법:

`Stored Procedure`에서 입력된 값이 특정 조건(`-1` 또는 `NULL`)일 때 `WHERE` 절에서 해당 조건을 제외하려면 `COALESCE` 또는 조건문(`IF`, `CASE`)을 사용하여 처리할 수 있습니다. 이렇게 하면 조건을 동적으로 제어할 수 있습니다. 

아래는 PostgreSQL의 예제입니다:

### 1. **단순한 `IF` 조건 사용**
```sql
CREATE OR REPLACE FUNCTION get_filtered_data(
    _customer_id integer,
    _group_id integer
)
RETURNS TABLE(customer_id integer, group_id integer, data_value text) AS
$$
BEGIN
    RETURN QUERY
    SELECT customer_id, group_id, data_value
    FROM my_table
    WHERE 
        (customer_id = _customer_id OR _customer_id = -1)
        AND 
        (group_id = _group_id OR _group_id = -1);
END;
$$ LANGUAGE plpgsql;
```

### 2. **`COALESCE` 함수 사용**
   - `COALESCE` 함수는 `NULL` 값을 대체하는 데 사용되며, 첫 번째로 `NULL`이 아닌 값을 반환합니다.
   - 이를 통해 `NULL`이 입력될 경우 조건을 무시할 수 있습니다.

```sql
CREATE OR REPLACE FUNCTION get_filtered_data(
    _customer_id integer,
    _group_id integer
)
RETURNS TABLE(customer_id integer, group_id integer, data_value text) AS
$$
BEGIN
    RETURN QUERY
    SELECT customer_id, group_id, data_value
    FROM my_table
    WHERE 
        (customer_id = COALESCE(_customer_id, customer_id))
        AND 
        (group_id = COALESCE(_group_id, group_id));
END;
$$ LANGUAGE plpgsql;
```

### 3. **동적 SQL 사용**
   - 동적 SQL을 사용하여 `WHERE` 절을 동적으로 생성하는 방법도 있습니다.

```sql
CREATE OR REPLACE FUNCTION get_filtered_data(
    _customer_id integer,
    _group_id integer
)
RETURNS TABLE(customer_id integer, group_id integer, data_value text) AS
$$
DECLARE
    query text;
BEGIN
    query := 'SELECT customer_id, group_id, data_value FROM my_table WHERE 1=1';

    IF _customer_id IS NOT NULL AND _customer_id != -1 THEN
        query := query || ' AND customer_id = ' || _customer_id;
    END IF;

    IF _group_id IS NOT NULL AND _group_id != -1 THEN
        query := query || ' AND group_id = ' || _group_id;
    END IF;

    RETURN QUERY EXECUTE query;
END;
$$ LANGUAGE plpgsql;
```

### 설명
- **첫 번째 방법:** `OR` 조건을 사용하여 `_customer_id`나 `_group_id`가 특정 값(`-1`)이면 조건을 무시합니다.
- **두 번째 방법:** `COALESCE`를 사용하여 `_customer_id` 또는 `_group_id`가 `NULL`일 경우 조건을 무시하고, 입력된 값을 그대로 사용합니다.
- **세 번째 방법:** 동적 SQL을 사용하여 필요한 조건만 `WHERE` 절에 추가합니다. 이 방법은 조건이 많아질 때 유용합니다.

이 예제들을 통해 `Stored Procedure`에서 조건부로 `WHERE` 절을 적용하여 동적으로 쿼리를 생성할 수 있습니다.
----------------------
```sql
WHERE appname LIKE '%' || keyword || '%'
```
- `||` 연산자를 사용하여 문자열을 결합합니다.
- `keyword` 변수를 `%`와 함께 결합하여 부분 문자열 검색을 수행합니다.

### 예시:
```sql
CREATE OR REPLACE FUNCTION search_by_keyword(keyword text)
RETURNS TABLE(appname text) AS
$$
BEGIN
    RETURN QUERY
    SELECT appname
    FROM my_table
    WHERE appname LIKE '%' || keyword || '%';
END;
$$ LANGUAGE plpgsql;
```

### `PL/pgSQL` 예시
```sql
CREATE OR REPLACE FUNCTION search_by_keyword(_keyword text)
RETURNS TABLE(appname text) AS
$$
BEGIN
    RETURN QUERY
    SELECT appname
    FROM my_table
    WHERE appname LIKE '%' || _keyword || '%';
END;
$$ LANGUAGE plpgsql;
```

### 동적 SQL 예시
```sql
CREATE OR REPLACE FUNCTION search_by_keyword(_keyword text)
RETURNS TABLE(appname text) AS
$$
DECLARE
    query text;
BEGIN
    query := 'SELECT appname FROM my_table WHERE appname LIKE ''%' || _keyword || '%''';
    RETURN QUERY EXECUTE query;
END;
$$ LANGUAGE plpgsql;
```

### 주의사항
- 문자열 결합을 할 때 `||` 연산자를 사용합니다.
- 동적 SQL을 사용할 경우에는 싱글쿼트(`'`)를 적절히 이스케이프(`''`)해야 합니다.

이렇게 하면 `keyword` 변수를 포함하여 부분 검색을 수행할 수 있습니다.
----------------------
수동으로 쿼리 결과를 원하는 클래스에 매핑하는 방법을 사용할 수 있습니다. 이 방법에서는 각 결과 행을 수동으로 객체에 매핑하여 리스트에 추가합니다.

아래는 수동으로 쿼리 결과를 `AppUsageTopAppSP` 클래스에 매핑하는 예제입니다.

### 수동 매핑 예제 코드

```java
import org.hibernate.query.NativeQuery;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.ArrayList;

@Service
public class YourService {

    @PersistenceContext
    private EntityManager entityManager;

    public List<AppUsageTopAppSP> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5) {
        Query query = entityManager.createNativeQuery(
                "SELECT devicedate, devicecount, customerid, groupid, ... " + // 필요한 컬럼들 추가
                "FROM kaiappinfo.fn_appusage_datausage_daily_00(:customerId, :groupId, :appVersion, :deviceDateStart, :deviceDateEnd, :appUID, :appName1, :pkgName1, :appName2, :pkgName2, :appName3, :pkgName3, :appName4, :pkgName4, :appName5, :pkgName5)")
                .setParameter("customerId", customerId)
                .setParameter("groupId", groupId)
                .setParameter("appVersion", appVersion)
                .setParameter("deviceDateStart", deviceDateStart)
                .setParameter("deviceDateEnd", deviceDateEnd)
                .setParameter("appUID", appUID)
                .setParameter("appName1", appName1)
                .setParameter("pkgName1", pkgName1)
                .setParameter("appName2", appName2)
                .setParameter("pkgName2", pkgName2)
                .setParameter("appName3", appName3)
                .setParameter("pkgName3", pkgName3)
                .setParameter("appName4", appName4)
                .setParameter("pkgName4", pkgName4)
                .setParameter("appName5", appName5)
                .setParameter("pkgName5", pkgName5);

        List<Object[]> resultList = query.getResultList();
        
        List<AppUsageTopAppSP> appUsageTopAppList = new ArrayList<>();

        for (Object[] row : resultList) {
            AppUsageTopAppSP appUsageTopAppSP = new AppUsageTopAppSP();
            appUsageTopAppSP.setDevicedate((Date) row[0]);
            appUsageTopAppSP.setDevicecount((Long) row[1]);
            appUsageTopAppSP.setCustomerid((String) row[2]);
            appUsageTopAppSP.setGroupid((Integer) row[3]);
            // 필요한 모든 필드 매핑
            // appUsageTopAppSP.setOtherField((FieldType) row[N]);

            appUsageTopAppList.add(appUsageTopAppSP);
        }

        return appUsageTopAppList;
    }
}
```

### AppUsageTopAppSP 클래스

아래는 `AppUsageTopAppSP` 클래스의 예시입니다. 클래스의 필드와 매핑할 결과 컬럼의 순서 및 타입이 일치해야 합니다.

```java
public class AppUsageTopAppSP {
    private Date devicedate;
    private Long devicecount;
    private String customerid;
    private Integer groupid;
    // 필요한 다른 필드들

    // Getter와 Setter 메소드들

    public Date getDevicedate() {
        return devicedate;
    }

    public void setDevicedate(Date devicedate) {
        this.devicedate = devicedate;
    }

    public Long getDevicecount() {
        return devicecount;
    }

    public void setDevicecount(Long devicecount) {
        this.devicecount = devicecount;
    }

    public String getCustomerid() {
        return customerid;
    }

    public void setCustomerid(String customerid) {
        this.customerid = customerid;
    }

    public Integer getGroupid() {
        return groupid;
    }

    public void setGroupid(Integer groupid) {
        this.groupid = groupid;
    }

    // 필요한 다른 필드들의 Getter와 Setter 메소드들
}
```

이 예제에서는 `Object[]` 배열의 각 인덱스를 사용하여 수동으로 필드 값을 설정합니다. 이 방식은 유연성을 제공하며, Hibernate의 `Transformers`를 사용하지 않기 때문에 문제가 발생할 가능성이 적습니다.

----------------------
트랜스폼하기 전에 쿼리 결과를 미리 프린트해서 볼 수 있습니다. `query.getResultList()`를 호출하여 결과를 리스트로 받고, 이를 로그나 콘솔에 출력할 수 있습니다. 아래는 트랜스폼 전에 결과를 출력하는 예제입니다.

### 수정된 예제 코드

```java
import org.hibernate.query.NativeQuery;
import org.hibernate.transform.Transformers;
import org.hibernate.type.StandardBasicTypes;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Map;

@Service
public class YourService {

    @PersistenceContext
    private EntityManager entityManager;

    public List<AppUsageTopAppSP> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5) {
        Query query = entityManager.createNativeQuery(
                "SELECT devicedate, devicecount, customerid, groupid, ... " + // 필요한 컬럼들 추가
                "FROM kaiappinfo.fn_appusage_datausage_daily_00(:customerId, :groupId, :appVersion, :deviceDateStart, :deviceDateEnd, :appUID, :appName1, :pkgName1, :appName2, :pkgName2, :appName3, :pkgName3, :appName4, :pkgName4, :appName5, :pkgName5)")
                .setParameter("customerId", customerId)
                .setParameter("groupId", groupId)
                .setParameter("appVersion", appVersion)
                .setParameter("deviceDateStart", deviceDateStart)
                .setParameter("deviceDateEnd", deviceDateEnd)
                .setParameter("appUID", appUID)
                .setParameter("appName1", appName1)
                .setParameter("pkgName1", pkgName1)
                .setParameter("appName2", appName2)
                .setParameter("pkgName2", pkgName2)
                .setParameter("appName3", appName3)
                .setParameter("pkgName3", pkgName3)
                .setParameter("appName4", appName4)
                .setParameter("pkgName4", pkgName4)
                .setParameter("appName5", appName5)
                .setParameter("pkgName5", pkgName5);

        List<Object[]> resultList = query.getResultList();
        
        // 결과 출력
        for (Object[] row : resultList) {
            System.out.println("Row: ");
            for (Object column : row) {
                System.out.print(column + " ");
            }
            System.out.println();
        }

        // 트랜스폼 후 결과 반환
        query.unwrap(NativeQuery.class)
                .addScalar("devicedate", StandardBasicTypes.DATE)
                .addScalar("devicecount", StandardBasicTypes.LONG)
                .addScalar("customerid", StandardBasicTypes.STRING)
                .addScalar("groupid", StandardBasicTypes.INTEGER)
                .setResultTransformer(Transformers.aliasToBean(AppUsageTopAppSP.class));
        
        return query.getResultList();
    }
}
```

### 필요한 import 추가

```java
import org.hibernate.transform.Transformers;
import org.hibernate.type.StandardBasicTypes;
import java.util.List;
import java.util.Map;
```

위 코드에서 쿼리 실행 후 결과를 `List<Object[]>`로 받아 각 행을 출력합니다. 그런 다음 트랜스폼을 적용하여 최종 결과를 반환합니다. 이렇게 하면 트랜스폼 전에 쿼리 결과를 미리 확인할 수 있습니다.

--------------------
`DateType.INSTANCE`는 Hibernate에서 사용하는 특정 데이터 유형을 지정하는 데 사용됩니다. 하지만 이 예제에서 잘못 사용되었으며, 대신 `org.hibernate.type.StandardBasicTypes`를 사용하여 올바른 데이터 유형을 지정할 수 있습니다.

아래는 수정된 예제입니다:

### 수정된 예제 코드

```java
import org.hibernate.query.NativeQuery;
import org.hibernate.transform.Transformers;
import org.hibernate.type.StandardBasicTypes;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class YourService {

    @PersistenceContext
    private EntityManager entityManager;

    public List<AppUsageTopAppSP> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5) {
        Query query = entityManager.createNativeQuery(
                "SELECT devicedate, devicecount, customerid, groupid, ... " + // 필요한 컬럼들 추가
                "FROM kaiappinfo.fn_appusage_datausage_daily_00(:customerId, :groupId, :appVersion, :deviceDateStart, :deviceDateEnd, :appUID, :appName1, :pkgName1, :appName2, :pkgName2, :appName3, :pkgName3, :appName4, :pkgName4, :appName5, :pkgName5)")
                .setParameter("customerId", customerId)
                .setParameter("groupId", groupId)
                .setParameter("appVersion", appVersion)
                .setParameter("deviceDateStart", deviceDateStart)
                .setParameter("deviceDateEnd", deviceDateEnd)
                .setParameter("appUID", appUID)
                .setParameter("appName1", appName1)
                .setParameter("pkgName1", pkgName1)
                .setParameter("appName2", appName2)
                .setParameter("pkgName2", pkgName2)
                .setParameter("appName3", appName3)
                .setParameter("pkgName3", pkgName3)
                .setParameter("appName4", appName4)
                .setParameter("pkgName4", pkgName4)
                .setParameter("appName5", appName5)
                .setParameter("pkgName5", pkgName5);

        query.unwrap(NativeQuery.class)
                .addScalar("devicedate", StandardBasicTypes.DATE)
                .addScalar("devicecount", StandardBasicTypes.LONG)
                .addScalar("customerid", StandardBasicTypes.STRING)
                .addScalar("groupid", StandardBasicTypes.INTEGER)
                .setResultTransformer(Transformers.aliasToBean(AppUsageTopAppSP.class));
        
        return query.getResultList();
    }
}
```

### 필요한 import 추가

```java
import org.hibernate.transform.Transformers;
import org.hibernate.type.StandardBasicTypes;
```

위 코드는 Hibernate의 `StandardBasicTypes`를 사용하여 결과 세트의 각 컬럼에 대한 데이터 유형을 명시적으로 지정합니다. 이렇게 하면 Hibernate가 결과를 DTO 필드로 변환할 때 올바른 데이터 유형을 사용할 수 있습니다.

--------------------
첫 번째 방법에서 Hibernate의 `Transformers`를 사용하여 DTO로 매핑할 때 발생하는 `cannot resolve property access` 오류는 일반적으로 DTO 클래스의 필드와 데이터베이스 쿼리의 컬럼 이름이 일치하지 않기 때문에 발생합니다. 필드 이름과 쿼리 결과의 컬럼 이름이 정확히 일치하는지 확인해야 합니다.

여기서 문제가 되는 `devicedate` 필드를 예로 들어 해결 방법을 설명하겠습니다.

### 1. DTO 클래스 수정

DTO 클래스의 필드 이름이 쿼리 결과의 컬럼 이름과 일치하는지 확인합니다. 만약 컬럼 이름과 필드 이름이 다르면, `@Column` 어노테이션을 사용하여 매핑할 수 있습니다.

```java
import java.util.Date;

public class AppUsageTopAppSP {
    private Date devicedate;
    private Long devicecount;
    private String customerid;
    private Integer groupid;

    // Getters and setters

    public Date getDevicedate() {
        return devicedate;
    }

    public void setDevicedate(Date devicedate) {
        this.devicedate = devicedate;
    }

    public Long getDevicecount() {
        return devicecount;
    }

    public void setDevicecount(Long devicecount) {
        this.devicecount = devicecount;
    }

    public String getCustomerid() {
        return customerid;
    }

    public void setCustomerid(String customerid) {
        this.customerid = customerid;
    }

    public Integer getGroupid() {
        return groupid;
    }

    public void setGroupid(Integer groupid) {
        this.groupid = groupid;
    }

    // 다른 필드들에 대한 getter와 setter도 추가
}
```

### 2. 네이티브 쿼리와 Transformers 사용

`Transformers.aliasToBean()`을 사용할 때, 쿼리에서 반환하는 컬럼 이름과 DTO 필드 이름이 일치하는지 확인합니다. 여기서는 SQL 쿼리에서 별칭을 사용하여 컬럼 이름을 DTO 필드 이름과 일치시킵니다.

```java
import org.hibernate.query.NativeQuery;
import org.hibernate.transform.Transformers;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class YourService {

    @PersistenceContext
    private EntityManager entityManager;

    public List<AppUsageTopAppSP> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5) {
        Query query = entityManager.createNativeQuery(
                "SELECT devicedate, devicecount, customerid, groupid, ... " + // 필요한 컬럼들 추가
                "FROM kaiappinfo.fn_appusage_datausage_daily_00(:customerId, :groupId, :appVersion, :deviceDateStart, :deviceDateEnd, :appUID, :appName1, :pkgName1, :appName2, :pkgName2, :appName3, :pkgName3, :appName4, :pkgName4, :appName5, :pkgName5)")
                .setParameter("customerId", customerId)
                .setParameter("groupId", groupId)
                .setParameter("appVersion", appVersion)
                .setParameter("deviceDateStart", deviceDateStart)
                .setParameter("deviceDateEnd", deviceDateEnd)
                .setParameter("appUID", appUID)
                .setParameter("appName1", appName1)
                .setParameter("pkgName1", pkgName1)
                .setParameter("appName2", appName2)
                .setParameter("pkgName2", pkgName2)
                .setParameter("appName3", appName3)
                .setParameter("pkgName3", pkgName3)
                .setParameter("appName4", appName4)
                .setParameter("pkgName4", pkgName4)
                .setParameter("appName5", appName5)
                .setParameter("pkgName5", pkgName5);

        query.unwrap(NativeQuery.class)
                .addScalar("devicedate", DateType.INSTANCE)
                .addScalar("devicecount", LongType.INSTANCE)
                .addScalar("customerid", StringType.INSTANCE)
                .addScalar("groupid", IntegerType.INSTANCE)
                .setResultTransformer(Transformers.aliasToBean(AppUsageTopAppSP.class));
        
        return query.getResultList();
    }
}
```

여기서 `addScalar()` 메서드를 사용하여 결과의 각 컬럼을 명시적으로 지정하여 Hibernate가 결과를 DTO 필드로 변환할 수 있도록 합니다.

이 방법을 통해 필드 이름과 컬럼 이름이 일치하지 않아 발생하는 문제를 해결할 수 있습니다.

---------------------

`Transformers`는 Hibernate에서 사용되는 클래스입니다. 이를 사용하려면 Hibernate 관련 라이브러리를 포함해야 합니다. 또한, 결과가 `List<AppUsageTopAppSPProjection>`에 제대로 매핑되지 않는 문제는 Projection이 아닌 Entity 클래스를 사용하여 직접 DTO로 변환하는 방법으로 해결할 수 있습니다.

여기서는 `Transformers`를 사용하는 방법과 `JpaRepository`에서 네이티브 쿼리를 사용하여 결과를 매핑하는 방법을 모두 제시하겠습니다.

### 방법 1: 네이티브 쿼리와 Hibernate Transformers 사용

#### 의존성 추가

먼저, `pom.xml` 파일에 Hibernate 관련 의존성을 추가합니다.

```xml
<dependency>
    <groupId>org.hibernate</groupId>
    <artifactId>hibernate-core</artifactId>
    <version>5.4.32.Final</version>
</dependency>
```

#### 코드 구현

```java
import org.hibernate.query.NativeQuery;
import org.hibernate.transform.Transformers;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;
import org.springframework.stereotype.Service;
import java.util.List;

@Service
public class YourService {

    @PersistenceContext
    private EntityManager entityManager;

    public List<AppUsageTopAppSP> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5) {
        Query query = entityManager.createNativeQuery("SELECT * FROM kaiappinfo.fn_appusage_datausage_daily_00(:customerId, :groupId, :appVersion, :deviceDateStart, :deviceDateEnd, :appUID, :appName1, :pkgName1, :appName2, :pkgName2, :appName3, :pkgName3, :appName4, :pkgName4, :appName5, :pkgName5)")
                .setParameter("customerId", customerId)
                .setParameter("groupId", groupId)
                .setParameter("appVersion", appVersion)
                .setParameter("deviceDateStart", deviceDateStart)
                .setParameter("deviceDateEnd", deviceDateEnd)
                .setParameter("appUID", appUID)
                .setParameter("appName1", appName1)
                .setParameter("pkgName1", pkgName1)
                .setParameter("appName2", appName2)
                .setParameter("pkgName2", pkgName2)
                .setParameter("appName3", appName3)
                .setParameter("pkgName3", pkgName3)
                .setParameter("appName4", appName4)
                .setParameter("pkgName4", pkgName4)
                .setParameter("appName5", appName5)
                .setParameter("pkgName5", pkgName5);

        query.unwrap(NativeQuery.class)
                .setResultTransformer(Transformers.aliasToBean(AppUsageTopAppSP.class));
        
        return query.getResultList();
    }
}
```

### 방법 2: JPA 네이티브 쿼리와 Result Set Mapping 사용

#### Entity 클래스 및 Result Set Mapping 정의

```java
import javax.persistence.Entity;
import javax.persistence.Id;
import javax.persistence.SqlResultSetMapping;
import javax.persistence.EntityResult;
import javax.persistence.ColumnResult;

@Entity
@SqlResultSetMapping(
    name = "AppUsageTopAppSPMapping",
    entities = @EntityResult(
        entityClass = AppUsageTopAppSP.class,
        fields = {
            @FieldResult(name = "devicedate", column = "devicedate"),
            @FieldResult(name = "devicecount", column = "devicecount"),
            @FieldResult(name = "customerid", column = "customerid"),
            @FieldResult(name = "groupid", column = "groupid"),
            // 다른 필드들에 대해서도 마찬가지로 매핑합니다.
        }
    )
)
public class AppUsageTopAppSP {
    @Id
    private Long id;

    private Date devicedate;
    private Long devicecount;
    private String customerid;
    private Integer groupid;
    // 다른 필드들도 추가합니다.
}
```

#### Repository 인터페이스

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;

import java.util.List;

@Repository
public interface AppUsageRepository extends JpaRepository<AppUsageTopAppSP, Long> {

    @Query(value = "SELECT * FROM kaiappinfo.fn_appusage_datausage_daily_00(:customerId, :groupId, :appVersion, :deviceDateStart, :deviceDateEnd, :appUID, :appName1, :pkgName1, :appName2, :pkgName2, :appName3, :pkgName3, :appName4, :pkgName4, :appName5, :pkgName5)", nativeQuery = true)
    List<AppUsageTopAppSP> callYourStoredProcedure(@Param("customerId") String customerId,
                                                    @Param("groupId") Long groupId,
                                                    @Param("appVersion") String appVersion,
                                                    @Param("deviceDateStart") String deviceDateStart,
                                                    @Param("deviceDateEnd") String deviceDateEnd,
                                                    @Param("appUID") Long appUID,
                                                    @Param("appName1") String appName1,
                                                    @Param("pkgName1") String pkgName1,
                                                    @Param("appName2") String appName2,
                                                    @Param("pkgName2") String pkgName2,
                                                    @Param("appName3") String appName3,
                                                    @Param("pkgName3") String pkgName3,
                                                    @Param("appName4") String appName4,
                                                    @Param("pkgName4") String pkgName4,
                                                    @Param("appName5") String appName5,
                                                    @Param("pkgName5") String pkgName5);
}
```

### 서비스에서 호출

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

import java.util.List;

@Service
public class YourService {

    @Autowired
    private AppUsageRepository appUsageRepository;

    public List<AppUsageTopAppSP> getAppUsageData(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5) {
        return appUsageRepository.callYourStoredProcedure(customerId, groupId, appVersion, deviceDateStart, deviceDateEnd, appUID, appName1, pkgName1, appName2, pkgName2, appName3, pkgName3, appName4, pkgName4, appName5, pkgName5);
    }
}
```

이 두 가지 방법을 통해 원하는 결과를 얻을 수 있습니다. 첫 번째 방법은 Hibernate의 `Transformers`를 사용하여 직접 DTO로 매핑하는 방법이고, 두 번째 방법은 JPA 네이티브 쿼리와 Result Set Mapping을 사용하는 방법입니다. 적합한 방법을 선택하여 사용하시기 바랍니다.

-------------------------------

이 오류는 JPA 쿼리 결과를 원하는 DTO 클래스(`AppUsageTopAppSP`)로 변환할 수 없음을 의미합니다. 이 문제를 해결하기 위해 다음 두 가지 방법 중 하나를 사용할 수 있습니다.

### 방법 1: 네이티브 쿼리와 Result Transformer 사용

네이티브 쿼리를 사용하고 결과를 원하는 DTO로 매핑하는 방법입니다. 이 방법은 쿼리 결과를 명시적으로 DTO로 변환하기 때문에 `ConverterNotFoundException` 문제를 해결할 수 있습니다.

```java
import org.hibernate.transform.Transformers;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;
import java.util.List;

@Service
public class YourService {

    @PersistenceContext
    private EntityManager entityManager;

    public List<AppUsageTopAppSP> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5) {
        Query query = entityManager.createNativeQuery("SELECT * FROM kaiappinfo.fn_appusage_datausage_daily_00(:customerId, :groupId, :appVersion, :deviceDateStart, :deviceDateEnd, :appUID, :appName1, :pkgName1, :appName2, :pkgName2, :appName3, :pkgName3, :appName4, :pkgName4, :appName5, :pkgName5)")
                .setParameter("customerId", customerId)
                .setParameter("groupId", groupId)
                .setParameter("appVersion", appVersion)
                .setParameter("deviceDateStart", deviceDateStart)
                .setParameter("deviceDateEnd", deviceDateEnd)
                .setParameter("appUID", appUID)
                .setParameter("appName1", appName1)
                .setParameter("pkgName1", pkgName1)
                .setParameter("appName2", appName2)
                .setParameter("pkgName2", pkgName2)
                .setParameter("appName3", appName3)
                .setParameter("pkgName3", pkgName3)
                .setParameter("appName4", appName4)
                .setParameter("pkgName4", pkgName4)
                .setParameter("appName5", appName5)
                .setParameter("pkgName5", pkgName5);

        query.unwrap(org.hibernate.query.NativeQuery.class)
                .setResultTransformer(Transformers.aliasToBean(AppUsageTopAppSP.class));
        
        return query.getResultList();
    }
}
```

### 방법 2: JPA Projection 사용

JPA Projection을 사용하여 인터페이스 기반의 DTO를 사용할 수도 있습니다. 이렇게 하면 자동으로 매핑됩니다.

```java
public interface AppUsageTopAppSPProjection {
    Date getDevicedate();
    Long getDevicecount();
    String getCustomerid();
    Integer getGroupid();
    String getGroupname();
    String getAppname();
    String getPackagename();
    String getAppversion();
    Integer getAppuid();
    Double getAvgbatteryusagepercentage();
    Double getAvgbackgroundtime();
    Double getAvgforegroundtime();
    Double getAvgtotaltime();
    Long getDevicestotalbatteryusagepercentage();
    Long getDevicestotalbackgroundtime();
    Long getDevicestotalforegroundtime();
    Long getDevicestotalusagetime();
    Double getNormalizedbatteryconsumption();
    Double getAvgnetworkusage();
    Double getAvgmobileusage();
    Double getAvgwifiusage();
    Long getDevicestotalmobileusage();
    Long getDevicestotalmobileforegroundusage();
    Long getDevicestotalmobilebackgroundusage();
    Long getDevicestotalwifiusage();
    Long getDevicestotalwififoregroundusage();
    Long getDevicestotalwifibackgroundusage();
    Long getDevicestotalnetworkusage();
}

public interface YourRepository extends JpaRepository<YourEntity, Long> {

    @Query(value = "SELECT * FROM kaiappinfo.fn_appusage_datausage_daily_00(?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9, ?10, ?11, ?12, ?13, ?14, ?15)", nativeQuery = true)
    List<AppUsageTopAppSPProjection> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5);
}
```

### 요약

위의 두 가지 방법 중 하나를 사용하여 문제를 해결할 수 있을 것입니다. 첫 번째 방법은 네이티브 쿼리와 Hibernate Transformer를 사용하여 결과를 DTO로 매핑하는 방법이고, 두 번째 방법은 JPA Projection을 사용하여 인터페이스 기반의 DTO를 자동으로 매핑하는 방법입니다.

--------------------------
JPA를 사용하여 PostgreSQL의 저장 프로시저를 호출할 때 문제를 해결하기 위해 `@Procedure`를 사용할 때 발생하는 문제는 여러 가지가 있을 수 있습니다. 저장 프로시저 호출에서 스키마와 관련된 문제를 해결하고 제대로 동작하도록 하기 위해 다음과 같은 몇 가지 방법을 시도해 볼 수 있습니다.

### 방법 1: `@Query`와 네이티브 쿼리 사용

스프링 데이터 JPA에서 `@Query` 애노테이션을 사용하여 네이티브 쿼리를 직접 실행할 수 있습니다. 이는 스키마 문제를 회피하고 직접적으로 저장 프로시저를 호출하는 방법입니다.

```java
public interface YourRepository extends JpaRepository<YourEntity, Long> {

    @Query(value = "SELECT * FROM kaiappinfo.fn_appusage_datausage_daily_00(?1, ?2, ?3, ?4, ?5, ?6, ?7, ?8, ?9, ?10, ?11, ?12, ?13, ?14, ?15)", nativeQuery = true)
    List<YourEntity> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5);
}
```

이 방법은 `@Procedure` 대신에 네이티브 SQL 쿼리를 사용하여 저장 프로시저를 호출하는 방법입니다.

### 방법 2: `StoredProcedureQuery`를 사용한 JPA 저장 프로시저 호출

`EntityManager`를 사용하여 `StoredProcedureQuery`를 생성하고 실행할 수 있습니다. 이는 JPA에서 제공하는 방식으로, 보다 세밀하게 제어할 수 있습니다.

```java
import javax.persistence.EntityManager;
import javax.persistence.ParameterMode;
import javax.persistence.PersistenceContext;
import javax.persistence.StoredProcedureQuery;
import java.util.List;

@Service
public class YourService {

    @PersistenceContext
    private EntityManager entityManager;

    public List<Object[]> callYourStoredProcedure(String customerId, Long groupId, String appVersion, String deviceDateStart, String deviceDateEnd, Long appUID, String appName1, String pkgName1, String appName2, String pkgName2, String appName3, String pkgName3, String appName4, String pkgName4, String appName5, String pkgName5) {
        StoredProcedureQuery query = entityManager.createStoredProcedureQuery("kaiappinfo.fn_appusage_datausage_daily_00");
        query.registerStoredProcedureParameter(1, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(2, Long.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(3, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(4, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(5, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(6, Long.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(7, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(8, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(9, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(10, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(11, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(12, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(13, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(14, String.class, ParameterMode.IN);
        query.registerStoredProcedureParameter(15, String.class, ParameterMode.IN);
        
        query.setParameter(1, customerId);
        query.setParameter(2, groupId);
        query.setParameter(3, appVersion);
        query.setParameter(4, deviceDateStart);
        query.setParameter(5, deviceDateEnd);
        query.setParameter(6, appUID);
        query.setParameter(7, appName1);
        query.setParameter(8, pkgName1);
        query.setParameter(9, appName2);
        query.setParameter(10, pkgName2);
        query.setParameter(11, appName3);
        query.setParameter(12, pkgName3);
        query.setParameter(13, appName4);
        query.setParameter(14, pkgName4);
        query.setParameter(15, appName5);
        
        query.execute();
        
        return query.getResultList();
    }
}
```

이 방법은 `EntityManager`를 사용하여 저장 프로시저를 호출하고 결과를 처리하는 방법입니다. 

### 요약

위의 두 가지 방법 중 하나를 사용하여 문제를 해결할 수 있을 것입니다. 첫 번째 방법은 간단하게 네이티브 쿼리를 사용하여 저장 프로시저를 호출하는 것이고, 두 번째 방법은 `StoredProcedureQuery`를 사용하여 더 세밀하게 제어할 수 있는 방법입니다. 권한 문제를 해결했음에도 불구하고 `@Procedure`가 제대로 동작하지 않는다면 이 방법들을 사용하여 문제를 해결할 수 있을 것입니다.

-----------------------------
`jdbcTemplate.execute("SELECT 1");`는 단순히 쿼리를 실행하기 위한 명령으로, 쿼리 결과를 반환하지 않습니다. 쿼리 결과를 확인하려면 `JdbcTemplate`의 `queryForObject`나 `query` 메서드를 사용해야 합니다.

다음은 `queryForObject`를 사용하여 간단한 쿼리 결과를 확인하는 방법입니다:

### 예제: 간단한 쿼리 결과 확인

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

@Service
public class DatabaseService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public Integer testConnection() {
        return jdbcTemplate.queryForObject("SELECT 1", Integer.class);
    }
}
```

이 코드는 `SELECT 1` 쿼리를 실행하고 결과를 `Integer`로 반환합니다. 

### 예제: 저장 프로시저 호출 및 결과 확인

저장 프로시저를 호출하고 그 결과를 확인하려면 `JdbcTemplate`의 `query` 메서드를 사용합니다. 예를 들어, `hello_world_with_return` 저장 프로시저를 호출하고 결과를 확인하는 방법은 다음과 같습니다:

#### 1. 저장 프로시저 정의 (다시 확인)

```sql
CREATE OR REPLACE FUNCTION hello_world_with_return()
RETURNS TABLE(message TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT 'Hello, World!' AS message;
END;
$$;
```

#### 2. Spring Boot 서비스 클래스에서 저장 프로시저 호출 및 결과 처리

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Service;

import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.List;

@Service
public class HelloWorldService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public List<String> callHelloWorldProcedure() {
        String sql = "SELECT * FROM hello_world_with_return()";
        return jdbcTemplate.query(sql, new RowMapper<String>() {
            @Override
            public String mapRow(ResultSet rs, int rowNum) throws SQLException {
                return rs.getString("message");
            }
        });
    }
}
```

#### 3. 컨트롤러에서 서비스 호출

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api")
public class HelloWorldController {

    @Autowired
    private HelloWorldService helloWorldService;

    @GetMapping("/hello")
    public List<String> sayHello() {
        return helloWorldService.callHelloWorldProcedure();
    }
}
```

### 테스트 및 결과 확인

1. **서버 실행**: Spring Boot 애플리케이션을 실행합니다.
2. **API 호출**: `http://localhost:8080/api/hello`를 호출하여 저장 프로시저를 실행합니다.
3. **결과 확인**: 반환된 JSON 응답에서 "Hello, World!" 메시지를 확인할 수 있습니다.

이 예제는 저장 프로시저의 반환값을 처리하고 결과를 클라이언트에게 반환하는 방법을 보여줍니다. 이러한 방법을 사용하여 스프링 애플리케이션에서 저장 프로시저 호출 및 결과 확인을 할 수 있습니다.

-------------------------

네, 권한 문제로 인해 저장 프로시저 호출에서 `does not exist` 오류가 발생할 수 있습니다. 하지만 권한 문제로 인해 발생하는 일반적인 오류와는 약간의 차이가 있을 수 있습니다. 권한 문제로 인해 발생할 수 있는 오류는 보통 다음과 같습니다:

1. **권한 부족 오류 (Permission Denied)**:
   - **오류 메시지**: `permission denied for procedure <procedure_name>`
   - **설명**: 이 오류는 현재 사용자에게 해당 저장 프로시저를 실행할 권한이 없을 때 발생합니다.

2. **저장 프로시저 또는 함수 존재 여부 오류**:
   - **오류 메시지**: `procedure <procedure_name> does not exist` 또는 `function <function_name> does not exist`
   - **설명**: 저장 프로시저가 정의되어 있지 않거나, 호출할 수 있는 권한이 없는 경우 발생합니다. 이 경우에는 프로시저가 스키마 내에 존재하는지 확인해야 합니다.

### 권한 문제를 확인하고 해결하는 방법

1. **프로시저 권한 확인**:
   - PostgreSQL에서 저장 프로시저의 권한을 확인하려면 다음 쿼리를 사용할 수 있습니다:
     ```sql
     SELECT * 
     FROM pg_proc
     JOIN pg_namespace ON pg_proc.pronamespace = pg_namespace.oid
     WHERE pg_proc.proname = 'your_procedure_name';
     ```
   - 이 쿼리는 저장 프로시저의 정보를 반환합니다. 사용자가 해당 프로시저를 실행할 권한이 있는지 확인합니다.

2. **프로시저 권한 부여**:
   - 권한이 부족한 경우, 관리자 계정에서 다음 쿼리를 사용하여 권한을 부여할 수 있습니다:
     ```sql
     GRANT EXECUTE ON FUNCTION your_procedure_name TO your_user;
     ```
   - 이 쿼리는 특정 사용자에게 저장 프로시저를 실행할 권한을 부여합니다.

3. **스키마 설정 확인**:
   - 저장 프로시저가 정의된 스키마가 애플리케이션의 기본 스키마와 일치하는지 확인합니다. 기본 스키마를 설정하려면 `application.yml` 파일에 스키마를 명시합니다:
     ```yaml
     spring:
       datasource:
         url: jdbc:postgresql://localhost:5432/your_database
         username: your_username
         password: your_password
         driver-class-name: org.postgresql.Driver
         default-schema: kaiappinfo
     ```
   - **주의**: 스키마 이름이 `default-schema`로 올바르게 설정되었는지 확인합니다.

4. **스프링의 데이터베이스 연결 테스트**:
   - 스프링에서 직접 데이터베이스 연결이 가능한지 확인하려면, `JdbcTemplate`을 사용하여 간단한 쿼리를 실행해보는 것도 도움이 됩니다. 예를 들어, 다음과 같은 간단한 쿼리로 연결을 테스트할 수 있습니다:
     ```java
     @Autowired
     private JdbcTemplate jdbcTemplate;

     public void testConnection() {
         jdbcTemplate.execute("SELECT 1");
     }
     ```

이와 같은 접근 방법을 사용하여 저장 프로시저 호출과 관련된 권한 문제를 해결할 수 있습니다. 권한이 문제의 원인인 경우, 위의 단계를 통해 확인하고 수정할 수 있습니다.

----------------------------

저장 프로시저의 반환값을 보여주고 싶다면, 저장 프로시저에서 반환값을 `OUT` 파라미터로 설정하거나 `RESULT SET`을 반환해야 합니다. `JdbcTemplate`을 사용하여 이러한 반환값을 처리하는 방법을 아래에 설명합니다.

### 1. **저장 프로시저 정의**

저장 프로시저가 `OUT` 파라미터를 사용하여 값을 반환하도록 정의합니다. PostgreSQL에서 `OUT` 파라미터를 사용하는 간단한 저장 프로시저를 작성해 보겠습니다.

```sql
CREATE OR REPLACE FUNCTION hello_world_with_return()
RETURNS TABLE(message TEXT)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT 'Hello, World!' AS message;
END;
$$;
```

이 저장 프로시저는 `message`라는 `TEXT` 형식의 값을 반환합니다.

### 2. **Spring Boot 애플리케이션에서 저장 프로시저 호출**

Spring Boot에서 `JdbcTemplate`을 사용하여 저장 프로시저를 호출하고 반환값을 처리하는 방법을 설명합니다.

#### **2.1. 저장 프로시저 호출 및 반환값 처리**

`JdbcTemplate`을 사용하여 저장 프로시저를 호출하고 반환된 결과를 처리합니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.CallableStatementCreator;
import org.springframework.jdbc.core.CallableStatementCallback;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.stereotype.Service;

import java.sql.CallableStatement;
import java.sql.Connection;
import java.sql.ResultSet;
import java.sql.SQLException;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;

@Service
public class HelloWorldService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public List<String> callHelloWorldProcedure() {
        return jdbcTemplate.execute(
            (CallableStatementCreator) conn -> {
                CallableStatement cs = conn.prepareCall("{ ? = call hello_world_with_return() }");
                cs.registerOutParameter(1, java.sql.Types.REF_CURSOR);
                return cs;
            },
            (CallableStatementCallback<List<String>>) cs -> {
                cs.execute();
                ResultSet rs = (ResultSet) cs.getObject(1);
                List<String> messages = new ArrayList<>();
                while (rs.next()) {
                    messages.add(rs.getString("message"));
                }
                return messages;
            }
        );
    }
}
```

#### **2.2. 컨트롤러에서 서비스 호출**

컨트롤러에서 `HelloWorldService`를 호출하여 저장 프로시저의 반환값을 클라이언트에게 반환합니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;

@RestController
@RequestMapping("/api")
public class HelloWorldController {

    @Autowired
    private HelloWorldService helloWorldService;

    @GetMapping("/hello")
    public List<String> sayHello() {
        return helloWorldService.callHelloWorldProcedure();
    }
}
```

### 3. **테스트 및 결과 확인**

1. **서버 실행**: Spring Boot 애플리케이션을 실행합니다.
2. **API 호출**: `http://localhost:8080/api/hello`를 호출하여 저장 프로시저를 실행합니다.
3. **결과 확인**: 반환된 JSON 응답에서 "Hello, World!" 메시지를 확인할 수 있습니다.

이 예제는 저장 프로시저의 반환값을 `ResultSet` 형태로 처리하고, 이를 클라이언트에 JSON 형식으로 반환하는 방법을 보여줍니다. 저장 프로시저의 반환값이 `OUT` 파라미터로 제공될 경우, `CallableStatement`의 `getXXX` 메소드를 사용하여 값을 읽을 수 있습니다.

-----------------------
다른 데이터베이스 테이블들은 잘 불러와지는데 특정 저장 프로시저만 제대로 호출되지 않는 경우, 다음과 같은 문제를 점검해볼 수 있습니다:

### 1. 저장 프로시저 이름 및 스키마 확인
저장 프로시저 이름과 스키마가 정확하게 지정되었는지 확인합니다. 특히, 스키마와 저장 프로시저 이름이 정확히 일치하는지 검토하세요.

- 저장 프로시저 이름 및 스키마가 정확한지 확인합니다.
- `@Procedure` 애노테이션의 `procedureName` 속성이 저장 프로시저의 전체 이름과 일치하는지 확인합니다.

```java
@Procedure(procedureName = "kaiappinfo.fn_appusage_datausage_daily_00")
List<AppUsageTopApp> getAppUsageDataUsageDaily_00(...);
```

### 2. 저장 프로시저 파라미터 및 반환 타입 확인
저장 프로시저의 파라미터와 반환 타입이 Java 메소드와 일치하는지 확인합니다.

- 프로시저가 사용하는 파라미터의 순서와 타입이 Java 메소드의 파라미터와 일치해야 합니다.
- 프로시저의 반환 타입이 Java 메소드에서 처리 가능한 타입인지 확인합니다.

### 3. 저장 프로시저 권한 확인
저장 프로시저를 호출하는 데이터베이스 사용자에게 적절한 권한이 부여되어 있는지 확인합니다.

```sql
GRANT EXECUTE ON FUNCTION kaiappinfo.fn_appusage_datausage_daily_00 TO your_user_name;
```

### 4. 저장 프로시저의 존재 여부 확인
저장 프로시저가 데이터베이스에 실제로 존재하는지 확인합니다.

```sql
SELECT proname
FROM pg_proc
WHERE proname = 'fn_appusage_datausage_daily_00'
AND pronamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'kaiappinfo');
```

### 5. 직접 JDBC를 통한 테스트
Spring Data JPA를 사용하지 않고, `JdbcTemplate`을 이용하여 저장 프로시저를 호출해 보세요. 이는 Spring Data JPA의 문제를 분리하여 확인하는 데 도움이 됩니다.

#### JdbcTemplate을 사용한 예제
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.CallableStatementCreator;
import org.springframework.jdbc.core.CallableStatementCallback;
import org.springframework.jdbc.core.CallableStatement;
import org.springframework.stereotype.Service;

import java.sql.CallableStatement;
import java.sql.Connection;
import java.sql.SQLException;
import java.sql.Types;
import java.util.HashMap;
import java.util.Map;

@Service
public class TestService {

    @Autowired
    private JdbcTemplate jdbcTemplate;

    public Map<String, Object> callStoredProc() {
        return jdbcTemplate.execute(
            (CallableStatementCreator) conn -> {
                CallableStatement cs = conn.prepareCall("{ call kaiappinfo.fn_appusage_datausage_daily_00(?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?, ?) }");
                cs.setString(1, "customerId");
                cs.setLong(2, 123L);
                cs.setString(3, "appVersion");
                cs.setString(4, "startdate");
                cs.setString(5, "enddate");
                cs.setLong(6, 456L);
                // set other parameters as needed
                return cs;
            },
            (CallableStatementCallback<Map<String, Object>>) cs -> {
                cs.execute();
                Map<String, Object> resultMap = new HashMap<>();
                // Process result set or output parameters here
                return resultMap;
            }
        );
    }
}
```

### 6. 로그 확인 및 디버깅
애플리케이션 로그 레벨을 DEBUG로 설정하고 저장 프로시저 호출 시 로그를 확인합니다. 이를 통해 오류 메시지나 스택 트레이스를 확인할 수 있습니다.

#### `application.yml`에서 로그 레벨 설정
```yaml
logging:
  level:
    org.springframework: DEBUG
    org.hibernate: DEBUG
```

### 7. 스프링 JPA 버전 및 JDBC 드라이버 확인
사용 중인 Spring JPA와 JDBC 드라이버의 버전이 호환되는지 확인합니다. 때로는 드라이버의 버전 불일치로 인해 문제가 발생할 수 있습니다.

### 8. Spring Boot 자동 설정 확인
Spring Boot가 자동으로 설정하는 데이터 소스와 관련된 설정이 적절하게 구성되어 있는지 확인합니다.

이 단계를 통해 문제를 해결할 수 있을 것입니다. 저장 프로시저와의 연결이 다른 데이터베이스 테이블과 다르게 동작하는 이유를 파악하고 문제를 해결해보세요.
--------------------------

저장 프로시저가 존재하지 않는다는 오류가 계속 발생하는 경우, 몇 가지 가능한 원인을 고려해볼 수 있습니다. 아래는 문제를 해결하기 위해 확인해 볼 몇 가지 단계입니다.

### 1. 스키마 지정 확인
저장 프로시저 호출 시 스키마를 명시적으로 지정합니다. 예를 들어, `@Procedure` 어노테이션을 사용할 때 스키마를 포함하여 완전한 이름을 사용합니다.

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.query.Procedure;
import org.springframework.stereotype.Repository;

@Repository
public interface TestRepository extends JpaRepository<SomeEntity, Long> {

    @Procedure("kaiappinfo.hello_world")
    String callHelloWorld();
}
```

### 2. 데이터베이스 연결 및 기본 스키마 설정 확인
`application.yml` 또는 `application.properties` 파일에서 기본 스키마 설정을 올바르게 구성합니다.

#### application.yml 예시
```yaml
spring:
  datasource:
    url: jdbc:postgresql://your-db-host:your-db-port/your-db-name
    username: your-db-username
    password: your-db-password
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        default_schema: kaiappinfo
```

### 3. 기본 스키마 설정 후 명시적으로 스키마 지정
기본 스키마를 설정하더라도, 프로시저 호출 시 명시적으로 스키마를 지정해보십시오.

### 4. SQL 쿼리로 직접 확인
애플리케이션 외부에서 데이터베이스에 직접 연결하여 저장 프로시저가 제대로 생성되고 호출되는지 확인합니다.

```sql
SET search_path TO kaiappinfo;

SELECT kaiappinfo.hello_world();
```

### 5. 데이터베이스 사용자의 권한 확인
데이터베이스 사용자가 해당 스키마와 저장 프로시저에 대한 권한이 있는지 확인합니다.

```sql
GRANT EXECUTE ON FUNCTION kaiappinfo.hello_world TO your_user_name;
```

### 6. 저장 프로시저의 존재 여부 확인
`pg_proc` 테이블에서 저장 프로시저가 존재하는지 확인합니다.

```sql
SELECT * 
FROM pg_proc 
WHERE proname = 'hello_world' 
AND pronamespace = (SELECT oid FROM pg_namespace WHERE nspname = 'kaiappinfo');
```

### 예제 코드

간단한 저장 프로시저를 호출하는 예제입니다.

#### 간단한 저장 프로시저 생성
```sql
CREATE OR REPLACE FUNCTION kaiappinfo.hello_world()
RETURNS text AS $$
BEGIN
    RETURN 'Hello, World!';
END;
$$ LANGUAGE plpgsql;
```

#### Repository 인터페이스
```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.query.Procedure;
import org.springframework.stereotype.Repository;

@Repository
public interface TestRepository extends JpaRepository<SomeEntity, Long> {

    @Procedure("kaiappinfo.hello_world")
    String callHelloWorld();
}
```

#### 서비스 계층
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class TestService {

    @Autowired
    private TestRepository testRepository;

    public String callHelloWorld() {
        return testRepository.callHelloWorld();
    }
}
```

#### 테스트 코드
```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringBootTest
public class StoredProcTest {

    @Autowired
    private TestService testService;

    @Test
    public void testCallHelloWorld() {
        String result = testService.callHelloWorld();
        assertEquals("Hello, World!", result);
    }
}
```

### 요약
이 단계들을 통해 저장 프로시저가 올바르게 인식되지 않는 문제를 해결할 수 있습니다. 스키마와 사용자 권한, 그리고 저장 프로시저의 존재 여부를 꼼꼼히 확인하는 것이 중요합니다.
--------------------

Spring 애플리케이션에서 저장 프로시저가 제대로 연결되고 호출되는지 간단하게 테스트하려면, 다음과 같은 방법을 사용할 수 있습니다. 

### 1. 간단한 저장 프로시저 생성
먼저, 테스트를 위해 간단한 저장 프로시저를 생성합니다. 예를 들어, 간단한 "Hello, World!" 메시지를 반환하는 저장 프로시저를 만들어 보겠습니다.

```sql
CREATE OR REPLACE FUNCTION kaiappinfo.hello_world()
RETURNS text AS $$
BEGIN
    RETURN 'Hello, World!';
END;
$$ LANGUAGE plpgsql;
```

### 2. Spring Repository 설정
이제 Spring 애플리케이션에서 저장 프로시저를 호출하는 간단한 리포지토리를 만듭니다.

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.query.Procedure;
import org.springframework.stereotype.Repository;

@Repository
public interface TestRepository extends JpaRepository<SomeEntity, Long> {

    @Procedure(name = "kaiappinfo.hello_world")
    String callHelloWorld();
}
```

`SomeEntity`는 테스트를 위한 임의의 엔티티입니다. 저장 프로시저를 테스트하는 데 실제 데이터베이스 엔티티는 필요하지 않습니다.

### 3. Service 및 Test 코드 작성
이제 서비스 계층과 테스트 코드를 작성합니다.

#### 서비스 계층
```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class TestService {

    @Autowired
    private TestRepository testRepository;

    public String callHelloWorld() {
        return testRepository.callHelloWorld();
    }
}
```

#### 테스트 코드
```java
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;

import static org.junit.jupiter.api.Assertions.assertEquals;

@SpringBootTest
public class StoredProcTest {

    @Autowired
    private TestService testService;

    @Test
    public void testCallHelloWorld() {
        String result = testService.callHelloWorld();
        assertEquals("Hello, World!", result);
    }
}
```

### 4. Spring Boot Application 설정
`application.yml` 또는 `application.properties`에서 데이터베이스 설정을 적절히 구성합니다.

#### application.yml 예시
```yaml
spring:
  datasource:
    url: jdbc:postgresql://your-db-host:your-db-port/your-db-name
    username: your-db-username
    password: your-db-password
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: none
    properties:
      hibernate:
        default_schema: kaiappinfo
```

### 5. 실행 및 확인
위의 테스트 코드를 실행하면, 저장 프로시저가 제대로 호출되고 "Hello, World!" 메시지를 반환하는지 확인할 수 있습니다.

이렇게 간단한 예제를 통해 저장 프로시저가 스프링 애플리케이션에서 올바르게 호출되는지 테스트할 수 있습니다. 기존의 복잡한 저장 프로시저로 테스트하기 전에, 간단한 프로시저를 통해 기본 연결과 호출이 문제 없는지 확인하는 것이 좋습니다.

------------------------

네, 일반 테이블에는 접근할 수 있지만, 저장 프로시저(Stored Procedure, SP)에 접근할 수 없는 경우가 있을 수 있습니다. 이는 데이터베이스 사용자의 권한 설정에 따라 다릅니다.

PostgreSQL에서 특정 사용자에게 저장 프로시저에 대한 권한이 부여되지 않은 경우, 그 사용자는 해당 프로시저를 실행할 수 없습니다. 저장 프로시저에 대한 접근 권한이 제대로 설정되어 있는지 확인하는 것이 필요합니다.

### 권한 확인 및 부여 방법

1. **권한 확인**:
   먼저, 데이터베이스 관리자로 로그인한 상태에서 다음 쿼리를 실행하여 저장 프로시저에 대한 권한을 확인합니다.

   ```sql
   \df+ kaiappinfo.fn_appusage_datausage_daily_00
   ```

2. **권한 부여**:
   만약 특정 사용자에게 저장 프로시저에 대한 실행 권한이 없다면, 다음과 같은 명령을 사용하여 권한을 부여할 수 있습니다.

   ```sql
   GRANT EXECUTE ON FUNCTION kaiappinfo.fn_appusage_datausage_daily_00 TO your_user_name;
   ```

   여기서 `your_user_name`은 저장 프로시저를 실행할 데이터베이스 사용자입니다.

### 예제

```sql
-- kaiappinfo 스키마의 모든 저장 프로시저에 대해 권한을 부여
GRANT EXECUTE ON ALL FUNCTIONS IN SCHEMA kaiappinfo TO your_user_name;
```

### 참고 사항

1. **스키마 지정**: 저장 프로시저 호출 시 스키마를 명시적으로 지정하는 것이 좋습니다. 예를 들어 `@Procedure("kaiappinfo.fn_appusage_datausage_daily_00")`와 같이 스키마를 명시적으로 지정합니다.

2. **연결 설정**: `application.yml` 또는 `application.properties`에서 기본 스키마를 설정하여 데이터베이스 연결 시 해당 스키마를 기본으로 사용하도록 설정합니다.

### 권한 확인 방법 예제

PostgreSQL에서 특정 사용자가 저장 프로시저에 접근할 수 있는지 확인하려면 다음 명령을 사용합니다:

```sql
-- 저장 프로시저의 권한 정보 확인
SELECT 
  n.nspname as "Schema",
  p.proname as "Name",
  pg_catalog.pg_get_function_result(p.oid) as "Result data type",
  pg_catalog.pg_get_function_arguments(p.oid) as "Argument data types",
  CASE
    WHEN p.proisagg THEN 'agg'
    WHEN p.proiswindow THEN 'window'
    WHEN p.prorettype = 'pg_catalog.trigger'::pg_catalog.regtype THEN 'trigger'
    ELSE 'normal'
  END as "Type",
  r.rolname as "Owner",
  p.proacl
FROM 
  pg_catalog.pg_proc p
     LEFT JOIN pg_catalog.pg_namespace n ON n.oid = p.pronamespace
     LEFT JOIN pg_catalog.pg_roles r ON r.oid = p.proowner
WHERE 
  pg_catalog.pg_function_is_visible(p.oid)
  AND n.nspname <> 'pg_catalog'
  AND n.nspname <> 'information_schema'
  AND n.nspname = 'kaiappinfo'
ORDER BY 
  1, 2, 4;
```

위 방법들을 통해 데이터베이스 사용자에게 필요한 권한을 부여하고, 스키마 설정 및 프로시저 호출 방식을 확인하여 문제를 해결할 수 있습니다.

--------------------------
이 에러는 JPA에서 호출한 저장 프로시저의 이름이나 매개변수 타입이 PostgreSQL에 정의된 저장 프로시저와 일치하지 않기 때문에 발생합니다. 몇 가지 점검할 사항을 통해 이를 해결해보겠습니다.

### 1. 매개변수 타입 일치 여부 점검
PostgreSQL 저장 프로시저 정의에서 매개변수의 타입이 JPA 리포지토리에서 호출한 타입과 일치하는지 확인해야 합니다. 예를 들어, 저장 프로시저에서 `bigint` 타입을 사용했다면 JPA 리포지토리에서도 `Long` 타입을 사용해야 합니다.

### 2. 명시적 타입 캐스팅 추가
매개변수 타입이 일치하지 않는 경우, 명시적 타입 캐스팅을 추가하여 호출합니다.

### 3. 프로시저 네임스페이스 확인
저장 프로시저 이름에 네임스페이스가 포함되어 있는지 확인합니다.

### 예제 수정
1. 저장 프로시저 정의 확인:

```sql
CREATE OR REPLACE FUNCTION kaiappinfo.fn_appusage_datausage_daily_00(
    cust_id character varying,
    grp_id bigint,
    app_ver character varying,
    startdate character varying,
    enddate character varying,
    auid bigint,
    appname1 character varying,
    packagename1 character varying,
    appname2 character varying,
    packagename2 character varying,
    appname3 character varying,
    packagename3 character varying,
    appname4 character varying,
    packagename4 character varying,
    appname5 character varying,
    packagename5 character varying
)
RETURNS TABLE(
    devicedate date,
    devicecount bigint,
    customerid character varying,
    groupid integer,
    groupname character varying,
    appname character varying,
    packagename character varying,
    appversion character varying,
    appuid integer,
    avgbatteryusagepercentage double precision,
    avgbackgroundtime double precision,
    avgforegroundtime double precision,
    avgtotaltime double precision,
    devicestotalbatteryusagepercentage bigint,
    devicestotalbackgroundtime bigint,
    devicestotalforegroundtime bigint,
    devicestotalusagetime bigint,
    normalizedbatteryconsumption double precision,
    avgnetworkusage double precision,
    avgmobileusage double precision,
    avgwifiusage double precision,
    devicestotalmobileusage bigint,
    devicestotalmobileforegroundusage bigint,
    devicestotalmobilebackgroundusage bigint,
    devicestotalwifiusage bigint,
    devicestotalwififoregroundusage bigint,
    devicestotalwifibackgroundusage bigint,
    devicestotalnetworkusage bigint
)
AS $$
BEGIN
    -- Function implementation here
END;
$$ LANGUAGE plpgsql;
```

2. JPA 리포지토리 수정:

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.query.Procedure;
import org.springframework.stereotype.Repository;
import java.util.List;

@Repository
public interface AppUsageRepository extends JpaRepository<AppUsageData, Long> {

    @Procedure("kaiappinfo.fn_appusage_datausage_daily_00")
    List<AppUsageData> getAppUsageDataUsageDaily_00(
        String cust_id, Long grp_id, String app_ver, String startdate, 
        String enddate, Long auid, String appname1, String packagename1, 
        String appname2, String packagename2, String appname3, String packagename3, 
        String appname4, String packagename4, String appname5, String packagename5
    );
}
```

3. 서비스 클래스:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import java.util.List;

@Service
public class AppUsageService {

    @Autowired
    private AppUsageRepository appUsageRepository;

    @Transactional
    public List<AppUsageData> getAppUsageDataUsageDaily_00(
        String cust_id, Long grp_id, String app_ver, String startdate, 
        String enddate, Long auid, String appname1, String packagename1, 
        String appname2, String packagename2, String appname3, String packagename3, 
        String appname4, String packagename4, String appname5, String packagename5
    ) {
        return appUsageRepository.getAppUsageDataUsageDaily_00(
            cust_id, grp_id, app_ver, startdate, enddate, auid, 
            appname1, packagename1, appname2, packagename2, appname3, packagename3, 
            appname4, packagename4, appname5, packagename5
        );
    }
}
```

4. 컨트롤러 클래스:

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;
import org.springframework.web.bind.annotation.RestController;
import java.util.List;

@RestController
public class AppUsageController {

    @Autowired
    private AppUsageService appUsageService;

    @GetMapping("/app-usage")
    public List<AppUsageData> getAppUsageDataUsageDaily_00(
        @RequestParam String cust_id, @RequestParam Long grp_id, @RequestParam String app_ver, 
        @RequestParam String startdate, @RequestParam String enddate, @RequestParam Long auid, 
        @RequestParam String appname1, @RequestParam String packagename1, @RequestParam String appname2, 
        @RequestParam String packagename2, @RequestParam String appname3, @RequestParam String packagename3, 
        @RequestParam String appname4, @RequestParam String packagename4, @RequestParam String appname5, 
        @RequestParam String packagename5
    ) {
        return appUsageService.getAppUsageDataUsageDaily_00(
            cust_id, grp_id, app_ver, startdate, enddate, auid, 
            appname1, packagename1, appname2, packagename2, appname3, packagename3, 
            appname4, packagename4, appname5, packagename5
        );
    }
}
```

이제 저장 프로시저의 이름과 매개변수 타입이 일치하도록 수정되었습니다. 또한, 네임스페이스가 포함되어 있는지 확인하고 호출합니다. 이렇게 하면 `fn_appusage_datausage_daily_00` 저장 프로시저가 제대로 호출될 것입니다.

-------------------------------

Spring에서 저장 프로시저(Stored Procedure)를 호출하는 방법에는 여러 가지가 있습니다. `@Procedure` 애노테이션을 사용하거나, `JdbcTemplate` 및 `SimpleJdbcCall` 클래스를 활용할 수 있습니다. 여기서는 두 가지 방법을 설명하겠습니다.

### 방법 1: `@Procedure` 애노테이션 사용

Spring Data JPA에서 `@Procedure` 애노테이션을 사용하여 저장 프로시저를 호출하는 방법입니다.

#### 1.1 저장 프로시저 생성

먼저, 데이터베이스에 저장 프로시저를 생성합니다. 예를 들어, PostgreSQL에서 다음과 같은 프로시저를 생성합니다:

```sql
CREATE OR REPLACE FUNCTION add_numbers(a INTEGER, b INTEGER) RETURNS INTEGER AS $$
BEGIN
    RETURN a + b;
END;
$$ LANGUAGE plpgsql;
```

#### 1.2 JPA 엔티티 리포지토리에서 저장 프로시저 호출

Spring Data JPA 리포지토리 인터페이스에서 `@Procedure` 애노테이션을 사용하여 저장 프로시저를 호출합니다.

```java
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.query.Procedure;
import org.springframework.stereotype.Repository;

@Repository
public interface MyRepository extends JpaRepository<MyEntity, Long> {

    @Procedure("add_numbers")
    Integer addNumbers(Integer a, Integer b);
}
```

#### 1.3 저장 프로시저 호출

서비스 클래스에서 리포지토리를 사용하여 저장 프로시저를 호출합니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

@Service
public class MyService {

    @Autowired
    private MyRepository myRepository;

    @Transactional
    public Integer addNumbers(Integer a, Integer b) {
        return myRepository.addNumbers(a, b);
    }
}
```

### 방법 2: `JdbcTemplate` 및 `SimpleJdbcCall` 사용

`JdbcTemplate`과 `SimpleJdbcCall`을 사용하여 저장 프로시저를 호출하는 방법입니다.

#### 2.1 저장 프로시저 생성

데이터베이스에 동일한 저장 프로시저를 생성합니다.

```sql
CREATE OR REPLACE FUNCTION add_numbers(a INTEGER, b INTEGER) RETURNS INTEGER AS $$
BEGIN
    RETURN a + b;
END;
$$ LANGUAGE plpgsql;
```

#### 2.2 `JdbcTemplate` 및 `SimpleJdbcCall` 설정

`JdbcTemplate` 및 `SimpleJdbcCall`을 사용하여 저장 프로시저를 호출할 수 있도록 설정합니다.

```java
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.simple.SimpleJdbcCall;
import org.springframework.stereotype.Service;

import javax.annotation.PostConstruct;
import java.util.Map;

@Service
public class MyService {

    @Autowired
    private JdbcTemplate jdbcTemplate;
    private SimpleJdbcCall simpleJdbcCall;

    @PostConstruct
    public void init() {
        simpleJdbcCall = new SimpleJdbcCall(jdbcTemplate)
            .withFunctionName("add_numbers");
    }

    public Integer addNumbers(Integer a, Integer b) {
        Map<String, Object> result = simpleJdbcCall.execute(a, b);
        return (Integer) result.get("returnvalue");
    }
}
```

### 정리

1. **@Procedure 애노테이션 사용**: Spring Data JPA 리포지토리에서 `@Procedure` 애노테이션을 사용하여 저장 프로시저를 호출합니다.
2. **JdbcTemplate 및 SimpleJdbcCall 사용**: `JdbcTemplate` 및 `SimpleJdbcCall`을 사용하여 저장 프로시저를 호출합니다.

이 두 가지 방법 모두 저장 프로시저를 호출할 수 있지만, 첫 번째 방법은 Spring Data JPA를 사용하여 더 간결한 코드로 저장 프로시저를 호출할 수 있습니다. 두 번째 방법은 더 많은 유연성을 제공하지만, 코드가 다소 복잡해질 수 있습니다.


----------------------------

Spring JPA에서 실행되는 SQL 쿼리의 내용을 로그로 확인하려면, Hibernate의 로깅 설정을 활성화해야 합니다. Hibernate는 Spring Data JPA의 기본 ORM 툴이기 때문에, Hibernate 로깅을 설정하면 Spring JPA에서 실행되는 쿼리를 텍스트로 볼 수 있습니다.

### 방법 1: `application.properties` 파일을 사용한 설정

Spring Boot 애플리케이션의 `src/main/resources/application.properties` 파일에 다음과 같은 설정을 추가합니다:

```properties
# 기본 SQL 로그 활성화
spring.jpa.show-sql=true

# SQL 쿼리의 형식 지정
spring.jpa.properties.hibernate.format_sql=true

# Hibernate SQL 쿼리 로깅 활성화
logging.level.org.hibernate.SQL=DEBUG

# Hibernate 파라미터 로깅 활성화
logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE
```

### 방법 2: `application.yml` 파일을 사용한 설정

`application.yml` 파일을 사용하는 경우, 다음과 같이 설정할 수 있습니다:

```yaml
spring:
  jpa:
    show-sql: true
    properties:
      hibernate:
        format_sql: true

logging:
  level:
    org.hibernate.SQL: DEBUG
    org.hibernate.type.descriptor.sql.BasicBinder: TRACE
```

### 예제 설명

- `spring.jpa.show-sql=true`: SQL 쿼리를 콘솔에 출력합니다.
- `spring.jpa.properties.hibernate.format_sql=true`: SQL 쿼리를 읽기 쉽게 포맷팅합니다.
- `logging.level.org.hibernate.SQL=DEBUG`: Hibernate가 생성하는 SQL 쿼리를 로깅합니다.
- `logging.level.org.hibernate.type.descriptor.sql.BasicBinder=TRACE`: SQL 쿼리의 파라미터를 로깅합니다.

### 추가 설정

Spring Boot 로그 출력을 조절하려면 `application.properties` 또는 `application.yml` 파일에서 로깅 설정을 추가로 변경할 수 있습니다. 예를 들어, 로그 파일에 쿼리를 출력하려면 다음과 같이 설정할 수 있습니다:

```properties
# 로그 파일 설정
logging.file.name=logs/spring-boot-app.log
logging.file.path=/path/to/logs
```

이 설정을 통해 Spring JPA 애플리케이션에서 실행되는 모든 SQL 쿼리를 텍스트 형식으로 확인할 수 있습니다. 이를 통해 개발 및 디버깅 과정에서 쿼리 내용을 쉽게 추적할 수 있습니다.

------------------------------

최적화된 인덱스를 만드는 것은 데이터베이스의 성능을 크게 향상시킬 수 있습니다. 이를 위해서는 쿼리 패턴을 분석하고, 자주 사용되는 조건을 기반으로 인덱스를 설계해야 합니다. 아래는 빅테이블 A와 빅테이블 B를 조인하는 뷰를 최적화하기 위한 인덱스 설계 방법입니다.

### 1. 쿼리 패턴 분석

우선, 쿼리에서 자주 사용되는 조건과 조인 패턴을 파악해야 합니다. 예를 들어, 아래와 같은 쿼리를 자주 사용한다고 가정합니다:

```sql
SELECT a.customerId, a.dateRange, a.deviceCount, b.appName, b.pkgName
FROM bigtable_a a
JOIN bigtable_b b ON a.groupId = b.groupId
WHERE a.customerId = ? AND a.dateRange BETWEEN ? AND ?;
```

### 2. 조인과 필터 조건에 대한 인덱스 설계

위의 쿼리 패턴을 기반으로, 주로 사용하는 조건과 조인 조건에 맞는 인덱스를 설계합니다.

#### 빅테이블 A에 대한 인덱스

1. `groupId`에 대한 인덱스:
   - 조인 조건에 사용되므로, `groupId`에 인덱스를 생성합니다.
   ```sql
   CREATE INDEX idx_bigtable_a_groupId ON bigtable_a (groupId);
   ```

2. 복합 인덱스:
   - 필터 조건과 조인 조건을 포함하는 복합 인덱스를 생성합니다.
   ```sql
   CREATE INDEX idx_bigtable_a_customerId_dateRange_groupId ON bigtable_a (customerId, dateRange, groupId);
   ```

#### 빅테이블 B에 대한 인덱스

1. `groupId`에 대한 인덱스:
   - 조인 조건에 사용되므로, `groupId`에 인덱스를 생성합니다.
   ```sql
   CREATE INDEX idx_bigtable_b_groupId ON bigtable_b (groupId);
   ```

### 3. 인덱스 설계 예제

#### 빅테이블 A에 대한 인덱스

```sql
-- groupId에 대한 인덱스
CREATE INDEX idx_bigtable_a_groupId ON bigtable_a (groupId);

-- customerId, dateRange, groupId에 대한 복합 인덱스
CREATE INDEX idx_bigtable_a_customerId_dateRange_groupId ON bigtable_a (customerId, dateRange, groupId);
```

#### 빅테이블 B에 대한 인덱스

```sql
-- groupId에 대한 인덱스
CREATE INDEX idx_bigtable_b_groupId ON bigtable_b (groupId);
```

### 4. 인덱스 적용 확인

인덱스가 쿼리에 제대로 적용되는지 확인하려면 `EXPLAIN` 명령을 사용하여 실행 계획을 확인할 수 있습니다.

```sql
EXPLAIN SELECT a.customerId, a.dateRange, a.deviceCount, b.appName, b.pkgName
FROM bigtable_a a
JOIN bigtable_b b ON a.groupId = b.groupId
WHERE a.customerId = 'example_id' AND a.dateRange BETWEEN '2023-01-01' AND '2023-12-31';
```

### 5. 인덱스 유지 관리

인덱스는 데이터베이스의 쓰기 성능에 영향을 미칠 수 있으므로, 정기적으로 인덱스의 효율성을 모니터링하고 필요에 따라 재구성(`REINDEX`)하거나 삭제(`DROP INDEX`)해야 합니다.

### 결론

최적화된 인덱스를 설계하려면 쿼리 패턴을 분석하고, 자주 사용되는 조건과 조인 조건을 기반으로 인덱스를 생성해야 합니다. 빅테이블 A와 빅테이블 B를 조인하는 뷰의 성능을 최적화하기 위해, `groupId`를 기준으로 각각의 테이블에 인덱스를 생성하고, 필터 조건과 조인 조건을 포함하는 복합 인덱스를 추가로 생성하여 쿼리 성능을 향상시킬 수 있습니다.

------------------------

PostgreSQL에서 인덱스를 생성하는 방법은 다양한 옵션과 방법이 있으며, 이로 인해 쿼리 성능을 크게 향상시킬 수 있습니다. 아래는 PostgreSQL에서 인덱스를 생성하고 사용하는 방법에 대한 상세한 설명입니다.

### 기본 인덱스 생성

기본 인덱스는 `CREATE INDEX` 명령을 사용하여 생성할 수 있습니다. 예를 들어, `users` 테이블의 `username` 열에 인덱스를 생성하려면 다음과 같이 합니다:

```sql
CREATE INDEX idx_username ON users (username);
```

이 인덱스는 `users` 테이블의 `username` 열에 대해 기본 B-tree 인덱스를 생성합니다.

### 고유 인덱스 (Unique Index)

고유 인덱스는 특정 열에 중복된 값이 들어갈 수 없도록 합니다. `UNIQUE` 키워드를 사용하여 생성할 수 있습니다:

```sql
CREATE UNIQUE INDEX idx_username_unique ON users (username);
```

### 복합 인덱스 (Composite Index)

복합 인덱스는 여러 열을 결합하여 인덱스를 생성합니다. 예를 들어, `users` 테이블의 `lastname`과 `firstname` 열에 대한 복합 인덱스를 생성하려면 다음과 같이 합니다:

```sql
CREATE INDEX idx_lastname_firstname ON users (lastname, firstname);
```

복합 인덱스는 인덱스의 열 순서에 따라 다르게 작동하므로, 쿼리에서 사용되는 열의 순서와 일치시키는 것이 중요합니다.

### 부분 인덱스 (Partial Index)

부분 인덱스는 특정 조건을 만족하는 행에 대해서만 인덱스를 생성합니다. 예를 들어, `active` 상태가 `true`인 `users`만 인덱스하려면 다음과 같이 합니다:

```sql
CREATE INDEX idx_active_users ON users (username) WHERE active = true;
```

### 표현식 인덱스 (Expression Index)

표현식 인덱스는 열의 값이 아닌 계산된 표현식을 기반으로 인덱스를 생성합니다. 예를 들어, `lower(username)`을 인덱스하려면 다음과 같이 합니다:

```sql
CREATE INDEX idx_lower_username ON users (lower(username));
```

### GiST, GIN, BRIN 인덱스

PostgreSQL은 다양한 인덱스 타입을 지원합니다. 특정 데이터 타입이나 쿼리 패턴에 대해 더 나은 성능을 제공하는 인덱스 타입이 있습니다.

- **GiST 인덱스**: 범위 쿼리나 근접 검색에 적합합니다.
- **GIN 인덱스**: 배열이나 JSONB 데이터 타입의 요소를 검색할 때 유용합니다.
- **BRIN 인덱스**: 매우 큰 테이블에서 특정 범위의 데이터를 검색할 때 효율적입니다.

예를 들어, JSONB 열에 GIN 인덱스를 생성하려면:

```sql
CREATE INDEX idx_jsonb_data ON documents USING gin (data);
```

### 인덱스 관리

#### 인덱스 삭제

더 이상 필요하지 않은 인덱스는 `DROP INDEX` 명령을 사용하여 삭제할 수 있습니다:

```sql
DROP INDEX idx_username;
```

#### 인덱스 리빌드

인덱스를 재구성하여 성능을 최적화할 수 있습니다:

```sql
REINDEX INDEX idx_username;
```

### 인덱스 사용 모니터링

`EXPLAIN` 명령을 사용하여 쿼리가 인덱스를 사용하는지 확인할 수 있습니다:

```sql
EXPLAIN SELECT * FROM users WHERE username = 'john_doe';
```

이 명령은 쿼리 실행 계획을 보여주며, 인덱스를 사용하는지 여부를 알 수 있습니다.

### 예제: 다양한 인덱스 생성

아래는 PostgreSQL에서 다양한 인덱스를 생성하는 예제입니다.

```sql
-- 기본 인덱스
CREATE INDEX idx_username ON users (username);

-- 고유 인덱스
CREATE UNIQUE INDEX idx_username_unique ON users (username);

-- 복합 인덱스
CREATE INDEX idx_lastname_firstname ON users (lastname, firstname);

-- 부분 인덱스
CREATE INDEX idx_active_users ON users (username) WHERE active = true;

-- 표현식 인덱스
CREATE INDEX idx_lower_username ON users (lower(username));

-- GiST 인덱스 (예: 범위 데이터)
CREATE INDEX idx_range_data ON data_table USING gist (range_column);

-- GIN 인덱스 (예: JSONB 데이터)
CREATE INDEX idx_jsonb_data ON documents USING gin (data);

-- BRIN 인덱스 (예: 매우 큰 테이블)
CREATE INDEX idx_brin_data ON large_table USING brin (date_column);
```

이와 같은 다양한 인덱스 옵션을 통해 쿼리 성능을 최적화하고 데이터베이스의 응답 속도를 향상시킬 수 있습니다.

---------------------------
PostgreSQL에서 뷰(View)는 기본적으로 실시간 데이터를 조회하는 가상 테이블입니다. 그러나 미리 계산된 결과를 저장하고 이를 효율적으로 조회하기 위해서는 "물리적인" 테이블에 데이터를 저장하는 방법이 필요합니다. 이를 위해 사용할 수 있는 방법 중 하나는 **물리적 테이블**을 만들어 주기적으로 데이터를 갱신하는 것입니다. 이를 수행하는 데 트리거와 정기적인 배치 작업을 활용할 수 있습니다.

또한 PostgreSQL 9.3 이상에서는 "마테리얼라이즈드 뷰(Materialized View)"라는 기능을 사용할 수 있습니다. 마테리얼라이즈드 뷰는 뷰의 결과를 실제로 저장하고, 필요할 때마다 이 저장된 결과를 참조합니다.

### 마테리얼라이즈드 뷰

마테리얼라이즈드 뷰는 뷰의 결과를 디스크에 저장하는 기능입니다. 저장된 데이터를 쿼리할 때는 물리적으로 저장된 데이터를 참조하므로 성능이 향상될 수 있습니다. 단, 기본 테이블의 데이터가 변경되더라도 마테리얼라이즈드 뷰는 자동으로 갱신되지 않으며, 수동으로 갱신해야 합니다.

#### 마테리얼라이즈드 뷰 생성

다음은 마테리얼라이즈드 뷰를 생성하는 예제입니다:

```sql
CREATE MATERIALIZED VIEW active_employees AS
SELECT employee_id, first_name, last_name, email
FROM employees
WHERE active = true;
```

#### 마테리얼라이즈드 뷰 갱신

기본 테이블의 데이터가 변경된 후에는 마테리얼라이즈드 뷰를 갱신해야 최신 데이터를 반영할 수 있습니다:

```sql
REFRESH MATERIALIZED VIEW active_employees;
```

마테리얼라이즈드 뷰는 기본 테이블의 데이터를 캐싱하여 성능을 향상시키지만, 데이터가 최신 상태인지 확인하기 위해 주기적으로 갱신해야 합니다.

### 정기적인 배치 작업을 통한 물리적 테이블 갱신

마테리얼라이즈드 뷰 외에도 정기적인 배치 작업을 통해 물리적 테이블을 갱신할 수 있습니다. 예를 들어, `cron` 작업 또는 PostgreSQL의 `pg_cron` 확장을 사용하여 주기적으로 데이터를 갱신할 수 있습니다.

#### 물리적 테이블 생성 및 갱신 예제

1. **물리적 테이블 생성**

   ```sql
   CREATE TABLE active_employees (
       employee_id SERIAL PRIMARY KEY,
       first_name VARCHAR(50),
       last_name VARCHAR(50),
       email VARCHAR(100)
   );
   ```

2. **데이터 삽입 또는 갱신을 위한 스크립트**

   ```sql
   INSERT INTO active_employees (employee_id, first_name, last_name, email)
   SELECT employee_id, first_name, last_name, email
   FROM employees
   WHERE active = true
   ON CONFLICT (employee_id) DO UPDATE
   SET first_name = EXCLUDED.first_name,
       last_name = EXCLUDED.last_name,
       email = EXCLUDED.email;
   ```

3. **정기적인 갱신을 위한 `pg_cron` 설정**

   `pg_cron` 확장을 사용하면 PostgreSQL 내에서 정기적으로 작업을 실행할 수 있습니다.

   ```sql
   -- pg_cron 확장 설치
   CREATE EXTENSION pg_cron;

   -- 매일 자정에 active_employees 테이블 갱신
   SELECT cron.schedule('0 0 * * *', $$INSERT INTO active_employees (employee_id, first_name, last_name, email)
   SELECT employee_id, first_name, last_name, email
   FROM employees
   WHERE active = true
   ON CONFLICT (employee_id) DO UPDATE
   SET first_name = EXCLUDED.first_name,
       last_name = EXCLUDED.last_name,
       email = EXCLUDED.email;$$);
   ```

### 요약

- **마테리얼라이즈드 뷰**: 저장된 데이터로 성능 향상을 얻을 수 있지만, 데이터 갱신은 수동으로 이루어집니다.
- **물리적 테이블과 배치 작업**: 데이터를 주기적으로 갱신하여 저장된 데이터를 최신 상태로 유지할 수 있습니다.

이 방법들을 통해 PostgreSQL에서 미리 계산된 데이터를 저장하고, 조회 성능을 최적화할 수 있습니다.


## PostgreSQL View 및 Materialized View 비교

**View**는 기존 테이블의 데이터를 기반으로 가상 테이블을 만드는 기능입니다. View는 실제 테이블처럼 보이지만 실제 데이터는 저장하지 않습니다. 쿼리가 View를 참조하면 PostgreSQL은 View 정의에 지정된 쿼리를 실행하고 결과를 반환합니다.

**Materialized View**는 View와 유사하지만 실제 데이터를 저장합니다. Materialized View는 처음 만들 때 쿼리가 실행되어 데이터가 채워지고, 이후에는 정기적으로 새로 고쳐집니다. Materialized View를 사용하면 복잡한 쿼리의 성능을 향상시킬 수 있습니다.

**View와 Materialized View의 주요 차이점은 다음과 같습니다.**

| 기능 | View | Materialized View |
|---|---|---|
| 데이터 저장 | 저장하지 않음 | 저장 |
| 쿼리 실행 | 쿼리 실행 시마다 실행 | 정기적으로 새로 고침 |
| 성능 | 일반적으로 Materialized View가 더 빠름 | 쿼리 유형에 따라 다름 |
| 유지 관리 | 유지 관리가 간편 | 데이터 변경에 따라 새로 고쳐야 함 |
| 용도 | 데이터 요약, 간단한 쿼리 | 복잡한 쿼리, 성능 향상 |

**어떤 것을 사용해야 할까요?**

* **데이터 요약, 간단한 쿼리**에 사용하려면 **View**를 사용하는 것이 좋습니다. View는 유지 관리가 간편하고 실제 테이블에 영향을 미치지 않습니다.
* **복잡한 쿼리, 성능 향상**에 사용하려면 **Materialized View**를 사용하는 것이 좋습니다. Materialized View는 View보다 빠르지만 데이터 변경에 따라 새로 고쳐야 합니다.

**Materialized View 사용 시 고려 사항:**

* Materialized View는 정기적으로 새로 고쳐야 합니다. 새로 고침 빈도는 데이터 변경 빈도와 성능 요구 사항에 따라 결정해야 합니다.
* Materialized View는 실제 테이블 공간을 차지합니다. Materialized View를 만들기 전에 충분한 디스크 공간이 있는지 확인하십시오.
* Materialized View는 데이터 변경에 따라 자동으로 업데이트되지 않습니다. 데이터 변경 후에는 수동으로 새로 고쳐야 합니다.

**추가 정보:**

* PostgreSQL View 문서: [유효하지 않은 URL 삭제됨]
* PostgreSQL Materialized View 문서: [https://postgresql.kr/docs/13/sql-creatematerializedview.html](https://postgresql.kr/docs/13/sql-creatematerializedview.html)
* PostgreSQL View vs Materialized View: [https://www.postgresqltutorial.com/postgresql-views/postgresql-materialized-views/](https://www.postgresqltutorial.com/postgresql-views/postgresql-materialized-views/)

어떤 상황에 어떤 것을 사용해야 할지 잘 모르겠다면 항상 성능 전문가와 상담하는 것이 좋습니다.


--------------------------

PostgreSQL에서 뷰(View)는 미리 데이터를 저장해두는 것이 아니라, 쿼리가 실행될 때마다 정의된 SELECT 문을 실행하여 결과를 생성하는 일종의 가상 테이블입니다. 따라서 뷰는 실시간으로 데이터를 조회할 때 사용되며, 뷰에 대한 쿼리가 실행될 때마다 최신 데이터를 조회할 수 있습니다.

### 뷰의 특징

- **가상 테이블**: 뷰는 물리적으로 데이터를 저장하지 않고, SELECT 문을 저장하고 있는 논리적 테이블입니다.
- **실시간 데이터**: 뷰에 대한 쿼리가 실행될 때마다 원본 테이블의 데이터를 조회하여 최신 결과를 반환합니다.
- **데이터 보안 및 단순화**: 뷰를 사용하면 복잡한 쿼리를 단순화하고, 특정 데이터에 대한 접근을 제어할 수 있습니다.

### 뷰 생성

뷰를 생성하는 기본 구문은 다음과 같습니다:

```sql
CREATE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

### 예제

예를 들어, `employees` 테이블에서 활성 상태의 직원들만 조회하는 뷰를 생성할 수 있습니다:

```sql
CREATE VIEW active_employees AS
SELECT employee_id, first_name, last_name, email
FROM employees
WHERE active = true;
```

이제 `active_employees` 뷰를 조회하면, 항상 `employees` 테이블에서 `active` 열이 `true`인 최신 데이터를 조회할 수 있습니다:

```sql
SELECT * FROM active_employees;
```

### 뷰의 사용

뷰는 다음과 같은 방식으로 사용됩니다:

- **데이터 조회**: 뷰는 일반 테이블과 동일하게 조회할 수 있습니다.
- **데이터 변경**: 특정 조건을 만족하는 경우, 뷰를 통해 데이터 변경(INSERT, UPDATE, DELETE)이 가능합니다. 하지만 대부분의 경우 뷰는 읽기 전용으로 사용됩니다.

### 예제: 읽기 전용 뷰

읽기 전용 뷰는 데이터 변경을 허용하지 않습니다. 예를 들어, 복잡한 조인을 포함하는 뷰는 읽기 전용으로 사용됩니다:

```sql
CREATE VIEW department_summary AS
SELECT d.department_id, d.department_name, COUNT(e.employee_id) AS num_employees
FROM departments d
LEFT JOIN employees e ON d.department_id = e.department_id
GROUP BY d.department_id, d.department_name;
```

이 뷰를 통해 각 부서의 직원 수를 조회할 수 있습니다:

```sql
SELECT * FROM department_summary;
```

### 뷰 업데이트

뷰를 업데이트하는 구문은 다음과 같습니다:

```sql
CREATE OR REPLACE VIEW view_name AS
SELECT column1, column2, ...
FROM table_name
WHERE condition;
```

기존의 `active_employees` 뷰를 업데이트하려면:

```sql
CREATE OR REPLACE VIEW active_employees AS
SELECT employee_id, first_name, last_name, email, department_id
FROM employees
WHERE active = true;
```

### 요약

뷰는 미리 데이터를 저장하지 않고, 쿼리가 실행될 때마다 최신 데이터를 조회하는 가상 테이블입니다. 이를 통해 복잡한 쿼리를 단순화하고, 데이터 접근을 제어할 수 있습니다. 뷰는 항상 최신 데이터를 반환하기 때문에 실시간 데이터 조회가 필요한 경우에 유용합니다.


PostgreSQL에서 **읽기 성능**이 가장 좋은 인덱스 유형은 **쿼리 유형과 데이터 특성**에 따라 다릅니다. 

**일반적인 권장 사항은 다음과 같습니다.**

* **대부분의 경우 B-tree 인덱스가 가장 좋은 선택입니다.** B-tree 인덱스는 범위 검색, 정확히 일치하는 검색 및 부분 문자열 검색에 효율적입니다.
* **해시 인덱스는 정확히 일치하는 검색에 매우 빠르지만 범위 검색에는 적합하지 않습니다.** 또한 해시 인덱스는 데이터가 많이 변경되는 경우 유지 관리하기 어려울 수 있습니다.
* **GiST 인덱스는 공간 데이터 검색에 적합합니다.** 예를 들어, 지도 응용 프로그램에서 사용자의 위치와 가까운 레스토랑을 찾는 경우 GiST 인덱스를 사용할 수 있습니다.
* **SP-GiST 인덱스는 복잡한 공간 데이터 검색에 적합합니다.** 예를 들어, 도로 네트워크에서 두 지점 사이의 최단 경로를 찾는 경우 SP-GiST 인덱스를 사용할 수 있습니다.
* **GIN 인덱스는 텍스트 검색, 패턴 검색 및 부분 일치 검색에 적합합니다.**
* **BRIN 인덱스는 데이터가 많은 테이블에서 특정 범위의 값을 빠르게 검색하는 데 적합합니다.**

**어떤 인덱스 유형을 사용할지 결정하는 데 도움이 되는 몇 가지 추가 요소는 다음과 같습니다.**

* **테이블의 데이터 양:** 테이블의 데이터가 많을수록 인덱스 선택이 중요해집니다.
* **쿼리 실행 빈도:** 특정 쿼리가 자주 실행되는 경우 해당 쿼리를 빠르게 처리하는 데 도움이 되는 인덱스를 만들어야 합니다.
* **데이터 변경 빈도:** 데이터가 자주 변경되는 경우 유지 관리하기 쉬운 인덱스 유형을 선택해야 합니다.

**인덱스 성능을 최적화하려면 다음 방법을 사용할 수도 있습니다.**

* **복합 인덱스 사용:** 여러 칼럼에 인덱스를 만들 수 있습니다. 이는 쿼리가 여러 칼럼을 사용하는 경우 유용할 수 있습니다.
* **부분 인덱스 사용:** 테이블의 모든 행이 아닌 테이블의 특정 행만 포함하는 인덱스를 만들 수 있습니다.
* **인덱스 제거:** 인덱스가 쿼리 성능을 향상시키지 못하는 경우 제거할 수 있습니다.

**인덱스 유형 선택 및 성능 최적화에 대한 자세한 내용은 다음 리소스를 참조하십시오.**

* PostgreSQL 인덱스 문서: [https://postgresql.kr/docs/13/sql-createindex.html](https://postgresql.kr/docs/13/sql-createindex.html)
* PostgreSQL 인덱스 사용법: [https://techblog.cjenm.com/postgresql-index-%EC%82%AC%EC%9A%A9%EB%B2%95-5d02995aa34e](https://techblog.cjenm.com/postgresql-index-%EC%82%AC%EC%9A%A9%EB%B2%95-5d02995aa34e)
* PostgreSQL에서 인덱스 성능 조정: [https://www.postgresql.org/docs/](https://www.postgresql.org/docs/)


어떤 유형의 인덱스가 특정 상황에 가장 적합한지 잘 모르겠다면 항상 성능 전문가와 상담하는 것이 좋습니다.

--------------------------

## PostgreSQL에서 인덱스 만드는 방법

PostgreSQL에서 인덱스를 만들려면 `CREATE INDEX` 명령을 사용합니다. 기본 구문은 다음과 같습니다.

```sql
CREATE INDEX index_name
ON table_name (column1, column2, ...);
```

여기서:

* `index_name`은 인덱스의 이름입니다.
* `table_name`은 인덱스를 만들 테이블의 이름입니다.
* `column1`, `column2`는 인덱스가 만들어질 칼럼입니다. 

**예제:**

고객 테이블에 `customer_id`와 `name` 칼럼에 인덱스를 만들려면 다음 명령을 사용합니다.

```sql
CREATE INDEX customer_id_name_idx
ON customers (customer_id, name);
```

**인덱스 유형 지정:**

PostgreSQL은 B-tree, hash, GiST, SP-GiST, GIN, BRIN 등 다양한 유형의 인덱스를 지원합니다. 특정 인덱스 유형을 사용하려면 `USING` 절을 사용할 수 있습니다.

```sql
CREATE INDEX index_name
ON table_name (column1, column2, ...)
USING index_type;
```

예를 들어, `customer_id` 칼럼에 해시 인덱스를 만들려면 다음 명령을 사용합니다.

```sql
CREATE INDEX customer_id_idx
ON customers (customer_id)
USING HASH;
```

**부분 인덱스 만들기:**

전체 테이블 데이터가 아닌 테이블의 특정 부분에만 인덱스를 만들 수 있습니다. WHERE 절을 사용하여 부분 인덱스를 만들 수 있습니다.

```sql
CREATE INDEX index_name
ON table_name (column1, column2, ...)
WHERE condition;
```

예를 들어, `orders` 테이블에서 `order_status`가 'shipped'인 주문에만 인덱스를 만들려면 다음 명령을 사용합니다.

```sql
CREATE INDEX shipped_orders_idx
ON orders (order_id)
WHERE order_status = 'shipped';
```

**인덱스 삭제:**

인덱스는 `DROP INDEX` 명령을 사용하여 삭제할 수 있습니다.

```sql
DROP INDEX index_name;
```

예를 들어, `customer_id_name_idx` 인덱스를 삭제하려면 다음 명령을 사용합니다.

```sql
DROP INDEX customer_id_name_idx;
```

**주의 사항:**

* 인덱스는 항상 성능 향상을 보장하지는 않습니다. 실제로 잘못 사용하면 성능 저하를 초래할 수도 있습니다.
* 인덱스를 만들기 전에 쿼리 실행 계획을 확인하여 해당 쿼리에 인덱스가 도움이 되는지 확인하는 것이 좋습니다.
* 많은 칼럼에 인덱스를 만들면 오히려 성능 저하를 초래할 수 있으므로 주의해야 합니다.

**추가 정보:**

* PostgreSQL 인덱스 문서: [https://postgresql.kr/docs/13/sql-createindex.html](https://postgresql.kr/docs/13/sql-createindex.html)
* PostgreSQL 인덱스 사용법: [https://techblog.cjenm.com/postgresql-index-%EC%82%AC%EC%9A%A9%EB%B2%95-5d02995aa34e](https://techblog.cjenm.com/postgresql-index-%EC%82%AC%EC%9A%A9%EB%B2%95-5d02995aa34e)
* PostgreSQL에서 인덱스 성능 조정: [유효하지 않은 URL 삭제됨]

PostgreSQL에서 **읽기 성능**이 가장 좋은 인덱스 유형은 **쿼리 유형과 데이터 특성**에 따라 다릅니다. 

**일반적인 권장 사항은 다음과 같습니다.**

* **대부분의 경우 B-tree 인덱스가 가장 좋은 선택입니다.** B-tree 인덱스는 범위 검색, 정확히 일치하는 검색 및 부분 문자열 검색에 효율적입니다.
* **해시 인덱스는 정확히 일치하는 검색에 매우 빠르지만 범위 검색에는 적합하지 않습니다.** 또한 해시 인덱스는 데이터가 많이 변경되는 경우 유지 관리하기 어려울 수 있습니다.
* **GiST 인덱스는 공간 데이터 검색에 적합합니다.** 예를 들어, 지도 응용 프로그램에서 사용자의 위치와 가까운 레스토랑을 찾는 경우 GiST 인덱스를 사용할 수 있습니다.
* **SP-GiST 인덱스는 복잡한 공간 데이터 검색에 적합합니다.** 예를 들어, 도로 네트워크에서 두 지점 사이의 최단 경로를 찾는 경우 SP-GiST 인덱스를 사용할 수 있습니다.
* **GIN 인덱스는 텍스트 검색, 패턴 검색 및 부분 일치 검색에 적합합니다.**
* **BRIN 인덱스는 데이터가 많은 테이블에서 특정 범위의 값을 빠르게 검색하는 데 적합합니다.**

**어떤 인덱스 유형을 사용할지 결정하는 데 도움이 되는 몇 가지 추가 요소는 다음과 같습니다.**

* **테이블의 데이터 양:** 테이블의 데이터가 많을수록 인덱스 선택이 중요해집니다.
* **쿼리 실행 빈도:** 특정 쿼리가 자주 실행되는 경우 해당 쿼리를 빠르게 처리하는 데 도움이 되는 인덱스를 만들어야 합니다.
* **데이터 변경 빈도:** 데이터가 자주 변경되는 경우 유지 관리하기 쉬운 인덱스 유형을 선택해야 합니다.

**인덱스 성능을 최적화하려면 다음 방법을 사용할 수도 있습니다.**

* **복합 인덱스 사용:** 여러 칼럼에 인덱스를 만들 수 있습니다. 이는 쿼리가 여러 칼럼을 사용하는 경우 유용할 수 있습니다.
* **부분 인덱스 사용:** 테이블의 모든 행이 아닌 테이블의 특정 행만 포함하는 인덱스를 만들 수 있습니다.
* **인덱스 제거:** 인덱스가 쿼리 성능을 향상시키지 못하는 경우 제거할 수 있습니다.

**인덱스 유형 선택 및 성능 최적화에 대한 자세한 내용은 다음 리소스를 참조하십시오.**

* PostgreSQL 인덱스 문서: [https://postgresql.kr/docs/13/sql-createindex.html](https://postgresql.kr/docs/13/sql-createindex.html)
* PostgreSQL 인덱스 사용법: [https://techblog.cjenm.com/postgresql-index-%EC%82%AC%EC%9A%A9%EB%B2%95-5d02995aa34e](https://techblog.cjenm.com/postgresql-index-%EC%82%AC%EC%9A%A9%EB%B2%95-5d02995aa34e)
* PostgreSQL에서 인덱스 성능 조정: [https://www.postgresql.org/docs/](https://www.postgresql.org/docs/)


어떤 유형의 인덱스가 특정 상황에 가장 적합한지 잘 모르겠다면 항상 성능 전문가와 상담하는 것이 좋습니다.
-------------------------
PostgreSQL에서 인덱스를 생성하는 방법 및 예제를 아래에 설명하겠습니다.

### 인덱스를 만드는 이유
인덱스는 테이블의 검색 성능을 향상시키기 위해 사용됩니다. 인덱스를 사용하면 특정 열에 대한 검색, 정렬 및 조인을 더 빠르게 수행할 수 있습니다. 하지만 인덱스를 만들면 데이터 삽입, 업데이트, 삭제 시 추가적인 오버헤드가 발생할 수 있으므로, 인덱스의 사용은 신중해야 합니다.

### 인덱스 생성 방법
PostgreSQL에서 인덱스를 생성하려면 `CREATE INDEX` 명령을 사용합니다.

#### 기본 구문
```sql
CREATE INDEX index_name
ON table_name (column_name);
```

### 예제

1. **단일 열 인덱스 생성**
   - 테이블 `employees`에 `last_name` 열에 대한 인덱스를 생성합니다.
   ```sql
   CREATE INDEX idx_employees_last_name
   ON employees (last_name);
   ```

2. **다중 열 인덱스 생성 (복합 인덱스)**
   - 테이블 `employees`에 `last_name`과 `first_name` 열에 대한 복합 인덱스를 생성합니다.
   ```sql
   CREATE INDEX idx_employees_last_first_name
   ON employees (last_name, first_name);
   ```

3. **고유 인덱스 생성**
   - 고유 인덱스는 인덱스가 생성된 열의 값이 중복되지 않도록 합니다.
   ```sql
   CREATE UNIQUE INDEX idx_employees_employee_id
   ON employees (employee_id);
   ```

4. **함수 기반 인덱스 생성**
   - 함수 기반 인덱스는 특정 함수나 표현식을 기준으로 인덱스를 생성합니다. 예를 들어, `LOWER` 함수를 사용하여 `email` 열에 대해 대소문자를 구분하지 않는 인덱스를 생성합니다.
   ```sql
   CREATE INDEX idx_employees_lower_email
   ON employees (LOWER(email));
   ```

5. **조건부 인덱스 생성 (파셜 인덱스)**
   - 조건부 인덱스는 특정 조건을 만족하는 행에 대해서만 인덱스를 생성합니다. 예를 들어, `active` 열이 `true`인 행에 대해서만 인덱스를 생성합니다.
   ```sql
   CREATE INDEX idx_employees_active
   ON employees (last_name)
   WHERE active = true;
   ```

### 인덱스 삭제
인덱스를 더 이상 사용하지 않거나 성능에 부정적인 영향을 미친다면 삭제할 수 있습니다.
```sql
DROP INDEX index_name;
```
예를 들어, `idx_employees_last_name` 인덱스를 삭제하려면:
```sql
DROP INDEX idx_employees_last_name;
```

### 참고 사항
- 인덱스는 검색 성능을 향상시키지만, 데이터 삽입, 업데이트 및 삭제 성능에는 영향을 미칠 수 있습니다.
- 필요한 열에 대해서만 인덱스를 생성하여 성능을 최적화하는 것이 중요합니다.
- 복합 인덱스는 자주 함께 사용되는 열에 대해 유용합니다.
- 인덱스는 디스크 공간을 추가로 사용합니다.

PostgreSQL에서 인덱스를 생성하는 방법과 예제를 통해 검색 성능을 최적화할 수 있습니다. 사용 사례에 맞는 적절한 인덱스를 설계하는 것이 중요합니다.

-----------------------------
Root 인터페이스를 사용하여 JPA 쿼리를 작성할 때, 런타임에 어떤 DB 테이블을 참조하는지에 대한 정보는 Root에 대한 메타데이터에서 얻을 수 있습니다. JPA 구현 (예: Hibernate)은 엔터티 클래스에 대한 메타데이터를 사용하여 실제 DB 테이블과 매핑된 정보를 저장하고 있습니다.

Root 인터페이스를 생성하고 엔터티 클래스를 지정한 후에, Root 객체에서 `getModel()` 메서드를 호출하여 해당 엔터티 클래스의 메타모델 정보를 얻을 수 있습니다. 메타모델은 엔터티 클래스의 필드와 테이블 매핑 정보를 포함하고 있습니다.

예를 들어, 다음은 Root를 사용하여 엔터티 클래스의 테이블 이름을 얻는 방법입니다:

```java
Root<EntityClass> root = criteriaQuery.from(EntityClass.class);
String tableName = root.getModel().getName();
```

위 코드에서 "EntityClass"는 엔터티 클래스의 이름을 나타냅니다. `root.getModel().getName()`을 호출하면 해당 엔터티 클래스가 매핑된 실제 DB 테이블의 이름을 얻을 수 있습니다.

이렇게 얻은 테이블 이름은 JPA 구현에 의해 자동으로 생성된 것이며, 실제 실행 환경 (예: 데이터베이스 서버)에 따라 다를 수 있습니다. 개발 중인 애플리케이션에서 사용 중인 데이터베이스에 대한 테이블 이름을 얻고자 한다면 해당 코드를 런타임 환경에서 실행시켜서 값을 확인해보세요.


-----------------------------


JPA `TypedQuery`에서 결과를 `distinct`로 받기 위해서는 쿼리에서 `DISTINCT` 키워드를 사용하면 됩니다. 다음은 `TypedQuery`에서 결과를 `distinct`로 받는 예제입니다.

```java
import javax.persistence.EntityManager;
import javax.persistence.TypedQuery;
import java.util.List;

public class YourJpaClass {

    private final EntityManager entityManager;

    // Constructor

    public List<EntityClass> findDistinctResults() {
        String jpql = "SELECT DISTINCT e FROM EntityClass e";
        TypedQuery<EntityClass> query = entityManager.createQuery(jpql, EntityClass.class);

        return query.getResultList();
    }
}
```

위의 코드에서 `EntityClass`는 실제 엔터티 클래스의 이름으로 대체되어야 합니다. 이 쿼리는 `DISTINCT` 키워드를 사용하여 중복을 제거하고 모든 엔터티를 검색합니다.

이를 호출하는 예제 코드는 다음과 같을 것입니다:

```java
List<EntityClass> distinctResults = yourJpaClass.findDistinctResults();

// distinctResults를 사용하여 결과를 처리하거나 출력
```

중요한 점은 `DISTINCT`를 사용할 때는 결과가 모든 필드에 대해 동등한지 확인해야 합니다. 만약 엔터티의 특정 필드만 고려해야 한다면 `SELECT DISTINCT e.someField FROM EntityClass e`와 같이 사용할 수 있습니다.


JPA Criteria API를 사용하여 `TypedQuery`에서 결과를 `distinct`로 받으려면, `CriteriaQuery`의 `distinct` 메서드를 사용할 수 있습니다. 다음은 해당 방법을 사용한 예제입니다.

```java
import javax.persistence.EntityManager;
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Root;
import java.util.List;

public class YourJpaClass {

    private final EntityManager entityManager;

    // Constructor

    public List<EntityClass> findDistinctResults() {
        CriteriaBuilder criteriaBuilder = entityManager.getCriteriaBuilder();
        CriteriaQuery<EntityClass> criteriaQuery = criteriaBuilder.createQuery(EntityClass.class);
        Root<EntityClass> root = criteriaQuery.from(EntityClass.class);

        // distinct 메서드를 사용하여 중복을 제거
        criteriaQuery.distinct(true);

        criteriaQuery.select(root); // 모든 엔터티를 선택하거나, 필요에 따라 다양한 조건을 추가

        return entityManager.createQuery(criteriaQuery).getResultList();
    }
}
```

위의 코드에서 `EntityClass`는 실제 엔터티 클래스의 이름으로 대체되어야 합니다. `criteriaQuery.distinct(true)`를 사용하여 중복을 제거하고, 나머지 쿼리 작성은 필요에 따라 조절할 수 있습니다.

이를 호출하는 예제 코드는 다음과 같을 것입니다:

```java
List<EntityClass> distinctResults = yourJpaClass.findDistinctResults();

// distinctResults를 사용하여 결과를 처리하거나 출력
```

이 방법은 JPQL을 사용하지 않고도 Criteria API를 활용하여 `TypedQuery`에서 `distinct`한 결과를 얻을 수 있습니다.


`CriteriaQuery`를 사용하여 Criteria API로 JPQL의 `DISTINCT`와 유사한 동작을 수행하는 방법은, 보통 결과를 가져온 후에 자바 코드 상에서 중복을 제거하는 방식을 활용하는 것입니다. 이를 위해 `Set`을 활용할 수 있습니다.

다음은 이러한 방식으로 중복을 제거하는 예제 코드입니다:

```java
import javax.persistence.EntityManager;
import javax.persistence.criteria.CriteriaBuilder;
import javax.persistence.criteria.CriteriaQuery;
import javax.persistence.criteria.Root;
import java.util.HashSet;
import java.util.List;
import java.util.Set;

public class YourJpaClass {

    private final EntityManager entityManager;

    // Constructor

    public List<EntityClass> findDistinctResults() {
        CriteriaBuilder criteriaBuilder = entityManager.getCriteriaBuilder();
        CriteriaQuery<EntityClass> criteriaQuery = criteriaBuilder.createQuery(EntityClass.class);
        Root<EntityClass> root = criteriaQuery.from(EntityClass.class);

        criteriaQuery.select(root);

        List<EntityClass> resultList = entityManager.createQuery(criteriaQuery).getResultList();

        // 중복 제거를 위해 Set을 사용
        Set<EntityClass> distinctResults = new HashSet<>(resultList);

        return new ArrayList<>(distinctResults);
    }
}
```

위의 코드에서는 `Set`을 사용하여 중복을 제거합니다. 결과를 `Set`에 넣어 중복을 제거한 후, 다시 `ArrayList`로 변환하여 반환합니다.

이러한 방식은 Criteria API 자체에서는 DISTINCT 키워드와 같은 직접적인 기능을 제공하지 않을 때 사용할 수 있는 방법 중 하나입니다.


JPA Criteria API에서 `criteriaQuery.distinct(true)`를 사용하여 `DISTINCT`를 지정하면, 실제 데이터베이스에서의 실행 결과가 예상과 다를 수 있습니다. 이는 JPA Criteria API의 특성과 데이터베이스 드라이버의 동작에 따라 다르게 나타날 수 있습니다.

일반적으로 `criteriaQuery.distinct(true)`는 JPA 레벨에서 결과를 중복 없이 가져올 것으로 예상되지만, 이것이 데이터베이스 엔진까지 동일하게 영향을 미치지 않을 수 있습니다. 몇 가지 이유로 인해 데이터베이스에서 결과를 가져오는 시점에서 중복이 발생할 수 있습니다.

이런 경우에는 데이터베이스 벤더나 드라이버의 특성에 따라 다를 수 있습니다. 몇 가지 대안적인 방법을 고려해 볼 수 있습니다:

1. **GROUP BY 사용**: 대부분의 경우 `GROUP BY` 절을 사용하여 중복된 결과를 방지할 수 있습니다. 다음은 `GROUP BY`를 사용하는 예제입니다.

    ```java
    criteriaQuery.groupBy(root.get("yourField"));
    ```

    `yourField`는 중복을 제거하고자 하는 필드명으로 대체되어야 합니다.

2. **Subquery 사용**: 서브쿼리를 사용하여 중복을 방지하는 방법도 있습니다. 다음은 `Subquery`를 사용하는 예제입니다.

    ```java
    Subquery<EntityClass> subquery = criteriaQuery.subquery(EntityClass.class);
    Root<EntityClass> subRoot = subquery.from(EntityClass.class);
    subquery.select(subRoot);
    subquery.distinct(true);

    criteriaQuery.where(root.in(subquery));
    ```

    위의 코드에서 `root.in(subquery)`는 메인 쿼리에서 서브쿼리의 결과를 필터링하는 역할을 합니다.

이러한 대안을 시도하고 여전히 문제가 있다면, 사용하는 JPA 구현체(예: Hibernate)와 데이터베이스의 버전, 설정 등을 고려하여 문제를 해결해야 합니다. 데이터베이스 벤더에 따라 `DISTINCT`를 적용하는 방식이 다를 수 있기 때문입니다.

------------------------

`CriteriaQuery.distinct(true)`를 사용하여 JPA Criteria API에서 DISTINCT한 결과를 얻는 방법은 일반적으로 문제가 없어야 합니다. 그러나 경우에 따라 데이터베이스 및 JPA 구현체에 따라 다를 수 있습니다. 만약 예상대로 작동하지 않는다면 몇 가지 확인해야 할 사항이 있습니다.

1. **JPA 구현체 버전 확인**: 사용 중인 JPA 구현체 (예: Hibernate)의 버전을 확인하세요. 특정 버전에서는 DISTINCT 처리에 대한 동작이 다를 수 있습니다.

2. **데이터베이스 지원 확인**: 데이터베이스가 DISTINCT를 올바르게 지원하는지 확인해야 합니다. 어떤 데이터베이스는 DISTINCT를 특정한 방식으로 처리할 수 있습니다.

3. **쿼리 확인**: 생성된 JPQL 쿼리를 확인하여 DISTINCT가 정상적으로 반영되었는지 확인하세요. Hibernate의 경우 `org.hibernate.SQL` 로거를 통해 실제 실행되는 SQL 쿼리를 확인할 수 있습니다.

   ```xml
   <!-- logback.xml 또는 log4j2.xml에 추가 -->
   <logger name="org.hibernate.SQL" level="DEBUG" />
   ```

4. **영속성 컨텍스트 확인**: 결과가 영속성 컨텍스트에 이미 로드되어 있는 경우, 중복 제거가 영속성 컨텍스트 내에서 이루어질 수 있습니다. 새로운 영속성 컨텍스트를 생성하여 테스트해보세요.

5. **다른 방법 시도**: 다른 방법으로 DISTINCT를 수행하는 것도 고려할 수 있습니다. 예를 들어, `GROUP BY`를 사용하거나 서브쿼리를 활용할 수 있습니다.

```java
CriteriaQuery<EntityClass> criteriaQuery = criteriaBuilder.createQuery(EntityClass.class);
Root<EntityClass> root = criteriaQuery.from(EntityClass.class);

Subquery<EntityClass> subquery = criteriaQuery.subquery(EntityClass.class);
Root<EntityClass> subRoot = subquery.from(EntityClass.class);
subquery.select(subRoot);
subquery.distinct(true);

criteriaQuery.where(root.in(subquery));
```

만약 여전히 문제가 있다면, 사용 중인 JPA 구현체와 데이터베이스에 대한 자세한 정보를 확인하여 해당 문제를 해결하는 데 도움이 되는 것들을 찾을 수 있습니다.
