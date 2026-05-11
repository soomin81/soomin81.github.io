+++
date = '2026-05-11T22:40:55+09:00'
draft = false
title = 'Spring Data JDBC + jOOQ 함께 쓰기 - Kotlin DB 접근 전략'
categories = ["개발"]
tags = ["kotlin", "spring", "jooq", "spring-data-jdbc", "database", "cqrs", "backend"]
+++

## 들어가며

Kotlin + Spring Boot 환경에서 DB 접근 기술을 고를 때, Spring Data JDBC와 jOOQ는 자주 비교 대상에 오릅니다.  
그런데 사실 이 둘은 **경쟁 관계가 아닙니다.**

각자 잘하는 영역이 명확하게 다르기 때문에, 함께 쓰면 오히려 서로의 단점을 메워줍니다.

- **Spring Data JDBC**: 단순한 저장/수정/삭제, Repository 패턴
- **jOOQ**: 복잡한 조회, 동적 쿼리, 집계/통계

이 포스팅에서는 두 기술을 함께 사용하는 구조와 실전 패턴을 정리해 보겠습니다.

---

## 1. 왜 함께 쓰는가

### JPA의 대안으로

JPA(Hibernate)는 Kotlin과 궁합이 썩 좋지 않습니다.

- `data class`에 `open` 키워드 강제 → `allopen` 플러그인 필요
- 지연 로딩으로 인한 예측 어려운 쿼리
- N+1 문제를 항상 의식해야 하는 피로감
- 영속성 컨텍스트 동작을 이해해야 하는 높은 진입 장벽

Spring Data JDBC와 jOOQ는 이런 문제 없이 Kotlin답게 DB를 다룰 수 있게 해줍니다.

### 역할 분리가 자연스럽다

두 기술의 강점이 딱 맞아떨어집니다.

```
쓰기 (Command)          읽기 (Query)
─────────────────       ─────────────────────────
Spring Data JDBC        jOOQ

- save()                - 복잡한 JOIN 조회
- delete()              - 동적 검색 조건
- PK 기반 단건 조회     - 집계 / 통계
- 단순 조건 조회        - 페이지네이션
```

이 구조는 **CQRS(Command Query Responsibility Segregation)** 패턴과도 자연스럽게 맞아떨어집니다.

---

## 2. 프로젝트 구조

패키지 구조는 다음과 같이 역할을 명확하게 나누는 것을 권장합니다.

```
com.yourcompany.project
├── domain
│   └── order
│       ├── Order.kt                     # 엔티티 (data class)
│       ├── OrderRepository.kt           # Spring Data JDBC Repository
│       └── OrderService.kt             # Command 처리
│
├── query
│   └── order
│       ├── OrderQueryRepository.kt      # jOOQ 조회 전담
│       └── OrderDto.kt                 # 조회 전용 DTO
│
└── generated
    └── jooq                            # jOOQ 자동 생성 코드
        ├── tables
        │   └── Orders.kt
        └── ...
```

---

## 3. 의존성 설정

```kotlin
// build.gradle.kts
dependencies {
    // Spring Data JDBC
    implementation("org.springframework.boot:spring-boot-starter-data-jdbc")

    // jOOQ
    implementation("org.springframework.boot:spring-boot-starter-jooq")

    // jOOQ Kotlin 확장
    implementation("org.jooq:jooq-kotlin")

    // DB 드라이버 (MySQL 기준)
    runtimeOnly("com.mysql.cj:mysql-connector-j")
}

// jOOQ 코드 생성 플러그인
plugins {
    id("nu.studer.jooq") version "9.0"
}

jooq {
    configurations {
        create("main") {
            jooqConfiguration.apply {
                jdbc.apply {
                    driver = "com.mysql.cj.jdbc.Driver"
                    url = "jdbc:mysql://localhost:3306/yourdb"
                    user = "root"
                    password = "password"
                }
                generator.apply {
                    database.apply {
                        name = "org.jooq.meta.mysql.MySQLDatabase"
                        inputSchema = "yourdb"
                    }
                    target.apply {
                        packageName = "com.yourcompany.generated.jooq"
                        directory = "src/generated/jooq"
                    }
                }
            }
        }
    }
}
```

