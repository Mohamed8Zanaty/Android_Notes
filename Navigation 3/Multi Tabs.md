## 📁 Project Structure

```
navigation/
├── Route.kt                # Serializable route definitions
├── BottomNavItem.kt        # Maps routes to icons/labels
├── NavigationState.kt      # State holder (map of back stacks + current tab)
├── Navigator.kt            # Navigation actions (navigate / goBack)
├── BottomNavBar.kt         # Bottom bar UI
└── NavRoot.kt              # Scaffold + NavDisplay integration
```

---

## 1️⃣ Defining Routes (`Route.kt`)

```kotlin
package com.example.communityboard.navigation

import androidx.navigation3.runtime.NavKey
import kotlinx.serialization.Serializable

@Serializable
sealed interface Route : NavKey {
    @Serializable data object Feed : Route
    @Serializable data object Saved : Route
}
```

- `NavKey` is the library’s required marker for navigation keys.
- `@Serializable` allows the navigation state to be automatically saved/restored (e.g., after process death).

---

## 2️⃣ Bottom Navigation Items (`BottomNavItem.kt`)

```kotlin
data class BottomNavItem(val icon: Int, val label: String)

val TOP_LEVEL_DESTINATIONS = mapOf(
    Route.Feed to BottomNavItem(icon = R.drawable.feed, label = "Feed"),
    Route.Saved to BottomNavItem(icon = R.drawable.saved, label = "Saved")
)
```

---

## 3️⃣ Navigation State with Per‑Tab Back Stacks (`NavigationState.kt`)

This is the **heart** of the solution.

```kotlin
class NavigationState(
    val startRoute: NavKey,
    topLevelRoute: MutableState<NavKey>,
    val backStacks: Map<NavKey, NavBackStack<NavKey>>
) {
    var topLevelRoute by topLevelRoute
    val stacksInUse: List<NavKey>
        get() = if (topLevelRoute == startRoute) listOf(startRoute)
                else listOf(startRoute, topLevelRoute)
}
```

- `backStacks` – each top‑level destination gets its own `NavBackStack` (automatically remembers its history).
- `stacksInUse` – the start route’s stack is **always kept alive** so switching back to it doesn’t lose its history.

### **Key Helper Functions in `NavigationState.kt`**

#### `rememberNavigationState` – creates the state with full serialization support

```kotlin
@Composable
fun rememberNavigationState(
    startRoute: NavKey,
    topLevelRoutes: Set<NavKey>
): NavigationState {
    val topLevelRoute = rememberSerializable(
        startRoute, topLevelRoutes,
        configuration = serializersConfig,
        serializer = MutableStateSerializer(PolymorphicSerializer(NavKey::class))
    ) { mutableStateOf(startRoute) }

    val backStacks = topLevelRoutes.associateWith { key ->
        rememberNavBackStack(configuration = serializersConfig, key)
    }

    return remember(startRoute, topLevelRoute) {
        NavigationState(startRoute, topLevelRoute, backStacks)
    }
}
```

#### `serializersConfig` – enables polymorphic serialization of `NavKey`

```kotlin
val serializersConfig = SavedStateConfiguration {
    serializersModule = SerializersModule {
        polymorphic(NavKey::class) {
            subclass(Route.Feed::class, Route.Feed.serializer())
            subclass(Route.Saved::class, Route.Saved.serializer())
        }
    }
}
```

Without this, the system wouldn’t know how to serialize `Route.Feed` or `Route.Saved` when saving the back stacks.

#### `toEntries` – converts back stacks into `NavEntry` list for `NavDisplay`

```kotlin
@Composable
fun NavigationState.toEntries(
    entryProvider: (NavKey) -> NavEntry<NavKey>
): SnapshotStateList<NavEntry<NavKey>> {
    val decoratedEntries = backStacks.mapValues { (_, stack) ->
        val decorators = listOf(
            rememberSaveableStateHolderNavEntryDecorator<NavKey>(),
            rememberViewModelStoreNavEntryDecorator()
        )
        rememberDecoratedNavEntries(
            backStack = stack,
            entryDecorators = decorators,
            entryProvider = entryProvider
        )
    }
    return stacksInUse
        .flatMap { decoratedEntries[it] ?: emptyList() }
        .toMutableStateList()
}
```

