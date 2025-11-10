# 🧠 Today I Learned — PostgreSQL Performance Tuning & Index Internals

최근 대규모 쿼리 실행 중 **데이터 로딩 지연**과 **타임아웃**이 발생하면서  
PostgreSQL에서 쿼리가 어떻게 실행되고, 왜 느려지는지,  
그리고 인덱스가 어떤 역할을 하는지 전반적으로 정리한 내용이다.

---

## 🚨 문제 상황 요약

- 특정 리스트 데이터를 불러오는 API에서 타임아웃 발생.
- 실행 계획을 보니 서브쿼리 중 하나가 **Full Sequential Scan**으로 동작 중.
- 이 서브쿼리는 `WHERE` 조건으로 특정 키 컬럼(`order_number`와 유사한 식별자)을 비교하고 있었지만,
  **인덱스가 적용되지 않아 테이블 전체를 반복적으로 탐색**하고 있었다.

---

## ⚙️ 원인 분석 요약

1️⃣ **인덱스가 없음**  
   - 해당 키 컬럼에 인덱스가 없거나,  
     인덱스가 존재하더라도 PostgreSQL이 인덱스를 사용하지 못하는 상황이었다.

2️⃣ **컬럼 타입 불일치 (`varchar` vs `text`)**  
   - 서로 다른 타입 간 비교 시 PostgreSQL은 자동으로 캐스팅(`::text`)을 수행한다.  
   - 이 경우, 인덱스는 더 이상 “원래 정의된 자료형”과 일치하지 않아 **무효화**된다.

3️⃣ **서브쿼리 구조 (Correlated Subquery)**  
   - 메인 테이블의 각 행마다 서브쿼리가 실행되는 구조였다.  
   - 예: 상위 테이블이 1000행이면, 서브쿼리도 1000번 실행됨.  
   - 인덱스가 없을 경우 → 매번 전체 테이블을 스캔해야 하므로 시간 복잡도는 O(N²)에 가까워진다.

---

## 🧩 PostgreSQL의 JOIN 및 WHERE 절 작동 방식

### 1. 기본 흐름
PostgreSQL이 JOIN을 수행할 때는 보통 다음 중 하나의 전략을 선택한다.

| 전략 | 설명 | 주로 쓰이는 상황 |
|------|------|----------------|
| **Nested Loop Join** | 한쪽 테이블의 각 행마다 다른 테이블을 탐색 | 작은 테이블 + 인덱스 있음 |
| **Hash Join** | 한쪽 테이블을 메모리에 해시 테이블로 저장하고 다른 쪽을 순회 | 양쪽 다 큼, 메모리에 올릴 수 있을 때 |
| **Merge Join** | 두 테이블을 정렬 후 순차 병합 | 둘 다 정렬된 상태이거나 인덱스로 정렬 가능할 때 |

---

### 2. Nested Loop Join의 실제 동작 예시

```sql
SELECT *
FROM orders o
LEFT JOIN notes n
  ON n.order_number = o.order_number;
```

1️⃣ `orders`에서 한 행을 읽는다.  
2️⃣ 그 행의 `order_number`를 이용해 `notes`를 탐색한다.  
3️⃣ 조건에 맞는 데이터를 찾으면 JOIN 결과로 합친다.  
4️⃣ 다음 행으로 넘어간다.

즉, 실제로는  
> “한쪽 테이블을 한 줄씩 읽으며, 다른 테이블에서 매번 ‘찾기’ 연산을 수행하는 구조”  

이다.

---

### 3. 인덱스의 존재 여부에 따른 차이

| 상황 | PostgreSQL 동작 | 결과 |
|------|------------------|------|
| 인덱스 없음 | 조건(`WHERE`)으로 테이블 전체 탐색 | Full Seq Scan |
| 인덱스 있음 | 정렬된 인덱스 트리에서 조건 검색 | Index Scan |

예를 들어 상위 테이블이 1000행, 하위 테이블이 5만 행이라면:
- **인덱스 없음:** 1000 × 5만 = 5천만 건 스캔 💣  
- **인덱스 있음:** 1000 × `O(logN)` 탐색 = 수백~수천 배 빠름 ⚡

