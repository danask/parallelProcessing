
YAML 빌드 시 에러가 나는 기준은 **메서드가 아니라 DTO 클래스의 필드 구조**입니다. 즉, YAML을 Java 객체로 매핑하는 과정에서 스프링이나 SnakeYAML은 다음을 기준으로 에러 여부를 판단합니다:

---

### ✅ 빌드 오류 판단 기준

1. **타겟 클래스 (예: `@ConfigurationProperties`로 바인딩되는 DTO)**
   → `FieldConfig`, `CategoryConfig`, `DdeMetadataProperties` 같은 DTO 클래스

2. **DTO 내부의 필드 이름과 타입이 YAML 구조와 일치하는지**
   → 예: YAML에 `joins: { measure: [...] }`인데 DTO가 `Map<String, List<JoinDetail>>`가 아니면 에러

3. **중첩 구조가 DTO에 없는데 YAML에 있는 경우**
   → 예: DTO에 `joins` 필드가 없는데 YAML에 `joins:`가 있으면 "unrecognized field" 오류 발생

4. **잘못된 리스트 구조**
   → 예: YAML에 리스트가 필요한데 객체처럼 정의하거나, 객체가 필요한데 리스트로 쓰면 오류

5. **타입 불일치**
   → 예: YAML에서 `sum: 123`인데 Java에서 `Map<String, String>`이면 타입 미스매치 오류

---

### ⚠️ 예시로 보는 문제 구조

#### ❌ 잘못된 경우: YAML

```yaml
joins:
  measure:
    target: "..."
    joinType: "..."
    on: "..."
```

#### 🧨 에러 메시지 예측

> `Cannot construct instance of java.util.List` (expected list but found object)

---

### ✅ 올바른 DTO 매핑 구조 예시

#### YAML 구조

```yaml
dde:
  measure:
    mv_app_usage:
      fields:
        totalRunTime:
          label: "Total Run Time"
          metric:
            sum: "Sum (EA)"
          joins:
            measure:
              - target: "..."
                joinType: "left"
                on: "..."
            dimension:
              - target: "..."
                joinType: "inner"
                on: "..."
```

#### DTO 구조

```java
@Data
public class DdeMetadataProperties {
    private Map<String, Map<String, CategoryConfig>> dde;
}

@Data
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields;
}

@Data
public class FieldConfig {
    private String label;
    private Map<String, String> metric;
    private Map<String, String> operator;

    // YAML에 맞게 수정
    private Map<String, List<JoinDetail>> joins;
}

@Data
public class JoinDetail {
    private String target;
    private String joinType;
    private String on;
}
```

---

### ✅ 결론: 어디서 판단하는가?

* **YAML 바인딩 시점**에 DTO 클래스 기준으로 오류 판단
* DTO 클래스에 정의되지 않은 필드가 있거나, 타입 불일치가 있으면 **build 시 실패**
* 메서드는 전혀 상관없음

---

필요하시면 현재 사용하는 `DdeMetadataProperties`, `CategoryConfig`, `FieldConfig` 구조를 전체 점검해서 YAML과 완전히 매칭되도록 만들어 드릴 수 있습니다. 원하시면 YAML 구조 예시와 함께 다시 알려주세요.


빌드 에러가 발생하는 이유는 거의 확실히 **YAML 구조와 Java 클래스(`FieldConfig`, `JoinTargets`) 간의 불일치** 때문입니다. 특히 이 부분이 문제입니다:

```yaml
joins:
  measure:
    - target: "measure:mv_app_usage:foreground_usage"
      joinType: "left"
      on: "measure:mv_app_usage:app_id = dimension:package:app_id"
```

Java에서 `FieldConfig.joins` 필드는 `JoinTargets` 타입인데, 이 클래스는 내부에 이렇게 되어 있죠:

```java
private List<JoinDetail> measure;
private List<JoinDetail> dimension;
private List<JoinDetail> filter;
```

즉, YAML에선 `joins.measure`가 `List<JoinDetail>` 여야 합니다.
**하지만 YAML에서 `joins.measure`가 `Map<String, Object>`처럼 해석되고 있을 가능성**이 높습니다. 이유는 다음과 같습니다:

---

### 🔍 문제 요약

```yaml
joins:
  measure:
    - target: ...
```

이 구조는 Spring Boot 또는 Jackson이 파싱할 때 문제 없이 `List<JoinDetail>`로 바꾸는 게 맞지만, 문제는 상위 구조입니다. `joins`는 `JoinTargets`이고, 이 객체를 Jackson이 직렬화할 때 예상하는 형식은 다음과 같아야 합니다:

```yaml
joins:
  measure:
    - target: ...
      joinType: ...
      on: ...
  dimension:
    - ...
```

이 경우 일반적으로 **`JoinTargets` 클래스가 정확히 등록되어야 하고, 파서가 이를 `FieldConfig`에 바인딩 가능해야** 합니다. 하지만 `FieldConfig.getJoins()`가 `JoinTargets`가 아닌 `Map`을 리턴하도록 오버라이드하면 타입 매칭이 안 되어 빌드 에러 또는 런타임 오류가 납니다.

---

### ✅ 해결 방법

#### 1. `FieldConfig.getJoins()` 메서드 잘못된 부분 확인

지금 `getJoinRecommendations()`에서는 다음과 같이 사용 중입니다:

```java
if (currentField == null || currentField.getJoins() == null) continue;

for (Map.Entry<String, List<JoinDetail>> entry : currentField.getJoins().asMap().entrySet()) {
```

즉, `joins`는 반드시 `JoinTargets` 타입이어야 하고, 그 내부 필드는 `List<JoinDetail>` 여야 합니다.

**→ `FieldConfig.getJoins()`를 `Map<String, List<JoinDetail>>`로 만들면 안 됩니다.**

#### 2. YAML 구조가 잘못되었을 가능성

지금 이 부분을 보세요:

```yaml
on: "dimension:device:device_id = dimension:device:device_id"
```

위 표현도 문제가 될 수 있습니다. 일반적으로 `"x = x"`는 쓸모가 없거나, 의미가 모호할 수 있습니다. **동일한 값을 비교하는 조인은 대부분 의미 없는 동등조건**으로 간주될 수 있어 필터링 대상이 됩니다. 더군다나 값에 `:`이 들어가면 YAML 해석에서 혼란을 줄 수 있으니 꼭 **문자열 전체를 쌍따옴표로 감싸주세요.**

---

### 🧪 정리된 예시 YAML

```yaml
dde:
  measure:
    mv_app_usage:
      label: "App Usage Daily"
      fields:
        totalRunTime:
          label: "Total Run Time"
          metric:
            sum: "Sum (EA)"
            avg: "Avg (%)"
          operator:
            lt: "<"
            eq: "="
            gt: ">"
          joins:
            measure:
              - target: "measure:mv_app_usage:foreground_usage"
                joinType: "left"
                on: "measure:mv_app_usage:app_id = dimension:package:app_id"
            dimension:
              - target: "dimension:device:device_id"
                joinType: "inner"
                on: "dimension:device:device_id = dimension:device:device_id"
              - target: "dimension:app:app_name"
                joinType: "left"
                on: "dimension:app:app_name = dimension:app:app_name"
```

---

### ✅ 점검 포인트

* `FieldConfig.joins` → `JoinTargets` 타입인지 확인
* `JoinTargets` 내부는 `List<JoinDetail>`인지 확인
* YAML 구조가 그에 맞게 `joins.measure`, `joins.dimension`이 배열로 돼 있는지 확인
* `application.yml` 전체에 `tabs` 대신 `spaces`만 사용하고 있는지 확인

---

필요하면 해당 YAML에 맞춰 전체 DTO 구조를 다시 정리해드릴게요. 원하시면 말씀 주세요!


```java

import java.util.*;
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class JoinRecommendation {
    private String from;  // e.g., "dimension:device:device_id"
    private String to;    // e.g., "dimension:app:app_name"
    private String joinType;
    private String on;
}

public class JoinGraphUtil {

    public static List<JoinRecommendation> getJoinRecommendations(List<String> startKeys, Map<String, Set<String>> graph) {
        List<JoinRecommendation> recommendations = new ArrayList<>();
        Set<String> visited = new HashSet<>();
        Queue<String> queue = new LinkedList<>(startKeys);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (visited.contains(current)) continue;
            visited.add(current);

            FieldConfig currentField = getFieldConfig(current);
            if (currentField == null || currentField.getJoins() == null) continue;

            for (Map.Entry<String, List<JoinDetail>> entry : currentField.getJoins().asMap().entrySet()) {
                String group = entry.getKey();
                List<JoinDetail> joinList = entry.getValue();
                if (joinList == null) continue;

                for (JoinDetail join : joinList) {
                    String target = join.getTarget();
                    if (!visited.contains(target)) queue.add(target);

                    String on = convertToFullPathOn(current, target, join.getOn());

                    recommendations.add(new JoinRecommendation(
                            current,
                            target,
                            join.getJoinType(),
                            on
                    ));
                }
            }

            Set<String> neighbors = graph.getOrDefault(current, Set.of());
            for (String neighbor : neighbors) {
                if (!visited.contains(neighbor)) queue.add(neighbor);
            }
        }

        return recommendations;
    }

    private static String convertToFullPathOn(String from, String to, String on) {
        // Example: "device_id = device_id" => "dimension:device:device_id = dimension:app:device_id"
        String[] parts = on.split("=");
        if (parts.length != 2) return on;
        String left = parts[0].trim();
        String right = parts[1].trim();

        String fromLeft = from.substring(0, from.lastIndexOf(":")) + ":" + left;
        String toRight = to.substring(0, to.lastIndexOf(":")) + ":" + right;

        return fromLeft + " = " + toRight;
    }

    // Dummy stub: Replace this with your actual implementation
    public static FieldConfig getFieldConfig(String fullKey) {
        return null;
    }
}

@Data
@NoArgsConstructor
@AllArgsConstructor
class JoinDetail {
    private String target;     // full path: e.g., "dimension:app:app_name"
    private String joinType;   // e.g., "inner", "left"
    private String on;         // e.g., "device_id = device_id"
}

@Data
@NoArgsConstructor
@AllArgsConstructor
class JoinTargets {
    private List<JoinDetail> measure;
    private List<JoinDetail> dimension;
    private List<JoinDetail> filter;

    public Map<String, List<JoinDetail>> asMap() {
        Map<String, List<JoinDetail>> map = new LinkedHashMap<>();
        if (measure != null) map.put("measure", measure);
        if (dimension != null) map.put("dimension", dimension);
        if (filter != null) map.put("filter", filter);
        return map;
    }
}

@Data
@NoArgsConstructor
@AllArgsConstructor
class FieldConfig {
    private String label;
    private Map<String, String> metric;
    private Map<String, String> operator;
    private JoinTargets joins;
}


```

------

좋아요! `getJoinRecommendations` 메소드에서 깊이 우선 탐색(DFS)을 너비 우선 탐색(BFS)으로 변경한 버전을 아래에 전체 코드로 제공드릴게요.

---

### ✅ 변경 사항 요약:

* **탐색 방식**: `DFS → BFS`
* **클래스 구조**는 그대로 (`JoinRecommendation`, `JoinConfig`, `JoinTargets` 등)

---

### 🔧 `JoinGraphUtil.java` (BFS 적용)

```java
import java.util.*;

public class JoinGraphUtil {

    public static List<JoinRecommendation> getJoinRecommendations(List<String> startKeys, DdeMetadataProperties dde) {
        Set<String> visited = new HashSet<>();
        Queue<String> queue = new LinkedList<>();
        List<JoinRecommendation> recommendations = new ArrayList<>();

        queue.addAll(startKeys);
        visited.addAll(startKeys);

        while (!queue.isEmpty()) {
            String currentKey = queue.poll();
            FieldConfig field = getFieldConfig(currentKey, dde);
            if (field == null || field.getJoins() == null) continue;

            processJoinList(currentKey, field.getJoins().getMeasure(), dde, visited, queue, recommendations);
            processJoinList(currentKey, field.getJoins().getDimension(), dde, visited, queue, recommendations);
            processJoinList(currentKey, field.getJoins().getFilter(), dde, visited, queue, recommendations);
        }

        return recommendations;
    }

    private static void processJoinList(
            String sourceKey,
            List<JoinConfig> joins,
            DdeMetadataProperties dde,
            Set<String> visited,
            Queue<String> queue,
            List<JoinRecommendation> recommendations
    ) {
        if (joins == null) return;

        for (JoinConfig join : joins) {
            String targetKey = join.getTarget();
            recommendations.add(new JoinRecommendation(
                    sourceKey, targetKey, join.getJoinType(), join.getOn()
            ));
            if (!visited.contains(targetKey)) {
                visited.add(targetKey);
                queue.add(targetKey);
            }
        }
    }

    public static FieldConfig getFieldConfig(String fullKey, DdeMetadataProperties dde) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        String group = parts[0];
        String category = parts[1];
        String field = parts[2];

        Map<String, CategoryConfig> groupMap = getGroupMap(group, dde);
        if (groupMap == null) return null;
        CategoryConfig categoryConfig = groupMap.get(category);
        if (categoryConfig == null) return null;
        return categoryConfig.getFields().get(field);
    }

    public static Map<String, CategoryConfig> getGroupMap(String group, DdeMetadataProperties dde) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dde.getDimension();
            case "measure" -> dde.getMeasure();
            case "filter" -> dde.getFilter();
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }
}
```

---

필요한 보조 클래스들은 이전에 작성해드린 내용에서 그대로 사용하시면 됩니다.

* `JoinRecommendation.java`
* `JoinConfig.java`
* `JoinTargets.java`

---

이제 `getJoinRecommendations`는 BFS 방식으로 탐색하여 조인 경로를 순차적으로 추천하며, 동일 노드의 중복 방문 없이 처리됩니다.
필요하시면 전체 파일 구조를 다시 `.zip` 형태로 묶거나, 테스트 코드까지 확장해드릴 수 있어요.


-----

좋습니다. `getJoinRecommendations` 메소드에서 `Map<String, Object>` 같은 구조를 사용하지 않고, 전용 엔티티 클래스를 사용하는 방식으로 리팩토링한 전체 코드를 아래에 정리했습니다.

---

### 🔧 1. `JoinRecommendation.java`

```java
import lombok.AllArgsConstructor;
import lombok.Data;

@Data
@AllArgsConstructor
public class JoinRecommendation {
    private String source;
    private String target;
    private String joinType;
    private String on;
}
```

---

### 🔧 2. `JoinGraphUtil.java` (업데이트된 부분 포함)

```java
public class JoinGraphUtil {

    public static List<JoinRecommendation> getJoinRecommendations(List<String> keys, DdeMetadataProperties dde) {
        Set<String> visited = new HashSet<>();
        Set<String> result = new LinkedHashSet<>();
        List<JoinRecommendation> recommendations = new ArrayList<>();

        for (String key : keys) {
            dfsWithRecommendation(key, dde, visited, result, recommendations);
        }
        return recommendations;
    }

    private static void dfsWithRecommendation(
            String currentKey,
            DdeMetadataProperties dde,
            Set<String> visited,
            Set<String> result,
            List<JoinRecommendation> recommendations
    ) {
        if (visited.contains(currentKey)) return;
        visited.add(currentKey);

        FieldConfig field = getFieldConfig(currentKey, dde);
        if (field != null && field.getJoins() != null) {
            if (field.getJoins().getMeasure() != null) {
                for (JoinConfig join : field.getJoins().getMeasure()) {
                    recommendations.add(new JoinRecommendation(
                            currentKey, join.getTarget(), join.getJoinType(), join.getOn()));
                    dfsWithRecommendation(join.getTarget(), dde, visited, result, recommendations);
                }
            }
            if (field.getJoins().getDimension() != null) {
                for (JoinConfig join : field.getJoins().getDimension()) {
                    recommendations.add(new JoinRecommendation(
                            currentKey, join.getTarget(), join.getJoinType(), join.getOn()));
                    dfsWithRecommendation(join.getTarget(), dde, visited, result, recommendations);
                }
            }
            if (field.getJoins().getFilter() != null) {
                for (JoinConfig join : field.getJoins().getFilter()) {
                    recommendations.add(new JoinRecommendation(
                            currentKey, join.getTarget(), join.getJoinType(), join.getOn()));
                    dfsWithRecommendation(join.getTarget(), dde, visited, result, recommendations);
                }
            }
        }
    }

    // toKey, getFieldConfig 등 기존 메서드는 그대로 유지
}
```

---

### 🔧 3. `JoinTargets.java` (구조 변경)

```java
import lombok.Data;
import java.util.List;

@Data
public class JoinTargets {
    private List<JoinConfig> measure;
    private List<JoinConfig> dimension;
    private List<JoinConfig> filter;
}
```

---

### 🔧 4. `JoinConfig.java` (새 클래스)

```java
import lombok.Data;

@Data
public class JoinConfig {
    private String target;
    private String joinType;
    private String on;
}
```

---

이제 YAML에서 설정한 `target`, `joinType`, `on`이 정상적으로 바인딩되고, `getJoinRecommendations(...)` 호출 시에도 `Map<String, Object>` 대신 구조화된 `JoinRecommendation` 객체 리스트가 반환됩니다.

빌드가 되지 않던 원인은 `JoinTargets`의 리스트 항목이 `String`이 아닌 복합 객체여야 했기 때문이며, 지금 구조에서는 YAML 파싱도 `JoinConfig` 객체로 잘 매핑됩니다.

필요하면 테스트 코드도 같이 만들어드릴게요.

-----------------

알겠어요!
기존에 빠진 `getJoinRecommendations` 메서드와 `getCategoryAndFields` 메서드를 포함하고, `getFieldMetadata` 결과도 DTO로 처리하도록 리팩토링한 완전한 예시를 만들어드릴게요.

---

## 1. DTO 클래스들 추가

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;
import java.util.Map;

// JoinRecommendationResponse DTO
@Data
@NoArgsConstructor
@AllArgsConstructor
public class JoinRecommendationResponse {
    private List<String> target;
    private String joinType;
    private List<String> on;
}

// Field DTO (기존 FieldConfig에 UI나 API용 필드명 추가)
@Data
@AllArgsConstructor
public class Field {
    private String field;  // 필드 키
    private String label;
    private Map<String, String> metric;
    private Map<String, String> operator;
    private JoinTargets joins;
}

// CategoryWithFields DTO
@Data
@AllArgsConstructor
public class CategoryWithFields {
    private String category;
    private String label;
    private List<Field> fields;
}

// FieldMetadataResponse DTO (getFieldMetadata 결과용)
@Data
@AllArgsConstructor
@NoArgsConstructor
public class FieldMetadataResponse {
    private String label;
    private Map<String, String> operator;
}
```

---

## 2. JoinGraphUtil.java 수정 (메서드 포함)

```java
public class JoinGraphUtil {

    public static Map<String, Set<String>> buildJoinGraph(DdeMetadataProperties dde) {
        Map<String, Set<String>> graph = new HashMap<>();
        buildFromCategory("dimension", dde.getDimension(), graph);
        buildFromCategory("filter", dde.getFilter(), graph);
        buildFromCategory("measure", dde.getMeasure(), graph);
        return graph;
    }

