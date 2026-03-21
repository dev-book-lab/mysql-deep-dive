# Spring에서 Read/Write 분리 — AbstractRoutingDataSource와 Replica 지연 대응

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AbstractRoutingDataSource`로 `@Transactional(readOnly=true)`를 Replica로 라우팅하는 구현 패턴은?
- Replica Lag으로 인한 Stale Read 문제를 트랜잭션 전파 설정으로 방어하는 전략은?
- 쓰기 직후 읽기에서 Stale Read가 발생하지 않도록 보장하는 방법은?
- 여러 Replica에 부하를 분산하는 방법은?
- Source 장애 시 Failover와 Spring DataSource 재설정 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

### Replica Lag을 모르고 Read/Write 분리하면 사용자 혼란

```
잘못된 Read/Write 분리 구현의 결과:
  Write: Source (INSERT/UPDATE)
  Read: Replica (SELECT) ← 항상

  사용자 경험:
    T=0: 주문 등록 → Source INSERT 완료
    T=0: 클라이언트에 "주문 완료" 응답
    T=0.1: 주문 목록 조회 → Replica SELECT
    T=0.1: Replica Lag 2초 → 방금 등록한 주문 없음!
    → "내가 주문했는데 왜 목록에 없어요?" → CS 문의 급증

올바른 설계 원칙:
  ① 쓰기 작업 + 같은 트랜잭션의 읽기 → Source
  ② 독립적인 읽기 트랜잭션 (Lag 허용) → Replica
  ③ 중요 읽기 (결제 전 재고, 중복 확인) → Source 강제

핵심:
  @Transactional(readOnly=true) → Replica
  @Transactional(readOnly=false) → Source
  쓰기 후 같은 트랜잭션 내 읽기 → Source (트랜잭션 전파로 보장)
```

---

## 😱 흔한 실수 (Before)

### 1. 쓰기 후 즉시 읽기에서 Stale Read 발생

```java
// Before: Stale Read가 발생하는 패턴

@Service
public class OrderService {

    @Transactional  // Source 사용 (쓰기)
    public OrderResponse createOrder(OrderRequest req) {
        Order order = orderRepository.save(new Order(req));
        // 바로 상세 조회 (다른 메서드 호출)
        return getOrderDetail(order.getId());  // Stale Read 위험!
    }

    @Transactional(readOnly = true)  // Replica 사용 (읽기)
    public OrderResponse getOrderDetail(Long orderId) {
        // 방금 저장했지만 Replica에 아직 없을 수 있음
        Order order = orderRepository.findById(orderId)
            .orElseThrow();  // 없으면 예외!
        return new OrderResponse(order);
    }
}

// 문제:
// createOrder()는 Source, getOrderDetail()은 Replica
// @Transactional AOP vs readOnly AOP 실행 순서에 따라
// getOrderDetail()이 별도 Replica 커넥션을 획득할 수 있음
```

### 2. LazyConnectionDataSourceProxy 없이 readOnly 라우팅

```java
// Before: 일반 DataSource로 라우팅 설정

@Bean
public DataSource routingDataSource(...) {
    RoutingDataSource routing = new RoutingDataSource();
    // ... 설정
    return routing;  // LazyConnectionDataSourceProxy 없음!
}

// 문제:
// Spring 트랜잭션은 시작 시점에 커넥션을 획득
// 이 시점에 readOnly 플래그가 아직 설정 안 됨
// → determineCurrentLookupKey() 호출 시 readOnly 정보 없음
// → 항상 Source DataSource 선택 (기본값)
// → Replica 라우팅이 동작하지 않음

// 해결: LazyConnectionDataSourceProxy 필수
// 실제 쿼리 실행 시점까지 커넥션 획득 지연
// → readOnly 플래그 확인 가능
```

### 3. AOP 순서를 고려하지 않음

```java
// Before: @Order 없이 AOP 설정

@Aspect
@Component
// @Order 없음 → 실행 순서 불확실
public class ReadOnlyDataSourceAspect {
    @Around("@annotation(Transactional)")
    public Object route(ProceedingJoinPoint pjp) throws Throwable {
        // DataSource 결정
    }
}

