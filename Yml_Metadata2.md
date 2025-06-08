

```java
좋습니다. 정리해주신 요구사항(1\~5)을 기준으로, 현재 시스템 구조에 맞게 **추천 로직을 M → D → F 순서로 리팩토링**하되, 다음 요구사항을 먼저 반영했습니다:

---

## ✅ 적용된 요구사항 정리 (1\~3번 중심):

### **1. 구조: Measure → Dimension → Filter 순서 유지**

* 추천 흐름은 다음과 같습니다:

  1. `Measure` 기준으로 Dimension 교집합 및 Filter 합집합 계산
  2. 선택된 Dimension에 기반해 Measure 추천
  3. Filter는 M이나 D에서 연결된 대상들을 union 하여 추천

---

### **2. Unit 이 다른 경우 추천에서 제외**

* `fieldConfig.getMetric()`을 조회할 때 unit 기준이 다르면 추천에서 제외합니다.

```java
private boolean isSameUnit(FieldConfig a, FieldConfig b) {
    return a.getMetric().entrySet().stream().anyMatch(aMetric ->
        b.getMetric().entrySet().stream().anyMatch(bMetric ->
            Objects.equals(aMetric.getValue().getUnit(), bMetric.getValue().getUnit())));
}
```

---

### **3. Filter는 항상 Measure/Dimension의 Join 타겟 기준으로 추천**

* `Filter` 추천은 Measure/Dimension가 선언한 join 정보 기준으로만 계산되며,
* YAML에서 선언된 filter join 정보만 사용되고, filter 자체 정의는 사용하지 않습니다.

---

## 🔄 개선된 `getJoinRecommendations()` 개요 (1\~3 적용됨)

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
                addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, key));
                recommendedMeasureKeys.add(key);
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
            Set<String> relatedMeasures = new HashSet<>();

            for (String dimKey : selectedDimensionKeys) {
                FieldConfig field = getFieldConfig(dimKey);
                if (field == null) continue;

                getJoinTargets(field, REPORT_MEASURE).forEach(j -> relatedMeasures.add(j.getTarget()));
                getJoinTargets(field, REPORT_FILTER).forEach(j -> recommendedFilterKeys.add(j.getTarget()));
            }

            for (String key : relatedMeasures) {
                if (!selectedMeasureKeys.contains(key)) {
                    addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, key));
                    recommendedMeasureKeys.add(key);
                }
            }

            for (String key : getAllFieldKeys(REPORT_DIMENSION)) {
                if (!selectedDimensionKeys.contains(key)) {
                    addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, key));
                    recommendedDimensionKeys.add(key);
                }
            }

            // 선택된 차원 제외
            response.getDimension().removeIf(info ->
                    selectedDimensionKeys.contains(toFullKey(REPORT_DIMENSION,
                            new CategoryFieldKey(info.getCategory(), info.getField(), info.getMetric()))));
        }

        // --- FILTER ---
        // F = 자기 자신 + 추천된 M + 추천된 D 기준 추천, 중복 제거
//        Set<String> totalFilterCandidates = new HashSet<>();
//        Stream.concat(selectedMeasureKeys.stream(), selectedDimensionKeys.stream())
//                .filter(key -> !selectedFilterKeys.contains(key))
//                .forEach(key -> addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, key)));


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
        // 모든 key를 metric이 있으면 :까지 포함해서, 없으면 마지막 : 붙여서 통일
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

    private static final String REPORT_MEASURE = "measure";
    private static final String REPORT_DIMENSION = "dimension";
    private static final String REPORT_FILTER = "filter";
}


```

---

## ✅ Helper 메소드

```java
private List<JoinConfig> getJoinTargets(FieldConfig field, String joinType) {
    return Optional.ofNullable(field.getJoins())
        .map(map -> map.get(joinType))
        .orElse(Collections.emptyList());
}

private boolean isCompatibleUnit(String candidateKey, Set<String> selectedMeasureKeys) {
    FieldConfig candidate = getFieldConfig(candidateKey);
    if (candidate == null) return false;

    for (String selectedKey : selectedMeasureKeys) {
        FieldConfig selected = getFieldConfig(selectedKey);
        if (selected == null || !isSameUnit(candidate, selected)) {
            return false;
        }
    }
    return true;
}

private boolean isSameUnit(FieldConfig a, FieldConfig b) {
    if (a == null || b == null) return false;

    return a.getMetric().entrySet().stream().anyMatch(aMetric ->
        b.getMetric().entrySet().stream().anyMatch(bMetric ->
            Objects.equals(
                aMetric.getValue().getUnit(),
                bMetric.getValue().getUnit()
            )
        )
    );
}



```
`Field` 객체에 unit 정보를 추가하려면 다음 두 가지를 처리해야 합니다:

---

### ✅ 1. `Field` 클래스에 `unit` 필드 추가

```java
public class Field {
    private String field;
    private String label;
    private Map<String, MetricConfig> metric;
    private String unit; // 🔹 추가

    // builder, getter, setter 등 정의 필요
}
```

---

### ✅ 2. `getCategoryAndFields` 수정 (unit 추출 추가)

아래와 같이 `unit`을 metric 정보에서 추출해서 `Field` 객체에 넣어주면 됩니다:

```java
public List<CategoryWithFields> getCategoryAndFields(String group) {
    Map<String, CategoryConfig> groupMap = getGroupMap(group);
    if (groupMap == null) return Collections.emptyList();

    return groupMap.entrySet().stream()
            .map(e -> {
                String category = e.getKey();
                CategoryConfig categoryConfig = e.getValue();
                List<Field> fields = categoryConfig.getFields() != null ?
                        categoryConfig.getFields().entrySet().stream()
                                .map(entry -> {
                                    FieldConfig fieldConfig = entry.getValue();
                                    // 🔹 unit 추출 (metric 여러 개인 경우 우선 하나만 표시하거나 가공)
                                    String unit = Optional.ofNullable(fieldConfig.getMetric())
                                            .flatMap(metrics -> metrics.values().stream()
                                                    .map(MetricConfig::getUnit)
                                                    .filter(Objects::nonNull)
                                                    .findFirst())
                                            .orElse(null);

                                    return Field.builder()
                                            .field(entry.getKey())
                                            .label(fieldConfig.getLabel())
                                            .metric(fieldConfig.getMetric())
                                            .unit(unit) // 🔹 추가
                                            .build();
                                }).toList()
                        : Collections.emptyList();
                return new CategoryWithFields(category, categoryConfig.getLabel(), fields);
            }).toList();
}
```

---

### 📝 유의사항

* `unit`을 metric 여러 개 중 대표 하나만 표시하고 싶을 경우 위 로직이 적절합니다.
* 만약 **metric별 unit 목록 전체**를 따로 보여줘야 한다면 `Map<String, String> metricToUnitMap` 형태로 빌드해야 합니다.

필요하면 그 방식도 알려드릴게요.

---

## 🔜 다음 단계 (4, 5번 항목)