    private static void buildFromCategory(String group, Map<String, CategoryConfig> map, Map<String, Set<String>> graph) {
        for (Map.Entry<String, CategoryConfig> categoryEntry : map.entrySet()) {
            String category = categoryEntry.getKey();
            Map<String, FieldConfig> fields = categoryEntry.getValue().getFields();
            if (fields == null) continue;

            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                String field = fieldEntry.getKey();
                String sourceKey = toKey(group, category, field);

                FieldConfig fieldConfig = fieldEntry.getValue();
                if (fieldConfig.getJoins() != null) {
                    addJoinsToGraph(sourceKey, fieldConfig.getJoins(), graph);
                }
            }
        }
    }

    private static void addJoinsToGraph(String sourceKey, JoinTargets joins, Map<String, Set<String>> graph) {
        addJoinList(sourceKey, joins.getMeasure(), graph);
        addJoinList(sourceKey, joins.getDimension(), graph);
        addJoinList(sourceKey, joins.getFilter(), graph);
    }

    private static void addJoinList(String sourceKey, List<String> targets, Map<String, Set<String>> graph) {
        if (targets == null) return;
        for (String target : targets) {
            graph.computeIfAbsent(sourceKey, k -> new HashSet<>()).add(target);
        }
    }

    public static String toKey(String group, String category, String field) {
        return group + ":" + category + ":" + field;
    }

    public static FieldConfig getFieldConfig(String group, String category, String field) {
        Map<String, CategoryConfig> groupMap = getGroupMap(group);
        if (groupMap == null) return null;
        CategoryConfig categoryConfig = groupMap.get(category);
        if (categoryConfig == null) return null;
        return categoryConfig.getFields() != null ? categoryConfig.getFields().get(field) : null;
    }

    public static Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> DdeMetadataHolder.getInstance().getDdeMetadataProperties().getDimension();
            case "measure" -> DdeMetadataHolder.getInstance().getDdeMetadataProperties().getMeasure();
            case "filter" -> DdeMetadataHolder.getInstance().getDdeMetadataProperties().getFilter();
            default -> null;
        };
    }

    /** getFieldMetadata 메서드 DTO 리턴용으로 변경 */
    public static FieldMetadataResponse getFieldMetadata(String group, String category, String field) {
        FieldConfig field = getFieldConfig(group, category, field);
        if (field == null) return new FieldMetadataResponse();
        return new FieldMetadataResponse(field.getLabel(), field.getOperator());
    }

    /**
     * 누락된 getJoinRecommendations 메서드 추가
     * 필드의 join 추천 정보를 DTO로 반환
     */
    public JoinRecommendationResponse getJoinRecommendations(String group, String category, String field) {
        FieldConfig fieldConfig = getFieldConfig(group, category, field);
        if (fieldConfig == null || fieldConfig.getJoins() == null) {
            return new JoinRecommendationResponse();
        }

        JoinTargets joins = fieldConfig.getJoins();
        // 예시로 on 조건 변환 (convertOnCondition은 구현 필요)
        List<String> convertedOn = joins.getOn() != null ?
                joins.getOn().stream()
                        .map(this::convertOnCondition)
                        .toList() : List.of();

        return new JoinRecommendationResponse(joins.getTarget(), joins.getJoinType(), convertedOn);
    }

    /** 
     * 예시용 on 조건 변환 메서드 (실제 구현 필요)
     */
    private String convertOnCondition(String onCondition) {
        // 여기서 변환 작업 진행
        return onCondition;
    }

    /**
     * getCategoryAndFields 메서드도 DTO로 반환하도록 수정
     */
    public List<CategoryWithFields> getCategoryAndFields(String group) {
        Map<String, CategoryConfig> groupMap = getGroupMap(group);
        if (groupMap == null) return Collections.emptyList();

        return groupMap.entrySet().stream()
                .map(e -> {
                    String category = e.getKey();
                    CategoryConfig catConfig = e.getValue();
                    List<Field> fields = catConfig.getFields() != null ?
                            catConfig.getFields().entrySet().stream()
                                    .map(entry -> {
                                        FieldConfig f = entry.getValue();
                                        return new Field(
                                                entry.getKey(),
                                                f.getLabel(),
                                                f.getMetric(),
                                                f.getOperator(),
                                                f.getJoins()
                                        );
                                    }).toList()
                            : Collections.emptyList();
                    return new CategoryWithFields(category, catConfig.getLabel(), fields);
                }).toList();
    }
}
```

---

### 주의:

* `DdeMetadataHolder.getInstance().getDdeMetadataProperties()`는 싱글톤 또는 DI를 통해 `DdeMetadataProperties`를 얻는 예시로, 실제 환경에 맞게 바꾸세요.
* `convertOnCondition` 내부 로직은 필요에 따라 구현해야 합니다.
* `joins.getOn()`과 `joins.getTarget()`, `joins.getJoinType()` 필드는 기존에 DTO `JoinTargets`에 없던 필드라면 추가해야 합니다. (아래 참고)

---

## 3. JoinTargets.java 수정 (필요한 필드 추가)

```java
import lombok.Data;

import java.util.List;

@Data
public class JoinTargets {
    private List<String> measure;
    private List<String> dimension;
    private List<String> filter;

    // 추가 필드 예시
    private List<String> on;       // 조인 조건 리스트
    private List<String> target;   // 조인 대상
    private String joinType;       // 조인 타입 (예: INNER, LEFT 등)
}
```

---

필요한 모든 기능 포함해서 완전한 코드 형태로 정리해 드렸습니다.
필요하면 더 구체적으로 도와드릴 수 있어요!
이대로 적용해 보시고 궁금한 점 알려주세요.



네, 주신 기존 그래프 유틸리티(`JoinGraphUtil`) 코드와 메타데이터 설정(`DdeMetadataProperties` 및 관련 DTO들)을 포함한 전체 구조를 다시 정리해서,
**Lombok 생성자 포함, full path 기준 조인 키 처리, 그래프 빌드 로직 모두 완전 통합한 파일들**로 깔끔하게 만들어 드리겠습니다.

---

# 1. DdeMetadataProperties.java (메타데이터 + 기능 포함)

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.*;
import java.util.stream.Collectors;

@Data
@NoArgsConstructor
@AllArgsConstructor
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {

    private Map<String, CategoryConfig> dimension;
    private Map<String, CategoryConfig> filter;
    private Map<String, CategoryConfig> measure;

    /**
     * group별 Map 가져오기
     */
    public Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            case "filter" -> filter;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    /**
     * 카테고리 key 조회 by label
     */
    public String getCategoryKeyByLabel(String group, String label) {
        return getGroupMap(group).entrySet().stream()
                .filter(e -> label.equalsIgnoreCase(e.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst().orElse(null);
    }

    /**
     * 필드명 조회 by label
     */
    public String getFieldName(String group, String category, String label) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        return config.getFields().entrySet().stream()
                .filter(e -> label.equalsIgnoreCase(e.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst().orElse(null);
    }

    /**
     * 필드 라벨 조회
     */
    public String getFieldLabel(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        FieldConfig field = config.getFields().get(fieldName);
        return field != null ? field.getLabel() : null;
    }

    /**
     * 필드 라벨 Map 조회
     */
    public Map<String, String> getFieldLabels(String group, String category) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return Map.of();
        return config.getFields().entrySet().stream()
                .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().getLabel()));
    }

    /**
     * 필드 메트릭 조회
     */
    public Map<String, String> getFieldMetrics(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null) return Map.of();
        FieldConfig field = config.getFields().get(fieldName);
        return field != null && field.getMetric() != null ? field.getMetric() : Map.of();
    }

    /**
     * FieldConfig 조회
     */
    public FieldConfig getFieldConfig(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        return config != null ? config.getFields().get(fieldName) : null;
    }

    /**
     * 그룹의 카테고리 라벨 리스트
     */
    public List<String> getCategoryLabels(String group) {
        return getGroupMap(group).values().stream()
                .map(CategoryConfig::getLabel)
                .toList();
    }

    /**
     * label로 그룹+카테고리 key 찾기 (전체 그룹 조회)
     */
    public String findCategoryByLabelAcrossGroups(String label) {
        for (String group : List.of("dimension", "measure", "filter")) {
            String match = getCategoryKeyByLabel(group, label);
            if (match != null) return group + ":" + match;
        }
        return null;
    }

    /**
     * label로 그룹+카테고리+필드 key 찾기
     */
    public String findFieldNameAcrossGroups(String categoryLabel, String fieldLabel) {
        for (String group : List.of("dimension", "measure", "filter")) {
            Map<String, CategoryConfig> groupMap = getGroupMap(group);
            for (Map.Entry<String, CategoryConfig> entry : groupMap.entrySet()) {
                if (categoryLabel.equalsIgnoreCase(entry.getValue().getLabel())) {
                    String fieldKey = entry.getValue().getFields().entrySet().stream()
                            .filter(f -> fieldLabel.equalsIgnoreCase(f.getValue().getLabel()))
                            .map(Map.Entry::getKey)
                            .findFirst().orElse(null);
                    if (fieldKey != null) return group + ":" + entry.getKey() + ":" + fieldKey;
                }
            }
        }
        return null;
    }
}
```

---

# 2. CategoryConfig.java

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields;
}
```

---

# 3. FieldConfig.java

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class FieldConfig {
    private String label;
    private Map<String, String> metric;
    private Map<String, String> operator;
    private JoinTargets joins;
}
```

---

# 4. JoinTargets.java

```java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class JoinTargets {
    private List<String> measure;
    private List<String> dimension;
    private List<String> filter;
}
```

---

# 5. JoinGraphUtil.java (그래프 빌드 및 메타데이터 접근 유틸)

```java
import java.util.*;

public class JoinGraphUtil {

    /**
     * 전체 그룹(dimension, filter, measure)으로부터 조인 그래프 생성
     * key: group:category:field
     * value: set of 연결된 조인 대상 full key (string)
     */
    public static Map<String, Set<String>> buildJoinGraph(DdeMetadataProperties dde) {
        Map<String, Set<String>> graph = new HashMap<>();
        buildFromCategory("dimension", dde.getDimension(), graph);
        buildFromCategory("filter", dde.getFilter(), graph);
        buildFromCategory("measure", dde.getMeasure(), graph);
        return graph;
    }

    private static void buildFromCategory(String group, Map<String, CategoryConfig> map, Map<String, Set<String>> graph) {
        for (Map.Entry<String, CategoryConfig> categoryEntry : map.entrySet()) {
            String category = categoryEntry.getKey();
            Map<String, FieldConfig> fields = categoryEntry.getValue().getFields();
            if (fields == null) continue;

            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                String field = fieldEntry.getKey();
                String sourceKey = toKey(group, category, field);

                FieldConfig fieldConfig = fieldEntry.getValue();
                if (fieldConfig.getJoins() != null) {
                    addJoinsToGraph(sourceKey, fieldConfig.getJoins(), graph);
                }
            }
        }
    }

    private static void addJoinsToGraph(String sourceKey, JoinTargets joins, Map<String, Set<String>> graph) {
        addJoinList(sourceKey, joins.getMeasure(), graph);
        addJoinList(sourceKey, joins.getDimension(), graph);
        addJoinList(sourceKey, joins.getFilter(), graph);
    }

    private static void addJoinList(String sourceKey, List<String> targets, Map<String, Set<String>> graph) {
        if (targets == null) return;
        for (String target : targets) {
            graph.computeIfAbsent(sourceKey, k -> new HashSet<>()).add(target);
        }
    }

    /**
     * group:category:field 형식 키 생성
     */
    public static String toKey(String group, String category, String field) {
        return group + ":" + category + ":" + field;
    }

    /**
     * fullKey 기준 FieldConfig 조회
     */
    public static FieldConfig getFieldConfig(String fullKey, DdeMetadataProperties dde) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        String group = parts[0];
        String category = parts[1];
        String field = parts[2];

        Map<String, CategoryConfig> groupMap = getGroupMap(group, dde);
        if (groupMap == null) return null;
        CategoryConfig categoryConfig = groupMap.get(category);
        if (categoryConfig == null) return null;
        return categoryConfig.getFields().get(field);
    }

    /**
     * fullKey 기준 메타데이터 간단 조회 (label, operator)
     */
    public static Map<String, Object> getFieldMetadata(String fullKey, DdeMetadataProperties dde) {
        FieldConfig field = getFieldConfig(fullKey, dde);
        if (field == null) return Map.of();

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("label", field.getLabel());
        result.put("operator", field.getOperator());
        return result;
    }

   
```


```yaml
dde:
  measure:
    mv_app_usage:
      label: "App Usage Daily"
      fields:
        totalRunTime:
          label: "Total Run Time"
          metric:
            sum: "Sum (EA)"
            avg: "Avg (%)"
          operator:
            lt: "<"
            eq: "="
            gt: ">"
          joins:
            measure:
              - target: "measure:mv_app_usage:foreground_usage"
                joinType: "left"
                on: "app_id = app_id"
            dimension:
              - target: "dimension:device:device_id"
                joinType: "inner"
                on: "device_id = device_id"
              - target: "dimension:app:app_name"
                joinType: "left"
                on: "app_name = app_name"

```
```java
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.*;
import java.util.stream.Collectors;

@Data
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {
    private Map<String, CategoryConfig> dimension;
    private Map<String, CategoryConfig> filter;
    private Map<String, CategoryConfig> measure;

    public Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            case "filter" -> filter;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    public String getCategoryKeyByLabel(String group, String label) {
        return getGroupMap(group).entrySet().stream()
                .filter(e -> label.equalsIgnoreCase(e.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst().orElse(null);
    }

    public String getFieldName(String group, String category, String label) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        return config.getFields().entrySet().stream()
                .filter(e -> label.equalsIgnoreCase(e.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst().orElse(null);
    }

    public String getFieldLabel(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        FieldConfig field = config.getFields().get(fieldName);
        return field != null ? field.getLabel() : null;
    }

    public Map<String, String> getFieldLabels(String group, String category) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return Map.of();
        return config.getFields().entrySet().stream()
                .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().getLabel()));
    }

    public Map<String, String> getFieldMetrics(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null) return Map.of();
        FieldConfig field = config.getFields().get(fieldName);
        return field != null && field.getMetric() != null ? field.getMetric() : Map.of();
    }

    public FieldConfig getFieldConfig(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        return config != null ? config.getFields().get(fieldName) : null;
    }

    public List<String> getCategoryLabels(String group) {
        return getGroupMap(group).values().stream()
                .map(CategoryConfig::getLabel)
                .toList();
    }

    public String findCategoryByLabelAcrossGroups(String label) {
        for (String group : List.of("dimension", "measure", "filter")) {
            String match = getCategoryKeyByLabel(group, label);
            if (match != null) return group + ":" + match;
        }
        return null;
    }

    public String findFieldNameAcrossGroups(String categoryLabel, String fieldLabel) {
        for (String group : List.of("dimension", "measure", "filter")) {
            Map<String, CategoryConfig> groupMap = getGroupMap(group);
            for (Map.Entry<String, CategoryConfig> entry : groupMap.entrySet()) {
                if (categoryLabel.equalsIgnoreCase(entry.getValue().getLabel())) {
                    String fieldKey = entry.getValue().getFields().entrySet().stream()
                            .filter(f -> fieldLabel.equalsIgnoreCase(f.getValue().getLabel()))
                            .map(Map.Entry::getKey)
                            .findFirst().orElse(null);
                    if (fieldKey != null) return group + ":" + entry.getKey() + ":" + fieldKey;
                }
            }
        }
        return null;
    }

    public List<CategoryFieldsDto> getCategoryAndFields(String group) {
        Map<String, CategoryConfig> groupMap = getGroupMap(group);
        List<CategoryFieldsDto> result = new ArrayList<>();
        for (Map.Entry<String, CategoryConfig> categoryEntry : groupMap.entrySet()) {
            String category = categoryEntry.getKey();
            CategoryConfig config = categoryEntry.getValue();
            Map<String, FieldConfig> fields = config.getFields();
            if (fields == null) continue;

            List<FieldDto> fieldList = new ArrayList<>();
            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                FieldConfig fc = fieldEntry.getValue();
                fieldList.add(new FieldDto(fieldEntry.getKey(), fc.getLabel(), fc.getMetric(), fc.getOperator(), fc.getJoins()));
            }
            result.add(new CategoryFieldsDto(category, config.getLabel(), fieldList));
        }
        return result;
    }

    public JoinRecommendationResponse getJoinRecommendations(String group, String category, String fieldName) {
        FieldConfig config = getFieldConfig(group, category, fieldName);
        if (config == null || config.getJoins() == null) return new JoinRecommendationResponse();
        JoinTargets joins = config.getJoins();
        return new JoinRecommendationResponse(joins.getTarget(), joins.getJoinType(), joins.getOn());
    }
}

@Data
class JoinRecommendationResponse {
    private List<String> target;
    private List<String> joinType;
    private List<String> on;

    public JoinRecommendationResponse() {
        this.target = new ArrayList<>();
        this.joinType = new ArrayList<>();
        this.on = new ArrayList<>();
    }

    public JoinRecommendationResponse(List<String> target, List<String> joinType, List<String> on) {
        this.target = target != null ? target : new ArrayList<>();
        this.joinType = joinType != null ? joinType : new ArrayList<>();
        this.on = on != null ? on : new ArrayList<>();
    }
}

@Data
class CategoryFieldsDto {
    private String category;
    private String label;
    private List<FieldDto> fields;

    public CategoryFieldsDto(String category, String label, List<FieldDto> fields) {
        this.category = category;
        this.label = label;
        this.fields = fields;
    }
}

@Data
class FieldDto {
    private String field;
    private String label;
    private Map<String, String> metric;
    private List<String> operator;
    private JoinTargets joins;

    public FieldDto(String field, String label, Map<String, String> metric, List<String> operator, JoinTargets joins) {
        this.field = field;
        this.label = label;
        this.metric = metric;
        this.operator = operator;
        this.joins = joins;
    }
}


```
--------------------
``` java

// JoinTarget.java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class JoinTarget {
    private String target;    // 예: "dimension:device:device_id"
    private String joinType;  // 예: "LEFT", "OUTER", "NONE"
    private String on;        // 예: "app_usage:device_id = device:dev_id"
}

// JoinRelation.java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.List;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class JoinRelation {
    private List<JoinTarget> measure;
    private List<JoinTarget> dimension;
    private List<JoinTarget> filter;
}

// FieldConfig.java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

@Data
@NoArgsConstructor
@AllArgsConstructor
public class FieldConfig {
    private String label;
    private Map<String, String> operator;
    private JoinRelation joins;
}

// JoinFieldInfo.java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class JoinFieldInfo {
    private String group;
    private String category;
    private String field;
    private String label;
    private Map<String, String> operator;

    private String joinType; // 추가
    private String on;       // 추가
}

// JoinRecommendationResponse.java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.ArrayList;
import java.util.List;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class JoinRecommendationResponse {
    private List<JoinFieldInfo> measure = new ArrayList<>();
    private List<JoinFieldInfo> dimension = new ArrayList<>();
    private List<JoinFieldInfo> filter = new ArrayList<>();
}

// JoinUtil.java
import java.util.*;
import java.util.stream.Collectors;

public class JoinUtil {

    public static JoinRecommendationResponse getJoinRecommendations(
            String group, String category, String field,
            Map<String, List<DdeMetadataProperties.JoinEdge>> graph,
            DdeMetadataProperties dde
    ) {
        String fromKey = group + ":" + category + ":" + field;
        List<DdeMetadataProperties.JoinEdge> connections = graph.getOrDefault(fromKey, List.of());

        JoinRecommendationResponse response = new JoinRecommendationResponse();

        for (DdeMetadataProperties.JoinEdge edge : connections) {
            JoinFieldInfo info = createJoinFieldInfo(edge.getTargetKey(), dde, false);
            if (info == null) continue;

            switch (info.getGroup()) {
                case "measure" -> response.getMeasure().add(info);
                case "dimension" -> response.getDimension().add(info);
            }
        }

        FieldConfig source = dde.getFieldConfig(group, category, field);
        if (source != null && source.getJoins() != null && source.getJoins().getFilter() != null) {
            for (JoinTarget jt : source.getJoins().getFilter()) {
                JoinFieldInfo info = createJoinFieldInfo(jt.getTarget(), dde, true);
                if (info != null) {
                    info.setJoinType(jt.getJoinType());
                    info.setOn(jt.getOn());
                    response.getFilter().add(info);
                }
            }
        }

        return response;
    }

    public static List<JoinFieldInfo> getAllFieldsByGroup(String group, DdeMetadataProperties dde) {
        Map<String, CategoryConfig> groupMap = dde.getGroupMap(group);
        if (groupMap == null) return List.of();

        return groupMap.entrySet().stream()
            .flatMap(entry -> entry.getValue().getFields().entrySet().stream()
                .map(field -> new JoinFieldInfo(
                    group,
                    entry.getKey(),
                    field.getKey(),
                    field.getValue().getLabel(),
                    field.getValue().getOperator(),
                    null,
                    null
                ))
            ).collect(Collectors.toList());
    }

    public static List<JoinFieldInfo> findFieldsByLabel(String label, DdeMetadataProperties dde) {
        List<JoinFieldInfo> results = new ArrayList<>();
        for (String group : List.of("dimension", "measure", "filter")) {
            Map<String, CategoryConfig> groupMap = dde.getGroupMap(group);
            for (Map.Entry<String, CategoryConfig> entry : groupMap.entrySet()) {
                for (Map.Entry<String, FieldConfig> field : entry.getValue().getFields().entrySet()) {
                    if (label.equalsIgnoreCase(field.getValue().getLabel())) {
                        results.add(new JoinFieldInfo(
                            group,
                            entry.getKey(),
                            field.getKey(),
                            field.getValue().getLabel(),
                            field.getValue().getOperator(),
                            null,
                            null
                        ));
                    }
                }
            }
        }
        return results;
    }

    public static JoinFieldInfo getFieldInfo(String group, String category, String field, DdeMetadataProperties dde) {
        FieldConfig fc = dde.getFieldConfig(group, category, field);
        if (fc == null) return null;
        return new JoinFieldInfo(group, category, field, fc.getLabel(), fc.getOperator(), null, null);
    }

    public static JoinFieldInfo getFieldInfo(String fullKey, DdeMetadataProperties dde, boolean includeOperator) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        return createJoinFieldInfo(fullKey, dde, includeOperator);
    }

    private static JoinFieldInfo createJoinFieldInfo(String fullKey, DdeMetadataProperties dde, boolean includeOperator) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        String g = parts[0], cat = parts[1], fld = parts[2];
        FieldConfig fc = dde.getFieldConfig(g, cat, fld);
        if (fc == null) return null;
        return new JoinFieldInfo(g, cat, fld, fc.getLabel(), includeOperator ? fc.getOperator() : null, null, null);
    }
}
```


