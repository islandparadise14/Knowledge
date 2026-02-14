---
tags: [cs, algorithm, big-o, time-complexity, performance]
---

# 시간 복잡도 (Big-O)

## 💡 핵심 개념

**Big-O 표기법**은 알고리즘의 성능을 입력 크기(n)에 대한 함수로 표현하는 방법이다. 최악의 경우 실행 시간이 어떻게 증가하는지를 나타내며, 상수 계수와 낮은 차수 항은 무시한다.

## 📌 왜 필요한가?

Android에서 RecyclerView에 수천 개 아이템을 표시하거나 Room DB에서 대량 조회할 때, 알고리즘 복잡도가 프레임 드롭과 ANR에 직접적 영향을 준다. 16ms 안에 한 프레임을 그려야 하는 환경에서 복잡도 이해는 필수다.

## 🔍 자세히

### 주요 복잡도 단계

| 복잡도 | 이름 | 예시 | n=1,000 |
|---|---|---|---|
| O(1) | 상수 | HashMap 조회 | 1 |
| O(log n) | 로그 | 이진 탐색 | ~10 |
| O(n) | 선형 | 리스트 순회 | 1,000 |
| O(n log n) | 선형 로그 | 정렬 (TimSort) | ~10,000 |
| O(n²) | 이차 | 이중 반복문 | 1,000,000 |
| O(2ⁿ) | 지수 | 부분집합 열거 | ~10³⁰¹ |

### Kotlin 코드로 보는 복잡도

```kotlin
// O(1) - 상수 시간
fun getFirst(list: List<Int>): Int = list[0]
map["key"]  // HashMap 조회

// O(log n) - 이진 탐색
fun binarySearch(sorted: List<Int>, target: Int): Int {
    var low = 0; var high = sorted.lastIndex
    while (low <= high) {
        val mid = (low + high) / 2
        when {
            sorted[mid] == target -> return mid
            sorted[mid] < target -> low = mid + 1
            else -> high = mid - 1
        }
    }
    return -1
}

// O(n) - 선형 탐색
fun findMax(list: List<Int>): Int {
    var max = list[0]
    for (item in list) if (item > max) max = item
    return max
}

// O(n²) - 이중 반복
fun hasDuplicate(list: List<Int>): Boolean {
    for (i in list.indices)
        for (j in i + 1 until list.size)
            if (list[i] == list[j]) return true
    return false
}
// → HashSet으로 O(n)으로 개선 가능
fun hasDuplicateFast(list: List<Int>): Boolean {
    return list.size != list.toHashSet().size
}
```

### Android 실무에서의 복잡도

```kotlin
// DiffUtil - O(n + d²), d는 변경된 아이템 수
// notifyDataSetChanged() - O(n) 전체 재배치 (비효율)
// → DiffUtil/ListAdapter 사용이 훨씬 효율적

// Room 인덱스 유무
// 인덱스 없음: SELECT * FROM users WHERE name = ? → O(n) 풀 스캔
// 인덱스 있음: → O(log n) B-Tree 탐색
@Entity(indices = [Index("name")])
data class User(@PrimaryKey val id: Int, val name: String)

// Kotlin 컬렉션 연산 체이닝
users
    .filter { it.age > 20 }    // O(n)
    .sortedBy { it.name }      // O(n log n)
    .take(10)                   // O(1)
// 전체: O(n log n)

// Sequence로 lazy 최적화 (중간 리스트 생성 방지)
users.asSequence()
    .filter { it.age > 20 }
    .sortedBy { it.name }
    .take(10)
    .toList()
```

### Kotlin 컬렉션 주요 연산 복잡도

| 연산 | ArrayList | LinkedList | HashMap | TreeMap |
|---|---|---|---|---|
| get(index) | O(1) | O(n) | - | - |
| get(key) | - | - | O(1) | O(log n) |
| add(끝) | O(1)* | O(1) | O(1)* | O(log n) |
| add(중간) | O(n) | O(1)** | - | - |
| contains | O(n) | O(n) | O(1) | O(log n) |
| remove | O(n) | O(1)** | O(1) | O(log n) |

\* 배열 확장 시 O(n), ** 위치를 알고 있을 때

### 공간 복잡도도 중요하다

```kotlin
// O(1) 공간 - in-place
fun reverseInPlace(arr: IntArray) {
    var l = 0; var r = arr.lastIndex
    while (l < r) {
        arr[l] = arr[r].also { arr[r] = arr[l] }
        l++; r--
    }
}

// O(n) 공간 - 새 리스트
val reversed = list.reversed()
```

Android는 메모리 제한이 있으므로, Bitmap 처리나 대용량 데이터 변환 시 공간 복잡도도 항상 고려해야 한다.

## 🔗 관련 개념

- [[00-CS-기초/알고리즘/정렬-알고리즘|정렬 알고리즘]]
- [[00-CS-기초/알고리즘/탐색-알고리즘|탐색 알고리즘]]
- [[00-CS-기초/자료구조/해시맵|해시맵]]
- [[00-CS-기초/자료구조/배열-vs-연결리스트|배열 vs 연결리스트]]

## 📚 더 보기

- [Big-O Cheat Sheet](https://www.bigocheatsheet.com/)
- [Kotlin Collections 성능](https://kotlinlang.org/docs/collections-overview.html)

---

**핵심 요약:** Big-O는 입력 크기 증가에 따른 알고리즘 성능 변화를 나타내며, Android의 16ms 프레임 제한 안에서 효율적 코드를 작성하려면 반드시 이해해야 한다.
