# 인덱스 실습 예제 코드

> 목표: 인덱스 유무에 따라 실행 계획과 조회 성능이 어떻게 달라지는지 직접 확인한다.

이 실습은 MySQL 8.0 기준이다.

실습에서 확인할 내용은 다음이다.

```text
1. 인덱스가 없을 때와 있을 때의 차이
2. 카디널리티가 낮은 컬럼의 인덱스 효과
3. 다중 칼럼 인덱스의 컬럼 순서
4. LIKE 검색에서 인덱스를 사용할 수 있는 경우와 없는 경우
5. 인덱스 컬럼에 함수를 적용했을 때의 문제
6. 커버링 인덱스
7. IS NULL과 IS NOT NULL
```

---

# 1. 실습 준비

## 1.1 DB 생성

```sql
DROP DATABASE IF EXISTS index_study;
CREATE DATABASE index_study;
USE index_study;
```

---

## 1.2 테스트 테이블 생성

```sql
DROP TABLE IF EXISTS members;

CREATE TABLE members (
    id BIGINT NOT NULL AUTO_INCREMENT,
    email VARCHAR(100) NOT NULL,
    name VARCHAR(30) NOT NULL,
    gender CHAR(1) NOT NULL,
    status VARCHAR(20) NOT NULL,
    birth_date DATE NOT NULL,
    created_at DATETIME NOT NULL,
    deleted_at DATETIME NULL,
    theme_id INT NOT NULL,
    reservation_date DATE NOT NULL,
    score INT NOT NULL,
    PRIMARY KEY (id)
) ENGINE = InnoDB;
```

테이블 컬럼 설명:

| 컬럼 | 설명 | 인덱스 실습 포인트 |
|---|---|---|
| id | PK | 클러스터링 인덱스 |
| email | 거의 고유한 값 | 카디널리티 높은 컬럼 |
| gender | M/F | 카디널리티 낮은 컬럼 |
| status | READY/DONE/CANCEL | 카디널리티 낮은 컬럼 |
| created_at | 생성일 | 범위 검색 |
| deleted_at | NULL 포함 | IS NULL 실습 |
| theme_id | 테마 ID | 다중 칼럼 인덱스 |
| reservation_date | 예약 날짜 | 다중 칼럼 인덱스 |
| score | 점수 | 정렬, 범위 검색 |

---

# 2. 데이터 100만 건 넣기

## 2.1 숫자 생성용 임시 테이블 만들기

아래 방식은 0부터 999,999까지 숫자 100만 개를 만든 뒤, 이를 이용해 테스트 데이터를 넣는다.

```sql
DROP TABLE IF EXISTS numbers;

CREATE TABLE numbers (
    n INT NOT NULL PRIMARY KEY
) ENGINE = InnoDB;
```

```sql
INSERT INTO numbers(n)
SELECT
    ones.n
    + tens.n * 10
    + hundreds.n * 100
    + thousands.n * 1000
    + ten_thousands.n * 10000
    + hundred_thousands.n * 100000 AS n
FROM
    (SELECT 0 n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
     UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) ones
CROSS JOIN
    (SELECT 0 n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
     UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) tens
CROSS JOIN
    (SELECT 0 n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
     UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) hundreds
CROSS JOIN
    (SELECT 0 n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
     UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) thousands
CROSS JOIN
    (SELECT 0 n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
     UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) ten_thousands
CROSS JOIN
    (SELECT 0 n UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
     UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) hundred_thousands;
```

확인:

```sql
SELECT COUNT(*) FROM numbers;
```

예상 결과:

```text
1,000,000
```

---

## 2.2 members 테이블에 100만 건 삽입

```sql
INSERT INTO members (
    email,
    name,
    gender,
    status,
    birth_date,
    created_at,
    deleted_at,
    theme_id,
    reservation_date,
    score
)
SELECT
    CONCAT('user', LPAD(n, 7, '0'), '@test.com') AS email,
    CONCAT('user', n) AS name,
    IF(n % 2 = 0, 'M', 'F') AS gender,
    CASE n % 3
        WHEN 0 THEN 'READY'
        WHEN 1 THEN 'DONE'
        ELSE 'CANCEL'
    END AS status,
    DATE_ADD('1970-01-01', INTERVAL (n % 15000) DAY) AS birth_date,
    DATE_ADD('2020-01-01', INTERVAL (n % 2000) DAY) AS created_at,
    IF(n % 10 = 0, DATE_ADD('2024-01-01', INTERVAL (n % 365) DAY), NULL) AS deleted_at,
    (n % 100) + 1 AS theme_id,
    DATE_ADD('2026-01-01', INTERVAL (n % 365) DAY) AS reservation_date,
    n % 1000 AS score
FROM numbers;
```

