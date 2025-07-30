
아래는 주어진 YAML 기반 설정(`dde.measure`)에서 `sum_total_run_time`과 같은 metric name을 가지고 관련된 `group`, `category`, `fieldKey`, `label` 등을 역으로 찾아오는 전체 로직입니다. 이미 파싱된 구조에서 `Map<String, FieldConfig>` 형식을 사용하는 것을 전제로 구성되어 있으며, `group`(예: `dde`), `category`(예: `mv_app_usage`), `fieldKey`(예: `totalRunTime`), `label`(예: `Total Run Time`), `metricLabel`(예: `Sum`), `metricName`(예: `sum_total_run_time`) 등을 찾아냅니다.

### ✅ 전제 구조

```java
class FieldConfig {
    String label;
    Map<String, MetricConfig> metric;
    Map<String, String> operator;
    JoinConfig joins;
}

class MetricConfig {
    String label;
    String unit;
    String name;
}
```

### ✅ 전체 검색 로직

```java
public class FieldFinder {

    public static class Result {
        String group;
        String category;
        String fieldKey;
        String fieldLabel;
        String metricKey;
        String metricLabel;
        String metricName;

        @Override
        public String toString() {
            return String.format(
                "group=%s, category=%s, fieldKey=%s, fieldLabel=%s, metricKey=%s, metricLabel=%s, metricName=%s",
                group, category, fieldKey, fieldLabel, metricKey, metricLabel, metricName
            );
        }
    }

    public static Optional<Result> findFieldByMetricName(
        Map<String, Map<String, Map<String, FieldConfig>>> allGroups, String metricNameToFind
    ) {
        for (Map.Entry<String, Map<String, Map<String, FieldConfig>>> groupEntry : allGroups.entrySet()) {
            String group = groupEntry.getKey(); // ex) "dde"
            Map<String, Map<String, FieldConfig>> categories = groupEntry.getValue();

            for (Map.Entry<String, Map<String, FieldConfig>> categoryEntry : categories.entrySet()) {
                String category = categoryEntry.getKey(); // ex) "mv_app_usage"
                Map<String, FieldConfig> fields = categoryEntry.getValue();

                for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
                    String fieldKey = fieldEntry.getKey(); // ex) "totalRunTime"
                    FieldConfig fieldConfig = fieldEntry.getValue();

                    if (fieldConfig.getMetric() != null) {
                        for (Map.Entry<String, MetricConfig> metricEntry : fieldConfig.getMetric().entrySet()) {
                            String metricKey = metricEntry.getKey(); // ex) "sum"
                            MetricConfig metric = metricEntry.getValue();
                            if (metricNameToFind.equals(metric.getName())) {
                                Result result = new Result();
                                result.group = group;
                                result.category = category;
                                result.fieldKey = fieldKey;
                                result.fieldLabel = fieldConfig.getLabel();
                                result.metricKey = metricKey;
                                result.metricLabel = metric.getLabel();
                                result.metricName = metric.getName();
                                return Optional.of(result);
                            }
                        }
                    }
                }
            }
        }
        return Optional.empty();
    }
}
```

### ✅ 사용 예시

```java
Optional<FieldFinder.Result> result = FieldFinder.findFieldByMetricName(allGroups, "sum_total_run_time");

result.ifPresent(System.out::println);
```

---

### ✅ 입력 데이터 구조 예시

`allGroups` 구조는 아래처럼 구성된다고 가정합니다:

```java
Map<String, Map<String, Map<String, FieldConfig>>> allGroups = new HashMap<>();
// group = "dde"
// category = "mv_app_usage"
// fieldKey = "totalRunTime"
```

---

필요하시면 YAML 파싱부터 이 구조로 만드는 코드도 추가해드릴 수 있습니다.
이제 `sum_total_run_time` 같은 metric name을 하나만 가지고도 전체 구조를 역추적해서 찾아낼 수 있습니다.


-----

좋은 질문입니다. 지금 말씀하신 `YamlNode root` 대신에 다른 방식, 예컨대 Jackson의 `JsonNode`, SnakeYAML의 `YamlMapping`, 또는 Spring 환경이라면 `Map<String, Object>` 같은 구조를 사용하는 방식도 당연히 가능합니다. 어떤 걸 쓸 수 있는지는 **전체 YAML 구조 파싱 방식**과 \*\*당신의 목적(예: 탐색, 변경, 추천 로직 등)\*\*에 따라 달라집니다.

---

### 🔍 이전 로직에서 어떤 걸 사용했는지 짚어봅시다

이전 대화에서 YAML 기반 Join Graph 를 구성할 때 보통 아래 중 하나를 사용했습니다:

#### ✅ 1. SnakeYAML + Map 구조

```java
Yaml yaml = new Yaml();
InputStream inputStream = new FileInputStream(new File("your.yml"));
Map<String, Object> data = yaml.load(inputStream);
```

이 경우 `data`는 중첩된 `Map<String, Object>` 구조이며, key 값을 따라 탐색하는 식으로 `sum_total_run_time`에 접근할 수 있습니다.

---

### 🧭 목표: `sum_total_run_time`을 기반으로 아래 값을 얻고 싶다고 하셨습니다

* `group`: `mv_app_usage`
* `category`: `totalRunTime`
* `fieldkey`: `sum`
* `label`: `"Sum"`
* 그 외에 unit, name 등도 가능

---

### ✨ 예시 코드 (기존 방식 Map\<String, Object> 기반 탐색)

```java
public Map<String, Object> findFieldInfoByMetricName(Map<String, Object> yamlRoot, String metricName) {
    Map<String, Object> result = new HashMap<>();

    Map<String, Object> measures = (Map<String, Object>) ((Map<String, Object>) yamlRoot.get("dde")).get("measure");

    for (String groupKey : measures.keySet()) {
        Map<String, Object> group = (Map<String, Object>) measures.get(groupKey);
        Map<String, Object> fields = (Map<String, Object>) group.get("fields");

        for (String categoryKey : fields.keySet()) {
            Map<String, Object> field = (Map<String, Object>) fields.get(categoryKey);
            Map<String, Object> metrics = (Map<String, Object>) field.get("metric");

            for (String fieldKey : metrics.keySet()) {
                Map<String, Object> metric = (Map<String, Object>) metrics.get(fieldKey);

                if (metricName.equals(metric.get("name"))) {
                    result.put("group", groupKey);
                    result.put("category", categoryKey);
                    result.put("fieldkey", fieldKey);
                    result.put("label", metric.get("label"));
                    result.put("unit", metric.get("unit"));
                    result.put("name", metric.get("name"));
                    return result;
                }
            }
        }
    }

    return null;
}
```

