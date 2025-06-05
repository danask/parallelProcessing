좋아요 Daniel. 요청하신 전체 로직을 다음과 같이 정리해 드릴게요. 아래 코드는 **선택된 필드(M, D, F)를 기반으로 추천할 M, D, F 리스트를 생성**하는 완전한 예제입니다.

---

## ✅ 전제 정보

### 📌 DTO 구조 예시

```java
public class SelectedField {
    private String group;    // measure, dimension, filter
    private String category;
    private String field;
    private String metric;   // nullable
}
```

### 📌 `fieldConfigMap` 예시 (기 생성된 상태)

```java
Map<String, FieldConfig> fieldConfigMap; // key: group:category:field[:metric]
```

---

## ✅ 추천 로직 구현

```java
public class RecommendationService {

    public FieldRecommendationResponse getRecommendations(List<SelectedField> selectedFields,
                                                          Map<String, FieldConfig> fieldConfigMap) {
        // 1. 선택된 필드들을 group별로 분류
        Map<String, List<SelectedField>> selectedByGroup = selectedFields.stream()
            .collect(Collectors.groupingBy(SelectedField::getGroup));

        Set<String> selectedKeys = selectedFields.stream()
            .map(this::toKey)
            .collect(Collectors.toSet());

        // 2. 추천 대상 준비
        List<SelectedField> allMeasures = getFieldsByGroup("measure", fieldConfigMap);
        List<SelectedField> allDimensions = getFieldsByGroup("dimension", fieldConfigMap);
        List<SelectedField> allFilters = getFieldsByGroup("filter", fieldConfigMap);

        // 3. M 추천
        List<SelectedField> recommendedMeasures;
        if (selectedByGroup.get("measure") == null || selectedByGroup.get("measure").isEmpty()) {
            // M이 선택되지 않은 경우 → 전체 M
            recommendedMeasures = allMeasures;
        } else {
            Set<String> relatedMeasures = getRelatedMeasures(selectedByGroup.get("measure"), fieldConfigMap);
            recommendedMeasures = allMeasures.stream()
                .filter(m -> relatedMeasures.contains(toKey(m)) && !selectedKeys.contains(toKey(m)))
                .collect(Collectors.toList());
        }

        // 4. D 추천
        List<SelectedField> recommendedDimensions;
        if (selectedByGroup.get("dimension") == null || selectedByGroup.get("dimension").isEmpty()) {
            Set<String> relatedDimensions = getCommonDimensions(selectedByGroup.get("measure"), fieldConfigMap);
            recommendedDimensions = allDimensions.stream()
                .filter(d -> relatedDimensions.contains(toKey(d)) && !selectedKeys.contains(toKey(d)))
                .collect(Collectors.toList());
        } else {
            Set<String> relatedDimensions = getRelatedDimensions(selectedByGroup.get("dimension"), fieldConfigMap);
            recommendedDimensions = allDimensions.stream()
                .filter(d -> relatedDimensions.contains(toKey(d)) && !selectedKeys.contains(toKey(d)))
                .collect(Collectors.toList());
        }

        // 5. F 추천
        List<SelectedField> recommendedFilters;
        Set<String> relatedFilters = getUnionFilters(selectedFields, fieldConfigMap);
        recommendedFilters = allFilters.stream()
            .filter(f -> relatedFilters.contains(toKey(f)) && !selectedKeys.contains(toKey(f)))
            .collect(Collectors.toList());

        // 6. 반환
        return new FieldRecommendationResponse(recommendedMeasures, recommendedDimensions, recommendedFilters);
    }

    private String toKey(SelectedField f) {
        return (f.getMetric() != null && !f.getMetric().isBlank())
            ? String.format("%s:%s:%s:%s", f.getGroup(), f.getCategory(), f.getField(), f.getMetric())
            : String.format("%s:%s:%s", f.getGroup(), f.getCategory(), f.getField());
    }

    private List<SelectedField> getFieldsByGroup(String group, Map<String, FieldConfig> fieldConfigMap) {
        return fieldConfigMap.entrySet().stream()
            .filter(e -> e.getKey().startsWith(group + ":"))
            .map(e -> {
                String[] parts = e.getKey().split(":");
                SelectedField f = new SelectedField();
                f.setGroup(parts[0]);
                f.setCategory(parts[1]);
                f.setField(parts[2]);
                if (parts.length > 3) f.setMetric(parts[3]);
                return f;
            })
            .collect(Collectors.toList());
    }

    private Set<String> getRelatedMeasures(List<SelectedField> selectedMeasures, Map<String, FieldConfig> map) {
        // 선택된 M 외 나머지 중 연결된 M만 포함 (즉, D 교집합 기준으로)
        Set<String> commonDimensions = getCommonDimensions(selectedMeasures, map);
        return map.entrySet().stream()
            .filter(e -> e.getKey().startsWith("measure:"))
            .filter(e -> {
                FieldConfig fc = e.getValue();
                return fc.getJoins() != null && fc.getJoins().containsKey("dimension") &&
                       !Collections.disjoint(fc.getJoins().get("dimension").stream()
                           .map(JoinConfig::getTarget).collect(Collectors.toSet()), commonDimensions);
            })
            .map(Map.Entry::getKey)
            .collect(Collectors.toSet());
    }

    private Set<String> getCommonDimensions(List<SelectedField> measures, Map<String, FieldConfig> map) {
        List<Set<String>> dimensionSets = new ArrayList<>();
        for (SelectedField m : measures) {
            String key = toKey(m);
            FieldConfig fc = map.get(key);
            if (fc == null || fc.getJoins() == null || !fc.getJoins().containsKey("dimension")) continue;
            Set<String> joined = fc.getJoins().get("dimension").stream()
                .map(JoinConfig::getTarget)
                .collect(Collectors.toSet());
            dimensionSets.add(joined);
        }
        return dimensionSets.stream().reduce((a, b) -> {
            a.retainAll(b);
            return a;
        }).orElse(Collections.emptySet());
    }

    private Set<String> getRelatedDimensions(List<SelectedField> dims, Map<String, FieldConfig> map) {
        Set<String> relatedDims = new HashSet<>();
        for (SelectedField d : dims) {
            String key = toKey(d);
            relatedDims.add(key);
        }
        return relatedDims;
    }

    private Set<String> getUnionFilters(List<SelectedField> selected, Map<String, FieldConfig> map) {
        Set<String> unionFilters = new HashSet<>();
        for (SelectedField f : selected) {
            String key = toKey(f);
            FieldConfig fc = map.get(key);
            if (fc == null || fc.getJoins() == null) continue;
            List<JoinConfig> filterJoins = fc.getJoins().get("filter");
            if (filterJoins != null) {
                for (JoinConfig join : filterJoins) {
                    unionFilters.add(join.getTarget());
                }
            }
        }
        return unionFilters;
    }
}
```

