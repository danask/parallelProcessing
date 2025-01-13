

문제를 보니 IntelliJ에서 제공하는 수정 제안이 `EntityManager`의 처리 방식과 호환되지 않는 것 같습니다. `Supplier<EntityManager>`로의 캐스팅을 시도하는 방식은 잘못된 접근일 가능성이 높으며, 특히 Spring Boot와 JPA 표준의 관점에서는 더 명확하고 일관된 설정 방법을 사용하는 것이 중요합니다.

아래에 문제를 해결하기 위한 올바른 접근 방법을 단계별로 설명하겠습니다.

---

### 1. **Spring Boot에서 `@PersistenceContext`의 기본 설정 확인**
`@PersistenceContext`는 일반적으로 기본 `EntityManager`를 주입받습니다. `unitName`을 지정하는 것은 다중 데이터 소스 환경에서만 필요합니다. 단일 데이터 소스 설정인 경우에는 `unitName`을 제거하고 기본 `EntityManager`를 사용하도록 설정합니다.

```java
@PersistenceContext
private EntityManager entityManager;
```

---

### 2. **다중 데이터 소스 환경에서 `unitName` 설정**
만약 다중 데이터 소스 환경이라면, Spring Boot에서는 `@Primary` 애너테이션과 함께 데이터 소스를 설정하고 `@PersistenceContext`와 `unitName`을 연결해야 합니다.

#### 데이터 소스 및 `EntityManagerFactory` 설정:
```java
@Configuration
@EnableTransactionManagement
public class PrimaryDataSourceConfig {

    @Primary
    @Bean(name = "primaryDataSource")
    @ConfigurationProperties(prefix = "spring.datasource.primary")
    public DataSource dataSource() {
        return DataSourceBuilder.create().build();
    }

    @Primary
    @Bean(name = "primaryEntityManagerFactory")
    public LocalContainerEntityManagerFactoryBean entityManagerFactory(
            EntityManagerFactoryBuilder builder,
            @Qualifier("primaryDataSource") DataSource dataSource) {
        return builder
                .dataSource(dataSource)
                .packages("com.example.yourpackage")
                .persistenceUnit("primaryEntityManager")
                .build();
    }

    @Primary
    @Bean(name = "primaryTransactionManager")
    public PlatformTransactionManager transactionManager(
            @Qualifier("primaryEntityManagerFactory") EntityManagerFactory entityManagerFactory) {
        return new JpaTransactionManager(entityManagerFactory);
    }
}
```

#### `@PersistenceContext` 사용:
```java
@PersistenceContext(unitName = "primaryEntityManager")
private EntityManager entityManager;
```

---

### 3. **Spring Boot 3.x에서 `Supplier<EntityManager>` 처리**
Hibernate 6.x 및 Spring Boot 3.x에서는 `EntityManager`의 생성 방식이 약간 변경되었습니다. 대신, `EntityManagerFactory`를 사용하여 `EntityManager`를 생성할 수 있습니다.

#### 수동으로 `EntityManager` 생성:
```java
@Autowired
private EntityManagerFactory entityManagerFactory;

public void findCustomData() {
    EntityManager entityManager = entityManagerFactory.createEntityManager();
    JPAQueryFactory queryFactory = new JPAQueryFactory(entityManager);
    // Your QueryDSL logic here
}
```

---

### 4. **정확한 QueryDSL 설정**
최신 QueryDSL 설정 방법에 맞게 `JPAQueryFactory`를 빈으로 등록합니다.

```java
@Configuration
public class QueryDslConfig {

    @Autowired
    private EntityManagerFactory entityManagerFactory;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManagerFactory.createEntityManager());
    }
}
```

---

### 5. **테스트 환경 점검**
다음 테스트 코드를 작성해 QueryDSL 설정이 올바르게 작동하는지 확인하세요:

```java
@SpringBootTest
public class QueryDslIntegrationTest {

    @Autowired
    private JPAQueryFactory queryFactory;

    @Test
    public void testQueryFactoryInitialization() {
        assertNotNull(queryFactory, "JPAQueryFactory should not be null");
    }
}
```

---

### 6. **의존성 확인**
`QueryDSL`과 JPA 설정이 충돌하지 않도록 Maven 의존성을 다시 확인하세요.

```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <version>5.0.0</version>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <version>5.0.0</version>
    <scope>provided</scope>
</dependency>
```

---

### 7. **캐스팅 문제 해결**
`Supplier<EntityManager>`를 사용하는 IntelliJ의 자동 수정 제안은 오히려 잘못된 접근입니다. 위의 방법을 사용하여 캐스팅 없이 올바르게 설정하도록 변경합니다.

---