---

## 🔍 인덱스의 개념과 동작 방식

### 1. 인덱스란?
> 데이터를 빠르게 찾기 위한 별도의 정렬된 자료구조.  
> PostgreSQL은 기본적으로 **B-Tree** 인덱스를 사용한다.

즉, 전체 테이블을 스캔하지 않고  
정렬된 트리를 타고 올라가 원하는 값을 **O(logN)** 시간에 찾아낸다.

---

### 2. 인덱스가 빠른 이유
- 인덱스는 실제 데이터 테이블과는 별도로 관리되는 **정렬된 키-포인터 구조**다.
- 쿼리가 특정 조건을 만족하는 행을 찾을 때,
  테이블 전체를 순회하지 않고 인덱스 트리에서 바로 위치를 찾는다.

---

### 3. 인덱스의 장단점

| 장점 | 단점 |
|------|------|
| 검색(`SELECT`, `JOIN`, `ORDER BY`) 속도 향상 | 쓰기(`INSERT/UPDATE/DELETE`) 시 인덱스도 갱신해야 해서 느려짐 |
| 정렬 비용 감소 | 디스크 공간 추가 사용 |
| Full Scan 회피 가능 | 너무 많으면 오히려 쿼리 계획 복잡해짐 |

---

## 🧠 인덱스가 적용되지 않는 대표적인 이유

| 원인 | 설명 |
|------|------|
| **타입 불일치** | `varchar` vs `text`처럼 연산자 다르면 인덱스 무시 |
| **함수/캐스팅 사용** | `col::text`, `lower(col)` 등은 인덱스 비활성화 |
| **와일드카드 검색** | `LIKE '%abc'`는 B-Tree 인덱스 못 탐색 |
| **조건 포괄적** | 결과가 너무 많으면 PostgreSQL이 Full Scan을 더 싸다고 판단 |
| **통계 오래됨** | 옵티마이저가 잘못된 선택을 하므로 `ANALYZE` 필요 |
| **콜레이션 불일치** | 정렬 기준(locale)이 다르면 인덱스 무시 가능 |

---

## ⚙️ 인덱스 종류 (PostgreSQL 기준)

| 종류 | 특징 | 사용 예시 |
|------|------|-----------|
| **B-Tree** | 일반적. `=`, `<`, `>`, `ORDER BY` 지원 | 대부분의 조건 검색 |
| **GIN** | JSONB, 배열, full-text 검색 | `@>`, `?`, `jsonb_ops` |
| **GiST** | 공간, 범위, 유사도 검색 | 위치, 거리, 근접 검색 |
| **BRIN** | 정렬된 큰 테이블 (시간순 등) | 로그, 시계열 데이터 |
| **Hash** | 단순 동등 비교 전용 | 거의 사용 안 함 |

---

## 🧱 인덱스 설계 예시

### 1️⃣ 단일 컬럼 인덱스
```sql
CREATE INDEX idx_notes_order_number
  ON notes (order_number);
```
- `WHERE order_number = ?` 빠르게 탐색

---

### 2️⃣ 복합 인덱스 (검색 + 정렬)
```sql
CREATE INDEX idx_notes_order_number_created
  ON notes (order_number, created_at DESC);
```
- `WHERE order_number = ? ORDER BY created_at DESC`  
  → 조건 + 정렬 동시 최적화

---

### 3️⃣ 부분 인덱스 (조건부 인덱스)
```sql
CREATE INDEX idx_notes_active
  ON notes (order_number)
  WHERE deleted_at IS NULL;
```
- 특정 조건(`deleted_at IS NULL`)만 인덱싱하여 크기 절감

---

### 4️⃣ 커버링 인덱스 (INCLUDE)
```sql
CREATE INDEX idx_notes_cover
  ON notes (order_number)
  INCLUDE (created_at, user_id);
```
- SELECT 시 인덱스만으로 응답 가능  
  (테이블 접근 없이 Index Only Scan)

---

## 🔎 성능 분석 방법 (EXPLAIN / ANALYZE)