```yaml
joins:
  measure:
    - target: "measure:app_usage:foreground_usage"
      joinType: NONE
      on: "app_usage.device_id = device.dev_id"
  dimension:
    - target: "dimension:device:device_id"
      joinType: LEFT
      on: "app_usage.device_id = device.dev_id"
  filter:
    - target: "event:app_event:event_type"
      joinType: OUTER
      on: "app_usage.device_id = app_event.device_id"


```

--------------------

```java
import lombok.Data;
import lombok.AllArgsConstructor;
import lombok.NoArgsConstructor;

import java.util.*;
import java.util.stream.Collectors;

@Data
public class JoinAll {

    // ===== JoinFieldInfo =====
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class JoinFieldInfo {
        private String group;
        private String category;
        private String field;
        private String label;
        private Map<String, String> operator; // nullable
    }

    // ===== JoinRecommendationResponse =====
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class JoinRecommendationResponse {
        private List<JoinFieldInfo> measure = new ArrayList<>();
        private List<JoinFieldInfo> dimension = new ArrayList<>();
        private List<JoinFieldInfo> filter = new ArrayList<>();
    }

    // ===== JoinUtil =====
    public static class JoinUtil {

        public static JoinRecommendationResponse getJoinRecommendations(
                String group, String category, String field,
                Map<String, List<DdeMetadataProperties.JoinEdge>> graph,
                DdeMetadataProperties dde
        ) {
            String fromKey = group + ":" + category + ":" + field;
            List<DdeMetadataProperties.JoinEdge> connections = graph.getOrDefault(fromKey, List.of());

            JoinRecommendationResponse response = new JoinRecommendationResponse();

            for (DdeMetadataProperties.JoinEdge edge : connections) {
                JoinFieldInfo info = createJoinFieldInfo(edge.getTargetKey(), dde, false);
                if (info == null) continue;

                switch (info.getGroup()) {
                    case "measure" -> response.getMeasure().add(info);
                    case "dimension" -> response.getDimension().add(info);
                    case "filter" -> response.getFilter().add(info);
                }
            }

            FieldConfig source = dde.getFieldConfig(group, category, field);
            if (source != null && source.getJoins() != null && source.getJoins().getFilter() != null) {
                for (JoinTarget jt : source.getJoins().getFilter()) {
                    JoinFieldInfo info = createJoinFieldInfo(jt.getTarget(), dde, true);
                    if (info != null) {
                        response.getFilter().add(info);
                    }
                }
            }

            return response;
        }

        public static List<JoinFieldInfo> getAllFieldsByGroup(String group, DdeMetadataProperties dde) {
            Map<String, CategoryConfig> groupMap = dde.getGroupMap(group);
            if (groupMap == null) return List.of();

            return groupMap.entrySet().stream()
                    .flatMap(entry -> entry.getValue().getFields().entrySet().stream()
                            .map(field -> new JoinFieldInfo(
                                    group,
                                    entry.getKey(),
                                    field.getKey(),
                                    field.getValue().getLabel(),
                                    field.getValue().getOperator()
                            ))
                    ).collect(Collectors.toList());
        }

        public static List<JoinFieldInfo> findFieldsByLabel(String label, DdeMetadataProperties dde) {
            List<JoinFieldInfo> results = new ArrayList<>();
            for (String group : List.of("dimension", "measure", "filter")) {
                Map<String, CategoryConfig> groupMap = dde.getGroupMap(group);
                if (groupMap == null) continue;
                for (Map.Entry<String, CategoryConfig> entry : groupMap.entrySet()) {
                    for (Map.Entry<String, FieldConfig> field : entry.getValue().getFields().entrySet()) {
                        if (label.equalsIgnoreCase(field.getValue().getLabel())) {
                            results.add(new JoinFieldInfo(
                                    group,
                                    entry.getKey(),
                                    field.getKey(),
                                    field.getValue().getLabel(),
                                    field.getValue().getOperator()
                            ));
                        }
                    }
                }
            }
            return results;
        }

        public static JoinFieldInfo getFieldInfo(String group, String category, String field, DdeMetadataProperties dde) {
            FieldConfig fc = dde.getFieldConfig(group, category, field);
            if (fc == null) return null;
            return new JoinFieldInfo(group, category, field, fc.getLabel(), fc.getOperator());
        }

        public static JoinFieldInfo getFieldInfo(String fullKey, DdeMetadataProperties dde, boolean includeOperator) {
            String[] parts = fullKey.split(":");
            if (parts.length != 3) return null;
            return createJoinFieldInfo(fullKey, dde, includeOperator);
        }

        private static JoinFieldInfo createJoinFieldInfo(String fullKey, DdeMetadataProperties dde, boolean includeOperator) {
            String[] parts = fullKey.split(":");
            if (parts.length != 3) return null;
            String g = parts[0], cat = parts[1], fld = parts[2];
            FieldConfig fc = dde.getFieldConfig(g, cat, fld);
            if (fc == null) return null;
            return new JoinFieldInfo(g, cat, fld, fc.getLabel(), includeOperator ? fc.getOperator() : null);
        }
    }

    // ===== DdeMetadataProperties =====
    @Data
    public static class DdeMetadataProperties {
        private Map<String, Map<String, CategoryConfig>> groups;

        public Map<String, CategoryConfig> getGroupMap(String group) {
            return groups.get(group);
        }

        public FieldConfig getFieldConfig(String group, String category, String field) {
            if (groups == null) return null;
            Map<String, CategoryConfig> categoryMap = groups.get(group);
            if (categoryMap == null) return null;
            CategoryConfig catConfig = categoryMap.get(category);
            if (catConfig == null) return null;
            return catConfig.getFields().get(field);
        }

        @Data
        public static class JoinEdge {
            private String sourceKey;
            private String targetKey;

            public JoinEdge() {}

            public JoinEdge(String sourceKey, String targetKey) {
                this.sourceKey = sourceKey;
                this.targetKey = targetKey;
            }
        }
    }

    // ===== CategoryConfig =====
    @Data
    public static class CategoryConfig {
        private Map<String, FieldConfig> fields;
    }

    // ===== FieldConfig =====
    @Data
    public static class FieldConfig {
        private String label;
        private Map<String, String> operator;
        private JoinTargets joins;
    }

    // ===== JoinTargets =====
    @Data
    public static class JoinTargets {
        private List<JoinTarget> filter;
        private List<JoinTarget> measure;
        private List<JoinTarget> dimension;
    }

    // ===== JoinTarget =====
    @Data
    @AllArgsConstructor
    @NoArgsConstructor
    public static class JoinTarget {
        private String target;
    }
}


```

--------------

``` java

// JoinQueryBuilder.java
import jakarta.persistence.criteria.*;
import java.util.Map;

public class JoinQueryBuilder {

    public static <T> void applyJoins(Root<T> root, CriteriaQuery<?> query, CriteriaBuilder cb,
                                      String sourceKey,
                                      Map<String, List<DdeMetadataProperties.JoinEdge>> graph,
                                      DdeMetadataProperties dde,
                                      Map<String, Join<?, ?>> joinsOut) {

        if (!graph.containsKey(sourceKey)) return;

        for (DdeMetadataProperties.JoinEdge edge : graph.get(sourceKey)) {
            String[] parts = edge.getTargetKey().split(":");
            if (parts.length != 3) continue;
            String group = parts[0], category = parts[1], field = parts[2];

            String joinAlias = group + "_" + category + "_" + field;
            Join<?, ?> join;

            switch (edge.getJoinType().toUpperCase()) {
                case "LEFT" -> join = root.join(category, JoinType.LEFT);
                case "OUTER" -> join = root.join(category, JoinType.RIGHT);
                case "INNER" -> join = root.join(category, JoinType.INNER);
                default -> continue;
            }

            joinsOut.put(joinAlias, join);
        }
    }
}

// MetadataController.java
import lombok.RequiredArgsConstructor;
import org.springframework.web.bind.annotation.*;

import java.util.List;

@RestController
@RequestMapping("/api/metadata")
@RequiredArgsConstructor
public class MetadataController {

    private final DdeMetadataProperties dde;

    @GetMapping("/{group}/fields")
    public List<JoinFieldInfo> getFields(@PathVariable String group) {
        return JoinUtil.getAllFieldsByGroup(group, dde);
    }

    @GetMapping("/field")
    public JoinFieldInfo getField(@RequestParam String group,
                                  @RequestParam String category,
                                  @RequestParam String field) {
        return JoinUtil.getFieldInfo(group, category, field, dde);
    }

    @GetMapping("/recommend")
    public JoinRecommendationResponse getRecommendation(@RequestParam String group,
                                                        @RequestParam String category,
                                                        @RequestParam String field) {
        var graph = DdeMetadataProperties.buildJoinGraph(dde);
        return JoinUtil.getJoinRecommendations(group, category, field, graph, dde);
    }
}

// JoinUtilTest.java (JUnit)
import org.junit.jupiter.api.Test;
import static org.junit.jupiter.api.Assertions.*;

import java.util.List;

public class JoinUtilTest {

    @Test
    void testFindFieldByLabel() {
        DdeMetadataProperties dde = TestUtil.loadSampleMetadata();
        List<JoinFieldInfo> result = JoinUtil.findFieldsByLabel("Total Run Time", dde);
        assertFalse(result.isEmpty());
        assertEquals("totalRunTime", result.get(0).getField());
    }

    @Test
    void testGetJoinRecommendations() {
        DdeMetadataProperties dde = TestUtil.loadSampleMetadata();
        var graph = DdeMetadataProperties.buildJoinGraph(dde);
        var result = JoinUtil.getJoinRecommendations("measure", "app_usage", "totalRunTime", graph, dde);
        assertNotNull(result);
        assertFalse(result.getDimension().isEmpty());
    }
}

// TestUtil.java
public class TestUtil {
    public static DdeMetadataProperties loadSampleMetadata() {
        // In-memory setup or parse from YAML (e.g., SnakeYAML if loading real YAML)
        // Simplified stub for now
        DdeMetadataProperties dde = new DdeMetadataProperties();
        // ... populate dde.measure, dimension, filter
        return dde;
    }
}


// JoinFieldInfo.java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.Map;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class JoinFieldInfo {
    private String group;
    private String category;
    private String field;
    private String label;
    private Map<String, String> operator; // nullable
}

// JoinRecommendationResponse.java
import lombok.AllArgsConstructor;
import lombok.Data;
import lombok.NoArgsConstructor;

import java.util.ArrayList;
import java.util.List;

@Data
@AllArgsConstructor
@NoArgsConstructor
public class JoinRecommendationResponse {
    private List<JoinFieldInfo> measure = new ArrayList<>();
    private List<JoinFieldInfo> dimension = new ArrayList<>();
    private List<JoinFieldInfo> filter = new ArrayList<>();
}

// JoinUtil.java
import java.util.*;
import java.util.stream.Collectors;

public class JoinUtil {

    public static JoinRecommendationResponse getJoinRecommendations(
            String group, String category, String field,
            Map<String, List<DdeMetadataProperties.JoinEdge>> graph,
            DdeMetadataProperties dde
    ) {
        String fromKey = group + ":" + category + ":" + field;
        List<DdeMetadataProperties.JoinEdge> connections = graph.getOrDefault(fromKey, List.of());

        JoinRecommendationResponse response = new JoinRecommendationResponse();

        for (DdeMetadataProperties.JoinEdge edge : connections) {
            JoinFieldInfo info = createJoinFieldInfo(edge.getTargetKey(), dde, false);
            if (info == null) continue;

            switch (info.getGroup()) {
                case "measure" -> response.getMeasure().add(info);
                case "dimension" -> response.getDimension().add(info);
            }
        }

        // filter section by intent
        FieldConfig source = dde.getFieldConfig(group, category, field);
        if (source != null && source.getJoins() != null && source.getJoins().getFilter() != null) {
            for (JoinTarget jt : source.getJoins().getFilter()) {
                JoinFieldInfo info = createJoinFieldInfo(jt.getTarget(), dde, true);
                if (info != null) {
                    response.getFilter().add(info);
                }
            }
        }

        return response;
    }

    public static List<JoinFieldInfo> getAllFieldsByGroup(String group, DdeMetadataProperties dde) {
        Map<String, CategoryConfig> groupMap = dde.getGroupMap(group);
        if (groupMap == null) return List.of();

        return groupMap.entrySet().stream()
            .flatMap(entry -> entry.getValue().getFields().entrySet().stream()
                .map(field -> new JoinFieldInfo(
                    group,
                    entry.getKey(),
                    field.getKey(),
                    field.getValue().getLabel(),
                    field.getValue().getOperator()
                ))
            ).collect(Collectors.toList());
    }

    public static List<JoinFieldInfo> findFieldsByLabel(String label, DdeMetadataProperties dde) {
        List<JoinFieldInfo> results = new ArrayList<>();
        for (String group : List.of("dimension", "measure", "filter")) {
            Map<String, CategoryConfig> groupMap = dde.getGroupMap(group);
            for (Map.Entry<String, CategoryConfig> entry : groupMap.entrySet()) {
                for (Map.Entry<String, FieldConfig> field : entry.getValue().getFields().entrySet()) {
                    if (label.equalsIgnoreCase(field.getValue().getLabel())) {
                        results.add(new JoinFieldInfo(
                            group,
                            entry.getKey(),
                            field.getKey(),
                            field.getValue().getLabel(),
                            field.getValue().getOperator()
                        ));
                    }
                }
            }
        }
        return results;
    }

    public static JoinFieldInfo getFieldInfo(String group, String category, String field, DdeMetadataProperties dde) {
        FieldConfig fc = dde.getFieldConfig(group, category, field);
        if (fc == null) return null;
        return new JoinFieldInfo(group, category, field, fc.getLabel(), fc.getOperator());
    }

    public static JoinFieldInfo getFieldInfo(String fullKey, DdeMetadataProperties dde, boolean includeOperator) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        return createJoinFieldInfo(fullKey, dde, includeOperator);
    }

    private static JoinFieldInfo createJoinFieldInfo(String fullKey, DdeMetadataProperties dde, boolean includeOperator) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        String g = parts[0], cat = parts[1], fld = parts[2];
        FieldConfig fc = dde.getFieldConfig(g, cat, fld);
        if (fc == null) return null;
        return new JoinFieldInfo(g, cat, fld, fc.getLabel(), includeOperator ? fc.getOperator() : null);
    }
}

```

---

### 🎯 목표

**사용자 입력 기반으로 CriteriaBuilder에서 필요한 Join 구문을 자동으로 판단**하려는 것입니다.
그런데 `app_usage`, `device`, `app_event` 등은 **조인 관계가 다릅니다:**

| 관계                           | Join 방식      | 이유                  |
| ---------------------------- | ------------ | ------------------- |
| `app_usage.foreground_usage` | ❌ 조인 없음      | 같은 테이블 내 필드         |
| `app_usage` ↔ `device`       | ✅ LEFT JOIN  | 외부 테이블, optional 관계 |
| `app_usage` ↔ `app_event`    | ✅ OUTER JOIN | 외부 테이블, nullable 관계 |

---

### ✅ 표현 방법

이런 조인 전략을 YAML 메타데이터에서 표현하기 위해서는 **join target에 joinType 필드를 명시**할 수 있습니다.

---

### ✅ 개선된 YAML 예시

```yaml
dde:
  measure:
    app_usage:
      label: "App Usage"
      fields:
        background_usage:
          label: "Background Usage"
          joins:
            dimension:
              - target: dimension:device:device_id
                joinType: LEFT
            measure:
              - target: measure:app_usage:foreground_usage  # 같은 테이블: No Join
                joinType: NONE
            filter:
              - target: event:app_event:event_type
                joinType: OUTER
```

---

### ✅ JoinTarget 클래스 확장

```java
@Data
public class JoinTarget {
    private String target;
    private String label;
    private String joinType; // "LEFT", "OUTER", "INNER", "NONE" 등
}
```

---

### ✅ JoinGraph 생성 시 joinType도 포함시키기

JoinGraph는 단순한 `Map<String, Set<String>>` 이 아니라, 다음처럼 바꿔야 이런 추가 속성을 유지할 수 있습니다:

```java
Map<String, List<JoinEdge>> graph;

@Data
public class JoinEdge {
    private String targetKey;
    private String joinType;
}
```

이렇게 바꾸면 `CriteriaBuilder`를 생성할 때:

```java
for (JoinEdge edge : graph.get(fromKey)) {
    switch (edge.getJoinType()) {
        case "LEFT"  -> cb.leftJoin(...);
        case "OUTER" -> cb.outerJoin(...);
        case "INNER" -> cb.innerJoin(...);
        default      -> // skip join
    }
}
```

---

### ✅ CriteriaBuilder 내 적용 예

```java
Join<?, ?> join;
if ("LEFT".equalsIgnoreCase(joinType)) {
    join = root.join("device", JoinType.LEFT);
} else if ("OUTER".equalsIgnoreCase(joinType)) {
    join = root.join("app_event", JoinType.RIGHT); // or RIGHT OUTER depending on JPA provider
} else if ("INNER".equalsIgnoreCase(joinType)) {
    join = root.join("app_event", JoinType.INNER);
} else {
    join = null; // same-table reference, no join needed
}
```

---

### ✅ 요약

| 구성 요소             | 설명                                                 |
| ----------------- | -------------------------------------------------- |
| `joinType`        | `LEFT`, `OUTER`, `INNER`, `NONE` 등의 문자열로 조인 방식 명시  |
| `JoinTarget`      | `target`, `label`, `joinType` 포함                   |
| `JoinEdge`        | join graph 를 표현할 때 `targetKey` + `joinType` 구조로 확장 |
| `CriteriaBuilder` | 이 정보를 기반으로 동적 조인 수행 가능                             |

