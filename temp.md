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
