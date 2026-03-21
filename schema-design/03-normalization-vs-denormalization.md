# 정규화 vs 비정규화 — 조인 비용과 데이터 중복의 트레이드오프

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 3NF 정규화가 조회 시 JOIN 비용을 증가시키는 원리는?
- 비정규화로 데이터를 중복 저장할 때 UPDATE 정합성 위험은?
- 읽기 중심 vs 쓰기 중심 서비스에서 정규화 수준을 어떻게 결정하는가?
- 비정규화가 실제로 조회 성능을 얼마나 향상시키는가?
- 부분적 비정규화(중요 컬럼만)의 설계 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### 정규화의 아이러니 — 데이터는 깔끔한데 쿼리가 느리다

```
실제 성능 이슈:
  주문 목록 API (완전 정규화 스키마):
  SELECT
    o.id, u.name, u.email,
    p.name AS product_name, p.price,
    oi.quantity, c.name AS category
  FROM orders o
  JOIN users u ON u.id = o.user_id
  JOIN order_items oi ON oi.order_id = o.id
  JOIN products p ON p.id = oi.product_id
  JOIN categories c ON c.id = p.category_id
  WHERE o.user_id = 1
  ORDER BY o.created_at DESC
  LIMIT 20;
  → 5 테이블 JOIN, 0.5초

  이 API가 초당 100번 호출 → 초당 50초 DB 부하

  비정규화 후:
  SELECT o.id, o.user_name, o.user_email,
         o.product_name, o.price, o.quantity, o.category_name
  FROM orders o
  WHERE o.user_id = 1
  ORDER BY o.created_at DESC
  LIMIT 20;
  → 단일 테이블, 0.01초

  트레이드오프: 사용자 이름이 바뀌면 orders 테이블도 UPDATE 필요
```

---

## 😱 흔한 실수 (Before)

### 1. 완전 정규화 고집으로 API 성능 문제

```sql
-- Before: 완전 정규화 스키마 (3NF)
-- 이론적으로 완벽하지만 조회 시 5개 테이블 JOIN

-- 주문 목록 쿼리:
SELECT o.id, u.name, p.name, p.price, oi.quantity, c.name AS category
FROM orders o
JOIN users u ON u.id = o.user_id
JOIN order_items oi ON oi.order_id = o.id
JOIN products p ON p.id = oi.product_id
JOIN categories c ON c.id = p.category_id;

-- 문제:
-- 각 JOIN마다 인덱스 탐색 비용
-- orders 100만 건 × 각 JOIN 비용 = 느린 응답
-- 인덱스 추가해도 JOIN 횟수는 줄지 않음
```

### 2. 비정규화 후 정합성 관리 없음

```sql
-- After: 비정규화했지만 정합성 관리 없음
ALTER TABLE orders ADD COLUMN user_name VARCHAR(100);

-- 이후 사용자 이름 변경 시:
UPDATE users SET name = '홍길동_새이름' WHERE id = 1;
-- orders.user_name은 업데이트 안 됨!
-- users.name과 orders.user_name 불일치

-- 조회 시 혼란:
SELECT o.id, u.name AS current_name, o.user_name AS order_time_name
FROM orders o JOIN users u ON u.id = o.user_id;
-- current_name과 order_time_name이 다름
```

### 3. 모든 테이블을 무조건 비정규화

```sql
-- Before: 정합성 중요한 데이터도 비정규화
-- 예: 가격 정보 비정규화
ALTER TABLE order_items ADD COLUMN product_price DECIMAL(10,2);

-- 상품 가격이 변경되면:
-- 기존 주문의 product_price를 바꿔야 할까? → 바꾸면 안 됨!
-- 주문 시점의 가격이 맞는 금액

-- 이 경우 비정규화가 오히려 올바름
-- (주문 시점 가격 보존 = 의도된 비정규화)
```

---

## ✨ 올바른 접근 (After)