### 4. `anrEvent`와 `fcEvent` 간 상호 연결 시 한쪽만 선언해도 다른 쪽 추천 가능하게 하기

* 현재는 `joins.measure` 양방향 선언이 필요하므로, 관계 정의만 한쪽에 있더라도 추천 가능하도록 `역추적`을 통해 파악해야 합니다.
* `reverse lookup map`을 만들어야 함 (예: `Map<String, List<String>> reverseMeasureJoinMap`).

### 5. Dimension의 join은 measure group에 이미 선언되어 있다면 따로 dimension group에서는 생략 가능

* `dimension` 필드의 `joins.dimension`이 비어 있더라도 `measure`에서 해당 필드를 참조하고 있다면 추천에 포함시켜야 함.

---

필요하시면 **4, 5번까지 완성한 전체 코드**도 이어서 드릴 수 있습니다. 먼저 1\~3번이 잘 동작하는지 확인해보시고, 다음 작업 이어갈까요?


```

좋은 질문이에요. 요약해서 말씀드리면:

---

### ❓ 질문: 아래 조건 만족하나요?

> `if (!field.getOperator().isEmpty())` 조건으로 `filterKey`를 추가하는 방식이
> **같은 M 내에서 한쪽만 join 선언된 경우에도 filter로 포함**되나요?

---

### 📌 답변: **그 조건만으로는 포함되지 않습니다.**

해당 로직은 단지 `operator`가 정의되어 있는 필드를 기준으로 filterKey를 만들 뿐이고,
**join 선언이 한쪽에만 있어도 자동으로 포함되게 하려면** 다음이 필요합니다:

---

### ✅ 해결책: **역방향 관계를 반영한 필터 자동 포함 로직 필요**

현재 코드는 `reverseJoinMap`을 사용해서 D → M, D → F 관계를 복구하지만,
**M → F (또는 F → M), D → F 중에서 한쪽만 선언된 경우**를 보완하려면
아래와 같이 filter 후보군을 구성하는 로직도 `reverseJoinMap` 기반으로 확장해야 합니다.

---

### ✅ 예시: filterUnion 구성 시, 정방향 + 역방향 둘 다 포함

```java
Set<String> filterUnion = new HashSet<>();

// 1. M → F 정방향 Join 기반 필터 추가
for (CategoryFieldKey k : selectedMeasures) {
    String fullKey = toFullKey(REPORT_MEASURE, k);
    FieldConfig field = fieldConfigMap.get(fullKey);
    if (field == null) continue;

    getJoinList(field, REPORT_FILTER).stream()
        .map(JoinConfig::getTarget)
        .forEach(filterUnion::add);

    // 2. 역방향 F → M 기반 필터도 포함
    reverseJoinMap.getOrDefault(fullKey, Set.of()).forEach(target -> {
        if (target.startsWith(REPORT_FILTER + ":")) {
            filterUnion.add(target);
        }
    });

    // 3. operator 기반 직접 필터 키도 포함
    if (!field.getOperator().isEmpty()) {
        filterUnion.add(String.format("%s:%s:%s:", REPORT_FILTER, k.getCategory(), k.getField()));
    }
}
```

---

### ✨ 이렇게 하면

* filter의 join 선언이 **양방향 중 한 쪽에만 있어도 후보로 포함**
* `operator` 기반으로 직접 정의한 filter 필드도 자동 포함
* 결국 **M, D에 정의된 필드가 자동으로 filter 후보가 됨**

---

필요하시면 이 방식 반영해서 `JoinGraphHelper.java` 업데이트해드릴까요?


아래는 수정된 `JoinGraphHelper.java` 전체 코드입니다.
`FieldConfig` 내 `metric` 구조에 맞게 `unit` 비교가 `MetricConfig` 단위로 정확히 이루어지도록 반영했습니다.

```java
import java.util.*;
import java.util.stream.Collectors;

public class JoinGraphHelper {

    private final Map<String, FieldConfig> fieldConfigMap;
    private final Map<String, Set<String>> reverseJoinMap;

    public JoinGraphHelper(Map<String, FieldConfig> fieldConfigMap) {
        this.fieldConfigMap = fieldConfigMap;
        this.reverseJoinMap = buildReverseJoinMap(fieldConfigMap);
    }

    public JoinRecommendationResponse getJoinRecommendations(
            Set<CategoryFieldKey> selectedMeasures,
            Set<CategoryFieldKey> selectedDimensions,
            Set<CategoryFieldKey> selectedFilters
    ) {
        JoinRecommendationResponse response = new JoinRecommendationResponse();

        boolean hasM = !selectedMeasures.isEmpty();
        boolean hasD = !selectedDimensions.isEmpty();
        boolean hasF = !selectedFilters.isEmpty();

        Set<String> selectedMeasureKeys = selectedMeasures.stream()
                .map(k -> toFullKey(REPORT_MEASURE, k))
                .collect(Collectors.toSet());
        Set<String> selectedDimensionKeys = selectedDimensions.stream()
                .map(k -> toFullKey(REPORT_DIMENSION, k))
                .collect(Collectors.toSet());
        Set<String> selectedFilterKeys = selectedFilters.stream()
                .map(k -> toFullKey(REPORT_FILTER, k))
                .collect(Collectors.toSet());

        Set<String> allMKeys = getAllFieldKeys(REPORT_MEASURE);
        Set<String> allDKeys = getAllFieldKeys(REPORT_DIMENSION);
        Set<String> allFKeys = getAllFieldKeys(REPORT_FILTER);

        // 선택된 measure이 없으면 기본적으로 measure 리스트 보여주기
        if (!hasM && !hasD && !hasF) {
            allMKeys.forEach(k -> addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, k)));
            return response;
        }

        if (hasM) {
            Set<String> dimensionIntersection = null;
            Set<String> filterUnion = new HashSet<>();

            for (CategoryFieldKey k : selectedMeasures) {
                String fullKey = toFullKey(REPORT_MEASURE, k);
                FieldConfig field = fieldConfigMap.get(truncateMetric(fullKey));
                if (field == null) continue;

                List<JoinConfig> dimJoins = getJoinList(field, REPORT_DIMENSION);
                List<JoinConfig> filterJoins = getJoinList(field, REPORT_FILTER);

                Set<String> dimTargets = dimJoins.stream()
                        .map(JoinConfig::getTarget)
                        .filter(target -> isSameUnit(fullKey, target))
                        .collect(Collectors.toSet());
                if (dimensionIntersection == null) dimensionIntersection = new HashSet<>(dimTargets);
                else dimensionIntersection.retainAll(dimTargets);

                filterJoins.stream().map(JoinConfig::getTarget).forEach(filterUnion::add);
            }

            allMKeys.stream().filter(k -> !selectedMeasureKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, k)));
            if (dimensionIntersection != null) {
                dimensionIntersection.stream().filter(k -> !selectedDimensionKeys.contains(k))
                        .forEach(k -> addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, k)));
            }
            filterUnion.stream().filter(k -> !selectedFilterKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, k)));
        }

        if (hasD) {
            Set<String> measureUnion = new HashSet<>();
            Set<String> filterUnion = new HashSet<>();

            for (CategoryFieldKey k : selectedDimensions) {
                String fullKey = toFullKey(REPORT_DIMENSION, k);
                FieldConfig field = fieldConfigMap.get(truncateMetric(fullKey));
                if (field == null) continue;

                getJoinList(field, REPORT_MEASURE).stream()
                        .map(JoinConfig::getTarget)
                        .filter(target -> isSameUnit(fullKey, target))
                        .forEach(measureUnion::add);
                getJoinList(field, REPORT_FILTER).stream().map(JoinConfig::getTarget).forEach(filterUnion::add);

                reverseJoinMap.getOrDefault(fullKey, Set.of()).forEach(target -> {
                    if (target.startsWith(REPORT_MEASURE + ":") && isSameUnit(fullKey, target)) measureUnion.add(target);
                    else if (target.startsWith(REPORT_FILTER + ":")) filterUnion.add(target);
                });
            }

            allDKeys.stream().filter(k -> !selectedDimensionKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, k)));

            // 선택된 차원 제외
            response.getDimension().removeIf(info ->
                    selectedDimensionKeys.contains(toFullKey(REPORT_DIMENSION,
                            new CategoryFieldKey(info.getCategory(), info.getField(), info.getMetric()))));

            measureUnion.stream().filter(k -> !selectedMeasureKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, k)));

            filterUnion.stream().filter(k -> !selectedFilterKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, k)));
        }

        if (hasF) {
            allFKeys.stream().filter(k -> !selectedFilterKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, k)));
        }

        return response;
    }

    /**
     * Metric 단위가 unit이 같은지 비교
     */
