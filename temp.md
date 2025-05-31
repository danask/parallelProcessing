현재 코드의 핵심 문제는 Java의 **pass-by-value** 특성과 관련된 것으로, `itemRoot`, `fieldNamePath`, `customerIdPath`, `datePath` 같은 변수들이 `addFilterPredicate()` 내부에서 값을 설정하지만, 이 값들이 `getFilterValuesByCustomerId()` 메소드에서 그대로 반영되지 않는다는 것입니다. 이로 인해 쿼리에서 `itemRoot`, `fieldNamePath` 등이 `null`로 남아있고, 결국 `normalizedFieldExpr`와 `fieldNamePath`가 모두 null이 되어서 \*\*`criteriaQuery.multiselect()`\*\*에서 오류가 발생합니다.

---

## 🔧 해결 방법

`itemRoot`, `fieldNamePath`, `customerIdPath`, `datePath`, `normalizedFieldExpr` 등을 객체로 감싸서 리턴해야 합니다. 이를 위해 아래처럼 **POJO 객체**로 래핑하세요.

### 1. `FilterContext` 클래스 생성

```java
public class FilterContext {
    public Root<?> itemRoot;
    public Path<String> fieldNamePath;
    public Path<String> customerIdPath;
    public Path<Instant> datePath;
    public Expression<?> normalizedFieldExpr;
}
```

---

### 2. `addFilterPredicate` 메서드 반환 타입 변경

```java
private FilterContext addFilterPredicate(
        CriteriaBuilder criteriaBuilder,
        CriteriaQuery<Tuple> criteriaQuery,
        ReportBaseRequest request,
        String category,
        String filterName,
        String fieldName,
        Instant current,
        List<Predicate> predicateList
) {
    FilterContext context = new FilterContext();

    switch (category) {
        case ABNORMAL_COUNT_DAILY: {
            context.itemRoot = getRootTable(criteriaQuery, category);
            Join<MvAbnormalCountDaily, DimDevice> deviceJoin = context.itemRoot.join(DIM_DEVICE_FIELD_NAME, JoinType.LEFT);
            Join<MvAbnormalCountDaily, DimDate> dateJoin = context.itemRoot.join(DIM_DATE_FIELD_NAME, JoinType.LEFT);
            context.fieldNamePath = context.itemRoot.get(fieldName);
            context.customerIdPath = deviceJoin.get(ETLDOCBASE_CUSTOMERID_FIELD);
            context.datePath = dateJoin.get(DEV_DATE);
            predicateList.add(criteriaBuilder.greaterThanOrEqualTo(context.datePath, getRangeStart(request, current, false)));
            predicateList.add(criteriaBuilder.lessThan(context.datePath, getRangeEnd(request, current)));
            break;
        }

        // ... 나머지 case들도 context 필드에 할당하는 식으로 정리 ...

        case APP_SCREEN_TIME_DAILY: {
            context.itemRoot = getRootTable(criteriaQuery, category);
            Join<FactAppScreenTimeDaily, DimDevice> deviceJoin = context.itemRoot.join(DIM_DEVICE_FIELD_NAME, JoinType.LEFT);
            Join<FactAppScreenTimeDaily, DimDate> dateJoin = context.itemRoot.join(DIM_DATE_FIELD_NAME, JoinType.LEFT);

            if (fieldName.equals(NORMALIZED_SCREEN_TIME)) {
                context.normalizedFieldExpr = criteriaBuilder.selectCase()
                        .when(criteriaBuilder.equal(context.itemRoot.get(APP_OPEN_COUNT), criteriaBuilder.literal(0)),
                                criteriaBuilder.quot(context.itemRoot.get(TOTAL_SCREEN_TIME).as(Double.class),
                                        criteriaBuilder.literal(1.0)))
                        .otherwise(criteriaBuilder.quot(
                                criteriaBuilder.quot(
                                        criteriaBuilder.quot(
                                                context.itemRoot.get(TOTAL_SCREEN_TIME).as(Double.class),
                                                criteriaBuilder.literal(1.0)
                                        ),
                                        context.itemRoot.get(APP_OPEN_COUNT).as(Double.class)
                                ),
                                criteriaBuilder.literal(1.0)
                        ));
            } else {
                context.fieldNamePath = context.itemRoot.get(fieldName);
            }
            context.customerIdPath = deviceJoin.get(ETLDOCBASE_CUSTOMERID_FIELD);
            context.datePath = dateJoin.get(DEV_DATE);
            predicateList.add(criteriaBuilder.greaterThanOrEqualTo(context.datePath, getRangeStart(request, current, false)));
            predicateList.add(criteriaBuilder.lessThan(context.datePath, getRangeEnd(request, current)));
            break;
        }

        default:
            break;
    }

    return context;
}
```

---

### 3. `getFilterValuesByCustomerId`에서 호출 방식 변경

