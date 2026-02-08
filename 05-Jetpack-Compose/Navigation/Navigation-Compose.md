# Navigation Compose

> Jetpack Compose에서의 화면 전환

## 기본 설정

```kotlin
// build.gradle
implementation("androidx.navigation:navigation-compose:2.7.0")
```

## NavHost 설정

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onNavigateToDetail = { id ->
                    navController.navigate("detail/$id")
                }
            )
        }

        composable(
            route = "detail/{id}",
            arguments = listOf(navArgument("id") { type = NavType.StringType })
        ) { backStackEntry ->
            val id = backStackEntry.arguments?.getString("id")
            DetailScreen(id = id)
        }
    }
}
```

## 화면 전환

```kotlin
// 기본 이동
navController.navigate("detail/123")

// 백스택 제어
navController.navigate("home") {
    popUpTo("home") { inclusive = true }
}

// 뒤로 가기
navController.popBackStack()
```

## Type-Safe Navigation (권장)

```kotlin
// 경로 정의
@Serializable
object Home

@Serializable
data class Detail(val id: String)

// NavHost
NavHost(navController, startDestination = Home) {
    composable<Home> {
        HomeScreen(onNavigate = { navController.navigate(Detail("123")) })
    }
    composable<Detail> { backStackEntry ->
        val detail: Detail = backStackEntry.toRoute()
        DetailScreen(detail.id)
    }
}
```

## ViewModel과 함께

```kotlin
composable("detail/{id}") {
    val viewModel: DetailViewModel = hiltViewModel()
    DetailScreen(viewModel = viewModel)
}
```

## 딥링크

```kotlin
composable(
    route = "detail/{id}",
    deepLinks = listOf(navDeepLink { uriPattern = "app://detail/{id}" })
) {
    // ...
}
```

## Bottom Navigation

```kotlin
@Composable
fun MainScreen() {
    val navController = rememberNavController()

    Scaffold(
        bottomBar = {
            NavigationBar {
                items.forEach { item ->
                    NavigationBarItem(
                        selected = currentRoute == item.route,
                        onClick = {
                            navController.navigate(item.route) {
                                popUpTo(navController.graph.startDestinationId)
                                launchSingleTop = true
                            }
                        },
                        icon = { Icon(item.icon, null) },
                        label = { Text(item.label) }
                    )
                }
            }
        }
    ) {
        NavHost(navController, startDestination = "home") {
            // ...
        }
    }
}
```

---

## 관련 문서
- [[07-Jetpack-Libraries/Navigation/Navigation-Component|Navigation Component]]
- [[05-Jetpack-Compose/기초/Composition|Composition]]

[← Jetpack Compose](05-Jetpack-Compose/README.md)
