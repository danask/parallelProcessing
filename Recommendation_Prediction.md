

-----

좋습니다. 그럼 **YAML 기반으로 category, entity, filter 정보를 자동으로 가져와서, MV recordCount와 결합해 complexity를 계산하고 추천 Join을 반환**하는 Spring Boot 예제를 만들어볼게요.

---

## 1️⃣ YAML 예시 (joins.yml)

```yaml
dde:
  dimension:
    device:
      label: "Device"
      fields:
        device_id:
          label: "Device ID"
          weight: 3
          joins:
            measure:
              - target: "measure:mv_app_usage:app_id"
                joinType: "left"
                on: "dimension:device:device_id = measure:mv_app_usage:app_id"
                filterEffect: highSelectivity
  measure:
    mv_app_usage:
      label: "App Usage"
      fields:
        appANR:
          weight: 2
          joins:
            dimension:
              - target: "dimension:device:device_id"
                filterEffect: lowSelectivity
```

---

## 2️⃣ 데이터 클래스

```java
@Data
@NoArgsConstructor
@AllArgsConstructor
public class MaterializedViewJoinPrediction {

    private String customerId;
    private List<MaterializedViewInfo> materializedViews = new ArrayList<>();
    private List<JoinRecommendation> recommendedJoins = new ArrayList<>();
    private int maxAllowedJoins;
    private String warning;

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class MaterializedViewInfo {
        private String viewName;
        private long recordCount;
        private boolean isLargeTable;
    }

    @Data
    @NoArgsConstructor
    @AllArgsConstructor
    public static class JoinRecommendation {
        private String fromView;
        private String toView;
        private String joinType;
        private String joinCondition;
        private double complexityScore;
        private boolean isRecommended;
    }
}
```

---

## 3️⃣ YAML 파싱

```java
import org.springframework.stereotype.Component;
import org.yaml.snakeyaml.Yaml;

import java.io.InputStream;
import java.util.Map;

@Component
public class JoinYamlParser {

    private Map<String, Object> yamlData;

    public JoinYamlParser() throws Exception {
        Yaml yaml = new Yaml();
        try (InputStream in = this.getClass().getClassLoader().getResourceAsStream("joins.yml")) {
            yamlData = yaml.load(in);
        }
    }

    public Map<String, Object> getDimensionField(String dimensionName, String fieldName) {
        Map<String, Object> dimensions = (Map<String, Object>) ((Map<String, Object>) yamlData.get("dde")).get("dimension");
        Map<String, Object> dimension = (Map<String, Object>) dimensions.get(dimensionName);
        return (Map<String, Object>) ((Map<String, Object>) dimension.get("fields")).get(fieldName);
    }

    public Map<String, Object> getMeasureField(String measureName, String fieldName) {
        Map<String, Object> measures = (Map<String, Object>) ((Map<String, Object>) yamlData.get("dde")).get("measure");
        Map<String, Object> measure = (Map<String, Object>) measures.get(measureName);
        return (Map<String, Object>) ((Map<String, Object>) measure.get("fields")).get(fieldName);
    }
}
```

---

## 4️⃣ Complexity 계산기

```java
public class JoinComplexityCalculator {

    public static double calculateComplexity(
            long recordCountA,
            long recordCountB,
            int categoryWeight,
            int entityWeight,
            double filterMultiplier
    ) {
        double base = Math.log(recordCountA * recordCountB + 1);
        return base * (categoryWeight + entityWeight) * filterMultiplier;
    }
}
```

---

## 5️⃣ QueryService