```
정규화/비정규화 선택 기준:

정규화 선택 기준 (쓰기 중심):
  ① 데이터 변경이 잦음 (사용자 정보, 상품 재고)
  ② 데이터 정합성이 최우선 (금융, 결제)
  ③ 쓰기:읽기 = 5:5 이상
  ④ 실시간 최신 데이터 필요

비정규화 선택 기준 (읽기 중심):
  ① 읽기:쓰기 = 8:2 이상
  ② 동일 JOIN 패턴이 반복됨
  ③ 이미 JOIN 쿼리가 느린 것이 측정으로 확인됨
  ④ 중복 데이터의 변경 정책이 명확함

부분 비정규화 (현실적 접근):
  핵심 정규화: FK 관계, 자주 변경되는 데이터
  선택적 비정규화: 자주 JOIN되는 컬럼, 변경 드문 데이터
  예:
    users: id, name, email (정규화 유지)
    orders: id, user_id, user_name(비정규화), created_at
    → user_name은 주문 시점 기록용 (이름 변경 영향 없음)
```

---

## 🔬 내부 동작 원리

### 1. JOIN 비용 이해

```
JOIN 비용 구조:
  Nested Loop Join (일반적):
    Outer Table 스캔 × Inner Table 인덱스 탐색
    비용: Outer Rows × Inner Index Lookup Cost

  예시:
    orders (100만 건) JOIN users (10만 건)
    ON orders.user_id = users.id

    users.id에 PK 인덱스:
      100만 × (인덱스 탐색 1회) = 100만 인덱스 탐색
      B-Tree 높이 ~3 → 100만 × 3 = 300만 페이지 읽기

  JOIN 수 증가 시:
    5 테이블 JOIN:
      orders → users: 100만 인덱스 탐색
      orders → order_items: 100만 → 결과 300만 인덱스 탐색
      order_items → products: 300만 인덱스 탐색
      products → categories: 300만 인덱스 탐색
      총: 1,000만+ 인덱스 탐색
      → 메모리에서 처리되면 빠르지만 캐시 미스 시 느림

비정규화 후:
  단일 테이블 스캔:
    orders (user_name 포함)
    → 100만 × 1 = 100만 페이지 읽기
    → JOIN 오버헤드 없음
```

### 2. 정규화 이론 (1NF, 2NF, 3NF)

```
제1정규형 (1NF):
  원자값: 각 셀에 하나의 값만 (배열, 반복 그룹 금지)
  위반 예: tags = 'express,gift,fragile' → 분리 필요
  적용: 배열은 별도 테이블로 분리

제2정규형 (2NF):
  1NF + 부분 함수 종속 제거
  복합 PK의 일부에만 종속된 컬럼 분리
  예: (order_id, product_id) PK에서
      product_name은 product_id에만 종속 → products 테이블 분리

제3정규형 (3NF):
  2NF + 이행 함수 종속 제거
  PK가 아닌 컬럼이 다른 비PK 컬럼을 결정하면 분리
  예: users 테이블에 zip_code, city가 함께 있을 때
      zip_code → city (zip만 알면 city 결정됨)
      → zip_code, city를 별도 테이블로 분리

BCNF (Boyce-Codd Normal Form):
  3NF의 강화 버전
  모든 결정자가 후보 키여야 함
  실무에서는 3NF까지 적용 후 성능 고려

실무 관점:
  3NF까지 정규화 → 이론적 완벽함
  그 다음: 측정된 성능 문제 있는 곳만 선택적 비정규화
```

### 3. 비정규화 전략 패턴

```sql
-- 패턴 1: 집계값 사전 계산 (캐싱)
-- 상품별 리뷰 수, 평점 - 매번 COUNT 대신 저장
ALTER TABLE products ADD COLUMN review_count INT DEFAULT 0;
ALTER TABLE products ADD COLUMN avg_rating DECIMAL(3,2) DEFAULT 0.00;

-- 리뷰 INSERT 시 트리거 또는 애플리케이션에서 업데이트
CREATE TRIGGER after_review_insert
AFTER INSERT ON reviews
FOR EACH ROW
  UPDATE products
  SET review_count = review_count + 1,
      avg_rating = (avg_rating * (review_count - 1) + NEW.rating) / review_count
  WHERE id = NEW.product_id;

-- 패턴 2: 자주 사용되는 JOIN 컬럼 복사 (이름 등)
ALTER TABLE orders ADD COLUMN user_name VARCHAR(100);
ALTER TABLE orders ADD COLUMN product_name VARCHAR(200);
-- 주문 생성 시 당시 이름을 저장 (이력 보존)

-- 패턴 3: 파생 컬럼 저장 (계산 비용 절감)
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(12,2);
-- total_amount = SUM(order_items.price × quantity) 미리 계산
-- 매번 SUM 계산 불필요

-- 패턴 4: 요약 테이블 (Aggregate Table)
CREATE TABLE daily_sales_summary (
    sale_date   DATE PRIMARY KEY,
    total_sales DECIMAL(15,2),
    order_count INT,
    product_count INT
);
-- 주문 집계 쿼리 대신 요약 테이블 조회
```

