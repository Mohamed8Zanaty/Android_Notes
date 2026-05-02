## 📚 Table of Contents

1. [What is Room?](#1-what-is-room)
2. [Setup & Dependencies](#2-setup--dependencies)
3. [Entity — Your Table](#3-entity--your-table)
4. [DAO — Your Queries](#4-dao--your-queries)
5. [Database — The Main Class](#5-database--the-main-class)
6. [Repository](#6-repository)
7. [ViewModel](#7-viewmodel)
8. [Compose UI](#8-compose-ui)
9. [Full Flow Summary](#9-full-flow-summary)
10. [Project Structure](#10-project-structure)
11. [Room vs Retrofit — When to Use Each](#11-room-vs-retrofit--when-to-use-each)
12. [Quick Reference](#12-quick-reference)

---

## 1. What is Room?

**Room** is a local database library built on top of SQLite. It lets you store data **on the device** — no internet needed. Instead of writing raw SQL and managing cursors manually, Room lets you work with Kotlin classes and simple annotated functions.

| Without Room | With Room |
|---|---|
| Write raw SQL strings | Use annotations like `@Query`, `@Insert` |
| Manage SQLite cursors manually | Room returns Kotlin objects directly |
| Handle threads yourself | `suspend fun` + Flow handle it |
| Easy to make SQL typos (crash at runtime) | SQL is verified **at compile time** ✅ |

> 💡 **Analogy:** Room is like a smart filing cabinet. You tell it what to store and how to find it — it handles the actual filing, organizing, and retrieval for you.

---

## 2. Setup & Dependencies

### **Step 1 — `libs.versions.toml`** (the single place for all versions)

```toml
[versions]
room = "2.8.4"  
lifecycle = "2.10.0"  
coroutines = "1.10.2"  
ksp = "2.3.6"

[libraries]
room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }
lifecycle-viewmodel-compose = { group = "androidx.lifecycle", name = "lifecycle-viewmodel-compose", version.ref = "lifecycle" }
coroutines-android = { group = "org.jetbrains.kotlinx", name = "kotlinx-coroutines-android", version.ref = "coroutines" }

[plugins]
ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
```

---
### Step 2 — `build.gradle.kts` (project level)**

```kotlin
plugins {
    alias(libs.plugins.ksp) apply false
}
```

---

### Step 3 — `build.gradle.kts` (app level)**

```kotlin
plugins {
    alias(libs.plugins.ksp)   // KSP replaces kapt — faster & modern
}

dependencies {
    implementation(libs.room.runtime)
    implementation(libs.room.ktx)
    ksp(libs.room.compiler)   // ksp instead of kapt

    implementation(libs.lifecycle.viewmodel.compose)
    implementation(libs.coroutines.android)
}
```
---

## 3. Entity — Your Table

An **Entity** is a Kotlin data class that represents a **table** in your database. Each field = one column.

```kotlin
// Note.kt
import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "notes")
data class Note(

    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,           // Auto-incremented ID — Room handles this

    val title: String,
    val content: String,
    val timestamp: Long = System.currentTimeMillis()
)
```

### Annotation Breakdown

| Annotation | What it does |
|---|---|
| `@Entity(tableName = "notes")` | Marks this class as a database table named "notes" |
| `@PrimaryKey(autoGenerate = true)` | Makes `id` the unique key, auto-incremented by Room |
| `val id: Int = 0` | Default 0 so Room knows to generate the ID on insert |

> 💡 Every table **must** have a `@PrimaryKey`. Without it, Room won't compile.

---

## 4. DAO — Your Queries

**DAO** (Data Access Object) is an interface where you define all your database operations. Room generates the actual implementation for you at compile time.

```kotlin
// NoteDao.kt
import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface NoteDao {

    // Get all notes, ordered by newest first
    // Flow means the UI auto-updates when data changes
    @Query("SELECT * FROM notes ORDER BY timestamp DESC")
    fun getAllNotes(): Flow<List<Note>>

    // Get a single note by ID
    @Query("SELECT * FROM notes WHERE id = :noteId")
    suspend fun getNoteById(noteId: Int): Note?

    // Insert a new note (or replace if same ID exists)
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertNote(note: Note)

    // Update an existing note
    @Update
    suspend fun updateNote(note: Note)

    // Delete a note
    @Delete
    suspend fun deleteNote(note: Note)

    // Delete all notes
    @Query("DELETE FROM notes")
    suspend fun deleteAllNotes()
}
```

### Key Concepts

| Concept | Explanation |
|---|---|
| `Flow<List<Note>>` | A live stream — Compose auto-updates when the database changes |
| `suspend fun` | Runs in the background so it doesn't block the UI thread |
| `@Insert` | Room generates the SQL INSERT statement for you |
| `@Update` | Room generates UPDATE based on the primary key |
| `@Delete` | Room generates DELETE based on the primary key |
| `@Query("...")` | For custom SQL — Room **validates it at compile time** |
| `OnConflictStrategy.REPLACE` | If a record with the same ID exists, replace it |

> ✅ **Tip:** Use `Flow` (not `suspend`) for read queries so the UI reacts automatically to any data change — no manual refresh needed.

---

## 5. Database — The Main Class

This is the entry point to your database. It connects the Entity and DAO together.

```kotlin
// NoteDatabase.kt
import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(
    entities = [Note::class],   // List all your @Entity classes here
    version = 1,                // Increment this when you change the schema
    exportSchema = false
)
abstract class NoteDatabase : RoomDatabase() {

    abstract fun noteDao(): NoteDao  // Room generates this implementation

    companion object {
        @Volatile
        private var INSTANCE: NoteDatabase? = null

        fun getDatabase(context: Context): NoteDatabase {
            // Return existing instance or create a new one
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    NoteDatabase::class.java,
                    "note_database"         // The name of the .db file on disk
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}
```

> ⚠️ **Important:** When you change your Entity (add/remove/rename a column), you must **increment the version number** and provide a Migration — otherwise Room will crash on existing installs.

---

## 6. Repository

The Repository is an optional but **highly recommended** layer between the DAO and ViewModel. It gives you one clean place to manage data — local DB, remote API, or both.

```kotlin
// NoteRepository.kt
import kotlinx.coroutines.flow.Flow

class NoteRepository(private val noteDao: NoteDao) {

    // Expose the Flow directly from the DAO
    val allNotes: Flow<List<Note>> = noteDao.getAllNotes()

    suspend fun insertNote(note: Note) {
        noteDao.insertNote(note)
    }

    suspend fun updateNote(note: Note) {
        noteDao.updateNote(note)
    }

    suspend fun deleteNote(note: Note) {
        noteDao.deleteNote(note)
    }

    suspend fun deleteAllNotes() {
        noteDao.deleteAllNotes()
    }
}
```

> 💡 **Why Repository?** The ViewModel shouldn't know or care whether data comes from a local database, a remote API, or both. The Repository hides that complexity — making your code much easier to test and maintain.

---

## 7. ViewModel

The ViewModel exposes data from the Repository to the UI, and handles user actions (insert, delete, update).

```kotlin
// NoteViewModel.kt
import android.app.Application
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.viewModelScope
import kotlinx.coroutines.flow.SharingStarted
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.flow.stateIn
import kotlinx.coroutines.launch

data class NoteUiState(
    val notes: List<Note> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null
)

class NoteViewModel(application: Application) : AndroidViewModel(application) {

    private val repository: NoteRepository

    init {
        // Build DB and repository
        val db = NoteDatabase.getDatabase(application)
        repository = NoteRepository(db.noteDao())
    }

    // Convert Flow → StateFlow so Compose can observe it
    val uiState: StateFlow<NoteUiState> = repository.allNotes
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5000),
            initialValue = NoteUiState(isLoading = true)
        )
        .let { flow ->
            // Map the list into our UiState
            kotlinx.coroutines.flow.MutableStateFlow(NoteUiState()).also { state ->
                viewModelScope.launch {
                    repository.allNotes.collect { notes ->
                        state.value = NoteUiState(notes = notes)
                    }
                }
            }
        }

    fun addNote(title: String, content: String) {
        if (title.isBlank()) return
        viewModelScope.launch {
            repository.insertNote(
                Note(title = title.trim(), content = content.trim())
            )
        }
    }

    fun deleteNote(note: Note) {
        viewModelScope.launch {
            repository.deleteNote(note)
        }
    }

    fun deleteAll() {
        viewModelScope.launch {
            repository.deleteAllNotes()
        }
    }
}
```

---

## 8. Compose UI

The UI observes the ViewModel's state and renders accordingly, plus handles user input to add/delete notes.

```kotlin
// NoteScreen.kt
import androidx.compose.foundation.layout.*
import androidx.compose.foundation.lazy.LazyColumn
import androidx.compose.foundation.lazy.items
import androidx.compose.material.icons.Icons
import androidx.compose.material.icons.filled.Delete
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.lifecycle.viewmodel.compose.viewModel

@Composable
fun NoteScreen(viewModel: NoteViewModel = viewModel()) {

    val uiState by viewModel.uiState.collectAsState()

    var title by remember { mutableStateOf("") }
    var content by remember { mutableStateOf("") }

    Column(modifier = Modifier.fillMaxSize().padding(16.dp)) {

        Text(
            text = "My Notes",
            style = MaterialTheme.typography.headlineMedium
        )

        Spacer(modifier = Modifier.height(16.dp))

        // --- Input Fields ---
        OutlinedTextField(
            value = title,
            onValueChange = { title = it },
            label = { Text("Title") },
            modifier = Modifier.fillMaxWidth()
        )

        Spacer(modifier = Modifier.height(8.dp))

        OutlinedTextField(
            value = content,
            onValueChange = { content = it },
            label = { Text("Content") },
            modifier = Modifier.fillMaxWidth(),
            minLines = 3
        )

        Spacer(modifier = Modifier.height(8.dp))

        Button(
            onClick = {
                viewModel.addNote(title, content)
                title = ""     // Clear fields after saving
                content = ""
            },
            modifier = Modifier.align(Alignment.End),
            enabled = title.isNotBlank()
        ) {
            Text("Save Note")
        }

        Spacer(modifier = Modifier.height(16.dp))

        Divider()

        Spacer(modifier = Modifier.height(8.dp))

        // --- Notes List ---
        if (uiState.notes.isEmpty()) {
            Box(
                modifier = Modifier.fillMaxSize(),
                contentAlignment = Alignment.Center
            ) {
                Text(
                    text = "No notes yet. Add one above!",
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
            }
        } else {
            LazyColumn(modifier = Modifier.fillMaxSize()) {
                items(
                    items = uiState.notes,
                    key = { it.id }           // Helps Compose animate list changes
                ) { note ->
                    NoteCard(
                        note = note,
                        onDelete = { viewModel.deleteNote(note) }
                    )
                }
            }
        }
    }
}

@Composable
fun NoteCard(note: Note, onDelete: () -> Unit) {
    Card(
        modifier = Modifier
            .fillMaxWidth()
            .padding(vertical = 4.dp)
    ) {
        Row(
            modifier = Modifier
                .fillMaxWidth()
                .padding(16.dp),
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = note.title,
                    style = MaterialTheme.typography.titleMedium
                )
                if (note.content.isNotBlank()) {
                    Spacer(modifier = Modifier.height(4.dp))
                    Text(
                        text = note.content,
                        style = MaterialTheme.typography.bodySmall,
                        color = MaterialTheme.colorScheme.onSurfaceVariant
                    )
                }
            }
            IconButton(onClick = onDelete) {
                Icon(
                    imageVector = Icons.Default.Delete,
                    contentDescription = "Delete note",
                    tint = MaterialTheme.colorScheme.error
                )
            }
        }
    }
}
```

### Don't forget `MainActivity.kt`

```kotlin
class MainActivity : ComponentActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContent {
            MyAppTheme {
                NoteScreen()
            }
        }
    }
}
```

---

## 9. Full Flow Summary

```
User types a note and taps "Save"
          ↓
NoteViewModel.addNote(title, content)
          ↓
viewModelScope.launch { ... }         ← Background coroutine
          ↓
NoteRepository.insertNote(note)
          ↓
NoteDao.insertNote(note)
          ↓
Room writes to SQLite on disk 💾
          ↓
Flow in DAO emits the updated list
          ↓
Repository passes it to ViewModel
          ↓
StateFlow updates → Compose re-renders ✅
```

---

## 10. Project Structure

```
app/
├── data/
│   ├── local/
│   │   ├── entity/
│   │   │   └── Note.kt             ← @Entity — defines the table
│   │   ├── dao/
│   │   │   └── NoteDao.kt          ← @Dao — all queries
│   │   └── NoteDatabase.kt         ← @Database — singleton entry point
│   └── repository/
│       └── NoteRepository.kt       ← Bridges DAO and ViewModel
├── ui/
│   ├── viewmodel/
│   │   └── NoteViewModel.kt        ← Holds UiState, handles actions
│   └── screen/
│       └── NoteScreen.kt           ← Composable UI
└── MainActivity.kt
```

---

## 11. Room vs Retrofit — When to Use Each

| | Room | Retrofit |
|---|---|---|
| **Where is data stored?** | On the device (local) | On a remote server (internet) |
| **Works offline?** | ✅ Yes, always | ❌ No, needs internet |
| **Use case** | User preferences, saved items, offline cache | Fetching fresh data from an API |
| **Data lives** | Until app is uninstalled | Only while you're connected |
| **Best together?** | ✅ Yes — fetch with Retrofit, cache with Room | ✅ Yes — "offline-first" architecture |

> 💡 **Pro tip:** In production apps, you often use **both together** — Retrofit fetches data from the API, Room stores it locally, and the UI always reads from Room. This is called the **offline-first** pattern and is what Google recommends.

```
Retrofit → fetches from API → saves to Room → UI reads from Room
```

---

## 12. Quick Reference

| Question | Answer |
|---|---|
| What is `@Entity`? | Marks a data class as a database table |
| What is `@Dao`? | Interface where you define all database operations |
| What is `@PrimaryKey`? | The unique identifier for each row — every Entity needs one |
| What is `Flow` in DAO? | A live data stream — UI auto-updates when the DB changes |
| Why `suspend fun`? | Runs DB operations in the background, off the main thread |
| What is `@Database`? | The main class that connects all entities and DAOs |
| Why Repository? | Decouples ViewModel from data sources — easier to test and scale |
| What is `version = 1`? | The database schema version — must increment when you change the structure |
| What is `OnConflictStrategy`? | What to do when inserting a duplicate key: REPLACE, IGNORE, or ABORT |
| When to use `@Query`? | For custom SQL that `@Insert`, `@Update`, `@Delete` can't cover |

---

> ✅ **You're ready!** Room + Compose follows the same MVVM pattern as Retrofit — Entity → DAO → Database → Repository → ViewModel → UI. Once you know both, combine them for a powerful offline-first app.