---

## 📦 예시: `FieldRecommendationResponse`

```java
public class FieldRecommendationResponse {
    private List<SelectedField> measures;
    private List<SelectedField> dimensions;
    private List<SelectedField> filters;

    // constructor, getters, setters
}
```

---

## 🔁 연결 객체 예시

```java
public class JoinConfig {
    private String target;
    // ... 기타 속성들
}
```

```java
public class FieldConfig {
    private Map<String, List<JoinConfig>> joins;
    // ...
}
```

---

필요하면 이걸 **Spring Service 형태**로 리팩토링해서 REST API로도 구성해드릴게요.
또한, 테스트 케이스나 로그 추가가 필요하면 말씀 주세요!


-----

좋아요 Daniel, 아래는 너가 원하는 동작을 정확히 반영해서 `getJoinRecommendations` 메서드를 리팩토링한 버전이야. 각 조건을 정리하고, 그에 따라 동작이 다르게 구성되어 있어:

---

### ✅ 기능 요약:

1. **선택된 M 없음 → 전체 M 보여줌**
2. **선택된 M 있음 →**

   * 자기 자신 제외한 M 중 관련된 것 보여줌
   * 관련 D: **선택된 M들 간 D 교집합**
   * 관련 F: **선택된 M들 간 F 합집합**
3. **선택된 D 있음 →**

   * 자기 자신 제외한 D 중 관련된 것 보여줌
   * 관련 M: **선택된 D들이 참조하는 M 중, 선택되지 않은 것**
   * 관련 F: **선택된 D들이 참조하는 F 중, 선택되지 않은 것**
4. **선택된 F 있음 →**

   * 자기 자신 제외한 F 중 관련된 것 보여줌

---

### ✅ 개선된 로직:

```java
public JoinRecommendationResponse getJoinRecommendations(
        Set<CategoryFieldKey> selectedMeasures,
        Set<CategoryFieldKey> selectedDimensions,
        Set<CategoryFieldKey> selectedFilters
) {
    JoinRecommendationResponse response = new JoinRecommendationResponse();

    // -------------------------
    // 1. 처리 기준 설정
    // -------------------------
    boolean hasM = !selectedMeasures.isEmpty();
    boolean hasD = !selectedDimensions.isEmpty();
    boolean hasF = !selectedFilters.isEmpty();

    // -------------------------
    // 2. M 로직
    // -------------------------
    Set<String> allMKeys = getAllFieldKeys("measure");
    if (!hasM) {
        for (String mKey : allMKeys) {
            JoinFieldInfo info = createJoinFieldInfo("measure", mKey);
            if (info != null) response.getMeasure().add(info);
        }
    } else {
        Set<String> dimensionIntersection = null;
        Set<String> filterUnion = new HashSet<>();

        for (CategoryFieldKey mKey : selectedMeasures) {
            String fullKey = toFullKey("measure", mKey);
            FieldConfig field = getFieldConfig(fullKey);
            if (field == null) continue;

            List<JoinConfig> dimJoins = Optional.ofNullable(field.getJoins())
                    .map(j -> j.get("dimension")).orElse(List.of());
            List<JoinConfig> filterJoins = Optional.ofNullable(field.getJoins())
                    .map(j -> j.get("filter")).orElse(List.of());

            Set<String> dimTargets = dimJoins.stream().map(JoinConfig::getTarget).collect(Collectors.toSet());
            if (dimensionIntersection == null) {
                dimensionIntersection = new HashSet<>(dimTargets);
            } else {
                dimensionIntersection.retainAll(dimTargets);
            }

            filterJoins.stream().map(JoinConfig::getTarget).forEach(filterUnion::add);
        }

        for (String mKey : allMKeys) {
            CategoryFieldKey key = fromFullKey(mKey);
            if (selectedMeasures.contains(key)) continue;

            JoinFieldInfo info = createJoinFieldInfo("measure", mKey);
            if (info != null) response.getMeasure().add(info);
        }

        if (dimensionIntersection != null) {
            for (String dKey : dimensionIntersection) {
                CategoryFieldKey key = fromFullKey(dKey);
                if (!selectedDimensions.contains(key)) {
                    JoinFieldInfo info = createJoinFieldInfo("dimension", dKey);
                    if (info != null) response.getDimension().add(info);
                }
            }
        }

        for (String fKey : filterUnion) {
            CategoryFieldKey key = fromFullKey(fKey);
            if (!selectedFilters.contains(key)) {
                JoinFieldInfo info = createJoinFieldInfo("filter", fKey);
                if (info != null) response.getFilter().add(info);
            }
        }
    }

    // -------------------------
    // 3. D 로직
    // -------------------------
    if (hasD) {
        Set<String> recommendedMeasures = new HashSet<>();
        Set<String> filterUnion = new HashSet<>();

        for (CategoryFieldKey dKey : selectedDimensions) {
            String fullKey = toFullKey("dimension", dKey);
            FieldConfig field = getFieldConfig(fullKey);
            if (field == null) continue;

            List<JoinConfig> measureJoins = Optional.ofNullable(field.getJoins())
                    .map(j -> j.get("measure")).orElse(List.of());
            List<JoinConfig> filterJoins = Optional.ofNullable(field.getJoins())
                    .map(j -> j.get("filter")).orElse(List.of());

            measureJoins.stream().map(JoinConfig::getTarget).forEach(recommendedMeasures::add);
            filterJoins.stream().map(JoinConfig::getTarget).forEach(filterUnion::add);
        }

        for (String dKey : getAllFieldKeys("dimension")) {
            CategoryFieldKey key = fromFullKey(dKey);
            if (!selectedDimensions.contains(key)) {
                JoinFieldInfo info = createJoinFieldInfo("dimension", dKey);
                if (info != null) response.getDimension().add(info);
            }
        }

        for (String mKey : recommendedMeasures) {
            CategoryFieldKey key = fromFullKey(mKey);
            if (!selectedMeasures.contains(key)) {
                JoinFieldInfo info = createJoinFieldInfo("measure", mKey);
                if (info != null) response.getMeasure().add(info);
            }
        }

        for (String fKey : filterUnion) {
            CategoryFieldKey key = fromFullKey(fKey);
            if (!selectedFilters.contains(key)) {
                JoinFieldInfo info = createJoinFieldInfo("filter", fKey);
                if (info != null) response.getFilter().add(info);
            }
        }
    }

    // -------------------------
    // 4. F 로직
    // -------------------------
    if (hasF) {
        for (String fKey : getAllFieldKeys("filter")) {
            CategoryFieldKey key = fromFullKey(fKey);
            if (!selectedFilters.contains(key)) {
                JoinFieldInfo info = createJoinFieldInfo("filter", fKey);
                if (info != null) response.getFilter().add(info);
            }
        }
    }

    return response;
}
```

---

### ✅ 보조 메서드 예시:

```java
private Set<String> getAllFieldKeys(String group) {
    // 예: YAML 전체 로딩 후 group별로 전체 키 리스트 뽑기
    return fieldConfigMap.keySet().stream()
            .filter(k -> k.startsWith(group + ":"))
            .collect(Collectors.toSet());
}
```