### 정리
1. 단일 데이터 소스 환경에서는 `@PersistenceContext`만으로 충분합니다.
2. 다중 데이터 소스 환경에서는 `unitName`을 통해 명시적으로 연결하세요.
3. `EntityManagerFactory`를 직접 사용하여 `EntityManager`를 생성하는 방법도 가능합니다.
4. IntelliJ의 자동 수정 제안은 잘못된 접근이므로 무시하고, 표준적인 Spring Boot 설정을 따르세요.

위의 방법으로도 해결되지 않으면, 추가 에러 메시지나 관련 설정을 공유해 주시면 더 도움을 드릴 수 있습니다.


----------------

Spring Boot 3.2.7과 JDK 17 환경에서 `EntityManager` 관련 문제가 발생하면서, 다른 JPA API(예: `CriteriaBuilder`)는 잘 동작한다면, 특정한 문제를 더 좁혀 확인할 필요가 있습니다. 아래는 해당 문제를 해결하기 위한 몇 가지 점검 사항과 수정 방법입니다.

---

### 주요 원인 분석

1. **QueryDSL 설정 문제**  
   - `JPAQueryFactory`는 `EntityManager`를 기반으로 작동하므로, `EntityManager`를 올바르게 주입받아야 합니다. 
   - `CriteriaBuilder`가 동작한다면, `EntityManager`는 이미 JPA 컨텍스트에서 제대로 생성되고 있습니다.  
     따라서, 문제는 `@PersistenceContext`와 QueryDSL 설정 간의 호환성 또는 사용 방식의 차이로 보입니다.

2. **Spring Boot 3.2.7 / Hibernate 6.x 변경 사항**  
   Hibernate 6.x와 Spring Boot 3.x 환경에서는 일부 API와 기본 동작이 변경되었습니다.  
   예를 들어, Hibernate 6에서는 `EntityManager` 사용과 관련한 동작이 약간 다르게 처리될 수 있습니다.

3. **JPAQueryFactory 초기화 시점 문제**  
   `EntityManager`가 Spring 컨텍스트에서 완전히 초기화되기 전에 `JPAQueryFactory`를 생성하려 하면 문제가 발생할 수 있습니다.

---

### 해결 방안

#### 1. **기본적인 QueryDSL 설정 점검**
QueryDSL 설정은 `EntityManager`와 `JPAQueryFactory`를 정확히 연결해야 합니다. 다음과 같이 설정합니다:

```java
@Configuration
public class QueryDslConfig {

    @PersistenceContext
    private EntityManager entityManager;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

---

#### 2. **EntityManagerFactory에서 EntityManager 공급**
Spring Boot 3.x에서는 `EntityManager`를 `Supplier<EntityManager>`로 사용해야 할 수 있습니다. 아래와 같이 설정을 변경해 보세요:

```java
@Configuration
public class QueryDslConfig {

    @PersistenceUnit
    private EntityManagerFactory entityManagerFactory;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManagerFactory.createEntityManager());
    }
}
```

---

#### 3. **동적으로 EntityManager 주입**
Spring 컨텍스트에서 동적으로 `EntityManager`를 주입할 수 있습니다. `@Bean` 등록 시 컨텍스트에서 동작하도록 변경합니다:

```java
@Configuration
public class QueryDslConfig {

    @Bean
    @Scope("prototype")
    public EntityManager entityManager(EntityManagerFactory emf) {
        return emf.createEntityManager();
    }

    @Bean
    public JPAQueryFactory jpaQueryFactory(EntityManager entityManager) {
        return new JPAQueryFactory(entityManager);
    }
}
```

---

#### 4. **QueryDSL을 직접 테스트**
QueryDSL이 작동하는지 확인하려면, 테스트 코드를 작성해 확인합니다.  
`JPAQueryFactory`가 올바르게 생성되는지 확인하기 위해 다음과 같이 테스트합니다:

```java
@RunWith(SpringRunner.class)
@SpringBootTest
public class QueryDslTest {

    @Autowired
    private JPAQueryFactory queryFactory;

    @Test
    public void testQueryDslSetup() {
        Assertions.assertNotNull(queryFactory, "JPAQueryFactory should not be null");
    }
}
```

---

#### 5. **Annotation Processor 활성화**
QueryDSL의 Q 클래스 생성이 제대로 되지 않으면 문제가 발생할 수 있습니다. `QTable` 관련 클래스가 프로젝트에 존재하는지 확인하고, IntelliJ 설정에서 **Annotation Processor**가 활성화되어 있는지 확인합니다:

1. `Preferences > Build, Execution, Deployment > Compiler > Annotation Processors`
2. **Enable annotation processing** 체크.

---

#### 6. **의존성 충돌 확인**
QueryDSL 의존성과 JPA 관련 의존성이 충돌하지 않도록 확인합니다. Maven 의존성을 아래처럼 설정하세요:

```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <version>5.0.0</version>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <version>5.0.0</version>
    <scope>provided</scope>
