---
tags: [CS, 데이터베이스, SQL, NoSQL, SQLite, Firebase, Firestore]
---

# SQL vs NoSQL

## 💡 핵심 개념

SQL(관계형) 데이터베이스는 정해진 스키마와 테이블 구조로 데이터를 저장하며, NoSQL(비관계형)은 유연한 스키마로 문서, 키-값, 그래프 등 다양한 형태로 데이터를 저장한다. Android에서는 로컬 SQL로 SQLite/Room을, NoSQL로는 Firebase Realtime Database나 Firestore를 주로 사용한다.

## 📌 왜 필요한가?

Android 앱 설계 시 로컬 저장(Room)과 서버 동기화(Firestore) 중 어떤 전략을 쓸지, 혹은 둘을 어떻게 조합할지 결정해야 한다. 각 방식의 장단점을 알아야 올바른 아키텍처를 설계할 수 있다.

## 🔍 자세히

### 핵심 비교

| 항목 | SQL (Room/SQLite) | NoSQL (Firestore) |
|------|--------------------|--------------------|
| 스키마 | 고정 스키마 (Entity) | 유연한 스키마 (Document) |
| 쿼리 | 복잡한 JOIN 가능 | 단순 쿼리, JOIN 불가 |
| 확장성 | 수직 확장 | 수평 확장 |
| 정합성 | 강한 일관성 (ACID) | 최종 일관성 (Eventually Consistent) |
| 위치 | 로컬 (디바이스) | 클라우드 (서버) |
| 실시간 | 수동 구현 | 내장 실시간 동기화 |

### SQL 방식 - Room (SQLite)

```kotlin
// 1. 엄격한 스키마 정의
@Entity(tableName = "product")
data class Product(
    @PrimaryKey val id: String,
    val name: String,
    val price: Double,
    val category: String,
    val stock: Int
)

// 2. 타입 안전한 쿼리
@Dao
interface ProductDao {
    @Query("""
        SELECT p.*, c.name as categoryName
        FROM product p
        INNER JOIN category c ON p.category = c.id
        WHERE p.price BETWEEN :min AND :max
        ORDER BY p.price ASC
    """)
    fun getProductsInRange(min: Double, max: Double): Flow<List<ProductWithCategory>>

    @Query("SELECT category, AVG(price) as avgPrice FROM product GROUP BY category")
    suspend fun getCategoryAvgPrice(): List<CategoryPrice>
}

// 3. 컴파일 타임 검증
// → 잘못된 SQL은 빌드 시점에 에러 발생
```

### NoSQL 방식 - Firestore

```kotlin
// 1. 유연한 문서 구조
data class Product(
    val id: String = "",
    val name: String = "",
    val price: Double = 0.0,
    val category: String = "",
    val stock: Int = 0,
    val tags: List<String> = emptyList(),      // 배열 가능
    val metadata: Map<String, Any> = emptyMap() // 중첩 객체 가능
)

// 2. 실시간 데이터 수신
class ProductRepository(
    private val firestore: FirebaseFirestore
) {
    fun getProducts(): Flow<List<Product>> = callbackFlow {
        val listener = firestore.collection("products")
            .whereGreaterThan("price", 1000)
            .orderBy("price")
            .addSnapshotListener { snapshot, error ->
                if (error != null) { close(error); return@addSnapshotListener }
                val products = snapshot?.toObjects(Product::class.java) ?: emptyList()
                trySend(products)
            }
        awaitClose { listener.remove() }
    }

    // 3. 문서 쓰기 (스키마 변경 불필요)
    suspend fun addProduct(product: Product) {
        firestore.collection("products")
            .document(product.id)
            .set(product)
            .await()
    }
}
```

### 데이터 모델링 차이