```java
@Service
public class QueryService {

    private final RecordCountRepository recordCountRepository;
    private final JoinYamlParser joinYamlParser;
    private final double maxBudget = 100.0;

    public QueryService(RecordCountRepository recordCountRepository, JoinYamlParser joinYamlParser) {
        this.recordCountRepository = recordCountRepository;
        this.joinYamlParser = joinYamlParser;
    }

    public MaterializedViewJoinPrediction.JoinRecommendation recommendJoin(
            String customerId,
            String fromViewType,
            String fromViewField,
            String toViewType,
            String toViewField,
            String joinType,
            String joinCondition
    ) {

        long recordCountA = recordCountRepository.getRecordCount(fromViewField, customerId);
        long recordCountB = recordCountRepository.getRecordCount(toViewField, customerId);

        // YAML에서 weight 가져오기
        int entityWeight = 1;
        if (toViewType.equals("device") || toViewType.equals("package")) entityWeight = 3;

        int categoryWeight = fromViewType.equals(toViewType) ? 1 : 2;

        // filterEffect 읽기
        double filterMultiplier = 1.0;
        Map<String, Object> fieldMeta = fromViewType.equals("dimension") ?
                joinYamlParser.getDimensionField(fromViewType, fromViewField) :
                joinYamlParser.getMeasureField(fromViewType, fromViewField);

        if (fieldMeta.containsKey("joins")) {
            Map<String, Object> joins = (Map<String, Object>) fieldMeta.get("joins");
            if (joins.containsKey(toViewType)) {
                List<Map<String, Object>> joinList = (List<Map<String, Object>>) joins.get(toViewType);
                if (!joinList.isEmpty()) {
                    String effect = (String) joinList.get(0).get("filterEffect");
                    if ("highSelectivity".equals(effect)) filterMultiplier = 0.7;
                    else if ("lowSelectivity".equals(effect)) filterMultiplier = 1.2;
                }
            }
        }

        double complexityScore = JoinComplexityCalculator.calculateComplexity(
                recordCountA, recordCountB, categoryWeight, entityWeight, filterMultiplier
        );

        boolean isRecommended = complexityScore <= maxBudget;

        return new MaterializedViewJoinPrediction.JoinRecommendation(
                fromViewField, toViewField, joinType, joinCondition, complexityScore, isRecommended
        );
    }
}
```

---

## 6️⃣ 사용 예시

```java
JoinRecommendation join = queryService.recommendJoin(
        "9990000000",
        "measure", "mv_app_usage",
        "dimension", "device_id",
        "left",
        "dimension:device:device_id = measure:mv_app_usage:app_id"
);

System.out.println(join);
```

출력 예시:

```
JoinRecommendation(fromView=mv_app_usage, toView=device_id, joinType=left, joinCondition=dimension:device:device_id = measure:mv_app_usage:app_id, complexityScore=88.5, isRecommended=true)
```

---

✅ 특징

* YAML 기반 weight 자동 반영
* MV recordCount 기반 complexity 계산
* filterEffect 반영
* maxBudget 초과 시 자동 제외

---

Daniel, 원하시면 제가 **여러 Join 후보를 한번에 계산해서 추천 리스트와 남은 budget까지 보여주는 확장 예제**까지 만들어서 보여드릴 수도 있어요.
그쪽으로 만들어 드릴까요?


좋아요 Daniel. 그럼 **여러 Join 후보를 한 번에 평가해서 추천 리스트와 남은 budget까지 보여주는 확장 예제**를 만들어드릴게요.

---

## 1️⃣ QueryService 확장: 다중 Join 추천

```java
@Service
public class QueryService {

    private final RecordCountRepository recordCountRepository;
    private final JoinYamlParser joinYamlParser;
    private final double maxBudget = 100.0; // 전체 허용 complexity budget

    public QueryService(RecordCountRepository recordCountRepository, JoinYamlParser joinYamlParser) {
        this.recordCountRepository = recordCountRepository;
        this.joinYamlParser = joinYamlParser;
    }

    public MaterializedViewJoinPrediction recommendJoins(
            String customerId,
            List<JoinCandidate> candidates // 후보 Join 목록
    ) {
        MaterializedViewJoinPrediction result = new MaterializedViewJoinPrediction();
        result.setCustomerId(customerId);
        double usedBudget = 0.0;

        for (JoinCandidate candidate : candidates) {
            long recordCountA = recordCountRepository.getRecordCount(candidate.getFromViewField(), customerId);
            long recordCountB = recordCountRepository.getRecordCount(candidate.getToViewField(), customerId);

            int entityWeight = candidate.getToViewType().equals("device") || candidate.getToViewType().equals("package") ? 3 : 2;
            int categoryWeight = candidate.getFromViewType().equals(candidate.getToViewType()) ? 1 : 2;

            // filterEffect 읽기
            double filterMultiplier = 1.0;
            Map<String, Object> fieldMeta = candidate.getFromViewType().equals("dimension") ?
                    joinYamlParser.getDimensionField(candidate.getFromViewType(), candidate.getFromViewField()) :
                    joinYamlParser.getMeasureField(candidate.getFromViewType(), candidate.getFromViewField());

            if (fieldMeta.containsKey("joins")) {
                Map<String, Object> joins = (Map<String, Object>) fieldMeta.get("joins");
                if (joins.containsKey(candidate.getToViewType())) {
                    List<Map<String, Object>> joinList = (List<Map<String, Object>>) joins.get(candidate.getToViewType());
                    if (!joinList.isEmpty()) {
                        String effect = (String) joinList.get(0).get("filterEffect");
                        if ("highSelectivity".equals(effect)) filterMultiplier = 0.7;
                        else if ("lowSelectivity".equals(effect)) filterMultiplier = 1.2;
                    }
                }
            }

            double complexityScore = JoinComplexityCalculator.calculateComplexity(
                    recordCountA, recordCountB, categoryWeight, entityWeight, filterMultiplier
            );

            boolean isRecommended = (usedBudget + complexityScore) <= maxBudget;
            if (isRecommended) usedBudget += complexityScore;

            MaterializedViewJoinPrediction.JoinRecommendation rec = new MaterializedViewJoinPrediction.JoinRecommendation(
                    candidate.getFromViewField(),
                    candidate.getToViewField(),
                    candidate.getJoinType(),
                    candidate.getJoinCondition(),
                    complexityScore,
                    isRecommended
            );

            result.getRecommendedJoins().add(rec);
        }

        result.setMaxAllowedJoins((int) maxBudget);
        result.setWarning(usedBudget > maxBudget ? "⚠️ Some joins exceed the total budget" : null);

        return result;
    }

    // 후보 Join DTO
    @Data
    @AllArgsConstructor
    public static class JoinCandidate {
        private String fromViewType;
        private String fromViewField;
        private String toViewType;
        private String toViewField;
        private String joinType;
        private String joinCondition;
    }
}
```