private boolean isSameUnit(String sourceKey, String targetKey) {
    String[] sourceParts = sourceKey.split(":");
    String[] targetParts = targetKey.split(":");

    // 최소한 group, category, field, metric이 있어야 함
    if (sourceParts.length < 4 || targetParts.length < 4) {
        return true;  // metric 없으면 단위 비교 안 함
    }

    String sourceGroup = sourceParts[0];
    String sourceCategory = sourceParts[1];
    String sourceField = sourceParts[2];
    String sourceMetric = sourceParts[3];

    String targetGroup = targetParts[0];
    String targetCategory = targetParts[1];
    String targetField = targetParts[2];
    String targetMetric = targetParts[3];

    // metric 단위 비교
    FieldConfig sourceFieldConfig = fieldConfigMap.get(String.format("%s:%s:%s:", sourceGroup, sourceCategory, sourceField));
    FieldConfig targetFieldConfig = fieldConfigMap.get(String.format("%s:%s:%s:", targetGroup, targetCategory, targetField));

    if (sourceFieldConfig == null || targetFieldConfig == null) return false;

    MetricConfig sourceMetricConfig = sourceFieldConfig.getMetric().get(sourceMetric);
    MetricConfig targetMetricConfig = targetFieldConfig.getMetric().get(targetMetric);

    if (sourceMetricConfig == null || targetMetricConfig == null) return false;

    return Objects.equals(sourceMetricConfig.getUnit(), targetMetricConfig.getUnit());
}


    private List<JoinConfig> getJoinList(FieldConfig field, String group) {
        return Optional.ofNullable(field.getJoins())
                .map(j -> j.get(group)).orElse(List.of());
    }

    private void addIfNotNull(List<JoinFieldInfo> list, JoinFieldInfo info) {
        if (info != null) list.add(info);
    }

    private Map<String, Set<String>> buildReverseJoinMap(Map<String, FieldConfig> configMap) {
        Map<String, Set<String>> reverseMap = new HashMap<>();
        for (Map.Entry<String, FieldConfig> entry : configMap.entrySet()) {
            String sourceKey = entry.getKey();
            FieldConfig config = entry.getValue();
            if (config.getJoins() != null) {
                for (Map.Entry<String, List<JoinConfig>> joinGroup : config.getJoins().entrySet()) {
                    for (JoinConfig jc : joinGroup.getValue()) {
                        reverseMap.computeIfAbsent(jc.getTarget(), k -> new HashSet<>()).add(sourceKey);
                    }
                }
            }
        }
        return reverseMap;
    }

    private String toFullKey(String group, CategoryFieldKey k) {
        return k.getMetric() == null ?
                String.format("%s:%s:%s:", group, k.getCategory(), k.getField()) :
                String.format("%s:%s:%s:%s", group, k.getCategory(), k.getField(), k.getMetric());
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

    // fullKey에서 metric 제거한 key로 FieldConfig 조회 시 사용
    private String truncateMetric(String fullKey) {
        String[] parts = fullKey.split(":");
        if (parts.length == 4) {
            return String.format("%s:%s:%s:", parts[0], parts[1], parts[2]);
        }
        return fullKey;
    }

    private static final String REPORT_MEASURE = "measure";
    private static final String REPORT_DIMENSION = "dimension";
    private static final String REPORT_FILTER = "filter";
}
```

---

### 함께 제공되는 DTO 등 기본 클래스 예시

```java
public class FieldConfig {
    private String label;
    private Map<String, MetricConfig> metric = new HashMap<>();
    private Map<String, String> operator = new HashMap<>();
    private Map<String, List<JoinConfig>> joins = new HashMap<>();

    // getters & setters
}

public class MetricConfig {
    private String label;
    private String name;
    private String unit;

    // getters & setters
}

public class JoinConfig {
    private String target;

    // getters & setters
}

public class JoinRecommendationResponse {
    private List<JoinFieldInfo> measure = new ArrayList<>();
    private List<JoinFieldInfo> dimension = new ArrayList<>();
    private List<JoinFieldInfo> filter = new ArrayList<>();

    // getters & setters
}

public class JoinFieldInfo {
    private String category;
    private String field;
    private String metric;

    public JoinFieldInfo(String category, String field, String metric) {
        this.category = category;
        this.field = field;
        this.metric = metric;
    }

    // getters & setters
}

public class CategoryFieldKey {
    private String category;
    private String field;
    private String metric;

    public CategoryFieldKey(String category, String field, String metric) {
        this.category = category;
        this.field = field;
        this.metric = metric;
    }

    // getters & setters
}
```

---

필요하면 위 클래스들도 함께 묶어 파일로 드릴 수 있습니다.
원하시면 말씀해주세요!



``` java
// --- JoinGraphHelper.java ---

public class JoinGraphHelper {

    private final Map<String, FieldConfig> fieldConfigMap;
    private final Map<String, Set<String>> reverseJoinMap;

    public JoinGraphHelper(Map<String, FieldConfig> fieldConfigMap) {
        this.fieldConfigMap = fieldConfigMap;
        this.reverseJoinMap = buildReverseJoinMap(fieldConfigMap);
    }

