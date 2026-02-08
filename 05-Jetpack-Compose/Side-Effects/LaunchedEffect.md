---
tags: [compose, side-effect, launched-effect]
---

# LaunchedEffect

## 💡 핵심 개념

**LaunchedEffect**는 Composable 생명주기에 맞춰 코루틴을 실행한다. key 변경 시 재시작된다.

## 🔍 자세히

### 기본 사용법

```kotlin
@Composable
fun MyScreen(userId: String) {
    LaunchedEffect(userId) {  // userId 변경 시 재시작
        val user = loadUser(userId)
        // ...
    }
}

// 한 번만 실행
LaunchedEffect(Unit) {
    // 초기 로딩
}
```

### 취소

```kotlin
@Composable
fun SearchScreen() {
    val query by viewModel.query.collectAsStateWithLifecycle()
    
    LaunchedEffect(query) {
        // query 변경 시 이전 작업 취소되고 새로 시작
        delay(300)
        viewModel.search(query)
    }
}
```

### Side Effect 종류

```kotlin
// LaunchedEffect - 코루틴 실행
LaunchedEffect(key) { }

// DisposableEffect - 정리 필요
DisposableEffect(key) {
    onDispose { }
}

// rememberCoroutineScope - 이벤트 핸들러
val scope = rememberCoroutineScope()
Button(onClick = {
    scope.launch { }
})
```

## 💻 실전 예제

```kotlin
@Composable
fun UserScreen(viewModel: UserViewModel) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    
    // 초기 로딩
    LaunchedEffect(Unit) {
        viewModel.loadUser()
    }
    
    // Snackbar
    val snackbarHostState = remember { SnackbarHostState() }
    
    LaunchedEffect(uiState) {
        if (uiState is UiState.Error) {
            snackbarHostState.showSnackbar(uiState.message)
        }
    }
}
```

## 🔗 관련 개념

- [[05-Jetpack-Compose/Side-Effects/DisposableEffect]]
- [[02-Coroutines/기초/Coroutine이란]]

---

**핵심 요약:** Composable 생명주기에 맞춘 코루틴. key 변경 시 재시작. Unit으로 한 번만 실행.
