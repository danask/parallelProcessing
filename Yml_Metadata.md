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
// DTO for metric info
public class MetricConfig {
    private Map<String, String> metric = new HashMap<>();

    // getters and setters
}

// DTO for each field
public class FieldConfig {
    private String label;
    private Map<String, String> metric = new HashMap<>();

    // getters and setters
}

// DTO for each category
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields = new HashMap<>();

    // getters and setters
}

// Main configuration class
@Component
@ConfigurationProperties(prefix = "dde")
public class DdeMetadataProperties {
    private Map<String, Map<String, CategoryConfig>> dimension = new HashMap<>();
    private Map<String, Map<String, CategoryConfig>> measure = new HashMap<>();

    // Get group map based on group name
    private Map<String, CategoryConfig> getGroupMap(String group) {
        return switch (group.toLowerCase()) {
            case "dimension" -> dimension;
            case "measure" -> measure;
            default -> throw new IllegalArgumentException("Unknown group: " + group);
        };
    }

    public List<String> getCategoryLabels(String group) {
        return new ArrayList<>(getGroupMap(group).values().stream()
                .map(CategoryConfig::getLabel).toList());
    }

    public Optional<String> getCategoryByLabel(String group, String label) {
        return getGroupMap(group).entrySet().stream()
            .filter(e -> label.equals(e.getValue().getLabel()))
            .map(Map.Entry::getKey)
            .findFirst();
    }

    public Optional<String> getFieldNameByLabel(String group, String category, String label) {
        Map<String, FieldConfig> fields = Optional.ofNullable(getGroupMap(group).get(category))
            .map(CategoryConfig::getFields)
            .orElse(Map.of());

        return fields.entrySet().stream()
            .filter(e -> label.equals(e.getValue().getLabel()))
            .map(Map.Entry::getKey)
            .findFirst();
    }

    public Optional<String> getFieldLabel(String group, String category, String fieldName) {
        return Optional.ofNullable(getGroupMap(group).get(category))
            .map(CategoryConfig::getFields)
            .map(f -> f.get(fieldName))
            .map(FieldConfig::getLabel);
    }

    public Optional<FieldConfig> getFieldConfig(String group, String category, String fieldName) {
        return Optional.ofNullable(getGroupMap(group).get(category))
            .map(CategoryConfig::getFields)
            .map(f -> f.get(fieldName));
    }

    public Map<String, FieldConfig> getFields(String group, String category) {
        return Optional.ofNullable(getGroupMap(group).get(category))
            .map(CategoryConfig::getFields)
            .orElse(Map.of());
    }

    // Group-less search by label
    public Optional<String> findCategoryByLabelAcrossGroups(String label) {
        for (var group : List.of("dimension", "measure")) {
            var result = getCategoryByLabel(group, label);
            if (result.isPresent()) return result;
        }
        return Optional.empty();
    }

    public Optional<String> findFieldNameAcrossGroups(String label) {
        for (var group : List.of("dimension", "measure")) {
            var groupMap = getGroupMap(group);
            for (var category : groupMap.keySet()) {
                var result = getFieldNameByLabel(group, category, label);
                if (result.isPresent()) return result;
            }
        }
        return Optional.empty();
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
