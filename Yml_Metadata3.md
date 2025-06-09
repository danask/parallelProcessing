
좋습니다. 아래는 JoinGraphHelper#getJoinRecommendations 메서드의 전체 흐름을 해석, 의사코드, 도식화, 그리고 그래프 알고리즘과의 연관성 관점에서 발표용으로 정리한 내용입니다.

🔍 전체 구조 요약 

이 메서드는 다음 세 종류의 입력(선택된 항목)을 받아:

Set<CategoryFieldKey> selectedMeasures

Set<CategoryFieldKey> selectedDimensions

Set<CategoryFieldKey> selectedFilters

아래 항목을 추천 목록으로 반환합니다:

List<JoinFieldInfo> measure

List<JoinFieldInfo> dimension

List<JoinFieldInfo> filter

🔁 전반적 흐름 요약 (Flowchart) ┌─────────────┐ │ Input: M/D/F│ └─────┬───────┘ │ ▼ ┌───────────────────┐ │ measure 로직 │ │ - D 교집합 │ │ - filter union │ │ - join 관련 measure│ └─────┬─────────────┘ ▼ ┌────────────────────┐ │ dimension 로직 │ │ - 관련 measure 추천 │ │ - D 교집합 │ │ - filter union │ └─────┬──────────────┘ ▼ ┌────────────────────┐ │ filter 로직 │ │ - M/D 기준 조인 필터 │ └────────────────────┘ 🧠 내부 로직 해석 1. MEASURE 로직 

선택된 measure 없을 때: 전부 추천.

있을 때:

각 measure가 연결 가능한 dimension의 교집합 추출.

각 measure가 연결하는 filter를 union으로 수집.

다른 measure 중 조인 관계가 있고, 단위(단위 필터 포함)가 호환되면 추천.

⚙️ 그래프 해석:
measure들에서 dimension/filter로 향하는 outbound edge를 모으고, dimension은 intersection, filter는 union 집합으로 취합.

2. DIMENSION 로직 

선택된 dimension이 있을 때:

관련된 measure 추출 (dim ➝ measure)

관련된 filter 추출 (dim ➝ filter)

연결된 다른 dimension들과의 교집합 계산 (dim ➝ dim)

⚙️ 그래프 해석:
dimension 노드 기준으로 measure/filter/dimension에 대해 각각 outbound edge 추적.
그래프 탐색 중 intersection을 통해 dimension 간 공통 target을 계산.

3. FILTER 로직 

모든 선택된 measure + dimension에 대해 연결된 filter 수집 (filterFromJoins).

이 필터 중 아직 선택되지 않은 것만 추천.

⚙️ 그래프 해석:
measure/dimension 노드에서 filter로 향하는 단방향 edge 탐색.
기존 선택된 filter는 추천에서 제외.

🧩 주요 함수 요약 (역할 중심) 메서드 역할 getFieldConfig 특정 필드의 config 조회 getJoinTargets 해당 필드가 연결하는 그룹(join 대상) 조회 isMeasureJoinRelated measure 간 양방향 관계 확인 (그래프 edge 양방향 존재 여부) isCompatibleUnit 단위 호환성 확인 (측정 단위 기반 비교) addIfNotNull null 방지용 유틸리티 getAllFieldKeys 특정 그룹(measure, dimension 등)의 모든 필드 key 반환 🧮 의사코드 (Measure 로직 기준 예시) if selectedMeasures.isEmpty(): recommend all measures else: for m in selectedMeasures: dimTargets = m.joinTargets(dimension) if first measure: dimensionIntersection = dimTargets else: dimensionIntersection = dimensionIntersection ∩ dimTargets filterUnion += m.joinTargets(filter) for m2 in allMeasures: if m2 not in selected: if unit compatible and join exists: recommend m2 for d in dimensionIntersection: if d not in selected: recommend d for f in filterUnion: if f not in selected: recommend f 🔗 그래프 알고리즘 연관성 구성 요소 그래프 모델링 방식 FieldConfig Vertex (노드) JoinConfig Directed Edge (단방향 화살표) measure-dimension-filter 관계 Directed Graph 교집합 추천 (intersection) BFS/Set intersection 필터 union 추천 BFS + Set union 양방향 연결 체크 Edge(u→v) ∨ Edge(v→u) 

🎯 결국 이 시스템은 join graph를 구성한 뒤:

추천: 탐색 + 조건 필터링 (intersect/union)

구성 정보 최소화: 단방향 선언으로 양방향 추론 (isMeasureJoinRelated)

📈 시각 예 (간단한 예시 도식) measure:m1 │ ▼ dimension:d1 ←───── dimension:d2 │ │ ▼ │ filter:f1 measure:m2 

m1 → d1 → f1

d2 → m2

추천 시: m1 선택 시 → d1, f1 추천됨.

d2 선택 시 → m2 추천됨.

💡 발표 키포인트 요약 

YML 정의 기반 join graph를 구성해 추천

노드 간 join 관계는 Directed Graph로 추상화

교집합 & 유사 단위 비교 기반 추천은 전형적인 그래프 탐색 문제

그래프 선언을 한쪽만 해도 양방향 유추 가능하도록 설계 (isMeasureJoinRelated)

추천 로직은 "확장 가능한 쿼리 빌더 + 사용자 경험 중심 인터랙션"을 위한 핵심 엔진 역할

필요하시면 발표용 슬라이드 구조나 실제 도식 그래픽도 만들어드릴 수 있어요. 원하시나요?




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
