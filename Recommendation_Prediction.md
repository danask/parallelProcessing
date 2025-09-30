

좋아요. 그럼 **Spring Boot + Java에서 MV 레코드 수, join 수, filter 수를 기반으로 α, β를 단순 선형회귀로 추정하는 예제**를 보여드릴게요.

---

## 1️⃣ 가정

* `mv_customer_records`에서 각 MV record 수를 가져온다.
* 쿼리 실행 로그에서 join 수, filter 수, 실제 실행 시간(ms)을 가져온다.
* 단순화: 선형회귀 라이브러리는 **Apache Commons Math** 사용.

---

## 2️⃣ Maven 의존성

```xml
<dependency>
    <groupId>org.apache.commons</groupId>
    <artifactId>commons-math3</artifactId>
    <version>3.6.1</version>
</dependency>
```

---

## 3️⃣ Java 예제 코드

```java
import org.apache.commons.math3.stat.regression.OLSMultipleLinearRegression;
import java.util.*;

public class JoinScoreRegression {

    public static void main(String[] args) {
        // 1. 데이터 준비
        List<QueryStats> stats = List.of(
            new QueryStats(2, 1000, 1, 150),
            new QueryStats(3, 10000, 2, 800),
            new QueryStats(1, 500, 0, 50),
            new QueryStats(4, 100000, 3, 2000)
        );

        // 2. 회귀용 배열 변환
        double[][] x = new double[stats.size()][2]; // joinCount^2, log(recordCount+1)
        double[] y = new double[stats.size()];      // 실제 실행시간(ms)

        for (int i = 0; i < stats.size(); i++) {
            QueryStats s = stats.get(i);
            x[i][0] = Math.pow(s.getJoinCount(), 2);              // joinCount^2
            x[i][1] = Math.log(s.getRecordCount() + 1);          // log(recordCount+1)
            y[i] = s.getExecutionTimeMs();
        }

        // 3. OLS 선형회귀
        OLSMultipleLinearRegression regression = new OLSMultipleLinearRegression();
        regression.newSampleData(y, x);

        double[] beta = regression.estimateRegressionParameters(); 
        System.out.println("Intercept: " + beta[0]);
        System.out.println("α (joinCount^2 coefficient): " + beta[1]);
        System.out.println("β (log(recordCount) coefficient): " + beta[2]);
    }

    public static class QueryStats {
        private final int joinCount;
        private final long recordCount;
        private final int filterCount;
        private final long executionTimeMs;

        public QueryStats(int joinCount, long recordCount, int filterCount, long executionTimeMs) {
            this.joinCount = joinCount;
            this.recordCount = recordCount;
            this.filterCount = filterCount;
            this.executionTimeMs = executionTimeMs;
        }

        public int getJoinCount() { return joinCount; }
        public long getRecordCount() { return recordCount; }
        public int getFilterCount() { return filterCount; }
        public long getExecutionTimeMs() { return executionTimeMs; }
    }
}
```

---

## 4️⃣ 설명

1. `x[i][0]` → joinCount², `x[i][1]` → log(recordCount+1)
2. `y[i]` → 실제 쿼리 실행 시간(ms)
3. `OLSMultipleLinearRegression`로 회귀 수행 → α, β를 추정
4. 추정된 α, β를 기존 **TotalScore 계산** 공식에 적용 가능

---

이렇게 하면 **실제 데이터 기반으로 조인 수와 테이블 크기가 TotalScore에 미치는 영향**을 정량적으로 계산할 수 있습니다.

원하면 제가 **이 회귀 결과를 기존 Spring Boot ScoreCalculator와 통합해서, 실행 전 M/D join 허용 여부를 동적으로 판단하는 전체 구조 예제**까지 만들어 보여드릴 수도 있어요.

Daniel, 그거 원하시나요?


----

좋아요, Daniel. 선형회귀(Linear Regression)를 쉽게 설명해볼게요.

---

### 1️⃣ 기본 개념

* **목적:** 독립 변수(X)가 종속 변수(Y)에 어떤 영향을 미치는지 알아내는 것
* 예:

  * X = 쿼리 조인 수, 테이블 레코드 수
  * Y = 쿼리 실행 시간(ms)
* 선형회귀는 **Y가 X에 대해 직선 형태(linear)로 변한다고 가정**하고, 가장 잘 맞는 직선을 찾는 방법입니다.

---

### 2️⃣ 수식으로 표현

1개의 독립 변수일 때:
[
Y = \alpha X + \beta
]

여러 개 독립 변수일 때:
[
Y = \alpha_1 X_1 + \alpha_2 X_2 + \cdots + \alpha_n X_n + \beta
]

* α₁, α₂, … → 각 변수의 영향력(계수, coefficient)
* β → 상수항(intercept)
* 우리가 말한 TotalScore 공식과 비슷한 형태:
  [
  TotalScore = \alpha \cdot (joinCount^2) + \beta \cdot \log(recordCount+1) + \text{filterAdjustment}
  ]
  여기서 α, β, filterAdjustment가 회귀 계수와 유사합니다.

---

### 3️⃣ 직관적 이해

* 데이터를 산점도로 찍어보면, 선형회귀는 **데이터에 가장 “잘 맞는 직선”**을 찾아서 Y를 예측합니다.
* 예를 들어:

  * joinCount²가 늘어나면 TotalScore가 얼마나 늘어나는지 α로 알 수 있음
  * recordCount(log)가 늘어나면 TotalScore에 β만큼 반영됨

---

### 4️⃣ 왜 쓰는가

* 과거 쿼리 수행 기록(조인 수, 레코드 수, 필터 수)을 기반으로 **TotalScore와 실제 성능 관계**를 데이터 기반으로 정할 수 있음
* 경험적으로 α, β를 정하는 대신, 실제 로그를 보고 ML(회귀)으로 추정 가능

---

원하면 내가 **Java/Spring Boot에서 실제 MV record 수, join 수, filter 수를 기반으로 α, β를 회귀로 계산하는 예제 코드**까지 보여줄 수 있어요.