---

### 📘 출력 예시

```java
{
  group=mv_app_usage,
  category=totalRunTime,
  fieldkey=sum,
  label=Sum,
  unit=count,
  name=sum_total_run_time
}
```

---

### 🧠 그럼 YamlNode 를 안 쓰는 이유?

`YamlNode`는 Java 17+에서 등장한 SnakeYAML Engine의 객체 모델로, 탐색은 용이하지만:

* 타입 캐스팅이 번거롭고
* Spring 기반 시스템과 호환성 떨어짐
* 많은 프로젝트에서 여전히 Map\<String, Object> 기반 파싱을 선호

따라서 단순 탐색이라면 `Map<String, Object>` 방식이 더 직관적이고 실용적입니다.

---

필요하다면 이걸 바탕으로 추천 시스템, join graph 매핑, 혹은 field 선택 UI에 연동하는 것도 가능합니다.

추가로 DSL 기반으로 만들거나, 성능 고려한 구조화가 필요하면 알려주세요.


----

`sum_total_run_time`과 같은 **metric name**을 가지고 관련된 정보(예: group, category, fieldKey, label 등)를 역으로 찾으려면, 다음 방식으로 YAML 전체를 탐색하는 역매핑(reverse mapping) 로직이 필요합니다.

---

### ✅ 1. 매핑 기준 이해

너가 말한 구성은 대략 다음처럼 구성되어 있어:

* **group**: `mv_app_usage`
* **category** (optional): `totalRunTime`
* **fieldKey**: `totalRunTime`
* **metric name**: `sum_total_run_time`
* **label**: `"Total Run Time"` (field), `"Sum"` (metric)
* **join 정보**: `joins.measure` 등에 위치

---

### ✅ 2. 역탐색 예시 로직

```java
class MetricInfo {
    String group;
    String fieldKey;
    String metricKey;
    String metricName;
    String label;
    String unit;
    // + other info like join target if needed
}
```

### 자바 코드 로직 예시 (Jackson 기준)

```java
public MetricInfo findMetricByName(YamlNode root, String targetMetricName) {
    for (Map.Entry<String, Object> groupEntry : root.get("dde").get("measure").entrySet()) {
        String group = groupEntry.getKey();
        Map<String, Object> fields = (Map<String, Object>) ((Map) groupEntry.getValue()).get("fields");

        for (Map.Entry<String, Object> fieldEntry : fields.entrySet()) {
            String fieldKey = fieldEntry.getKey();
            Map<String, Object> metricObj = (Map<String, Object>) ((Map) fieldEntry.getValue()).get("metric");
            for (Map.Entry<String, Object> metricEntry : metricObj.entrySet()) {
                String metricKey = metricEntry.getKey();
                Map<String, Object> metricInfo = (Map<String, Object>) metricEntry.getValue();
                if (targetMetricName.equals(metricInfo.get("name"))) {
                    MetricInfo info = new MetricInfo();
                    info.group = group;
                    info.fieldKey = fieldKey;
                    info.metricKey = metricKey;
                    info.metricName = (String) metricInfo.get("name");
                    info.label = (String) metricInfo.get("label");
                    info.unit = (String) metricInfo.get("unit");
                    return info;
                }
            }
        }
    }
    return null;
}
```

---

### ✅ 예시 결과 (입력: `sum_total_run_time`)

| 항목         | 값                                        |
| ---------- | ---------------------------------------- |
| group      | `mv_app_usage`                           |
| fieldKey   | `totalRunTime`                           |
| metricKey  | `sum`                                    |
| metricName | `sum_total_run_time`                     |
| label      | `Sum` (metric), `Total Run Time` (field) |
| unit       | `count`                                  |

---

### ✅ 만약 `joins`, `filters`, `operator` 도 필요하다면?

`fieldEntry.getValue()` 안에 포함되어 있는 `joins`, `operator` 등도 같은 방식으로 추출 가능합니다.

---

### ✅ 정리

* `sum_total_run_time`은 **metric name**
* 이 값을 키로 역으로 `group`, `fieldKey`, `label`, `unit`, `metricKey` 등을 찾으려면

  * measure → field → metric → name 으로 내려가서 역매핑
* Java, Python, JavaScript 등에서 **YAML 트리 traversal** 로 해결 가능

필요하면 파싱 도구도 같이 정리해 줄게 (예: SnakeYAML for Java).


----

좋아, 그럼 지금까지 나눈 모든 유효성 검사를 하나로 모아서 `validateSelectedFields(...)` 함수로 통합해줄게.
이 함수는 `getJoinRecommendations(...)` 전에 호출되도록 하면 돼.

---

## ✅ 전체 유효성 체크 메소드: `validateSelectedFields(...)`

```java
private void validateSelectedFields(Set<String> selectedMeasureKeys,
                                     Set<String> selectedDimensionKeys,
                                     Set<String> selectedFilterKeys) {

    // --- 1. Measure 간 조인 가능성 ---
    if (selectedMeasureKeys.size() > 1 && !isMeasurePairJoinable(selectedMeasureKeys)) {
        throw new IllegalArgumentException("선택된 measure들 간 조인이 성립하지 않습니다.");
    }

    // --- 2. Dimension 간 조인 가능성 ---
    if (selectedDimensionKeys.size() > 1 && !isDimensionPairJoinable(selectedDimensionKeys)) {
        throw new IllegalArgumentException("선택된 dimension들 간 조인이 성립하지 않습니다.");
    }

    // --- 3. Measure ↔ Dimension 간 연결성 ---
    if (!selectedMeasureKeys.isEmpty() && !selectedDimensionKeys.isEmpty()) {
        if (!isMeasureDimensionJoinable(selectedMeasureKeys, selectedDimensionKeys)) {
            throw new IllegalArgumentException("선택된 measure과 dimension 간 조인이 성립하지 않습니다.");
        }
    }

    // --- 4. Measure가 하나인데 연결 가능한 대상이 없는 경우 ---
    if (selectedMeasureKeys.size() == 1) {
        String m = selectedMeasureKeys.iterator().next();
        if (!isMeasureConnected(m)) {
            throw new IllegalArgumentException("해당 measure는 연결 가능한 dimension/filter가 없습니다.");
        }
    }

    // --- 5. Dimension이 하나인데 연결 가능한 대상이 없는 경우 ---
    if (selectedDimensionKeys.size() == 1) {
        String d = selectedDimensionKeys.iterator().next();
        if (!isDimensionConnected(d)) {
            throw new IllegalArgumentException("해당 dimension은 연결 가능한 measure/filter가 없습니다.");
        }
    }

    // --- (선택) Filter는 독립적이기 때문에 유효성 검사 생략 가능 ---
}
```