---

## 4. 엔티티와 Repository 정의 (Spring Data JDBC)

Spring Data JDBC는 Kotlin `data class`를 그대로 엔티티로 사용할 수 있습니다.  
`allopen`이나 `@Entity` 같은 JPA 전용 어노테이션이 필요 없습니다.

```kotlin
// Order.kt
@Table("orders")
data class Order(
    @Id val id: Long? = null,
    val userId: Long,
    val status: String,
    val totalAmount: Long,
    val createdAt: LocalDateTime = LocalDateTime.now(),
    val updatedAt: LocalDateTime = LocalDateTime.now()
)
```

```kotlin
// OrderRepository.kt
interface OrderRepository : CrudRepository<Order, Long> {

    fun findByUserId(userId: Long): List<Order>

    fun findByUserIdAndStatus(userId: Long, status: String): List<Order>
}
```

단순 CRUD와 기본 조건 조회는 이것만으로 충분합니다.

---

## 5. 복잡한 조회 처리 (jOOQ)

Spring Data JDBC로 처리하기 까다로운 쿼리는 jOOQ가 맡습니다.

### 기본 구조

```kotlin
// OrderQueryRepository.kt
@Repository
class OrderQueryRepository(
    private val dsl: DSLContext
) {
    // ...
}
```

### 동적 검색 조건

```kotlin
data class OrderSearchRequest(
    val userId: Long? = null,
    val status: String? = null,
    val from: LocalDateTime? = null,
    val to: LocalDateTime? = null,
    val page: Int = 0,
    val size: Int = 20
)

fun search(request: OrderSearchRequest): List<OrderDto> {
    val conditions = buildList {
        request.userId?.let { add(ORDERS.USER_ID.eq(it)) }
        request.status?.let { add(ORDERS.STATUS.eq(it)) }
        request.from?.let { add(ORDERS.CREATED_AT.greaterOrEqual(it)) }
        request.to?.let { add(ORDERS.CREATED_AT.lessOrEqual(it)) }
    }

    return dsl
        .select(
            ORDERS.ID,
            ORDERS.STATUS,
            ORDERS.TOTAL_AMOUNT,
            ORDERS.CREATED_AT,
            USERS.NAME.`as`("userName")
        )
        .from(ORDERS)
        .join(USERS).on(ORDERS.USER_ID.eq(USERS.ID))
        .where(conditions)
        .orderBy(ORDERS.CREATED_AT.desc())
        .limit(request.size)
        .offset(request.page * request.size)
        .fetchInto(OrderDto::class.java)
}
```

### 집계 쿼리

```kotlin
fun getStatusSummaryByUser(userId: Long): List<OrderStatusSummary> {
    return dsl
        .select(
            ORDERS.STATUS,
            DSL.count().`as`("count"),
            DSL.sum(ORDERS.TOTAL_AMOUNT).`as`("totalAmount")
        )
        .from(ORDERS)
        .where(ORDERS.USER_ID.eq(userId))
        .groupBy(ORDERS.STATUS)
        .fetchInto(OrderStatusSummary::class.java)
}
```

### 커서 기반 페이지네이션

```kotlin
fun findNextPage(userId: Long, lastId: Long, size: Int): List<OrderDto> {
    return dsl
        .selectFrom(ORDERS)
        .where(
            ORDERS.USER_ID.eq(userId)
                .and(ORDERS.ID.lt(lastId))
        )
        .orderBy(ORDERS.ID.desc())
        .limit(size)
        .fetchInto(OrderDto::class.java)
}
```

---

## 6. Service에서 함께 사용하기

Command와 Query를 Service에서 자연스럽게 함께 사용합니다.

