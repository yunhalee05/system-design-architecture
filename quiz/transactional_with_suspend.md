# @Transactional 과 suspend를 함께 사용할 수 있을까?

## 답
- 현재 기준으로는 Spring에서 @Transactional과 suspend 함수의 공식적인 완벽한 통합은 제공하지 않는다.

## @Transactional 동작 원리

### 1. 프록시 기반 AOP
Spring은 @Transactional 메서드를 프록시로 감싸서 트랜잭션을 관리합니다.

```kotlin
// 원본 코드
@Transactional
fun createMenu(menu: Menu): Menu {
    return menuRepository.save(menu)
}

// 실제 생성되는 프록시 코드 (개념적)
class MenuServiceProxy(private val target: MenuService) {
    fun createMenu(menu: Menu): Menu {
        val txManager = getTransactionManager()
        val status = txManager.getTransaction(definition)

        try {
            val result = target.createMenu(menu)  // 실제 메서드 호출
            txManager.commit(status)
            return result
        } catch (e: Exception) {
            txManager.rollback(status)
            throw e
        }
    }
}
```

### 2. 프록시 생성 과정
```
1. 스프링 애플리케이션 시작
2. @Transactional이 있는 빈 스캔
3. CGLIB/JDK Dynamic Proxy로 프록시 클래스 생성
4. 빈 컨테이너에 프록시 객체 등록 (원본 대신)
```

### 3. 트랜잭션 실행 흐름
```
클라이언트 호출
   ↓
프록시 객체 (TransactionInterceptor)
   ↓
TransactionManager.getTransaction() → 트랜잭션 시작
   ↓
실제 메서드 실행
   ↓
성공 → commit() / 실패 → rollback()
   ↓
결과 반환
```

### 4. ThreadLocal과 트랜잭션 컨텍스트
```kotlin
// Spring은 트랜잭션 정보를 ThreadLocal에 저장
class TransactionSynchronizationManager {
    private val resources: ThreadLocal<Map<Object, Object>> = ThreadLocal()
    private val synchronizations: ThreadLocal<Set<TransactionSynchronization>> = ThreadLocal()

    // 같은 스레드 내에서 트랜잭션 공유
    fun getCurrentTransactionName(): String? {
        return synchronizations.get()?.name
    }
}

// 예시
@Transactional
fun outerMethod() {
    // 트랜잭션 시작 → ThreadLocal에 저장
    innerMethod()  // 같은 스레드, 같은 트랜잭션
}

fun innerMethod() {
    // ThreadLocal에서 현재 트랜잭션 조회
    // 이미 존재하면 같은 트랜잭션 사용
}
```

### 5. 왜 suspend와 호환이 안 될까?
```kotlin
suspend fun suspendMethod() {
    // 코루틴이 중단되면...
    delay(100)  // ← 여기서 스레드 변경 가능!
    // 다른 스레드에서 재개될 수 있음
}

// 문제 상황
@Transactional
suspend fun createMenu() {  // 스레드 A에서 트랜잭션 시작 (ThreadLocal에 저장)
    delay(100)              // 코루틴 중단
    menuRepository.save()   // 스레드 B에서 재개 → ThreadLocal 못 찾음!
}
```

**핵심 문제:**
- @Transactional: ThreadLocal 기반 (스레드에 종속)
- Coroutine: 스레드를 넘나들며 실행
- → 트랜잭션 컨텍스트 손실 가능 


## 현재 가능한 방법들
### 1. TransactionalOperator (Reactive)

#### TransactionalOperator란?
- Spring WebFlux에서 사용하는 **프로그래밍 방식의 트랜잭션 관리 도구**
- Reactive Stream (Mono, Flux)과 함께 사용
- @Transactional 대신 명시적으로 트랜잭션 경계를 제어