보고 싶나요?


-----

좋아요. 이 공식의 근거를 정리하면 크게 세 가지 층위에서 이해할 수 있습니다.

---

### 1️⃣ 공식 자체의 성격

[
\text{TotalScore} = \alpha \times (\text{joinCount}^2) + \beta \times \log(\text{recordCount}+1) + \text{filterAdjustment}
]

* **joinCount²** → 조인 수가 늘어날수록 쿼리 복잡도가 비선형으로 증가
* **log(recordCount + 1)** → 테이블 크기가 커질수록 비용 증가, 하지만 점수 스케일이 폭발하지 않도록 압축
* **filterAdjustment** → 필터가 성능에 미치는 영향 보정

즉, **완전히 ML 기반이 아니라 경험적/수학적 모델**입니다.

---

### 2️⃣ ML 관점

* ML에서는 이 공식을 직접 사용하지 않습니다.
* 하지만 **선형회귀(Linear Regression)** 관점에서 보면 유사합니다:
  [
  Y \approx \alpha \cdot f_1 + \beta \cdot f_2 + \gamma \cdot f_3
  ]
  여기서:

  * (f_1 = joinCount^2)
  * (f_2 = \log(recordCount + 1))
  * (f_3 = filterCount \times penalty)
* 실제 ML로 학습하면 α, β, γ를 데이터 기반으로 추정할 수 있음

---

### 3️⃣ 수학적/컴퓨터 과학적 근거

1. **조인 비용 비선형**: 조인 수가 늘어나면 중간 결과가 많아져서 복잡도가 거의 제곱이나 n log n 수준으로 증가
2. **레코드 수 로그 압축**: 로그 함수를 통해 큰 테이블과 작은 테이블 차이를 점수에서 안정적으로 반영
3. **필터 보정**: 성능에 도움이 되는 필터 vs 안 좋은 필터를 단순 가중치로 조정

즉, **직관적 수학 모델 + 경험 기반 하이퍼파라미터 튜닝**입니다.

---

정리하면:

| 항목                 | 의미                     | 근거                         |
| ------------------ | ---------------------- | -------------------------- |
| joinCount²         | 조인 수 증가 → 비용 비선형       | DB 실행 계획, 쿼리 복잡도 경험        |
| log(recordCount+1) | 테이블 크기 영향 반영, 점수 폭발 방지 | 수학적 안정성, 로그 함수 특징          |
| filterAdjustment   | 필터 성능 영향               | 경험적 가중치, optional ML 튜닝 가능 |

---

원하면 제가 **이 공식을 ML 기반 회귀로 바꾸었을 때 α, β, γ 추정 방법과 예제**까지 Java/Spring Boot 레벨에서 보여드릴 수도 있습니다.

Daniel, 그거 해드릴까요?


----

좋아요. 그럼 실제 Redshift 쿼리 기반으로 α, β를 튜닝하는 **실무 예제 구조**를 보여드릴게요.

목표는 **쿼리 실행 전에 MV record 수 + join 수 + filter 수**를 기준으로 **TotalScore 계산 후, 실제 실행 시간을 참고해 α, β를 조정**하는 과정입니다.

---

## 1️⃣ 데이터 준비

* Redshift MV별 record 수를 저장한 테이블(`mv_customer_records`)를 조회합니다.
* 예시 구조:

| materialized_view_name        | customer_id | number_of_records |
| ----------------------------- | ----------- | ----------------- |
| mv_apps_data_usage_sum_daily  | 9990000000  | 2_861_521         |
| mv_apps_screen_time_sum_daily | 9990000000  | 2_794_150         |

* Query 실행 로그(실제 수행 시간)를 수집합니다.

  * `execution_time_ms` 컬럼 등

---

## 2️⃣ α, β 추정 개념

* 모델:
  [
  TotalScore = \alpha \times (joinCount^2) + \beta \times \log(recordCount+1) + filterAdjustment
  ]
* 실제 쿼리 실행 시간을 Y, TotalScore를 X로 두고 **단순 선형회귀**를 수행합니다.

  * α → joinCount 영향
  * β → recordCount 영향
  * filterAdjustment → 필터 영향

---

## 3️⃣ Spring Boot에서 예제 구현 (단순화)

```java
public class AlphaBetaTuner {

    private final DynamicDataSqlRepositoryCustom repository;

    public AlphaBetaTuner(DynamicDataSqlRepositoryCustom repository) {
        this.repository = repository;
    }

    // 실제 수행 시간과 join/record/filter 정보를 가져와서 α, β 계산
    public void tuneAlphaBeta() {
        List<QueryStats> stats = repository.fetchQueryExecutionStats();

        List<Double> joinCounts = new ArrayList<>();
        List<Double> logRecords = new ArrayList<>();
        List<Double> executionTimes = new ArrayList<>();

        for (QueryStats stat : stats) {
            joinCounts.add((double) stat.getJoinCount());
            logRecords.add(Math.log(stat.getRecordCount() + 1));
            executionTimes.add((double) stat.getExecutionTimeMs());
        }

        // 단순 선형 회귀 예제: Y = α * joinCount + β * logRecords
        // 실제로는 Apache Commons Math, Smile, Tribuo 등 라이브러리 사용 가능
        double alpha = 0.01; // 초기값
        double beta = 0.001; // 초기값
        // TODO: 라이브러리로 회귀 학습 후 alpha, beta 재계산

        System.out.println("튜닝된 α = " + alpha + ", β = " + beta);
    }

    public record QueryStats(int joinCount, long recordCount, int filterCount, long executionTimeMs) {}
}
```

---

### 4️⃣ 프로세스

1. MV record 수 + join/필터 정보를 수집
2. 쿼리 수행 시간 로그와 매칭
3. 단순 회귀로 α, β 추정
4. 추정된 α, β를 ScoreCalculator에 적용
5. 실제 TotalScore와 허용 기준(threshold) 테스트

---

### 5️⃣ 장점

