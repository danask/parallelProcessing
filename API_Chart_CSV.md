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