</dependency>
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

---

#### 7. **Spring Boot 및 Hibernate 설정 확인**
Spring Boot 3.x와 Hibernate 6.x에서 JPA 설정을 다시 확인하세요. `application.yml` 또는 `application.properties`에서 아래 설정을 확인합니다:

```properties
spring.jpa.hibernate.ddl-auto=update
spring.jpa.open-in-view=true
spring.datasource.url=jdbc:postgresql://localhost:5432/yourdb
spring.datasource.username=youruser
spring.datasource.password=yourpassword
```

---

### 추가 확인: 
1. **`@Entity` 클래스의 정의 확인**: QueryDSL Q 클래스가 정상적으로 생성되었는지 확인합니다.
2. **로그 확인**: 실행 시 `entityManager` 관련 에러 로그를 확인하세요.
3. **수동으로 EntityManagerFactory 사용**: 필요한 경우 `EntityManagerFactory`를 사용해 수동으로 `EntityManager`를 생성하는 방식도 가능합니다.

위의 방법으로도 문제가 해결되지 않으면 추가 로그와 환경 정보를 공유해 주세요.

---------------

`@PersistenceContext`를 사용할 때 `EntityManager`가 `cannot resolve`로 표시되는 문제는 주로 다음과 같은 원인으로 발생합니다:

---

### 1. **의존성 누락**
`EntityManager`는 JPA의 핵심 클래스이므로, 프로젝트의 `pom.xml`에 JPA 관련 의존성이 추가되어 있어야 합니다. 확인해야 할 의존성:

#### Maven 의존성:
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-data-jpa</artifactId>
</dependency>
```

---

### 2. **Spring Context 설정 문제**
JPA 관련 설정이 제대로 구성되지 않았다면 `EntityManager`를 주입받을 수 없습니다. 이를 해결하려면 `@EnableJpaRepositories`를 올바르게 설정하고, JPA 설정 파일이 존재해야 합니다.

#### 주요 설정:
- `@EnableJpaRepositories`를 적절한 패키지 경로에 설정.
- `application.yml` 또는 `application.properties`에 데이터베이스 연결 정보 추가.

```properties
spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
spring.datasource.username=myuser
spring.datasource.password=mypassword
spring.jpa.hibernate.ddl-auto=update
spring.jpa.show-sql=true
```

---

### 3. **EntityManager 사용 시 Context 문제**
`EntityManager`는 `@PersistenceContext`를 통해 관리되며, 이는 Spring의 `ApplicationContext`에 의해 관리됩니다. 만약 스프링 부트 프로젝트가 아니라거나, JPA 관련 빈 설정이 누락되었다면 이 문제가 발생할 수 있습니다.

#### 해결 방법:
- `@Configuration` 클래스에 JPA 설정 추가:
```java
@Configuration
@EnableJpaRepositories(basePackages = "com.example.repository")
public class JpaConfig {
}
```

---

### 4. **Spring Boot 버전 문제**
Spring Boot 3.x에서는 Hibernate 6.x 버전이 사용되며, 일부 API가 변경되었습니다. `EntityManager`를 직접 사용하는 경우에는 `Supplier<EntityManager>`를 사용하는 방식으로 변경이 필요할 수 있습니다.

#### 수정 예제:
```java
@PersistenceContext
private Supplier<EntityManager> entityManagerSupplier;

public List<MyEntity> findCustomData() {
    JPAQueryFactory queryFactory = new JPAQueryFactory(entityManagerSupplier.get());
    // Your QueryDSL logic here
}
```

---

### 5. **라이브러리 충돌 문제**
`querydsl-jpa` 버전과 `spring-boot-starter-data-jpa`의 호환성을 확인하세요. QueryDSL 5.x 버전과 Spring Boot 3.x는 호환됩니다.

#### 의존성 정리:
```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <version>5.0.0</version>
</dependency>
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-apt</artifactId>
    <version>5.0.0</version>
    <scope>provided</scope>
