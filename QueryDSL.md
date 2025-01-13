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