---

## ✅ 보조 함수들 요약

이미 만들어뒀지만 한 번 더 정리해 줄게:

### `isMeasurePairJoinable(...)`

```java
private boolean isMeasurePairJoinable(Set<String> selectedMeasureKeys) {
    if (selectedMeasureKeys.size() <= 1) return true;

    List<String> keys = new ArrayList<>(selectedMeasureKeys);
    for (int i = 0; i < keys.size(); i++) {
        for (int j = i + 1; j < keys.size(); j++) {
            String m1 = keys.get(i), m2 = keys.get(j);
            if (isJoinable(Set.of(m1), Set.of(m2), REPORT_MEASURE) ||
                isJoinable(Set.of(m2), Set.of(m1), REPORT_MEASURE)) return true;
        }
    }
    return false;
}
```

### `isDimensionPairJoinable(...)`

(동일 구조)

### `isMeasureDimensionJoinable(...)`

```java
private boolean isMeasureDimensionJoinable(Set<String> selectedMeasureKeys, Set<String> selectedDimensionKeys) {
    if (selectedMeasureKeys.isEmpty() || selectedDimensionKeys.isEmpty()) return true;

    return isJoinable(selectedMeasureKeys, selectedDimensionKeys, REPORT_DIMENSION) ||
           isJoinable(selectedDimensionKeys, selectedMeasureKeys, REPORT_MEASURE);
}
```

### `isMeasureConnected(...)`

```java
private boolean isMeasureConnected(String measureKey) {
    FieldConfig field = fieldConfigMap.get(measureKey);
    if (field == null) return false;

    return !getJoinTargets(field, REPORT_DIMENSION).isEmpty()
        || !getJoinTargets(field, REPORT_FILTER).isEmpty()
        || !getJoinTargets(field, REPORT_MEASURE).isEmpty();
}
```

### `isDimensionConnected(...)`

```java
private boolean isDimensionConnected(String dimensionKey) {
    FieldConfig field = fieldConfigMap.get(dimensionKey);
    if (field == null) return false;

    return !getJoinTargets(field, REPORT_MEASURE).isEmpty()
        || !getJoinTargets(field, REPORT_FILTER).isEmpty()
        || !getJoinTargets(field, REPORT_DIMENSION).isEmpty();
}
```

---

## ✅ 사용 위치

`getJoinRecommendations(...)` 안에서 키 추출 직후 다음과 같이 호출하면 돼요:

```java
Set<String> selectedMeasureKeys = selectedMeasures.stream()
    .map(k -> toFullKey(REPORT_MEASURE, k)).collect(Collectors.toSet());
Set<String> selectedDimensionKeys = selectedDimensions.stream()
    .map(k -> toFullKey(REPORT_DIMENSION, k)).collect(Collectors.toSet());
Set<String> selectedFilterKeys = selectedFilters.stream()
    .map(k -> toFullKey(REPORT_FILTER, k)).collect(Collectors.toSet());

validateSelectedFields(selectedMeasureKeys, selectedDimensionKeys, selectedFilterKeys);
```

---

필요하면 로그나 사용자 친화 메시지로 더 개선해줄 수도 있어요.
또는 `Set<CategoryFieldKey>` 기반으로 바로 받아들이게 수정도 가능해요. 필요할까?

---

맞아, 아주 좋은 포인트야 — 이건 실제 시스템에서 **굉장히 중요한 예외적 케이스**이기도 해.
예를 들어:

> “이 measure(m1)은 독립적으로 사용되어야 하고, 다른 measure들과는 절대 join이 안 된다.”

또는

> “이 measure는 특정 dimension(d1)과만 join 가능하다.”

이런 제약이 있을 수 있어.

---

## 🔎 문제 정의

지금 로직은:

* **measure가 1개만 선택되었을 때는 `isMeasurePairJoinable`이 무조건 `true`를 반환**해.
* 하지만 실상은 그 1개가 “join 가능한 dimension이 전혀 없는 measure”일 수도 있음.

→ 결과적으로 **선택 자체는 valid로 통과되지만, 실제 추천은 아무것도 되지 않음.**

---

## ✅ 해결책

measure가 하나만 선택된 경우라도,
**그 measure가 최소한 하나의 dimension 또는 filter와 연결되어 있어야 유효**하다고 판단해야 해.

---

## 🔧 보완된 로직

### ① measure 1개일 때 dimension/filter와 연결 가능한지 확인

```java
private boolean isMeasureConnected(String measureKey) {
    FieldConfig field = fieldConfigMap.get(measureKey);
    if (field == null) return false;

    return !getJoinTargets(field, REPORT_DIMENSION).isEmpty()
        || !getJoinTargets(field, REPORT_FILTER).isEmpty()
        || !getJoinTargets(field, REPORT_MEASURE).isEmpty();
}
```

---

### ② dimension 1개일 때도 유사하게

```java
private boolean isDimensionConnected(String dimensionKey) {
    FieldConfig field = fieldConfigMap.get(dimensionKey);
    if (field == null) return false;

    return !getJoinTargets(field, REPORT_MEASURE).isEmpty()
        || !getJoinTargets(field, REPORT_DIMENSION).isEmpty()
        || !getJoinTargets(field, REPORT_FILTER).isEmpty();
}
```

---

## 🧠 `getJoinRecommendations` 앞단에서 사용 예시

