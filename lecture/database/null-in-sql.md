# SQL의 NULL: 에러가 안 나는 게 문제다

## 개념 정의

대부분의 프로그래밍 언어에서 `null == null`은 참이다. **SQL은 다르다.**

```sql
SELECT NULL = NULL;    -- TRUE가 아니라 UNKNOWN
SELECT NULL + 5;       -- NULL
SELECT 'abc' || NULL;  -- NULL
```

SQL의 `NULL`은 **"값이 없음"이 아니라 "모름(UNKNOWN)"** 이다. 모르는 것과 비교하면 결과도 모른다. 그래서 SQL은 참·거짓에 UNKNOWN을 더한 **3치 논리**로 동작한다.

여기서 하나가 따라 나온다. **`WHERE` 절은 TRUE인 행만 통과시킨다.** UNKNOWN은 통과하지 못한다.

그래서 `WHERE col = NULL`은 에러가 아니라 **항상 0행**을 돌려준다. `IS NULL`을 써야 한다.

이 규칙 하나에서 아래 함정들이 전부 파생된다. 그리고 **전부 에러가 안 난다.** 그게 이 글의 제목이다.

## 동작 원리

### ① `NOT IN` + NULL = 빈 집합

가장 위험한 형태다. 결과가 그냥 사라진다.

```sql
-- enroll.student_id에 NULL이 하나라도 있으면 결과가 무조건 0행
SELECT s.name FROM student s
WHERE s.id NOT IN (SELECT student_id FROM enroll);
```

전개하면 이렇게 된다.

```
s.id != 1  AND  s.id != 2  AND  s.id != NULL
                                 ↑ UNKNOWN
```

`TRUE AND UNKNOWN`은 `UNKNOWN`이다. 그래서 **전체 조건이 절대 TRUE가 되지 않는다.**

해결은 `NOT EXISTS`다.

```sql
SELECT s.name FROM student s
WHERE NOT EXISTS (SELECT 1 FROM enroll e WHERE e.student_id = s.id);
```

`NOT EXISTS`는 "행이 있느냐 없느냐"만 보므로 NULL에 영향받지 않고, 하나 찾으면 즉시 멈춰서 대개 더 빠르다.

### ② 평균의 분모가 조용히 달라진다

`SUM`과 `AVG`는 `NULL`을 **빼고** 계산한다.

```sql
SELECT COUNT(*),       -- 100  (NULL 포함)
       COUNT(score),   --  20  (NULL 제외)  ← 분모가 다르다
       AVG(score)      --  20명의 평균이지 100명의 평균이 아니다
FROM students;
```

**"평균 점수가 왜 이렇게 높지"의 가장 흔한 원인이다.** 점수가 없는 학생이 0점으로 세어지는 게 아니라 **모집단에서 빠진다.**

### ③ 외래키가 NULL이면 `INNER JOIN`에서 사라진다

`ON DELETE SET NULL`로 부모를 지우면 자식의 외래키가 `NULL`이 된다. 그 상태에서 `INNER JOIN`을 하면 **그 행들이 통째로 결과에서 빠진다.**

설계 시점의 삭제 정책 결정이 몇 달 뒤 리포트에서 "합계가 안 맞는다"로 나타난다. 대응은 `LEFT JOIN` + `COALESCE`.

### ④ 정렬 순서가 DBMS마다 다르다

| DBMS | `ORDER BY col ASC`에서 NULL 위치 |
|---|---|
| PostgreSQL / Oracle | 마지막 |
| MySQL | 먼저 |
| SQL Server | 가장 앞 |

같은 쿼리를 DB만 바꿔 옮기면 **1페이지에 뜨던 게 마지막 페이지로 간다.** 명시하려면 PostgreSQL·Oracle은 `NULLS LAST`를 쓴다.

### ⑤ 0으로 나누기를 NULL로 피한다

여기서는 `NULL`이 문제가 아니라 **해법**이 된다.

```sql
(revenue - prev) / NULLIF(prev, 0) * 100
```

`NULLIF(a, b)`는 두 값이 같으면 `NULL`, 다르면 `a`를 준다. 분모가 0이면 `NULL`이 되고, **`NULL`로 나눈 결과는 에러가 아니라 `NULL`이다.** 쿼리가 죽지 않는다.

## 직접 확인한 예시

이커머스 스키마 실습에서 마지막 문항이 **"0으로 나눠도 리포트가 중단되지 않게 만들라"** 였다.

### 먼저 실제로 터뜨려 봤다

```sql
SELECT 100 / 0;
-- ERROR: division by zero
```

에러가 나면서 **쿼리 전체가 중단된다.** 채널별 평균 주문금액을 구하는 리포트에서 주문이 0건인 채널이 하나라도 있으면, 그 채널 때문에 **리포트 전체가 안 나온다.**

### 두 가지 방법으로 막았고, 나누는 기준을 정했다

**방법 1 — 사용자 정의 함수**

```sql
CREATE OR REPLACE FUNCTION safe_divide(numerator NUMERIC, denominator NUMERIC)
RETURNS NUMERIC LANGUAGE sql IMMUTABLE AS $$
  SELECT COALESCE(numerator / NULLIF(denominator, 0), 0)
$$;
```