확인:

```sql
SELECT COUNT(*) FROM members;
```

예상 결과:

```text
1,000,000
```

간단히 데이터 확인:

```sql
SELECT *
FROM members
LIMIT 5;
```

---

# 3. 실행 계획 확인 방법

MySQL 8.0에서는 다음 두 가지를 사용할 수 있다.

## 3.1 EXPLAIN

```sql
EXPLAIN
SELECT *
FROM members
WHERE email = 'user0500000@test.com';
```

주로 볼 컬럼:

| 컬럼 | 의미 |
|---|---|
| type | 테이블 접근 방식 |
| possible_keys | 사용할 수 있어 보이는 인덱스 |
| key | 실제 사용한 인덱스 |
| rows | 읽을 것으로 예상한 row 수 |
| Extra | 추가 정보 |

---

## 3.2 EXPLAIN ANALYZE

실제로 실행하면서 분석한다.

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE email = 'user0500000@test.com';
```

주의:

```text
EXPLAIN ANALYZE는 실제 쿼리를 실행한다.
INSERT, UPDATE, DELETE에는 함부로 사용하지 말자.
```

---

# 4. 실습 1: 인덱스가 없을 때

아직 `email` 인덱스가 없다.

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE email = 'user0500000@test.com';
```

확인할 것:

```text
type이 ALL에 가까운지
rows가 크게 나오는지
실행 시간이 긴지
```

예상 흐름:

```text
email 인덱스가 없음
→ 테이블 전체에서 email 값을 확인
→ 100만 건 가까이 읽을 수 있음
```

---

# 5. 실습 2: 단일 컬럼 인덱스 생성

## 5.1 email 인덱스 생성

```sql
CREATE INDEX idx_members_email
ON members(email);
```

인덱스 확인:

```sql
SHOW INDEX FROM members;
```

---

## 5.2 email 조건 재실행

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE email = 'user0500000@test.com';
```

확인할 것:

```text
key가 idx_members_email인지
rows가 크게 줄었는지
실행 시간이 줄었는지
```

예상 흐름:

```text
idx_members_email에서 user0500000@test.com 검색
→ 해당 row의 PK 확인
→ 실제 row 조회
```

정리:

> 카디널리티가 높은 컬럼은 인덱스 효과가 크다.

---

# 6. 실습 3: 카디널리티가 낮은 컬럼

`gender`는 값이 `M`, `F` 두 개뿐이다.

## 6.1 gender 인덱스 생성

```sql
CREATE INDEX idx_members_gender
ON members(gender);
```

---

## 6.2 gender 조건 조회

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE gender = 'M';
```

확인할 것:

```text
전체 100만 건 중 약 50만 건이 결과
인덱스를 쓰더라도 많은 row를 읽어야 함
```

비교용:

```sql
EXPLAIN ANALYZE
SELECT COUNT(*)
FROM members
WHERE gender = 'M';
```

정리:

> 인덱스는 결과 범위를 많이 줄여줄 때 효과가 크다.  
> 값의 종류가 적고 결과가 많은 컬럼은 인덱스 효과가 작을 수 있다.

---

# 7. 실습 4: 범위 검색

## 7.1 created_at 인덱스 생성

```sql
CREATE INDEX idx_members_created_at
ON members(created_at);
```

---

## 7.2 범위 조건 조회

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE created_at >= '2022-01-01'
  AND created_at < '2022-02-01';
```

확인할 것:

```text
idx_members_created_at 사용 여부
읽는 row 수
```

B-Tree는 정렬된 구조라서 시작 지점과 끝 지점을 찾을 수 있다.

```text
2022-01-01 ← 시작
...
2022-02-01 ← 종료
```

정리:

> B-Tree 인덱스는 `=`, `>`, `<`, `BETWEEN` 같은 범위 검색에 사용할 수 있다.

---

# 8. 실습 5: LIKE 검색

`email` 인덱스를 사용한다.

## 8.1 앞부분 일치

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE email LIKE 'user0500%';
```