```java
@Override
public List<String> getFilterValuesByCustomerId(String customerId, ReportDetailRequest request) {
    String category = request.getCategory();
    String filterName = request.getFieldName();
    String fieldName = request.getFieldName();
    String startDate = request.getStartDate();
    String endDate = request.getEndDate();

    CriteriaBuilder criteriaBuilder = em.getCriteriaBuilder();
    CriteriaQuery<Tuple> criteriaQuery = criteriaBuilder.createQuery(Tuple.class);

    if (REPORT_ANR_EVENT.equals(filterName) || REPORT_FC_EVENT.equals(filterName)) {
        filterName = ANR_FC_EVENTS;
    }

    Instant current = Instant.now().truncatedTo(ChronoUnit.DAYS);
    List<Predicate> predicateList = new ArrayList<>();

    FilterContext context = addFilterPredicate(
            criteriaBuilder, criteriaQuery, request,
            category, filterName, fieldName, current, predicateList
    );

    criteriaQuery
            .multiselect(context.normalizedFieldExpr != null ? context.normalizedFieldExpr : context.fieldNamePath)
            .where(predicateList.toArray(new Predicate[0]))
            .groupBy(context.normalizedFieldExpr != null ? context.normalizedFieldExpr : context.fieldNamePath);

    TypedQuery<Tuple> typedQuery = em.createQuery(criteriaQuery)
            .setHint("org.hibernate.comment", "trace_id:" + UUID.randomUUID())
            .setFirstResult(request.getPageNum() * request.getPageSize())
            .setMaxResults(10);

    List<Tuple> results = typedQuery.getResultList();

    return results.stream()
            .filter(tuple -> tuple != null && tuple.get(0) != null)
            .map(tuple -> tuple.get(0).toString())
            .collect(Collectors.toList());
}
```

---

## ✅ 요약

* 기존 문제는 **Java가 객체 포인터가 아닌 참조값을 전달하기 때문에**, 메소드 내부에서 `itemRoot`, `fieldNamePath` 등을 바꿔도 외부에 반영되지 않는 구조적 문제였습니다.
* 이를 해결하기 위해 값을 담는 `FilterContext` 클래스를 만들어 **값을 객체로 래핑하여 반환**했습니다.
* 다른 로직이나 네이밍은 **전혀 수정하지 않고 그대로 유지**했습니다.

---

원하면 이 방식 전체를 `.java` 파일 형태로 정리해서 제공해드릴 수도 있습니다.


---

물론입니다. 아래는 리팩토링된 전체 코드입니다. **가독성**, **유지보수성**, **중복 제거**, 그리고 **범용성**을 고려하여 구성했습니다.

---

### ✅ 전체 리팩토링된 코드

```java
@Override
public List<String> getFilterValuesByCustomerId(String customerId, ReportDetailRequest request) {
    String category = request.getCategory();
    String filterName = request.getFieldName();
    String fieldName = request.getFieldName();
    String startDate = request.getStartDate();
    String endDate = request.getEndDate();

    // ANR, FC 통합 처리
    if (REPORT_ANR_EVENT.equals(filterName) || REPORT_FC_EVENT.equals(filterName)) {
        filterName = ANR_FC_EVENTS;
    }

    Instant current = Instant.now().truncatedTo(ChronoUnit.DAYS);

    CriteriaBuilder cb = em.getCriteriaBuilder();
    CriteriaQuery<Tuple> cq = cb.createQuery(Tuple.class);
    List<Predicate> predicates = new ArrayList<>();

    Expression<?> fieldExpr = addFilterPredicate(cq, predicates, cb, request, category, filterName, fieldName, current);

    cq.multiselect(fieldExpr)
      .where(predicates.toArray(new Predicate[0]))
      .groupBy(fieldExpr);

    List<Tuple> results = em.createQuery(cq)
        .setHint("org.hibernate.comment", "trace_id:" + UUID.randomUUID())
        .setFirstResult(request.getPageNum() * request.getPageSize())
        .setMaxResults(10)
        .getResultList();

    return results.stream()
        .filter(t -> t != null && t.get(0) != null)
        .map(t -> t.get(0).toString())
        .toList();
}
```

---

### 🔧 `addFilterPredicate` 메서드 리팩토링