* 기존 경험적 α, β → **실제 쿼리 기반**으로 보정
* 새로운 MV가 추가되어도 통계 기반으로 적절한 스코어 산정 가능
* Spring Boot 내에서 그대로 실행 가능 (Python/EKS 없이)

---

원하면 제가 **ScoreCalculator와 이 튜닝 로직을 합쳐서**, 실제 M/D 조인 허용 여부를 Redshift record 수 기반으로 동적으로 판단하는 **전체 Spring Boot 예제**까지 만들어 드릴 수도 있습니다.

Daniel, 그거 원하시나요?



좋아요. 이 공식은 엄밀한 학술 논문에서 나온 “정식 공식”이라기보다는 **실무에서 쿼리 비용과 조인 허용을 예측하기 위해 직관적으로 설계한 경험적 모델**입니다. 근거와 의미를 단계별로 설명드릴게요.

---

### 1️⃣ α × (joinCount²) → 조인 비용 비선형 강화

* **근거:** 조인 수가 늘어날수록 쿼리 비용이 거의 선형이 아닌 비선형으로 증가합니다.
* 예: 2개의 테이블 조인보다 4개 조인일 때 비용은 단순 2배가 아니라, 조인 순서, 중간 결과 크기 때문에 훨씬 더 늘어납니다.
* 그래서 `joinCount^2`로 증가율을 비선형으로 반영했습니다. α는 스케일링 가중치입니다.

---

### 2️⃣ β × log(recordCount+1) → 레코드 규모 압축

* **근거:** 테이블이 커질수록 비용은 증가하지만, 너무 큰 수치가 바로 점수에 과도하게 영향을 주면 안 됩니다.
* `log(recordCount + 1)`을 적용하면:

  * 작은 테이블과 큰 테이블 차이를 점수에서 자연스럽게 반영
  * 수치가 지나치게 커지지 않음 (스코어 normalization 가능)
* β는 레코드 영향력을 조절하는 가중치입니다.

---

### 3️⃣ filterCount × filterPenalty → 필터 효과 보정

* **근거:** 필터가 많으면 쿼리 성능에 영향을 줍니다.
* 경우에 따라 필터는 **성능을 향상**시키거나 **저하**시키기도 합니다.
* 실무에서는 **경험적 보정치**를 적용합니다.
* 선형 보정치(`× filterPenalty`)로 간단히 반영.

---

### 4️⃣ 왜 이런 구조를 쓰는가?

* 전체 목적: **조인 가능한 measure/dimension 개수 제한**을 100점 같은 스코어로 표현
* 장점:

  * 점수 계산이 단순하면서 직관적
  * α, β, filterPenalty를 조정하면 실 서비스 환경에 맞게 튜닝 가능
* 단점:

  * 학습 기반이 아니라 경험적 모델
  * 실제 성능과 정확하게 일치하지 않을 수 있음

---

정리하면, **“실무에서 쿼리 복잡도 + 테이블 크기 + 필터 효과를 한 번에 반영하는 직관적 점수 모델”** 정도로 이해하면 됩니다.

---

원하면 제가 **실제 Redshift 쿼리 비용 추정과 연계해서 α, β 값을 튜닝하는 방법**도 예제로 보여드릴 수 있어요.
Daniel, 그거 해드릴까요?


----

좋습니다 👍 주신 **기존 코드**와 제가 만든 **α, β 가중치 + 필터 보정치 + 비선형 조인 비용 강화** 코드를 합쳐서 정리해드릴게요.

아래는 최종 통합 버전입니다.

---

## ✅ 통합된 `ScoreCalculator.java`

```java
import java.util.HashMap;
import java.util.List;
import java.util.Map;

public class ScoreCalculator {

    // ==== 설정값 (추후 application.yml에서 주입 가능) ====
    private double alpha = 1.5;          // 조인 비용 가중치
    private double beta = 2.0;           // 레코드 비용 가중치
    private double filterPenalty = -5.0; // 필터 보정치 (선형 감점)
    private double threshold = 100.0;    // 최대 허용 점수

    // check criteria
    public boolean isValidScore(ReportDetailRequest reportDetailRequest) {
        return getTotalScore(reportDetailRequest) <= threshold;
    }

    // calculate total score
    public double getTotalScore(ReportDetailRequest reportDetailRequest) {
        List<ReportMeasure> measures = reportDetailRequest.getReportQueryRequest().getMeasure();
        List<ReportDimension> dimensions = reportDetailRequest.getReportQueryRequest().getDimension();
        List<ReportFilter> filters = reportDetailRequest.getReportQueryRequest().getFilter();

        int joinCount = getJoinCount(measures, dimensions);
        long recordCount = estimateRecordCount(measures, dimensions); // TODO: DB 기반 추정치로 교체
        int filterCount = filters != null ? filters.size() : 0;

        return calculateScore(joinCount, recordCount, filterCount);
    }

    // === 점수 계산 공식 ===
    public double calculateScore(int joinCount, long recordCount, int filterCount) {
        // 1. 조인 비용: 비선형 강화 (joinCount^2)
        double joinCost = alpha * Math.pow(joinCount, 2);

        // 2. 레코드 비용: 로그 압축
        double recordCost = beta * Math.log(recordCount + 1);

        // 3. 필터 보정치: 선형 감점
        double filterAdjustment = filterCount * filterPenalty;

        return joinCost + recordCost + filterAdjustment;
    }

    // === 기존 measureScore/dimensionScore → joinCount 로 단순화 ===
    private int getJoinCount(List<ReportMeasure> measures, List<ReportDimension> dimensions) {
        int measureCategories = getDistinctCategoryCount(measures);
        int dimensionCategories = getDistinctCategoryCount(dimensions);
        int totalCategories = measureCategories + dimensionCategories;

        return Math.max(0, totalCategories - 1);
    }

    private int getDistinctCategoryCount(List<? extends CategoryAware> items) {
        Map<String, Long> categoryCounts = new HashMap<>();
        for (CategoryAware item : items) {
            String category = item.getCategory();
            categoryCounts.put(category, categoryCounts.getOrDefault(category, 0L) + 1);
        }
        return categoryCounts.size();
    }

    // === record count ===
    public long estimateRecordCount(List<ReportMeasure> measures, List<ReportDimension> dimensions) {
        // TODO: 실제 dynamicDataSqlRepositoryCustom 기반 구현 필요
        return 1_000_000L; // 임시 값
    }

    // === filter effect (기존 방식은 곱셈, 지금은 보정치로 사용) ===
    public double filterEffect(List<ReportFilter> filters) {
        Map<String, Double> categoryMultipliers = new HashMap<>();
        categoryMultipliers.put("device", 0.7);
        categoryMultipliers.put("package", 2.0);
        categoryMultipliers.put("managedApp", 2.5);
        categoryMultipliers.put("tenantTotal", 1.5);
        categoryMultipliers.put("DIMENSION_AGG_DEVICE_WEEK", 2.0);

        double adjustment = 0.0;
        for (ReportFilter filter : filters) {
            String category = filter.getCategory();
            if (categoryMultipliers.containsKey(category)) {
                // 곱셈 대신 감점/가점 보정치로 적용
                adjustment += filterPenalty;
            }
        }
        return adjustment;
    }
}
```