```java
if (selectedMeasureKeys.size() == 1) {
    String m = selectedMeasureKeys.iterator().next();
    if (!isMeasureConnected(m)) {
        throw new IllegalArgumentException("해당 measure는 연결 가능한 dimension/filter가 없습니다.");
    }
}

if (selectedDimensionKeys.size() == 1) {
    String d = selectedDimensionKeys.iterator().next();
    if (!isDimensionConnected(d)) {
        throw new IllegalArgumentException("해당 dimension은 연결 가능한 measure/filter가 없습니다.");
    }
}
```

---

## ✅ 이렇게 하면 완벽히 보장됨:

| 선택 조합                      | 처리                                |
| -------------------------- | --------------------------------- |
| M 하나, D 하나 → 연결됨           | OK                                |
| M 하나, D 하나 → 연결 안됨         | ❌ 오류                              |
| M 하나, D 없음 → M이 연결돼 있으면 OK | ✅                                 |
| M 하나, D 없음 → M이 고립돼 있으면    | ❌ 오류                              |
| M 둘 → 서로 연결 안 됨            | ❌ 오류 (`isMeasurePairJoinable` 체크) |

---

필요하다면 이걸 하나의 `validateSelectedFields(...)` 유틸로 만들어줄 수도 있어!
통합할까?

---

좋아, 그럼 `isMeasurePairJoinable` / `isDimensionPairJoinable`을 **쌍방 확인 포함** 형태로 안전하게 다시 만들어줄게.
이제는 **measure 간** 혹은 **dimension 간**에

* `m1 → m2` 또는 `m2 → m1`
* `d1 → d2` 또는 `d2 → d1`
  둘 중 하나라도 있으면 **joinable**하다고 판단해.

---

## ✅ `isMeasurePairJoinable` (쌍방 체크)

```java
private boolean isMeasurePairJoinable(Set<String> selectedMeasureKeys) {
    if (selectedMeasureKeys.size() <= 1) return true;
    List<String> keys = new ArrayList<>(selectedMeasureKeys);
    int size = keys.size();

    for (int i = 0; i < size; i++) {
        String m1 = keys.get(i);
        for (int j = i + 1; j < size; j++) {
            String m2 = keys.get(j);

            if (isJoinable(Set.of(m1), Set.of(m2), REPORT_MEASURE) ||
                isJoinable(Set.of(m2), Set.of(m1), REPORT_MEASURE)) {
                return true;
            }
        }
    }
    return false;
}
```

---

## ✅ `isDimensionPairJoinable` (쌍방 체크)

```java
private boolean isDimensionPairJoinable(Set<String> selectedDimensionKeys) {
    if (selectedDimensionKeys.size() <= 1) return true;
    List<String> keys = new ArrayList<>(selectedDimensionKeys);
    int size = keys.size();

    for (int i = 0; i < size; i++) {
        String d1 = keys.get(i);
        for (int j = i + 1; j < size; j++) {
            String d2 = keys.get(j);

            if (isJoinable(Set.of(d1), Set.of(d2), REPORT_DIMENSION) ||
                isJoinable(Set.of(d2), Set.of(d1), REPORT_DIMENSION)) {
                return true;
            }
        }
    }
    return false;
}



private boolean isMeasureDimensionJoinable(Set<String> selectedMeasureKeys, Set<String> selectedDimensionKeys) {
    if (selectedMeasureKeys.isEmpty() || selectedDimensionKeys.isEmpty()) return true;

    // M → D or D → M 둘 다 가능해야 하므로 양방 체크
    return isJoinable(selectedMeasureKeys, selectedDimensionKeys, REPORT_DIMENSION) ||
           isJoinable(selectedDimensionKeys, selectedMeasureKeys, REPORT_MEASURE);
}

```

---

## 🔄 내부적으로 사용되는 `isJoinable` 재확인:

```java
private boolean isJoinable(Set<String> fromKeys, Set<String> toKeys, String toGroup) {
    for (String fromKey : fromKeys) {
        FieldConfig from = fieldConfigMap.get(fromKey);
        if (from == null) continue;

        List<JoinConfig> joins = getJoinTargets(from, toGroup);
        for (JoinConfig join : joins) {
            if (toKeys.contains(join.getTarget())) {
                return true;
            }
        }
    }
    return false;
}
```

---

### ✅ 이제 다음을 보장함:

| 조합             | 정의 방향 | 판단 가능 여부   |
| -------------- | ----- | ---------- |
| `m1 → m2`만 정의됨 | 한 방향  | ✅ 가능       |
| `d2 → d1`만 정의됨 | 한 방향  | ✅ 가능       |
| `m1 ↔ m2` 정의됨  | 양방향   | ✅ 가능       |
| 아무 조인 없음       | 없음    | ❌ false 반환 |

---

필요하면 이 로직을 `getJoinRecommendations` 앞단에 넣어주는 흐름까지 정리해줄 수 있어. 원해?


----

좋아, 그럼 네가 요청한 `isMeasurePairJoinable`, `isDimensionPairJoinable` 및 일반화된 `isJoinable(...)` 함수까지 포함해서 아래와 같이 정리해줄게.

---

## ✅ 1. `isJoinable(...)` (범용)

```java
private boolean isJoinable(Set<String> fromKeys, Set<String> toKeys, String toGroup) {
    for (String fromKey : fromKeys) {
        FieldConfig from = fieldConfigMap.get(fromKey);
        if (from == null) continue;

        List<JoinConfig> joins = getJoinTargets(from, toGroup);
        for (JoinConfig join : joins) {
            if (toKeys.contains(join.getTarget())) {
                return true;
            }
        }
    }
    return false;
}
```

---

## ✅ 2. `isMeasurePairJoinable(...)`

```java
private boolean isMeasurePairJoinable(Set<String> selectedMeasureKeys) {
    for (String m1 : selectedMeasureKeys) {
        for (String m2 : selectedMeasureKeys) {
            if (m1.equals(m2)) continue;
            if (isJoinable(Set.of(m1), Set.of(m2), REPORT_MEASURE)) {
                return true;
            }
        }
    }
    return false;
}
```

---

## ✅ 3. `isDimensionPairJoinable(...)`

```java
private boolean isDimensionPairJoinable(Set<String> selectedDimensionKeys) {
    for (String d1 : selectedDimensionKeys) {
        for (String d2 : selectedDimensionKeys) {
            if (d1.equals(d2)) continue;
            if (isJoinable(Set.of(d1), Set.of(d2), REPORT_DIMENSION)) {
                return true;
            }
        }
    }
    return false;
}
```

