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

구조가 안정적이면서 유연해서, category와 group이 늘어나도 대응하기 쉬운 방식입니다.

원하시는 출력 포맷이나 추가 필터가 더 있을까요?


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