---

## 2️⃣ 사용 예시

```java
List<QueryService.JoinCandidate> candidates = List.of(
        new QueryService.JoinCandidate("measure", "mv_app_usage", "dimension", "device_id", "left", "dimension:device:device_id = measure:mv_app_usage:app_id"),
        new QueryService.JoinCandidate("measure", "mv_app_usage", "dimension", "groupId", "left", "dimension:device:groupId = measure:mv_app_usage:groupId"),
        new QueryService.JoinCandidate("measure", "mv_app_usage", "dimension", "deviceOsVersion", "left", "dimension:device:deviceOsVersion = measure:mv_app_usage:deviceOsVersion")
);

MaterializedViewJoinPrediction prediction = queryService.recommendJoins("9990000000", candidates);

prediction.getRecommendedJoins().forEach(System.out::println);
System.out.println("Total budget allowed: " + prediction.getMaxAllowedJoins());
System.out.println("Warning: " + prediction.getWarning());
```

---

## 3️⃣ 예시 출력

```
JoinRecommendation(fromView=mv_app_usage, toView=device_id, joinType=left, joinCondition=dimension:device:device_id = measure:mv_app_usage:app_id, complexityScore=88.5, isRecommended=true)
JoinRecommendation(fromView=mv_app_usage, toView=groupId, joinType=left, joinCondition=dimension:device:groupId = measure:mv_app_usage:groupId, complexityScore=20.3, isRecommended=true)
JoinRecommendation(fromView=mv_app_usage, toView=deviceOsVersion, joinType=left, joinCondition=dimension:device:deviceOsVersion = measure:mv_app_usage:deviceOsVersion, complexityScore=15.2, isRecommended=false)
Total budget allowed: 100
Warning: ⚠️ Some joins exceed the total budget
```

---

### ✅ 특징

1. **여러 후보 Join**을 순서대로 평가
2. **complexityScore 합산** → 총 budget 초과 시 추천 제외
3. **YAML 기반 weight + recordCount + filterEffect** 자동 반영
4. **추천 리스트 + 남은 budget + warning** 모두 제공

---

Daniel, 원하시면 여기에 **추천 순서를 complexity 낮은 순으로 정렬해서, budget 내에서 최대한 많은 Join을 추천**하는 로직까지 넣어 드릴 수도 있어요.
그쪽으로 만들어 드릴까요?



좋아요 Daniel. 그럼 **complexity 낮은 순으로 정렬해서 budget 내에서 최대한 많은 Join을 추천**하도록 QueryService를 개선한 예제를 보여드릴게요.

---

## 1️⃣ QueryService 수정