</dependency>
```

---

### 6. **IntelliJ 설정 문제**
IntelliJ의 `Build` 또는 `Maven` 설정이 올바르게 동작하지 않아 오류가 발생할 수 있습니다.

#### 해결 방법:
1. `File > Invalidate Caches and Restart`를 실행.
2. `pom.xml`을 다시 새로 고침.
3. `Preferences > Build, Execution, Deployment > Compiler > Annotation Processors`에서 `Enable annotation processing`을 활성화.

---

### 7. **Hibernate 6와 Spring Boot 3에서 변경된 사항**
Hibernate 6.x에서 일부 `EntityManager` 관련 동작이 변경되었습니다. `entityManager`를 빈으로 등록하지 않으면 `cannot resolve` 문제가 발생할 수 있습니다.

#### 변경된 코드:
```java
@Bean
public Supplier<EntityManager> entityManagerSupplier(EntityManagerFactory emf) {
    return emf::createEntityManager;
}
```

---

### 최종 코드 예제:

```java
@Configuration
public class QueryDslConfig {

    @PersistenceContext
    private Supplier<EntityManager> entityManagerSupplier;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManagerSupplier.get());
    }
}
``` 

위 방법으로 문제가 해결되지 않으면 `EntityManagerFactory` 설정 문제를 확인해야 합니다.


---------------

`JPAQueryFactory`에서 `EntityManager`를 직접 주입받지 않고 다른 방법으로 사용하는 방법도 있습니다. 예를 들어, SQL Repository에서 `EntityManager`를 간접적으로 활용할 수 있습니다. 여기서는 몇 가지 대안을 설명하고 예제를 제공하겠습니다.

---

### 1. **`@PersistenceContext`를 직접 사용하는 방법**

`EntityManager`를 주입받아 `JPAQueryFactory`를 SQL Repository에서 직접 생성할 수 있습니다.

#### Repository 예제:
```java
@Repository
public class CustomQueryRepository {

    @PersistenceContext
    private EntityManager entityManager;

    public List<MyEntity> findCustomData() {
        JPAQueryFactory queryFactory = new JPAQueryFactory(entityManager);

        QMyEntity myEntity = QMyEntity.myEntity;

        return queryFactory.selectFrom(myEntity)
                .where(myEntity.field.eq("value"))
                .fetch();
    }
}
```

#### 장점:
- `EntityManager`를 SQL Repository에서 직접 사용할 수 있습니다.
- 간단한 설정으로 유연하게 사용할 수 있습니다.

---

### 2. **`EntityManager`를 Constructor Injection으로 주입받기**

`EntityManager`를 생성자로 주입받아 `JPAQueryFactory`를 활용할 수도 있습니다.

#### Repository 예제:
```java
@Repository
public class CustomQueryRepository {

    private final JPAQueryFactory queryFactory;

    public CustomQueryRepository(EntityManager entityManager) {
        this.queryFactory = new JPAQueryFactory(entityManager);
    }

    public List<MyEntity> findCustomData() {
        QMyEntity myEntity = QMyEntity.myEntity;

        return queryFactory.selectFrom(myEntity)
                .where(myEntity.field.eq("value"))
                .fetch();
    }
}
```

#### 장점:
- 의존성 주입을 통해 `EntityManager`를 깔끔하게 관리할 수 있습니다.
- 테스트 시 `EntityManager`를 쉽게 Mocking 할 수 있습니다.

---

### 3. **Spring Configuration에서 Bean으로 선언**

`JPAQueryFactory`를 Spring Bean으로 등록하고 Repository에서 주입받아 사용하는 방법입니다.

#### Configuration:
```java
@Configuration
public class QueryDslConfig {

    @Bean
    public JPAQueryFactory jpaQueryFactory(EntityManager entityManager) {
        return new JPAQueryFactory(entityManager);
    }
}
```

#### Repository:
```java
@Repository
public class CustomQueryRepository {

    private final JPAQueryFactory queryFactory;

    public CustomQueryRepository(JPAQueryFactory queryFactory) {
        this.queryFactory = queryFactory;
    }

    public List<MyEntity> findCustomData() {
        QMyEntity myEntity = QMyEntity.myEntity;

        return queryFactory.selectFrom(myEntity)
                .where(myEntity.field.eq("value"))
                .fetch();
    }
}
```

#### 장점:
- `JPAQueryFactory`를 여러 Repository에서 재사용할 수 있습니다.
- `EntityManager`를 명시적으로 처리하지 않아 코드가 간결해집니다.

---

### 4. **Native SQL로 처리하기 (EntityManager 직접 사용)**

`JPAQueryFactory` 대신 `EntityManager`의 Native SQL Query 기능을 사용해도 됩니다.

#### Repository 예제:
```java
@Repository
public class CustomQueryRepository {

    @PersistenceContext
    private EntityManager entityManager;