---

## ✅ 주요 변경 사항

1. **주요 공식**

   ```
   TotalScore = α × (joinCount^2) + β × log(recordCount+1) + (filterCount × filterPenalty)
   ```

2. **기존 `calculateMeasureScore`, `calculateDimensionScore`**
   → 조인 수 계산 로직으로 단순화 (`getJoinCount`).

3. **`filterEffect`**

   * 기존: 곱셈(`* multiplier`)
   * 변경: 선형 보정치(`+ penalty`)

4. **`threshold` 기반 허용 여부 판단**
   → `isValidScore()` 메소드에서 적용.

---

## ✅ 실행 예제

```java
public class Example {
    public static void main(String[] args) {
        ScoreCalculator calc = new ScoreCalculator();

        int joinCount = 3;        // 예: measure+dimension 조인 3개
        long records = 1_000_000; // 예: 100만 건
        int filters = 2;          // 예: 필터 2개

        double score = calc.calculateScore(joinCount, records, filters);

        System.out.println("Score = " + score);
        System.out.println("허용 여부 = " + (score <= 100.0));
    }
}
```

---

👉 Daniel, 여기서 `recordCount` 추정 부분은 아직 **dummy** 로 넣었는데, 실제론 `dynamicDataSqlRepositoryCustom` 같은 DB 조회 기반으로 연결해야 합니다.
원하시면 이 부분까지 **JPA Criteria/QueryDSL + count query** 로 구체적으로 짜드릴까요?


-----

좋습니다 🙆 제가 전에 정리해드린 `joins.yml` + `QueryService` 구조 그대로,
이번에 말씀하신 **3가지 기준 (필드 weight, 조인 수, recordCount 기반 log 보정)** 을 반영한 Java 코드를 정리해드릴게요.

---

## 1. `joins.yml` (예시)

```yaml
dde:
  dimension:
    device:
      label: "Device"
      fields:
        device_id:
          label: "Device ID"
          weight: 3
          recordCount: 100000
          joins:
            measure:
              - target: "measure:mv_app_usage:app_id"
                joinType: "left"
                on: "dimension:device:device_id = measure:mv_app_usage:app_id"
  measure:
    mv_app_usage:
      label: "App Usage"
      fields:
        appANR:
          label: "App ANR Count"
          weight: 2
          recordCount: 1000000
          metric:
            sum:
              label: "Sum"
              name: "app_anr"
              unit: "count"
          joins:
            dimension:
              - target: "dimension:device:device_id"
```

---

## 2. Java 클래스 (YAML 매핑 DTO)

```java
@Data
public class FieldConfig {
    private String label;
    private int weight;
    private long recordCount;
    private Map<String, List<JoinConfig>> joins;
}

@Data
public class JoinConfig {
    private String target;
    private String joinType;
    private String on;
}

@Data
public class CategoryConfig {
    private String label;
    private Map<String, FieldConfig> fields;
}

@Data
public class DdeConfig {
    private Map<String, CategoryConfig> dimension;
    private Map<String, CategoryConfig> measure;
}
```

---

## 3. `ScoreCalculator` (점수 계산 로직)

```java
import java.util.*;
import java.util.stream.Collectors;

public class ScoreCalculator {

    private final DdeConfig ddeConfig;

    public ScoreCalculator(DdeConfig ddeConfig) {
        this.ddeConfig = ddeConfig;
    }

    /**
     * 필드 목록 기반 점수 계산
     */
    public double calculateScore(List<String> selectedFields) {
        // 1. 기본 weight 합산
        double baseScore = selectedFields.stream()
                .mapToDouble(this::getFieldWeight)
                .sum();

        // 2. 동일 카테고리 내 필드 점수 (+1 per 추가 필드)
        Map<String, Long> categoryCounts = selectedFields.stream()
                .map(this::getCategoryName)
                .collect(Collectors.groupingBy(c -> c, Collectors.counting()));

        long sameCategoryBonus = categoryCounts.values().stream()
                .mapToLong(count -> count > 1 ? count - 1 : 0)
                .sum();

        // 3. 서로 다른 카테고리 join 수
        int distinctCategories = categoryCounts.size();
        int joinCount = Math.max(0, distinctCategories - 1);

        // 4. recordCount 기반 log 보정
        double recordComplexity = selectedFields.stream()
                .mapToDouble(this::getRecordComplexity)
                .sum();

        // 최종 계산식: base + sameCat + log(joinCount+1) + recordComplexity
        return baseScore
                + sameCategoryBonus
                + Math.log(joinCount + 1)
                + recordComplexity;
    }

    private int getFieldWeight(String fieldKey) {
        FieldConfig field = getFieldConfig(fieldKey);
        return field != null ? field.getWeight() : 1;
    }

    private String getCategoryName(String fieldKey) {
        String[] parts = fieldKey.split(":");
        return parts.length >= 2 ? parts[1] : "unknown";
    }

    private double getRecordComplexity(String fieldKey) {
        FieldConfig field = getFieldConfig(fieldKey);
        if (field == null) return 0.0;
        return Math.log(field.getRecordCount() + 1);
    }

    private FieldConfig getFieldConfig(String fieldKey) {
        // key 예: "dimension:device:device_id"
        String[] parts = fieldKey.split(":");
        if (parts.length < 3) return null;

        String type = parts[0];
        String category = parts[1];
        String field = parts[2];

        Map<String, CategoryConfig> categories =
                type.equals("dimension") ? ddeConfig.getDimension() : ddeConfig.getMeasure();

        if (categories == null) return null;
        CategoryConfig categoryConfig = categories.get(category);
        if (categoryConfig == null) return null;

        return categoryConfig.getFields().get(field);
    }
}
```