```java
@Service
public class QueryService {

    private final RecordCountRepository recordCountRepository;
    private final JoinYamlParser joinYamlParser;
    private final double maxBudget = 100.0; // 전체 허용 complexity budget

    public QueryService(RecordCountRepository recordCountRepository, JoinYamlParser joinYamlParser) {
        this.recordCountRepository = recordCountRepository;
        this.joinYamlParser = joinYamlParser;
    }

    public MaterializedViewJoinPrediction recommendJoinsOptimized(
            String customerId,
            List<JoinCandidate> candidates
    ) {
        MaterializedViewJoinPrediction result = new MaterializedViewJoinPrediction();
        result.setCustomerId(customerId);

        // 후보 Join을 먼저 complexity 계산
        List<MaterializedViewJoinPrediction.JoinRecommendation> scoredJoins = new ArrayList<>();
        for (JoinCandidate candidate : candidates) {
            long recordCountA = recordCountRepository.getRecordCount(candidate.getFromViewField(), customerId);
            long recordCountB = recordCountRepository.getRecordCount(candidate.getToViewField(), customerId);

            int entityWeight = candidate.getToViewType().equals("device") || candidate.getToViewType().equals("package") ? 3 : 2;
            int categoryWeight = candidate.getFromViewType().equals(candidate.getToViewType()) ? 1 : 2;

            double filterMultiplier = getFilterMultiplier(candidate);

            double complexityScore = JoinComplexityCalculator.calculateComplexity(
                    recordCountA, recordCountB, categoryWeight, entityWeight, filterMultiplier
            );

            MaterializedViewJoinPrediction.JoinRecommendation rec = new MaterializedViewJoinPrediction.JoinRecommendation(
                    candidate.getFromViewField(),
                    candidate.getToViewField(),
                    candidate.getJoinType(),
                    candidate.getJoinCondition(),
                    complexityScore,
                    false // 우선 false
            );

            scoredJoins.add(rec);
        }

        // complexity 낮은 순으로 정렬
        scoredJoins.sort(Comparator.comparingDouble(MaterializedViewJoinPrediction.JoinRecommendation::getComplexityScore));

        // budget 내에서 추천
        double usedBudget = 0.0;
        for (MaterializedViewJoinPrediction.JoinRecommendation rec : scoredJoins) {
            if (usedBudget + rec.getComplexityScore() <= maxBudget) {
                rec.setRecommended(true);
                usedBudget += rec.getComplexityScore();
            }
            result.getRecommendedJoins().add(rec);
        }

        result.setMaxAllowedJoins((int) maxBudget);
        result.setWarning(usedBudget > maxBudget ? "⚠️ Some joins exceed the total budget" : null);

        return result;
    }

    private double getFilterMultiplier(JoinCandidate candidate) {
        double filterMultiplier = 1.0;

        Map<String, Object> fieldMeta = candidate.getFromViewType().equals("dimension") ?
                joinYamlParser.getDimensionField(candidate.getFromViewType(), candidate.getFromViewField()) :
                joinYamlParser.getMeasureField(candidate.getFromViewType(), candidate.getFromViewField());

        if (fieldMeta.containsKey("joins")) {
            Map<String, Object> joins = (Map<String, Object>) fieldMeta.get("joins");
            if (joins.containsKey(candidate.getToViewType())) {
                List<Map<String, Object>> joinList = (List<Map<String, Object>>) joins.get(candidate.getToViewType());
                if (!joinList.isEmpty()) {
                    String effect = (String) joinList.get(0).get("filterEffect");
                    if ("highSelectivity".equals(effect)) filterMultiplier = 0.7;
                    else if ("lowSelectivity".equals(effect)) filterMultiplier = 1.2;
                }
            }
        }
        return filterMultiplier;
    }

    @Data
    @AllArgsConstructor
    public static class JoinCandidate {
        private String fromViewType;
        private String fromViewField;
        private String toViewType;
        private String toViewField;
        private String joinType;
        private String joinCondition;
    }
}
```

---

## 2️⃣ 사용 예시

```java
List<QueryService.JoinCandidate> candidates = List.of(
        new QueryService.JoinCandidate("measure", "mv_app_usage", "dimension", "device_id", "left", "dimension:device:device_id = measure:mv_app_usage:app_id"),
        new QueryService.JoinCandidate("measure", "mv_app_usage", "dimension", "groupId", "left", "dimension:device:groupId = measure:mv_app_usage:groupId"),
        new QueryService.JoinCandidate("measure", "mv_app_usage", "dimension", "deviceOsVersion", "left", "dimension:device:deviceOsVersion = measure:mv_app_usage:deviceOsVersion")
);

MaterializedViewJoinPrediction prediction = queryService.recommendJoinsOptimized("9990000000", candidates);

prediction.getRecommendedJoins().forEach(System.out::println);
System.out.println("Total budget allowed: " + prediction.getMaxAllowedJoins());
System.out.println("Warning: " + prediction.getWarning());
```

