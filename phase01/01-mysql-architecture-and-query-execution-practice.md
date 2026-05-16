# MySQL 기본 구조와 실행 계획

## 인덱스(Index) 유무에 따른 MySQL 실행 계획 비교 실험

## 실험 목표

- 인덱스 유무에 따라, MySQL의 실행 계획이 어떻게 달라지는지 확인한다.
- 같은 데이터를 조회하더라도 조건과 인덱스에 따라, 옵티마이저가 어떤 실행 방식을 선택하는지 `EXPLAIN`으로 비교하고 해석한다.

## 실험 흐름

1. 10만 건의 테스트 데이터를 생성한다.
2. [Before] 인덱스가 없는 상태에서 특정 조건으로 `SELECT` 쿼리를 실행하고 `EXPLAIN`을 확인한다.
3. 해당 조건 컬럼에 인덱스를 추가한다.
4. [After] 같은 쿼리를 다시 실행하여 `EXPLAIN` 결과와 실제 실행 시간을 비교한다.

---

## 0. 실험 준비 (테이블 및 데이터 생성)

실험에 사용할 사원(`employees`) 테이블을 생성하고, MySQL 8.0의 CTE(재귀 쿼리)를 이용해 10만 건의 더미 데이터를 삽입한다.

#### 1. 실험용 테이블 생성 (인덱스 없음)

```sql
CREATE TABLE employees
(
    emp_no          INT AUTO_INCREMENT PRIMARY KEY,
    emp_name        VARCHAR(50) NOT NULL,
    department_code INT         NOT NULL,
    created_at      DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

#### 2. 10만 건 더미 데이터 삽입 (CTE 활용)

```sql
SET SESSION cte_max_recursion_depth = 200000;

INSERT INTO employees (emp_name, department_code)
WITH RECURSIVE cte (n) AS (SELECT 1
                           UNION ALL
                           SELECT n + 1
                           FROM cte
                           WHERE n < 100000)
SELECT CONCAT('Employee_', n), -- 이름: Employee_1, Employee_2 ...
       FLOOR(RAND() * 100) + 1 -- 부서코드: 1 ~ 100 사이의 랜덤 숫자
FROM cte;

-- 3. 데이터 건수 확인
SELECT COUNT(*)
FROM employees; -- 결과: 100,000건
```

---

## 실험 1: 부서 코드(department_code) 검색 비교

중복된 값이 많은 컬럼(1~100 사이의 랜덤 값)을 검색할 때의 차이를 확인한다.

### [Before]: 인덱스가 없을 때

```SQL
EXPLAIN
SELECT *
FROM employees
WHERE department_code = 50;
```

- type: `ALL` => Full Table Scan (풀 테이블 스캔)
    - 옵티마이저가 선택할 수 있는 최악의 수. 목차가 없어서 책의 첫 페이지부터 끝 페이지까지 10만 줄을 다 읽었다는 뜻이다.
- possible_keys, key: `NULL`
    - 사용할 수 있는 인덱스도, 실제 사용한 인덱스도 없다.
- rows: `100066`
    - 이 결과를 찾기 위해 옵티마이저가 약 10만 건을 모두 읽어야 한다고 예측했다.

### [After]: 인덱스 추가 후

```SQL
CREATE INDEX idx_department_code ON employees (department_code);
EXPLAIN
SELECT *
FROM employees
WHERE department_code = 50;
```

- type: `ref` => Non-Unique Index Scan (비고유 인덱스 스캔)
    - 테이블을 다 뒤지지 않고, 우리가 만든 인덱스(목차)를 타고 들어가서 50번 부서 데이터가 모여있는 곳만 뽑아왔다는 뜻이다.
    - ALL에서 ref로 바뀐 것이 성능 개선의 핵심이다.
- key: `idx_department_code`
    - 방금 생성한 인덱스를 옵티마이저가 정상적으로 사용했다.
- rows: `969` (정확한 수치가 아니라, 매번 다를 수 있음)
    - 10만 건을 뒤지던 예측치가 969건으로 획기적으로 줄어들었다.
    - 랜덤 값인 이유는?
        - 옵티마이저가 통계 데이터를 기반으로 예측한 추정치이기 때문이다.
        - InnoDB 엔진은 속도를 위해 매번 정확한 개수를 세지 않고, 대략적인 통계를 사용한다.

---

## 실험 2: 사원 이름(emp_name) 검색 비교

고유값에 가까운 특정 사원 1명을 검색할 때, 실제 실행 시간(EXPLAIN ANALYZE)이 어떻게 변하는지 비교한다.

### [Before]: 인덱스가 없을 때

```SQL
EXPLAIN
ANALYZE
SELECT *
FROM employees
WHERE emp_name = 'Employee_50000';
```

```
-> Filter: (employees.emp_name = 'Employee_50000') (cost=10095 rows=10007) (actual time=22.4..40.4 rows=1 loops=1)
    -> Table scan on employees (cost=10095 rows=100066) (actual time=0.146..25.1 rows=100000 loops=1)
