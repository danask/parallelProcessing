
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