    public List<Object[]> findCustomData() {
        String sql = "SELECT * FROM my_table WHERE field = :value";
        return entityManager.createNativeQuery(sql)
                .setParameter("value", "desired_value")
                .getResultList();
    }
}
```

#### 장점:
- 복잡한 쿼리를 작성하거나 QueryDSL이 지원하지 않는 기능을 활용할 수 있습니다.
- 직접 SQL을 작성해야 하는 상황에서 유용합니다.

---

### 정리

- **QueryDSL**을 이용한 `JPAQueryFactory` 생성은 `@Bean`, 생성자 주입, 혹은 `@PersistenceContext`를 사용하여 쉽게 구현 가능합니다.
- **SQL Repository**에서는 `EntityManager`를 통해 Native SQL과 QueryDSL을 자유롭게 혼합할 수 있습니다.
- **권장 방식**: 대부분의 경우, `JPAQueryFactory`를 Spring Bean으로 등록하고 Constructor Injection을 사용하는 방법이 가장 깔끔하고 확장 가능성이 높습니다. 

원하는 방식에 따라 위 예제를 응용하면 됩니다!
-----------------

`pom.xml`에 필요한 의존성을 추가했다고 하더라도, **특정 설정이나 동작을 구현**하기 위해 추가적인 `@Configuration` 설정이 필요할 수 있습니다. 이는 의존성만으로는 런타임에서 자동으로 동작을 수행하도록 설정되지 않기 때문입니다.

### **필요한 추가 설정이 필요한 이유**
1. **의존성만으로는 Bean이 자동 생성되지 않는 경우**
   - `JPAQueryFactory`와 같은 객체는 Spring이 기본적으로 관리하는 Bean이 아니기 때문에 명시적으로 설정해야 합니다.
   - 예: `@PersistenceContext`를 사용해 `EntityManager`를 주입하고 이를 기반으로 `JPAQueryFactory` Bean을 생성.

2. **자동 설정이 비활성화된 경우**
   - Spring Boot Starter가 제공하는 기본 자동 설정이 제대로 동작하지 않는 경우, 명시적으로 구성 파일을 만들어야 합니다.
   - 예: Spring Data JPA가 활성화되었는지 확인(`@EnableJpaRepositories`).

3. **커스터마이징이 필요한 경우**
   - 기본 동작을 변경하거나 특정 요구사항(예: 특정 Bean 이름, 프로파일별 설정 등)에 따라 동작을 조정하려면 설정 파일이 필요합니다.
   - 예: 특정 데이터 소스를 다르게 설정하거나 다중 데이터 소스를 사용하는 경우.

---

### **QueryDSL 설정 예제**
#### `pom.xml`
```xml
<dependency>
    <groupId>com.querydsl</groupId>
    <artifactId>querydsl-jpa</artifactId>
    <version>5.0.0</version>
</dependency>
<dependency>
    <groupId>javax.annotation</groupId>
    <artifactId>javax.annotation-api</artifactId>
    <version>1.3.2</version>
</dependency>
<plugin>
    <groupId>com.mysema.maven</groupId>
    <artifactId>apt-maven-plugin</artifactId>
    <version>1.1.3</version>
    <executions>
        <execution>
            <goals>
                <goal>process</goal>
            </goals>
            <configuration>
                <outputDirectory>target/generated-sources</outputDirectory>
                <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
            </configuration>
        </execution>
    </executions>
</plugin>
```

#### **Spring Configuration**
`JPAQueryFactory`를 사용하는 경우, 아래와 같이 Bean으로 등록해야 합니다.
```java
@Configuration
public class QueryDslConfig {

