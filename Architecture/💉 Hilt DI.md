## 📚 Table of Contents

1. [What is Dependency Injection?](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#1-what-is-dependency-injection)
2. [What is Hilt?](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#2-what-is-hilt)
3. [Setup & Dependencies](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#3-setup--dependencies)
4. [Initialize Hilt](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#4-initialize-hilt)
5. [Providing Dependencies](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#5-providing-dependencies)
6. [Inject into ViewModel](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#6-inject-into-viewmodel)
7. [Inject into Composable Screen](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#7-inject-into-composable-screen)
8. [Full Project Wiring](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#8-full-project-wiring)
9. [Before vs After Hilt](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#9-before-vs-after-hilt)
10. [Project Structure](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#10-project-structure)
11. [Quick Reference](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#11-quick-reference)

---

## 1. What is Dependency Injection?

A **dependency** is any object your class needs to work. **Dependency Injection (DI)** means you don't create those objects inside the class — someone else creates them and hands them in.

```kotlin
// ❌ Without DI — class creates its own dependencies
class PostViewModel : ViewModel() {
    private val api = RetrofitClient.apiService         // created inside
    private val db = AppDatabase.getDatabase(context)  // created inside
    private val repo = PostRepositoryImpl(api, db.postDao())
}

// ✅ With DI — dependencies are handed in from outside
class PostViewModel(
    private val repo: PostRepository   // injected from outside
) : ViewModel()
```

> 💡 **Analogy:** Instead of a chef going to the farm to get eggs, the eggs are delivered to the kitchen door. The chef just cooks — no supply chain management needed.

---

## 2. What is Hilt?

**Hilt** is Google's official DI library for Android, built on top of Dagger. It reads your annotations and **automatically generates all the wiring code** at compile time — no manual `MyApp` setup needed.

|Manual DI (`MyApp`)|Hilt|
|---|---|
|You create every object by hand|Hilt creates and manages objects automatically|
|You pass dependencies manually|Hilt injects them where needed|
|Easy to forget to pass something|Compile-time safety — missing deps = build error|
|Works fine for small apps|Scales to any app size|

---

## 3. Setup & Dependencies

### `libs.versions.toml`

```toml
[versions]
hilt = "2.51.1"
hilt-compose = "1.2.0"
ksp = "1.9.22-1.0.17"

[libraries]
hilt-android = { group = "com.google.dagger", name = "hilt-android", version.ref = "hilt" }
hilt-compiler = { group = "com.google.dagger", name = "hilt-android-compiler", version.ref = "hilt" }
hilt-navigation-compose = { group = "androidx.hilt", name = "hilt-navigation-compose", version.ref = "hilt-compose" }

[plugins]
hilt = { id = "com.google.dagger.hilt.android", version.ref = "hilt" }
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

### `build.gradle.kts` (project level)

```kotlin
plugins {
    alias(libs.plugins.hilt) apply false
    alias(libs.plugins.ksp) apply false
}
```

### `build.gradle.kts` (app level)

```kotlin
plugins {
    alias(libs.plugins.hilt)
    alias(libs.plugins.ksp)
}

dependencies {
    implementation(libs.hilt.android)
    ksp(libs.hilt.compiler)

    // For injecting into Composable screens via viewModel()
    implementation(libs.hilt.navigation.compose)
}
```

---

## 4. Initialize Hilt

### Step 1 — Annotate your Application class

```kotlin
// MyApp.kt
import android.app.Application
import dagger.hilt.android.HiltAndroidApp

@HiltAndroidApp   // ← This one annotation replaces all manual setup in MyApp
class MyApp : Application()
```

That's it — no more manually creating `RetrofitClient`, `AppDatabase`, or `Repository` here.

### Step 2 — Register in `AndroidManifest.xml`

```xml
<application
    android:name=".MyApp"
    ... >
```

---

## 5. Providing Dependencies

A **Module** is where you tell Hilt _how_ to create your dependencies. You write this once — Hilt handles the rest everywhere.

### Providing Retrofit

```kotlin
// di/NetworkModule.kt
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)   // lives as long as the app
object NetworkModule {

    @Provides
    @Singleton                          // only one instance created ever
    fun provideRetrofit(): Retrofit {
        return Retrofit.Builder()
            .baseUrl("https://jsonplaceholder.typicode.com/")
            .addConverterFactory(GsonConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
        // Hilt automatically passes the Retrofit from above ↑
    }
}
```

### Providing Room

```kotlin
// di/DatabaseModule.kt
import android.content.Context
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(
        @ApplicationContext context: Context    // Hilt provides Context automatically
    ): AppDatabase {
        return AppDatabase.getDatabase(context)
    }

    @Provides
    @Singleton
    fun providePostDao(db: AppDatabase): PostDao {
        return db.postDao()
        // Hilt automatically passes the AppDatabase from above ↑
    }
}
```

### Providing Repository

```kotlin
// di/RepositoryModule.kt
import dagger.Binds
import dagger.Module
import dagger.hilt.InstallIn
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    // Tells Hilt: when PostRepository is needed, provide PostRepositoryImpl
    abstract fun bindPostRepository(
        impl: PostRepositoryImpl
    ): PostRepository
}
```

> 💡 Use `@Binds` when binding an interface to its implementation — it's more efficient than `@Provides`.

---

## 6. Inject into ViewModel

```kotlin
// PostViewModel.kt
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltViewModel                               // ← tells Hilt to inject this ViewModel
class PostViewModel @Inject constructor(
    private val repository: PostRepository   // ← Hilt injects this automatically
) : ViewModel() {

    private val _posts = MutableStateFlow<List<NetworkPost>>(emptyList())
    val posts: StateFlow<List<NetworkPost>> = _posts

    val savedPosts = repository.getSavedPosts()

    init {
        fetchPosts()
    }

    fun fetchPosts() {
        viewModelScope.launch {
            try {
                _posts.value = repository.getAllPosts()
            } catch (e: Exception) {
                // handle error
            }
        }
    }

    fun savePost(post: NetworkPost) {
        viewModelScope.launch {
            repository.savePost(post)
        }
    }

    fun unsavePost(postId: Int) {
        viewModelScope.launch {
            repository.unsavePost(postId)
        }
    }
}
```

> ✅ No more `(application as MyApp).postRepo` — Hilt injects `PostRepository` directly.

---

## 7. Inject into Composable Screen

```kotlin
// PostScreen.kt
import androidx.compose.runtime.*
import androidx.hilt.navigation.compose.hiltViewModel  // ← use this, not viewModel()

@Composable
fun PostScreen(
    viewModel: PostViewModel = hiltViewModel()  // ← hiltViewModel() not viewModel()
) {
    val posts by viewModel.posts.collectAsState()
    val savedPosts by viewModel.savedPosts.collectAsState(emptyList())

    LazyColumn {
        items(posts, key = { it.id }) { post ->
            PostCard(
                post = post,
                isSaved = savedPosts.any { it.id == post.id },
                onSave = { viewModel.savePost(post) },
                onUnsave = { viewModel.unsavePost(post.id) }
            )
        }
    }
}
```

> ⚠️ Always use `hiltViewModel()` instead of `viewModel()` in Compose when using Hilt — otherwise Hilt won't know how to inject the ViewModel's dependencies.

---

## 8. Full Project Wiring

Hilt reads your annotations and builds this wiring automatically:

```
@HiltAndroidApp (MyApp)
        ↓
NetworkModule          DatabaseModule
    ↓                       ↓
Retrofit → ApiService   AppDatabase → PostDao
                ↓               ↓
          RepositoryModule
                ↓
         PostRepositoryImpl
                ↓
          @HiltViewModel
          PostViewModel
                ↓
          hiltViewModel()
          PostScreen
```

---

## 9. Before vs After Hilt

### Before (manual DI in `MyApp`):

```kotlin
class MyApp : Application() {
    lateinit var postRepo: PostRepository

    override fun onCreate() {
        super.onCreate()

        val retrofit = Retrofit.Builder()
            .baseUrl("https://...")
            .addConverterFactory(GsonConverterFactory.create())
            .build()

        val api = retrofit.create(ApiService::class.java)
        val db = AppDatabase.getDatabase(this)

        postRepo = PostRepositoryImpl(
            apiService = api,
            postDao = db.postDao()
        )
    }
}

// In ViewModel — ugly manual access
class PostViewModel(application: Application) : AndroidViewModel(application) {
    private val repo = (application as MyApp).postRepo
}
```

### After (Hilt):

```kotlin
// MyApp.kt — nothing to wire manually
@HiltAndroidApp
class MyApp : Application()

// ViewModel — clean injection
@HiltViewModel
class PostViewModel @Inject constructor(
    private val repo: PostRepository
) : ViewModel()
```

---

## 10. Project Structure

```
app/
├── di/                                     ← all Hilt modules live here
│   ├── NetworkModule.kt                    ← provides Retrofit + ApiService
│   ├── DatabaseModule.kt                   ← provides AppDatabase + DAOs
│   └── RepositoryModule.kt                 ← binds interfaces to implementations
│
├── data/
│   ├── local/
│   │   ├── dao/
│   │   │   └── PostDao.kt
│   │   ├── model/
│   │   │   └── PostEntity.kt
│   │   └── AppDatabase.kt
│   │
│   ├── remote/
│   │   ├── model/
│   │   │   └── NetworkPost.kt
│   │   ├── ApiService.kt
│   │   └── RetrofitClient.kt
│   │
│   └── repository/
│       ├── PostRepository.kt               ← interface
│       └── PostRepositoryImpl.kt           ← implementation
│
├── domain/
│   └── model/
│       └── Post.kt                         ← UI model (if needed)
│
├── ui/
│   ├── screen/
│   │   └── PostScreen.kt
│   └── viewmodel/
│       └── PostViewModel.kt
│
├── MyApp.kt                                ← @HiltAndroidApp
└── MainActivity.kt
```

---

## 11. Quick Reference

|Annotation|Where|What it does|
|---|---|---|
|`@HiltAndroidApp`|`Application` class|Initializes Hilt for the whole app|
|`@Module`|Object/class|Marks a class as a Hilt module (dependency provider)|
|`@InstallIn(SingletonComponent::class)`|Module|Makes dependencies live as long as the app|
|`@Provides`|Function in module|Tells Hilt how to create a dependency|
|`@Binds`|Abstract function|Binds an interface to its implementation|
|`@Singleton`|Anywhere|Only one instance created for the whole app|
|`@Inject constructor(...)`|Class constructor|Tells Hilt to inject this class's dependencies|
|`@HiltViewModel`|ViewModel|Enables Hilt injection into a ViewModel|
|`@ApplicationContext`|Parameter|Hilt injects the app Context automatically|
|`hiltViewModel()`|Composable|Gets a Hilt-injected ViewModel in Compose|

---

## Component Scopes — How Long Does a Dependency Live?

|Component|Scope Annotation|Lives as long as|
|---|---|---|
|`SingletonComponent`|`@Singleton`|The entire app|
|`ActivityComponent`|`@ActivityScoped`|One Activity|
|`ViewModelComponent`|`@ViewModelScoped`|One ViewModel|
|`FragmentComponent`|`@FragmentScoped`|One Fragment|

> 💡 For `Retrofit`, `AppDatabase`, and `Repository` — always use `@Singleton`. You only ever need one instance of these.

---

> ✅ **You're ready!** Hilt handles all the wiring you were doing manually in `MyApp`. Add `@HiltAndroidApp`, create your modules in `di/`, annotate your ViewModel with `@HiltViewModel`, and use `hiltViewModel()` in Compose. That's the complete setup for a production-grade Android app.