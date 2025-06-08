
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