---

필요하시면 `JoinEdge` 기반의 `JoinGraphUtil` 업데이트 코드도 제공드릴게요. 진행할까요?




--------------------------

좋습니다. `group` 이름에 따라 `category`와 그 안의 `fields` 목록만 추출하고 싶다면 아래와 같은 메서드를 제공할 수 있습니다:

---

### ✅ 원하는 출력 형태

예를 들어 `group = "dimension"`일 때:

```json
[
  {
    "category": "device",
    "fields": [
      { "field": "device_id", "label": "Device ID" },
      { "field": "group_id", "label": "Group ID" }
    ]
  },
  {
    "category": "app",
    "fields": [
      { "field": "app_name", "label": "App Name" }
    ]
  }
]
```

---

### ✅ 메서드 구현

```java
public List<Map<String, Object>> getCategoryAndFields(String group) {
    Map<String, CategoryConfig> groupMap = getGroupMap(group);
    if (groupMap == null) return List.of();

    List<Map<String, Object>> result = new ArrayList<>();

    for (Map.Entry<String, CategoryConfig> categoryEntry : groupMap.entrySet()) {
        String categoryKey = categoryEntry.getKey();
        CategoryConfig categoryConfig = categoryEntry.getValue();

        Map<String, String> categoryMeta = Map.of(
            "field", categoryKey,
            "label", categoryConfig.getLabel()
        );

        List<Map<String, String>> fields = categoryConfig.getFields().entrySet().stream()
            .map(entry -> Map.of(
                "field", entry.getKey(),
                "label", entry.getValue().getLabel()
            ))
            .collect(Collectors.toList());

        Map<String, Object> categoryBlock = new LinkedHashMap<>();
        categoryBlock.put("category", categoryMeta);
        categoryBlock.put("fields", fields);
        result.add(categoryBlock);
    }

    return result;
}

```

---

### ✅ 사용 예시

```java
List<Map<String, Object>> dimensionInfo = dde.getCategoryAndFields("dimension");
```

---

### ✅ 필요한 클래스에 포함

이 메서드는 `DdeMetadataProperties` 안에 넣으면 됩니다.
`getGroupMap(group)`을 이미 구현해두었으니 간단히 붙일 수 있습니다.

---

원하신다면 이 메서드도 문서에 바로 추가해드릴게요. 추가할까요?



```java
// DdeMetadataProperties.java
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.*;

@Data
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {
    private Map<String, CategoryConfig> dimension;
    private Map<String, CategoryConfig> filter;
    private Map<String, CategoryConfig> measure;

    public Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            case "filter" -> filter;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    public FieldConfig getFieldConfig(String group, String category, String fieldName) {
        Map<String, CategoryConfig> groupMap = getGroupMap(group);
        if (groupMap == null) return null;
        CategoryConfig categoryConfig = groupMap.get(category);
        if (categoryConfig == null || categoryConfig.getFields() == null) return null;
        return categoryConfig.getFields().get(fieldName);
    }

    public static FieldConfig getFieldConfig(String fullKey, DdeMetadataProperties dde) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        return dde.getFieldConfig(parts[0], parts[1], parts[2]);
    }

    public static Map<String, List<Map<String, Object>>> getJoinRecommendationsByIntent(
        String group, String category, String field,
        Map<String, Set<String>> graph,
        DdeMetadataProperties dde
    ) {
        String fromKey = group + ":" + category + ":" + field;
        Set<String> connections = graph.getOrDefault(fromKey, Set.of());

        Map<String, List<Map<String, Object>>> result = new LinkedHashMap<>();

        for (String targetKey : connections) {
            String[] parts = targetKey.split(":");
            if (parts.length != 3) continue;

            String actualGroup = parts[0];
            String cat = parts[1];
            String fld = parts[2];

            FieldConfig fc = getFieldConfig(targetKey, dde);
            if (fc == null) continue;

            Map<String, Object> entry = new LinkedHashMap<>();
            entry.put("group", actualGroup);
            entry.put("category", cat);
            entry.put("field", fld);
            entry.put("label", fc.getLabel());

            result.computeIfAbsent(actualGroup, k -> new ArrayList<>()).add(entry);
        }

        // 추가: joins.filter → 실제 filter 그룹이 아님, 의도된 filter 필드로 구성
        FieldConfig source = dde.getFieldConfig(group, category, field);
        if (source != null && source.getJoins() != null && source.getJoins().getFilter() != null) {
            List<Map<String, Object>> filterSection = new ArrayList<>();
            for (String filterKey : source.getJoins().getFilter()) {
                String[] parts = filterKey.split(":" );
                if (parts.length != 3) continue;
                String g = parts[0], cat = parts[1], fld = parts[2];

                FieldConfig fc = dde.getFieldConfig(g, cat, fld);
                if (fc == null) continue;

                Map<String, Object> entry = new LinkedHashMap<>();
                entry.put("group", g);
                entry.put("category", cat);
                entry.put("field", fld);
                entry.put("label", fc.getLabel());
                entry.put("operator", fc.getOperator());
                filterSection.add(entry);
            }
            result.put("filter", filterSection);
        }

        return result;
    }
}

// CategoryConfig.java
import lombok.Data;
import java.util.Map;

@Data
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields;
}

// FieldConfig.java
import lombok.Data;
import java.util.Map;

@Data
public class FieldConfig {
    private String label;
    private Map<String, String> metric;
    private Map<String, String> operator;
    private JoinTargets joins;
}

// JoinTargets.java
import lombok.Data;
import java.util.List;

@Data
public class JoinTargets {
    private List<String> measure;
    private List<String> dimension;
    private List<String> filter;
}


```

----------------

`getFieldConfig()` 메서드는 아래와 같은 방식으로 구현되어 있습니다. 핵심은 `group:category:field` 세 가지 값을 이용해 해당 `FieldConfig` 객체를 찾아내는 것입니다.

---

## ✅ 메서드 시그니처

```java
public FieldConfig getFieldConfig(String group, String category, String fieldName)
```

---

## ✅ 구현

```java
public FieldConfig getFieldConfig(String group, String category, String fieldName) {
    Map<String, CategoryConfig> groupMap = getGroupMap(group);
    if (groupMap == null) return null;

    CategoryConfig categoryConfig = groupMap.get(category);
    if (categoryConfig == null || categoryConfig.getFields() == null) return null;

    return categoryConfig.getFields().get(fieldName);
}
```

---

## ✅ 호출 예시

```java
FieldConfig config = dde.getFieldConfig("measure", "mv_app_usage", "totalRunTime");
if (config != null) {
    System.out.println(config.getLabel()); // "Total Run Time"
}
```

---

## ✅ 조건

이 메서드가 잘 작동하려면 다음 조건이 충족되어야 합니다:

1. `DdeMetadataProperties` 클래스에 `getGroupMap(String group)` 메서드가 구현되어 있어야 함:

```java
public Map<String, CategoryConfig> getGroupMap(String group) {
    return switch (group.toLowerCase()) {
        case "dimension" -> dimension;
        case "measure" -> measure;
        case "filter" -> filter;
        default -> throw new IllegalArgumentException("Unknown group: " + group);
    };
}
```

2. 해당 그룹 안에 category와 fieldName이 정확히 존재해야 함

---

필요하면 `"measure:mv_app_usage:totalRunTime"` 같은 키 문자열을 파싱해서 호출하는 헬퍼 함수도 만들어 드릴 수 있어요. 원하시나요?


-----------
아, 정확히 짚으셨습니다. 질문은:

> `JoinGraphUtil.getFieldConfig(targetKey, dde)` — 이게 어떻게 되는 거냐?

---

### ✅ 정답: `JoinGraphUtil` 클래스에 직접 만든 **static 유틸 메서드**입니다.

우리가 지금까지 만든 `DdeMetadataProperties.getFieldConfig(group, category, field)`는 **3개의 인자를 각각 받는 방식**이고,
`targetKey`는 문자열 하나 (`"measure:mv_app_usage:totalRunTime"`)이기 때문에, 이걸 파싱해서 내부적으로 호출하는 **도우미 메서드**가 필요합니다.

---

### ✅ 실제 구현 예시 (JoinGraphUtil 안에 정의)

```java
public static FieldConfig getFieldConfig(String fullKey, DdeMetadataProperties dde) {
    String[] parts = fullKey.split(":");
    if (parts.length != 3) return null;

    String group = parts[0];
    String category = parts[1];
    String field = parts[2];

    return dde.getFieldConfig(group, category, field);
}
```

---

### ✅ 사용 예

```java
String key = "measure:mv_app_usage:totalRunTime";
FieldConfig fc = JoinGraphUtil.getFieldConfig(key, dde);
```

이렇게 쓰면 `fullKey` 하나만 있으면 바로 `FieldConfig` 객체를 가져올 수 있어서 `graph` 탐색 시 유용합니다.

---

### ✅ 요약

* `JoinGraphUtil.getFieldConfig(String fullKey, DdeMetadataProperties dde)` 는 직접 만든 static 유틸 함수
* 내부적으로 `group`, `category`, `field` 로 분해해서 `dde.getFieldConfig(...)` 호출

필요하면 이걸 `JoinGraphUtil`에 추가해드릴 수도 있어요. 추가해드릴까요?

---

## ✅ 입력 형식

```json
{
  "group": "measure",
  "category": "mv_app_usage",
  "field": "totalRunTime"
}
```

## ✅ 동작

1. `group:category:field` → `"measure:mv_app_usage:totalRunTime"` 형태의 join key 구성
2. `JoinGraphUtil.buildJoinGraph()` 으로 전체 join graph 생성
3. 해당 키 기준으로 연결된 노드를 검색
4. 연결된 키들을 각각 `group`, `category`, `field` 로 파싱
5. 라벨 포함 정보로 응답

---

## ✅ 출력 예시

```json
{
  "measure": [
    {
      "group": "measure",
      "category": "mv_app_usage",
      "field": "foregroundUsage",
      "label": "Foreground Usage"
    }
  ],
  "dimension": [
    {
      "group": "dimension",
      "category": "device",
      "field": "device_id",
      "label": "Device ID"
    }
  ],
  "filter": [
    {
      "group": "dimension",
      "category": "app",
      "field": "app_name",
      "label": "App Name"
    }
  ]
}
```

---

## ✅ 구현 메서드 예시

```java
public static Map<String, List<Map<String, String>>> getJoinRecommendations(
        String group, String category, String field,
        Map<String, Set<String>> graph,
        DdeMetadataProperties dde
) {
    String fromKey = JoinGraphUtil.toKey(group, category, field);
    Set<String> connected = graph.getOrDefault(fromKey, Set.of());

    Map<String, List<Map<String, String>>> result = new LinkedHashMap<>();
    for (String targetKey : connected) {
        String[] parts = targetKey.split(":");
        if (parts.length != 3) continue;

        String tg = parts[0];
        String cat = parts[1];
        String fld = parts[2];
        FieldConfig fc = JoinGraphUtil.getFieldConfig(targetKey, dde);
        if (fc == null) continue;

        Map<String, String> entry = new LinkedHashMap<>();
        entry.put("group", tg);
        entry.put("category", cat);
        entry.put("field", fld);
        entry.put("label", fc.getLabel());

        result.computeIfAbsent(tg, k -> new ArrayList<>()).add(entry);
    }

    return result;
}
```

---

## ✅ 사용 예시

```java
Map<String, Set<String>> graph = JoinGraphUtil.buildJoinGraph(dde);
Map<String, List<Map<String, String>>> res = getJoinRecommendations(
    "measure", "mv_app_usage", "totalRunTime", graph, dde
);

ObjectMapper mapper = new ObjectMapper();
System.out.println(mapper.writerWithDefaultPrettyPrinter().writeValueAsString(res));
```

---

원하신 `group, category, field` 입력 → 관련된 group별 연결 리스트를 구조화된 JSON으로 리턴하는 로직입니다. 이걸 기반으로 API 응답 포맷도 구성 가능합니다.

필요하시면 REST 컨트롤러 버전도 제공드릴게요.


---

### ✅ 변경 요약

1. **YAML 구조 변경에 따른 반영**

   * `joins` 필드는 `List<String>` 형태로 단순화 (`target` 포함한 key 직접 기입)
   * `operator` 필드 추가 → 각 필드 기준 비교 연산자 조회 가능
   * `label` → 여전히 field 및 category 수준에 존재

2. **Java 모델 변경**

   * `FieldConfig`에 `Map<String, String> operator` 필드 추가
   * `JoinTargets` → `List<String> measure`, `dimension`, `filter` 형태로 간소화

3. **새 유틸 메서드**

   * `getFieldConfig(String fullKey, DdeMetadataProperties dde)` – 필드 정보 조회
   * `getFieldMetadata(String fullKey, DdeMetadataProperties dde)` – `label`, `operator` 반환

---

### 🛠 다음 작업

* YML 구조도 이에 맞게 아래와 같이 변경해야 합니다:

```yaml
dde:
  measure:
    mv_app_usage:
      label: "App Usage Daily"
      fields:
        totalRunTime:
          label: "Total Run Time"
          metric:
            sum: "Sum (EA)"
            avg: "Avg (%)"
          operator:
            lt: "<"
            eq: "="
            gt: ">"
          joins:
            measure:
              - "measure:mv_app_usage:foreground_usage"
            dimension:
              - "dimension:device:device_id"
              - "dimension:device:group_id"
              - "dimension:app:app_name"
            filter:
              - "measure:appUsage:foreground_usage"
              - "dimension:device:device_id"
              - "dimension:device:group_id"
              - "dimension:app:app_name"
```

---

Java 모델 클래스들도 YML 변경 사항에 맞춰 업데이트해야 합니다. 원하신다면 해당 Java 클래스들 (`FieldConfig`, `JoinTargets` 등)도 새 구조에 맞게 리팩토링해 드릴게요. 진행할까요?


```java

// graph.java
// JoinGraphUtil.java
public class JoinGraphUtil {

    public static Map<String, Set<String>> buildJoinGraph(DdeMetadataProperties dde) {
        Map<String, Set<String>> graph = new HashMap<>();
        buildFromCategory("dimension", dde.getDimension(), graph);
        buildFromCategory("filter", dde.getFilter(), graph);
        buildFromCategory("measure", dde.getMeasure(), graph);
        return graph;
    }

    private static void buildFromCategory(String group, Map<String, CategoryConfig> map, Map<String, Set<String>> graph) {
        for (Map.Entry<String, CategoryConfig> categoryEntry : map.entrySet()) {
            String category = categoryEntry.getKey();
            Map<String, FieldConfig> fields = categoryEntry.getValue().getFields();
            if (fields == null) continue;

            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                String field = fieldEntry.getKey();
                String sourceKey = toKey(group, category, field);

                FieldConfig fieldConfig = fieldEntry.getValue();
                if (fieldConfig.getJoins() != null) {
                    addJoinsToGraph(sourceKey, fieldConfig.getJoins(), graph);
                }
            }
        }
    }

    private static void addJoinsToGraph(String sourceKey, JoinTargets joins, Map<String, Set<String>> graph) {
        addJoinList(sourceKey, joins.getMeasure(), graph);
        addJoinList(sourceKey, joins.getDimension(), graph);
        addJoinList(sourceKey, joins.getFilter(), graph);
    }

    private static void addJoinList(String sourceKey, List<String> targets, Map<String, Set<String>> graph) {
        if (targets == null) return;
        for (String target : targets) {
            graph.computeIfAbsent(sourceKey, k -> new HashSet<>()).add(target);
        }
    }

    public static String toKey(String group, String category, String field) {
        return group + ":" + category + ":" + field;
    }

    public static FieldConfig getFieldConfig(String fullKey, DdeMetadataProperties dde) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        String group = parts[0];
        String category = parts[1];
        String field = parts[2];

        Map<String, CategoryConfig> groupMap = getGroupMap(group, dde);
        if (groupMap == null) return null;
        CategoryConfig categoryConfig = groupMap.get(category);
        if (categoryConfig == null) return null;
        return categoryConfig.getFields().get(field);
    }

    public static Map<String, Object> getFieldMetadata(String fullKey, DdeMetadataProperties dde) {
        FieldConfig field = getFieldConfig(fullKey, dde);
        if (field == null) return Map.of();

        Map<String, Object> result = new LinkedHashMap<>();
        result.put("label", field.getLabel());
        result.put("operator", field.getOperator());
        return result;
    }

    public static Map<String, CategoryConfig> getGroupMap(String group, DdeMetadataProperties dde) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dde.getDimension();
            case "measure" -> dde.getMeasure();
            case "filter" -> dde.getFilter();
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }
}


// DdeMetadataProperties.java
import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

import java.util.Map;

@Data
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {
    private Map<String, CategoryConfig> dimension;
    private Map<String, CategoryConfig> filter;
    private Map<String, CategoryConfig> measure;

    public Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            case "filter" -> filter;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    public String getCategoryKeyByLabel(String group, String label) {
        return getGroupMap(group).entrySet().stream()
                .filter(e -> label.equalsIgnoreCase(e.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst().orElse(null);
    }

    public String getFieldName(String group, String category, String label) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        return config.getFields().entrySet().stream()
                .filter(e -> label.equalsIgnoreCase(e.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst().orElse(null);
    }

    public String getFieldLabel(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        FieldConfig field = config.getFields().get(fieldName);
        return field != null ? field.getLabel() : null;
    }

    public Map<String, String> getFieldLabels(String group, String category) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return Map.of();
        return config.getFields().entrySet().stream()
                .collect(java.util.stream.Collectors.toMap(Map.Entry::getKey, e -> e.getValue().getLabel()));
    }

    public Map<String, String> getFieldMetrics(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null) return Map.of();
        FieldConfig field = config.getFields().get(fieldName);
        return field != null && field.getMetric() != null ? field.getMetric() : Map.of();
    }

    public FieldConfig getFieldConfig(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        return config != null ? config.getFields().get(fieldName) : null;
    }

    public java.util.List<String> getCategoryLabels(String group) {
        return getGroupMap(group).values().stream()
                .map(CategoryConfig::getLabel)
                .toList();
    }

    public String findCategoryByLabelAcrossGroups(String label) {
        for (String group : java.util.List.of("dimension", "measure", "filter")) {
            String match = getCategoryKeyByLabel(group, label);
            if (match != null) return group + ":" + match;
        }
        return null;
    }

    public String findFieldNameAcrossGroups(String categoryLabel, String fieldLabel) {
        for (String group : java.util.List.of("dimension", "measure", "filter")) {
            Map<String, CategoryConfig> groupMap = getGroupMap(group);
            for (Map.Entry<String, CategoryConfig> entry : groupMap.entrySet()) {
                if (categoryLabel.equalsIgnoreCase(entry.getValue().getLabel())) {
                    String fieldKey = entry.getValue().getFields().entrySet().stream()
                            .filter(f -> fieldLabel.equalsIgnoreCase(f.getValue().getLabel()))
                            .map(Map.Entry::getKey)
                            .findFirst().orElse(null);
                    if (fieldKey != null) return group + ":" + entry.getKey() + ":" + fieldKey;
                }
            }
        }
        return null;
    }
}

// CategoryConfig.java
import lombok.Data;
import java.util.Map;

@Data
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields;
}

// FieldConfig.java
import lombok.Data;
import java.util.Map;

@Data
public class FieldConfig {
    private String label;
    private Map<String, String> metric;
    private Map<String, String> operator;
    private JoinTargets joins;
}

// JoinTargets.java
import lombok.Data;
import java.util.List;

@Data
public class JoinTargets {
    private List<String> measure;
    private List<String> dimension;
    private List<String> filter;
}
```
--------------

`JoinGraphUtil` 클래스에 다음과 같은 메서드들을 모두 반영하여 개정했습니다:

* `getGroupMap`: 그룹명에 따른 config 맵 반환
* `getCategoryLabels`: 해당 group의 label 목록
* `getCategoryKeyByLabel`: label을 기반으로 category 키 추출
* `getFieldName`: label로부터 field 키 반환
* `getFieldLabel`: field 키로 label 반환
* `getFieldLabels`: category 내 모든 field 키-라벨 매핑
* `getFieldMetrics`: metric 정보 반환 (`Map<String, String>` 구조로 간주)

이제 이 유틸만으로 YML 기반의 메타데이터 탐색, label ↔ key 매핑, join graph 생성 등 대부분의 메타 처리 기능을 커버할 수 있습니다. 필요한 경우 테스트 유닛도 도와드릴 수 있습니다.


```java

// JoinGraphUtil.java
public class JoinGraphUtil {

    public static Map<String, Set<String>> buildJoinGraph(DdeMetadataProperties dde) {
        Map<String, Set<String>> graph = new HashMap<>();
        buildFromCategory("dimension", dde.getDimension(), graph);
        buildFromCategory("filter", dde.getFilter(), graph);
        buildFromCategory("measure", dde.getMeasure(), graph);
        return graph;
    }

    private static void buildFromCategory(String group, Map<String, CategoryConfig> map, Map<String, Set<String>> graph) {
        for (Map.Entry<String, CategoryConfig> categoryEntry : map.entrySet()) {
            String category = categoryEntry.getKey();
            Map<String, FieldConfig> fields = categoryEntry.getValue().getFields();
            if (fields == null) continue;

            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                String field = fieldEntry.getKey();
                String sourceKey = toKey(group, category, field);

                FieldConfig fieldConfig = fieldEntry.getValue();
                if (fieldConfig.getJoins() != null) {
                    addJoinsToGraph(sourceKey, fieldConfig.getJoins(), graph);
                }
            }
        }
    }

    private static void addJoinsToGraph(String sourceKey, JoinTargets joins, Map<String, Set<String>> graph) {
        addJoinList(sourceKey, "measure", joins.getMeasure(), graph);
        addJoinList(sourceKey, "dimension", joins.getDimension(), graph);
        addJoinList(sourceKey, "filter", joins.getFilter(), graph);
    }

    private static void addJoinList(String sourceKey, String targetGroup, List<JoinTarget> targets, Map<String, Set<String>> graph) {
        if (targets == null) return;
        for (JoinTarget jt : targets) {
            String[] parts = jt.getTarget().split("\\.");
            if (parts.length == 2) {
                String category = parts[0];
                String field = parts[1];
                String targetKey = toKey(targetGroup, category, field);
                graph.computeIfAbsent(sourceKey, k -> new HashSet<>()).add(targetKey);
            }
        }
    }

    public static boolean isJoinable(String from, String to, Map<String, Set<String>> graph) {
        Set<String> visited = new HashSet<>();
        Queue<String> queue = new LinkedList<>();
        queue.add(from);
        visited.add(from);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (current.equals(to)) return true;
            for (String neighbor : graph.getOrDefault(current, Set.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.add(neighbor);
                }
            }
        }
        return false;
    }

    public static List<String> recommendJoinKeys(String from, Map<String, Set<String>> graph) {
        return new ArrayList<>(graph.getOrDefault(from, Set.of()));
    }

    public static Map<String, String> recommendLabelWithKey(String from, Map<String, Set<String>> graph, DdeMetadataProperties dde) {
        Map<String, String> result = new LinkedHashMap<>();
        for (String key : graph.getOrDefault(from, Set.of())) {
            String label = findLabelByKey(key, dde);
            if (label != null) {
                result.put(key, label);
            }
        }
        return result;
    }

    public static Map<String, String> recommendByGroup(String from, String targetGroup, Map<String, Set<String>> graph, DdeMetadataProperties dde) {
        Map<String, String> result = new LinkedHashMap<>();
        for (String key : graph.getOrDefault(from, Set.of())) {
            if (key.startsWith(targetGroup + ":")) {
                String label = findLabelByKey(key, dde);
                if (label != null) {
                    result.put(key, label);
                }
            }
        }
        return result;
    }

    public static String toKey(String group, String category, String field) {
        return group + ":" + category + ":" + field;
    }

    public static Map<String, CategoryConfig> getGroupMap(String group, DdeMetadataProperties dde) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dde.getDimension();
            case "measure" -> dde.getMeasure();
            case "filter" -> dde.getFilter();
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    public static List<String> getCategoryLabels(String group, DdeMetadataProperties dde) {
        return getGroupMap(group, dde).values().stream()
                .map(CategoryConfig::getLabel)
                .toList();
    }

    public static String getCategoryKeyByLabel(String group, String label, DdeMetadataProperties dde) {
        return getGroupMap(group, dde).entrySet().stream()
                .filter(entry -> entry.getValue().getLabel().equalsIgnoreCase(label))
                .map(Map.Entry::getKey)
                .findFirst()
                .orElse(null);
    }

    public static String getFieldName(String group, String category, String label, DdeMetadataProperties dde) {
        CategoryConfig config = getGroupMap(group, dde).get(category);
        if (config == null) return null;
        return config.getFields().entrySet().stream()
                .filter(entry -> label.equalsIgnoreCase(entry.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst()
                .orElse(null);
    }

    public static String getFieldLabel(String group, String category, String fieldName, DdeMetadataProperties dde) {
        CategoryConfig config = getGroupMap(group, dde).get(category);
        if (config == null || config.getFields() == null) return null;
        FieldConfig field = config.getFields().get(fieldName);
        return field != null ? field.getLabel() : null;
    }

    public static Map<String, String> getFieldLabels(String group, String category, DdeMetadataProperties dde) {
        CategoryConfig config = getGroupMap(group, dde).get(category);
        if (config == null || config.getFields() == null) return Map.of();
        return config.getFields().entrySet().stream()
                .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().getLabel()));
    }

    public static Map<String, String> getFieldMetrics(String group, String category, String fieldName, DdeMetadataProperties dde) {
        CategoryConfig config = getGroupMap(group, dde).get(category);
        if (config == null) return Map.of();
        FieldConfig field = config.getFields().get(fieldName);
        return field != null && field.getMetric() != null ? field.getMetric().entrySet().stream()
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue)) : Map.of();
    }

    private static String findLabelByKey(String fullKey, DdeMetadataProperties dde) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        String group = parts[0];
        String category = parts[1];
        String field = parts[2];

        Map<String, CategoryConfig> groupMap = getGroupMap(group, dde);
        CategoryConfig config = groupMap.get(category);
        if (config == null || config.getFields() == null) return null;
        FieldConfig fieldConfig = config.getFields().get(field);
        return fieldConfig != null ? fieldConfig.getLabel() : null;
    }
}


```

-----------------------

아주 현실적이고 중요한 질문입니다. 실제 데이터 모델링에서 **measure/dimension/filter에 같은 category 이름이나 같은 field name**이 존재할 수 있으며, **이들이 다른 물리 테이블**을 의미하는 경우가 많습니다.
예: `dimension.device.event_type` ≠ `measure.device.event_type`

---

## ✅ 문제 요약

| 문제            | 설명                                    |
| ------------- | ------------------------------------- |
| 카테고리 이름이 겹침   | `device`가 measure에도 있고 dimension에도 있음 |
| 필드 이름이 겹침     | `event_type` 같은 필드명이 여러 카테고리에 존재      |
| DB 실제 테이블은 다름 | 그런데 이름만 보면 구분이 안 됨                    |

→ 이 상태로는 **조인 그래프, label 매핑, 쿼리 생성 등에서 충돌** 발생 가능

---

## ✅ 해결 전략

### 🔹 1. **Unique Fully Qualified Key (고유 식별자 키) 사용**

**구분 가능한 식별자 체계**를 만들어야 합니다.

```text
<group>:<category>:<field>
예: measure:device:event_type
     dimension:device:event_type
```

→ 그래프 key 및 조회 시 항상 **이 고유키로 관리**

---

### 🔹 2. YAML 구조는 그대로 두되, 내부 키 관리만 고유 키 사용

예: YAML은 다음처럼 간결하게 유지하고,

```yaml
dde:
  measure:
    device:
      fields:
        event_type:
          label: "Event Type"
```

Java 내부에서 이걸 처리할 때 다음 방식으로 변환:

```java
String fieldKey = "measure:device:event_type";
```

그래프 및 검색에서 이 키를 사용함으로써 **동명이인 문제를 방지**할 수 있습니다.

---

## ✅ 구현 예

### 1. 그래프 노드 키를 `group:category:field` 형태로 생성

```java
String nodeKey = group + ":" + category + ":" + fieldName;
```

그래프는:

```java
graph.put("dimension:device:event_type", Set.of("measure:device:event_type"));
```

---

### 2. label 조회 시도 시에도 이 키를 기준으로 처리

```java
String[] parts = key.split(":"); // [group, category, field]
```

* `group`으로 `dde.getMeasure()`/`getDimension()`/`getFilter()` 중 선택
* `category`로 `get(category)`
* `field`로 `getFields().get(field)`

---

### 3. join 관계도 고유 키 기반으로 생성

join YAML은 여전히 간단히:

```yaml
joins:
  measure:
    - target: device.event_type
      label: "Device Event Type"
```

Java에서 파싱 시 `target`을 `"measure:device:event_type"`으로 자동 변환

---

## ✅ 보너스: label 충돌 방지

이렇게 되면 `"Event Type"`이 여러 군데 있어도 다음처럼 명확하게 라벨을 쓸 수 있습니다:

| Key                           | Label                  |
| ----------------------------- | ---------------------- |
| `measure:device:event_type`   | Event Type (Measure)   |
| `dimension:device:event_type` | Event Type (Dimension) |

→ 필요 시 라벨에도 구분자를 자동 추가할 수 있습니다.

---

## ✅ 결론

| 선택지                      | 추천 여부   | 설명         |
| ------------------------ | ------- | ---------- |
| 그룹/카테고리/필드명 조합으로 고유 키 생성 | ✅ 적극 추천 | 가장 안정적인 방식 |
| YAML에서 직접 prefix 붙이기     | △       | 사람이 보기 불편함 |
| 필드 이름만으로 처리              | ❌ 위험    | 충돌 확률 높음   |

---

### 원하시면

* `group:category:field` 기반 키 생성기
* 전체 그래프 구성 유틸 수정
* `findLabel(...)` / `recommend...` 들을 고유 키 기반으로 리팩토링

도 한번에 도와드릴 수 있습니다. 적용해볼까요?


```java
// === 고유키 기반 Join 추천 확장 ===

public class JoinGraphUtil {

    public static Map<String, Set<String>> buildJoinGraph(DdeMetadataProperties dde) {
        Map<String, Set<String>> graph = new HashMap<>();

        buildFromCategory("dimension", dde.getDimension(), graph);
        buildFromCategory("filter", dde.getFilter(), graph);
        buildFromMeasure("measure", dde.getMeasure(), graph);

        return graph;
    }

    private static void buildFromCategory(String group, Map<String, CategoryConfig> map, Map<String, Set<String>> graph) {
        for (Map.Entry<String, CategoryConfig> categoryEntry : map.entrySet()) {
            String category = categoryEntry.getKey();
            Map<String, FieldConfig> fields = categoryEntry.getValue().getFields();
            if (fields == null) continue;

            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                String field = fieldEntry.getKey();
                String sourceKey = toKey(group, category, field);

                FieldConfig fieldConfig = fieldEntry.getValue();
                if (fieldConfig.getJoins() != null) {
                    addJoinsToGraph(sourceKey, fieldConfig.getJoins(), graph);
                }
            }
        }
    }

    private static void buildFromMeasure(String group, Map<String, MeasureConfig> map, Map<String, Set<String>> graph) {
        for (Map.Entry<String, MeasureConfig> categoryEntry : map.entrySet()) {
            String category = categoryEntry.getKey();
            Map<String, FieldConfig> fields = categoryEntry.getValue().getFields();
            if (fields == null) continue;

            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                String field = fieldEntry.getKey();
                String sourceKey = toKey(group, category, field);

                FieldConfig fieldConfig = fieldEntry.getValue();
                if (fieldConfig.getJoins() != null) {
                    addJoinsToGraph(sourceKey, fieldConfig.getJoins(), graph);
                }
            }
        }
    }

    private static void addJoinsToGraph(String sourceKey, JoinTargets joins, Map<String, Set<String>> graph) {
        addJoinList(sourceKey, "measure", joins.getMeasure(), graph);
        addJoinList(sourceKey, "dimension", joins.getDimension(), graph);
        addJoinList(sourceKey, "filter", joins.getFilter(), graph);
    }

    private static void addJoinList(String sourceKey, String targetGroup, List<JoinTarget> targets, Map<String, Set<String>> graph) {
        for (JoinTarget jt : targets) {
            String[] parts = jt.getTarget().split("\\.");
            if (parts.length == 2) {
                String category = parts[0];
                String field = parts[1];
                String targetKey = toKey(targetGroup, category, field);
                graph.computeIfAbsent(sourceKey, k -> new HashSet<>()).add(targetKey);
            }
        }
    }

    public static boolean isJoinable(String from, String to, Map<String, Set<String>> graph) {
        Set<String> visited = new HashSet<>();
        Queue<String> queue = new LinkedList<>();
        queue.add(from);
        visited.add(from);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (current.equals(to)) return true;
            for (String neighbor : graph.getOrDefault(current, Set.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.add(neighbor);
                }
            }
        }
        return false;
    }

    public static boolean hasCycleBFS(Map<String, Set<String>> graph) {
        Set<String> visited = new HashSet<>();
        for (String start : graph.keySet()) {
            if (!visited.contains(start)) {
                if (detectCycleFrom(start, graph, visited)) {
                    return true;
                }
            }
        }
        return false;
    }

    private static boolean detectCycleFrom(String start, Map<String, Set<String>> graph, Set<String> globalVisited) {
        Map<String, String> parent = new HashMap<>();
        Queue<String> queue = new LinkedList<>();
        Set<String> visited = new HashSet<>();

        queue.add(start);
        visited.add(start);
        globalVisited.add(start);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            for (String neighbor : graph.getOrDefault(current, Set.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    globalVisited.add(neighbor);
                    parent.put(neighbor, current);
                    queue.add(neighbor);
                } else if (!neighbor.equals(parent.get(current))) {
                    return true;
                }
            }
        }
        return false;
    }

    public static List<String> recommendJoinKeys(String from, Map<String, Set<String>> graph) {
        return new ArrayList<>(graph.getOrDefault(from, Set.of()));
    }

    public static List<String> recommendJoinLabels(String from, Map<String, Set<String>> graph, DdeMetadataProperties dde) {
        List<String> keys = recommendJoinKeys(from, graph);
        List<String> labels = new ArrayList<>();
        for (String key : keys) {
            String label = findLabelByKey(key, dde);
            if (label != null) {
                labels.add(label);
            }
        }
        return labels;
    }

    public static Map<String, String> recommendLabelWithKey(String from, Map<String, Set<String>> graph, DdeMetadataProperties dde) {
        Map<String, String> result = new LinkedHashMap<>();
        for (String key : graph.getOrDefault(from, Set.of())) {
            String label = findLabelByKey(key, dde);
            if (label != null) {
                result.put(key, label);
            }
        }
        return result;
    }

    public static Map<String, String> recommendByGroup(String from, String targetGroup, Map<String, Set<String>> graph, DdeMetadataProperties dde) {
        Map<String, String> result = new LinkedHashMap<>();
        for (String key : graph.getOrDefault(from, Set.of())) {
            if (key.startsWith(targetGroup + ":")) {
                String label = findLabelByKey(key, dde);
                if (label != null) {
                    result.put(key, label);
                }
            }
        }
        return result;
    }

    public static String toKey(String group, String category, String field) {
        return group + ":" + category + ":" + field;
    }

    private static String findLabelByKey(String fullKey, DdeMetadataProperties dde) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        String group = parts[0];
        String category = parts[1];
        String field = parts[2];

        switch (group) {
            case "measure":
                MeasureConfig m = dde.getMeasure().get(category);
                if (m != null && m.getFields().containsKey(field)) return m.getFields().get(field).getLabel();
                break;
            case "dimension":
                CategoryConfig d = dde.getDimension().get(category);
                if (d != null && d.getFields().containsKey(field)) return d.getFields().get(field).getLabel();
                break;
            case "filter":
                CategoryConfig f = dde.getFilter().get(category);
                if (f != null && f.getFields().containsKey(field)) return f.getFields().get(field).getLabel();
                break;
        }
        return null;
    }
}

/////////////

// DdeMetadataProperties.java
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {
    private Map<String, CategoryConfig> dimension;
    private Map<String, CategoryConfig> filter;
    private Map<String, MeasureConfig> measure;

    // getters and setters
    public Map<String, CategoryConfig> getDimension() { return dimension; }
    public void setDimension(Map<String, CategoryConfig> dimension) { this.dimension = dimension; }

    public Map<String, CategoryConfig> getFilter() { return filter; }
    public void setFilter(Map<String, CategoryConfig> filter) { this.filter = filter; }

    public Map<String, MeasureConfig> getMeasure() { return measure; }
    public void setMeasure(Map<String, MeasureConfig> measure) { this.measure = measure; }
}

// CategoryConfig.java
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields;

    // getters and setters
    public String getLabel() { return label; }
    public void setLabel(String label) { this.label = label; }

    public Map<String, FieldConfig> getFields() { return fields; }
    public void setFields(Map<String, FieldConfig> fields) { this.fields = fields; }
}

// MeasureConfig.java
public class MeasureConfig {
    private String label;
    private Map<String, FieldConfig> fields;

    // getters and setters
    public String getLabel() { return label; }
    public void setLabel(String label) { this.label = label; }

    public Map<String, FieldConfig> getFields() { return fields; }
    public void setFields(Map<String, FieldConfig> fields) { this.fields = fields; }
}

// FieldConfig.java
public class FieldConfig {
    private String label;
    private JoinTargets joins;
    private MetricConfig metric;

    // getters and setters
    public String getLabel() { return label; }
    public void setLabel(String label) { this.label = label; }

    public JoinTargets getJoins() { return joins; }
    public void setJoins(JoinTargets joins) { this.joins = joins; }

    public MetricConfig getMetric() { return metric; }
    public void setMetric(MetricConfig metric) { this.metric = metric; }
}

// JoinTargets.java
public class JoinTargets {
    private List<JoinTarget> measure;
    private List<JoinTarget> dimension;
    private List<JoinTarget> filter;

    // getters and setters
    public List<JoinTarget> getMeasure() { return measure; }
    public void setMeasure(List<JoinTarget> measure) { this.measure = measure; }

    public List<JoinTarget> getDimension() { return dimension; }
    public void setDimension(List<JoinTarget> dimension) { this.dimension = dimension; }

    public List<JoinTarget> getFilter() { return filter; }
    public void setFilter(List<JoinTarget> filter) { this.filter = filter; }
}

// JoinTarget.java
public class JoinTarget {
    private String target;
    private String label;

    // getters and setters
    public String getTarget() { return target; }
    public void setTarget(String target) { this.target = target; }

    public String getLabel() { return label; }
    public void setLabel(String label) { this.label = label; }
}

// MetricConfig.java
public class MetricConfig {
    private Map<String, String> sum;
    private Map<String, String> avg;
    private List<JoinTarget> measure;
    private List<JoinTarget> dimension;
    private List<JoinTarget> filter;

    // getters and setters
    public Map<String, String> getSum() { return sum; }
    public void setSum(Map<String, String> sum) { this.sum = sum; }

    public Map<String, String> getAvg() { return avg; }
    public void setAvg(Map<String, String> avg) { this.avg = avg; }

    public List<JoinTarget> getMeasure() { return measure; }
    public void setMeasure(List<JoinTarget> measure) { this.measure = measure; }

    public List<JoinTarget> getDimension() { return dimension; }
    public void setDimension(List<JoinTarget> dimension) { this.dimension = dimension; }

    public List<JoinTarget> getFilter() { return filter; }
    public void setFilter(List<JoinTarget> filter) { this.filter = filter; }
}




```