    public JoinRecommendationResponse getJoinRecommendations(
            Set<CategoryFieldKey> selectedMeasures,
            Set<CategoryFieldKey> selectedDimensions,
            Set<CategoryFieldKey> selectedFilters
    ) {
        JoinRecommendationResponse response = new JoinRecommendationResponse();

        boolean hasM = !selectedMeasures.isEmpty();
        boolean hasD = !selectedDimensions.isEmpty();
        boolean hasF = !selectedFilters.isEmpty();

        Set<String> selectedMeasureKeys = selectedMeasures.stream()
                .map(k -> toFullKey(REPORT_MEASURE, k))
                .collect(Collectors.toSet());
        Set<String> selectedDimensionKeys = selectedDimensions.stream()
                .map(k -> toFullKey(REPORT_DIMENSION, k))
                .collect(Collectors.toSet());
        Set<String> selectedFilterKeys = selectedFilters.stream()
                .map(k -> toFullKey(REPORT_FILTER, k))
                .collect(Collectors.toSet());

        Set<String> allMKeys = getAllFieldKeys(REPORT_MEASURE);
        Set<String> allDKeys = getAllFieldKeys(REPORT_DIMENSION);
        Set<String> allFKeys = getAllFieldKeys(REPORT_FILTER);

        if (hasM) {
            Set<String> dimensionIntersection = null;
            Set<String> filterUnion = new HashSet<>();

            for (CategoryFieldKey k : selectedMeasures) {
                String fullKey = toFullKey(REPORT_MEASURE, k);
                FieldConfig field = fieldConfigMap.get(fullKey);
                if (field == null) continue;

                List<JoinConfig> dimJoins = getJoinList(field, REPORT_DIMENSION);
                List<JoinConfig> filterJoins = getJoinList(field, REPORT_FILTER);

                Set<String> dimTargets = dimJoins.stream()
                        .map(JoinConfig::getTarget)
                        .filter(target -> isSameUnit(fullKey, target))
                        .collect(Collectors.toSet());
                if (dimensionIntersection == null) dimensionIntersection = new HashSet<>(dimTargets);
                else dimensionIntersection.retainAll(dimTargets);

                filterJoins.stream().map(JoinConfig::getTarget).forEach(filterUnion::add);
            }

            allMKeys.stream().filter(k -> !selectedMeasureKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, k)));
            if (dimensionIntersection != null) {
                dimensionIntersection.stream().filter(k -> !selectedDimensionKeys.contains(k))
                        .forEach(k -> addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, k)));
            }
            filterUnion.stream().filter(k -> !selectedFilterKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, k)));
        }

        if (hasD) {
            Set<String> measureUnion = new HashSet<>();
            Set<String> filterUnion = new HashSet<>();

            for (CategoryFieldKey k : selectedDimensions) {
                String fullKey = toFullKey(REPORT_DIMENSION, k);
                FieldConfig field = fieldConfigMap.get(fullKey);
                if (field == null) continue;

                getJoinList(field, REPORT_MEASURE).stream()
                        .map(JoinConfig::getTarget)
                        .filter(target -> isSameUnit(fullKey, target))
                        .forEach(measureUnion::add);
                getJoinList(field, REPORT_FILTER).stream().map(JoinConfig::getTarget).forEach(filterUnion::add);

                reverseJoinMap.getOrDefault(fullKey, Set.of()).forEach(target -> {
                    if (target.startsWith(REPORT_MEASURE + ":") && isSameUnit(fullKey, target)) measureUnion.add(target);
                    else if (target.startsWith(REPORT_FILTER + ":")) filterUnion.add(target);
                });
            }

            allDKeys.stream().filter(k -> !selectedDimensionKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getDimension(), createJoinFieldInfo(REPORT_DIMENSION, k)));

            response.getDimension().removeIf(info ->
                    selectedDimensionKeys.contains(toFullKey(REPORT_DIMENSION,
                            new CategoryFieldKey(info.getCategory(), info.getField(), info.getMetric()))));

            measureUnion.stream().filter(k -> !selectedMeasureKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, k)));

            filterUnion.stream().filter(k -> !selectedFilterKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, k)));
        }

        if (hasF) {
            allFKeys.stream().filter(k -> !selectedFilterKeys.contains(k))
                    .forEach(k -> addIfNotNull(response.getFilter(), createJoinFieldInfo(REPORT_FILTER, k)));
        }

        if (!hasM && !hasD && !hasF) {
            allMKeys.forEach(k -> addIfNotNull(response.getMeasure(), createJoinFieldInfo(REPORT_MEASURE, k)));
        }

        return response;
    }

    private boolean isSameUnit(String sourceKey, String targetKey) {
        FieldConfig source = fieldConfigMap.get(sourceKey);
        FieldConfig target = fieldConfigMap.get(targetKey);
        if (source == null || target == null) return false;
        return Objects.equals(source.getUnit(), target.getUnit());
    }

    private List<JoinConfig> getJoinList(FieldConfig field, String group) {
        return Optional.ofNullable(field.getJoins())
                .map(j -> j.get(group)).orElse(List.of());
    }

    private void addIfNotNull(List<JoinFieldInfo> list, JoinFieldInfo info) {
        if (info != null) list.add(info);
    }

    private Map<String, Set<String>> buildReverseJoinMap(Map<String, FieldConfig> configMap) {
        Map<String, Set<String>> reverseMap = new HashMap<>();
        for (Map.Entry<String, FieldConfig> entry : configMap.entrySet()) {
            String sourceKey = entry.getKey();
            FieldConfig config = entry.getValue();
            if (config.getJoins() != null) {
                for (Map.Entry<String, List<JoinConfig>> joinGroup : config.getJoins().entrySet()) {
                    for (JoinConfig jc : joinGroup.getValue()) {
                        reverseMap.computeIfAbsent(jc.getTarget(), k -> new HashSet<>()).add(sourceKey);
                    }
                }
            }
        }
        return reverseMap;
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

    private static final String REPORT_MEASURE = "measure";
    private static final String REPORT_DIMENSION = "dimension";
    private static final String REPORT_FILTER = "filter";
}


```
```java
// --- JoinGraphHelper.java ---

public class JoinGraphHelper {

    private final Map<String, FieldConfig> fieldConfigMap;
    private final Map<String, Set<String>> reverseJoinMap;

    public JoinGraphHelper(Map<String, FieldConfig> fieldConfigMap) {
        this.fieldConfigMap = fieldConfigMap;
        this.reverseJoinMap = buildReverseJoinMap(fieldConfigMap);
    }

