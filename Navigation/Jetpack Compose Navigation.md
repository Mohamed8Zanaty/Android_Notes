# 🗺️ Jetpack Compose Navigation: The Complete Guide

> [!abstract] Overview
> 
> Navigation in Compose is built on the **Navigation Component**. It allows you to transition between different "screens" (Composables) while maintaining a backstack and handling state automatically. This guide focuses on the modern **Type-Safe** approach (Navigation 2.8.0+).

---

## 🏗️ 1. Core Components

To implement navigation, you need three primary pieces:

| **Component**       | **Responsibility**                                                                        |
| ------------------- | ----------------------------------------------------------------------------------------- |
| **`NavController`** | The central API. It tracks the backstack and handles the actual "moving" between screens. |
| **`NavHost`**       | A container that links the `NavController` with a navigation graph.                       |
| **`NavGraph`**      | A map that defines all possible "destinations" (screens) in your app.                     |

---

## ⚙️ 2. Type-Safe Implementation

Modern Compose navigation uses **Kotlin [[Serialization]]** instead of error-prone strings.

### Step A: Define Routes

Routes are defined as `@Serializable` objects or classes.

Kotlin

```kotlin
import kotlinx.serialization.Serializable

@Serializable
object Home

@Serializable
data class Details(val itemId: Int)
```

### Step B: Setting up the NavHost

This is where you map your routes to your Composable screens.

Kotlin

```kotlin
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = Home
    ) {
        // Destination 1: Home
        composable<Home> {
            HomeScreen(onNavigateToDetails = { id -> 
                navController.navigate(Details(itemId = id)) 
            })
        }

        // Destination 2: Details (Extracting arguments automatically)
        composable<Details> { backStackEntry ->
            val args = backStackEntry.toRoute<Details>()
            DetailsScreen(
                itemId = args.itemId,
                onBack = { navController.popBackStack() }
            )
        }
    }
}
```

---

## 🧪 3. Usage Scenario: State Hoisting

To keep your UI testable and clean, **never** pass the `NavController` directly into low-level Composables. Use lambdas instead.

> [!check] Professional Pattern
> 
> Kotlin
> 
> ```kotlin
> @Composable
> fun HomeScreen(onNavigateToDetails: (Int) -> Unit) {
>     Button(onClick = { onNavigateToDetails(42) }) {
>         Text("View Item 42")
>     }
> }
> ```

---

## ⚠️ 4. Troubleshooting & Best Practices

|**Common Issue**|**Solution**|
|---|---|
|**Backstack Spam**|Use `launchSingleTop = true` to prevent duplicate screens on the stack.|
|**Data Loss on Rotate**|Use `rememberSaveable` for UI state inside your Composables.|
|**Typo Crashes**|Use the **Type-Safe** (Serialization) method shown above to catch errors at compile-time.|

> [!tip] Single Top Navigation
> 
> To prevent multiple instances of the same screen when clicking a button repeatedly:
> 
> Kotlin
> 
> ```kotlin
> navController.navigate(route) {
>     launchSingleTop = true
>     restoreState = true
> }
> ```

---