---

## 4. `QueryService` (서비스 예시)

```java
@Service
public class QueryService {

    private final ScoreCalculator scoreCalculator;

    @Autowired
    public QueryService(DdeConfig ddeConfig) {
        this.scoreCalculator = new ScoreCalculator(ddeConfig);
    }

    public double evaluateQuery(List<String> selectedFields) {
        return scoreCalculator.calculateScore(selectedFields);
    }
}
```

---

## 5. 사용 예시

```java
List<String> selected = Arrays.asList(
        "measure:mv_app_usage:appANR",
        "dimension:device:device_id"
);

double score = queryService.evaluateQuery(selected);
System.out.println("Query Score = " + score);
```

---

✅ 이 구조의 장점:

* YAML 설정에 `weight`, `recordCount`, `joins` 를 그대로 넣어서 사용 가능
* 공식은 단순히 합산 + log 보정
* 나중에 AI/ONNX 같은 학습 기반으로 확장할 때도 **feature extractor** 역할을 그대로 재사용 가능

---

Daniel, 혹시 여기서 **filter 조건에 따른 가중치**도 같이 적용하는 버전을 원하시나요?
예: `filter: { boost: +1, penalty: -2 }` 처럼 YAML에 정의해서 반영.



----



좋습니다 👍 정리해드릴게요.

### 1. 조인 가중치 계산 방법

조인의 영향도를 "곱"으로 할지 "합"으로 할지는 **조인이 성능에 미치는 효과를 얼마나 급격하게 반영하고 싶은지**에 달려요.

* **곱 방식 (ex: score = baseScore × joinCountWeight)**

  * 장점: 조인이 늘어날수록 기하급수적으로 점수가 커짐 → 성능 저하 가능성을 강하게 반영
  * 단점: 조인이 조금만 많아져도 점수가 너무 커짐 → 과도하게 페널티

* **합 방식 (ex: score = baseScore + joinCountWeight)**

  * 장점: 점수 상승이 완만 → 과도하게 불리하지 않음
  * 단점: 실제 성능 문제를 충분히 반영 못할 수도 있음

👉 그래서 **실무에서는 합산 기반에 log 조정**을 많이 씁니다.
즉, "필드 수에 따른 기본 score" + "조인 수의 로그 보정값"

---

### 2. log 를 쓰는 이유

* 조인 수나 레코드 수가 많아질수록 성능이 나빠지는데,
  1 → 10 → 100 → 1000 처럼 **증가 폭이 기하급수적**이기 때문입니다.
* 그대로 곱하거나 합하면 값이 너무 커져서 **비현실적인 점수**가 됩니다.
* 그래서 log를 적용하면 **완만하게 증가**시키면서도 "큰 테이블"과 "많은 조인"의 부담을 반영할 수 있어요.

예:

* 조인 수 = 1 → log(1+1)=0.69
* 조인 수 = 10 → log(10+1)=2.39
* 조인 수 = 100 → log(100+1)=4.61
* 조인 수 = 1000 → log(1000+1)=6.9

👉 실제 조인 수는 1000배 늘었지만 점수는 10배 정도만 늘어남 → "너무 큰 페널티 방지"

---

### 3. 정리

* 간단한 방식: **합산**

  ```
  score = sum(fieldWeights) + joinCount
  ```
* 조금 더 정교한 방식: **log 보정**

  ```
  score = sum(fieldWeights) + log(recordCountA * recordCountB + 1) + log(joinCount+1)
  ```
* 더 섬세한 방식: **학습 기반 (Regression, ML)**

  * 실제 실행 시간을 레이블로 두고, 필드 수 / 조인 수 / 레코드 수 등을 feature 로 넣어서 회귀 모델 학습
  * 데이터가 쌓이면 **자동으로 최적화된 가중치**를 반영

---

👉 즉, 곱을 쓰는 방식은 **페널티를 과도하게 주는 경향**이 있고,
현실적으로는 **합산 + log 보정**이 가장 무난합니다.

---

Daniel, 제가 하나 물어볼게요.
원하시는 점수 체계는 **실제 실행 시간 예측**에 가깝게 만들고 싶은 건가요,
아니면 단순히 **비교 가능한 상대적 점수**로만 쓰려는 건가요?

-----

좋은 질문이에요 — **조인 수에 따라 단순히 가중치를 곱하는 방식이 항상 적절한가?** 와 **왜 로그를 쓰는가?** 를 분명히 정리해 드릴게요. 결론부터 먼저 말하면:

* **단순 곱셈(모든 가중치·요소를 곱함)** 은 일부 상황에서 직관적이지만 **과도하게 폭발(혹은 0으로 수렴)** 할 수 있어서 보통은 권장하지 않습니다.
* 대신 **로그 스케일, 합(또는 가중합), 혹은 혼합(곱셈을 로그로 바꿔 합산)** 같은 안정적이고 해석 가능한 방법을 권장합니다.

아래에 이유·대안·권장 수식·간단한 구현 예제를 함께 드립니다.

