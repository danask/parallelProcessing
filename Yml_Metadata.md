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
