# Recomposition 최적화

> 불필요한 Recomposition을 방지하여 성능 향상

## Recomposition이란?

State가 변경되면 해당 State를 읽는 Composable이 다시 실행됨.

## 최적화 원칙

### 1. State 읽기 지연 (람다 사용)

```kotlin
// 나쁜 예: offset 변경 시 전체 Recomposition
@Composable
fun MyComposable(offset: Int) {
    Box(modifier = Modifier.offset(y = offset.dp))
}

// 좋은 예: 람다로 읽기 지연
@Composable
fun MyComposable(offset: () -> Int) {
    Box(modifier = Modifier.offset { IntOffset(0, offset()) })
}
```

### 2. Stable/Immutable 클래스 사용

```kotlin
// 불안정한 클래스 - 항상 Recomposition
data class User(var name: String)

// 안정적인 클래스
@Stable
data class User(val name: String)

// 또는 불변
@Immutable
data class User(val name: String)
```

### 3. key() 사용

```kotlin
LazyColumn {
    items(users, key = { it.id }) { user ->
        UserItem(user)
    }
}
```

### 4. derivedStateOf

```kotlin
val listState = rememberLazyListState()

// 나쁜 예: 스크롤마다 Recomposition
val showButton = listState.firstVisibleItemIndex > 0

// 좋은 예: 값이 변경될 때만
val showButton by remember {
    derivedStateOf { listState.firstVisibleItemIndex > 0 }
}
```

### 5. remember 활용

```kotlin
// 나쁜 예: 매번 새 객체 생성
@Composable
fun MyComposable() {
    val formatter = DateFormatter()  // Recomposition마다 생성
}

// 좋은 예: 캐싱
@Composable
fun MyComposable() {
    val formatter = remember { DateFormatter() }
}
```

## Recomposition 범위 최소화

```kotlin
// 나쁜 예: 전체 Recomposition
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }

    Text("Count: $count")
    ExpensiveComposable()  // count와 무관하지만 Recomposition
}

// 좋은 예: 범위 분리
@Composable
fun Parent() {
    var count by remember { mutableStateOf(0) }

    CountText(count)  // 별도 Composable로 분리
    ExpensiveComposable()
}

@Composable
fun CountText(count: Int) {
    Text("Count: $count")
}
```

## 디버깅

Layout Inspector에서 Recomposition 횟수 확인 가능.

---

## 관련 문서
- [[05-Jetpack-Compose/기초/Recomposition|Recomposition]]
- [[05-Jetpack-Compose/상태-관리/State-Hoisting|State Hoisting]]

[← Jetpack Compose](05-Jetpack-Compose/README.md)