// 문제:
// @Transactional AOP가 먼저 실행되면:
//   1. 트랜잭션 시작 → 커넥션 획득 (readOnly 미설정)
//   2. DataSource AOP → DataSource 결정 (이미 커넥션 있음)
//   → 라우팅 의미 없음

// 해결:
@Order(Ordered.HIGHEST_PRECEDENCE)  // 트랜잭션보다 먼저 실행
public class ReadOnlyDataSourceAspect { ... }
```

---

## ✨ 올바른 접근 (After)

```
정확한 구현 구조:

LazyConnectionDataSourceProxy (필수):
  실제 쿼리 실행 시점까지 커넥션 획득 지연
  → @Transactional 시작 후 readOnly 플래그 확인 가능

AOP 순서 (@Order):
  DataSource 라우팅 AOP: HIGHEST_PRECEDENCE
  @Transactional AOP: 그 다음
  순서: DataSource 결정 → 트랜잭션 시작 → 커넥션 획득

트랜잭션 전파로 Stale Read 방어:
  쓰기 메서드 (Source): @Transactional
  쓰기 내에서 호출되는 읽기: @Transactional(readOnly=true, propagation=REQUIRED)
  → REQUIRED: 부모 트랜잭션(Source) 참여 → Source에서 읽기 보장

독립 읽기 (Replica 허용):
  @Transactional(readOnly=true, propagation=REQUIRES_NEW)
  또는 @Transactional(readOnly=true) (새 트랜잭션 시작 시)
  → 새 트랜잭션 → Replica DataSource 선택
```

---

## 🔬 내부 동작 원리

### 1. AbstractRoutingDataSource 구현

```java
// 1. DataSource 타입 정의
public enum DataSourceType {
    SOURCE, REPLICA
}

// 2. ThreadLocal로 현재 스레드의 DataSource 타입 관리
public class DataSourceContextHolder {
    private static final ThreadLocal<DataSourceType> context = new ThreadLocal<>();

    public static void set(DataSourceType type) { context.set(type); }
    public static DataSourceType get() { return context.get(); }
    public static void clear() { context.remove(); }
}

// 3. 라우팅 DataSource 구현
@Component
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Override
    protected Object determineCurrentLookupKey() {
        DataSourceType type = DataSourceContextHolder.get();
        return (type != null) ? type : DataSourceType.SOURCE;  // 기본: Source
    }
}

// 4. DataSource Bean 설정
@Configuration
public class DataSourceConfig {

    @Bean
    @ConfigurationProperties("spring.datasource.source")
    public DataSource sourceDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    @ConfigurationProperties("spring.datasource.replica")
    public DataSource replicaDataSource() {
        return DataSourceBuilder.create().type(HikariDataSource.class).build();
    }

    @Bean
    public DataSource routingDataSource(
            @Qualifier("sourceDataSource") DataSource source,
            @Qualifier("replicaDataSource") DataSource replica) {

        RoutingDataSource routing = new RoutingDataSource();
        Map<Object, Object> sources = new HashMap<>();
        sources.put(DataSourceType.SOURCE, source);
        sources.put(DataSourceType.REPLICA, replica);
        routing.setTargetDataSources(sources);
        routing.setDefaultTargetDataSource(source);
        return routing;
    }

    @Bean
    @Primary
    public DataSource dataSource(@Qualifier("routingDataSource") DataSource routing) {
        // LazyConnectionDataSourceProxy: 커넥션 획득을 쿼리 시점으로 지연
        // readOnly 플래그 확인이 가능해짐
        return new LazyConnectionDataSourceProxy(routing);
    }
}
```

### 2. AOP 기반 자동 라우팅

```java
@Aspect
@Component
@Order(Ordered.HIGHEST_PRECEDENCE)  // 트랜잭션 AOP보다 먼저!
public class ReadOnlyDataSourceRoutingAspect {

    // @Transactional이 붙은 모든 메서드에 적용
    @Around("@annotation(transactional)")
    public Object route(ProceedingJoinPoint pjp,
                        Transactional transactional) throws Throwable {

        DataSourceType type = transactional.readOnly()
            ? DataSourceType.REPLICA
            : DataSourceType.SOURCE;

        DataSourceContextHolder.set(type);
        try {
            return pjp.proceed();
        } finally {
            DataSourceContextHolder.clear();
        }
    }
}