예상:

```text
idx_members_email 사용 가능
```

이유:

```text
user0500으로 시작하는 범위를 찾을 수 있음
```

---

## 8.2 뒷부분 일치

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE email LIKE '%500000@test.com';
```

예상:

```text
인덱스를 작업 범위 결정 조건으로 사용하기 어려움
```

이유:

```text
앞부분을 알 수 없어서 B-Tree에서 시작 지점을 잡기 어려움
```

정리:

| 조건 | 인덱스 활용 |
|---|---|
| `LIKE 'abc%'` | 좋음 |
| `LIKE '%abc'` | 어려움 |
| `LIKE '%abc%'` | 어려움 |

---

# 9. 실습 6: 인덱스 컬럼에 함수 적용

`created_at` 인덱스를 사용한다.

## 9.1 함수 사용

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE YEAR(created_at) = 2022;
```

예상:

```text
idx_members_created_at을 작업 범위 결정 조건으로 사용하기 어려움
```

이유:

```text
인덱스는 created_at 원본 값으로 정렬되어 있음
하지만 쿼리는 YEAR(created_at) 결과로 비교함
```

---

## 9.2 범위 조건으로 개선

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE created_at >= '2022-01-01'
  AND created_at < '2023-01-01';
```

예상:

```text
idx_members_created_at 사용 가능
```

정리:

> 인덱스 컬럼은 가능하면 가공하지 말고 원본 컬럼 그대로 비교하자.

---

# 10. 실습 7: 다중 칼럼 인덱스

## 10.1 다중 칼럼 인덱스 생성

```sql
CREATE INDEX idx_members_theme_date
ON members(theme_id, reservation_date);
```

인덱스 구조는 다음처럼 이해하면 된다.

```text
1차 정렬: theme_id
2차 정렬: reservation_date
```

---

## 10.2 선행 컬럼을 사용하는 경우

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE theme_id = 10
  AND reservation_date = '2026-05-01';
```

예상:

```text
idx_members_theme_date 사용 가능
```

---

## 10.3 선행 컬럼만 사용하는 경우

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE theme_id = 10;
```

예상:

```text
idx_members_theme_date 사용 가능
```

---

## 10.4 선행 컬럼이 빠진 경우

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE reservation_date = '2026-05-01';
```

예상:

```text
idx_members_theme_date를 효율적으로 사용하기 어려움
```

이유:

```text
인덱스가 theme_id 기준으로 먼저 정렬되어 있기 때문
```

---

## 10.5 조건 순서는 중요할까?

다음 두 쿼리를 비교한다.

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE theme_id = 10
  AND reservation_date = '2026-05-01';
```

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE reservation_date = '2026-05-01'
  AND theme_id = 10;
```

예상:

```text
둘 다 같은 인덱스를 사용할 가능성이 높음
```

정리:

```text
중요하지 않은 것
→ WHERE 절에 적은 조건 순서

중요한 것
→ 인덱스를 생성한 컬럼 순서
```

---

# 11. 실습 8: 커버링 인덱스

커버링 인덱스는 쿼리에 필요한 컬럼을 인덱스만으로 해결할 수 있는 경우다.

## 11.1 인덱스만으로 처리 가능한 쿼리

```sql
EXPLAIN ANALYZE
SELECT email
FROM members
WHERE email = 'user0500000@test.com';
```

확인할 것:

```text
Extra에 Using index가 나오는지
```

`idx_members_email`에는 email 값이 들어 있다.

따라서 `email`만 조회하면 실제 row를 다시 읽지 않아도 될 수 있다.

---

## 11.2 실제 row 조회가 필요한 쿼리

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE email = 'user0500000@test.com';
```

이 쿼리는 모든 컬럼이 필요하다.

따라서 email 인덱스에서 PK를 찾은 뒤 실제 row를 다시 조회해야 한다.

정리:

> 커버링 인덱스는 실제 row 조회를 줄일 수 있어서 빠를 수 있다.

---

# 12. 실습 9: IS NULL과 IS NOT NULL

MySQL에서는 NULL 값도 인덱스에 저장된다.

## 12.1 deleted_at 인덱스 생성

```sql
CREATE INDEX idx_members_deleted_at
ON members(deleted_at);
```

---

## 12.2 IS NULL

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE deleted_at IS NULL;
```