#### 생성 및 설정
```kotlin
@Configuration
class TransactionConfig {

    @Bean
    fun transactionalOperator(
        transactionManager: ReactiveTransactionManager
    ): TransactionalOperator {
        return TransactionalOperator.create(transactionManager)
    }

    @Bean
    fun transactionManager(
        connectionFactory: ConnectionFactory
    ): ReactiveTransactionManager {
        return R2dbcTransactionManager(connectionFactory)
    }
}
```

#### 기본 사용법
```kotlin
@Service
class MenuService(
    private val menuRepository: MenuRepository,
    private val transactionalOperator: TransactionalOperator
) {

    fun createMenu(menu: Menu): Mono<Menu> {
        return menuRepository.save(menu)
            .as(transactionalOperator::transactional)
            // transactionalOperator가 트랜잭션 시작/커밋/롤백 처리
    }

    // 여러 작업을 하나의 트랜잭션으로
    fun createMenuWithCategory(menu: Menu, category: Category): Mono<Menu> {
        return categoryRepository.save(category)
            .flatMap { savedCategory ->
                menu.categoryId = savedCategory.id
                menuRepository.save(menu)
            }
            .as(transactionalOperator::transactional)
            // 모든 작업이 하나의 트랜잭션에서 실행됨
    }
}
```

#### 동작 원리
```kotlin
// TransactionalOperator 내부 동작 (개념적)
class TransactionalOperator(
    private val transactionManager: ReactiveTransactionManager
) {
    fun <T> transactional(flux: Flux<T>): Flux<T> {
        return Flux.usingWhen(
            // 1. 트랜잭션 시작
            transactionManager.getReactiveTransaction(definition),

            // 2. 비즈니스 로직 실행
            { status -> flux },

            // 3. 성공 시 커밋
            { status -> transactionManager.commit(status) },

            // 4. 에러 시 롤백
            { status, error -> transactionManager.rollback(status) },

            // 5. 취소 시 롤백
            { status -> transactionManager.rollback(status) }
        )
    }
}
```

#### 실행 흐름
```
Mono/Flux 체인 구성
   ↓
.as(transactionalOperator::transactional) 호출
   ↓
ReactiveTransactionManager.getReactiveTransaction() → 트랜잭션 시작
   ↓
Reactive Stream 실행 (non-blocking)
   ↓
성공: commit() / 에러: rollback()
   ↓
결과 반환 (Mono/Flux)
```

#### Reactive Context로 트랜잭션을 보장하는 방법

**핵심 개념: Context는 Reactive Stream을 따라 전파된다**

```kotlin
// Reactor의 Context (Project Reactor 제공)
interface Context {
    // Key-Value 저장소 (Immutable)
    fun <T> get(key: Any): T?
    fun put(key: Any, value: Any): Context
    fun hasKey(key: Any): Boolean
}

// Context는 Mono/Flux 체인을 따라 downstream으로 전파
Mono.just("data")
    .contextWrite { context ->
        context.put("txId", "tx-123")  // Context에 트랜잭션 정보 저장
    }
    .flatMap { data ->
        // 여기서 Context 조회 가능
        Mono.deferContextual { ctx ->
            val txId = ctx.get<String>("txId")  // "tx-123"
            Mono.just("$data-$txId")
        }
    }
```

#### TransactionalOperator의 실제 구현

```kotlin
// TransactionalOperator가 Context에 트랜잭션을 저장하는 방식
class TransactionalOperator(
    private val transactionManager: ReactiveTransactionManager
) {

    companion object {
        // Context에서 트랜잭션 정보를 찾는 Key
        private val TRANSACTION_CONTEXT_KEY = TransactionContext::class.java
    }

    fun <T> transactional(publisher: Publisher<T>): Flux<T> {
        return Flux.usingWhen(
            // 1. 트랜잭션 시작 → TransactionStatus 생성
            transactionManager.getReactiveTransaction(definition),

            // 2. TransactionStatus를 Context에 저장하고 비즈니스 로직 실행
            { status ->
                Flux.from(publisher)
                    .contextWrite { context ->
                        // ⭐ Context에 트랜잭션 정보 저장
                        context.put(
                            TRANSACTION_CONTEXT_KEY,
                            TransactionContext(status)
                        )
                    }
            },

            // 3. 성공 시 커밋
            { status -> transactionManager.commit(status) },

            // 4. 에러 시 롤백
            { status, error -> transactionManager.rollback(status) },

            // 5. 취소 시 롤백
            { status -> transactionManager.rollback(status) }
        )
    }
}
```

