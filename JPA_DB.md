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