좋아, Daniel. 주어진 코드 조각은 `Map<String, CategoryConfig>` 구조를 기반으로 **Join Graph를 구축**하는 로직이야. 여기서 `fieldConfigMap`을 만들고 싶다면, 각 필드를 고유 키 (`group:category:field[:metric]`) 로 flatten 해서 저장하면 돼.

아래에 정리해줄게:

---

### ✅ 전제 구조 예시

```yaml
measure:
  device:
    fields:
      deviceModel:
        metric: sum
        joins:
          dimension:
            - target: dimension:device:manufacturer
```

Java에서는 이게 다음과 같이 매핑된다고 가정해:

```java
Map<String, CategoryConfig> map = ... // group 단위
// 예: map.get("measure") → { "device" : CategoryConfig(deviceModel, ...) }
```

---

### ✅ 목표: `Map<String, FieldConfig>` 형태 만들기

```java
Map<String, FieldConfig> fieldConfigMap = new HashMap<>();
```

---

### ✅ 구현 코드

```java
Map<String, FieldConfig> fieldConfigMap = new HashMap<>();

for (Map.Entry<String, CategoryConfig> categoryEntry : map.entrySet()) {
    String category = categoryEntry.getKey();
    CategoryConfig categoryConfig = categoryEntry.getValue();
    Map<String, FieldConfig> fields = categoryConfig.getFields();
    if (fields == null) continue;

    for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
        String fieldName = fieldEntry.getKey();
        FieldConfig fieldConfig = fieldEntry.getValue();

        // metric 이 존재하는 경우엔 같이 붙이고, 없으면 생략
        String metric = fieldConfig.getMetric();
        String key = metric == null
            ? String.format("%s:%s:%s", group, category, fieldName)
            : String.format("%s:%s:%s:%s", group, category, fieldName, metric);

        fieldConfigMap.put(key, fieldConfig);
    }
}
```

> 🔁 이 코드를 `group` 루프 안에서 실행해야 해 (group = "measure", "dimension", "filter" 등). 전체 구조는 아래 참고.

---

### ✅ 전체 구조 예시

```java
Map<String, Map<String, CategoryConfig>> allGroups = ...; // measure/dimension/filter

Map<String, FieldConfig> fieldConfigMap = new HashMap<>();

for (Map.Entry<String, Map<String, CategoryConfig>> groupEntry : allGroups.entrySet()) {
    String group = groupEntry.getKey(); // "measure", "dimension", "filter"
    Map<String, CategoryConfig> categoryMap = groupEntry.getValue();

    for (Map.Entry<String, CategoryConfig> categoryEntry : categoryMap.entrySet()) {
        String category = categoryEntry.getKey();
        CategoryConfig categoryConfig = categoryEntry.getValue();
        Map<String, FieldConfig> fields = categoryConfig.getFields();
        if (fields == null) continue;

        for (Map.Entry<String, FieldConfig> fieldEntry : fields.entrySet()) {
            String fieldName = fieldEntry.getKey();
            FieldConfig fieldConfig = fieldEntry.getValue();

            String metric = fieldConfig.getMetric();
            String key = metric == null
                ? String.format("%s:%s:%s", group, category, fieldName)
                : String.format("%s:%s:%s:%s", group, category, fieldName, metric);

            fieldConfigMap.put(key, fieldConfig);
        }
    }
}
```

---

### ✅ 보너스: `toKey` 함수 추출

```java
public static String toKey(String group, String category, String field, String metric) {
    return metric == null ? String.format("%s:%s:%s", group, category, field)
                          : String.format("%s:%s:%s:%s", group, category, field, metric);
}
```

필요하면 `FieldDescriptor` 같은 클래스로 키를 객체로 만들어도 돼.
지금처럼 단순한 맵이면 String 키로 충분하고 성능도 좋고 단순함.

---

필요하면 `fieldConfigMap`을 테스트할 수 있는 mock 데이터를 같이 만들어줄 수도 있어.
YAML 구조나 `CategoryConfig`, `FieldConfig` 구조 예제도 더 주면 정확히 맞춰줄 수 있어.



> `fieldConfigMap`은 `Map<String, FieldConfig>` 형태로 전체 필드 설정을 로딩해두는 전역 또는 의존성으로 두면 됨.

---

### ✨ 요약:

| 선택된 항목 | 추천되는 항목                     |
| ------ | --------------------------- |
| M 없음   | 전체 M                        |
| M 있음   | 나머지 M, M-D 교집합 D, M-F 합집합 F |
| D 있음   | 나머지 D, 관련 M, 관련 F           |
| F 있음   | 나머지 F                       |

---

필요하면 이 로직을 기준으로 더 모듈화하거나 YAML 예제까지 확장해줄게.
추가 요청이나 버그가 있으면 바로 말해줘.