#### R2DBC에서 트랜잭션 실행 과정

```kotlin
// 실제 R2DBC가 Context에서 트랜잭션을 가져오는 방식
class R2dbcTransactionManager(
    private val connectionFactory: ConnectionFactory
) : ReactiveTransactionManager {

    override fun getReactiveTransaction(
        definition: TransactionDefinition
    ): Mono<ReactiveTransaction> {
        return Mono.defer {
            // 1. DB Connection 생성
            Mono.from(connectionFactory.create())
                .flatMap { connection ->
                    // 2. BEGIN TRANSACTION 실행
                    Mono.from(connection.beginTransaction())
                        .thenReturn(
                            R2dbcTransactionObject(connection, definition)
                        )
                }
        }
    }

    override fun commit(transaction: ReactiveTransaction): Mono<Void> {
        return Mono.defer {
            val txObject = transaction as R2dbcTransactionObject
            // COMMIT 실행
            Mono.from(txObject.connection.commitTransaction())
        }
    }

    override fun rollback(transaction: ReactiveTransaction): Mono<Void> {
        return Mono.defer {
            val txObject = transaction as R2dbcTransactionObject
            // ROLLBACK 실행
            Mono.from(txObject.connection.rollbackTransaction())
        }
    }
}

// Repository가 Context에서 Connection을 가져오는 방식
class R2dbcRepository {

    fun save(entity: Menu): Mono<Menu> {
        return Mono.deferContextual { ctx ->
            // ⭐ Context에서 현재 트랜잭션의 Connection 조회
            val txContext = ctx.getOrDefault(
                TRANSACTION_CONTEXT_KEY,
                null
            ) as? TransactionContext

            val connection = txContext?.connection
                ?: throw IllegalStateException("No transaction")

            // 해당 Connection으로 SQL 실행
            Mono.from(
                connection.createStatement(
                    "INSERT INTO menu (name, price) VALUES ($1, $2)"
                )
                .bind(0, entity.name)
                .bind(1, entity.price)
                .execute()
            )
        }
    }
}
```

#### 실제 실행 예시로 보는 보장 메커니즘

```kotlin
@Service
class MenuService(
    private val menuRepository: R2dbcMenuRepository,
    private val categoryRepository: R2dbcCategoryRepository,
    private val transactionalOperator: TransactionalOperator
) {

    fun createMenuWithCategory(menu: Menu, category: Category): Mono<Menu> {
        return categoryRepository.save(category)      // ← 작업 1
            .flatMap { savedCategory ->
                menu.categoryId = savedCategory.id
                menuRepository.save(menu)             // ← 작업 2
            }
            .as(transactionalOperator::transactional) // ← 트랜잭션 경계
    }
}

// 실제 실행 흐름 (시간순)
//
// 1. transactionalOperator::transactional 호출
//    → BEGIN TRANSACTION (Connection#1 생성)
//    → Context에 Connection#1 저장
//
// 2. categoryRepository.save() 호출
//    → deferContextual로 Context 조회
//    → Connection#1 찾음
//    → INSERT INTO category ... (Connection#1 사용)
//
// 3. flatMap 실행
//    → Context는 자동으로 전파됨 (Reactor가 보장)
//
// 4. menuRepository.save() 호출
//    → deferContextual로 Context 조회
//    → Connection#1 찾음 (같은 Connection!)
//    → INSERT INTO menu ... (Connection#1 사용)
//
// 5. 모든 작업 성공
//    → COMMIT (Connection#1)
//    → Connection#1 반환
//
// ⭐ 핵심: Context가 Mono/Flux 체인을 따라 전파되므로
//         모든 작업이 같은 Connection을 사용함 = 같은 트랜잭션
```