분모가 **0이거나 NULL이면** 기본값 0을 돌려준다. 검증값 세 가지를 확인했다.

| 입력 | 결과 |
|---|---|
| `safe_divide(100, 4)` | **25** |
| `safe_divide(100, 0)` | **0** |
| `safe_divide(100, NULL)` | **0** |

**방법 2 — 기본 함수를 직접**

```sql
ROUND(COALESCE(SUM(line_total), 0) / NULLIF(COUNT(DISTINCT order_id), 0), 2)
```

**두 방법을 나눠 쓴 기준이 있었다.**

| | 쓴 곳 | 이유 |
|---|---|---|
| 사용자 정의 함수 | 여러 쿼리에서 반복 | **재사용성** |
| `NULLIF()` 직접 | Materialized View 생성 스크립트 | **배포 독립성** — 객체 의존성을 만들지 않는다 |

함수를 만들면 재사용은 편하지만, 그 함수에 의존하는 객체가 생긴다. Materialized View는 독립적으로 배포·재생성돼야 해서 기본 함수만 썼다.

### 검증까지 했다

채널별 주문 수가 `1,288 + 2,523 + 1,296 = 5,107`건으로 **전체 유효 주문 수와 정확히 일치**하는 것을 확인했다. 합계가 맞는다는 건 `LEFT JOIN`으로 주문 없는 채널을 유지하면서도 **중복이나 누락이 없었다**는 뜻이다.

`INNER JOIN`으로 짰다면 주문이 0건인 채널이 결과에서 빠졌을 것이고, **그게 바로 가장 알고 싶은 정보**였을 것이다.

## 자주 발생하는 오류

**`= NULL`로 비교한다**

에러 없이 0행이 나온다. `IS NULL`을 쓴다.

**`NOT IN`을 습관적으로 쓴다**

서브쿼리 컬럼이 `NOT NULL`이라고 확신할 수 없으면 `NOT EXISTS`를 쓴다.

**`COUNT(*)`와 `COUNT(col)`을 같은 것으로 안다**

전자는 NULL 포함, 후자는 제외다. **행 수를 세려면 `COUNT(*)`.**

**논리 연산 우선순위를 헷갈린다**

`NOT` > `AND` > `OR` 순이다. 괄호로 명시하는 게 안전하다.

**DB 전용 함수를 쓴다**

같은 일을 하는 함수가 셋인데 이식성이 다르다.

| 함수 | 이식성 |
|---|---|
| `COALESCE(a, b)` | **표준 SQL — 모든 DBMS** |
| `NVL(a, b)` | Oracle 전용 |
| `ISNULL(a, b)` | SQL Server 전용 |

**`COALESCE`만 쓰면 된다.**

## 프로젝트 적용

**집계 지표를 만들 때 "빠진 값을 어떻게 셀 것인가"를 의도적으로 정한다.**

응답이 실패한 요청의 점수가 `NULL`이면 `AVG(score)`는 **성공한 것만의 평균**이 된다. 실패를 0점으로 셀지 모집단에서 제외할지는 **결정이어야 하는데, 아무것도 안 하면 DB가 대신 "제외"를 골라 버린다.**

```sql
-- 실패를 0점으로 세겠다면 명시한다
AVG(COALESCE(score, 0))
```

**"한 번도 ~하지 않은" 류의 분석은 전부 `NOT EXISTS`로 짠다.**

한 번도 로그인하지 않은 가입자, 아직 처리되지 않은 문서 — 이탈·미처리 분석의 기본 형태다. `NOT IN`으로 짜면 조용히 0행이 나온다.

**애초에 NULL이 안 생기게 하는 게 최선이다.**

컬럼에 `NOT NULL` 제약을 걸면 이 문제의 상당수가 사라진다. 값이 없는 상태를 정말 표현해야 하는지, 아니면 기본값으로 충분한지를 **테이블을 만들 때** 정하는 게 가장 싸다.

## 배운 점

**에러가 나는 버그가 좋은 버그다.**

이 글에 나온 함정은 전부 조용하다.

| 함정 | 증상 |
|---|---|
| `NOT IN` + NULL | 결과가 0행 (에러 없음) |
| `AVG` 분모 축소 | 평균이 부풀려짐 (에러 없음) |
| 외래키 NULL | 행이 사라짐 (에러 없음) |
| DB 이식 후 정렬 변화 | 순서가 바뀜 (에러 없음) |

에러가 나면 스택트레이스가 있고 고칠 지점이 있다. 그런데 **쿼리가 정상 실행되고 숫자만 틀리면** 아무도 모르고, 대시보드를 보고 판단을 내린 뒤에야 발견된다.

역설적으로 이번 실습에서 유일하게 **에러를 내 준 `division by zero`가 가장 다루기 쉬운 문제였다.** 터졌으니까 고쳤다.

그래서 이런 종류의 쿼리를 짤 때 습관을 하나 만들었다. **결과를 보기 전에 "몇 행이 나와야 하는가"를 먼저 말해 보는 것.** 채널별 주문 수의 합이 전체 주문 수와 같아야 한다는 걸 미리 알고 있으면, 안 맞을 때 바로 안다.