// 사용:
@Service
public class OrderService {

    @Transactional              // → Source (쓰기)
    public Order createOrder(OrderRequest req) {
        return orderRepository.save(new Order(req));
    }

    @Transactional(readOnly = true)  // → Replica (읽기, Lag 허용)
    public List<Order> getOrders(Long userId) {
        return orderRepository.findByUserId(userId);
    }

    @Transactional(readOnly = true)  // → Source 강제 (중요 읽기)
    @SourceOnly  // 커스텀 어노테이션으로 Source 강제
    public int getStockCount(Long productId) {
        return stockRepository.countByProductId(productId);
    }
}
```

### 3. Stale Read 방어 — 트랜잭션 전파 설정

```java
@Service
@Transactional  // 기본: Source
public class OrderService {

    // 쓰기 후 즉시 읽기 — Stale Read 방어
    @Transactional
    public OrderResponse processOrder(OrderRequest req) {
        // 1. 주문 저장 (Source)
        Order order = orderRepository.save(new Order(req));

        // 2. 방금 저장한 주문 상세 조회
        // → 이 메서드도 Source에서 읽어야 함!
        return getOrderDetailForSource(order.getId());
    }

    // 부모 트랜잭션(Source)에 참여 → Source에서 읽기
    // Propagation.REQUIRED: 기존 트랜잭션 있으면 참여
    @Transactional(readOnly = true, propagation = Propagation.REQUIRED)
    public OrderResponse getOrderDetailForSource(Long orderId) {
        // DataSource 라우팅:
        // → readOnly=true이므로 Replica 라우팅 시도
        // → 하지만 부모 트랜잭션이 Source를 사용 중
        // → LazyConnectionDataSourceProxy가 기존 Source 커넥션 재사용
        // → Source에서 읽기 보장 (Stale Read 없음)
        return orderRepository.findById(orderId)
            .map(OrderResponse::new)
            .orElseThrow();
    }

    // 독립 읽기 — Replica 사용 (Lag 허용)
    // Propagation.REQUIRES_NEW: 항상 새 트랜잭션 시작
    @Transactional(readOnly = true, propagation = Propagation.REQUIRES_NEW)
    public List<Order> getRecentOrders(Long userId) {
        // 새 트랜잭션 → Replica DataSource 선택
        return orderRepository.findByUserIdOrderByCreatedAtDesc(userId);
    }
}

// 핵심 원리:
// REQUIRED + 부모가 Source → 커넥션 재사용 → Source 읽기
// REQUIRES_NEW → 새 커넥션 획득 → readOnly=true → Replica 선택
```

### 4. 여러 Replica 간 부하 분산

```java
// Round Robin으로 여러 Replica 분산
@Component
public class ReplicaLoadBalancer {

    private final List<DataSource> replicas;
    private final AtomicInteger counter = new AtomicInteger(0);

    public ReplicaLoadBalancer(
            @Qualifier("replica1") DataSource r1,
            @Qualifier("replica2") DataSource r2,
            @Qualifier("replica3") DataSource r3) {
        this.replicas = List.of(r1, r2, r3);
    }

    public DataSource getNext() {
        int idx = Math.abs(counter.getAndIncrement() % replicas.size());
        return replicas.get(idx);
    }
}

// RoutingDataSource에서 Replica 키를 동적으로 결정
@Component
public class RoutingDataSource extends AbstractRoutingDataSource {

    @Autowired
    private ReplicaLoadBalancer balancer;

    @Override
    protected Object determineCurrentLookupKey() {
        DataSourceType type = DataSourceContextHolder.get();
        if (type == DataSourceType.REPLICA) {
            // Round Robin으로 Replica 선택
            // 키: "replica_0", "replica_1", "replica_2"
            int idx = Math.abs(counter.getAndIncrement() % replicaCount);
            return "replica_" + idx;
        }
        return DataSourceType.SOURCE;
    }
}
```

### 5. Source 장애 시 Failover 대응

```java
// 방법 1: ProxySQL 사용 (권장)
// ProxySQL이 DB 레벨에서 Source/Replica 자동 전환
// Spring 코드 변경 없이 Failover 처리
// application.yml:
// spring.datasource.url: jdbc:mysql://proxysql-host:6033/mydb
// ProxySQL이 쿼리를 분석해 적절한 백엔드 DB로 라우팅