```kotlin
// SQL: 정규화 - 데이터를 테이블로 분리
// user 테이블 + post 테이블 + comment 테이블
// JOIN으로 결합

@Transaction
@Query("SELECT * FROM user WHERE id = :userId")
suspend fun getUserWithPostsAndComments(userId: Long): UserWithPostsAndComments

// NoSQL: 비정규화 - 필요한 데이터를 함께 저장
// users/{userId} 문서 안에 최근 게시글 포함

/*
users/user123 {
    name: "김철수",
    recentPosts: [
        { title: "첫 글", preview: "..." },
        { title: "두 번째", preview: "..." }
    ]
}

posts/post456 {
    title: "첫 글",
    content: "전체 내용...",
    authorName: "김철수",    // 중복 저장 (비정규화)
    authorId: "user123",
    comments: [...]          // 서브컬렉션 또는 내장
}
*/
```

### 하이브리드 전략 (실무 패턴)

```kotlin
// 서버: Firestore (실시간 동기화 + 멀티 디바이스)
// 로컬: Room (오프라인 캐시 + 복잡한 쿼리)

class ProductRepository(
    private val productDao: ProductDao,       // Room (로컬)
    private val firestore: FirebaseFirestore  // Firestore (서버)
) {
    // UI에서 사용: 로컬 DB를 Single Source of Truth로
    fun getProducts(): Flow<List<Product>> = productDao.getAll()

    // 백그라운드 동기화
    suspend fun sync() {
        val serverData = firestore.collection("products")
            .get().await()
            .toObjects(Product::class.java)

        // 서버 데이터를 로컬에 저장
        productDao.replaceAll(serverData)
    }

    // 쓰기: 서버에 먼저 쓰고, 로컬에도 반영
    suspend fun addProduct(product: Product) {
        firestore.collection("products")
            .document(product.id).set(product).await()
        productDao.insert(product.toEntity())
    }
}
```

### 선택 기준

```
Room (SQL)을 선택할 때:
├── 복잡한 관계형 데이터 (JOIN이 필요)
├── 오프라인 우선 앱
├── 집계/통계 쿼리가 많을 때
├── 강한 데이터 일관성이 필요할 때
└── 단일 디바이스 데이터

Firestore (NoSQL)를 선택할 때:
├── 실시간 동기화가 필요할 때
├── 멀티 디바이스/사용자 공유 데이터
├── 스키마가 자주 변경될 때
├── 서버 인프라 없이 빠르게 구축할 때
└── 수평 확장이 필요한 대규모 서비스

둘 다 사용할 때 (가장 일반적):
├── Firestore로 서버 동기화
├── Room으로 로컬 캐시 및 오프라인 지원
└── Room을 Single Source of Truth로 사용
```

### Android의 다른 저장소 옵션

| 저장소 | 타입 | 용도 |
|--------|------|------|
| Room | SQL | 구조화된 대량 데이터 |
| DataStore | Key-Value | 설정, 간단한 상태 |
| Firebase RTDB | NoSQL (JSON) | 실시간 소규모 데이터 |
| Firestore | NoSQL (Document) | 범용 서버 DB |
| Realm | NoSQL (Object) | 객체 지향 로컬 DB |

## 🔗 관련 개념

- [[00-CS-기초/데이터베이스/SQL-기초|SQL 기초]]
- [[00-CS-기초/데이터베이스/정규화|정규화]]
- [[00-CS-기초/데이터베이스/ORM|ORM]]
- [[00-CS-기초/데이터베이스/트랜잭션-ACID|트랜잭션 ACID]]

## 📚 더 보기

- [Firebase Firestore 문서](https://firebase.google.com/docs/firestore)
- [Room 공식 가이드](https://developer.android.com/training/data-storage/room)
- [오프라인 우선 앱 설계](https://developer.android.com/topic/architecture/data-layer/offline-first)

---

**핵심 요약:** SQL(Room)은 복잡한 쿼리와 오프라인 로컬 저장에, NoSQL(Firestore)은 실시간 동기화와 유연한 스키마에 강하며, 실무에서는 둘을 조합해 사용하는 것이 일반적이다.