    @PersistenceContext
    private EntityManager entityManager;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

#### **자동 설정 확인**
Spring Boot의 자동 설정이 정상적으로 동작하는지 확인하려면 다음을 점검해야 합니다.
1. `application.properties` 또는 `application.yml`에 JPA 관련 기본 설정이 포함되어야 합니다:
   ```properties
   spring.jpa.show-sql=true
   spring.jpa.hibernate.ddl-auto=update
   spring.datasource.url=jdbc:postgresql://localhost:5432/mydb
   spring.datasource.username=myuser
   spring.datasource.password=mypassword
   ```

2. `@SpringBootApplication` 클래스에 `@EnableJpaRepositories`가 포함되어 있어야 합니다:
   ```java
   @SpringBootApplication
   @EnableJpaRepositories
   public class Application {
       public static void main(String[] args) {
           SpringApplication.run(Application.class, args);
       }
   }
   ```

---

### **자동 설정과 커스터마이징의 조화**
- **자동 설정이 동작하지 않는 경우**: Spring Boot Starter의 의존성이 누락되었거나, 자동 설정을 명시적으로 비활성화했을 가능성이 있습니다.
- **직접 구성해야 하는 경우**: 기본 Bean 정의가 프로젝트 요구사항에 맞지 않는 경우 `@Configuration`을 이용해 커스터마이징이 필요합니다.

### **Spring Boot와 QueryDSL의 조합: Best Practice**
- 프로젝트 초기 단계에서는 기본 자동 설정과 표준 `@Configuration` 클래스를 사용합니다.
- 점진적으로 `QueryDSL`과 같은 동적 쿼리 기능을 추가하면서 필요한 설정만 추가해 복잡도를 관리합니다.

https://github.com/arifng/spring-jpa-querydsl/blob/master/src/main/java/com/arifng/jpaquerydsl/SpringJpaQuerydslApplication.java 

`resolveJoins` 메서드에서 동적 조인 처리 시 테이블 수와 스키마가 달라지는 문제를 해결하려면 아래와 같은 전략을 사용할 수 있습니다. 

### **1. 메타데이터 기반 동적 처리**
- **테이블 갯수와 스키마**를 메타데이터 객체에 저장.
- `JoinInfo` 객체에 테이블 간 관계 및 조인 조건을 정의.

#### **JoinInfo 클래스**
```java
public class JoinInfo {
    private QTable leftTable;
    private QTable rightTable;
    private String joinType; // "INNER", "LEFT", "RIGHT"
    private Predicate condition;

    // Constructor, getters, and setters
}
```

#### **QTable 인터페이스**
테이블 스키마를 동적으로 처리할 수 있도록 `QTable`을 인터페이스로 추상화하고 이를 구현한 각 `Q` 클래스를 생성합니다.
```java
public interface QTable {
    EntityPathBase<?> getEntity();
}
```

예를 들어, `QAppTable`을 구현:
```java
public class QAppTable implements QTable {
    public static final QAppTable appTable = new QAppTable();
    private QAppTable() {}

    public final StringPath appName = createString("appName");
    public final StringPath customerId = createString("customerId");

    @Override
    public EntityPathBase<?> getEntity() {
        return appTable;
    }
}
```

---

### **2. resolveJoins 메서드 구현**
`resolveJoins`는 입력된 테이블, 조건, 조인 타입에 따라 동적으로 쿼리를 생성합니다.

```java
public static JPAQuery<?> resolveJoins(JPAQuery<?> query, List<JoinInfo> joins, QTable... tables) {
    for (JoinInfo join : joins) {
        EntityPathBase<?> leftEntity = join.getLeftTable().getEntity();
        EntityPathBase<?> rightEntity = join.getRightTable().getEntity();

        switch (join.getJoinType().toUpperCase()) {
            case "INNER":
                query = query.innerJoin(rightEntity).on(join.getCondition());
                break;
            case "LEFT":
                query = query.leftJoin(rightEntity).on(join.getCondition());
                break;
            case "RIGHT":
                query = query.rightJoin(rightEntity).on(join.getCondition());
                break;
            default:
                throw new IllegalArgumentException("Invalid join type: " + join.getJoinType());
        }
    }
    return query;
}
```

---

### **3. 예제: 여러 테이블 조인**
#### **메타데이터 기반 조인 정의**
```java
List<JoinInfo> joinInfos = new ArrayList<>();
joinInfos.add(new JoinInfo(
    QAppTable.appTable,
    QCustomerTable.customerTable,
    "INNER",
    QAppTable.appTable.customerId.eq(QCustomerTable.customerTable.id)
));
joinInfos.add(new JoinInfo(
    QCustomerTable.customerTable,
    QOrderTable.orderTable,
    "LEFT",
    QCustomerTable.customerTable.id.eq(QOrderTable.orderTable.customerId)
));
```

#### **JPAQuery에서 조인 적용**
```java
JPAQuery<?> query = new JPAQuery<>(entityManager);
query = resolveJoins(query, joinInfos, QAppTable.appTable, QCustomerTable.customerTable, QOrderTable.orderTable);

List<Tuple> result = query.select(
        QAppTable.appTable.appName,
        QCustomerTable.customerTable.customerName,
        QOrderTable.orderTable.orderId
    )
    .from(QAppTable.appTable)
    .fetch();
```

---

### **4. 스키마 차이에 따른 처리**
스키마가 다른 경우 테이블 별로 별도의 네임스페이스를 활용하거나 동적 SQL을 생성하는 방식을 사용할 수 있습니다.
- 스키마 이름을 메타데이터에 저장하고 `resolveJoins`에서 `@Table(schema = "schemaName")` 정보를 활용.

---

### **5. 생성된 SQL 확인**
QueryDSL은 쿼리를 실행하기 전에 SQL을 출력할 수 있습니다:
```java
String sql = query.toString();
System.out.println(sql);
```

---

이 방식으로 동적 테이블과 조인을 처리하면 테이블 스키마와 관계 정의를 분리할 수 있어 유지보수성과 확장성이 크게 향상됩니다.


`EntityManager`가 `cannot resolve`라는 오류를 표시한다면, 문제는 주로 다음과 같은 원인 중 하나일 수 있습니다. 아래 해결 방법을 하나씩 확인해보세요.

---

### 1. **JPA 의존성이 누락됨**
   `EntityManager`는 JPA의 일부이므로 프로젝트에 JPA 관련 의존성이 추가되어야 합니다.