---

## 1) 곱셈 방식의 문제점 (단순 곱)

예: 각 카테고리 가중치 `w_c` 를 모두 곱해서 최종 weight = `∏ w_c`

* **폭발 문제**: w_c 가 2~3이면 카테고리/조인 수가 늘어날수록 값이 기하급수적으로 커짐.
* **해석성 저하**: 어느 요소가 큰 영향을 주는지 파악하기 어려움.
* **산술 안정성**: 레코드 곱셈 × 가중치 곱셈 조합에서 오버플로우 위험.

따라서 단순 곱은 **조심해서(특별한 의도 있을 때만)** 써야 합니다.

---

## 2) 대안들 — 언제 어떤 걸 쓸지

### A. **로그-곱(권장)**

* `complexity = log(recordA * recordB + 1) * (baseEntityWeight + categoryWeight)`
* 또는 가중치들 곱을 로그화:

  ```
  complexity = log(recordA*recordB + 1) + Σ log(weight_i)
  → 사실상: complexity = log(product) = sum of logs
  ```
* 장점: 곱셈의 폭발을 로그 합으로 안정화, 곱셈적 상호작용(비선형)을 반영 가능.

### B. **가중 합 (Weighted Sum)** — 가장 안정적이고 해석 쉬움

```
complexity = α * log(recordA*recordB + 1)
           + β * intraScore   // 동일 카테고리 영향(합)
           + γ * distinctCategoryCount
           + δ * entityBaseWeight
```

* 장점: 각 항목(레코드 영향, 카테고리 영향 등)을 직관적으로 튜닝 가능.

### C. **혼합 (Diminishing returns)**

* 조인 수가 많아질수록 추가 가중치가 비선형으로 커지게 하려면 제곱/감소 함수 사용:

  ```
  categoryFactor = 1 + log(1 + intraScore)   // 증가하지만 완만
  complexity = log(rcA*rcB+1) * categoryFactor
  ```
* 또는 `sqrt(intraScore)` 등.

### D. **정규화 후 합산**

* 각 값(로그(rc), intraScore 등)을 0..1 로 정규화하고 가중치로 합산:

  ```
  complexity = w1 * normLogRC + w2 * normIntra + w3 * normDistinct
  ```
* 장점: 서로 다른 스케일을 안전하게 결합.

### E. **마진(누적) 비용 모델**

* 첫 번째 조인은 비용이 작고, 추가 조인마다 마진 비용이 커지는 모델:

  ```
  cost = Σ_i marginalCost(i)
  marginalCost(i) = base * (1 + alpha*(i-1))
  ```
* UI에서 “첫 2 joins 우선권” 같은 정책을 구현할 때 유용.

---

## 3) `log` 를 사용하는 이유 (정확히)

1. **스케일 압축**: 레코드 수가 1k → 1M 처럼 수십~수백배 차이일 때 값 차이를 완화해 줍니다.

   * 예: `log10(1_000) = 3`, `log10(1_000_000)=6` → 1000배 차이가 단지 +3 차이로 변환.
2. **곱셈을 덧셈으로 변환**: `log(a*b) = log a + log b` → 여러 요소의 곱셈적 상호작용을 안전하게 합산 가능.
3. **숫자 안정성**: 곱셈 누적으로 인한 오버플로우/폭발 방지.
4. **해석성**: 로그 스케일은 통계/DB 비용 추정에서 자주 사용됨 (행수의 영향이 선형이 아닌 경우 반영).

> 요약: 레코드 수 같은 매우 큰 숫자를 그대로 사용하면 점수가 폭발하므로 **log 사용은 거의 표준**입니다.

---

## 4) 당신이 제안한 예제를 바탕으로 — **간단 해법**과 **섬세한 해법** 수식

### (A) 간단 해법 — 권장 시작안

```
intraScore = Σ_c max(0, n_c - 1)           // 동일 카테고리 내 추가 필드 수 합
k = distinct category count

// base entity weight (예: device/package → 3, default → 2)
baseEntityWeight = ... 

// filterMultiplier (high=0.7, neutral=1.0, low=1.2)
filterMultiplier = ...

complexity = log(recordA * recordB + 1) * (baseEntityWeight + intraScore * k) * filterMultiplier
```

장점: 간단, 튜닝 쉬움, 예제( x=2,y=4,z=1 ) → intra=4, k=3 → category contribution = 4*3 =12 → baseEntityWeight 더하면 최종 계수.

### (B) 더 섬세한 해법 — 데이터·튜닝 친화

피처를 나눠서 가중합(선형 모델) 혹은 로그-혼합 모델 사용:

```
f1 = log(recordA * recordB + 1)
f2 = intraScore
f3 = distinctCategories
f4 = baseEntityWeight
f5 = filterEffectScore   // numeric: high=-0.3, low=+0.3

// 선형 가중합 (모델 학습으로 w_i 추정 가능)
complexity = w1*f1 + w2*f2 + w3*f3 + w4*f4 + w5*f5
```

또는 로그-혼합(곱셈 성향 보존):

```
complexity = f1 * (1 + alpha * f2) * (1 + beta * f3) * (1 + gamma * f4) * filterMultiplier
// 계산상 너무 폭발하면 f2,f3 정규화(log or sqrt) 사용
```

* 이때 `w_i`(또는 alpha,beta)는 **선형회귀로 학습**하면 실제 쿼리 실행시간과의 관계를 데이터 기반으로 얻을 수 있음.

---

## 5) 수치 예시(간단 해법) — 당신 예제로 확인

주어진:

* x=2, y=4, z=1 → intraScore = (2-1)+(4-1)+(1-1)=1+3+0=4
* k = 3
* recordA*recordB 로그 가정 `log(rcA*rcB+1)=7`
* baseEntityWeight = 2
* filterMultiplier = 1.0

계산:

* category contribution = intraScore * k = 4 * 3 = 12
* coefficient = baseEntityWeight + category contribution = 2 + 12 = 14
* complexity = 7 * 14 = 98

