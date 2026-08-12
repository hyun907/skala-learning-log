# 윈도우 함수: 상관 서브쿼리를 바꿨더니 버퍼가 140분의 1이 됐다

## 개념 정의

`GROUP BY`는 **집계하면서 행을 잃는다.** 부서별 평균 연봉을 구하면 부서 수만큼의 행이 남고, 개별 사원은 사라진다.

그런데 실무에서 필요한 건 대개 이런 질문이다.

> "각 사원의 연봉과, **그 사원이 속한 부서의 평균**을 나란히 보고 싶다"

`GROUP BY`로는 안 된다. 접는 순간 사원이 없어지기 때문이다.

윈도우 함수는 **집계하면서 원래 행을 유지한다.**

| | `GROUP BY` | 윈도우 함수 |
|---|---|---|
| 결과 행 수 | 그룹 수만큼 (줄어든다) | **원본 그대로** |
| 개별 행 정보 | 사라진다 | **남는다** |
| 쓰는 곳 | 요약 | 행별 값 + 집계값 **동시에** |

문법은 `OVER` 절 하나다.

```sql
함수() OVER (
    [PARTITION BY 컬럼]   -- ① 무엇을 기준으로 나눌 것인가
    [ORDER BY 컬럼]       -- ② 그 안에서 어떤 순서로 볼 것인가
    [ROWS/RANGE 프레임]   -- ③ 현재 행 기준 어디까지 포함할 것인가
)
```

셋 다 생략할 수 있다. 아무것도 안 쓰면 전체가 하나의 창(window)이 된다.

## 동작 원리

### 순위 — 동점을 어떻게 처리할 것인가

점수가 `90, 85, 85, 80`인 4명일 때 함수마다 결과가 다르다.

| 함수 | 결과 | 규칙 |
|---|---|---|
| `ROW_NUMBER()` | **1, 2, 3, 4** | 동점이어도 무조건 고유 번호 |
| `RANK()` | **1, 2, 2, 4** | 동점은 같은 순위, 다음을 **건너뛴다** |
| `DENSE_RANK()` | **1, 2, 2, 3** | 동점은 같은 순위, 건너뛰지 않는다 |

**"각 그룹 상위 3명"을 뽑을 때는 보통 `ROW_NUMBER()`다.** `RANK()`로 짜면 동점 때문에 4명이 나올 수 있다. 그게 맞을 때도 있으니 **의도를 정하고 골라야 한다.**

```sql
SELECT major, name, gpa,
       ROW_NUMBER() OVER (PARTITION BY major ORDER BY gpa DESC, student_id) AS rn
FROM students;
```

`ORDER BY`에 `student_id`를 2차 기준으로 넣은 이유는 **동점일 때 순서가 매번 달라지는 걸 막기 위해서**다. 이게 없으면 같은 쿼리를 두 번 돌렸을 때 다른 사람이 뽑힐 수 있다.

한 가지 함정이 있다. **`WHERE rn <= 3`을 같은 `SELECT`에서 쓸 수 없다.** 윈도우 함수는 `SELECT` 단계에서 계산되는데 `WHERE`는 그보다 먼저 실행되기 때문이다. CTE로 한 번 감싸야 한다.

```sql
WITH ranked AS (
  SELECT major, name, gpa,
         ROW_NUMBER() OVER (PARTITION BY major ORDER BY gpa DESC, student_id) AS rn
  FROM students
)
SELECT * FROM ranked WHERE rn <= 3;
```

### 앞뒤 행을 당겨오기 — `LAG` / `LEAD`

증감률과 추이를 구하는 표준 도구다.

```sql
SELECT region, month, revenue,
  LAG(revenue, 1) OVER (PARTITION BY region ORDER BY month) AS prev_month,
  ROUND(
    (revenue - LAG(revenue,1) OVER (PARTITION BY region ORDER BY month))
    / NULLIF(LAG(revenue,1) OVER (PARTITION BY region ORDER BY month), 0) * 100
  , 1) AS growth_pct
FROM sales;
```

`NULLIF(이전값, 0)`이 중요하다. 이전 달 매출이 0이면 0으로 나누게 되는데, `NULLIF`가 0을 `NULL`로 바꿔 주면 **결과가 `NULL`이 되고 쿼리는 살아 있는다.** 없으면 에러로 중단된다.

전년 대비는 오프셋만 바꾸면 된다 — `LAG(revenue, 12)`.

### 프레임 — 어디까지 포함할 것인가

```sql
-- 누적합: 처음부터 현재까지
SUM(revenue) OVER (ORDER BY month
                   ROWS BETWEEN UNBOUNDED PRECEDING AND CURRENT ROW)

-- 3개월 이동평균: 직전 2개 + 현재
AVG(revenue) OVER (ORDER BY month
                   ROWS BETWEEN 2 PRECEDING AND CURRENT ROW)
```

`ROWS`는 **물리적 행 개수**, `RANGE`는 **`ORDER BY` 값의 범위** 기준이다. 같은 값을 가진 행이 여럿이면 결과가 달라진다.

## 직접 확인한 예시

이커머스 스키마에서 **"첫 구매 후 30일 안에 재구매한 고객 비율"** 을 구하는 쿼리를 튜닝했다. 이 문항이 윈도우 함수의 가치를 가장 선명하게 보여 줬다.

### 튜닝 전 — 고객마다 서브쿼리가 반복됐다

원래 구조는 이랬다.

1. 고객별로 첫 주문 시각을 구한다
2. **각 고객마다** "첫 주문 후 30일 안에 다른 주문이 있는가"를 상관 서브쿼리(`EXISTS`)로 확인한다
3. 그 조건을 두 집계식에서 **각각** 평가한다