---

## ✅ 4. 사용 예시 (조합 유효성 체크)

```java
if (!isJoinable(selectedMeasureKeys, selectedDimensionKeys, REPORT_DIMENSION)) {
    throw new IllegalArgumentException("선택된 measure과 dimension 간 조인이 성립하지 않습니다.");
}

if (!isMeasurePairJoinable(selectedMeasureKeys)) {
    throw new IllegalArgumentException("선택된 measure들 간 조인이 성립하지 않습니다.");
}

if (!isDimensionPairJoinable(selectedDimensionKeys)) {
    throw new IllegalArgumentException("선택된 dimension들 간 조인이 성립하지 않습니다.");
}
```

---

## 🔧 함께 쓰면 좋은 위치

이 로직들은 `getJoinRecommendations(...)` 메소드의 **초반부** (선택값을 key로 바꾸고 나서) 넣는 게 좋아요:

```java
Set<String> selectedMeasureKeys = selectedMeasures.stream()
    .map(k -> toFullKey(REPORT_MEASURE, k))
    .collect(Collectors.toSet());
// ...
// 조인 가능성 체크
if (!isJoinable(selectedMeasureKeys, selectedDimensionKeys, REPORT_DIMENSION)) {
    throw new IllegalArgumentException("선택된 measure과 dimension 간 조인이 성립하지 않습니다.");
}
```

---

필요하면 여기서 더 확장해서 `M-F`, `D-F` 체크도 추가해줄 수 있어.
혹시 그 부분도 원해?


----
아 아주 핵심 포인트를 정확하게 집어줬어👍
너가 말한 것처럼:

* 만약 입력된 `selectedMeasures`나 `selectedDimensions`끼리 **전혀 연결되어 있지 않다면**,
* 이는 그냥 “조건에 맞지 않는 조합”이라는 의미지.
* 이거는 **"isJoinable"** 같은 별도의 함수를 만들어서 확인할 수 있다고 생각해.

---

## 🔹아이디어 (isJoinable)

```java
private boolean isJoinable(Set<String> selectedMeasureKeys, Set<String> selectedDimensionKeys) {
    for (String measureKey : selectedMeasureKeys) {
        FieldConfig measure = fieldConfigMap.get(measureKey);
        if (measure == null) continue;

        List<JoinConfig> joins = getJoinTargets(measure, REPORT_DIMENSION);
        for (JoinConfig j : joins) {
            if (selectedDimensionKeys.contains(j.getTarget())) {
                return true; // 하나라도 연결되어 있으면 true
            }
        }
    }
    return false;
}
```

---

## 🔹사용

```java
if (!isJoinable(selectedMeasureKeys, selectedDimensionKeys)) {
    throw new IllegalArgumentException("조건에 맞지 않는 measure-dimension 조합입니다.");
}
```

---

## 🔹확장 (M-M이나 D-D의 연결 확인)

동일하게

```java
private boolean isMeasurePairJoinable(Set<String> selectedMeasureKeys) { … }
private boolean isDimensionPairJoinable(Set<String> selectedDimensionKeys) { … }
```

처럼 구현할 수 있어.

---

## 🔹추가 고려 (Generalized)

만약 한 가지 기능으로 풀려고 하면:

```java
private boolean isJoinable(Set<String> fromKeys, Set<String> toKeys, String fromType, String toType) {
    for (String fromKey : fromKeys) {
        FieldConfig from = fieldConfigMap.get(fromKey);
        if (from == null) continue;

        List<JoinConfig> joins = getJoinTargets(from, toType);
        for (JoinConfig j : joins) {
            if (toKeys.contains(j.getTarget())) {
                return true;
            }
        }
    }
    return false;
}
```

---

## 🔹정리:

✅ `isJoinable` 함수를 만들어서 **선택된 IDs끼리 연결되어 있는지 확인**할 수 있다.

✅ 만약 연결되어 있지 않다면 Exception이나 오류 메시지를 발생시켜서
사용자가 "조건에 맞지 않는 조합"이라는 것을 인지하게 만들면 된다.

---

만약 해당 기능과 Exception Handling 까지 포함해서 한꺼번에 리팩토링하거나
너의 `getJoinRecommendations` 내부에서 자연스러운 흐름으로 넣고싶다면 또 알려줘! 🌟


---

좋아요. 아래는 당신이 올린 `JoinGraphHelper` 클래스의 **`getJoinRecommendations(...)` 메소드 전체 흐름과 각 파트의 목적, 작동 원리**를 **상세하게 한 줄 한 줄 해설**한 것입니다.

---

## 🧠 개요: 목적

이 메서드는 사용자 선택에 따라 Measure / Dimension / Filter 필드들을 추천해주는 로직입니다.
추천 기준은 YAML로 정의된 field 간의 join 관계(graph) 기반이며, 다음 원칙에 따라 추천됩니다:

* **Measure → Dimension** : 교집합
* **Measure → Filter** : 종속
* **Measure ↔ Measure** : 연결된 다른 measure
* **Dimension ↔ Dimension** : 교집합
* **Dimension → Filter** : 종속
* **Dimension → Measure** : 연결된 measure

---

## 📦 주요 인풋

```java
Set<CategoryFieldKey> selectedMeasures,
Set<CategoryFieldKey> selectedDimensions,
Set<CategoryFieldKey> selectedFilters
```

사용자가 현재 선택한 Measure/Dimension/Filter 필드들입니다.

---

## 📤 리턴

```java
JoinRecommendationResponse
```

Measure, Dimension, Filter 각각에 대해 추천된 `JoinFieldInfo` 객체 목록을 담고 있습니다.

---

## ✅ 전체 구조와 각 파트 설명

```java
public JoinRecommendationResponse getJoinRecommendations(...) {
```

### 1. 초기 세팅

```java
JoinRecommendationResponse response = new JoinRecommendationResponse();
```

→ 반환할 추천 결과입니다.

```java
Set<String> selectedMeasureKeys = selectedMeasures.stream().map(k -> toFullKey(REPORT_MEASURE, k)).collect(Collectors.toSet());
Set<String> selectedDimensionKeys = selectedDimensions.stream().map(k -> toFullKey(REPORT_DIMENSION, k)).collect(Collectors.toSet());
Set<String> selectedFilterKeys = selectedFilters.stream().map(k -> toFullKey(REPORT_FILTER, k)).collect(Collectors.toSet());
```