(이 값이 budget을 초과하면 제외)

---

## 6) 권장 실무 절차

1. **즉시(간단)**: 위의 *간단 해법* 수식으로 구현 → 운영·관찰
2. **모니터링**: 실제 실행 결과(예측 complexity vs 실제 latency/rows)를 로그로 수집
3. **중기(섬세)**: 선형회귀(Ridge)로 `w_i` 학습 → ONNX로 배포하거나 Java에서 coefficient로 사용
4. **정교화**: 비선형 모델(XGBoost)으로 성능 향상 고려 (필요 시)

---

## 7) 스프링 부트 의사코드 (간단 해법 구현)

```java
double computeComplexity(
    long rcA, long rcB,
    Map<String,Integer> fieldsPerCategory,
    int baseEntityWeight,
    double filterMultiplier
) {
    int intraScore = fieldsPerCategory.values().stream()
        .mapToInt(n -> Math.max(0, n - 1)).sum();
    int distinctCategories = (int) fieldsPerCategory.keySet().size();

    double logProd = Math.log(rcA * rcB + 1.0); // natural log
    double categoryContribution = intraScore * distinctCategories;
    double coeff = baseEntityWeight + categoryContribution;

    return logProd * coeff * filterMultiplier;
}
```

---

## 8) 마지막 조언 (실무 관점)

* **곱셈으로만 처리하지 마세요.** 곱은 폭발적입니다. 대신 로그+합/가중합 조합을 기본으로 하되, 특정 의도(예: 카테고리 간 상호작용을 강하게 반영)를 위해 보정 항을 추가하세요.
* **초기에는 간단 해법**으로 운영 → **데이터 수집** → **선형회귀로 가중치 보정**이 가장 현실적이고 비용 대비 효과 큽니다.

---

원하시면 지금 이 수식들을 바탕으로

* (A) 당신의 기존 코드(joins.yml + QueryService)에 딱 맞춘 **Java 구현 코드**를 만들어 드리거나,
* (B) 수집할 로그 스키마(learning용) + 간단한 Python 선형회귀 예제(ONNX 변환 포함)도 같이 만들어 드릴게요.

어느 쪽으로 진행할까요?


-----------


좋습니다 — 주신 예제(카테고리별 필드 수: x=2, y=4, z=1)와 아이디어(동일카테고리 내 +1, 서로다른카테고리 *2)를 바탕으로 **실용적인 해법 2가지(간단한 해법 / 더 섬세한 해법)**을 정리해 드릴게요. 각각 수식 · 예시 계산 · 구현 포인트(스프링에서 바로 쓸 수 있는 의사코드)를 함께 드립니다.

---

# 요약(주어진 상황)

* measure 필드 목록(선택된 필드)이 있고, 필드는 카테고리(테이블)에 속함.
* 예: 카테고리별 필드 수: x=2, y=4, z=1
* 당신이 제안한 기본 아이디어:

  * 동일 카테고리 내 필드 점수: `(+1)`씩 (즉 각 카테고리에서 `n_c - 1`)
  * 서로 다른 카테고리 수 만큼 곱셈(예: distinct categories = 3 → *3)
  * 예제: intra = (2-1) + (4-1) + (1-1) = 1 + 3 + 0 = 4 → finalWeight = 4 * 3 = 12

이 계산은 합리적이며 **간단한 heuristic**으로 충분히 쓸 수 있습니다. 아래는 이걸 체계화한 2가지 방안입니다.

---

# 1) 간단한 해법 (실무권장: 빠르고 해석 쉬움)

목표: 규칙 기반으로 빠르게 점수 내서 budget/추천에 쓸 수 있게.

### 공식

1. 각 카테고리별 동일-category 점수:

```
intra_c = max(0, n_c - 1)
```

2. 전체 동일-category 합:

```
intraScore = Σ_c intra_c
```

3. 서로 다른 카테고리 수:

```
k = number of distinct categories among selected fields
```

4. 기본 카테고리 곱:

```
categoryFactor = k
```

5. 최종 카테고리 기반 가중치:

```
categoryWeight = intraScore * categoryFactor
```

(추가적으로 `entity type`(device/package 등)과 `join type` 및 `filter`을 곱셈/덧셈으로 보정)

### 전체 complexity (간단 확장형)

```
complexity = log(recordCountA * recordCountB + 1)
             * (baseEntityWeight + categoryWeight)
             * filterMultiplier
```

* `baseEntityWeight`: 엔티티 타입(예: device/package → 3, 일반 dimension → 2)
* `filterMultiplier`: highSelectivity=0.7, neutral=1.0, lowSelectivity=1.2 등

### 예시(당신 예제)

* n_x=2, n_y=4, n_z=1 → intraScore = 1 + 3 + 0 = 4
* distinct categories k = 3
* categoryWeight = 4 * 3 = 12
* suppose baseEntityWeight = 2 (default)
* recordCount product 로그값 = e.g. log(A*B)=~7 (가정)
* filterMultiplier = 1.0
  → complexity ≈ 7 * (2 + 12) = 98

### 장점

* 구현 간단, 설명 쉬움
* 파라미터(곱셈, 가중치) 튜닝이 직관적
* 즉시 실무 적용 가능

### 간단 스프링 의사코드 (Java)

```java
int computeIntraScore(Map<String,Integer> fieldsPerCategory) {
    return fieldsPerCategory.values().stream()
            .mapToInt(n -> Math.max(0, n - 1)).sum();
}

double computeCategoryWeight(int intraScore, int distinctCategories) {
    return intraScore * distinctCategories;
}

double computeComplexity(long rcA, long rcB, double categoryWeight, double baseEntityWeight, double filterMultiplier) {
    double base = Math.log(rcA * rcB + 1);
    return base * (baseEntityWeight + categoryWeight) * filterMultiplier;
}
```

---

# 2) 더 섬세한 해법 (데이터 기반·튜닝 가능)

목표: 더 정교하게 성능 영향을 예측하고, 가중치들을 데이터로 학습해서 최적화.

