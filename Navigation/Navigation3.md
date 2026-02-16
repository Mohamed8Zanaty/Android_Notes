# Navigation3 (Nav3): The Future of Compose Navigation

> [!danger] Experimental Status
> 
> Nav3 is currently part of the **Navigation 2.8.0+** library but is considered the "next-generation" approach. It aims to simplify navigation by making it a simple function of **State**.

---

## 🏗️ 1. What is Nav3?

Unlike the traditional `NavHost`, which manages its own internal backstack and lifecycle, **Nav3** is designed to be **stateless**. It treats navigation as a standard Composable that reacts to a list of "keys."

|**Feature**|**Traditional Navigation**|**Navigation3 (Nav3)**|
|---|---|---|
|**Logic**|Managed by `NavController`|Managed by your own `State`|
|**Routes**|Strings or Serializable classes|Any Kotlin object (Keys)|
|**Complexity**|High (Deeply nested internals)|Low (Simplified UI logic)|

---

## ⚙️ 2. Core Implementation

In Nav3, you define a map of how to "provide" a screen for a specific key.

### Step A: Define your Keys

Kotlin

```kotlin
sealed interface Screen {
    @Serializable object Home : Screen
    @Serializable data class Details(val id: String) : Screen
}
```

### Step B: Using `NavGraphBuilder`

Nav3 allows you to create a "Provider" that decides what to show based on the current state.

Kotlin

```kotlin
val backstack = remember { mutableStateListOf<Screen>(Screen.Home) }

NavHost(
    backstack = backstack,
    onBack = { backstack.removeLast() }
) { key ->
    when (key) {
        is Screen.Home -> {
            HomeScreen(onNavigate = { id -> 
                backstack.add(Screen.Details(id)) 
            })
        }
        is Screen.Details -> {
            DetailScreen(id = key.id)
        }
    }
}
```

---

## 📐 3. Key Improvements in Nav3

### 1. Unified State

Because the backstack is just a `MutableList` (or a similar State holder), you can manipulate it like any other data. You can clear the history, swap screens, or perform complex transitions without fighting the `NavController` API.

### 2. Predictive Back Support

Nav3 is built from the ground up to support **Android’s Predictive Back** gestures automatically, providing smoother animations when a user starts to swipe back.

### 3. Better Multi-Module Support

Since destinations are just types, you don't need to share a massive `NavGraph` across modules. You only need to share the "Key" objects.

---

## ⚠️ 4. When to use Nav3?

> [!tip] Recommendation
> 
> - **Use Traditional Nav:** For large production apps that need stable, battle-tested deep linking and complex lifecycle management.
>     
> - **Use Nav3:** For apps that want "Pure Compose" logic, custom backstack handling, or those who find the current `NavController` too restrictive.
>     

---

## 🔗 Related Notes

- [[Type-Safe Navigation (Modern Compose)]]
    
- [[Jetpack Compose State Management]]
    
- [[Serialization: The Complete Guide]]