#### ThreadLocal vs Reactive Context 비교

```kotlin
// ThreadLocal 방식 (@Transactional)
class ThreadLocalTransactionManager {

    companion object {
        // 스레드마다 별도의 트랜잭션 정보 저장
        private val holder = ThreadLocal<TransactionInfo>()
    }

    fun begin() {
        val connection = dataSource.connection
        connection.autoCommit = false
        holder.set(TransactionInfo(connection))  // 현재 스레드에 저장
    }

    fun getConnection(): Connection {
        return holder.get()?.connection  // 현재 스레드에서 조회
            ?: throw IllegalStateException("No transaction")
    }

    // ❌ 문제: 코루틴이 스레드를 바꾸면 조회 실패
    suspend fun example() {
        begin()  // Thread-1에 저장
        delay(100)  // 코루틴 중단
        getConnection()  // Thread-2에서 조회 → null!
    }
}

// Reactive Context 방식 (TransactionalOperator)
class ReactiveContextTransactionManager {

    fun begin(): Mono<Connection> {
        return Mono.from(connectionFactory.create())
            .flatMap { connection ->
                connection.beginTransaction()
                    .thenReturn(connection)
            }
    }

    fun <T> executeInTransaction(
        operation: (Connection) -> Mono<T>
    ): Mono<T> {
        return begin()
            .flatMap { connection ->
                operation(connection)
                    // ⭐ Context에 저장 (스레드 독립적)
                    .contextWrite { ctx ->
                        ctx.put("connection", connection)
                    }
            }
    }

    // ✅ 해결: Context는 Mono/Flux와 함께 이동
    fun example(): Mono<String> {
        return executeInTransaction { connection ->
            Mono.delay(Duration.ofMillis(100))  // 스레드 바뀔 수 있음
                .flatMap {
                    // Context는 여전히 유효!
                    Mono.deferContextual { ctx ->
                        val conn = ctx.get<Connection>("connection")
                        Mono.just("Success with $conn")
                    }
                }
        }
    }
}
```

#### 보장 메커니즘 정리

| 항목 | ThreadLocal | Reactive Context |
|-----|-------------|------------------|
| 저장 위치 | 스레드별 메모리 | Mono/Flux 체인에 첨부 |
| 전파 방식 | 같은 스레드 내에서만 | Reactive 연산자를 따라 자동 전파 |
| 스레드 전환 | ❌ 정보 손실 | ✅ 정보 유지 |
| 비동기 환경 | ❌ 부적합 | ✅ 적합 |
| 동시성 제어 | Synchronized 필요 | Immutable Context (복사본 생성) |

**핵심 차이:**
- ThreadLocal: "현재 스레드"에 정보를 저장 → 스레드 의존적
- Reactive Context: "현재 Reactive Stream"에 정보를 저장 → 스레드 독립적

**보장 원리:**
1. Context는 Immutable (불변)
2. 새로운 값을 추가하면 새로운 Context 복사본 생성
3. Reactor가 모든 연산자에서 Context 전파를 보장
4. 따라서 스레드가 바뀌어도 Context는 유지됨

#### Coroutine과 함께 사용 (R2DBC)
```kotlin
@Service
class MenuService(
    private val menuRepository: CoroutineR2dbcRepository,
    private val transactionalOperator: TransactionalOperator
) {

    // Coroutine → Reactive 변환 후 트랜잭션 적용
    suspend fun createMenu(menu: Menu): Menu {
        return mono {
            menuRepository.save(menu)  // suspend 함수
        }
            .as(transactionalOperator::transactional)
            .awaitSingle()
    }

    // 또는 직접 변환
    suspend fun createMenuWithItems(
        menu: Menu,
        items: List<MenuItem>
    ): Menu = coroutineScope {
        mono {
            val savedMenu = menuRepository.save(menu)
            items.forEach { item ->
                item.menuId = savedMenu.id
                menuItemRepository.save(item)
            }
            savedMenu
        }
            .as(transactionalOperator::transactional)
            .awaitSingle()
    }
}
```