    public JoinRecommendationResponse getJoinRecommendations(
            Set<CategoryFieldKey> selectedMeasures,
            Set<CategoryFieldKey> selectedDimensions,
            Set<CategoryFieldKey> selectedFilters
    ) {
        JoinRecommendationResponse response = new JoinRecommendationResponse();

        boolean hasM = !selectedMeasures.isEmpty();
        boolean hasD = !selectedDimensions.isEmpty();
        boolean hasF = !selectedFilters.isEmpty();

        Set<String> selectedMeasureKeys = selectedMeasures.stream()
                .map(k -> toFullKey(REPORT_MEASURE, k))
                .collect(Collectors.toSet());
        Set<String> selectedDimensionKeys = selectedDimensions.stream()
                .map(k -> toFullKey(REPORT_DIMENSION, k))
                .collect(Collectors.toSet());
        Set<String> selectedFilterKeys = selectedFilters.stream()
                .map(k -> toFullKey(REPORT_FILTER, k))
                .collect(Collectors.toSet());

        // measure
        Set<String> allMKeys = getAllFieldKeys(REPORT_MEASURE);
        if (!hasM) {
            for (String measureKey : allMKeys) {
                JoinFieldInfo info = createJoinFieldInfo(REPORT_MEASURE, measureKey);
                if (info != null) response.getMeasure().add(info);
            }
        } else {
            Set<String> dimensionIntersection = null;
            Set<String> filterUnion = new HashSet<>();

            for (CategoryFieldKey measureKey : selectedMeasures) {
                String fullKey = toFullKey(REPORT_MEASURE, measureKey);
                FieldConfig field = fieldConfigMap.get(fullKey);
                if (field == null) continue;

                List<JoinConfig> dimJoins = Optional.ofNullable(field.getJoins())
                        .map(j -> j.get(REPORT_DIMENSION)).orElse(List.of());
                List<JoinConfig> filterJoins = Optional.ofNullable(field.getJoins())
                        .map(j -> j.get(REPORT_FILTER)).orElse(List.of());

                Set<String> dimTargets = dimJoins.stream().map(JoinConfig::getTarget).collect(Collectors.toSet());
                if (dimensionIntersection == null) {
                    dimensionIntersection = new HashSet<>(dimTargets);
                } else {
                    dimensionIntersection.retainAll(dimTargets);
                }

                filterJoins.stream().map(JoinConfig::getTarget).forEach(filterUnion::add);
            }

            for (String measureKey : allMKeys) {
                if (!selectedMeasureKeys.contains(measureKey)) {
                    JoinFieldInfo info = createJoinFieldInfo(REPORT_MEASURE, measureKey);
                    if (info != null) response.getMeasure().add(info);
                }
            }

            if (dimensionIntersection != null) {
                for (String dimensionKey : dimensionIntersection) {
                    if (!selectedDimensionKeys.contains(dimensionKey)) {
                        JoinFieldInfo info = createJoinFieldInfo(REPORT_DIMENSION, dimensionKey);
                        if (info != null) response.getDimension().add(info);
                    }
                }
            }

            for (String filterKey : filterUnion) {
                if (!selectedFilterKeys.contains(filterKey)) {
                    JoinFieldInfo info = createJoinFieldInfo(REPORT_FILTER, filterKey);
                    if (info != null) response.getFilter().add(info);
                }
            }
        }

        // dimension
        if (hasD) {
            Set<String> recommendedMeasures = new HashSet<>();
            Set<String> filterUnion = new HashSet<>();

            for (CategoryFieldKey dimensionKey : selectedDimensions) {
                String fullKey = toFullKey(REPORT_DIMENSION, dimensionKey);
                FieldConfig field = fieldConfigMap.get(fullKey);
                if (field == null) continue;

                List<JoinConfig> measureJoins = Optional.ofNullable(field.getJoins())
                        .map(j -> j.get(REPORT_MEASURE)).orElse(List.of());
                List<JoinConfig> filterJoins = Optional.ofNullable(field.getJoins())
                        .map(j -> j.get(REPORT_FILTER)).orElse(List.of());

                measureJoins.stream().map(JoinConfig::getTarget).forEach(recommendedMeasures::add);
                filterJoins.stream().map(JoinConfig::getTarget).forEach(filterUnion::add);

                // 역참조 처리
                reverseJoinMap.getOrDefault(fullKey, Set.of()).stream()
                        .filter(k -> k.startsWith(REPORT_MEASURE + ":"))
                        .forEach(recommendedMeasures::add);
                reverseJoinMap.getOrDefault(fullKey, Set.of()).stream()
                        .filter(k -> k.startsWith(REPORT_FILTER + ":"))
                        .forEach(filterUnion::add);
            }

            for (String dimensionKey : getAllFieldKeys(REPORT_DIMENSION)) {
                if (!selectedDimensionKeys.contains(dimensionKey)) {
                    JoinFieldInfo info = createJoinFieldInfo(REPORT_DIMENSION, dimensionKey);
                    if (info != null) response.getDimension().add(info);
                }
            }

            response.getDimension().removeIf(info ->
                    selectedDimensionKeys.contains(toFullKey(REPORT_DIMENSION,
                            new CategoryFieldKey(info.getCategory(), info.getField(), info.getMetric())))
            );

            for (String measureKey : recommendedMeasures) {
                if (!selectedMeasureKeys.contains(measureKey)) {
                    JoinFieldInfo info = createJoinFieldInfo(REPORT_MEASURE, measureKey);
                    if (info != null) response.getMeasure().add(info);
                }
            }

            for (String filterKey : filterUnion) {
                if (!selectedFilterKeys.contains(filterKey)) {
                    JoinFieldInfo info = createJoinFieldInfo(REPORT_FILTER, filterKey);
                    if (info != null) response.getFilter().add(info);
                }
            }
        }

        // filter
        if (hasF) {
            for (String filterKey : getAllFieldKeys(REPORT_FILTER)) {
                if (!selectedFilterKeys.contains(filterKey)) {
                    JoinFieldInfo info = createJoinFieldInfo(REPORT_FILTER, filterKey);
                    if (info != null) response.getFilter().add(info);
                }
            }
        }

        return response;
    }

