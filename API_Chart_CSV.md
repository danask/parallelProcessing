완벽해! `ResponseEntity<byte[]>`로 처리하면 CSV도 **세련되고 선언적인 방식**으로 응답 가능하고,
테스트, Swagger 문서화, 프록시 환경 등에서도 더 안정적이야.

---

## ✅ 최종 정리: CSV API using `ResponseEntity<byte[]>`

### 🔧 `/metrics/csv` – CSV 다운로드 API (ResponseEntity 버전)

```java
@GetMapping("/metrics/csv")
public ResponseEntity<byte[]> downloadCsv() throws IOException {
    List<DateEntry> structuredData = getMetricsJson().getBody(); // JSON API 재사용

    Set<String> dimensionKeys = new TreeSet<>();
    Set<String> measureKeys = new TreeSet<>();

    for (DateEntry entry : structuredData) {
        for (DataPoint dp : entry.data()) {
            if (dp.dimension() != null) {
                dimensionKeys.addAll(dp.dimension().keySet());
            }
            measureKeys.addAll(dp.measure().keySet());
        }
    }

    List<String> headers = new ArrayList<>();
    headers.add("날짜");
    headers.addAll(dimensionKeys);
    headers.addAll(measureKeys);

    ByteArrayOutputStream byteOut = new ByteArrayOutputStream();
    byteOut.write(new byte[]{(byte) 0xEF, (byte) 0xBB, (byte) 0xBF}); // UTF-8 BOM

    try (PrintWriter writer = new PrintWriter(new OutputStreamWriter(byteOut, StandardCharsets.UTF_8))) {
        writer.println(String.join(",", headers));

        for (DateEntry entry : structuredData) {
            for (DataPoint dp : entry.data()) {
                List<String> row = new ArrayList<>();
                row.add(entry.date());

                for (String key : dimensionKeys) {
                    Object val = dp.dimension() != null ? dp.dimension().get(key) : "";
                    row.add(escapeCsv(String.valueOf(val != null ? val : "")));
                }
                for (String key : measureKeys) {
                    Object val = dp.measure().get(key);
                    row.add(escapeCsv(String.valueOf(val != null ? val : "")));
                }

                writer.println(String.join(",", row));
            }
        }

        writer.flush();
    }

    byte[] csvBytes = byteOut.toByteArray();

    String fileName = URLEncoder.encode("차트_데이터.csv", StandardCharsets.UTF_8).replace("+", "%20");

    HttpHeaders headersHttp = new HttpHeaders();
    headersHttp.setContentType(MediaType.parseMediaType("text/csv; charset=UTF-8"));
    headersHttp.setContentLength(csvBytes.length);
    headersHttp.set(HttpHeaders.CONTENT_DISPOSITION, "attachment; filename*=UTF-8''" + fileName);

    return new ResponseEntity<>(csvBytes, headersHttp, HttpStatus.OK);
}
```

---

### 🔧 `escapeCsv` 함수 (동일)

```java
private String escapeCsv(String value) {
    if (value == null) return "";
    String escaped = value.replace("\"", "\"\"");
    if (escaped.contains(",") || escaped.contains("\"") || escaped.contains("\n")) {
        return "\"" + escaped + "\"";
    }
    return escaped;
}
```

---

## ✅ 요약

| 항목        | JSON API                          | CSV API                   |
| --------- | --------------------------------- | ------------------------- |
| 경로        | `/metrics/json`                   | `/metrics/csv`            |
| 응답 타입     | `ResponseEntity<List<DateEntry>>` | `ResponseEntity<byte[]>`  |
| 포맷        | 구조화된 날짜-디멘전-메저 JSON               | 엑셀 친화적 CSV                |
| UTF-8 BOM | ✖️ 필요 없음                          | ✅ 필요 (엑셀용)                |
| 테스트 편의    | ✅ `ResponseEntity`로 완전 지원         | ✅ `ResponseEntity`로 완전 지원 |

---

추가로:

* dimension/measure 필드를 `@RequestParam`으로 선택 가능하게
* 파일 이름 동적 설정 (`?fileName=...`)
* CSV → Excel(XLSX) 확장 (`Apache POI` 등)

이런 확장도 가능해. 필요하면 바로 붙여줄게!


