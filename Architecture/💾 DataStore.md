## 📚 Table of Contents

1. [What is DataStore?](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#1-what-is-datastore)
2. [DataStore vs SharedPreferences vs Room](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#2-datastore-vs-sharedpreferences-vs-room)
3. [Setup & Dependencies](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#3-setup--dependencies)
4. [Preferences DataStore](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#4-preferences-datastore)
5. [DataStore Manager](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#5-datastore-manager)
6. [Provide with Hilt](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#6-provide-with-hilt)
7. [ViewModel](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#7-viewmodel)
8. [Compose UI](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#8-compose-ui)
9. [Proto DataStore (Overview)](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#9-proto-datastore-overview)
10. [Full Flow Summary](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#10-full-flow-summary)
11. [Project Structure](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#11-project-structure)
12. [Quick Reference](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#12-quick-reference)

---

## 1. What is DataStore?

**DataStore** is a data storage solution for saving small pieces of data — like user preferences, settings, tokens, and flags — directly on the device using Kotlin **Coroutines** and **Flow**.

It comes in two flavors:

|Flavor|What it stores|Use when|
|---|---|---|
|**Preferences DataStore**|Key-value pairs (like SharedPreferences)|Simple settings — theme, language, token|
|**Proto DataStore**|Typed objects using Protocol Buffers|Complex structured data with type safety|

> 💡 **Analogy:** DataStore is like a small notepad the app keeps on the device. Every time the user changes a setting, you write it down. Every time the app starts, you read it back.

---

## 2. DataStore vs SharedPreferences vs Room

||SharedPreferences|DataStore|Room|
|---|---|---|---|
|**API style**|Synchronous (blocks UI)|Coroutines + Flow (async) ✅|Coroutines + Flow ✅|
|**Thread safe**|❌ No|✅ Yes|✅ Yes|
|**Error handling**|❌ Silent failures|✅ Throws exceptions you can catch|✅|
|**Best for**|Legacy code only|Settings, tokens, flags|Lists, complex data|
|**Google recommended**|❌ Deprecated|✅ Yes|✅ Yes|

> ⚠️ **SharedPreferences is deprecated.** All new Android projects should use DataStore instead.

---

## 3. Setup & Dependencies

### `libs.versions.toml`

```toml
[versions]
datastore = "1.1.1"

[libraries]
datastore-preferences = { group = "androidx.datastore", name = "datastore-preferences", version.ref = "datastore" }
```

### `build.gradle.kts` (app level)

```kotlin
dependencies {
    implementation(libs.datastore.preferences)
}
```

---

## 4. Preferences DataStore

DataStore stores data as **key-value pairs**. Each key has a specific type — String, Int, Boolean, Float, Long, or Double.

### Defining your keys

```kotlin
// data/local/datastore/PreferencesKeys.kt
import androidx.datastore.preferences.core.booleanPreferencesKey
import androidx.datastore.preferences.core.stringPreferencesKey
import androidx.datastore.preferences.core.intPreferencesKey

object PreferencesKeys {
    val IS_DARK_THEME = booleanPreferencesKey("is_dark_theme")
    val AUTH_TOKEN    = stringPreferencesKey("auth_token")
    val USER_NAME     = stringPreferencesKey("user_name")
    val FONT_SIZE     = intPreferencesKey("font_size")
}
```

### Key types available

|Function|Stores|
|---|---|
|`booleanPreferencesKey("name")`|`Boolean`|
|`stringPreferencesKey("name")`|`String`|
|`intPreferencesKey("name")`|`Int`|
|`longPreferencesKey("name")`|`Long`|
|`floatPreferencesKey("name")`|`Float`|
|`doublePreferencesKey("name")`|`Double`|
|`stringSetPreferencesKey("name")`|`Set<String>`|

> 💡 Define all your keys in one `PreferencesKeys` object — makes it easy to find and manage them all in one place.

---

## 5. DataStore Manager

The **DataStore Manager** is your single interface to read and write preferences. Think of it as the Repository layer for DataStore.

```kotlin
// data/local/datastore/UserPreferencesManager.kt
import android.content.Context
import androidx.datastore.core.DataStore
import androidx.datastore.preferences.core.Preferences
import androidx.datastore.preferences.core.edit
import androidx.datastore.preferences.core.emptyPreferences
import androidx.datastore.preferences.preferencesDataStore
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.catch
import kotlinx.coroutines.flow.map
import java.io.IOException

// Create the DataStore instance — one per file name, at the top level
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "user_preferences")

class UserPreferencesManager(private val context: Context) {

    // ─── READ ───────────────────────────────────────────────

    val isDarkTheme: Flow<Boolean> = context.dataStore.data
        .catch { e ->
            // Handle read errors gracefully — emit empty prefs instead of crashing
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs ->
            prefs[PreferencesKeys.IS_DARK_THEME] ?: false   // default = false
        }

    val authToken: Flow<String?> = context.dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs ->
            prefs[PreferencesKeys.AUTH_TOKEN]   // null if not set
        }

    val userName: Flow<String> = context.dataStore.data
        .catch { e ->
            if (e is IOException) emit(emptyPreferences())
            else throw e
        }
        .map { prefs ->
            prefs[PreferencesKeys.USER_NAME] ?: ""
        }

    // ─── WRITE ──────────────────────────────────────────────

    suspend fun setDarkTheme(enabled: Boolean) {
        context.dataStore.edit { prefs ->
            prefs[PreferencesKeys.IS_DARK_THEME] = enabled
        }
    }

    suspend fun saveAuthToken(token: String) {
        context.dataStore.edit { prefs ->
            prefs[PreferencesKeys.AUTH_TOKEN] = token
        }
    }

    suspend fun saveUserName(name: String) {
        context.dataStore.edit { prefs ->
            prefs[PreferencesKeys.USER_NAME] = name
        }
    }

    // ─── CLEAR ──────────────────────────────────────────────

    // Clear a single key
    suspend fun clearAuthToken() {
        context.dataStore.edit { prefs ->
            prefs.remove(PreferencesKeys.AUTH_TOKEN)
        }
    }

    // Clear everything — useful for logout
    suspend fun clearAll() {
        context.dataStore.edit { prefs ->
            prefs.clear()
        }
    }
}
```

---

## 6. Provide with Hilt

```kotlin
// di/DataStoreModule.kt
import android.content.Context
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object DataStoreModule {

    @Provides
    @Singleton
    fun provideUserPreferencesManager(
        @ApplicationContext context: Context
    ): UserPreferencesManager {
        return UserPreferencesManager(context)
    }
}
```

---

## 7. ViewModel

```kotlin
// SettingsViewModel.kt
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.combine
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch
import javax.inject.Inject

data class SettingsUiState(
    val isDarkTheme: Boolean = false,
    val userName: String = "",
    val authToken: String? = null
)

@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val preferencesManager: UserPreferencesManager
) : ViewModel() {

    // Combine multiple flows into one UiState
    val uiState: StateFlow<SettingsUiState> = combine(
        preferencesManager.isDarkTheme,
        preferencesManager.userName,
        preferencesManager.authToken
    ) { isDark, name, token ->
        SettingsUiState(
            isDarkTheme = isDark,
            userName = name,
            authToken = token
        )
    }.stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000),
        initialValue = SettingsUiState()
    )

    fun toggleDarkTheme(enabled: Boolean) {
        viewModelScope.launch {
            preferencesManager.setDarkTheme(enabled)
        }
    }

    fun updateUserName(name: String) {
        viewModelScope.launch {
            preferencesManager.saveUserName(name)
        }
    }

    fun logout() {
        viewModelScope.launch {
            preferencesManager.clearAll()   // wipe all prefs on logout
        }
    }
}
```

---

## 8. Compose UI

```kotlin
// SettingsScreen.kt
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel

@Composable
fun SettingsScreen(
    viewModel: SettingsViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(24.dp)
    ) {

        Text(
            text = "Settings",
            style = MaterialTheme.typography.headlineMedium
        )

        // ─── Dark Theme Toggle ───────────────────────────────
        Row(
            modifier = Modifier.fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column {
                Text(
                    text = "Dark Theme",
                    style = MaterialTheme.typography.titleMedium
                )
                Text(
                    text = "Switch app appearance",
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
            Switch(
                checked = uiState.isDarkTheme,
                onCheckedChange = { viewModel.toggleDarkTheme(it) }
            )
        }

        Divider()

        // ─── Username ────────────────────────────────────────
        var nameInput by remember(uiState.userName) {
            mutableStateOf(uiState.userName)
        }

        OutlinedTextField(
            value = nameInput,
            onValueChange = { nameInput = it },
            label = { Text("Username") },
            modifier = Modifier.fillMaxWidth()
        )

        Button(
            onClick = { viewModel.updateUserName(nameInput) },
            modifier = Modifier.align(Alignment.End),
            enabled = nameInput.isNotBlank()
        ) {
            Text("Save Name")
        }

        Divider()

        // ─── Logout ──────────────────────────────────────────
        OutlinedButton(
            onClick = { viewModel.logout() },
            modifier = Modifier.fillMaxWidth(),
            colors = ButtonDefaults.outlinedButtonColors(
                contentColor = MaterialTheme.colorScheme.error
            )
        ) {
            Text("Logout & Clear All")
        }
    }
}
```

---

## 9. Proto DataStore (Overview)

**Proto DataStore** stores typed objects instead of key-value pairs. It uses **Protocol Buffers** (.proto files) to define the schema.

Use it when:

- You have complex settings that belong together as one object
- You need strict type safety across all fields
- Your preferences have nested structure

```protobuf
// user_settings.proto
syntax = "proto3";

message UserSettings {
    bool is_dark_theme = 1;
    string user_name = 2;
    int32 font_size = 3;
}
```

> 💡 **For most apps, Preferences DataStore is enough.** Only switch to Proto DataStore when your settings become complex enough that key-value pairs are hard to manage.

---

## 10. Full Flow Summary

```
User toggles Dark Theme switch
          ↓
SettingsViewModel.toggleDarkTheme(true)
          ↓
viewModelScope.launch { ... }
          ↓
UserPreferencesManager.setDarkTheme(true)
          ↓
dataStore.edit { prefs[IS_DARK_THEME] = true }
          ↓
DataStore writes to disk 💾
          ↓
isDarkTheme Flow emits new value
          ↓
uiState StateFlow updates
          ↓
Compose re-renders → Switch is ON ✅
```

---

## 11. Project Structure

```
app/
├── di/
│   ├── NetworkModule.kt
│   ├── DatabaseModule.kt
│   └── DataStoreModule.kt              ← provides UserPreferencesManager
│
├── data/
│   ├── local/
│   │   ├── dao/
│   │   ├── model/
│   │   ├── AppDatabase.kt
│   │   └── datastore/
│   │       ├── PreferencesKeys.kt      ← all key definitions
│   │       └── UserPreferencesManager.kt ← read/write logic
│   │
│   ├── remote/
│   └── repository/
│
├── ui/
│   ├── screen/
│   │   └── SettingsScreen.kt
│   └── viewmodel/
│       └── SettingsViewModel.kt
│
├── MyApp.kt
└── MainActivity.kt
```

---

## 12. Quick Reference

|Question|Answer|
|---|---|
|What is DataStore?|Modern key-value storage — replacement for SharedPreferences|
|What are the two types?|Preferences DataStore (key-value) and Proto DataStore (typed objects)|
|Is it thread-safe?|✅ Yes — built on Coroutines and Flow|
|How do you read data?|Returns a `Flow` — observe it like any other Flow|
|How do you write data?|`dataStore.edit { prefs[KEY] = value }` inside a `suspend` function|
|How do you delete one key?|`prefs.remove(KEY)` inside `dataStore.edit`|
|How do you clear everything?|`prefs.clear()` inside `dataStore.edit` — great for logout|
|What is `emptyPreferences()`?|A safe fallback emitted when a read `IOException` occurs|
|When to use Proto DataStore?|When your settings are complex typed objects, not simple key-value pairs|
|Should I replace SharedPreferences?|✅ Yes — in all new code use DataStore|

---

## Common Use Cases

|What to store|Key type|Example|
|---|---|---|
|Auth token|`stringPreferencesKey`|JWT from login API|
|Dark / light theme|`booleanPreferencesKey`|User theme preference|
|Onboarding shown|`booleanPreferencesKey`|Skip intro on relaunch|
|Selected language|`stringPreferencesKey`|`"en"`, `"ar"`, `"fr"`|
|Font size|`intPreferencesKey`|`14`, `16`, `18`|
|Last sync time|`longPreferencesKey`|Unix timestamp|

---

> ✅ **You're ready!** DataStore fits perfectly alongside Room and Retrofit — use **Retrofit** for remote data, **Room** for local lists, and **DataStore** for simple settings and tokens. Together they cover every data storage need in a modern Android app.