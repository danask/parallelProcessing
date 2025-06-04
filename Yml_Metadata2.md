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