→ 필드들을 전부 `"group:category:field:metric"` 포맷으로 풀 키로 변환. 키 기반 매핑 및 비교용.

---

### 2. MEASURE 추천 로직

```java
if (selectedMeasures.isEmpty()) {
```

▶ **선택된 measure가 하나도 없을 때** → 전체 measure를 다 추천함.

```java
for (String key : getAllFieldKeys(REPORT_MEASURE)) {
    addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, key));
}
```

---

```java
} else {
```

▶ **선택된 measure가 있는 경우** → 연결된 dimension과 filter를 기반으로 필터링된 추천을 수행.

#### 2-1. Dimension 교집합 찾기

```java
Set<String> dimensionIntersection = null;

for (String measureKey : selectedMeasureKeys) {
    FieldConfig field = getFieldConfig(measureKey);
    ...
    Set<String> dimTargets = getJoinTargets(field, REPORT_DIMENSION).stream().map(JoinConfig::getTarget).collect(Collectors.toSet());

    if (dimensionIntersection == null) dimensionIntersection = new HashSet<>(dimTargets);
    else dimensionIntersection.retainAll(dimTargets);
```

→ 선택된 모든 measure이 공통적으로 연결된 dimension만 남김 (intersection)

---

#### 2-2. Filter 추천

```java
getJoinTargets(field, REPORT_FILTER).forEach(j -> recommendedFilterKeys.add(j.getTarget()));
```

→ 각 measure이 직접 연결한 filter들을 추천 후보에 넣음

---

#### 2-3. 다른 Measure 추천 (양방향 관계 + 단위 호환)

```java
for (String key : getAllFieldKeys(REPORT_MEASURE)) {
    if (selectedMeasureKeys.contains(key)) continue;
    if (!isCompatibleUnit(key, selectedMeasureKeys)) continue;

    if (isMeasureJoinRelated(...)) {
        addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, key));
    }
}
```

* 이미 선택된 건 제외
* 단위(unit)가 호환되는지 확인
* measure 간 join 관계가 있다면 추천

---

#### 2-4. Dimension 교집합 추가

```java
if (dimensionIntersection != null) {
    for (String key : dimensionIntersection) {
        if (!selectedDimensionKeys.contains(key)) {
            addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, key));
        }
    }
}
```

→ measure들이 모두 연결된 dimension 중에서 아직 선택되지 않은 것들만 추천

---

### 3. DIMENSION 추천 로직

```java
if (!selectedDimensions.isEmpty()) {
```

#### 3-1. 교집합된 dimension 찾기 + 관련 measure 수집

```java
Set<String> dimensionIntersection = null;
Set<String> relatedMeasures = new HashSet<>();

for (String dimKey : selectedDimensionKeys) {
    FieldConfig field = getFieldConfig(dimKey);
    ...

    getJoinTargets(field, REPORT_MEASURE).forEach(j -> relatedMeasures.add(j.getTarget()));
    getJoinTargets(field, REPORT_FILTER).forEach(j -> recommendedFilterKeys.add(j.getTarget()));
```

→ dimension이 연결된 measure / filter 수집

---

#### 3-2. dimension 간 교집합

```java
Set<String> dimTargets = getJoinTargets(field, REPORT_DIMENSION).stream().map(JoinConfig::getTarget).collect(Collectors.toSet());
if (dimensionIntersection == null) dimensionIntersection = new HashSet<>(dimTargets);
else dimensionIntersection.retainAll(dimTargets);
```

→ 선택된 dimension 간 교집합을 구함 (dimension ↔ dimension)

---

#### 3-3. 관련된 measure 추천

```java
for (String key : relatedMeasures) {
    if (!selectedMeasureKeys.contains(key)) {
        addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, key));
    }
}
```

---

#### 3-4. dimension 교집합 추천

```java
if (dimensionIntersection != null) {
    for (String key : dimensionIntersection) {
        if (!selectedDimensionKeys.contains(key)) {
            addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, key));
        }
    }
}
```

---

#### 3-5. 이미 선택된 dimension은 추천에서 제거

```java
response.getDimension().removeIf(info ->
    selectedDimensionKeys.contains(toFullKey(REPORT_DIMENSION,
        new CategoryFieldKey(info.getCategory(), info.getField(), info.getMetric()))));
```

---

### 4. FILTER 추천 로직

#### 4-1. measure, dimension에서 연결된 filter 수집

```java
Set<String> filterFromJoins = new HashSet<>();
for (String key : selectedMeasureKeys) {
    ...
    getJoinTargets(config, REPORT_FILTER).forEach(j -> filterFromJoins.add(j.getTarget()));
}
```

---

#### 4-2. 후보 필터 = measure + dimension + filterFromJoins

```java
Set<String> totalFilterCandidates = new HashSet<>();
totalFilterCandidates.addAll(selectedMeasureKeys);
totalFilterCandidates.addAll(selectedDimensionKeys);
totalFilterCandidates.addAll(filterFromJoins);
```

---

#### 4-3. 아직 선택되지 않은 filter 중에서 추천

```java
for (String key : getAllFieldKeys(REPORT_FILTER)) {
    if (selectedFilterKeys.contains(key)) continue;
    if (totalFilterCandidates.contains(key)) {
        addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, key));
    }
}
```

---

#### 4-4. 필터에서 자기 자신 제거

```java
response.getFilter().removeIf(info ->
    selectedFilters.stream().anyMatch(selected ->
        Objects.equals(selected.getCategory(), info.getCategory()) &&
        Objects.equals(selected.getField(), info.getField()) &&
        (selected.getMetric() == null || Objects.equals(selected.getMetric(), info.getMetric()))
    )
);
```

---

## 🔁 보조 메소드 설명

* `toFullKey(group, key)`: `"group:category:field:metric"` 포맷 생성
* `fromFullKey(...)`: 다시 CategoryFieldKey로 파싱
* `createJoinFieldInfo(...)`: `JoinFieldInfo` 객체 생성
* `getJoinTargets(...)`: 필드에서 특정 group과 연결된 JoinConfig 리스트 반환
* `isCompatibleUnit(...)`: 단위(unit)가 겹치는지 여부
* `isMeasureJoinRelated(...)`: measure ↔ measure 간의 양방향 관계 판단