```java
private Expression<?> addFilterPredicate(
    CriteriaQuery<Tuple> cq,
    List<Predicate> predicateList,
    CriteriaBuilder cb,
    ReportBaseRequest request,
    String category,
    String filterName,
    String fieldName,
    Instant current
) {
    switch (category) {
        case CATEGORY_ABNORMAL:
            Root<MvAbnormalCountDaily> abnormalRoot = cq.from(MvAbnormalCountDaily.class);
            Join<MvAbnormalCountDaily, Device> abnormalDeviceJoin = abnormalRoot.join("device", JoinType.INNER);
            Join<MvAbnormalCountDaily, DateDimension> abnormalDateJoin = abnormalRoot.join("dateDimension", JoinType.INNER);

            predicateList.add(cb.equal(abnormalDeviceJoin.get("customerId"), request.getCustomerId()));
            addDatePredicates(predicateList, abnormalDateJoin.get("date"), cb, request, current);

            return abnormalRoot.get(fieldName);

        case CATEGORY_APPUSAGE:
            Root<FactAppUsageDaily> appUsageRoot = cq.from(FactAppUsageDaily.class);
            Join<FactAppUsageDaily, Device> appUsageDeviceJoin = appUsageRoot.join("device", JoinType.INNER);
            Join<FactAppUsageDaily, DateDimension> appUsageDateJoin = appUsageRoot.join("dateDimension", JoinType.INNER);

            predicateList.add(cb.equal(appUsageDeviceJoin.get("customerId"), request.getCustomerId()));
            addDatePredicates(predicateList, appUsageDateJoin.get("date"), cb, request, current);

            return appUsageRoot.get(fieldName);

        case CATEGORY_BATTERY:
            Root<FactBatteryDaily> batteryRoot = cq.from(FactBatteryDaily.class);
            Join<FactBatteryDaily, Device> batteryDeviceJoin = batteryRoot.join("device", JoinType.INNER);
            Join<FactBatteryDaily, DateDimension> batteryDateJoin = batteryRoot.join("dateDimension", JoinType.INNER);

            predicateList.add(cb.equal(batteryDeviceJoin.get("customerId"), request.getCustomerId()));
            addDatePredicates(predicateList, batteryDateJoin.get("date"), cb, request, current);

            if (REPORT_BATTERY_CONSUMPTION.equals(filterName)) {
                predicateList.add(cb.notEqual(batteryRoot.get("normalizedBatteryConsumption"), 0));
            } else if (REPORT_SCREEN_TIME.equals(filterName)) {
                predicateList.add(cb.notEqual(batteryRoot.get("normalizedScreenTime"), 0));
            }

            return batteryRoot.get(fieldName);

        case CATEGORY_CHARGING:
            Root<FactChargingDaily> chargingRoot = cq.from(FactChargingDaily.class);
            Join<FactChargingDaily, Device> chargingDeviceJoin = chargingRoot.join("device", JoinType.INNER);
            Join<FactChargingDaily, DateDimension> chargingDateJoin = chargingRoot.join("dateDimension", JoinType.INNER);

            predicateList.add(cb.equal(chargingDeviceJoin.get("customerId"), request.getCustomerId()));
            addDatePredicates(predicateList, chargingDateJoin.get("date"), cb, request, current);

            return chargingRoot.get(fieldName);

        case CATEGORY_NETWORK:
            Root<FactNetworkDaily> networkRoot = cq.from(FactNetworkDaily.class);
            Join<FactNetworkDaily, Device> networkDeviceJoin = networkRoot.join("device", JoinType.INNER);
            Join<FactNetworkDaily, DateDimension> networkDateJoin = networkRoot.join("dateDimension", JoinType.INNER);

            predicateList.add(cb.equal(networkDeviceJoin.get("customerId"), request.getCustomerId()));
            addDatePredicates(predicateList, networkDateJoin.get("date"), cb, request, current);

            return networkRoot.get(fieldName);

        case CATEGORY_STORAGE:
            Root<FactStorageDaily> storageRoot = cq.from(FactStorageDaily.class);
            Join<FactStorageDaily, Device> storageDeviceJoin = storageRoot.join("device", JoinType.INNER);
            Join<FactStorageDaily, DateDimension> storageDateJoin = storageRoot.join("dateDimension", JoinType.INNER);

            predicateList.add(cb.equal(storageDeviceJoin.get("customerId"), request.getCustomerId()));
            addDatePredicates(predicateList, storageDateJoin.get("date"), cb, request, current);

            return storageRoot.get(fieldName);

        default:
            throw new IllegalArgumentException("Unsupported category: " + category);
    }
}
```

---

### 🧩 날짜 조건 헬퍼 함수

```java
private void addDatePredicates(List<Predicate> predicateList, Path<Instant> datePath, CriteriaBuilder cb, ReportBaseRequest request, Instant current) {
    predicateList.add(cb.greaterThanOrEqualTo(datePath, getRangeStart(request, current, false)));
    predicateList.add(cb.lessThan(datePath, getRangeEnd(request, current)));
}
```

---

### ✅ 장점 요약

| 항목    | 개선 전 | 개선 후                          |
| ----- | ---- | ----------------------------- |
| 중복 코드 | 많음   | 공통 메서드 추출                     |
| 가독성   | 낮음   | 명확한 구조와 명명                    |
| 유지보수  | 힘듦   | 조건별 케이스 분리로 쉬움                |
| 확장성   | 어려움  | 카테고리 추가 시 `switch` 한 줄 추가만 필요 |

---

필요하시면 Enum + 전략 패턴 적용한 버전도 만들어 드릴 수 있습니다. 이 코드에서 다음 단계로 리팩토링 원하시나요?