```

- 해석: 옵티마이저가 Table scan(풀 테이블 스캔)을 선택했다.
    - Employee_50000이라는 사람 1명(rows=1)을 찾기 위해, 디스크에서 10만 건의 데이터(rows=100000)를 일일이 다 꺼내서 확인했다.

- 시간: 전체 데이터를 스캔하고 결과를 반환하는 데 약 25.1ms ~ 40.4ms가 소요되었다.

### [After]: 인덱스 추가 후

```SQL
CREATE INDEX idx_emp_name ON employees (emp_name);
EXPLAIN
ANALYZE
SELECT *
FROM employees
WHERE emp_name = 'Employee_50000';
```

```
-> Index lookup on employees using idx_emp_name (emp_name='Employee_50000') (cost=0.35 rows=1) (actual time=0.0337..0.036 rows=1 loops=1)
```

- 해석: 옵티마이저가 Index lookup(인덱스 탐색)을 선택했다. 인덱스(목차)를 보고 해당 데이터가 있는 위치로 찾아가서, 딱 1건(rows=1)만 꺼내왔다.

- 비용(Cost): 예상 작업 비용이 10095에서 0.35로 엄청나게 감소했다.

- 시간: 탐색 시간이 0.036ms로 끝났다. (인덱스 없을 때 대비 약 1,000배 이상 빨라짐!)

---

## EXPLAIN 결과 읽는 법 요약

실무에서 쿼리 튜닝을 할 때, EXPLAIN을 찍어보고 다음 3가지를 중점적으로 확인하면 된다.

**1. `type` 컬럼 (스캔 방식)**

- ALL이 보인다면 위험 신호다. (인덱스가 없거나, 있어도 못 타는 상황)
- `ref`, `eq_ref`, `const`, `range` 등으로 바뀌도록 인덱스를 설계해야 한다.
    - 참고) 성능 순서: system > const > eq_ref > ref > fulltext > ref_or_null > index_merge > unique_subquery >
      index_subquery > range > index > ALL
    - 일반적으로 `range`, `ref`, `eq_ref`, `const` 등이 효율적인 접근 방식으로 본다.

**2. `key` 컬럼 (사용된 인덱스)**

- `NULL`이면 인덱스를 못 탔다는 뜻이다. 내가 의도한 인덱스 이름이 적혀 있는지 확인해야 한다.

**3. `actual time` (실제 소요 시간 - EXPLAIN ANALYZE 사용 시)**

- `actual time=시작시간..종료시간` 을 의미한다.
- `actual time=0.0337..0.036` 은 첫 번째 데이터를 찾는 데 `0.0337ms`가 걸렸고, 조건에 맞는 모든 데이터를 찾는 데 총 `0.036ms`가 걸렸다는 뜻이다.

**4. -> 기호와 들여쓰기 (실행 순서 파악)**

- `EXPLAIN ANALYZE` 결과는 안쪽으로 들여쓰기가 깊을수록 먼저 실행되는 트리 구조를 가진다.
- `->` 기호는 하위(안쪽) 작업이 완료되어, 상위(바깥쪽) 작업으로 데이터를 올려보낸다는 의미다.
- 읽는 법: 가장 안쪽(오른쪽)으로 들여쓰기 된 줄부터 먼저 읽고, 바깥쪽(왼쪽 위)으로 읽어 올라오면 된다.
- 예시: 풀 테이블 스캔 결과

```
-> Table scan... (들여쓰기가 더 되어 있음, 먼저 실행): 일단 employees 테이블의 10만 건 데이터를 싹 다 읽어 들인다.
-> Filter... (바깥쪽, 나중에 실행): 위에서 읽어 올린 10만 건의 데이터들을 하나하나 필터링 하며, emp_name이 'Employee_50000'인 데이터만 걸러낸다."
```