### 4. 비정규화 정합성 유지 방법

```sql
-- 방법 1: 트리거 (자동 정합성)
CREATE TRIGGER after_user_update
AFTER UPDATE ON users
FOR EACH ROW
  IF OLD.name != NEW.name THEN
    UPDATE orders SET user_name = NEW.name
    WHERE user_id = NEW.id
    AND created_at > DATE_SUB(NOW(), INTERVAL 1 YEAR);  -- 최근 1년만
  END IF;

-- 방법 2: 애플리케이션 레벨 관리
@Transactional
public void updateUserName(Long userId, String newName) {
    userRepository.updateName(userId, newName);
    orderRepository.updateUserName(userId, newName);
    // 두 업데이트를 같은 트랜잭션에서
}

-- 방법 3: 이벤트 기반 동기화
-- User 변경 이벤트 → Kafka → Consumer → Order 업데이트
-- 비동기, 일시적 불일치 허용

-- 방법 4: 비정규화 포기 (정합성 불가능한 경우)
-- "주문 시점의 상품명/가격은 변경 불가" → 의도된 비정규화
-- 현재 상품명이 궁금하면 products 테이블 JOIN
-- 주문 시점 상품명은 order_items.product_name 사용
```

---

## 💻 실전 실험

### 실험 1: JOIN vs 비정규화 성능 비교

```sql
-- 정규화 스키마 (5 테이블)
CREATE TABLE u_users (id BIGINT PRIMARY KEY, name VARCHAR(100)) ENGINE=InnoDB;
CREATE TABLE u_orders (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id BIGINT, created_at DATETIME,
    INDEX idx_user (user_id, created_at)
) ENGINE=InnoDB;
CREATE TABLE u_products (id BIGINT PRIMARY KEY, name VARCHAR(200), category_id BIGINT) ENGINE=InnoDB;
CREATE TABLE u_order_items (
    id BIGINT AUTO_INCREMENT PRIMARY KEY,
    order_id BIGINT, product_id BIGINT, quantity INT, price DECIMAL(10,2),
    INDEX idx_order (order_id)
) ENGINE=InnoDB;
CREATE TABLE u_categories (id BIGINT PRIMARY KEY, name VARCHAR(100)) ENGINE=InnoDB;

-- 비정규화 스키마
CREATE TABLE d_orders (
    id           BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id      BIGINT,
    user_name    VARCHAR(100),    -- 비정규화
    product_name VARCHAR(200),    -- 비정규화
    category_name VARCHAR(100),   -- 비정규화
    quantity     INT,
    price        DECIMAL(10,2),
    created_at   DATETIME,
    INDEX idx_user (user_id, created_at)
) ENGINE=InnoDB;

-- 데이터 삽입 (생략: 각자 테스트 환경에서 삽입)

-- 성능 비교
SET @t = SYSDATE(6);
SELECT o.id, u.name, p.name, oi.quantity, c.name
FROM u_orders o
JOIN u_users u ON u.id = o.user_id
JOIN u_order_items oi ON oi.order_id = o.id
JOIN u_products p ON p.id = oi.product_id
JOIN u_categories c ON c.id = p.category_id
WHERE o.user_id = 1
ORDER BY o.created_at DESC LIMIT 20;
SELECT CONCAT('JOIN: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');

SET @t = SYSDATE(6);
SELECT id, user_name, product_name, category_name, quantity, price
FROM d_orders WHERE user_id = 1
ORDER BY created_at DESC LIMIT 20;
SELECT CONCAT('비정규화: ', TIMESTAMPDIFF(MICROSECOND, @t, SYSDATE(6))/1000, 'ms');
```

### 실험 2: 집계 캐싱 성능 비교