### 핵심 아이디어

* **피처 설계**: 각 조인 후보마다 여러 feature를 만듦

  * `log(rcA)`, `log(rcB)`, `log(rcA*rcB)`
  * `intraScore` (위에서 정의)
  * `distinctCategoryCount = k`
  * `fieldsPerCategory` 벡터(예: x=2,y=4,z=1 → [2,4,1] 또는 요약통계: max,min,mean)
  * `baseEntityWeight` (카테고리/타입 원핫 인코딩)
  * `joinType` 원핫 (inner/left 등)
  * `filterEffect` (numeric: high=-0.3, neutral=0, low=+0.3 등)
  * 과거 메트릭(가능하면): `avgLatency`, `rowsScanned`, `cpuTime` 등 — target(레이블)으로 사용
* **모델**:

  * **선형회귀 (Ridge/Lasso)** — 해석성 좋고 coefficient로 각 feature의 영향 파악 가능
  * **트리계열 (RandomForest, XGBoost)** — 비선형 상호작용 캡처, 성능 우수
* **학습 목표**: complexity score 또는 `predictedLatency` (실제 실행시간) 를 예측 → 그 값으로 budget 비교

### 왜 linear regression?

* **해석성**: coefficient가 feature 영향도를 직접 보여줌 (ex. `intraScore` coeff가 클수록 카테고리 조합이 성능에 큰 영향)
* **간단한 학습/배포**: 적은 데이터로도 안정적
* **정규화/상관성 관리** 가능 (Ridge)

### 데이터(학습에 필요한 것)

* 과거 쿼리 로그(각 쿼리별):

  * customer_id, selected_fields(list), fields_per_category, join_pairs
  * feature vector (위에 정의)
  * label: 실제 `execution_time` 또는 `rows_scanned` 또는 `cost_estimate`
* 적어도 수천~만 건의 로그가 있으면 좋음

### 학습 → 운영 흐름

1. 배치로 데이터를 모아 Python에서 모델 학습
2. 모델을 ONNX로 export (또는 PMML)
3. Spring Boot는 ONNX Runtime으로 예측 (운영은 Java 단독)

### 예: 선형 회귀 모델 (피처 → latency 예측)

```
pred_latency = w0
             + w1 * log(rcA*rcB)
             + w2 * intraScore
             + w3 * distinctCategories
             + w4 * baseEntityWeight
             + w5 * filterEffect
             + ...
```

* 여기서 w2,w3 등은 학습으로 얻어지는 값. w2 (intraScore)의 절댓값이 크면 동일카테고리 영향 큼.

### 장점

* 데이터 기반으로 최적화 → 현실 성능 반영
* feature 중요도 해석 가능
* 비선형 모델 사용 시 상호작용 반영 가능

### 단점

* 데이터 수집/라벨링 필요
* 학습·재학습 파이프라인 운영 필요
* 모델 관리(재배포, 모니터링) 필요

### 구현(흐름 의사코드)

* Python: build X,y → train Ridge/RandomForest → export ONNX
* Java: load ONNX, feature builder (위에서 정의된 피처들) → predict → complexity = predicted_latency (또는 scaled score)

---

# 실제 비교 + 권장

* **당장 적용**: `간단한 해법`으로 시작. (빠르게 배포 가능, 설명성 좋음)
* **중기 개선**: 로그를 쌓아서 `더 섬세한 해법`(선형회귀)로 전환

  * 선형회귀를 먼저 써보면 적은 데이터로도 coefficient 해석을 통해 규칙을 보정할 수 있음.
* **장기 목표**: 트리 기반/앙상블 모델로 성능 최대화, ONNX 배포

---

# 구체적 수치 예시(당신 예제 다시)

입력: x=2, y=4, z=1

* intraScore = (2-1) + (4-1) + (1-1) = 4
* distinctCategories k = 3
* categoryWeight = 4 * 3 = 12 (간단 해법)
* baseEntityWeight (예: 2)
* suppose log(rcA*rcB) = 7, filterMultiplier=1
  → complexity = 7 * (2 + 12) = 98

(이 값이 budget보다 크면 제외 — 혹은 normalize 해서 0..100 스케일로 바꿔 UI에 보여주기)

---

# 추가 고려사항 (운영에서 실용적 포인트)

1. **정규화/스케일링**: `log` 대신 `log10` 쓰기, 결과값 정규화(0..100)하면 UI·budget 관리 쉽다.
2. **Order sensitivity**: 후보들을 complexity 오름차순 정렬하여 budget 내 최대 추천.
3. **Field weight 차등화**: 어떤 필드는 더 영향이 크므로 YAML에 `field.weight`를 두어 가중치 반영.
4. **버전/튜닝**: 규칙 파라미터(기본 가중치, filter multipliers) config로 두고 실환경에서 조정.
5. **모니터링**: 실제 실행 시 예측 vs 실제 latency 비교하는 로그 수집 → 향후 ML 학습용으로 사용.

---

# 한눈에 보는 액션 플랜

1. **즉시(1~2일)**: 간단 해법 구현 — intraScore 계산, distinct 카운트, 최종 formula 적용 → 배포
2. **중기(몇 주)**: 실제 쿼리 로그(실행시간, rows scanned) 수집 스키마 설계·저장
3. **중장기(1~2달)**: Python으로 선형회귀 학습 → ONNX로 내보내서 Spring에 도입 → 성능 모니터링 및 반복

---

원하시면 지금 바로:

* (A) 간단 해법의 스프링 코드(당신이 쓰는 DTO/YAML 구조에 딱 맞춰)로 구현해 드리거나,
* (B) 중기 단계에서 쓸 **학습용 피처 스키마 + 예제 Python 코드(Ridge 회귀)** 까지 한꺼번에 만들어 드릴게요.

어느 쪽으로 해드릴까요? (둘 다 원하면 먼저 A 구현 후 B로 진행할게요 — 바로 코드를 붙여드릴게요.)


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