-------------------------
```java

// === 고유키 기반으로 Join 그래프를 구성하는 구조 ===

public class JoinGraphUtil {

    public static Map<String, Set<String>> buildJoinGraph(DdeMetadataProperties dde) {
        Map<String, Set<String>> graph = new HashMap<>();

        buildFromCategory("dimension", dde.getDimension(), graph);
        buildFromCategory("filter", dde.getFilter(), graph);
        buildFromMeasure("measure", dde.getMeasure(), graph);

        return graph;
    }

    private static void buildFromCategory(String group, Map<String, CategoryConfig> map, Map<String, Set<String>> graph) {
        for (Map.Entry<String, CategoryConfig> categoryEntry : map.entrySet()) {
            String category = categoryEntry.getKey();
            Map<String, FieldConfig> fields = categoryEntry.getValue().getFields();
            if (fields == null) continue;

            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                String field = fieldEntry.getKey();
                String sourceKey = toKey(group, category, field);

                FieldConfig fieldConfig = fieldEntry.getValue();
                if (fieldConfig.getJoins() != null) {
                    addJoinsToGraph(sourceKey, fieldConfig.getJoins(), graph);
                }
            }
        }
    }

    private static void buildFromMeasure(String group, Map<String, MeasureConfig> map, Map<String, Set<String>> graph) {
        for (Map.Entry<String, MeasureConfig> categoryEntry : map.entrySet()) {
            String category = categoryEntry.getKey();
            Map<String, FieldConfig> fields = categoryEntry.getValue().getFields();
            if (fields == null) continue;

            for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                String field = fieldEntry.getKey();
                String sourceKey = toKey(group, category, field);

                FieldConfig fieldConfig = fieldEntry.getValue();
                if (fieldConfig.getJoins() != null) {
                    addJoinsToGraph(sourceKey, fieldConfig.getJoins(), graph);
                }
            }
        }
    }

    private static void addJoinsToGraph(String sourceKey, JoinTargets joins, Map<String, Set<String>> graph) {
        addJoinList(sourceKey, "measure", joins.getMeasure(), graph);
        addJoinList(sourceKey, "dimension", joins.getDimension(), graph);
        addJoinList(sourceKey, "filter", joins.getFilter(), graph);
    }

    private static void addJoinList(String sourceKey, String targetGroup, List<JoinTarget> targets, Map<String, Set<String>> graph) {
        for (JoinTarget jt : targets) {
            String[] parts = jt.getTarget().split("\\.");
            if (parts.length == 2) {
                String category = parts[0];
                String field = parts[1];
                String targetKey = toKey(targetGroup, category, field);
                graph.computeIfAbsent(sourceKey, k -> new HashSet<>()).add(targetKey);
            }
        }
    }

    public static boolean isJoinable(String from, String to, Map<String, Set<String>> graph) {
        Set<String> visited = new HashSet<>();
        Queue<String> queue = new LinkedList<>();
        queue.add(from);
        visited.add(from);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (current.equals(to)) return true;
            for (String neighbor : graph.getOrDefault(current, Set.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.add(neighbor);
                }
            }
        }
        return false;
    }

    public static boolean hasCycleBFS(Map<String, Set<String>> graph) {
        Set<String> visited = new HashSet<>();
        for (String start : graph.keySet()) {
            if (!visited.contains(start)) {
                if (detectCycleFrom(start, graph, visited)) {
                    return true;
                }
            }
        }
        return false;
    }

    private static boolean detectCycleFrom(String start, Map<String, Set<String>> graph, Set<String> globalVisited) {
        Map<String, String> parent = new HashMap<>();
        Queue<String> queue = new LinkedList<>();
        Set<String> visited = new HashSet<>();

        queue.add(start);
        visited.add(start);
        globalVisited.add(start);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            for (String neighbor : graph.getOrDefault(current, Set.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    globalVisited.add(neighbor);
                    parent.put(neighbor, current);
                    queue.add(neighbor);
                } else if (!neighbor.equals(parent.get(current))) {
                    return true;
                }
            }
        }
        return false;
    }

    public static List<String> recommendJoinKeys(String from, Map<String, Set<String>> graph) {
        return new ArrayList<>(graph.getOrDefault(from, Set.of()));
    }

    public static List<String> recommendJoinLabels(String from, Map<String, Set<String>> graph, DdeMetadataProperties dde) {
        List<String> keys = recommendJoinKeys(from, graph);
        List<String> labels = new ArrayList<>();
        for (String key : keys) {
            String label = findLabelByKey(key, dde);
            if (label != null) {
                labels.add(label);
            }
        }
        return labels;
    }

    public static String toKey(String group, String category, String field) {
        return group + ":" + category + ":" + field;
    }

    private static String findLabelByKey(String fullKey, DdeMetadataProperties dde) {
        String[] parts = fullKey.split(":");
        if (parts.length != 3) return null;
        String group = parts[0];
        String category = parts[1];
        String field = parts[2];

        switch (group) {
            case "measure":
                MeasureConfig m = dde.getMeasure().get(category);
                if (m != null && m.getFields().containsKey(field)) return m.getFields().get(field).getLabel();
                break;
            case "dimension":
                CategoryConfig d = dde.getDimension().get(category);
                if (d != null && d.getFields().containsKey(field)) return d.getFields().get(field).getLabel();
                break;
            case "filter":
                CategoryConfig f = dde.getFilter().get(category);
                if (f != null && f.getFields().containsKey(field)) return f.getFields().get(field).getLabel();
                break;
        }
        return null;
    }
}



// === application.yml 예시 ===
dde:
  dimension:
    device:
      label: "Device Name"
      fields:
        device_model:
          label: "Device Model"
          joins:
            measure:
              - target: foreground_usage
                label: "Foreground Usage"
              - target: app_name
                label: "App Name"
            dimension:
              - target: device_id
                label: "Device ID"
              - target: group_id
                label: "Group ID"
            filter:
              - target: usage_date
                label: "Usage Date"


// === Java 클래스 구조 ===

public class DdeMetadataProperties {
    private Map<String, CategoryConfig> dimension = new HashMap<>();
    private Map<String, MeasureConfig> measure = new HashMap<>();
    private Map<String, CategoryConfig> filter = new HashMap<>();

    public Map<String, CategoryConfig> getDimension() {
        return dimension;
    }

    public Map<String, MeasureConfig> getMeasure() {
        return measure;
    }

    public Map<String, CategoryConfig> getFilter() {
        return filter;
    }
}

public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();

    public String getLabel() {
        return label;
    }

    public Map<String, FieldConfig> getFields() {
        return fields;
    }
}

public class MeasureConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();

    public String getLabel() {
        return label;
    }

    public Map<String, FieldConfig> getFields() {
        return fields;
    }
}

public class FieldConfig {
    private String label;
    private Map<String, String> metric = new HashMap<>();
    private JoinTargets joins;

    public String getLabel() {
        return label;
    }

    public Map<String, String> getMetric() {
        return metric;
    }

    public JoinTargets getJoins() {
        return joins;
    }
}

public class JoinTargets {
    private List<JoinTarget> measure = new ArrayList<>();
    private List<JoinTarget> dimension = new ArrayList<>();
    private List<JoinTarget> filter = new ArrayList<>();

    public List<JoinTarget> getMeasure() {
        return measure;
    }

    public List<JoinTarget> getDimension() {
        return dimension;
    }

    public List<JoinTarget> getFilter() {
        return filter;
    }
}

public class JoinTarget {
    private String target;
    private String label;

    public String getTarget() {
        return target;
    }

    public String getLabel() {
        return label;
    }
}


// === Join 그래프 유틸 ===

public class JoinGraphSimpleUtil {

    public static Map<String, Set<String>> buildJoinGraph(DdeMetadataProperties dde) {
        Map<String, Set<String>> graph = new HashMap<>();

        buildFromCategoryMap(dde.getDimension(), graph);
        buildFromCategoryMap(dde.getFilter(), graph);

        Map<String, MeasureConfig> measures = dde.getMeasure();
        for (String categoryName : measures.keySet()) {
            MeasureConfig config = measures.get(categoryName);
            if (config.getFields() == null) continue;

            for (String fieldName : config.getFields().keySet()) {
                FieldConfig field = config.getFields().get(fieldName);
                if (field.getJoins() == null) continue;
                addJoinsToGraph(fieldName, field.getJoins(), graph);
            }
        }

        return graph;
    }

    private static void buildFromCategoryMap(Map<String, CategoryConfig> map, Map<String, Set<String>> graph) {
        for (String categoryName : map.keySet()) {
            CategoryConfig config = map.get(categoryName);
            if (config.getFields() == null) continue;

            for (String fieldName : config.getFields().keySet()) {
                FieldConfig field = config.getFields().get(fieldName);
                if (field.getJoins() == null) continue;
                addJoinsToGraph(fieldName, field.getJoins(), graph);
            }
        }
    }

    private static void addJoinsToGraph(String source, JoinTargets joins, Map<String, Set<String>> graph) {
        for (JoinTarget jt : joins.getMeasure()) {
            graph.computeIfAbsent(source, new Function<String, Set<String>>() {
                public Set<String> apply(String k) {
                    return new HashSet<>();
                }
            }).add(jt.getTarget());
        }
        for (JoinTarget jt : joins.getDimension()) {
            graph.computeIfAbsent(source, new Function<String, Set<String>>() {
                public Set<String> apply(String k) {
                    return new HashSet<>();
                }
            }).add(jt.getTarget());
        }
        for (JoinTarget jt : joins.getFilter()) {
            graph.computeIfAbsent(source, new Function<String, Set<String>>() {
                public Set<String> apply(String k) {
                    return new HashSet<>();
                }
            }).add(jt.getTarget());
        }
    }

    public static boolean isJoinable(String from, String to, Map<String, Set<String>> graph) {
        Set<String> visited = new HashSet<>();
        Queue<String> queue = new LinkedList<>();
        queue.add(from);
        visited.add(from);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (current.equals(to)) return true;
            for (String neighbor : graph.getOrDefault(current, new HashSet<>())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.add(neighbor);
                }
            }
        }
        return false;
    }

    public static Set<String> recommendJoinTargets(String from, Map<String, Set<String>> graph) {
        return graph.getOrDefault(from, new HashSet<>());
    }

    public static List<String> recommendJoinLabels(String from, Map<String, Set<String>> graph, DdeMetadataProperties dde) {
        Set<String> targets = recommendJoinTargets(from, graph);
        List<String> labels = new ArrayList<>();
        for (String target : targets) {
            String label = findLabel(target, dde);
            if (label != null) {
                labels.add(label);
            }
        }
        return labels;
    }

    public static List<String> recommendJoinKeys(String from, Map<String, Set<String>> graph) {
        Set<String> targets = recommendJoinTargets(from, graph);
        return new ArrayList<>(targets);
    }

    private static String findLabel(String key, DdeMetadataProperties dde) {
        if (dde.getMeasure().containsKey(key)) {
            return dde.getMeasure().get(key).getLabel();
        }
        if (dde.getDimension().containsKey(key)) {
            return dde.getDimension().get(key).getLabel();
        }
        if (dde.getFilter().containsKey(key)) {
            return dde.getFilter().get(key).getLabel();
        }
        for (CategoryConfig cat : dde.getDimension().values()) {
            if (cat.getFields().containsKey(key)) {
                return cat.getFields().get(key).getLabel();
            }
        }
        for (CategoryConfig cat : dde.getFilter().values()) {
            if (cat.getFields().containsKey(key)) {
                return cat.getFields().get(key).getLabel();
            }
        }
        for (MeasureConfig cat : dde.getMeasure().values()) {
            if (cat.getFields().containsKey(key)) {
                return cat.getFields().get(key).getLabel();
            }
        }
        return null;
    }
}


```

---

## 🎯 목표:

`measure`, `dimension`, `filter` 세 가지 유형의 엔티티에 대해
**조인 가능성 여부**를 각각 명시하고, 예시로 `background_usage`가

* 다른 `measure`: `foreground_usage`
* `dimension`: `app_name`
* `filter`: `device_group`, `usage_date`

과 조인 가능함을 표현할 수 있어야 합니다.

---

## ✅ 1. YAML 구조 설계 (구분된 join 대상 타입)

```yaml
dde:
  measure:
    background_usage:
      label: "Background Usage"
      fields:
        device_id:
          label: "Device ID"
        usage_time:
          label: "Usage Time"
      joins:
        measures:
          - target: foreground_usage
            on:
              device_id: device_id
        dimensions:
          - target: app
            on:
              package_name: package_name
        filters:
          - target: device_group
            on:
              group_id: group_id
          - target: usage_date
            on:
              date: date

    foreground_usage:
      label: "Foreground Usage"
      fields:
        device_id:
          label: "Device ID"
        usage_time:
          label: "Usage Time"

  dimension:
    app:
      label: "App Name"
      fields:
        package_name:
          label: "Package Name"
        app_name:
          label: "App Name"

  filter:
    device_group:
      label: "Device Group"
      fields:
        group_id:
          label: "Group ID"

    usage_date:
      label: "Usage Date"
      fields:
        date:
          label: "Date"
```

---

## ✅ 2. Java 클래스 설계 (확장된 JoinConfig 포함)

```java
@Data
public class MeasureConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();
    private JoinGroups joins = new JoinGroups(); // NEW
}

@Data
public class JoinGroups {
    private List<JoinConfig> measures = new ArrayList<>();
    private List<JoinConfig> dimensions = new ArrayList<>();
    private List<JoinConfig> filters = new ArrayList<>();
}

@Data
public class JoinConfig {
    private String target;
    private Map<String, String> on = new HashMap<>();
}
```

→ `dde.measure.get("background_usage").getJoins().getFilters()` 로 접근 가능

---

## ✅ 3. 그래프 탐색 구성

```java
public Map<String, Set<String>> buildJoinGraphByType(
    Map<String, MeasureConfig> measures,
    Function<JoinGroups, List<JoinConfig>> joinSelector
) {
    Map<String, Set<String>> graph = new HashMap<>();

    for (Map.Entry<String, MeasureConfig> entry : measures.entrySet()) {
        String from = entry.getKey();
        List<JoinConfig> joins = joinSelector.apply(entry.getValue().getJoins());
        for (JoinConfig join : joins) {
            String to = join.getTarget();
            graph.computeIfAbsent(from, k -> new HashSet<>()).add(to);
            graph.computeIfAbsent(to, k -> new HashSet<>()).add(from);
        }
    }
    return graph;
}
```

사용 예:

```java
Map<String, Set<String>> measureGraph = buildJoinGraphByType(measures, JoinGroups::getMeasures);
Map<String, Set<String>> dimensionGraph = buildJoinGraphByType(measures, JoinGroups::getDimensions);
Map<String, Set<String>> filterGraph = buildJoinGraphByType(measures, JoinGroups::getFilters);
```

---

## ✅ 4. 장점

| 항목       | 설명                                                   |
| -------- | ---------------------------------------------------- |
| 관계 대상 구분 | `measure`, `dimension`, `filter` 별로 분리               |
| 탐색 유연성   | 필요한 관계 타입만 그래프로 구성 가능                                |
| YAML 관리  | Git 기반으로 선언적 버전 관리 가능                                |
| 향후 확장    | `join_type`, `optional`, `join_strength` 등도 쉽게 확장 가능 |

---

## ✅ 예시 시나리오 정리

> `background_usage`는 다음과 같은 조인 관계를 갖는다:

* **(measure)** `foreground_usage` — `device_id` 기준
* **(dimension)** `app` — `package_name` 기준
* **(filter)** `device_group`, `usage_date`

모두 YAML로 선언 가능하고, Java 로직에서 안전하게 분리 탐색 가능.

---

필요하시면 이 구조 전체 YAML 샘플 + Java 클래스 + 탐색 유틸을 코드로 묶어서 제공해드릴게요. 바로 원하시나요?

```java
// === application.yml 예시 ===
dde:
  measure:
    background_usage:
      label: "Background Usage"
      fields:
        device_id:
          label: "Device ID"
        usage_time:
          label: "Usage Time"
      joins:
        measures:
          - target: foreground_usage
            on:
              device_id: device_id
        dimensions:
          - target: app
            on:
              package_name: package_name
        filters:
          - target: device_group
            on:
              group_id: group_id
          - target: usage_date
            on:
              date: date

    foreground_usage:
      label: "Foreground Usage"
      fields:
        device_id:
          label: "Device ID"
        usage_time:
          label: "Usage Time"

  dimension:
    app:
      label: "App Name"
      fields:
        package_name:
          label: "Package Name"
        app_name:
          label: "App Name"

  filter:
    device_group:
      label: "Device Group"
      fields:
        group_id:
          label: "Group ID"

    usage_date:
      label: "Usage Date"
      fields:
        date:
          label: "Date"


// === Java 클래스 구조 ===

@Data
public class DdeMetadataProperties {
    private Map<String, MeasureConfig> measure = new HashMap<>();
    private Map<String, CategoryConfig> dimension = new HashMap<>();
    private Map<String, CategoryConfig> filter = new HashMap<>();
}

@Data
public class MeasureConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();
    private JoinGroups joins = new JoinGroups();
}

@Data
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();
}

@Data
public class FieldConfig {
    private String label;
    private Map<String, String> metric = new HashMap<>();
}

@Data
public class JoinGroups {
    private List<JoinConfig> measures = new ArrayList<>();
    private List<JoinConfig> dimensions = new ArrayList<>();
    private List<JoinConfig> filters = new ArrayList<>();
}

@Data
public class JoinConfig {
    private String target;
    private Map<String, String> on = new HashMap<>();
}


// === Join 그래프 생성 유틸 ===

public class JoinGraphBuilder {
    public static Map<String, Set<String>> buildJoinGraphByType(
            Map<String, MeasureConfig> measures,
            Function<JoinGroups, List<JoinConfig>> joinSelector) {

        Map<String, Set<String>> graph = new HashMap<>();

        for (Map.Entry<String, MeasureConfig> entry : measures.entrySet()) {
            String source = entry.getKey();
            List<JoinConfig> joins = joinSelector.apply(entry.getValue().getJoins());
            for (JoinConfig join : joins) {
                String target = join.getTarget();
                graph.computeIfAbsent(source, k -> new HashSet<>()).add(target);
                graph.computeIfAbsent(target, k -> new HashSet<>()).add(source);
            }
        }

        return graph;
    }
}

-----+options------

// === application.yml 예시 ===
dde:
  measure:
    background_usage:
      label: "Background Usage"
      fields:
        device_id:
          label: "Device ID"
        usage_time:
          label: "Usage Time"
      joins:
        measures:
          - target: foreground_usage
            on:
              device_id: device_id
            join_type: inner
            optional: false
            direction: bidirectional
        dimensions:
          - target: app
            on:
              package_name: package_name
            join_type: left
            optional: true
            direction: unidirectional
        filters:
          - target: device_group
            on:
              group_id: group_id
            join_type: inner
            optional: false
            direction: unidirectional
          - target: usage_date
            on:
              date: date
            join_type: inner
            optional: true
            direction: unidirectional

    foreground_usage:
      label: "Foreground Usage"
      fields:
        device_id:
          label: "Device ID"
        usage_time:
          label: "Usage Time"

  dimension:
    app:
      label: "App Name"
      fields:
        package_name:
          label: "Package Name"
        app_name:
          label: "App Name"

  filter:
    device_group:
      label: "Device Group"
      fields:
        group_id:
          label: "Group ID"

    usage_date:
      label: "Usage Date"
      fields:
        date:
          label: "Date"


// === Java 클래스 구조 ===

@Data
public class DdeMetadataProperties {
    private Map<String, MeasureConfig> measure = new HashMap<>();
    private Map<String, CategoryConfig> dimension = new HashMap<>();
    private Map<String, CategoryConfig> filter = new HashMap<>();
}

@Data
public class MeasureConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();
    private JoinGroups joins = new JoinGroups();
}

@Data
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();
}

@Data
public class FieldConfig {
    private String label;
    private Map<String, String> metric = new HashMap<>();
}

@Data
public class JoinGroups {
    private List<JoinConfig> measures = new ArrayList<>();
    private List<JoinConfig> dimensions = new ArrayList<>();
    private List<JoinConfig> filters = new ArrayList<>();
}

@Data
public class JoinConfig {
    private String target;
    private Map<String, String> on = new HashMap<>();
    private String join_type; // inner, left, etc.
    private Boolean optional = false; // true if optional
    private String direction = "bidirectional"; // or unidirectional
}


// === Join 그래프 생성 유틸 ===

public class JoinGraphBuilder {
    public static Map<String, Set<String>> buildJoinGraphByType(
            Map<String, MeasureConfig> measures,
            Function<JoinGroups, List<JoinConfig>> joinSelector) {

        Map<String, Set<String>> graph = new HashMap<>();

        for (Map.Entry<String, MeasureConfig> entry : measures.entrySet()) {
            String source = entry.getKey();
            List<JoinConfig> joins = joinSelector.apply(entry.getValue().getJoins());
            for (JoinConfig join : joins) {
                String target = join.getTarget();
                graph.computeIfAbsent(source, k -> new HashSet<>()).add(target);
                if (!"unidirectional".equalsIgnoreCase(join.getDirection())) {
                    graph.computeIfAbsent(target, k -> new HashSet<>()).add(source);
                }
            }
        }

        return graph;
    }
}


    // === 조인 유효성 검사 ===
    public static boolean isJoinable(String from, String to, Map<String, Set<String>> graph) {
        Set<String> visited = new HashSet<>();
        Queue<String> queue = new LinkedList<>();
        queue.add(from);
        visited.add(from);

        while (!queue.isEmpty()) {
            String current = queue.poll();
            if (current.equals(to)) return true;
            for (String neighbor : graph.getOrDefault(current, Set.of())) {
                if (!visited.contains(neighbor)) {
                    visited.add(neighbor);
                    queue.add(neighbor);
                }
            }
        }
        return false;
    }

    // === 조인 가능한 대상 추천 ===
    public static Set<String> suggestJoinableTargets(String from, Map<String, Set<String>> graph) {
        return graph.getOrDefault(from, Set.of());
    }

```