주의:

```text
deleted_at IS NULL인 데이터가 많으면 인덱스 효율이 낮을 수 있음
```

이 실습 데이터에서는 90%가 NULL이다.

따라서 인덱스를 사용하더라도 많은 row를 읽을 수 있다.

---

## 12.3 IS NOT NULL

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE deleted_at IS NOT NULL;
```

이 실습 데이터에서는 10%가 NOT NULL이다.

경우에 따라 `IS NOT NULL`이 더 적은 데이터를 읽을 수도 있다.

다만 B-Tree 가용성 설명에서는 일반적으로 `IS NOT NULL`은 작업 범위 결정 조건으로 효율이 떨어질 수 있음을 기억한다.

정리:

> MySQL은 NULL도 인덱스에 저장한다.  
> 하지만 실제 효율은 NULL 값의 비율에 따라 달라진다.

---

# 13. 실습 10: 타입 불일치

## 13.1 문자열 컬럼을 숫자로 비교

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE email = 123;
```

이 쿼리는 좋은 예가 아니다.

`email`은 문자열 컬럼인데 숫자와 비교하고 있다.

타입 변환이 발생할 수 있고, 인덱스 사용이 불리해질 수 있다.

---

## 13.2 타입을 맞춰 비교

```sql
EXPLAIN ANALYZE
SELECT *
FROM members
WHERE email = 'user0000123@test.com';
```

정리:

> 인덱스 컬럼과 비교 값의 타입을 맞추자.

---

# 14. 실습 후 정리 쿼리

생성된 인덱스 확인:

```sql
SHOW INDEX FROM members;
```

테이블 크기 확인:

```sql
SELECT
    table_name,
    table_rows,
    ROUND(data_length / 1024 / 1024, 2) AS data_mb,
    ROUND(index_length / 1024 / 1024, 2) AS index_mb
FROM information_schema.tables
WHERE table_schema = 'index_study'
  AND table_name = 'members';
```

실습 DB 삭제:

```sql
DROP DATABASE IF EXISTS index_study;
```

---

# 15. 실습 진행 순서 요약

```text
1. DB 생성
2. members 테이블 생성
3. numbers 테이블로 100만 건 데이터 생성
4. 인덱스 없는 상태에서 email 조회
5. email 인덱스 생성 후 조회 비교
6. gender처럼 카디널리티 낮은 컬럼 확인
7. created_at 범위 검색 확인
8. LIKE 'abc%'와 LIKE '%abc' 비교
9. YEAR(created_at)과 범위 조건 비교
10. 다중 칼럼 인덱스 순서 확인
11. 커버링 인덱스 확인
12. IS NULL, IS NOT NULL 확인
13. 타입 불일치 비교
```

---

# 16. 관찰 포인트

각 실습마다 다음을 확인한다.

```text
1. 어떤 인덱스를 사용했는가?
2. 예상 rows가 얼마나 줄었는가?
3. 실제 실행 시간은 어떻게 달라졌는가?
4. Extra에 어떤 정보가 나오는가?
```

특히 다음 컬럼을 집중해서 본다.

| 컬럼 | 볼 내용 |
|---|---|
| type | ALL인지, range인지, ref인지 |
| key | 실제 사용된 인덱스 |
| rows | 읽을 것으로 예상한 row 수 |
| Extra | Using index, Using where 등 |

---

# 17. 최종 정리

이번 실습의 핵심은 다음이다.

```text
인덱스가 없으면 많이 읽는다.
인덱스가 있으면 읽을 범위를 줄일 수 있다.
하지만 인덱스가 항상 좋은 것은 아니다.
```

B-Tree 인덱스는 다음 조건에 강하다.

```text
=
>, <
BETWEEN
LIKE 'abc%'
IS NULL
```

다음 조건은 주의해야 한다.

```text
<>
NOT IN
IS NOT NULL
LIKE '%abc'
인덱스 컬럼에 함수 적용
타입 불일치
복합 인덱스의 선행 컬럼 누락
```

마지막으로 한 문장으로 정리하면:

> 인덱스 튜닝은 무조건 인덱스를 많이 만드는 것이 아니라, 쿼리가 읽어야 하는 데이터 범위를 줄이도록 인덱스와 조건을 맞추는 작업이다.
