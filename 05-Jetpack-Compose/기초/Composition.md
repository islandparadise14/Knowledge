---
tags: [compose, composition]
---

# Composition

## 💡 핵심 개념

**Composition**은 Composable 함수를 실행해서 UI 트리를 구성하는 과정이다. Initial Composition과 Recomposition이 있다.

## 🔍 자세히

### Initial Composition

```kotlin
@Composable
fun MyScreen() {
    Column {
        Text("Hello")
        Button(onClick = {}) {
            Text("Click")
        }
    }
}
// 처음 실행 → Composition 트리 생성
```

### Composable 함수

```kotlin
@Composable
fun Greeting(name: String) {
    Text("Hello, $name")
}

// 사용
Greeting("민트")
```

### Composition Local

```kotlin
val LocalUser = compositionLocalOf<User> { error("No user") }

CompositionLocalProvider(LocalUser provides user) {
    // 하위 Composable에서 접근
    val currentUser = LocalUser.current
}
```

### Remember

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

## 💻 실전 예제

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    when (uiState) {
        is UiState.Loading -> LoadingView()
        is UiState.Success -> UserContent(uiState.data)
        is UiState.Error -> ErrorView(uiState.message)
    }
}
```

## 🔗 관련 개념

- [[05-Jetpack-Compose/기초/Recomposition]]
- [[05-Jetpack-Compose/상태-관리/Remember]]

---

**핵심 요약:** Composable 함수 실행 → UI 트리 생성. remember로 상태 보존.
