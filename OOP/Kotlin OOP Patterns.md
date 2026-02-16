# Kotlin Object-Oriented Programming Patterns

> [!abstract] Overview
> 
> Kotlin enhances traditional Object-Oriented Programming (OOP) by providing first-class support for modern patterns and reducing boilerplate code. While Kotlin supports standard OOP principles (Encapsulation, Inheritance, Polymorphism, Abstraction), it introduces unique features like **Delegation**, **Data Classes**, and **Companion Objects** that change how patterns are implemented.

---

## 🏗️ 1. Creational Patterns

### The Singleton Pattern (`object`)

In Kotlin, you don't need to write private constructors or `getInstance()` methods. The `object` keyword handles thread-safe initialization automatically.

Kotlin

```
object DatabaseConfig {
    val name = "AppDB"
    fun connect() = println("Connected to $name")
}

// Usage
DatabaseConfig.connect()
```

### The Factory Pattern

Use **Companion Objects** to provide factory methods. This is cleaner than standard static methods in Java.

Kotlin

```
class User private constructor(val email: String) {
    companion object {
        fun createFromSocial(id: String): User {
            return User("$id@social.com")
        }
    }
}

// Usage
val user = User.createFromSocial("dev_user")
```

---

## 📐 2. Structural Patterns

### The Delegation Pattern (`by`)

Kotlin has native support for delegation, which is a powerful alternative to inheritance. It allows one class to delegate its responsibilities to another object.

Kotlin

```
interface Printer {
    fun printMessage()
}

class LaserPrinter : Printer {
    override fun printMessage() = println("Printing with Laser...")
}

// Delegation using 'by'
class OfficeManager(printer: Printer) : Printer by printer

// Usage
val laser = LaserPrinter()
val manager = OfficeManager(laser)
manager.printMessage() // Delegates call to laser
```

### The Decorator Pattern

Kotlin's **Extension Functions** allow you to "decorate" or add functionality to a class without inheriting from it or wrapping it.

Kotlin

```
fun String.capitalizeWords(): String = 
    split(" ").joinToString(" ") { it.replaceFirstChar { char -> char.uppercase() } }

// Usage
val title = "jetpack compose navigation".capitalizeWords()
```

---

## 🎭 3. Behavioral Patterns

### The Strategy Pattern

Because functions are first-class citizens in Kotlin, the Strategy pattern can often be replaced by passing a lambda or a function reference.

Kotlin

```
class PaymentProcessor(val strategy: (Int) -> Unit) {
    fun process(amount: Int) = strategy(amount)
}

// Usage
val creditCardStrategy = { amount: Int -> println("Paying $amount via Credit Card") }
val processor = PaymentProcessor(creditCardStrategy)
processor.process(100)
```

### The Observer Pattern

In modern Android development, the Observer pattern is typically implemented using **Flow** or **LiveData** within the `ViewModel`.

Kotlin

```
class UserViewModel : ViewModel() {
    private val _uiState = MutableStateFlow<UiState>(UiState.Loading)
    val uiState: StateFlow<UiState> = _uiState
}
```

---

## 📊 4. Comparison: Java vs. Kotlin OOP

|**Feature**|**Java**|**Kotlin**|
|---|---|---|
|**Singleton**|Complex (Private constructor + Static)|Built-in via `object` keyword|
|**Data Storage**|Verbose POJOs (Getters/Setters)|Concise `data class`|
|**Inheritance**|Open by default|Closed by default (use `open`)|
|**Interfaces**|No state allowed|Can have properties (no backing fields)|

---

## 💡 5. Pro Tips for OOP in Kotlin

> [!tip] Favor Composition over Inheritance
> 
> Kotlin's `by` keyword makes composition so easy that you should rarely need deep inheritance trees. Deep trees are harder to maintain and test.

- **Use `data class` for Models:** They automatically generate `equals()`, `hashCode()`, and `toString()`.
    
- **Sealed Interfaces for State:** Combine OOP principles with functional exhaustiveness to manage complex UI states.
    
- **Internal Visibility:** Use the `internal` modifier to keep your OOP logic encapsulated within a specific Gradle module.