#### 한계점
```kotlin
// ❌ JPA는 사용 불가 (blocking)
@Entity
class Menu // JPA Entity

fun createMenu(menu: Menu): Mono<Menu> {
    return Mono.fromCallable {
        jpaRepository.save(menu)  // blocking call
    }
        .as(transactionalOperator::transactional)
    // → 동작은 하지만 blocking이라 의미 없음
}

// ✅ R2DBC만 제대로 동작 (non-blocking)
@Table("menu")
data class Menu(...)  // R2DBC Entity

fun createMenu(menu: Menu): Mono<Menu> {
    return r2dbcRepository.save(menu)  // non-blocking
        .as(transactionalOperator::transactional)
}
```

#### @Transactional vs TransactionalOperator 비교
```kotlin
// @Transactional: 선언적, 간편함, blocking
@Transactional
fun createMenu(menu: Menu): Menu {
    return menuRepository.save(menu)
}

// TransactionalOperator: 명시적, 복잡함, reactive
fun createMenu(menu: Menu): Mono<Menu> {
    return menuRepository.save(menu)
        .as(transactionalOperator::transactional)
}
```

| 특성 | @Transactional | TransactionalOperator |
|-----|---------------|----------------------|
| 방식 | 선언적 (AOP) | 프로그래밍 방식 |
| 스레드 모델 | ThreadLocal 기반 | Reactive Context 기반 |
| 사용 대상 | Blocking (JPA) | Non-blocking (R2DBC) |
| 코루틴 호환 | ❌ (스레드 변경 시 문제) | △ (Mono/Flux 변환 필요) |
| 세밀한 제어 | 어려움 | 쉬움 |

### 2. 가상스레드를 활용하기 (block 방식) 
**단, 가상 스레드 ≠ 비동기**
- 가상 스레드는 **경량 스레드**일 뿐
- 여전히 **blocking I/O 모델**
- 단지 **blocking해도 메모리 효율**이 좋을 뿐

**가상 스레드의 한계:**
- 여전히 blocking (스레드가 대기함)
- Non-blocking I/O 라이브러리 못 씀
- CPU 바운드 작업엔 도움 안 됨

**가상 스레드 = 더 많이 블로킹할 수 있는 스레드** 일뿐, 비동기는 아님 
#### 일반 스레드 (Platform Thread)
```kotlin
Thread.ofPlatform().start {
    // OS 네이티브 스레드// 메모리: ~1MB (스택)// 생성 비용: 비쌈// 최대 개수: ~수천 개
}

@Transactional
fun createMenu(menu: Menu): Menu {
    return jpaRepository.save(menu)  // blocking
    // 스레드 1개가 여기서 멈춰서 대기
    // 메모리: ~1MB per thread
}
```
#### 가상 스레드 (Virtual Thread)
```kotlin
Thread.ofVirtual().start {
    // JVM이 관리하는 경량 스레드// 메모리: ~1KB// 생성 비용: 매우 저렴// 최대 개수: 수백만 개 가능
}
@Transactional
fun createMenu(menu: Menu): Menu {
    return jpaRepository.save(menu)  // blocking
    // 가상 스레드 1개가 여기서 멈춰서 대기
    // 메모리: ~1KB per thread
    // ✅ 수천~수만 개 생성 가능
}
```

### 3. runBlocking으로 감싸기 (block 방식)
```kotlin
@Transactional
fun createMenu(menu: Menu) = runBlocking {
    // suspend 함수 호출
}
```



## 요약
공식적으로 **완벽하게 통합된 해법은 없다.**
- R2DBC + Reactive 방식 사용하거나
- `runBlocking`으로 감싸서 사용하거나
- 트랜잭션 경계를 명시적으로 관리해야 합니다