   #### 확인할 점
   `pom.xml`에 JPA 의존성이 포함되어 있는지 확인하세요.
   ```xml
   <dependencies>
       <dependency>
           <groupId>org.springframework.boot</groupId>
           <artifactId>spring-boot-starter-data-jpa</artifactId>
       </dependency>
       <!-- 데이터베이스 드라이버도 필요 -->
       <dependency>
           <groupId>org.postgresql</groupId>
           <artifactId>postgresql</artifactId>
       </dependency>
   </dependencies>
   ```

   #### 해결
   의존성이 누락된 경우 위 코드를 추가한 뒤, Maven의 `Reload Project`를 수행하세요.

---

### 2. **JPA 설정이 올바르게 되어 있지 않음**
   `EntityManager`는 JPA가 활성화된 상태에서만 주입됩니다. JPA가 설정되어 있는지 확인하세요.

   #### 확인할 점
   `application.properties` 또는 `application.yml`에 JPA와 관련된 설정이 포함되어 있는지 확인하세요.

   ##### 예: `application.properties`
   ```properties
   spring.datasource.url=jdbc:postgresql://localhost:5432/your_database
   spring.datasource.username=your_username
   spring.datasource.password=your_password
   spring.jpa.hibernate.ddl-auto=update
   spring.jpa.show-sql=true
   ```

   ##### 예: `application.yml`
   ```yaml
   spring:
     datasource:
       url: jdbc:postgresql://localhost:5432/your_database
       username: your_username
       password: your_password
     jpa:
       hibernate:
         ddl-auto: update
       show-sql: true
   ```

---

### 3. **Spring Data JPA의 Auto Configuration이 비활성화됨**
   JPA 관련 자동 구성이 비활성화되어 `EntityManager`가 주입되지 않을 수 있습니다.

   #### 확인할 점
   `@SpringBootApplication`에 `exclude` 옵션이 추가되어 있지 않은지 확인하세요.
   ```java
   @SpringBootApplication(exclude = {DataSourceAutoConfiguration.class})
   public class Application { ... }
   ```

   #### 해결
   `exclude`로 인해 JPA 또는 데이터베이스 구성이 비활성화된 경우, 해당 부분을 제거합니다.
   ```java
   @SpringBootApplication
   public class Application { ... }
   ```

---

### 4. **Maven 의존성 업데이트 문제**
   `EntityManager`를 포함한 클래스가 IntelliJ에서 인식되지 않을 수 있습니다.

   #### 확인할 점
   IntelliJ에서 Maven 프로젝트가 제대로 동기화되었는지 확인하세요.

   #### 해결
   1. IntelliJ의 Maven 탭에서 `Reload Project` 버튼을 클릭합니다.
   2. 또는, 터미널에서 아래 명령어를 실행하세요:
      ```bash
      mvn clean install
      ```

---

### 5. **잘못된 Import**
   `EntityManager`를 올바른 패키지에서 가져와야 합니다.

   #### 확인할 점
   `javax.persistence.EntityManager` 또는 `jakarta.persistence.EntityManager`로 import되어 있는지 확인하세요.

   #### 해결
   패키지가 올바르지 않다면, 다음 중 하나로 변경합니다:
   ```java
   import javax.persistence.EntityManager;  // Java 8/11
   // 또는
   import jakarta.persistence.EntityManager;  // Java 17+
   ```

---

### 6. **JDK 버전 확인**
   `EntityManager`의 패키지가 Java 버전에 따라 달라졌습니다.

   #### 확인할 점
   - **Java 8, 11**: `javax.persistence.EntityManager`
   - **Java 17+**: `jakarta.persistence.EntityManager`