    private Map<String, Set<String>> buildReverseJoinMap(Map<String, FieldConfig> configMap) {
        Map<String, Set<String>> reverseMap = new HashMap<>();
        for (Map.Entry<String, FieldConfig> entry : configMap.entrySet()) {
            String sourceKey = entry.getKey();
            FieldConfig config = entry.getValue();
            if (config.getJoins() != null) {
                for (Map.Entry<String, List<JoinConfig>> joinGroup : config.getJoins().entrySet()) {
                    for (JoinConfig jc : joinGroup.getValue()) {
                        reverseMap.computeIfAbsent(jc.getTarget(), k -> new HashSet<>()).add(sourceKey);
                    }
                }
            }
        }
        return reverseMap;
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

    private static final String REPORT_MEASURE = "measure";
    private static final String REPORT_DIMENSION = "dimension";
    private static final String REPORT_FILTER = "filter";
}


```

----

좋습니다. 아래 구조에서 기존 로직에서 무엇이 변경되어야 하는지 단계적으로 분석해 보겠습니다:

---

## 📌 변경된 YAML 구조 요약

```yaml
dde:
  dimension:
    device:
      fields:
        deviceId:
          metric: 
             sum: 
               label: "Sum"
               dbName: "sum_device_id"
               unit: "count"   ← ✅ 여기 추가됨!
```

즉, 기존에는 `metric`이 단일 레벨이거나 없었지만, 이제는 **metric 하위에 단위 정보(`unit`)까지 있는 다단계 구조**가 되었습니다.

또한 join 대상도 다음과 같이 **metric 경로를 포함**합니다:

```yaml
joins:
  measure:
    - target: "measure:device:deviceCount:sum:percent"
```

---

## ✅ 1. 영향 받는 구조/타입: `FieldConfig`, `MetricConfig`, `JoinConfig`

### 기존:

```java
class FieldConfig {
    String label;
    Map<String, JoinConfig> joins;
    ...
}
```

### 변경 필요:

```java
class FieldConfig {
    String label;
    Map<String, JoinConfig> joins;
    Map<String, MetricConfig> metric; // now nested by function (e.g., sum, avg)
    ...
}

class MetricConfig {
    String label;
    String dbName;
    String unit; // ← 여기 추가
}
```

---

## ✅ 2. 영향 받는 로직: `toFullKey(...)`, `createJoinFieldInfo(...)`, 추천 알고리즘

### 이전에는

`group:category:field` 또는 `group:category:field:metricKey`까지를 fullKey로 사용했다면,

---

### 이제는 `group:category:field:metricKey:unit` 또는 더 정확히는:

```plaintext
measure:device:deviceCount:sum:percent
```

* 여기서 `sum`은 **aggregation function**
* `percent`는 **unit or metric variation**

---

## ✅ 변경이 필요한 주요 포인트

### 1. `toFullKey(...)` 로직

기존에 `group:category:field`까지 처리했다면, 이제 **metric key와 unit까지 포함된 구조**로 확장해야 합니다.

```java
String toFullKey(String type, CategoryFieldKey key) {
    if (key.getMetricKey() != null && key.getUnit() != null) {
        return String.join(":", type, key.getGroup(), key.getCategory(), key.getField(), key.getMetricKey(), key.getUnit());
    } else if (key.getMetricKey() != null) {
        return String.join(":", type, key.getGroup(), key.getCategory(), key.getField(), key.getMetricKey());
    } else {
        return String.join(":", type, key.getGroup(), key.getCategory(), key.getField());
    }
}
```

> `CategoryFieldKey`에 `metricKey`, `unit` 필드가 없다면 추가 필요합니다.

---

### 2. `createJoinFieldInfo(...)` 수정

YAML에서 join target이 `"measure:device:deviceCount:sum:percent"`처럼 상세 경로를 가지므로,
`createJoinFieldInfo(...)`도 이에 맞게 **파싱**하여 구조체를 생성해야 합니다.

```java
JoinFieldInfo createJoinFieldInfo(String type, String fullKey) {
    String[] parts = fullKey.split(":");
    if (parts.length < 4) return null;

    JoinFieldInfo info = new JoinFieldInfo();
    info.setType(type);
    info.setGroup(parts[1]);
    info.setCategory(parts[2]);
    info.setField(parts[3]);

    if (parts.length >= 5) info.setMetricKey(parts[4]);
    if (parts.length >= 6) info.setUnit(parts[5]);

    return info;
}
```

---

### 3. 추천 알고리즘

추천 로직에서 `Set<String>`으로 `selectedMKeys`, `selectedDKeys` 등을 구성할 때에도,
**이제 fullKey에 metricKey/unit이 포함되어야 하므로** 다음처럼 확장됩니다:

```java
Set<String> selectedMKeys = selectedMeasures.stream()
    .map(k -> toFullKey("measure", k)) // metricKey, unit 포함
    .collect(Collectors.toSet());
```

또한 추천 결과 리스트를 만들 때 `JoinFieldInfo`에 `metricKey`, `unit`을 넣는 것도 필요합니다.

---

### 4. 정렬 (필드 순서 정렬 시)

기존 `fieldOrderMap`도 이제 다음처럼 변경해야 합니다:

```java
String fullKey = String.join(":", groupName, categoryName, fieldName, metricKey, unit);
fieldOrderMap.put(fullKey, order++);
```

> 혹은 metricKey, unit이 없는 필드는 생략한 shorter version을 처리해야 함.

---

## ✅ 결론

| 항목                         | 변경 필요 여부 | 변경 내용 요약                                          |
| -------------------------- | -------- | ------------------------------------------------- |
| `FieldConfig`              | ✅        | `Map<String, MetricConfig> metric` 및 `unit` 필드 추가 |
| `CategoryFieldKey`         | ✅        | `metricKey`, `unit` 필드 추가                         |
| `toFullKey(...)`           | ✅        | metricKey + unit 포함                               |
| `createJoinFieldInfo(...)` | ✅        | `parts.length >= 6`까지 파싱                          |
| 추천 알고리즘                    | ✅        | fullKey 기준 연산 확장                                  |
| 정렬 기준 (`fieldOrderMap`)    | ✅        | fullKey 확장 기반으로 키 구성                              |

---

필요하시면 `FieldConfig`, `CategoryFieldKey`, `JoinFieldInfo`, `createJoinFieldInfo`, 추천 정렬 전체 예시를 리팩토링된 코드로 만들어드릴게요.


------

좋은 질문입니다.
\*\*"YAML에 join 관계를 한쪽만 적어서 처리할 수 있느냐"\*\*는 질문은 결국 다음과 같은 질문입니다:

> 양방향 join 정보를 다 적지 않고, 한쪽에만 join 정보를 정의한 후 나머지는 코드로 역추적하거나 역추론할 수 있느냐?

---

## ✅ 가능 여부: **Yes, 충분히 가능**합니다.

### ✔ 핵심 아이디어

* `measure → dimension`, `measure → filter` 관계만 정의되어 있으면,
* 코드에서 이를 **역방향으로 자동 구성**하여,

  * `dimension → measure`
  * `filter → measure` 같은 관계를 **자동 생성**하면 됩니다.

---

## 🧠 어떻게 처리할까?

YAML 구조가 다음과 같이 **한쪽만 정의**되어 있다고 가정합니다:

```yaml
groups:
  - name: user
    categories:
      - name: behavior
        fields:
          - name: active_user
            joins:
              dimension:
                - target: user_info.age
              filter:
                - target: user_info.country
```

위 예시는 `measure=user.behavior.active_user`가 dimension `user_info.age` 및 filter `user_info.country`와 join된다는 의미입니다.

---

## ✅ 1. 역조인 맵 생성

서비스 시작 시, **모든 필드에 대해 역조인 관계를 구성**해두면 됩니다:

```java
Map<String, Set<String>> reverseDimensionJoins = new HashMap<>();
Map<String, Set<String>> reverseFilterJoins = new HashMap<>();

for (GroupConfig group : ddeMetadataProperties.getGroups()) {
    for (CategoryConfig category : group.getCategories()) {
        for (FieldConfig field : category.getFields()) {
            String sourceKey = String.join(":", group.getName(), category.getName(), field.getName());

            if (field.getJoins() != null) {
                List<JoinConfig> dimJoins = field.getJoins().getOrDefault("dimension", List.of());
                List<JoinConfig> filterJoins = field.getJoins().getOrDefault("filter", List.of());

                for (JoinConfig dimJoin : dimJoins) {
                    reverseDimensionJoins
                        .computeIfAbsent(dimJoin.getTarget(), k -> new HashSet<>())
                        .add(sourceKey); // dimension → measure
                }

                for (JoinConfig filterJoin : filterJoins) {
                    reverseFilterJoins
                        .computeIfAbsent(filterJoin.getTarget(), k -> new HashSet<>())
                        .add(sourceKey); // filter → measure
                }
            }
        }
    }
}
```

이렇게 하면, 나중에 `dimension → measure`, `filter → measure` 추천도 무리 없이 됩니다.

---

## ✅ 2. 기존 추천 로직에 적용

예를 들어 dimension 추천 시:

```java
// dimension 선택 → 추천 measure
for (CategoryFieldKey dKey : selectedDimensions) {
    String fullKey = toFullKey("dimension", dKey);
    Set<String> recommendedMeasures = reverseDimensionJoins.getOrDefault(fullKey, Set.of());
    for (String mKey : recommendedMeasures) {
        if (!selectedMKeys.contains(mKey)) {
            JoinFieldInfo info = createJoinFieldInfo("measure", mKey);
            if (info != null) response.getMeasure().add(info);
        }
    }
}
```

filter → measure도 같은 방식으로 사용합니다.

---

## ✅ 장점

* YAML 정의량 절반 이하로 줄어듬
* 실수 방지 (양쪽에 다 정의하다 보면 sync 오류 생기기 쉬움)
* 코드만 잘 작성되면 확장성도 좋음

---

## ✅ 주의점

* 역조인 맵은 서비스 시작 시 한 번만 구축 (캐시 또는 빈 초기화 시점)
* `getJoinRecommendations()` 등 추천 로직에서는 역조인 맵만 보고 로직 구성
* `group:category:field` → fullKey가 정확히 맞도록 `JoinConfig.getTarget()`을 정의해야 함

---

## ✨ 결론

> **한쪽만 선언된 YAML 기반으로도 join 관계는 충분히 역추론하여 자동화할 수 있습니다.**

당신의 YAML 정의는 훨씬 간결해지고, 추천 로직은 더 안전하고 유지보수하기 쉬워집니다.

---

원하신다면 이 구조 전체를 코드와 함께 모듈화하는 예시 (`YAML 로딩 → 역조인 맵 → 추천 메서드 개선`) 도 단계별로 제공해드릴 수 있어요.

---

소팅

좋아요! 지금 작성한 `getJoinRecommendations(...)` 메서드에 **YAML 순서를 기준으로 정렬**을 결합하려면, 다음 3단계로 하면 됩니다.

---

## ✅ 1. 우선순위 맵 준비: `fieldOrderMap`

우선 `group:category:field` 구조의 순서를 유지하기 위한 `Map<String, Integer>`을 만들어야 합니다. 이건 외부에서 불러온 YAML 구조를 바탕으로 한 번만 생성하면 됩니다. 예를 들어, `DdeMetadataProperties`에서 불러왔다고 가정하면:

```java
public Map<String, Integer> getFieldOrderMap() {
    Map<String, Integer> fieldOrderMap = new HashMap<>();
    int order = 0;

    for (GroupConfig group : ddeMetadataProperties.getGroups()) {
        for (CategoryConfig category : group.getCategories()) {
            for (FieldConfig field : category.getFields()) {
                String fullKey = String.join(":", group.getName(), category.getName(), field.getName());
                fieldOrderMap.put(fullKey, order++);
            }
        }
    }

    return fieldOrderMap;
}
```

> 이 메서드는 `DdeMetadataService` 같은 곳에 두고 한 번만 만들고 재사용할 수 있어요.

---

## ✅ 2. 정렬 메서드 만들기

이건 `JoinFieldInfo` 리스트에 정렬을 적용하는 유틸 메서드입니다:

```java
private void sortJoinFieldInfos(List<JoinFieldInfo> infos, Map<String, Integer> fieldOrderMap) {
    infos.sort(Comparator.comparingInt(info -> {
        String fullKey = String.join(":", info.getGroup(), info.getCategory(), info.getField());
        return fieldOrderMap.getOrDefault(fullKey, Integer.MAX_VALUE);
    }));
}
```

---

## ✅ 3. `getJoinRecommendations`에 결합

이제 `response`의 measure/dimension/filter 리스트를 정렬하면 됩니다. 아래처럼 **가장 마지막**에 추가하세요:

```java
JoinRecommendationResponse response = new JoinRecommendationResponse();

// ... 기존 추천 로직 그대로 ...

Map<String, Integer> fieldOrderMap = getFieldOrderMap(); // YAML 기준 순서 맵
sortJoinFieldInfos(response.getMeasure(), fieldOrderMap);
sortJoinFieldInfos(response.getDimension(), fieldOrderMap);
sortJoinFieldInfos(response.getFilter(), fieldOrderMap);

return response;
```

---

## ✨ 최종적으로 정리하면

### 🔹 `getJoinRecommendations(...)` 끝부분:

```java
Map<String, Integer> fieldOrderMap = getFieldOrderMap(); // YAML 로딩 기준
sortJoinFieldInfos(response.getMeasure(), fieldOrderMap);
sortJoinFieldInfos(response.getDimension(), fieldOrderMap);
sortJoinFieldInfos(response.getFilter(), fieldOrderMap);

return response;
```

---

## 💡 팁

* `getFieldOrderMap()`은 매번 호출하지 않고, 서비스 시작 시 캐시해두는 것이 좋습니다.
* 혹시 `group`, `category`, `field`가 `JoinFieldInfo`에 없으면, 생성 시 `JoinFieldInfo`에 YAML 정보를 주입해야 합니다. (`createJoinFieldInfo` 수정 필요)

---

필요하면 `createJoinFieldInfo(...)`와 `JoinFieldInfo`도 확인해서 `group`, `category`, `field`가 들어가는지 같이 도와드릴게요!



-----
Daniel, 아주 중요한 부분을 짚었어.
**왜 `dimension`에 자기 자신이 이미 포함돼 있는지** 의심되는 상황인데, 그건 코드 흐름상 **Dimension을 처리하는 시점에 이미 자기 자신이 추천 리스트에 들어가 버렸기 때문**이야.

---

### 🔍 이유: "자기 자신"이 추천 결과에 포함되는 시점

아래 D 로직 부분을 봐봐:

```java
for (String dKey : getAllFieldKeys("dimension")) {
    CategoryFieldKey key = fromFullKey(dKey);
    if (!selectedDimensions.contains(key)) {
        JoinFieldInfo info = createJoinFieldInfo("dimension", dKey);
        if (info != null) response.getDimension().add(info);
    }
}
```

여기서 `selectedDimensions.contains(key)` 조건이 false 라고 판단되면,
→ `createJoinFieldInfo()` 통해 `response.getDimension()`에 들어가게 되는데,
**문제는 이 `contains` 판단이 오작동하는 경우야.**

---

### ⚠️ 원인: `CategoryFieldKey.equals()` / `hashCode()` 가 제대로 구현 안 됨

만약 `CategoryFieldKey` 클래스가 `equals()`와 `hashCode()`를 오버라이드하지 않았거나,
비교할 때 `metric`이 `null`인데 하나는 null, 하나는 빈 문자열이라든가,
구성 요소가 살짝 달라지면 **같은 키로 인식되지 않아**.

---

### ✅ 해결 방법

#### 1. `CategoryFieldKey`에 `equals()`와 `hashCode()`가 정확히 다음 기준으로 구현되어야 해:

```java
@Override
public boolean equals(Object o) {
    if (this == o) return true;
    if (o == null || getClass() != o.getClass()) return false;
    CategoryFieldKey that = (CategoryFieldKey) o;
    return Objects.equals(category, that.category) &&
           Objects.equals(field, that.field) &&
           Objects.equals(metric, that.metric);
}

@Override
public int hashCode() {
    return Objects.hash(category, field, metric);
}
```

#### 2. 그리고 비교 전에 **full key string**으로 바꾸어서 비교하면 더 안전해.

```java
Set<String> selectedDKeys = selectedDimensions.stream()
    .map(d -> toFullKey("dimension", d))
    .collect(Collectors.toSet());
```

그 다음 이걸로 비교:

```java
for (String dKey : getAllFieldKeys("dimension")) {
    if (!selectedDKeys.contains(dKey)) {
        JoinFieldInfo info = createJoinFieldInfo("dimension", dKey);
        if (info != null) response.getDimension().add(info);
    }
}
```

---

### ✳️ 추가 보완 (중복 방지 안전장치)

위에 더해서 마지막에 안전하게 중복 제거하는 필터도 추가하면 좋아:

```java
response.getDimension().removeIf(info ->
    selectedDKeys.contains(toFullKey("dimension",
        new CategoryFieldKey(info.getCategory(), info.getField(), info.getMetric())))
);
```

---

### ✅ 결론

* 자기 자신이 포함되는 이유는 `contains()` 판단이 제대로 안 되는 경우
* `CategoryFieldKey`가 `equals/hashCode` 잘못됐거나
* 비교 시 `metric` null/empty 불일치 등이 원인
* 안전하게 하려면 `toFullKey()` 기반으로 비교하자

---

필요하면 `toFullKey(info)`처럼 `JoinFieldInfo → fullKey`도 만들어줄게.
도와줄까?


---

좋아, Daniel. 아래는 `measure`, `dimension`, `filter` 모두에 대해 **자기 자신을 제외하는 로직**을 `toFullKey()` 기준으로 정리한 **최종 리팩토링된 `getJoinRecommendations()` 전체 코드**야.

---

### ✅ 리팩토링된 전체 코드

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

    // fullKey 기준으로 비교하기 위한 Set 생성
    Set<String> selectedMKeys = selectedMeasures.stream()
            .map(k -> toFullKey("measure", k))
            .collect(Collectors.toSet());
    Set<String> selectedDKeys = selectedDimensions.stream()
            .map(k -> toFullKey("dimension", k))
            .collect(Collectors.toSet());
    Set<String> selectedFKeys = selectedFilters.stream()
            .map(k -> toFullKey("filter", k))
            .collect(Collectors.toSet());

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
            if (!selectedMKeys.contains(mKey)) {
                JoinFieldInfo info = createJoinFieldInfo("measure", mKey);
                if (info != null) response.getMeasure().add(info);
            }
        }

        if (dimensionIntersection != null) {
            for (String dKey : dimensionIntersection) {
                if (!selectedDKeys.contains(dKey)) {
                    JoinFieldInfo info = createJoinFieldInfo("dimension", dKey);
                    if (info != null) response.getDimension().add(info);
                }
            }
        }

        for (String fKey : filterUnion) {
            if (!selectedFKeys.contains(fKey)) {
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
            if (!selectedDKeys.contains(dKey)) {
                JoinFieldInfo info = createJoinFieldInfo("dimension", dKey);
                if (info != null) response.getDimension().add(info);
            }
        }

        for (String mKey : recommendedMeasures) {
            if (!selectedMKeys.contains(mKey)) {
                JoinFieldInfo info = createJoinFieldInfo("measure", mKey);
                if (info != null) response.getMeasure().add(info);
            }
        }

        for (String fKey : filterUnion) {
            if (!selectedFKeys.contains(fKey)) {
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
            if (!selectedFKeys.contains(fKey)) {
                JoinFieldInfo info = createJoinFieldInfo("filter", fKey);
                if (info != null) response.getFilter().add(info);
            }
        }
    }

    return response;
}
```

---

### ✅ 핵심 변경 사항 요약

* `Set<CategoryFieldKey>` → `Set<String>`으로 변환 (full key 기준)
* 추천 제외 비교는 항상 `fullKey`로 비교
* `measure`, `dimension`, `filter` 모두 동일 방식 적용

---

필요하면 이 메서드와 함께 쓰이는 `toFullKey`, `fromFullKey`, `createJoinFieldInfo` 유틸들도 정리해줄게.


----

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
좋아, Daniel. 말씀해준 대로 **필터(F) 관련 추천 로직만** 키 비교 방식으로 **명확하게 자기 자신을 제외**하도록 수정해볼게.

---

## ✅ 기존 문제

```java
for (String fKey : getAllFieldKeys("filter")) {
    CategoryFieldKey key = fromFullKey(fKey);
    if (!selectedFilters.contains(key)) {
        ...
    }
}
```

* `selectedFilters.contains(key)`가 정확하게 비교 안 될 수 있음 → `metric` 차이 등
* 해결: `toFullKey()`로 만들어진 키 기준으로 직접 비교

---

## ✅ 수정된 필터 추천 로직 (선택된 필터가 있을 경우)

```java
if (hasF) {
    Set<String> selectedFKeys = selectedFilters.stream()
        .map(k -> toFullKey("filter", k))
        .collect(Collectors.toSet());

    for (String fKey : getAllFieldKeys("filter")) {
        if (!selectedFKeys.contains(fKey)) {
            JoinFieldInfo info = createJoinFieldInfo("filter", fKey);
            if (info != null) response.getFilter().add(info);
        }
    }
}
```

---

## ✅ 수정된 필터 추천 로직 (선택된 M 또는 D로부터 연관된 F 추천 시)

```java
Set<String> selectedFKeys = selectedFilters.stream()
    .map(k -> toFullKey("filter", k))
    .collect(Collectors.toSet());

for (String fKey : filterUnion) {  // 또는 filterUnion 을 생성한 부분
    if (!selectedFKeys.contains(fKey)) {
        JoinFieldInfo info = createJoinFieldInfo("filter", fKey);
        if (info != null) response.getFilter().add(info);
    }
}
```

---

이렇게 하면 필터 추천 목록에서 **이미 선택된 필터를 정확히 제외**할 수 있어.

---

필요하다면 `CategoryFieldKey`, `toFullKey`, `fromFullKey`의 정의도 같이 점검해줄게.
다음으로 어떤 걸 수정하고 싶은지 말해줘.

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
