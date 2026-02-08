---
tags: [compose, recomposition, performance]
---

# Recomposition

## 💡 핵심 개념

**Recomposition**은 상태 변경 시 Composable 함수를 다시 실행하는 것이다. 변경된 부분만 업데이트한다.

## 🔍 자세히

### 동작 원리

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Column {
        Text("Count: $count")  // count 변경 시 이 부분만 recompose
        Button(onClick = { count++ }) {
            Text("Increment")
        }
    }
}
```

### 최적화

**1. 안정적인 타입 사용**
```kotlin
// 안정적 - 자동 최적화
@Composable
fun Item(text: String) { }

// 불안정 - 매번 recompose
@Composable
fun Item(list: MutableList<String>) { }  // 나쁨

// 안정적으로 변경
@Composable
fun Item(list: List<String>) { }  // 좋음
```

**2. remember 활용**
```kotlin
// 나쁨 - 매번 생성
@Composable
fun MyComposable() {
    val list = listOf(1, 2, 3)
}

// 좋음 - 캐싱
@Composable
fun MyComposable() {
    val list = remember { listOf(1, 2, 3) }
}
```

**3. derivedStateOf**
```kotlin
@Composable
fun List Items(items: List<Item>) {
    val filteredItems = remember(items) {
        derivedStateOf {
            items.filter { it.isActive }
        }
    }.value
}
```

## 💻 실전 예제

```kotlin
@Composable
fun UserList(users: List<User>) {
    LazyColumn {
        items(
            items = users,
            key = { it.id }  // key로 최적화
        ) { user ->
            UserItem(user)
        }
    }
}

@Composable
fun UserItem(user: User) {
    // user가 변경될 때만 recompose
    Row {
        Text(user.name)
        Text(user.email)
    }
}
```

## 🔗 관련 개념

- [[05-Jetpack-Compose/성능-최적화/Recomposition-최적화]]
- [[05-Jetpack-Compose/상태-관리/Remember]]

---

**핵심 요약:** 상태 변경 시 자동 UI 업데이트. 안정적 타입, remember, key 사용으로 최적화.