---

아 아주 핵심 포인트를 정확하게 집어내셨네요 👏
당신이 지적한 이 `hasJoinRelation` 조건문은
**“선택된 measure들과 연결되어 있는 다른 measure들만 추천 리스트에 넣자”**
는 의미입니다.
즉 **m↔m 연결** 만 고려되어 있습니다.

---

### 만약 같은 방식의 연결 확인이 **dim↔dim에도 필요하다면**

당연하게 **"선택된 dimension과 연결되어 있는 다른 dimension만 추천"** 할 수 있도록 비슷하게 구현해야죠.

다시 한마디로:

```java
// dimension에도 같은 식으로 연결 확인해야 함
boolean hasDimensionRelation = false;
for (String selectedKey : selectedDimensionKeys) {
    if (isDimensionJoinRelated(selectedKey, key)) { // 이 메서드를 만들어서
        hasDimensionRelation = true;
        break;
    }
}

if (hasDimensionRelation) {
    addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, key)); 
    recommendedDimensionKeys.add(key);
}
```

---

### 요령:

* `isMeasureJoinRelated`처럼

```java
private boolean isDimensionJoinRelated(String fromKey, String toKey) {
    FieldConfig fromConfig = fieldConfigMap.get(fromKey);
    FieldConfig toConfig = fieldConfigMap.get(toKey);
    if (fromConfig == null || toConfig == null) return false;

    List<JoinConfig> fromJoins = getJoinTargets(fromConfig, REPORT_DIMENSION);
    List<JoinConfig> toJoins = getJoinTargets(toConfig, REPORT_DIMENSION);

    return fromJoins.stream().anyMatch(j -> j.getTarget().equals(toKey)) ||
            toJoins.stream().anyMatch(j -> j.getTarget().equals(fromKey)); 
}
```

처럼 작성할 수 있습니다.

---

### 그렇게 하면…

✅ m↔m 연결과 유사하게,
✅ d↔d 연결에도 같은 원리로 한정해서 추천하게 됩니다.
✅ 이로 인해 \*\*“선택된 dimension과 연결되어 있는 dimension들만 추천된다”\*\*는 의미의 그래프 연결성을 고려하게 되어 더욱 깔끔하게 됩니다.

---

### 한 줄 요약:

➥ measure끼리 연결 확인할 수 있도록 한 것처럼
➥ dimension끼리 연결 확인 기능(ex. `isDimensionJoinRelated`)과 해당 조건문과 반복문을 넣으면
➥ d↔d 연결에도 동일하게 적용할 수 있습니다.

---

만약 해당 기능이나 코드 구현에도 도움이 또 궁금한 점이 있으면 말씀해주세요! 🌟


→ 현재 결과 필터링에는 쓰이지 않고, 단순히 `.add(...)`만 수행하므로 제거 가능.

---

필요하면 이 코드를 더 간결하고 확장성 있게 리팩토링하거나, 각 추천 종류별로 분리된 메서드 구조로 정리해드릴 수도 있어요.

---

