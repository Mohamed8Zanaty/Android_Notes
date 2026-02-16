# 🛠️ Jetpack Compose Navigation: Troubleshooting Guide

> [!abstract] Overview
> 
> Navigation in Compose can be tricky due to its asynchronous nature and the decoupling of the UI from the backstack. This guide covers common runtime crashes, logic errors, and architectural "gotchas" encountered when using the Navigation Component.

---

## 🚨 Common Runtime Errors

|**Error Message**|**Likely Cause**|**Solution**|
|---|---|---|
|**`IllegalArgumentException: Route X is not a valid destination`**|The route string or type used in `Maps()` hasn't been added to the `NavHost`.|Ensure the destination is defined using `composable<T>` or `composable("route")` inside the `NavHost`.|
|**`IllegalStateException: NavController is not available`**|Attempting to access the `NavController` before it is initialized or outside its valid scope.|Use `rememberNavController()` at the top level of your UI and pass events down via lambdas.|
|**`SerializationException`**|Missing `@Serializable` annotation or mismatch in data types when using Type-Safe navigation.|Verify all route classes are marked with `@Serializable` and match the `toRoute<T>()` call.|

---

## 🔄 Logic & Navigation Behavior Issues

### 1. The "Multiple Instances" Bug

**Symptom:** Pressing the back button multiple times shows the same screen repeatedly.

**Cause:** The app is adding a new instance of the destination to the backstack every time a navigation event occurs (common with Bottom Navigation).

**The Fix:**

Kotlin

```kotlin
navController.navigate(Screen.Profile) {
    // Pop up to the start destination to avoid building up a large stack
    popUpTo(navController.graph.findStartDestination().id) {
        saveState = true
    }
    // Avoid multiple copies of the same destination when reselecting the same item
    launchSingleTop = true
    // Restore state when reselecting a previously selected item
    restoreState = true
}
```

### 2. UI State Loss on Rotation

**Symptom:** Data in a `TextField` or scroll position disappears when navigating away and coming back.

**Cause:** Using `remember { ... }` instead of `rememberSaveable { ... }`.

**The Fix:** Use `rememberSaveable` for any UI-specific state that needs to survive process death or backstack transitions.

---

## 🏛️ Architectural Best Practices

> [!warning] ViewModel vs. NavController
> 
> **Never** pass a `NavController` into a `ViewModel`.
> 
> ViewModels should be agnostic of the UI's navigation framework. Instead, have the ViewModel expose a `State` or `Effect` that the Composable observes to trigger navigation.

### Guidelines for a Clean Graph:

- **State Hoisting:** Screens should be stateless. Pass `onNavigate` and `onBack` lambdas to your Composables.
    
- **Single Source of Truth:** Keep your navigation graph in a single `NavHost` file rather than spreading `navController.navigate` calls throughout every small UI component.
    
- **Encapsulate Routes:** Define your routes (Strings or Objects) in a dedicated `Routes.kt` file.
    

---

## 🔍 Debugging Tools

- **Logcat:** Filter by `NavController` to see route transitions and backstack changes.
    
- **Layout Inspector:** Use the "Compose" view in the Layout Inspector to see which Composables are currently in the composition tree.
    
- **Print the Backstack:**
    

Kotlin

```kotlin
navController.currentBackStack.value.forEach { entry ->
    println("Backstack: ${entry.destination.route}")
}
```