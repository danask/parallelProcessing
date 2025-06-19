
좋아, 그럼 `isMeasurePairJoinable` / `isDimensionPairJoinable`을 **쌍방 확인 포함** 형태로 안전하게 다시 만들어줄게.
이제는 **measure 간** 혹은 **dimension 간**에

* `m1 → m2` 또는 `m2 → m1`
* `d1 → d2` 또는 `d2 → d1`
  둘 중 하나라도 있으면 **joinable**하다고 판단해.

---

## ✅ `isMeasurePairJoinable` (쌍방 체크)

```java
private boolean isMeasurePairJoinable(Set<String> selectedMeasureKeys) {
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