양방향
```java
// --- JoinGraphHelper.java ---

public class JoinGraphHelper {

    private final Map<String, FieldConfig> fieldConfigMap;

    public JoinGraphHelper(Map<String, FieldConfig> fieldConfigMap) {
        this.fieldConfigMap = fieldConfigMap;
    }

    public JoinRecommendationResponse getJoinRecommendations(
        Set<CategoryFieldKey> selectedMeasures,
        Set<CategoryFieldKey> selectedDimensions,
        Set<CategoryFieldKey> selectedFilters
    ) {
        JoinRecommendationResponse response = new JoinRecommendationResponse();

        Set<String> selectedMeasureKeys = selectedMeasures.stream().map(k -> toFullKey(REPORT_MEASURE, k)).collect(Collectors.toSet());
        Set<String> selectedDimensionKeys = selectedDimensions.stream().map(k -> toFullKey(REPORT_DIMENSION, k)).collect(Collectors.toSet());
        Set<String> selectedFilterKeys = selectedFilters.stream().map(k -> toFullKey(REPORT_FILTER, k)).collect(Collectors.toSet());

        Set<String> recommendedMeasureKeys = new HashSet<>();
        Set<String> recommendedDimensionKeys = new HashSet<>();
        Set<String> recommendedFilterKeys = new HashSet<>();

        // --- MEASURE ---
        if (selectedMeasures.isEmpty()) {
            for (String key : getAllFieldKeys(REPORT_MEASURE)) {
                addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, key));
                recommendedMeasureKeys.add(key);
            }
        } else {
            Set<String> dimensionIntersection = null;

            for (String measureKey : selectedMeasureKeys) {
                FieldConfig field = getFieldConfig(measureKey);
                if (field == null) continue;

                List<JoinConfig> dimJoins = getJoinTargets(field, REPORT_DIMENSION);
                Set<String> dimTargets = dimJoins.stream().map(JoinConfig::getTarget).collect(Collectors.toSet());
                if (dimensionIntersection == null) dimensionIntersection = new HashSet<>(dimTargets);
                else dimensionIntersection.retainAll(dimTargets);

                getJoinTargets(field, REPORT_FILTER).forEach(j -> recommendedFilterKeys.add(j.getTarget()));
            }

            for (String key : getAllFieldKeys(REPORT_MEASURE)) {
                if (selectedMeasureKeys.contains(key)) continue;
                if (!isCompatibleUnit(key, selectedMeasureKeys)) continue;

                // 양방향 관계를 고려한 추천
                boolean hasJoinRelation = false;
                for (String selectedKey : selectedMeasureKeys) {
                    if (isMeasureJoinRelated(selectedKey, key)) {
                        hasJoinRelation = true;
                        break;
                    }
                }
                if (hasJoinRelation) {
                    addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, key));
                    recommendedMeasureKeys.add(key);
                }
            }

            if (dimensionIntersection != null) {
                for (String key : dimensionIntersection) {
                    if (!selectedDimensionKeys.contains(key)) {
                        addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, key));
                        recommendedDimensionKeys.add(key);
                    }
                }
            }
        }

        // --- DIMENSION ---
        if (!selectedDimensions.isEmpty()) {
            Set<String> dimensionIntersection = null;
            Set<String> relatedMeasures = new HashSet<>();

            for (String dimKey : selectedDimensionKeys) {
                FieldConfig field = getFieldConfig(dimKey);
                if (field == null) continue;

                getJoinTargets(field, REPORT_MEASURE).forEach(j -> relatedMeasures.add(j.getTarget()));
                getJoinTargets(field, REPORT_FILTER).forEach(j -> recommendedFilterKeys.add(j.getTarget()));

                List<JoinConfig> dimJoins = getJoinTargets(field, REPORT_DIMENSION);
                Set<String> dimTargets = dimJoins.stream().map(JoinConfig::getTarget).collect(Collectors.toSet());
                if (dimensionIntersection == null) dimensionIntersection = new HashSet<>(dimTargets);
                else dimensionIntersection.retainAll(dimTargets);
            }

            for (String key : relatedMeasures) {
                if (!selectedMeasureKeys.contains(key)) {
                    addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, key));
                    recommendedMeasureKeys.add(key);
                }
            }

            if (dimensionIntersection != null) {
                for (String key : dimensionIntersection) {
                    if (!selectedDimensionKeys.contains(key)) {
                        addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, key));
                        recommendedDimensionKeys.add(key);
                    }
                }
            }

            response.getDimension().removeIf(info ->
                    selectedDimensionKeys.contains(toFullKey(REPORT_DIMENSION,
                            new CategoryFieldKey(info.getCategory(), info.getField(), info.getMetric()))));
        }

        // --- FILTER ---
        Set<String> filterFromJoins = new HashSet<>();
        for (String key : selectedMeasureKeys) {
            FieldConfig config = getFieldConfig(key);
            if (config == null) continue;
            getJoinTargets(config, REPORT_FILTER).forEach(j -> filterFromJoins.add(j.getTarget()));
        }
        for (String key : selectedDimensionKeys) {
            FieldConfig config = getFieldConfig(key);
            if (config == null) continue;
            getJoinTargets(config, REPORT_FILTER).forEach(j -> filterFromJoins.add(j.getTarget()));
        }

        Set<String> totalFilterCandidates = new HashSet<>();
        totalFilterCandidates.addAll(selectedMeasureKeys);
        totalFilterCandidates.addAll(selectedDimensionKeys);
        totalFilterCandidates.addAll(filterFromJoins);

        for (String key : getAllFieldKeys(REPORT_FILTER)) {
            if (selectedFilterKeys.contains(key)) continue;
            if (totalFilterCandidates.contains(key)) {
                addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, key));
            }
        }

        response.getFilter().removeIf(info ->
            selectedFilters.stream().anyMatch(selected ->
                Objects.equals(selected.getCategory(), info.getCategory()) &&
                Objects.equals(selected.getField(), info.getField()) &&
                (selected.getMetric() == null || Objects.equals(selected.getMetric(), info.getMetric()))
            )
        );

        return response;
    }

    private FieldConfig getFieldConfig(String key) {
        return fieldConfigMap.get(key);
    }

    private List<JoinConfig> getJoinTargets(FieldConfig field, String group) {
        return Optional.ofNullable(field.getJoins()).map(j -> j.get(group)).orElse(List.of());
    }

    private boolean isCompatibleUnit(String candidateKey, Set<String> selectedKeys) {
        FieldConfig candidate = fieldConfigMap.get(candidateKey);
        if (candidate == null) return false;
        Set<String> candidateUnits = extractUnits(candidate);

        for (String key : selectedKeys) {
            FieldConfig selected = fieldConfigMap.get(key);
            if (selected == null) continue;
            Set<String> selectedUnits = extractUnits(selected);
            if (!Collections.disjoint(candidateUnits, selectedUnits)) {
                return true;
            }
        }
        return false;
    }

    private Set<String> extractUnits(FieldConfig config) {
        return Optional.ofNullable(config.getMetric())
                .map(m -> m.values().stream()
                        .map(MetricConfig::getUnit)
                        .filter(Objects::nonNull)
                        .collect(Collectors.toSet()))
                .orElse(Set.of());
    }

    private void addIfNotNull(List<JoinFieldInfo> list, JoinFieldInfo info) {
        if (info != null) list.add(info);
    }

    private String toFullKey(String group, CategoryFieldKey k) {
        return k.getMetric() == null ?
                String.format("%s:%s:%s:", group, k.getCategory(), k.getField()) :
                String.format("%s:%s:%s:%s", group, k.getCategory(), k.getField(), k.getMetric());
    }

    private JoinFieldInfo createJoinFieldInfo(String group, String fullKey) {
        FieldConfig config = fieldConfigMap.get(fullKey);
        if (config == null) return null;
        CategoryFieldKey parsed = fromFullKey(fullKey);
        return new JoinFieldInfo(parsed.getCategory(), parsed.getField(), parsed.getMetric());
    }

    private CategoryFieldKey fromFullKey(String fullKey) {
        String[] parts = fullKey.split(":");
        String category = parts[1];
        String field = parts[2];
        String metric = parts.length > 3 ? parts[3] : null;
        return new CategoryFieldKey(category, field, metric);
    }

    private Set<String> getAllFieldKeys(String group) {
        return fieldConfigMap.keySet().stream()
                .filter(k -> k.startsWith(group + ":"))
                .collect(Collectors.toSet());
    }

    // --- New logic for bidirectional measure joins ---
    private boolean isMeasureJoinRelated(String fromKey, String toKey) {
        FieldConfig fromConfig = fieldConfigMap.get(fromKey);
        FieldConfig toConfig = fieldConfigMap.get(toKey);
        if (fromConfig == null || toConfig == null) return false;

        List<JoinConfig> fromJoins = getJoinTargets(fromConfig, REPORT_MEASURE);
        List<JoinConfig> toJoins = getJoinTargets(toConfig, REPORT_MEASURE);

        return fromJoins.stream().anyMatch(j -> j.getTarget().equals(toKey)) ||
               toJoins.stream().anyMatch(j -> j.getTarget().equals(fromKey));
    }

    private static final String REPORT_MEASURE = "measure";
    private static final String REPORT_DIMENSION = "dimension";
    private static final String REPORT_FILTER = "filter";
}


```