// 방법 2: 헬스체크 기반 DataSource 교체
@Component
public class ReplicaHealthChecker {

    @Autowired
    private RoutingDataSource routingDataSource;

    @Scheduled(fixedRate = 5000)  // 5초마다
    public void checkReplicaHealth() {
        // Replica Lag 확인
        try (Connection conn = replicaDataSource.getConnection()) {
            Long lagSeconds = queryLag(conn);
            if (lagSeconds > 30) {
                log.warn("Replica lag {}s > threshold. Forcing reads to Source.", lagSeconds);
                // 임시로 Replica 라우팅 비활성화 → Source로 폴백
                DataSourceContextHolder.forceSource(true);
            } else {
                DataSourceContextHolder.forceSource(false);
            }
        } catch (Exception e) {
            // Replica 연결 불가 → Source로 폴백
            log.error("Replica unreachable, falling back to Source", e);
            DataSourceContextHolder.forceSource(true);
        }
    }
}

// 방법 3: Spring Cloud + 동적 설정
// Consul/etcd에 DB 주소 등록
// Spring Cloud Config로 변경 감지 → /actuator/refresh
// DataSource Bean 재생성
```

---

## 💻 실전 실험

### 실험 1: DataSource 라우팅 단위 테스트

```java
@SpringBootTest
class DataSourceRoutingTest {

    @Autowired
    private DataSource dataSource;

    @Test
    void readOnly_transaction_should_use_replica() throws SQLException {
        // readOnly 트랜잭션에서 실제 접속 URL 확인
        DataSourceContextHolder.set(DataSourceType.REPLICA);

        try (Connection conn = dataSource.getConnection()) {
            String url = conn.getMetaData().getURL();
            assertThat(url).contains("replica-host");
        } finally {
            DataSourceContextHolder.clear();
        }
    }

    @Test
    void write_transaction_should_use_source() throws SQLException {
        DataSourceContextHolder.set(DataSourceType.SOURCE);

        try (Connection conn = dataSource.getConnection()) {
            String url = conn.getMetaData().getURL();
            assertThat(url).contains("source-host");
        } finally {
            DataSourceContextHolder.clear();
        }
    }
}
```

### 실험 2: Stale Read 발생 확인

```java
@Test
void write_then_read_in_same_transaction_should_not_stale_read() {
    // 쓰기 + 같은 트랜잭션 읽기 → Source에서 읽어야 함
    OrderResponse response = orderService.processOrder(testRequest);

    // 응답에 방금 저장한 주문이 포함돼야 함
    assertThat(response.getOrderId()).isNotNull();
    assertThat(response.getStatus()).isEqualTo("PENDING");
    // Stale Read였다면 주문이 없어 예외 발생
}

@Test
void independent_read_may_have_stale_data() {
    // 독립 읽기 → Replica → Lag 있으면 과거 데이터 가능
    // 이 테스트는 Lag이 없는 테스트 환경에서만 정확
    List<Order> orders = orderService.getRecentOrders(userId);
    // 테스트 환경에서는 Lag 없음 → 최신 데이터 반환
}
```

### 실험 3: Lag 임계치 폴백 테스트

```java
@Test
void should_fallback_to_source_when_replica_lag_exceeds_threshold() {
    // Replica Lag을 시뮬레이션
    // (테스트 환경에서 Replica를 인위적으로 지연)

    // 헬스체커가 Lag 감지 → Source 강제 설정
    replicaHealthChecker.simulateLag(60);  // 60초 Lag

    // 읽기 요청이 Source로 라우팅되는지 확인
    DataSourceType currentType = DataSourceContextHolder.get();
    // Source로 폴백 확인
}
```

---

## 📊 성능/비용 비교

```
Read/Write 분리 효과:

Source 단독 (분리 전):
  모든 읽기/쓰기 → Source
  읽기:쓰기 = 8:2 비율이면 Source 부하 = 10
  Replica: 복제만 (유휴)
  Source 병목 → 전체 성능 저하