```sql
EXPLAIN (ANALYZE, BUFFERS)
SELECT *
FROM notes
WHERE order_number = 'ORD123';
```

### 주요 항목
| 항목 | 의미 |
|------|------|
| **Seq Scan** | 테이블 전체를 훑는 스캔 |
| **Index Scan** | 인덱스를 사용한 탐색 |
| **Bitmap Index Scan** | 여러 조건 결합 시 효율적 |
| **cost** | 실행 비용 추정치 |
| **actual time** | 실제 실행 시간 |
| **rows / loops** | 반환 행 수 및 반복 횟수 |

---

## 🧩 인덱스 튜닝 순서 요약

1️⃣ **타입 통일**  
   - 비교 대상 컬럼 타입(`text`, `varchar`) 동일하게 맞추기.  
   - 서로 다른 타입이면 인덱스 무효.

2️⃣ **필요한 인덱스 추가**  
   - WHERE 절과 JOIN 조건에 쓰이는 컬럼에 인덱스 생성.  
   - 정렬 자주 쓴다면 복합 인덱스로 최적화.

3️⃣ **캐스팅 / 함수 제거**  
   - `col::text`, `lower(col)` 등은 인덱스 타지 않음.  
   - 가능하면 쿼리 수정 or 표현식 인덱스 생성.

4️⃣ **통계 갱신**
   ```sql
   ANALYZE notes;
   ```

5️⃣ **실행 계획 재확인**  
   - `EXPLAIN ANALYZE`에서 `Seq Scan → Index Scan`으로 바뀌면 성공.

---

## ⚡ 성능 차이 예시

| 상황 | 실행 계획 | 결과 |
|------|-------------|------|
| 인덱스 없음 | `Seq Scan (cost=0.00..1300.00)` | 수 초 단위 |
| 인덱스 있음 | `Index Scan using idx_notes_order_number_created` | 수 ms 단위 |

---

## 🚀 결론

이 문제의 핵심 원인은  
1️⃣ **인덱스 미존재 또는 타입 불일치로 인한 인덱스 비활성화**,  
2️⃣ **서브쿼리 반복 실행 구조**였다.

이를 해결하기 위한 핵심 조치는 아래 두 가지다.

```sql
-- (1) 타입 통일
ALTER TABLE notes
  ALTER COLUMN order_number TYPE text;

-- (2) 인덱스 생성
CREATE INDEX idx_notes_order_number_created
  ON notes (order_number, created_at DESC);
```

적용 후 실행 계획이 `Index Scan`으로 변경되면  
쿼리 속도는 10배 이상 개선될 가능성이 높다 🚀

---

## 🧭 배운 점 요약

| 주제 | 핵심 요약 |
|------|------------|
| **JOIN 작동 방식** | 한쪽 테이블 row 기준으로 다른 테이블을 반복 탐색 |
| **인덱스의 역할** | 탐색을 O(N) → O(logN)으로 줄임 |
| **타입 일치 중요성** | `varchar` vs `text` 비교 시 인덱스 무효 |
| **서브쿼리 주의** | 반복 실행 구조는 성능 저하의 주범 |
| **EXPLAIN 분석** | 실행 경로를 보고 병목 지점을 파악 가능 |
| **복합 인덱스** | 검색 + 정렬 최적화 가능 |
| **Partial / Expression 인덱스** | 조건부 혹은 함수 기반 쿼리에 대응 |
| **통계 관리** | `ANALYZE`로 옵티마이저 판단 정확도 향상 |

---

## ✅ 최종 인사이트

> 쿼리 최적화의 핵심은 “데이터를 덜 읽게 만드는 것”이다.  
> 인덱스는 그 핵심 도구이며,  
> 타입 불일치나 잘못된 쿼리 구조 하나만으로도  
> 인덱스는 쉽게 무력화된다.

이 경험을 통해,
- 인덱스는 단순한 옵션이 아니라 쿼리 설계의 기본이며,  
- 실행 계획(`EXPLAIN ANALYZE`)을 통해 병목을 식별하고  
- 스키마 설계(타입, 키 컬럼, 정렬 구조)를 함께 고려해야 함을 배웠다.