--------------------

좋습니다. 아래에 전체 구성 코드를 다시 정리해서 올려드립니다. 다음 구조를 기준으로 합니다:

---

### ✅ `application.yml` 구조 예시

```yaml
dde:
  dimension:
    device:
      label: "Device Name"
      fields:
        device_model:
          label: "Device Model"
        device_id:
          label: "Device ID"
          metric:
            sum: "Sum (EA)"
        group_id:
          label: "Group ID"
  measure:
    package:
      label: "Package Name"
      fields:
        app_name:
          label: "App Name"
        package_name:
          label: "Package Name"
        app_version:
          label: "App Version"
        app_usage:
          label: "App Usage"
          metric:
            sum: "Sum (EA)"
            avg: "Avg (%)"
```

---

### ✅ Java Config 클래스: `DdeMetadataProperties`

```java
@Component
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {

    private Map<String, Map<String, CategoryConfig>> dimension = new HashMap<>();
    private Map<String, Map<String, CategoryConfig>> measure = new HashMap<>();

    public Map<String, Map<String, CategoryConfig>> getDimension() {
        return dimension;
    }

    public void setDimension(Map<String, Map<String, CategoryConfig>> dimension) {
        this.dimension = dimension;
    }

    public Map<String, Map<String, CategoryConfig>> getMeasure() {
        return measure;
    }

    public void setMeasure(Map<String, Map<String, CategoryConfig>> measure) {
        this.measure = measure;
    }

    // Get group map based on group name
    private Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension.getOrDefault("device", Map.of());
            case "measure" -> measure.getOrDefault("package", Map.of());
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    // 🔹 Get category labels by group (dimension/measure)
    public List<String> getCategoryLabels(String group) {
        Map<String, Map<String, CategoryConfig>> groupMap = switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
        return groupMap.values().stream()
                .flatMap(map -> map.values().stream())
                .map(CategoryConfig::getLabel)
                .toList();
    }

    // 🔹 Get category key by label
    public String getCategoryKeyByLabel(String group, String label) {
        return getGroupMap(group).entrySet().stream()
                .filter(entry -> entry.getValue().getLabel().equalsIgnoreCase(label))
                .map(Map.Entry::getKey)
                .findFirst()
                .orElse(null);
    }

    // 🔹 Get field name by label
    public String getFieldName(String group, String category, String label) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null) return null;
        return config.getFields().entrySet().stream()
                .filter(entry -> label.equalsIgnoreCase(entry.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst()
                .orElse(null);
    }

    // 🔹 Get field label by name
    public String getFieldLabel(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        FieldConfig field = config.getFields().get(fieldName);
        return field != null ? field.getLabel() : null;
    }

    // 🔹 Get all field labels by group and category
    public Map<String, String> getFieldLabels(String group, String category) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return Map.of();
        return config.getFields().entrySet().stream()
                .collect(Collectors.toMap(Map.Entry::getKey, e -> e.getValue().getLabel()));
    }

    // 🔹 Get field's metric info
    public Map<String, String> getFieldMetrics(String group, String category, String fieldName) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null) return Map.of();
        FieldConfig field = config.getFields().get(fieldName);
        return field != null ? field.getMetric() : Map.of();
    }
}
```

---

### ✅ CategoryConfig 클래스

```java
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();

    public String getLabel() {
        return label;
    }

    public void setLabel(String label) {
        this.label = label;
    }

    public Map<String, FieldConfig> getFields() {
        return fields;
    }

    public void setFields(Map<String, FieldConfig> fields) {
        this.fields = fields;
    }
}
```

---

### ✅ FieldConfig 클래스

```java
public class FieldConfig {
    private String label;
    private Map<String, String> metric = new HashMap<>();

    public String getLabel() {
        return label;
    }

    public void setLabel(String label) {
        this.label = label;
    }

    public Map<String, String> getMetric() {
        return metric;
    }

    public void setMetric(Map<String, String> metric) {
        this.metric = metric;
    }
}
```

---

### ⚠️ 주의사항

* `@ConfigurationPropertiesScan` 또는 `@EnableConfigurationProperties(DdeMetadataProperties.class)` 설정 필요.
* yml 들여쓰기/문법 정확히 맞춰야 하며, `fields` 밑에 `label`, `metric` 꼭 중첩되어야 합니다.

---

이제 이 구성으로 `field name → label`, `label → field`, `metric`, `group/category label` 검색까지 다 됩니다.
테스트용 예제 호출을 원하시나요?


-----------------------

전체 구조를 반영한 설정 코드를 정리해서 위에 구성했습니다. 이 설정은 다음 기능을 포함합니다:

* 카테고리 라벨 리스트 조회 (`getCategoryLabels`)
* 카테고리 라벨로 키 찾기 (`getCategoryByLabel`)
* 필드 라벨로 필드명 찾기 (`getFieldNameByLabel`)
* 필드명으로 라벨 찾기 (`getFieldLabel`)
* 필드 목록 조회 (`getFields`)
* 그룹 없이 라벨로 검색하기 (`findCategoryByLabelAcrossGroups`, `findFieldNameAcrossGroups`)
* 필드 메트릭 정보 포함 조회 (`getFieldConfig`)

원하는 사용 방식과 유연성을 갖추도록 설계되어 있습니다. 필요하면 JSON 응답 포맷을 만드는 메소드도 추가할 수 있어요. 출력 구조도 만들어 드릴까요?

```java
    
public class DdeMetadataProperties {

    private Map<String, Map<String, CategoryConfig>> dimension = new HashMap<>();
    private Map<String, Map<String, CategoryConfig>> measure = new HashMap<>();

    public Map<String, Map<String, CategoryConfig>> getDimension() {
        return dimension;
    }

    public Map<String, Map<String, CategoryConfig>> getMeasure() {
        return measure;
    }

    private Map<String, Map<String, CategoryConfig>> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    // 1. getCategoryLabels
    public List<String> getCategoryLabels(String group) {
        return getGroupMap(group).values().stream()
            .map(CategoryConfig::getLabel)
            .toList();
    }

    // 2. getCategoryByLabel
    public String getCategoryByLabel(String group, String label) {
        return getGroupMap(group).entrySet().stream()
            .filter(entry -> label.equals(entry.getValue().getLabel()))
            .map(Map.Entry::getKey)
            .findFirst()
            .orElse(null);
    }

    // 3. getFieldNameByLabel
    public String getFieldNameByLabel(String group, String category, String label) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null) return null;
        return config.getFields().entrySet().stream()
            .filter(entry -> label.equals(entry.getValue().getLabel()))
            .map(Map.Entry::getKey)
            .findFirst()
            .orElse(null);
    }

    // 4. getFieldLabel
    public String getFieldLabel(String group, String category, String field) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        FieldConfig fieldConfig = config.getFields().get(field);
        return fieldConfig != null ? fieldConfig.getLabel() : null;
    }

    // 5. getFields
    public List<String> getFields(String group, String category) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null) return List.of();
        return new ArrayList<>(config.getFields().keySet());
    }

    // 6. findCategoryByLabelAcrossGroups
    public String findCategoryByLabelAcrossGroups(String label) {
        for (String group : List.of("dimension", "measure")) {
            String result = getCategoryByLabel(group, label);
            if (result != null) return result;
        }
        return null;
    }

    // 7. findFieldNameAcrossGroups
    public String findFieldNameAcrossGroups(String label) {
        for (String group : List.of("dimension", "measure")) {
            Map<String, Map<String, CategoryConfig>> groupMap = getGroupMap(group);
            for (Map.Entry<String, CategoryConfig> entry : groupMap.entrySet()) {
                String category = entry.getKey();
                CategoryConfig config = entry.getValue();
                String field = config.getFields().entrySet().stream()
                    .filter(e -> label.equals(e.getValue().getLabel()))
                    .map(Map.Entry::getKey)
                    .findFirst()
                    .orElse(null);
                if (field != null) return field;
            }
        }
        return null;
    }

    // 8. getFieldConfig
    public FieldConfig getFieldConfig(String group, String category, String field) {
        CategoryConfig config = getGroupMap(group).get(category);
        if (config == null || config.getFields() == null) return null;
        return config.getFields().get(field);
    }

    // 9. getFieldMetricByLabel
    public Map<String, String> getFieldMetricByLabel(String group, String category, String label) {
        String field = getFieldNameByLabel(group, category, label);
        FieldConfig config = getFieldConfig(group, category, field);
        return config != null ? config.getMetric() : Map.of();
    }
}

    
        

```


-------------------

지금 구조처럼 `metric` 정보를 `field`별로 추가한 YAML 설정에서, 특정 필드에 대한 메트릭 정보도 함께 조회하고 싶다면, 우선 Java 클래스 구조를 이에 맞게 수정한 뒤, 메소드로 조회할 수 있도록 만들어야 합니다.

---

### ✅ YAML 구조 예

```yaml
dde:
  dimension:
    device:
      label: "Device Name"
      fields:
        device_model: "Device Model"
        device_id:
          label: "Device ID"
          metric:
            sum: "Sum (EA)"
        group_id: "Group ID"

  measure:
    package:
      label: "Package Name"
      fields:
        app_name: "App Name"
        package_name: "Package Name"
        app_version: "App Version"
        app_usage:
          label: "App Usage"
          metric:
            sum: "Sum (EA)"
            avg: "Avg (%)"
```

---

### ✅ Java 클래스 구조

```java
@Data
public class DDEMetadataProperties {

    private Map<String, Map<String, CategoryConfig>> dde = new HashMap<>();

    public Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dde.getOrDefault("dimension", Map.of());
            case "measure" -> dde.getOrDefault("measure", Map.of());
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    public Optional<FieldConfig> getFieldConfig(String group, String category, String field) {
        return Optional.ofNullable(getGroupMap(group).get(category))
                .map(CategoryConfig::getFields)
                .map(m -> m.get(field));
    }
}

@Data
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();
}

@Data
public class FieldConfig {
    private String label; // optional if simple string
    private Map<String, String> metric; // sum, avg, etc.
}
```

---

### ✅ `@ConfigurationProperties` 바인딩 방식

```java
@Component
@ConfigurationProperties(prefix = "dde")
public class DDEMetadataProperties {
    private Map<String, Map<String, CategoryConfig>> dimension = new HashMap<>();
    private Map<String, Map<String, CategoryConfig>> measure = new HashMap<>();
}
```

---

### ✅ 메소드 예시

```java
public Map<String, String> getMetrics(String group, String category, String field) {
    return getFieldConfig(group, category, field)
            .map(FieldConfig::getMetric)
            .orElse(Map.of());
}
```

---

### ✅ 사용 예

```java
Map<String, String> metrics = ddeMetadataProperties.getMetrics("measure", "package", "app_usage");
// 출력: {sum=Sum (EA), avg=Avg (%)}
```

---

필드 값이 단순한 문자열인 경우(`device_model: "Device Model"`), 바인딩 시 `FieldConfig`로 간주되도록 `CustomDeserializer`를 사용할 수도 있습니다. 그게 필요하면 알려주세요.

JSON/YAML 자동 바인딩을 더 유연하게 처리할 수도 있어요. 필요하신가요?


-----------
group 없이 `"Device Name"` 같은 label을 검색해서 해당하는 `category key`를 찾고 싶다면, **dimension + measure**를 모두 뒤져서 일치하는 label을 찾는 로직이 필요합니다.

---

### ✅ `getCategoryKeyByLabel(String label)` (group 없이)

```java
public String getCategoryKeyByLabel(String label) {
    // 먼저 dimension에서 찾기
    Optional<String> fromDimension = dimension.entrySet().stream()
            .filter(entry -> label.equals(entry.getValue().getLabel()))
            .map(Map.Entry::getKey)
            .findFirst();

    if (fromDimension.isPresent()) {
        return fromDimension.get();
    }

    // 그다음 measure에서 찾기
    return measure.entrySet().stream()
            .filter(entry -> label.equals(entry.getValue().getLabel()))
            .map(Map.Entry::getKey)
            .findFirst()
            .orElseThrow(() -> new IllegalArgumentException("Label not found: " + label));
}
```

---

### ✅ 예시

```java
String key = ddeMetadataProperties.getCategoryKeyByLabel("Device Name");
// 결과: "device"
```

---

label이 중복되지 않는다는 전제에서 동작하며,
만약 dimension과 measure에 동일한 label이 존재한다면, dimension 쪽이 우선됩니다.

label 중복 가능성도 고려해야 할까요?


---------------

전체적으로 요구하신 구조에 맞춰 YAML, Java 클래스, 메서드 구조를 **정리**해서 제공드립니다.

---

## ✅ 1. `application.yml` 또는 `dde-metadata.yml`

```yaml
dde:
  dimension:
    device:
      label: "Device Name"
      fields:
        device_model: "Device Model"
        device_id: "Device ID"
        group_id: "Group ID"

  measure:
    package:
      label: "Package Name"
      fields:
        app_name: "App Name"
        package_name: "Package Name"
        app_version: "App Version"
```

---

## ✅ 2. `DdeMetadataProperties.java`

```java
@Component
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {

    private Map<String, CategoryConfig> dimension = new LinkedHashMap<>();
    private Map<String, CategoryConfig> measure = new LinkedHashMap<>();

    @Getter @Setter
    public static class CategoryConfig {
        private String label;
        private Map<String, String> fields = new LinkedHashMap<>();
    }

    // getters & setters
    public Map<String, CategoryConfig> getDimension() {
        return dimension;
    }

    public void setDimension(Map<String, CategoryConfig> dimension) {
        this.dimension = dimension;
    }

    public Map<String, CategoryConfig> getMeasure() {
        return measure;
    }

    public void setMeasure(Map<String, CategoryConfig> measure) {
        this.measure = measure;
    }

    // ✅ 1. get category label list by group
    public List<String> getCategoryLabels(String group) {
        Map<String, CategoryConfig> target = getGroupMap(group);
        return target.values().stream()
                .map(CategoryConfig::getLabel)
                .toList();
    }

    // ✅ 2. get category name by label
    public Optional<String> getCategoryNameByLabel(String group, String label) {
        Map<String, CategoryConfig> target = getGroupMap(group);
        return target.entrySet().stream()
                .filter(e -> label.equalsIgnoreCase(e.getValue().getLabel()))
                .map(Map.Entry::getKey)
                .findFirst();
    }

    // ✅ 3. get field name and label list across all groups
    public Map<String, String> getAllFieldNameToLabelMap() {
        Map<String, String> result = new LinkedHashMap<>();
        Stream.of(dimension, measure).forEach(map ->
            map.forEach((category, config) ->
                config.getFields().forEach((k, v) -> result.put(k, v))
            )
        );
        return result;
    }

    // 🔧 group helper
    private Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }
}
```

---

## ✅ 3. 사용 예시

```java
// 1. categoryLabels by group
List<String> labels = ddeMetadataProperties.getCategoryLabels("dimension");
// 결과: ["Device Name"]

// 2. categoryLabelSearch with label
Optional<String> category = ddeMetadataProperties.getCategoryNameByLabel("dimension", "Device Name");
// 결과: Optional["device"]

// 3. fieldsSearch with name
Map<String, String> fieldMap = ddeMetadataProperties.getAllFieldNameToLabelMap();
String appLabel = fieldMap.get("app_name"); 
// 결과: "App Name"
```

---

좋습니다! 아래는 요청하신 메서드 `getFieldLabel(group, category, inputLabel)`을 추가하는 방법입니다.

---

### ✅ 1. 메서드 시그니처

```java
public Optional<String> getFieldKeyByLabel(String group, String category, String inputLabel)
```

즉, `"App Name"`이라는 *label*을 넣으면 해당 group과 category 내에서 `"app_name"`이라는 *key*를 찾아주는 기능입니다.

---

### ✅ 2. `DdeMetadataProperties`에 메서드 추가

```java
public Optional<String> getFieldKeyByLabel(String group, String category, String inputLabel) {
    Map<String, CategoryConfig> target = getGroupMap(group);
    CategoryConfig config = target.get(category);
    if (config == null || config.getFields() == null) {
        return Optional.empty();
    }
    return config.getFields().entrySet().stream()
            .filter(e -> e.getValue().equalsIgnoreCase(inputLabel))
            .map(Map.Entry::getKey)
            .findFirst();
}
```

---

### ✅ 3. 사용 예시

```java
Optional<String> fieldKey = ddeMetadataProperties.getFieldKeyByLabel("measure", "package", "App Name");
fieldKey.ifPresent(System.out::println);  // 출력: app_name
```

---

### 🔄 참고: 전체 구조 복사용 요약

* `group`: "dimension" or "measure"
* `category`: 예: "device", "package"
* `label`: 예: "App Name"
* 결과: 해당 label에 매핑된 key (예: `"app_name"`)

---

필요하시면 반대로 `getFieldLabelByKey(group, category, "app_name")`도 만들어드릴 수 있어요. 추가로 필요하신 기능 있을까요?


좋습니다!
`field list by group and category`는 다음처럼 \*\*group(dimension/measure)\*\*와 \*\*category(device/package 등)\*\*를 기준으로 해당 필드 목록 전체를 반환하는 기능입니다.

---

### ✅ 메서드 시그니처

```java
public Map<String, String> getFieldsByGroupAndCategory(String group, String category)
```

---

### ✅ 구현 예시 (`DdeMetadataProperties` 클래스 내부)