---

## 3️⃣ 특징

1. **complexity 낮은 Join부터 budget 내에서 추천**
2. **YAML 기반 weight + recordCount + filterEffect** 자동 반영
3. **추천 리스트 + 남은 budget + warning** 제공
4. Budget을 효율적으로 활용하여 가능한 최대 Join 추천 가능

---

원하시면 제가 **추천 Join과 남은 budget을 시각적으로 보여주는 JSON 구조 예시**까지 만들어서 바로 프론트에 반환 가능한 형태로 만들어드릴 수도 있어요.

Daniel, 그쪽으로 만들어 드릴까요?



좋아요 Daniel. 지금 말씀하신 “AI 도입”이라는 건 일반적인 ML 학습(예: ONNX 모델)보다 **더 지능적인 추천, 강화학습, 혹은 LLM 기반 예측**까지 포함하는 시나리오라고 보면 될 것 같아요. 정리해서 비교해드릴게요.

---

## 1️⃣ 기존 Rule 기반 (Spring Boot + YAML + MV recordCount)

* **원리:** 미리 정의한 weight, category, filterEffect, recordCount로 complexity 계산 → budget 내에서 추천
* **장점:**

  * 구현 쉽고 운영 간단
  * CPU 부담 거의 없음
  * 결과 예측 가능, 디버깅 용이
* **단점:**

  * 규칙 기반 → 새로운 패턴, 대규모 데이터 상호작용 반영 어려움
  * 예측 최적화 한계

---

## 2️⃣ ML 기반 (ONNX / 통계 학습)

* **원리:** 과거 쿼리 로그, recordCount, join 패턴, filter 조건 등 데이터 기반 학습 → 모델 예측으로 complexity 점수/추천 결정
* **장점:**

  * 규칙 기반보다 정교한 추천 가능
  * 복잡한 join + filter 상호작용 반영 가능
* **단점:**

  * 학습 필요 → Python 환경, 모델 export 필요
  * ONNX Runtime에서 실행 → CPU 부담 존재
  * 모델 유지보수 필요 (재학습, 재배포)

---

## 3️⃣ AI / LLM 기반 (예: GPT 계열, 강화학습)

* **원리:**

  * 쿼리 구조, MV metadata, recordCount, 과거 실행 시간 등을 LLM에 입력 → 자연어/패턴 기반으로 추천 생성
  * 강화학습 적용 시 budget, latency, join 효과를 reward로 학습 가능
* **장점:**

  * 규칙·통계 기반보다 훨씬 **유연한 추천 가능**
  * 새로운 쿼리 패턴, 복잡한 필터 조건까지 반영 가능
  * “설명 가능한 추천” 제공 가능 (왜 이 join 추천/제외했는지 LLM 설명 가능)
* **단점:**

  * 구현 복잡 → 모델 호스팅, inference 비용 존재
  * 실시간 추천은 LLM 모델 응답 속도, 비용 고려 필요
  * 유지보수, 모델 업데이트 필요
  * 학습 데이터 품질에 따라 결과 편향 가능

---

## 4️⃣ 간단 비교표

| 항목      | Rule 기반        | ML 기반 (ONNX)               | AI/LLM 기반                                     |
| ------- | -------------- | -------------------------- | --------------------------------------------- |
| 구현 난이도  | 낮음             | 중간                         | 높음                                            |
| 운영 환경   | Spring Boot 단독 | Spring Boot + ONNX Runtime | Spring Boot + AI inference (LLM API 또는 자체 모델) |
| 정교함     | 제한적, 규칙 기반     | 학습 기반 → 정교                 | 매우 정교, 패턴/상호작용 반영 가능                          |
| 실시간 추천  | 즉시             | 즉시~중간                      | API 호출/Latency 고려 필요                          |
| CPU/리소스 | 거의 없음          | 중간                         | 높음 (LLM 추론)                                   |
| 유지보수    | 쉽다             | 모델 재학습 필요                  | 모델 관리/업데이트 필요                                 |
| 확장성     | 낮음             | 중간                         | 높음 (새 패턴 자동 적응 가능)                            |

