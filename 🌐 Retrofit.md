## 📚 Table of Contents

1. [What is Retrofit?](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#1-what-is-retrofit)
2. [Setup & Dependencies](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#2-setup--dependencies)
3. [Data Model](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#3-data-model)
4. [API Interface](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#4-api-interface)
5. [Retrofit Client](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#5-retrofit-client)
6. [ViewModel](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#6-viewmodel)
7. [Compose UI](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#7-compose-ui)
8. [Full Flow Summary](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#8-full-flow-summary)
9. [Project Structure](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#9-project-structure)
10. [Quick Reference](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#10-quick-reference)

---

## 1. What is Retrofit?

**Retrofit** is a type-safe HTTP client for Android. Instead of writing complex network code, Retrofit lets you define your API as a simple Kotlin **interface** and it handles all the heavy lifting.

|Without Retrofit|With Retrofit|
|---|---|
|Write HTTP connections manually|Define an interface with annotations|
|Parse JSON by hand|Gson converts JSON → Kotlin automatically|
|Manage background threads yourself|`suspend fun` + Coroutines handle it|
|Hundreds of lines of boilerplate|~10 lines to make an API call|

> 💡 **Analogy:** Think of Retrofit like a waiter at a restaurant. You give it your order (API call), it goes to the kitchen (server), gets your food (data), and brings it back — you never have to enter the kitchen yourself.

---

## 2. Setup & Dependencies

### Step 1 — Add to `build.gradle` (app level)

```kotlin
dependencies {
    // Retrofit - the HTTP client
    implementation "com.squareup.retrofit2:retrofit:2.9.0"

    // Gson converter - converts JSON to Kotlin objects automatically
    implementation "com.squareup.retrofit2:converter-gson:2.9.0"

    // Coroutines - for async/background calls
    implementation "org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3"

    // ViewModel + Compose integration
    implementation "androidx.lifecycle:lifecycle-viewmodel-compose:2.7.0"
}
```

### Step 2 — Add Internet permission in `AndroidManifest.xml`

```xml
<!-- Inside <manifest> tag, BEFORE <application> -->
<uses-permission android:name="android.permission.INTERNET" />
```

> ⚠️ **Warning:** Forgetting this is the #1 beginner mistake. Your app will crash silently without it!

---

## 3. Data Model

We'll use the free **JSONPlaceholder** API (`https://jsonplaceholder.typicode.com`) as our example.

The API returns JSON that looks like this:

```json
{
  "id": 1,
  "name": "Leanne Graham",
  "email": "leanne@example.com"
}
```

Create a matching Kotlin data class — field names **must match** the JSON keys:

```kotlin
// User.kt
data class User(
    val id: Int,
    val name: String,
    val email: String
)
// That's it! Gson will automatically map JSON → User for you.
```

> ✅ **Tip:** The variable names in your data class must **exactly match** the JSON keys (case-sensitive). If the JSON says `"email"`, your class must also say `email`.

---

## 4. API Interface

This is the magic of Retrofit. You define an interface with annotated functions — each function = one API endpoint.

```kotlin
// ApiService.kt
import retrofit2.http.GET
import retrofit2.http.Path

interface ApiService {

    // GET https://jsonplaceholder.typicode.com/users
    @GET("users")
    suspend fun getUsers(): List<User>

    // GET https://jsonplaceholder.typicode.com/users/1
    @GET("users/{id}")
    suspend fun getUserById(@Path("id") id: Int): User
}
```

### Annotation Breakdown

|Annotation|What it does|
|---|---|
|`@GET("users")`|Calls GET /users — fetches all users|
|`suspend fun`|Runs in the background with Coroutines, doesn't block the UI|
|`List<User>`|Retrofit auto-converts the JSON array into a Kotlin list|
|`@Path("id")`|Replaces `{id}` in the URL with the actual value you pass|

> 💡 Other common annotations: `@POST`, `@PUT`, `@DELETE`, `@Query` (for URL params like `?page=1`), `@Body` (for sending JSON in the request body).

---

## 5. Retrofit Client

Create a **singleton** Retrofit instance. Build it once, reuse it everywhere.

```kotlin
// RetrofitClient.kt
import retrofit2.Retrofit
import retrofit2.converter.gson.GsonConverterFactory

object RetrofitClient {

    private const val BASE_URL = "https://jsonplaceholder.typicode.com/"

    // lazy = only created the first time it's accessed
    val apiService: ApiService by lazy {
        Retrofit.Builder()
            .baseUrl(BASE_URL)                       // The base of every URL
            .addConverterFactory(                    // Auto-convert JSON ↔ Kotlin
                GsonConverterFactory.create()
            )
            .build()
            .create(ApiService::class.java)          // Create our interface implementation
    }
}
```

> ✅ **Tip:** Use `object` (singleton) so you only ever create one Retrofit instance. Creating one per request wastes memory and is slow.

---

## 6. ViewModel

The ViewModel is responsible for fetching data and exposing it to the UI using **StateFlow**. Your Compose UI just observes and reacts — it never talks to Retrofit directly.

```kotlin
// UserViewModel.kt
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch

// Represents all possible states of the screen
sealed class UiState<out T> {
    object Loading : UiState<Nothing>()
    data class Success<T>(val data: T) : UiState<T>()
    data class Error(val message: String) : UiState<Nothing>()
}

class UserViewModel : ViewModel() {

    // Private — only this class can change the state
    private val _uiState = MutableStateFlow<UiState<List<User>>>(UiState.Loading)

    // Public — the UI reads this but cannot change it
    val uiState: StateFlow<UiState<List<User>>> = _uiState

    init {
        fetchUsers() // Auto-fetch when ViewModel is created
    }

    fun fetchUsers() {
        viewModelScope.launch {             // Runs in background thread
            _uiState.value = UiState.Loading
            try {
                val users = RetrofitClient.apiService.getUsers()
                _uiState.value = UiState.Success(users)
            } catch (e: Exception) {
                _uiState.value = UiState.Error("Failed to load: ${e.message}")
            }
        }
    }
}
```

### The 3 UI States

|State|When|What to show|
|---|---|---|
|`Loading`|Network request in flight|Spinner / progress bar|
|`Success`|Data arrived successfully|The list of users|
|`Error`|Something went wrong|Error message + Retry button|

---

## 7. Compose UI

The UI observes the ViewModel's state and re-renders automatically when it changes.

```kotlin
// UserScreen.kt
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun UserScreen(viewModel: UserViewModel = viewModel()) {

    // Observe state — re-renders whenever state changes
    val uiState by viewModel.uiState.collectAsState()

    when (val state = uiState) {

        // 🔄 Show spinner while loading
        is UiState.Loading -> {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                CircularProgressIndicator()
            }
        }

        // ✅ Show list when data arrives
        is UiState.Success -> {
            LazyColumn(modifier = Modifier.fillMaxSize()) {
                items(state.data) { user ->
                    UserCard(user = user)
                }
            }
        }

        // ❌ Show error with retry button
        is UiState.Error -> {
            Column(
                modifier = Modifier.fillMaxSize(),
                horizontalAlignment = Alignment.CenterHorizontally,
                verticalArrangement = Arrangement.Center
            ) {
                Text(
                    text = state.message,
                    color = MaterialTheme.colorScheme.error
                )
                Spacer(modifier = Modifier.height(12.dp))
                Button(onClick = { viewModel.fetchUsers() }) {
                    Text("Retry")
                }
            }
        }
    }
}

@Composable
fun UserCard(user: User) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(horizontal = 16.dp, vertical = 6.dp)
    ) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(
                text = user.name,
                style = MaterialTheme.typography.titleMedium
            )
            Text(
                text = user.email,
                style = MaterialTheme.typography.bodySmall
            )
        }
    }
}
```

> 💡 **`collectAsState()`** is the bridge between the ViewModel and Compose. Every time the `StateFlow` value changes, Compose automatically re-renders the relevant parts of the screen.

### Don't forget to add the screen in `MainActivity.kt`

```kotlin
// MainActivity.kt
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyAppTheme {
                UserScreen()
            }
        }
    }
}
```

---

## 8. Full Flow Summary

```
User opens screen
       ↓
ViewModel.init() → fetchUsers()
       ↓
viewModelScope.launch { ... }       ← Background coroutine starts
       ↓
RetrofitClient.apiService.getUsers()
       ↓
Retrofit builds HTTP GET request
       ↓
Server responds with JSON array
       ↓
Gson converts JSON → List<User>
       ↓
_uiState.value = UiState.Success(users)
       ↓
Compose re-renders → Shows the list ✅
```

---

## 9. Project Structure

```
app/
├── data/
│   ├── model/
│   │   └── User.kt                 ← Data class (maps JSON → Kotlin)
│   └── network/
│       ├── ApiService.kt           ← Interface with @GET, @POST etc.
│       └── RetrofitClient.kt       ← Singleton Retrofit instance
├── ui/
│   ├── viewmodel/
│   │   └── UserViewModel.kt        ← Fetches data, holds UiState
│   └── screen/
│       └── UserScreen.kt           ← Composable UI
└── MainActivity.kt                 ← Entry point, sets UserScreen()
```

---

## 10. Quick Reference

|Question|Answer|
|---|---|
|What does `@GET` do?|Marks a function as an HTTP GET request to that endpoint|
|What is `suspend fun` for?|Lets the function run in the background without blocking the UI thread|
|What does Gson do?|Converts JSON responses automatically into your Kotlin data classes|
|Why use `sealed class UiState`?|Safely represents all possible screen states: Loading, Success, Error|
|What is `collectAsState()`?|Bridges `StateFlow` → Compose so the UI auto-updates on state changes|
|Why `object RetrofitClient`?|Singleton pattern — one shared instance across the whole app|
|What is `viewModelScope`?|A coroutine scope tied to the ViewModel's lifecycle — auto-cancelled when destroyed|
|What is `LazyColumn`?|Compose's equivalent of `RecyclerView` — renders only visible items|

---

> ✅ **You're ready!** With this setup you can call any REST API, handle loading/error states gracefully, and display data in a clean Compose UI. The same pattern scales to any project — just swap out the data model, API interface, and UI card.