Source + Replica 분리:
  쓰기(2) → Source
  읽기(8) → Replica
  Source 부하: 80% 감소
  → 쓰기 처리량 증가
  → 읽기 응답 시간 개선 (Source 경합 감소)

Replica 수평 확장:
  Replica 3대: 각 Replica 읽기 부하 = 8/3 ≈ 2.7
  Source 부하 = 2 (쓰기만)
  → 읽기 부하 이론상 무제한 수평 확장

Stale Read 방어 비용:
  중요 읽기를 Source로 → Source 부하 약간 증가
  전체 읽기의 10~20%를 Source로 처리해도 분리 효과 유지
```

---

## ⚖️ 트레이드오프

```
Read/Write 분리 도입 트레이드오프:

이점:
  ✅ 읽기 부하 수평 확장 (Replica 추가)
  ✅ Source 쓰기 성능 보호
  ✅ Replica 장애 시 Source로 폴백 (서비스 유지)
  ✅ 읽기 지연 감소 (Source 경합 없음)

비용:
  ❌ Replica Lag → Stale Read (데이터 불일치 가능)
  ❌ 코드 복잡도 증가 (라우팅 로직, 전파 설정)
  ❌ Failover 시 DataSource 재설정 필요
  ❌ 트랜잭션 전파 설정 실수 → 의도치 않은 Stale Read

Stale Read 허용 범위:
  허용 (Replica 사용):
    게시판 목록, 검색, 통계 집계
    수 초 이내 지연이 사용자 경험에 무방한 경우

  불허 (Source 사용):
    결제 전 재고 확인, 잔액 확인
    중복 방지 로직 (email 중복 체크 등)
    쓰기 직후 같은 트랜잭션의 읽기

ProxySQL 활용 (권장):
  DB 레벨에서 쿼리 분석 → 자동 라우팅
  SELECT → Replica, INSERT/UPDATE/DELETE → Source
  Spring 코드 없이 구현 → 단순, 안정적
  Failover 자동 처리
  단, 추가 인프라 관리 비용

순수 Spring 구현 vs ProxySQL:
  소규모: Spring 구현 (인프라 추가 없음)
  중대규모: ProxySQL (안정성, 자동화)
```

---

## 📌 핵심 정리

```
Spring Read/Write 분리 핵심:

필수 컴포넌트:
  AbstractRoutingDataSource: 키 기반 DataSource 선택
  LazyConnectionDataSourceProxy: readOnly 플래그 인식 (필수!)
  @Order(HIGHEST_PRECEDENCE): 라우팅 AOP가 트랜잭션 AOP보다 먼저

Stale Read 방어:
  쓰기(@Transactional) 내에서 호출되는 읽기:
    propagation = REQUIRED → 부모 트랜잭션(Source) 커넥션 재사용
    → Source에서 읽기 보장

  독립 읽기:
    @Transactional(readOnly=true) → 새 트랜잭션 → Replica 선택

여러 Replica 분산:
  Round Robin, 가중치 기반으로 키 동적 결정
  헬스체크로 Lag 임계치 초과 시 Source 폴백

Failover 대응:
  ProxySQL: DB 레벨 자동 전환 (권장)
  @Scheduled 헬스체크: Replica 장애 시 Source 강제 라우팅
  Lag 모니터링: 30초 임계치 초과 시 Source 폴백
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 Stale Read가 발생하는 지점을 찾고, 두 가지 방법으로 수정하라.

```java
@Service
public class OrderService {

    @Transactional  // Source
    public Order createAndReturn(OrderRequest req) {
        Order order = orderRepository.save(new Order(req));
        // 방금 저장한 주문의 상세 조회
        return orderDetailService.getDetail(order.getId());
    }
}

@Service
public class OrderDetailService {

    @Transactional(readOnly = true)  // Replica
    public Order getDetail(Long orderId) {
        return orderRepository.findWithItemsById(orderId)
            .orElseThrow();
    }
}
```

<details>
<summary>해설 보기</summary>

