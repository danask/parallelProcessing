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