```sql
-- 매번 COUNT(*) vs 캐싱된 count
CREATE TABLE articles (
    id        BIGINT AUTO_INCREMENT PRIMARY KEY,
    title     VARCHAR(200),
    comment_count INT DEFAULT 0  -- 비정규화 캐시
);

CREATE TABLE comments (
    id         BIGINT AUTO_INCREMENT PRIMARY KEY,
    article_id BIGINT,
    content    TEXT,
    INDEX idx_article (article_id)
);

-- 매번 COUNT
EXPLAIN ANALYZE
SELECT a.id, a.title, COUNT(c.id) AS cnt
FROM articles a
LEFT JOIN comments c ON c.article_id = a.id
GROUP BY a.id, a.title;

-- 캐싱된 count
EXPLAIN ANALYZE
SELECT id, title, comment_count FROM articles;
-- 단순 스캔, JOIN 없음
```

---

## 📊 성능/비용 비교

```
정규화 vs 비정규화 성능 비교 (주문 목록, 100만 건):

5 테이블 JOIN 쿼리:
  단순 인덱스만 있을 때: 0.5~2초
  최적화 인덱스: 0.1~0.5초
  커버링 인덱스: 0.05~0.2초

단일 테이블 비정규화:
  인덱스 있음: 0.01~0.05초
  커버링 인덱스: 0.001~0.01초

성능 차이: 5~100배

쓰기 비용:
  정규화: 관련 테이블만 업데이트 (단순)
  비정규화: 비정규화된 모든 테이블 업데이트 필요

  users.name 변경:
  정규화: UPDATE users 1건
  비정규화: UPDATE users + UPDATE orders (100만 건 가능)
  → 비정규화 쓰기 비용 폭발 가능

  트레이드오프 측정:
    읽기 빈도 × 읽기 비용 절감 vs
    쓰기 빈도 × 쓰기 비용 증가
    → 읽기가 압도적으로 많으면 비정규화 정당화
```

---

## ⚖️ 트레이드오프

```
정규화 장단점:
  ✅ 데이터 정합성 보장
  ✅ UPDATE/DELETE 단순 (한 곳만)
  ✅ 스토리지 효율적
  ❌ 복잡한 JOIN 쿼리
  ❌ JOIN 비용 증가 → 읽기 성능 저하
  ❌ 캐시 효율 낮음 (여러 테이블)

비정규화 장단점:
  ✅ 빠른 읽기 (JOIN 없음)
  ✅ 단순한 쿼리
  ✅ 캐시 효율 높음
  ❌ 데이터 중복 → 저장 공간 증가
  ❌ 업데이트 시 여러 곳 관리 필요
  ❌ 정합성 오류 위험 증가

결정 프로세스:
  1. 먼저 정규화 (이론적 기준)
  2. 쿼리 성능 측정 (EXPLAIN ANALYZE)
  3. 성능 문제 있는 JOIN 패턴 식별
  4. 해당 JOIN 경로만 선택적 비정규화
  5. 비정규화 정합성 유지 방법 결정
  6. 모니터링 추가 (정합성 오류 탐지)

실무 경험칙:
  읽기:쓰기 > 10:1 → 비정규화 검토
  동일 JOIN 패턴이 초당 100회 이상 → 비정규화 효과 큼
  데이터 변경 빈도 낮음 → 비정규화 정합성 유지 쉬움
```

---

## 📌 핵심 정리