```java
public Map<String, String> getFieldsByGroupAndCategory(String group, String category) {
    Map<String, CategoryConfig> target = getGroupMap(group);
    CategoryConfig config = target.get(category);
    if (config == null || config.getFields() == null) {
        return Map.of(); // 빈 맵 반환
    }
    return config.getFields();
}
```

---

### ✅ 사용 예

```java
Map<String, String> fields = ddeMetadataProperties.getFieldsByGroupAndCategory("dimension", "device");
fields.forEach((key, value) -> System.out.println(key + " -> " + value));
```

출력 예시:

```
device_model -> Device Model
device_id -> Device ID
group_id -> Group ID
```

---
네, 가능합니다. 말씀하신 형태는 다음과 같이 **group → categoryLabel → fields** 구조의 `Map<String, Map<String, Map<String, String>>>` 형태로 반환하면 됩니다.

---

### ✅ 목표 형태

```yaml
dimension:
  "Device Name":
    device_model: "Device Model"
    device_id: "Device ID"
    group_id: "Group ID"
  "Another Label":
    ...
```

---

### ✅ Java 메서드 예시

```java
public Map<String, Map<String, String>> getLabeledFieldsByGroup(String group) {
    Map<String, CategoryConfig> target = getGroupMap(group);

    return target.values().stream()
        .collect(Collectors.toMap(
            CategoryConfig::getLabel,     // label을 key로
            CategoryConfig::getFields     // 해당 label의 fields 맵
        ));
}
```

---

### ✅ 사용 예시

```java
Map<String, Map<String, String>> dimensionFields = ddeMetadataProperties.getLabeledFieldsByGroup("dimension");

dimensionFields.forEach((label, fields) -> {
    System.out.println(label + ":");
    fields.forEach((field, desc) -> System.out.println("  " + field + " -> " + desc));
});
```

---

### 📌 주의점

* `label`이 중복되면 `Map`에서 `key` 충돌이 생기므로, `label`은 그룹 내에서 고유해야 합니다.
* 만약 label 중복 가능성이 있다면 key를 `"label (category)"` 형태로 구성하는 것도 고려해볼 수 있습니다.

원하시면 그런 식으로 키 이름을 조합하는 버전도 만들어드릴게요. 원하시나요?


좋습니다. 말씀하신 구조는 다음과 같은 **group → category → {label + fields}** 형태의 계층입니다:

---

### ✅ 목표 JSON/YAML 구조 예시

```yaml
dimension:
  device:
    label: "Device Name"
    device_model: "Device Model"
    device_id: "Device ID"
    group_id: "Group ID"
  anotherCategory:
    label: "Another Label"
    ...
```

---

### ✅ 반환 타입

이 구조는 다음과 같은 타입으로 매핑됩니다:

```java
Map<String, Map<String, Object>>
```

즉, `Map<categoryName, Map<"label"/fieldName, value>>`

---

### ✅ 메서드 구현

```java
public Map<String, Map<String, Object>> getCategoryDetailMapByGroup(String group) {
    Map<String, CategoryConfig> target = getGroupMap(group);

    return target.entrySet().stream()
        .collect(Collectors.toMap(
            Map.Entry::getKey, // category name
            entry -> {
                CategoryConfig config = entry.getValue();
                Map<String, Object> result = new LinkedHashMap<>();
                result.put("label", config.getLabel());
                result.putAll(config.getFields()); // fieldName -> fieldLabel
                return result;
            }
        ));
}
```

---

### ✅ 사용 예시

```java
Map<String, Map<String, Object>> dimensionConfig = ddeMetadataProperties.getCategoryDetailMapByGroup("dimension");

dimensionConfig.forEach((category, configMap) -> {
    System.out.println("Category: " + category);
    System.out.println("Label: " + configMap.get("label"));
    configMap.forEach((k, v) -> {
        if (!k.equals("label")) {
            System.out.println("  " + k + ": " + v);
        }
    });
});
```

---

### 🔄 요약

| 구조                           | 반환 메서드                                      |
| ---------------------------- | ------------------------------------------- |
| `label → fields`             | `getLabeledFieldsByGroup(String group)`     |
| `category → {label, fields}` | `getCategoryDetailMapByGroup(String group)` |

---

원하는 형태에 맞춰 동적으로 전환도 가능합니다. 두 가지 모두 유지할 수도 있고, 필요 시 하나를 기반으로 다른 형태로 변환해도 됩니다.

혹시 JSON 형태로도 결과 예시가 필요하신가요?



-----------------------------
--------------------------------
-------------------------------

좋습니다! 그럼 `getFieldName(...)`의 반대 역할을 하는 `getLabel(...)` 메서드도 함께 정의해드릴게요.

---

### ✅ 1. 목적

YAML 설정:

```yaml
dde:
  dimension:
    device:
      label: "Device Name"
      fields:
        device_model: "Device Model"
        device_id: "Device ID"
        group_id: "Group ID"

  measure:
    package:
      label: "Package Name"
      fields:
        app_name: "App Name"
        package_name: "Package Name"
        app_version: "App Version"
```

```java
// 조회 예시
String label = ddeMetadataProperties.getLabel("measure", "package", "app_name"); 
// 결과: "App Name"
```

---

### ✅ 2. `getLabel(...)` 메서드

```java
public String getLabel(String group, String category, String fieldName) {
    Map<String, CategoryConfig> groupMap = switch (group.toLowerCase()) {
        case "dimension" -> dimension;
        case "measure" -> measure;
        default -> throw new IllegalArgumentException("Unknown group: " + group);
    };

    CategoryConfig categoryConfig = groupMap.get(category);
    if (categoryConfig == null) {
        throw new IllegalArgumentException("Unknown category: " + category);
    }

    String label = categoryConfig.getFields().get(fieldName);
    if (label == null) {
        throw new IllegalArgumentException("Field '" + fieldName + "' not found in " + group + "/" + category);
    }

    return label;
}
```

---

### ✅ 전체 구조 요약

```java
public class DdeMetadataProperties {
    private Map<String, CategoryConfig> dimension = new LinkedHashMap<>();
    private Map<String, CategoryConfig> measure = new LinkedHashMap<>();

    // getFieldName(...) and getLabel(...) 메서드 포함

    public static class CategoryConfig {
        private String label;
        private Map<String, String> fields = new LinkedHashMap<>();
        // getters & setters
    }

    // Getters & setters for dimension and measure
}
```

---

이제 `필드명 <-> 라벨`을 자유롭게 변환할 수 있습니다.
필요하시면 양방향 매핑 유틸도 따로 만들어드릴 수 있어요.


--------------

네, 가능합니다! 이전 구조와 같이 `group`, `category`, 그리고 사용자가 입력한 `inputName`(예: `"App Name"`)을 기준으로 해당 \*\*필드 이름(fieldName)\*\*을 조회하는 메서드를 만들 수 있습니다.

---

### 🔁 목표

다음과 같은 YAML 구조에서:

```yaml
dde:
  dimension:
    device:
      label: "Device Name"
      fields:
        device_model: "Device Model"
        device_id: "Device ID"
        group_id: "Group ID"

  measure:
    package:
      label: "Package Name"
      fields:
        app_name: "App Name"
        package_name: "Package Name"
        app_version: "App Version"
```

아래와 같은 메서드를 호출해서:

```java
String fieldName = ddeMetadataProperties.getFieldName("measure", "package", "App Name");
```

결과가:

```
"app_name"
```

처럼 나오는 것을 목표로 합니다.

---

### ✅ 메서드 예시

```java
public String getFieldName(String group, String category, String inputLabel) {
    Map<String, CategoryConfig> groupMap = switch (group.toLowerCase()) {
        case "dimension" -> dimension;
        case "measure" -> measure;
        default -> throw new IllegalArgumentException("Unknown group: " + group);
    };

    CategoryConfig categoryConfig = groupMap.get(category);
    if (categoryConfig == null) {
        throw new IllegalArgumentException("Unknown category: " + category);
    }

    return categoryConfig.getFields().entrySet().stream()
        .filter(entry -> entry.getValue().equalsIgnoreCase(inputLabel))
        .map(Map.Entry::getKey)
        .findFirst()
        .orElseThrow(() -> new IllegalArgumentException("Field with label '" + inputLabel + "' not found in " + group + "/" + category));
}
```

---

### 📦 CategoryConfig 클래스 (다시)

```java
public class CategoryConfig {
    private String label;
    private Map<String, String> fields = new LinkedHashMap<>();

    // getters & setters
}
```

---

이 구조를 바탕으로 특정 그룹/카테고리 내에서 라벨을 키로 실제 필드명을 역조회할 수 있습니다.

혹시 반대로 `getLabel(String group, String category, String fieldName)`도 필요할까요?


----------------

좋습니다! 말씀하신 구조는 다음과 같은 YAML을 기준으로 합니다:

```yaml
dde:
  dimension:
    device:
      label: "Device Name"
      fields:
        device_model: "Device Model"
        device_id: "Device ID"
        group_id: "Group ID"

  measure:
    package:
      label: "Package Name"
      fields:
        app_name: "App Name"
        package_name: "Package Name"
        app_version: "App Version"
```

이제 이 구조에 맞춰, 주어진 `"App Name"` 같은 **필드 라벨**로부터 **(group, category, fieldName)** 정보를 찾는 메서드를 작성해드릴게요.

---

### ✅ 반환 구조 예

```java
public record FieldMatch(String group, String category, String field, String label) {}
```

---

### ✅ 메서드 구현 예시

```java
public List<FieldMatch> getFieldMatchesByLabel(String labelToFind) {
    List<FieldMatch> matches = new ArrayList<>();

    Map<String, Map<String, CategoryConfig>> groups = Map.of(
        "dimension", dimension,
        "measure", measure
    );

    for (Map.Entry<String, Map<String, CategoryConfig>> groupEntry : groups.entrySet()) {
        String groupName = groupEntry.getKey();
        Map<String, CategoryConfig> categories = groupEntry.getValue();

        for (Map.Entry<String, CategoryConfig> categoryEntry : categories.entrySet()) {
            String categoryName = categoryEntry.getKey();
            CategoryConfig config = categoryEntry.getValue();
            Map<String, String> fields = config.getFields();

            for (Map.Entry<String, String> fieldEntry : fields.entrySet()) {
                String fieldName = fieldEntry.getKey();
                String label = fieldEntry.getValue();

                if (label.equalsIgnoreCase(labelToFind)) {
                    matches.add(new FieldMatch(groupName, categoryName, fieldName, label));
                }
            }
        }
    }

    return matches;
}
```

---

### ✅ 사용 예

```java
List<FieldMatch> result = ddeMetadataProperties.getFieldMatchesByLabel("App Name");

// 출력 예 (실제 객체):
// [
//   FieldMatch("measure", "package", "app_name", "App Name")
// ]
```

---

### 📦 CategoryConfig 클래스 예시

```java
public class CategoryConfig {
    private String label;
    private Map<String, String> fields = new LinkedHashMap<>();

    // getters & setters
}
```

---

필요하다면 JSON 형태로 반환하거나 DTO로 가공해도 됩니다. 이 방식 괜찮을까요, 혹시 추가적으로 원하는 형태가 있을까요?


------------

좋습니다! 아래는 주어진 YAML 구조를 기반으로, 원하는 정보를 읽어내는 자바 메서드를 정의하는 방식입니다.

---

### ✅ YAML 구조 예시

```yaml
dde:
  dimension:
    device:
      label: "Device Name"
      fields:
        device_model: "Device Model"
        device_id: "Device ID"
        group_id: "Group ID"

  measure:
    package:
      label: "Package Name"
      fields:
        app_name: "App Name"
        package_name: "Package Name"
        app_version: "App Version"
```

---

### ✅ 자바 설정 클래스 (@ConfigurationProperties)

```java
@Component
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {

    private Map<String, Map<String, CategoryConfig>> dimension = new HashMap<>();
    private Map<String, Map<String, CategoryConfig>> measure = new HashMap<>();

    @Data
    public static class CategoryConfig {
        private String label;
        private Map<String, String> fields = new HashMap<>();
    }

    public Map<String, Map<String, CategoryConfig>> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    // ✅ 1. 그룹 안의 category 라벨 목록 가져오기
    public List<String> getCategoryLabels(String group) {
        return getGroupMap(group).values().stream()
            .flatMap(catMap -> catMap.values().stream())
            .map(CategoryConfig::getLabel)
            .toList();
    }

    // ✅ 2. 특정 category 의 label 가져오기
    public String getCategoryLabel(String group, String category) {
        return getGroupMap(group)
            .getOrDefault(category, Map.of())
            .get(category)
            .getLabel();
    }

    // ✅ 3. 특정 category 의 필드-라벨 매핑 가져오기
    public Map<String, String> getFieldLabels(String group, String category) {
        return getGroupMap(group)
            .getOrDefault(category, Map.of())
            .get(category)
            .getFields();
    }
}
```

---

### ✅ 사용 예시

```java
// 예시: dimension 그룹의 모든 category label
List<String> labels = ddeMetadataProperties.getCategoryLabels("dimension");
// → ["Device Name"]

// 예시: dimension 그룹에서 device 카테고리의 label
String deviceLabel = ddeMetadataProperties.getCategoryLabel("dimension", "device");
// → "Device Name"

// 예시: measure 그룹에서 package 카테고리의 필드 라벨 매핑
Map<String, String> fields = ddeMetadataProperties.getFieldLabels("measure", "package");
// → {app_name="App Name", package_name="Package Name", app_version="App Version"}
```

---

좋습니다. `fields: device_model with "Device Model", app_name with "App Name"` 이 예는 다음과 같은 요청이라고 해석할 수 있습니다:

* 여러 그룹(dimension, measure)에 속한 여러 카테고리(device, package 등) 중
* 특정 **필드 이름들**(`device_model`, `app_name`)에 대해
* 해당되는 \*\*라벨(label)\*\*을 가져오고 싶다는 의미입니다.

---

### ✅ 목표: 특정 필드 이름들에 해당하는 라벨(label)을 가져오는 메서드

예를 들어 다음 메서드를 만들 수 있습니다:

```java
/**
 * 모든 group + category를 순회하여
 * 주어진 필드명에 해당하는 라벨들을 찾아 반환
 */
public Map<String, String> getLabelsForFields(Set<String> fieldNames) {
    Map<String, String> result = new LinkedHashMap<>();

    Stream.of(dimension, measure)
        .flatMap(group -> group.values().stream()) // Map<String, CategoryConfig>
        .flatMap(categoryMap -> categoryMap.values().stream()) // CategoryConfig
        .forEach(category -> {
            category.getFields().forEach((field, label) -> {
                if (fieldNames.contains(field)) {
                    result.put(field, label);
                }
            });
        });

    return result;
}
```

---

### ✅ 사용 예시

```java
Set<String> fieldNames = Set.of("device_model", "app_name");
Map<String, String> labels = ddeMetadataProperties.getLabelsForFields(fieldNames);

// 결과:
// {
//   "device_model" -> "Device Model",
//   "app_name"     -> "App Name"
// }
```

---

이렇게 하면 어떤 group/dimension/package에 있는 필드든지 한 번에 해당 라벨을 찾을 수 있습니다.
필요하다면 group, category 정보도 같이 붙여줄 수도 있습니다.

원하시는 출력 포맷이 Map\<String, String> 외에 더 복잡한 구조인가요?


좋아요! 말씀하신 `"App Name"`이라는 **라벨값**으로 역으로 **필드명**을 찾고 싶은 경우, 즉:

```java
Map<String, String> result = getFieldsForLabel("App Name");
```

요청한 라벨에 해당하는 필드명을 찾아주는 메서드를 만들면 됩니다.

---

### ✅ 메서드 구현: 라벨로 필드명 찾기

```java
public Map<String, String> getFieldsForLabel(String labelToFind) {
    Map<String, String> result = new LinkedHashMap<>();

    Stream.of(dimension, measure)
        .flatMap(group -> group.values().stream()) // Map<String, CategoryConfig>
        .flatMap(categoryMap -> categoryMap.entrySet().stream()) // Entry<String, CategoryConfig>
        .forEach(entry -> {
            String categoryName = entry.getKey();
            Map<String, String> fields = entry.getValue().getFields();

            fields.forEach((field, label) -> {
                if (label.equalsIgnoreCase(labelToFind)) {
                    result.put(field, label); // 또는 result.put(categoryName + "." + field, label);
                }
            });
        });

    return result;
}
```

---

### ✅ 사용 예

```java
Map<String, String> fields = ddeMetadataProperties.getFieldsForLabel("App Name");

// 출력 예:
// {
//   "app_name" -> "App Name"
// }
```

---

### 🔁 참고: 전체 구조 다시 요약

`application.yml` 구조가 다음과 같다고 할 때:

```yaml
dde:
  dimension:  
    device:
      device_model: "Device Model"
      device_id: "Device ID"
      group_id: "Group ID"
  measure:
    package:
      app_name: "App Name"
      package_name: "Package Name"
      app_version: "App Version"
```

위 코드로 `"App Name"` → `"app_name"`을 잘 찾을 수 있습니다.

---

필요하면 `"category"` 정보도 함께 반환하는 구조로 바꿔드릴 수 있어요. 그렇게 해드릴까요?

-------------

좋은 방향이에요! 기존 방식에서 `dimension`, `measure`처럼 카테고리를 *세분화된 그룹 안에 묶는 방식*으로 변경하면, 설정 파일 구조가 한 단계 더 깊어지기 때문에 `@ConfigurationProperties`와 서비스 코드도 그에 맞춰 수정해야 합니다.

---

## ✅ 1. application.yml 구조

```yaml
dde:
  field:
    dimension:  
      device:
        device_model: "Device Model"
        device_id: "Device ID"
        group_id: "Group ID"
      package:
        app_name: "App Name"
    measure:
      package:
        app_version: "App Version"
```

---

## ✅ 2. `@ConfigurationProperties` 클래스 변경

```java
@Component
@ConfigurationProperties(prefix = "dde.field")
public class FieldMetadataProperties {

    private Map<String, Map<String, String>> dimension = new HashMap<>();
    private Map<String, Map<String, String>> measure = new HashMap<>();

    public Map<String, Map<String, String>> getDimension() {
        return dimension;
    }

    public void setDimension(Map<String, Map<String, String>> dimension) {
        this.dimension = dimension;
    }

    public Map<String, Map<String, String>> getMeasure() {
        return measure;
    }

    public void setMeasure(Map<String, Map<String, String>> measure) {
        this.measure = measure;
    }

    public Map<String, String> getCategoryFields(String group, String category) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension.getOrDefault(category, Map.of());
            case "measure" -> measure.getOrDefault(category, Map.of());
            default -> Map.of();
        };
    }
}
```

---

## ✅ 3. 서비스 메서드

```java
public Map<String, String> getFieldsByGroupAndCategory(String group, String category) {
    return fieldMetadataProperties.getCategoryFields(group, category);
}
```

---

## ✅ 4. Controller 예시

```java
@GetMapping("/fields/{group}/{category}")
public ResponseEntity<Map<String, String>> getFieldsByGroupAndCategory(
        @PathVariable String group,
        @PathVariable String category) {

    Map<String, String> result = metadataFieldService.getFieldsByGroupAndCategory(group, category);

    if (result.isEmpty()) {
        return ResponseEntity.status(HttpStatus.NOT_FOUND).body(Map.of());
    }

    return ResponseEntity.ok(result);
}
```

---

## ✅ 결과 예시

### 요청:

```
GET /api/metadata/fields/dimension/device
```

### 응답:

```json
{
  "device_model": "Device Model",
  "device_id": "Device ID",
  "group_id": "Group ID"
}
```

---

이 구조는 확장성도 뛰어나고, UI에도 `dimension`, `measure`를 탭처럼 나눠서 쓰기 쉬워서 추천드릴만합니다. 필요하시면 전체 DTO나 유틸도 만들어드릴게요. 더 확장할 계획도 있으신가요?
