---
tags: [compose, state-hoisting, state-management]
---

# State Hoisting

## 💡 핵심 개념

**State Hoisting**은 상태를 Composable 밖으로 끌어올려 제어권을 상위에 전달하는 패턴이다. Stateless Composable을 만든다.

## 🔍 자세히

### Before (Stateful)

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

### After (Stateless)

```kotlin
@Composable
fun Counter(
    count: Int,
    onIncrement: () -> Unit
) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}

// 사용
@Composable
fun CounterScreen() {
    var count by remember { mutableStateOf(0) }
    
    Counter(
        count = count,
        onIncrement = { count++ }
    )
}
```

### 장점

1. **재사용성**: 다양한 상황에서 사용 가능
2. **테스트**: Preview 쉬움
3. **제어**: 상위에서 상태 관리

## 💻 실전 예제

```kotlin
// Stateless TextField
@Composable
fun SearchBar(
    query: String,
    onQueryChange: (String) -> Unit,
    onSearch: () -> Unit
) {
    TextField(
        value = query,
        onValueChange = onQueryChange,
        trailingIcon = {
            IconButton(onClick = onSearch) {
                Icon(Icons.Default.Search, null)
            }
        }
    )
}

// 사용
@Composable
fun SearchScreen(viewModel: SearchViewModel) {
    val query by viewModel.query.collectAsStateWithLifecycle()
    
    SearchBar(
        query = query,
        onQueryChange = { viewModel.updateQuery(it) },
        onSearch = { viewModel.search() }
    )
}
```

## 🔗 관련 개념

- [[05-Jetpack-Compose/상태-관리/State]]
- [[06-Architecture/상태-관리/UiState]]

---

**핵심 요약:** 상태를 상위로. 재사용성, 테스트 용이. value + callback 패턴.
