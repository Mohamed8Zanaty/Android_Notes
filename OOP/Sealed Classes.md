# 🧱 Sealed Classes in Kotlin

> [!abstract] Definition **Sealed classes** and interfaces represent restricted class hierarchies that provide more control over inheritance. All subclasses of a sealed class are known at compile time, making them ideal for representing a fixed set of types.

---

## 🏗️ 1. Why Use Sealed Classes?

In standard inheritance, any class can be extended by an unknown number of subclasses. Sealed classes "seal" the hierarchy, ensuring that no other subclasses can be created outside of the module where the sealed class is defined.

### Key Benefits:

- **Exhaustive `when` Expressions:** The compiler knows all possible subclasses, so you don't need an `else` clause in a `when` block.
    
- **Type Safety:** It prevents accidental inheritance and ensures logic covers every possible state.
    
- **State Representation:** Frequently used to manage UI states (Loading, Success, Error).
    

---

## 📊 2. Sealed Classes vs. Enums

|Feature|Enum Classes|Sealed Classes|
|---|---|---|
|**Instance Limit**|Only one instance of each constant.|Can have multiple instances of each subclass.|
|**State**|All constants share the same properties.|Each subclass can have its own unique properties/data.|
|**Flexibility**|Rigid, constant values.|Dynamic; subclasses can be `data class`, `object`, or even another `sealed class`.|

---

## ⚙️ 3. Implementation Example

### Step A: Defining the Hierarchy

Sealed classes are perfect for handling the result of an operation, such as a network call.

Kotlin

```kotlin
sealed class Resource<out T> {
    data class Success<out T>(val data: T) : Resource<T>()
    data class Error(val message: String, val code: Int) : Resource<Nothing>()
    object Loading : Resource<Nothing>()
}
```

### Step B: Handling with `when`

Because the class is `sealed`, the compiler enforces that all branches are covered.

Kotlin

```kotlin
fun handleResponse(result: Resource<String>) {
    when (result) {
        is Resource.Success -> {
            println("Data received: ${result.data}")
        }
        is Resource.Error -> {
            println("Error ${result.code}: ${result.message}")
        }
        Resource.Loading -> {
            println("Loading...")
        }
        // No 'else' branch required!
    }
}
```

---

## 📐 4. Common Use Cases

### 1. UI State Management

In modern Android development, sealed classes are used to define the "State" of a screen.

Kotlin

```kotlin
sealed class UiState {
    object Idle : UiState()
    object Loading : UiState()
    data class Success(val items: List<String>) : UiState()
    data class Error(val exception: Throwable) : UiState()
}
```

### 2. Navigation Routes (Legacy/Traditional)

Before Type-Safe Navigation (Serialization), sealed classes were the standard way to define routes.

Kotlin

```kotlin
sealed class Screen(val route: String) {
    object Home : Screen("home")
    object Profile : Screen("profile")
    data class Details(val id: Int) : Screen("details/$id")
}
```

---

## 💡 5. Pro Tips

> [!tip] Objects vs. Classes Use `object` for subclasses that don't hold data (like `Loading` or `Idle`) to save memory. Use `data class` for subclasses that need to carry unique information (like `Success` or `Error`).

> [!warning] Module Restriction In Kotlin 1.5 and later, subclasses of a sealed class can be defined in different files, but they **must** reside in the same Gradle module and the same package.