----
```java
// === CsvConstants.java ===
public class CsvConstants {
    public static final String CSV_CONTENT_TYPE = "text/csv; charset=UTF-8";
    public static final byte[] UTF8_BOM = new byte[]{(byte) 0xEF, (byte) 0xBB, (byte) 0xBF};
    public static final String FILE_NAME = "차트_데이터.csv";
    public static final String CONTENT_DISPOSITION = "attachment; filename*=UTF-8''";
}


// === MetricsService.java ===
@Service
public class MetricsService {

    public List<DateEntry> getStructuredData(List<Map<String, Object>> flatData, Set<String> dimKeys, Set<String> metKeys) {
        Map<String, List<Map<String, Object>>> groupedByDate = flatData.stream()
                .collect(Collectors.groupingBy(row -> String.valueOf(row.get("date")),
                        TreeMap::new, Collectors.toList()));

        List<DateEntry> result = new ArrayList<>();

        for (Map.Entry<String, List<Map<String, Object>>> entry : groupedByDate.entrySet()) {
            String date = entry.getKey();
            List<Map<String, Object>> rows = entry.getValue();

            List<DataPoint> dataPoints = new ArrayList<>();

            for (Map<String, Object> row : rows) {
                Map<String, Object> dim = new LinkedHashMap<>();
                Map<String, Object> met = new LinkedHashMap<>();

                for (Map.Entry<String, Object> field : row.entrySet()) {
                    String key = field.getKey();
                    if ("date".equals(key)) continue;

                    if (dimKeys.contains(key)) {
                        dim.put(key, field.getValue());
                    } else if (metKeys.contains(key)) {
                        met.put(key, field.getValue());
                    }
                }

                dataPoints.add(new DataPoint(dim.isEmpty() ? null : dim, met));
            }

            result.add(new DateEntry(date, dataPoints, dimKeys.size(), metKeys.size()));
        }

        return result;
    }

    public byte[] generateCsvBytes(List<DateEntry> structuredData, Set<String> dimKeys, Set<String> metKeys) throws IOException {
        List<String> headers = new ArrayList<>();
        headers.add("날짜");
        headers.addAll(dimKeys);
        headers.addAll(metKeys);

        ByteArrayOutputStream out = new ByteArrayOutputStream();
        out.write(CsvConstants.UTF8_BOM);

        try (PrintWriter writer = new PrintWriter(new OutputStreamWriter(out, StandardCharsets.UTF_8))) {
            writer.println(String.join(",", headers));

            for (DateEntry entry : structuredData) {
                for (DataPoint dp : entry.data()) {
                    List<String> row = new ArrayList<>();
                    row.add(entry.date());
                    for (String key : dimKeys) {
                        Object val = dp.dimension() != null ? dp.dimension().get(key) : "";
                        row.add(escapeCsv(String.valueOf(val != null ? val : "")));
                    }
                    for (String key : metKeys) {
                        Object val = dp.measure().get(key);
                        row.add(escapeCsv(String.valueOf(val != null ? val : "")));
                    }
                    writer.println(String.join(",", row));
                }
            }
        }

        return out.toByteArray();
    }

    private String escapeCsv(String value) {
        if (value == null) return "";
        String escaped = value.replace("\"", "\"\"");
        if (escaped.contains(",") || escaped.contains("\"") || escaped.contains("\n")) {
            return "\"" + escaped + "\"";
        }
        return escaped;
    }

    public List<Map<String, Object>> getFlatData() {
        // 예시: DB 또는 다른 서비스에서 가져온 flat한 데이터
        return List.of(
            Map.of("date", "20250429", "appName", "Google", "packageName", "Google.com", "anrEvent", 774.68, "fcEvent", 123.45),
            Map.of("date", "20250429", "appName", "HP", "packageName", "HP.com", "anrEvent", 222.0)
        );
    }
}


// === MetricsController.java ===
@RestController
@RequestMapping("/metrics")
public class MetricsController {

    private final MetricsService metricsService;

    public MetricsController(MetricsService metricsService) {
        this.metricsService = metricsService;
    }

    @GetMapping("/json")
    public ResponseEntity<List<DateEntry>> getMetricsJson(
            @RequestParam Set<String> dimensions,
            @RequestParam Set<String> measures
    ) {
        List<Map<String, Object>> flatData = metricsService.getFlatData();
        List<DateEntry> structured = metricsService.getStructuredData(flatData, dimensions, measures);
        return ResponseEntity.ok(structured);
    }

    @GetMapping("/csv")
    public ResponseEntity<byte[]> downloadCsv(
            @RequestParam Set<String> dimensions,
            @RequestParam Set<String> measures
    ) throws IOException {
        List<Map<String, Object>> flatData = metricsService.getFlatData();
        List<DateEntry> structured = metricsService.getStructuredData(flatData, dimensions, measures);
        byte[] csvBytes = metricsService.generateCsvBytes(structured, dimensions, measures);

        String fileName = URLEncoder.encode(CsvConstants.FILE_NAME, StandardCharsets.UTF_8).replace("+", "%20");
        HttpHeaders headers = new HttpHeaders();
        headers.setContentType(MediaType.parseMediaType(CsvConstants.CSV_CONTENT_TYPE));
        headers.setContentLength(csvBytes.length);
        headers.set(HttpHeaders.CONTENT_DISPOSITION, CsvConstants.CONTENT_DISPOSITION + fileName);

        return new ResponseEntity<>(csvBytes, headers, HttpStatus.OK);
    }
}


// === DTOs ===
public record DataPoint(
    Map<String, Object> dimension,
    Map<String, Object> measure
) {}

public record DateEntry(
    String date,
    List<DataPoint> data,
    int numDimensions,
    int numMeasures
) {}

```
