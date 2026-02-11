---
tags: [CS, 자료구조, Set, Map, HashSet, TreeSet, LinkedHashSet]
---

# Set과 Map

## 💡 핵심 개념

Set은 **중복을 허용하지 않는** 원소의 집합이고, Map은 **고유한 키에 값을 매핑**하는 자료구조이다. 둘 다 내부 구현에 따라 Hash 기반(정렬 없음, O(1)), Tree 기반(정렬 유지, O(log n)), LinkedHash 기반(삽입 순서 유지, O(1)) 변형이 존재한다.

## 📌 왜 필요한가?

Android에서 중복 제거(Set), 키-값 저장(Map), Bundle/Intent extras(Map 구조) 등에 광범위하게 사용된다. Kotlin의 풍부한 컬렉션 API와 결합하면 간결하고 효율적인 코드를 작성할 수 있다.

## 🔍 자세히

### Set 구현체 비교

```kotlin
// HashSet - 순서 없음, O(1) 접근
val hashSet = hashSetOf("C", "A", "B")
println(hashSet) // 순서 보장 안 됨

// LinkedHashSet - 삽입 순서 유지, O(1) 접근
val linkedHashSet = linkedSetOf("C", "A", "B")
println(linkedHashSet) // [C, A, B] - 삽입 순서 보장

// TreeSet - 정렬 유지, O(log n) 접근
val treeSet = sortedSetOf("C", "A", "B")
println(treeSet) // [A, B, C] - 항상 정렬
```

| 구현체 | 순서 | 추가/삭제/검색 | null 허용 |
|--------|------|---------------|-----------|
| HashSet | 없음 | **O(1)** | 가능 |
| LinkedHashSet | 삽입 순서 | **O(1)** | 가능 |
| TreeSet | 정렬 순서 | O(log n) | 불가 |

### Set 연산 (집합 연산)

```kotlin
val setA = setOf(1, 2, 3, 4)
val setB = setOf(3, 4, 5, 6)

// 합집합
val union = setA union setB          // {1, 2, 3, 4, 5, 6}

// 교집합
val intersect = setA intersect setB  // {3, 4}

// 차집합
val subtract = setA subtract setB    // {1, 2}

// 포함 관계
println(3 in setA)                   // true

// 중복 제거 패턴
val names = listOf("Kim", "Lee", "Kim", "Park", "Lee")
val uniqueNames = names.toSet()      // {Kim, Lee, Park}
val uniqueList = names.distinct()    // [Kim, Lee, Park]
```

### Map 구현체 비교

```kotlin
// HashMap - 순서 없음, O(1) 접근
val hashMap = hashMapOf("b" to 2, "a" to 1, "c" to 3)

// LinkedHashMap - 삽입 순서 유지, O(1) 접근
val linkedHashMap = linkedMapOf("b" to 2, "a" to 1, "c" to 3)
println(linkedHashMap.keys) // [b, a, c] - 삽입 순서 보장

// TreeMap - 키 정렬 유지, O(log n) 접근
val treeMap = sortedMapOf("b" to 2, "a" to 1, "c" to 3)
println(treeMap.keys) // [a, b, c] - 항상 정렬
```

| 구현체 | 키 순서 | 추가/삭제/검색 | 특징 |
|--------|--------|---------------|------|
| HashMap | 없음 | **O(1)** | 가장 빠름 |
| LinkedHashMap | 삽입 순서 | **O(1)** | LRU 캐시 구현에 활용 |
| TreeMap | 정렬 순서 | O(log n) | 범위 쿼리 지원 |

### Kotlin Map 고급 활용

```kotlin
val scores = mapOf("Kim" to 90, "Lee" to 85, "Park" to 95)

// 변환
val doubled = scores.mapValues { (_, v) -> v * 2 }
val prefixed = scores.mapKeys { (k, _) -> "학생_$k" }

// 필터링
val passed = scores.filter { (_, score) -> score >= 90 }

// 그룹핑
data class Student(val name: String, val grade: Int)
val students = listOf(
    Student("Kim", 1), Student("Lee", 2),
    Student("Park", 1), Student("Choi", 2)
)
val byGrade: Map<Int, List<Student>> = students.groupBy { it.grade }

// getOrPut - 캐시 패턴
val cache = mutableMapOf<String, String>()
fun getResult(key: String): String {
    return cache.getOrPut(key) { "computed_$key" }
}

// 구조 분해
for ((name, score) in scores) {
    println("$name: $score")
}
```

### 불변 vs 가변

```kotlin
// 불변 (읽기 전용) - 기본 권장
val immutableSet: Set<String> = setOf("A", "B")
val immutableMap: Map<String, Int> = mapOf("a" to 1)

// 가변 - 수정이 필요할 때만
val mutableSet: MutableSet<String> = mutableSetOf("A", "B")
mutableSet.add("C")

val mutableMap: MutableMap<String, Int> = mutableMapOf("a" to 1)
mutableMap["b"] = 2

// 불변으로 변환
val frozen: Set<String> = mutableSet.toSet()
```

### Android 실전 활용

```kotlin
// 1. Bundle - Map 구조
val bundle = Bundle().apply {
    putString("title", "Hello")
    putInt("count", 42)
    putStringArrayList("tags", arrayListOf("android", "kotlin"))
}

// 2. RecyclerView DiffUtil - Set으로 변경 감지
class ItemDiffCallback(
    private val oldSet: Set<Item>,
    private val newSet: Set<Item>
) {
    val added = newSet - oldSet
    val removed = oldSet - newSet
}

// 3. LRU 캐시 - LinkedHashMap 기반
class LruCache<K, V>(private val maxSize: Int) : LinkedHashMap<K, V>(
    maxSize, 0.75f, true // accessOrder = true
) {
    override fun removeEldestEntry(eldest: MutableMap.MutableEntry<K, V>?): Boolean {
        return size > maxSize
    }
}

// 4. SharedPreferences - Map 구조
// val prefs = context.getSharedPreferences("settings", Context.MODE_PRIVATE)
// val allEntries: Map<String, *> = prefs.all

// 5. 태그/필터 관리
class FilterManager {
    private val activeFilters = mutableSetOf<String>()

    fun toggle(filter: String) {
        if (filter in activeFilters) activeFilters.remove(filter)
        else activeFilters.add(filter)
    }

    fun isActive(filter: String) = filter in activeFilters // O(1)
}

// 6. StateFlow로 Map 상태 관리
class SettingsViewModel : ViewModel() {
    private val _settings = MutableStateFlow<Map<String, Any>>(emptyMap())
    val settings: StateFlow<Map<String, Any>> = _settings.asStateFlow()

    fun updateSetting(key: String, value: Any) {
        _settings.update { it + (key to value) }
    }
}
```

## 🔗 관련 개념

- [[00-CS-기초/자료구조/해시맵|해시맵]]
- [[00-CS-기초/자료구조/트리|트리 (TreeMap/TreeSet)]]
- [[00-CS-기초/자료구조/배열-vs-연결리스트|배열 vs 연결리스트]]
- [[00-CS-기초/보안-기초/해싱|해싱]]

## 📚 더 보기

- [Kotlin Collections - Set](https://kotlinlang.org/docs/set-specific-operations.html)
- [Kotlin Collections - Map](https://kotlinlang.org/docs/map-specific-operations.html)
- [Android LruCache](https://developer.android.com/reference/android/util/LruCache)

---

**핵심 요약:** Set은 중복 없는 집합, Map은 키-값 매핑이며, 순서가 필요 없으면 Hash 계열, 삽입 순서면 LinkedHash 계열, 정렬이면 Tree 계열을 선택한다.