관찰 대상 고객이 2,084명이었으니, **서브쿼리가 그만큼 반복 실행됐다.**

### 튜닝 후 — 한 번의 정렬로 끝냈다

`ROW_NUMBER()`로 첫 주문을 식별하고, `LEAD()`로 **바로 다음 주문 시각을 같이 끌어왔다.**

```sql
WITH ordered AS (
  SELECT customer_id, order_ts,
         ROW_NUMBER() OVER (PARTITION BY customer_id ORDER BY order_ts) AS rn,
         LEAD(order_ts) OVER (PARTITION BY customer_id ORDER BY order_ts) AS next_ts
  FROM orders
  WHERE order_status IN ('paid', 'shipped', 'delivered')
)
SELECT COUNT(*) AS observed,
       COUNT(*) FILTER (WHERE next_ts <= order_ts + INTERVAL '30 days') AS repurchased
FROM ordered
WHERE rn = 1
  AND order_ts + INTERVAL '30 days' <= :기준시각;   -- 관찰기간이 끝난 고객만
```

첫 주문인지(`rn = 1`)와 다음 주문이 언제인지(`next_ts`)를 **한 번의 정렬로 동시에** 얻는다. 반복이 사라진다.

### 결과

| | 튜닝 전 | 튜닝 후 | 변화 |
|---|---|---|---|
| 실행시간 | 8.377 ms | 5.263 ms | −37.17% |
| **버퍼** | **12,594** | **90** | **약 140분의 1** |

실행계획도 **"고객별 반복 Index Scan" → "한 번의 정렬 + WindowAgg"** 로 바뀌었다.

**주목할 건 두 숫자의 격차다.** 실행시간은 37% 줄었을 뿐인데 읽은 페이지는 자릿수가 달라졌다. 지금은 데이터가 수천 건이라 시간 차이가 작지만, **반복 횟수는 고객 수에 비례하므로 사용자가 늘면 이 격차가 그대로 시간으로 나타난다.**

분석 결과는 관찰 대상 2,084명 중 963명 재구매, **재구매율 46.21%** 였다.

### 다른 문항에서도 썼다

- **제품별 누적매출 Top 20** — `RANK()`로 순위를 매겼다. 동점 처리를 확인하려고 `RANK()`와 `DENSE_RANK()`를 같이 계산해 비교했다
- **상위 1% 고객 추출** — 활성 고객 1,672명에 `CEIL(1672 × 0.01) = 17`을 적용했다. 매출 동점이 생길 수 있어서 `customer_id`를 보조 정렬 기준으로 넣어 **정확히 17명이 나오도록 순서를 고정**했다

## 자주 발생하는 오류

**`WHERE`에 윈도우 함수를 쓴다**

실행 순서 때문에 불가능하다. CTE나 서브쿼리로 감싸야 한다.

**`LAG` 결과가 `NULL`인데 나눗셈에 쓴다**

첫 행은 이전 값이 없어서 `NULL`이다. 여기에 0까지 섞이면 `division by zero`가 난다. `NULLIF`로 막는다.

**동점 처리를 정하지 않고 "상위 N개"를 뽑는다**

`RANK()`로 짜면 N개보다 많이 나올 수 있다. 정확히 N개가 필요하면 `ROW_NUMBER()`에 **2차 정렬 기준**까지 넣어야 순서가 고정된다.

**`LAST_VALUE`가 마지막 값을 안 준다**

기본 프레임이 "처음~현재 행"이라 현재 행 자기 자신이 나온다. `ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING`을 명시해야 한다.

**MySQL 8.0 미만에서는 아예 없다**

윈도우 함수는 MySQL 8.0부터 지원된다. 구버전이면 다른 방법을 찾아야 한다.

## 프로젝트 적용

**"상관 서브쿼리가 보이면 윈도우 함수를 의심한다."**

내가 겪은 패턴이 그대로 재사용된다. 바깥 행을 참조하는 서브쿼리는 **행마다 실행되고**, 그건 실행계획에서 `loops` 값이 큰 것으로 나타난다. 대부분 다음 셋 중 하나로 바꿀 수 있다.

| 원래 하려던 것 | 바꿀 것 |
|---|---|
| 각 행에서 "다음/이전 값"을 본다 | `LEAD` / `LAG` |
| 각 행에서 "그룹 내 순위"를 본다 | `ROW_NUMBER` / `RANK` |
| 각 행에서 "그룹 평균과 비교"한다 | `AVG() OVER (PARTITION BY ...)` |

**추이를 보는 지표에 특히 잘 맞는다.** 버전별 성능 변화, 주차별 잔존율, 응답시간 이동평균처럼 **"이전과 비교"가 들어가는 지표는 거의 전부** 윈도우 함수로 한 번에 계산된다. 애플리케이션에서 배열을 만들어 놓고 인덱스를 하나씩 밀어가며 비교하는 코드가 통째로 사라진다.

## 배운 점

**`GROUP BY`가 정보를 잃는다는 걸 알고 나면, 그동안 코드로 메우던 게 뭐였는지 보인다.**

"집계값을 구한 다음, 원본을 다시 순회하면서 각 행에 그 값을 붙인다" — 이런 코드를 짜 본 적이 있다면 그게 윈도우 함수가 하는 일이다. DB가 이미 정렬해 둔 결과를 한 번 훑으면서 처리하는 것과, 애플리케이션이 데이터를 전부 받아서 두 번 도는 것은 비용이 다르다.

**그리고 성능 개선을 실행시간으로만 재면 이 사례의 크기를 놓친다.** 37%와 140배는 다른 이야기이고, 앞으로 데이터가 늘었을 때 어떻게 될지를 말해 주는 쪽은 후자다.
