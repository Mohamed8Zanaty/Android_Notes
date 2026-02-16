# 🧩 Serialization: The Complete Guide

> [!abstract] Definition
> 
> **Serialization** is the process of converting an object's state (its data) into a format that can be easily stored or transmitted and then reconstructed later. The reverse process—turning that format back into an object—is called **Deserialization**.

---

## 🏗️ 1. Why is it Necessary?

Objects in memory are live instances with complex structures. However, when you need to send data over a network or save it to a file, that data must be in a flat, sequential format (like a string or a stream of bytes).

### Common Use Cases:

- **Network Requests:** Sending an object from a mobile app to a server API.
    
- **Persistence:** Saving user settings or app state to local storage.
    
- **Inter-process Communication:** Passing data between different parts of a system (e.g., between screens in an Android app).
    

---

## 📄 2. Common Formats

|**Format**|**Description**|**Pros**|**Cons**|
|---|---|---|---|
|**JSON**|Text-based, human-readable.|Universal support, easy to debug.|Larger file size, slower than binary.|
|**XML**|Tag-based markup.|Highly structured.|Very verbose, harder to parse.|
|**Protobuf**|Binary format by Google.|Extremely fast and small.|Not human-readable without tools.|

---

## ⚙️ 3. How it Works (Kotlin Example)

In modern development, libraries like `kotlinx.serialization` automate the conversion by using annotations.

### Step A: The Model

You mark a class as `@Serializable` to tell the compiler it can be converted.

Kotlin

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class User(
    val name: String,
    val age: Int
)
```

### Step B: The Conversion

Kotlin

```kotlin
val user = User("Alice", 30)

// Encoding (Serialization)
val jsonString = Json.encodeToString(user) 
// Result: {"name":"Alice","age":30}

// Decoding (Deserialization)
val obj = Json.decodeFromString<User>(jsonString)
```

---

## ⚠️ 4. Key Concepts to Remember

> [!warning] Security Risk
> 
> Never deserialize data from an untrusted source without validation. Maliciously crafted serialized data can lead to remote code execution (RCE) in some systems.

- **Transient Fields:** Sometimes you don't want certain data (like a password or a temporary counter) to be saved. You can mark these as `@Transient` to skip them during serialization.
    
- **Versioning:** If you change your class (e.g., add a new field) but try to load an old serialized file, the app might crash. You must handle "Schema Evolution" by providing default values.
    

---

## 🔗 Related Concepts

- [[JSON vs XML]]
    
- [[Deep Linking and Data Passing]]
    
- [[Persistent Storage Patterns]]