   #### 해결
   프로젝트의 Java 버전에 따라 의존성과 import를 수정하세요. Java 17 이상이라면, `spring-boot-starter-data-jpa`가 Jakarta Persistence를 사용하도록 자동으로 설정됩니다.

---

위 단계를 확인한 후에도 문제가 지속되면, IntelliJ의 캐시를 무효화하거나 프로젝트를 다시 빌드해보세요:

1. **IntelliJ 캐시 무효화**
   - `File > Invalidate Caches > Invalidate and Restart`

2. **프로젝트 다시 빌드**
   - `Build > Rebuild Project`

문제가 해결되지 않으면 현재의 설정 파일(`pom.xml`, `application.properties`)과 전체적인 프로젝트 구조를 공유해 주시면 추가로 도움을 드리겠습니다!



-------------

Spring Boot 3.2.7과 JDK 17 환경에서 기존에 `CriteriaBuilder`를 사용하는 JPA 리포지토리가 `primaryEntity`로 설정되어 있는 프로젝트에 QueryDSL을 통합하려면, 다음과 같은 설정이 필요합니다.

### 1. Maven 의존성 추가

`pom.xml` 파일에 QueryDSL 관련 의존성을 추가합니다.

```xml
<dependencies>
    <!-- QueryDSL JPA -->
    <dependency>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-jpa</artifactId>
        <version>5.0.0</version>
    </dependency>
    <!-- QueryDSL APT (Annotation Processor) -->
    <dependency>
        <groupId>com.querydsl</groupId>
        <artifactId>querydsl-apt</artifactId>
        <version>5.0.0</version>
        <scope>provided</scope>
    </dependency>
    <!-- 기타 필요한 의존성 -->
    <!-- ... -->
</dependencies>
```

또한, QueryDSL의 Q 클래스 생성을 위해 `maven-compiler-plugin`과 `maven-processor-plugin`을 설정합니다.

```xml
<build>
    <plugins>
        <!-- Maven Compiler Plugin -->
        <plugin>
            <groupId>org.apache.maven.plugins</groupId>
            <artifactId>maven-compiler-plugin</artifactId>
            <version>3.8.1</version>
            <configuration>
                <source>17</source>
                <target>17</target>
                <annotationProcessorPaths>
                    <path>
                        <groupId>com.querydsl</groupId>
                        <artifactId>querydsl-apt</artifactId>
                        <version>5.0.0</version>
                    </path>
                </annotationProcessorPaths>
            </configuration>
        </plugin>
        <!-- Maven Processor Plugin -->
        <plugin>
            <groupId>org.bsc.maven</groupId>
            <artifactId>maven-processor-plugin</artifactId>
            <version>3.3.3</version>
            <executions>
                <execution>
                    <goals>
                        <goal>process</goal>
                    </goals>
                    <phase>generate-sources</phase>
                    <configuration>
                        <processors>
                            <processor>com.querydsl.apt.jpa.JPAAnnotationProcessor</processor>
                        </processors>
                    </configuration>
                </execution>
            </executions>
        </plugin>
    </plugins>
</build>
```

### 2. QueryDSL 설정 클래스 추가

`JPAQueryFactory`를 빈으로 등록하여 애플리케이션 전역에서 사용할 수 있도록 설정합니다.

```java
import com.querydsl.jpa.impl.JPAQueryFactory;
import jakarta.persistence.EntityManager;
import jakarta.persistence.PersistenceContext;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class QueryDslConfig {

    @PersistenceContext(unitName = "primaryEntityManager")
    private EntityManager entityManager;

    @Bean
    public JPAQueryFactory jpaQueryFactory() {
        return new JPAQueryFactory(entityManager);
    }
}
```

### 3. Q 클래스 생성 확인

QueryDSL은 엔티티 클래스를 기반으로 Q 클래스를 생성합니다. IntelliJ IDEA에서 Q 클래스가 올바르게 생성되었는지 확인하고, 생성되지 않았다면 `mvn clean compile` 명령을 실행하여 생성합니다.

### 4. 기존 `CriteriaBuilder`와의 통합

기존에 `CriteriaBuilder`를 사용하는 리포지토리 로직과 QueryDSL을 함께 사용할 수 있습니다. 필요에 따라 두 가지 방식을 혼용하여 사용하거나, 점진적으로 QueryDSL로 마이그레이션할 수 있습니다.

### 5. GitHub 예제 프로젝트 참고

아래는 Spring Boot 3.x와 QueryDSL을 통합한 예제 프로젝트입니다. IntelliJ IDEA에서 바로 열어 참고할 수 있습니다.

- [Spring Boot 3와 QueryDSL 통합 예제](https://github.com/jskim1991/spring-boot-querydsl-example)

### 참고 자료

- [Spring Boot 3에서 QueryDSL 설정하기](https://jskim1991.medium.com/spring-boot-exploring-spring-boot-3-with-querydsl-part-2-7b563c382192)

위의 단계를 따라 설정하면, 기존 `CriteriaBuilder` 기반의 JPA 리포지토리와 함께 QueryDSL을 사용할 수 있습니다. 