```kotlin
@Service
@Transactional
class OrderService(
    private val orderRepository: OrderRepository,         // Spring Data JDBC
    private val orderQueryRepository: OrderQueryRepository // jOOQ
) {

    // 쓰기: Spring Data JDBC
    fun createOrder(command: CreateOrderCommand): Order {
        val order = Order(
            userId = command.userId,
            status = "PENDING",
            totalAmount = command.totalAmount
        )
        return orderRepository.save(order)
    }

    fun updateStatus(orderId: Long, status: String) {
        val order = orderRepository.findById(orderId)
            .orElseThrow { IllegalArgumentException("주문을 찾을 수 없습니다.") }
        orderRepository.save(order.copy(status = status))
    }

    // 읽기: jOOQ
    @Transactional(readOnly = true)
    fun searchOrders(request: OrderSearchRequest): List<OrderDto> {
        return orderQueryRepository.search(request)
    }

    @Transactional(readOnly = true)
    fun getStatusSummary(userId: Long): List<OrderStatusSummary> {
        return orderQueryRepository.getStatusSummaryByUser(userId)
    }
}
```

---

## 7. 트랜잭션 처리

두 기술 모두 Spring의 `@Transactional`을 그대로 사용하면 됩니다.  
같은 트랜잭션 안에서 Spring Data JDBC와 jOOQ를 함께 써도 문제없이 동작합니다.

```kotlin
@Transactional
fun placeOrder(command: PlaceOrderCommand): OrderDto {
    // Spring Data JDBC로 저장
    val order = orderRepository.save(
        Order(userId = command.userId, status = "PAID", totalAmount = command.amount)
    )

    // jOOQ로 저장 후 바로 연관 데이터 조회
    return orderQueryRepository.findWithUserInfo(order.id!!)
        ?: throw IllegalStateException("저장된 주문을 조회할 수 없습니다.")
}
```

---

## 8. 테스트 전략

두 기술을 함께 쓰면 테스트 계층도 자연스럽게 나뉩니다.

### Repository 테스트 (Spring Data JDBC)

```kotlin
@DataJdbcTest
class OrderRepositoryTest(
    @Autowired private val orderRepository: OrderRepository
) {
    @Test
    fun `userId로 주문 목록을 조회할 수 있다`() {
        val saved = orderRepository.save(Order(userId = 1L, status = "PAID", totalAmount = 10000))
        val result = orderRepository.findByUserId(1L)

        assertThat(result).hasSize(1)
        assertThat(result.first().id).isEqualTo(saved.id)
    }
}
```

### QueryRepository 테스트 (jOOQ + Testcontainers)

```kotlin
@SpringBootTest
@Testcontainers
class OrderQueryRepositoryTest(
    @Autowired private val orderQueryRepository: OrderQueryRepository
) {
    companion object {
        @Container
        val mysql = MySQLContainer("mysql:8.0")
    }

    @Test
    fun `동적 조건으로 주문을 검색할 수 있다`() {
        val request = OrderSearchRequest(status = "PAID", page = 0, size = 10)
        val result = orderQueryRepository.search(request)

        assertThat(result).allMatch { it.status == "PAID" }
    }
}
```

---

## 마무리

Spring Data JDBC와 jOOQ를 함께 쓰는 전략을 정리하면 다음과 같습니다.

| 역할 | 기술 | 이유 |
|------|------|------|
| 엔티티 저장 / 수정 / 삭제 | Spring Data JDBC | 단순하고 명확한 Repository 패턴 |
| PK 기반 단건 조회 | Spring Data JDBC | 오버엔지니어링 불필요 |
| 복잡한 조회 / 동적 검색 | jOOQ | 타입 안전한 SQL DSL |
| 집계 / 통계 / 리포팅 | jOOQ | 고급 SQL 완전 지원 |

두 기술을 적절히 나눠 사용하면, JPA 없이도 Kotlin답고 유지보수하기 좋은 DB 레이어를 만들 수 있습니다.  
특히 쓰기/읽기 책임이 명확하게 분리되어, 코드를 처음 보는 팀원도 어느 레이어에서 무슨 일이 일어나는지 파악하기 쉬워집니다.
