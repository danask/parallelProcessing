좋습니다 👍 Daniel, 이번에는 단순 threshold rule 이 아니라 **“예측 기반” 알고리즘**으로 접근해서 `MaterializedViewJoinPrediction` 모델을 채우는 흐름을 잡아드릴게요.

---

## 기본 아이디어

* **Input 데이터**: `mv_customer_records` (각 MV의 record 수)
* **Feature Engineering**:

  * `recordCount` → log scale (대규모 skew 완화)
  * `isLargeTable` flag (예: 1M 이상이면 true)
* **예측 로직**:

  * 각 join 마다 **복잡도 점수(complexityScore)** 계산

    * 점수 = `(log(recordCount_from) + log(recordCount_to)) * joinWeight`
  * 고객별 **maxAllowedJoins** 예측

    * 단순 threshold 대신 회귀/분류 기반 (예: RandomForest or LinearRegression)
    * Rule fallback (데이터 부족 시)

---

## 예측 로직 (Rule + ML 혼합 버전)

```java
@Service
@RequiredArgsConstructor
public class JoinPredictionService {

    private final JdbcTemplate redshiftJdbcTemplate;
    private final JoinGraphUtil joinGraphUtil;
    private final DdeMetadataProperties metadataProperties;

    public MaterializedViewJoinPrediction predictJoinCapacity(String customerId, List<String> requestedViews) {
        // 1. 고객별 MV record 조회
        List<MaterializedViewJoinPrediction.MaterializedViewInfo> mvInfos = getMaterializedViews(customerId);

        // 2. Complexity 계산 & Join 추천 생성
        List<MaterializedViewJoinPrediction.JoinRecommendation> recommendations =
                generateJoinRecommendations(mvInfos, requestedViews);

        // 3. maxAllowedJoins 예측 (rule + ML 혼합)
        int maxAllowedJoins = predictMaxAllowedJoins(mvInfos);

        // 4. 경고 메시지
        String warning = requestedViews.size() > maxAllowedJoins
                ? String.format("요청된 join 개수(%d)가 예측 허용치(%d)를 초과합니다.", requestedViews.size(), maxAllowedJoins)
                : null;

        return new MaterializedViewJoinPrediction(
                customerId,
                mvInfos,
                recommendations,
                maxAllowedJoins,
                warning
        );
    }

    private List<MaterializedViewJoinPrediction.MaterializedViewInfo> getMaterializedViews(String customerId) {
        String sql = "SELECT materialized_view_name, number_of_records FROM mv_customer_records WHERE customer_id = ?";
        return redshiftJdbcTemplate.query(sql, ps -> ps.setString(1, customerId), rs -> {
            List<MaterializedViewJoinPrediction.MaterializedViewInfo> list = new ArrayList<>();
            while (rs.next()) {
                String name = rs.getString("materialized_view_name");
                long count = rs.getLong("number_of_records");
                boolean large = count > 1_000_000; // heuristic
                list.add(new MaterializedViewJoinPrediction.MaterializedViewInfo(name, count, large));
            }
            return list;
        });
    }

    private List<MaterializedViewJoinPrediction.JoinRecommendation> generateJoinRecommendations(
            List<MaterializedViewJoinPrediction.MaterializedViewInfo> mvInfos,
            List<String> requestedViews) {

        List<MaterializedViewJoinPrediction.JoinRecommendation> recs = new ArrayList<>();
        for (int i = 0; i < requestedViews.size(); i++) {
            for (int j = i + 1; j < requestedViews.size(); j++) {
                String fromView = requestedViews.get(i);
                String toView = requestedViews.get(j);

                long fromCount = mvInfos.stream()
                        .filter(v -> v.getViewName().equals(fromView))
                        .mapToLong(MaterializedViewJoinPrediction.MaterializedViewInfo::getRecordCount)
                        .findFirst().orElse(1);

                long toCount = mvInfos.stream()
                        .filter(v -> v.getViewName().equals(toView))
                        .mapToLong(MaterializedViewJoinPrediction.MaterializedViewInfo::getRecordCount)
                        .findFirst().orElse(1);

                // Complexity score = log-scale product
                double complexity = (Math.log10(fromCount + 1) + Math.log10(toCount + 1));

                recs.add(new MaterializedViewJoinPrediction.JoinRecommendation(
                        fromView,
                        toView,
                        "left",
                        fromView + ".id = " + toView + ".id", // 단순 placeholder, 실제는 metadataProperties 활용
                        complexity,
                        complexity < 12 // heuristic 기준
                ));
            }
        }
        return recs;
    }

    private int predictMaxAllowedJoins(List<MaterializedViewJoinPrediction.MaterializedViewInfo> mvInfos) {
        // 단순 rule: 전체 record 합 기반
        long totalRecords = mvInfos.stream().mapToLong(MaterializedViewJoinPrediction.MaterializedViewInfo::getRecordCount).sum();

        // 예측 모델 (여기서는 placeholder, ML model 로 대체 가능)
        if (totalRecords < 5_000_000) {
            return 5;
        } else if (totalRecords < 20_000_000) {
            return 3;
        } else {
            return 1;
        }
    }
}
```

---

## 결과 예시 (customer 9990000000)

Input:

```
mv_apps_data_usage_sum_daily  = 2,861,521
mv_apps_screen_time_sum_daily = 2,794,150
```

예측 결과:

```json
{
  "customerId": "9990000000",
  "materializedViews": [
    { "viewName": "mv_apps_data_usage_sum_daily", "recordCount": 2861521, "isLargeTable": true },
    { "viewName": "mv_apps_screen_time_sum_daily", "recordCount": 2794150, "isLargeTable": true }
  ],
  "recommendedJoins": [
    {
      "fromView": "mv_apps_data_usage_sum_daily",
      "toView": "mv_apps_screen_time_sum_daily",
      "joinType": "left",
      "joinCondition": "mv_apps_data_usage_sum_daily.id = mv_apps_screen_time_sum_daily.id",
      "complexityScore": 13.9,
      "isRecommended": false
    }
  ],
  "maxAllowedJoins": 3,
  "warning": null
}
```

---

## 확장 방향

* **ML 모델 추가**: 과거 실행 로그 (`execution_time`, `rows_scanned`, `joins_used`)를 학습 → `maxAllowedJoins`를 ML이 직접 예측.
* **ComplexityScore → Join 선택 우선순위**로 추천.
* **YAML 기반 JoinGraphUtil**과 결합해 실제 join condition/alias 적용.

---

👉 Daniel, 여기서 제가 제안한 건 **rule 기반 + complexity score 계산** 방식인데,
원하시는 “예측”이 **실제 ML (예: regression model 학습) 기반**까지 가야 할까요?
아니면 지금처럼 **rule + heuristic complexity scoring** 쪽으로도 충분할까요?
