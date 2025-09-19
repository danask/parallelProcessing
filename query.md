좋아요 Daniel. 지금 말씀하신 “count의 avg 값”을 구하고 싶다면, 몇 가지 포인트를 짚어야 합니다.

---

### 1️⃣ 현재 COUNT 구문 문제

```sql
count(case when LOWER(mv_battery_soh_daily.battery_soh_status) = 'good'
           then mv_battery_soh_daily.battery_soh else 0 end)
```

* `COUNT(expr)`는 **NULL 아닌 값의 개수**를 셉니다.
* 그런데 `ELSE 0`를 넣으면 `"good"` 아닌 row도 0으로 들어가서, 사실상 모두 세어버리게 됩니다.
* **COUNT에서 `ELSE NULL`이 맞습니다.**

```sql
count(case when LOWER(mv_battery_soh_daily.battery_soh_status) = 'good'
           then 1 end)
```

* 이렇게 하면 `"good"`인 row만 카운트됨.

---

### 2️⃣ 날짜별 count의 avg

* 예를 들어, 7일간 `"good"` 상태 row 개수의 **평균**을 구하려면, **서브쿼리**로 날짜별 count를 먼저 만들고, 그 결과에서 `AVG`를 구해야 합니다.

```sql
with daily_counts as (
  select
    dim_date.dev_date as date_distinct,
    count(case when LOWER(mv_battery_soh_daily.battery_soh_status) = 'good' then 1 end) as cnt_battery_good
  from kai_dwh.mv_battery_soh_daily
  right join kai_dwh.dim_device
    on dim_device.dim_device_id = mv_battery_soh_daily.dim_device_id
  right join kai_dwh.dim_date
    on dim_date.dim_date_id = mv_battery_soh_daily.dim_date_id
  where dim_device.customer_id = '3508168036'
    and dim_date.dev_date between '2025-09-11' and '2025-09-17'
  group by dim_date.dev_date
)
select
  date_distinct,
  cnt_battery_good,
  avg(cnt_battery_good) over () as avg_count_per_day
from daily_counts
order by cnt_battery_good desc, date_distinct asc;
```

---

### 3️⃣ 포인트 정리

1. **COUNT에서 ELSE 0는 쓰지 마세요.** → NULL 처리
2. **날짜별 count를 만들고, 그 결과에서 AVG를 계산**
3. Redshift/Postgres에서는 **`AVG(...) OVER ()`** 로 전체 평균을 구할 수 있음

---

즉, Daniel이 원하는 게 \*\*“기간 내 날짜별 good 상태 배터리 개수와 그 평균”\*\*이라면 위 쿼리가 정확하게 맞습니다.

원하시면 제가 **HAVING 조건까지 포함한 최종 버전**으로 만들어서 보여드릴 수도 있어요. 보여드릴까요?