**Stale Read 발생 이유**:
`createAndReturn()`은 Source 트랜잭션에서 `save()` 후, `getDetail()`을 호출합니다. `getDetail()`은 `@Transactional(readOnly=true)`이므로 라우팅 AOP가 Replica DataSource를 선택합니다. Relay Log에 아직 적용되지 않은 상태라면 방금 저장한 주문이 없어 `orElseThrow()`에서 예외 발생.

**수정 방법 1 — Propagation.REQUIRED 유지 (기본)**:
```java
@Service
public class OrderDetailService {

    // propagation = REQUIRED (기본): 부모 트랜잭션(Source)에 참여
    // readOnly=true여도 부모가 Source면 Source 커넥션 재사용
    @Transactional(readOnly = true, propagation = Propagation.REQUIRED)
    public Order getDetail(Long orderId) {
        return orderRepository.findWithItemsById(orderId)
            .orElseThrow();
    }
}
```

**수정 방법 2 — 객체를 직접 반환 (DB 재조회 없음)**:
```java
@Service
public class OrderService {

    @Transactional  // Source
    public Order createAndReturn(OrderRequest req) {
        Order order = orderRepository.save(new Order(req));
        // 이미 메모리에 있는 order 반환 → DB 재조회 불필요
        // 연관 엔티티가 필요하면 save() 전에 join fetch
        return order;
    }
}
```

방법 2가 가장 깔끔합니다. save() 후 반환된 엔티티는 이미 영속 상태이므로 DB 재조회가 불필요합니다. `JOIN FETCH`가 필요하다면 save() 직후 `entityManager.refresh(order)` 또는 명시적 재조회 + Source 보장(방법 1)을 조합합니다.

</details>

---

**Q2.** 결제 전 재고 확인 로직에서 Replica를 사용하면 안 되는 이유를 설명하고, Source에서만 읽도록 강제하는 방법을 제시하라.

<details>
<summary>해설 보기</summary>

**Replica를 사용하면 안 되는 이유**:

Replica Lag이 1초만 있어도:
1. 사용자 A: 재고 10개 확인 (Replica: 10개) → 주문
2. 사용자 B: 재고 10개 확인 (Replica: 10개, Lag으로 아직 A의 주문 미반영) → 주문
3. 두 주문 모두 재고 부족 상태에서 결제 완료
→ 재고 오버셀 발생 → 심각한 비즈니스 문제

**Source에서만 읽도록 강제하는 방법**:

```java
// 방법 1: 커스텀 어노테이션으로 Source 강제
@Target(ElementType.METHOD)
@Retention(RetentionPolicy.RUNTIME)
public @interface ReadFromSource {}

@Aspect
@Order(Ordered.HIGHEST_PRECEDENCE)
public class SourceOnlyAspect {

    @Around("@annotation(ReadFromSource)")
    public Object forceSource(ProceedingJoinPoint pjp) throws Throwable {
        DataSourceContextHolder.set(DataSourceType.SOURCE);
        try {
            return pjp.proceed();
        } finally {
            DataSourceContextHolder.clear();
        }
    }
}

// 사용:
@Service
public class StockService {

    @Transactional
    @ReadFromSource  // Source DataSource 강제
    public void validateAndDeductStock(Long productId, int qty) {
        // Source에서 현재 재고 조회 (SELECT FOR UPDATE)
        Stock stock = stockRepository.findByIdWithLock(productId);
        if (stock.getQuantity() < qty) {
            throw new InsufficientStockException();
        }
        stock.deduct(qty);
        stockRepository.save(stock);
    }
}

// 방법 2: SELECT FOR UPDATE (비관적 락)
// Source 트랜잭션에서 FOR UPDATE로 재고 Row 락
// → 동시 접근 방지 + Source 보장
@Lock(LockModeType.PESSIMISTIC_WRITE)
@Query("SELECT s FROM Stock s WHERE s.productId = :id")
Optional<Stock> findByProductIdWithLock(@Param("id") Long productId);
```

결제, 재고, 잔액 등 금전적 손실이 발생할 수 있는 로직은 **항상 Source에서 읽고, 가능하면 SELECT FOR UPDATE와 함께 사용**합니다.

</details>

---

<div align="center">

**[⬅️ 병렬 복제](./06-parallel-replication.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 4 백업과 복구 ➡️](../backup-recovery/01-mysqldump-internals.md)**

</div>