```
정규화 vs 비정규화 핵심:

정규화:
  1NF: 원자값 (배열 금지)
  2NF: 부분 종속 제거
  3NF: 이행 종속 제거
  → 데이터 정합성 최우선 서비스

비정규화 선택:
  읽기:쓰기 = 8:2 이상
  반복적인 동일 JOIN 패턴
  EXPLAIN ANALYZE로 JOIN 비용 측정 후 결정

비정규화 패턴:
  집계값 캐싱 (review_count, total_amount)
  JOIN 컬럼 복사 (user_name, product_name)
  요약 테이블 (daily_sales_summary)

비정규화 정합성 유지:
  트리거: 자동, DB 레벨 (성능 주의)
  애플리케이션 트랜잭션: 명시적, 확실
  이벤트 동기화: 비동기, 일시적 불일치 허용

원칙:
  "먼저 정규화, 측정 후 선택적 비정규화"
  비정규화 전 반드시 성능 측정으로 정당화
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 비정규화 설계에서 정합성 문제가 발생하는 시나리오와 해결 방법을 제시하라.

```sql
CREATE TABLE orders (
    id            BIGINT AUTO_INCREMENT PRIMARY KEY,
    user_id       BIGINT NOT NULL,
    user_name     VARCHAR(100),  -- 비정규화
    product_id    BIGINT NOT NULL,
    product_price DECIMAL(10,2), -- 비정규화
    created_at    DATETIME
);
```

<details>
<summary>해설 보기</summary>

**두 컬럼의 성격이 다릅니다**:

**user_name**: 사용자가 이름을 변경하면 불일치 발생.
- 시나리오: user_id=1의 name을 '김철수' → '김민수'로 변경, orders.user_name은 여전히 '김철수'
- 해결: 주문 시점 이름을 "이력"으로 보존할지, 최신 이름을 유지할지 정책 결정 필요
  - 이력 보존 (법적 서류): 변경 금지 (의도된 비정규화)
  - 최신 유지: 트리거 또는 이벤트로 동기화

**product_price**: **의도된 비정규화 (변경하면 안 됨)**
- 주문 시점의 가격이 중요 (결제 금액 기준)
- 상품 가격이 변경돼도 orders.product_price는 변경하면 안 됨
- 환불, 분쟁 해결의 근거 데이터
- 현재 가격이 필요하면 `products.price` 조회

```sql
-- user_name 정책: 최신 이름 유지 트리거
CREATE TRIGGER sync_user_name AFTER UPDATE ON users
FOR EACH ROW
  UPDATE orders SET user_name = NEW.name WHERE user_id = NEW.id;

-- product_price는 절대 변경하지 않음 (이력 보존)
-- 현재 가격: products.price
-- 주문 가격: orders.product_price
```

</details>

---

**Q2.** 읽기:쓰기 = 9:1인 게시판 서비스에서 게시글 목록 조회 시 댓글 수를 표시한다. 매번 COUNT JOIN을 하는 방식과 비정규화 캐싱 방식을 비교하고 어느 것을 선택할지 결정하라.

<details>
<summary>해설 보기</summary>

**성능 측정 비교**:

```sql
-- 방법 A: 매번 JOIN COUNT
SELECT a.id, a.title, COUNT(c.id) AS cnt
FROM articles a
LEFT JOIN comments c ON c.article_id = a.id
GROUP BY a.id, a.title
ORDER BY a.created_at DESC LIMIT 20;

-- 방법 B: 캐싱된 comment_count
SELECT id, title, comment_count
FROM articles ORDER BY created_at DESC LIMIT 20;
```

**선택 기준**:

읽기:쓰기 = 9:1이므로 댓글 수 표시를 위한 COUNT JOIN이 전체 트래픽의 90%를 차지합니다.

게시글 1만 건, 댓글 100만 건 기준:
- 방법 A: GROUP BY + COUNT JOIN → 매번 100만 건 집계 비용
- 방법 B: 단순 SELECT → 인덱스 스캔만 (20건)

**비정규화(방법 B) 선택** 이유:
- 읽기 빈도가 압도적으로 높음
- 댓글 입력/삭제(10%) 시만 article.comment_count UPDATE 필요
- 동기화 비용이 낮음 (댓글 입력 1건 → UPDATE 1건)

```sql
-- 비정규화 구현
ALTER TABLE articles ADD COLUMN comment_count INT DEFAULT 0;

-- 댓글 INSERT 시 카운터 증가
DELIMITER //
CREATE TRIGGER after_comment_insert AFTER INSERT ON comments
FOR EACH ROW
  UPDATE articles SET comment_count = comment_count + 1 WHERE id = NEW.article_id;

CREATE TRIGGER after_comment_delete AFTER DELETE ON comments
FOR EACH ROW
  UPDATE articles SET comment_count = comment_count - 1 WHERE id = OLD.article_id;
//
DELIMITER ;
```

이처럼 읽기 빈도가 높고 집계 쿼리 비용이 클 때는 비정규화 캐싱이 명확한 성능 이득을 줍니다.

</details>

---

<div align="center">

**[⬅️ JSON 타입](./02-json-type-generated-column.md)** | **[홈으로 🏠](../README.md)** | **[다음: AUTO_INCREMENT 함정 ➡️](./04-auto-increment-pitfalls.md)**

</div>