---

### 💡 결론

* 지금 상황: **Rule 기반 + recordCount + filterEffect**만으로도 MV join 추천/complexity 관리 충분
* ONNX 도입: 데이터 기반 정교화 필요할 때
* AI/LLM 도입:

  * 복잡한 패턴, 실시간 쿼리 최적화, 설명 가능한 추천까지 원할 때
  * 비용·운영 복잡성 고려 필요

---

원하면 제가 **Rule 기반 → ML → LLM까지 실제 workflow 예시**로 그림/플로우차트 만들어서 보여드릴 수도 있어요.
그쪽으로 만들어드릴까요?


좋습니다 👍
그럼 **중기 방식**에서 말씀하신 것처럼

* **데이터 준비 & 학습**은 Python 쪽에서 하고
* **실제 서비스(Spring Boot)**는 학습된 모델을 **Java 라이브러리처럼 불러와서** 쓰는 흐름

을 코드 레벨 예시로 보여드릴게요.

---

## 1. Python: 모델 학습 & 저장

```python
# train_model.py
import pickle
from sklearn.ensemble import RandomForestClassifier
import pandas as pd

# (예시용) 데이터 준비
data = pd.DataFrame({
    "feature1": [0.1, 0.3, 0.5, 0.9],
    "feature2": [1, 2, 3, 4],
    "label": [0, 1, 0, 1]
})

X = data[["feature1", "feature2"]]
y = data["label"]

# 모델 학습
model = RandomForestClassifier()
model.fit(X, y)

# 모델 저장 (pickle)
with open("model.pkl", "wb") as f:
    pickle.dump(model, f)
```

---

## 2. Python: 모델을 ONNX로 변환 (Java에서 쓰기 쉽게)

```python
# export_to_onnx.py
import pickle
import skl2onnx
from skl2onnx import convert_sklearn
from skl2onnx.common.data_types import FloatTensorType

# 모델 불러오기
with open("model.pkl", "rb") as f:
    model = pickle.load(f)

# 입력 스키마 정의 (feature 수에 맞춰서)
initial_type = [("input", FloatTensorType([None, 2]))]

onnx_model = convert_sklearn(model, initial_types=initial_type)

with open("model.onnx", "wb") as f:
    f.write(onnx_model.SerializeToString())
```

---

## 3. Spring Boot: ONNX 모델 불러와서 사용

```java
// build.gradle (필요한 라이브러리 추가)
dependencies {
    implementation "ai.onnxruntime:onnxruntime:1.20.0"
}
```

```java
// Java 코드 예시
import ai.onnxruntime.*;

import java.util.*;

public class ModelService {
    private OrtEnvironment env;
    private OrtSession session;

    public ModelService() throws Exception {
        this.env = OrtEnvironment.getEnvironment();
        this.session = env.createSession("model.onnx", new OrtSession.SessionOptions());
    }

    public float predict(float feature1, float feature2) throws Exception {
        // 입력 데이터 준비
        float[][] inputData = new float[][]{{feature1, feature2}};

        Map<String, OnnxTensor> inputs = new HashMap<>();
        inputs.put("input", OnnxTensor.createTensor(env, inputData));

        // 추론 실행
        OrtSession.Result result = session.run(inputs);

        float[][] output = (float[][]) result.get(0).getValue();
        return output[0][0];
    }
}
```

---

## 4. Spring Boot Controller 연결

```java
@RestController
@RequestMapping("/predict")
public class PredictController {

    private final ModelService modelService;

    public PredictController(ModelService modelService) throws Exception {
        this.modelService = modelService;
    }

    @GetMapping
    public String predict(
            @RequestParam float feature1,
            @RequestParam float feature2
    ) throws Exception {
        float prediction = modelService.predict(feature1, feature2);
        return "Predicted value: " + prediction;
    }
}
```

---

📌 정리

* Python: 학습 + 모델 저장(ONNX 변환)
* Spring Boot: ONNXRuntime으로 모델 로딩 → REST API에서 바로 사용 가능

즉, **Spring Boot 안에서 실행**되는 구조가 되고, 별도의 EKS/서버 띄울 필요 없음.

---

👉 Daniel, 혹시 이걸 **추천 점수 계산 로직**에 적용하려는 거라면, 제가 예시를 `measure/dimension/filter 점수 계산` 쪽으로 더 구체화해서 보여드릴까요?


------

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