- **Decorators** – attach a `SaveableStateHolder` (retains `rememberSaveable` state) and a `ViewModelStore` (keeps `ViewModels` alive) to each entry.
- **`stacksInUse`** – only the start tab’s stack and the currently selected tab’s stack are **alive**; other tabs are frozen (the library handles this automatically when you switch).

---

## 4️⃣ Navigator Logic (`Navigator.kt`)

```kotlin
class Navigator(val state: NavigationState) {
    fun navigate(route: NavKey) {
        if (route in state.backStacks.keys) {
            // Top‑level destination → switch tab
            state.topLevelRoute = route
        } else {
            // Internal screen → push onto current tab’s back stack
            state.backStacks[state.topLevelRoute]?.add(route)
        }
    }

    fun goBack() {
        val currentBackStack = state.backStacks[state.topLevelRoute]
            ?: error("Back Stack for ${state.topLevelRoute} does not exist")
        val currentRoute = currentBackStack.last()
        if (currentRoute == state.topLevelRoute) {
            // At the root of current tab → fallback to start route
            state.topLevelRoute = state.startRoute
        } else {
            currentBackStack.removeLastOrNull()
        }
    }
}
```

- **Back button handling** – if you’re on a tab’s root screen (e.g., `Feed`), pressing back switches to the start route (here `Feed` again – but you could change it to something like a home screen).  
  In this implementation, pressing back from a root tab does **not** close the app; it remains on the same tab. To actually close the app, you’d need additional logic (e.g., check if the start route is the only thing left and then finish the activity).

---

## 5️⃣ Bottom Navigation Bar UI (`BottomNavBar.kt`)

A simple, recomposition‑friendly bar:

```kotlin
@Composable
fun BottomNavBar(
    modifier: Modifier = Modifier,
    selectedKey: NavKey,
    onSelectKey: (NavKey) -> Unit
) {
    BottomAppBar(modifier = modifier) {
        TOP_LEVEL_DESTINATIONS.forEach { (route, data) ->
            NavigationBarItem(
                selected = route == selectedKey,
                onClick = { onSelectKey(route) },
                icon = { Icon(painterResource(data.icon), data.label) },
                label = { Text(data.label) }
            )
        }
    }
}
```

---

## 6️⃣ Integrating Everything – `NavRoot.kt`

```kotlin
@Composable
fun NavRoot(modifier: Modifier = Modifier) {
    val navigationState = rememberNavigationState(
        startRoute = Route.Feed,
        topLevelRoutes = TOP_LEVEL_DESTINATIONS.keys
    )
    val navigator = remember { Navigator(navigationState) }

    Scaffold(
        modifier = modifier,
        bottomBar = {
            BottomNavBar(
                selectedKey = navigationState.topLevelRoute,
                onSelectKey = navigator::navigate
            )
        }
    ) { innerPadding ->
        NavDisplay(
            modifier = Modifier.fillMaxSize().padding(innerPadding),
            onBack = navigator::goBack,
            entries = navigationState.toEntries(
                entryProvider {
                    entry<Route.Feed> { FeedScreen() }
                    entry<Route.Saved> { SavedScreen() }
                }
            )
        )
    }
}
```

### How `NavDisplay` works

`NavDisplay` is a **custom composable** from the Navigation3 library that:

- Takes a list of `NavEntry` objects returned by `toEntries()`.
- Shows **only the screen that matches the currently selected tab** (the rest are kept in memory but not composed).
- Handles the back button via `onBack` lambda.
- Automatically animates transitions when you switch tabs or push/pop screens.

The `entryProvider` lambda defines **how to build a `NavEntry`** for a given route. In our case:
- When the route is `Route.Feed` → `FeedScreen()`
- When the route is `Route.Saved` → `SavedScreen()`
