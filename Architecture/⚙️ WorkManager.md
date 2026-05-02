## 📚 Table of Contents

1. [What is WorkManager?](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#1-what-is-workmanager)
2. [WorkManager vs Other Background Options](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#2-workmanager-vs-other-background-options)
3. [Setup & Dependencies](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#3-setup--dependencies)
4. [Creating a Worker](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#4-creating-a-worker)
5. [Work Requests](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#5-work-requests)
6. [Constraints](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#6-constraints)
7. [Chaining Work](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#7-chaining-work)
8. [Provide with Hilt](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#8-provide-with-hilt)
9. [ViewModel](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#9-viewmodel)
10. [Compose UI](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#10-compose-ui)
11. [Observing Work Status](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#11-observing-work-status)
12. [Full Flow Summary](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#12-full-flow-summary)
13. [Project Structure](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#13-project-structure)
14. [Quick Reference](https://claude.ai/chat/b8fcc497-55b8-4054-bc53-1aa14c0f3a03#14-quick-reference)

---

## 1. What is WorkManager?

**WorkManager** is a library for scheduling **guaranteed background tasks** — work that must complete even if the user closes the app, the system kills the process, or the device restarts.

It handles:

- Syncing data with a server in the background
- Uploading images or files
- Sending logs or analytics
- Periodic cleanup tasks
- Tasks that require certain conditions (Wi-Fi only, battery not low, etc.)

> 💡 **Analogy:** WorkManager is like setting an alarm on your phone. Even if you close the clock app, the alarm still rings at the right time. WorkManager guarantees your task runs — regardless of what happens to your app.

---

## 2. WorkManager vs Other Background Options

| |Coroutines|WorkManager|Foreground Service|
|---|---|---|---|
|**Survives app close**|❌ No|✅ Yes|✅ Yes|
|**Survives reboot**|❌ No|✅ Yes|❌ No|
|**Runs immediately**|✅ Yes|✅ Yes (with delay option)|✅ Yes|
|**Repeating tasks**|❌ Manual|✅ Built-in|❌ Manual|
|**Conditions (Wi-Fi etc.)**|❌ No|✅ Built-in|❌ No|
|**Best for**|In-app async work|Guaranteed deferred work|Music, location tracking|

> 💡 **Rule of thumb:** Use **Coroutines** for work that only matters while the app is open. Use **WorkManager** for work that must complete no matter what.

---

## 3. Setup & Dependencies

### `libs.versions.toml`

```toml
[versions]
workmanager = "2.9.0"
hilt-work = "1.2.0"

[libraries]
workmanager = { group = "androidx.work", name = "work-runtime-ktx", version.ref = "workmanager" }
hilt-work = { group = "androidx.hilt", name = "hilt-work", version.ref = "hilt-work" }
hilt-work-compiler = { group = "androidx.hilt", name = "hilt-compiler", version.ref = "hilt-work" }
```

### `build.gradle.kts` (app level)

```kotlin
dependencies {
    implementation(libs.workmanager)

    // Only if using Hilt with WorkManager
    implementation(libs.hilt.work)
    ksp(libs.hilt.work.compiler)
}
```

---

## 4. Creating a Worker

A **Worker** is the class where you put the actual work you want to run in the background. Override `doWork()` and return a `Result`.

### Simple Worker

```kotlin
// worker/SyncPostsWorker.kt
import android.content.Context
import androidx.work.CoroutineWorker
import androidx.work.WorkerParameters
import androidx.work.workDataOf

class SyncPostsWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {   // CoroutineWorker = supports suspend functions

    override suspend fun doWork(): Result {
        return try {

            // Do your background work here
            // e.g. fetch from API and save to Room
            val posts = RetrofitClient.apiService.getPosts()
            // postDao.insertAll(posts.map { it.toEntity() })

            // ✅ Work completed successfully
            Result.success()

        } catch (e: Exception) {

            // Retry if this was a temporary failure (network error etc.)
            if (runAttemptCount < 3) {
                Result.retry()
            } else {
                // ❌ Give up after 3 attempts
                Result.failure()
            }
        }
    }

    companion object {
        const val WORK_NAME = "sync_posts_work"
    }
}
```

### Worker with Input & Output Data

```kotlin
// worker/UploadImageWorker.kt
class UploadImageWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {

        // Read input data passed when scheduling the work
        val imageUri = inputData.getString(KEY_IMAGE_URI)
            ?: return Result.failure()

        return try {
            // upload image...
            val imageUrl = uploadImage(imageUri)

            // Pass output data to the next worker in a chain
            val outputData = workDataOf(KEY_IMAGE_URL to imageUrl)
            Result.success(outputData)

        } catch (e: Exception) {
            Result.retry()
        }
    }

    companion object {
        const val KEY_IMAGE_URI = "key_image_uri"
        const val KEY_IMAGE_URL = "key_image_url"
    }
}
```

### The 3 Results

|Result|Meaning|
|---|---|
|`Result.success()`|Work finished — don't run again|
|`Result.failure()`|Work failed permanently — give up|
|`Result.retry()`|Work failed temporarily — try again later (uses backoff policy)|

---

## 5. Work Requests

A **WorkRequest** defines _when_ and _how_ to run your Worker. There are two types:

### OneTimeWorkRequest — run once

```kotlin
// Run immediately
val syncRequest = OneTimeWorkRequestBuilder<SyncPostsWorker>()
    .build()

// Run with input data
val uploadRequest = OneTimeWorkRequestBuilder<UploadImageWorker>()
    .setInputData(
        workDataOf(UploadImageWorker.KEY_IMAGE_URI to "content://image/123")
    )
    .setInitialDelay(10, TimeUnit.SECONDS)    // wait 10s before starting
    .addTag("upload_work")                    // tag for observing/cancelling
    .build()

// Enqueue it
WorkManager.getInstance(context).enqueue(uploadRequest)
```

### PeriodicWorkRequest — run repeatedly

```kotlin
// Run every 6 hours
val periodicSyncRequest = PeriodicWorkRequestBuilder<SyncPostsWorker>(
    repeatInterval = 6,
    repeatIntervalTimeUnit = TimeUnit.HOURS
)
    .addTag("periodic_sync")
    .build()

// Enqueue as unique to avoid duplicates
WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    SyncPostsWorker.WORK_NAME,              // unique name
    ExistingPeriodicWorkPolicy.KEEP,        // if already scheduled, keep it
    periodicSyncRequest
)
```

### ExistingPeriodicWorkPolicy options

|Policy|Behavior|
|---|---|
|`KEEP`|If already scheduled, keep the existing one — ignore new request|
|`REPLACE`|Cancel existing and schedule the new one|
|`UPDATE`|Update the existing work with new parameters|

> ⚠️ **Minimum interval:** PeriodicWorkRequest has a minimum interval of **15 minutes** — Android enforces this to save battery.

---

## 6. Constraints

**Constraints** let you define conditions that must be met before the work runs.

```kotlin
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)  // needs internet
    .setRequiresBatteryNotLow(true)                 // don't run on low battery
    .setRequiresStorageNotLow(true)                 // don't run when storage is full
    .setRequiresCharging(false)                     // doesn't need to be charging
    .build()

val syncRequest = OneTimeWorkRequestBuilder<SyncPostsWorker>()
    .setConstraints(constraints)
    .build()

// WorkManager waits until ALL constraints are satisfied, then runs the work
```

### Available Constraints

|Constraint|Options|
|---|---|
|`setRequiredNetworkType()`|`NOT_REQUIRED`, `CONNECTED`, `UNMETERED` (Wi-Fi only), `METERED`|
|`setRequiresBatteryNotLow()`|`true` / `false`|
|`setRequiresCharging()`|`true` / `false`|
|`setRequiresStorageNotLow()`|`true` / `false`|
|`setRequiresDeviceIdle()`|`true` / `false` (API 23+)|

---

## 7. Chaining Work

WorkManager lets you chain multiple workers together — the output of one becomes the input of the next.

```kotlin
val workManager = WorkManager.getInstance(context)

// Sequential chain — runs one after another
workManager
    .beginWith(compressImageRequest)     // Step 1: compress image
    .then(uploadImageRequest)            // Step 2: upload compressed image
    .then(notifyServerRequest)           // Step 3: notify server
    .enqueue()

// Parallel + merge — run A and B in parallel, then C when both finish
workManager
    .beginWith(listOf(syncPostsRequest, syncUsersRequest))  // parallel
    .then(updateCacheRequest)                               // runs after both finish
    .enqueue()
```

---

## 8. Provide with Hilt

To inject dependencies into a Worker, use `@HiltWorker` and `@AssistedInject`:

```kotlin
// worker/SyncPostsWorker.kt
import androidx.hilt.work.HiltWorker
import dagger.assisted.Assisted
import dagger.assisted.AssistedInject

@HiltWorker
class SyncPostsWorker @AssistedInject constructor(
    @Assisted context: Context,
    @Assisted params: WorkerParameters,
    private val repository: PostRepository   // ← injected by Hilt
) : CoroutineWorker(context, params) {

    override suspend fun doWork(): Result {
        return try {
            repository.syncPosts()   // use injected repo directly
            Result.success()
        } catch (e: Exception) {
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }

    companion object {
        const val WORK_NAME = "sync_posts_work"
    }
}
```

### Initialize Hilt + WorkManager in `MyApp`

```kotlin
// MyApp.kt
import androidx.hilt.work.HiltWorkerFactory
import androidx.work.Configuration
import dagger.hilt.android.HiltAndroidApp
import javax.inject.Inject

@HiltAndroidApp
class MyApp : Application(), Configuration.Provider {

    @Inject
    lateinit var workerFactory: HiltWorkerFactory

    // Tell WorkManager to use Hilt's factory to create workers
    override val workManagerConfiguration: Configuration
        get() = Configuration.Builder()
            .setWorkerFactory(workerFactory)
            .build()
}
```

### Disable default WorkManager initializer in `AndroidManifest.xml`

```xml
<!-- Required when using custom WorkManager configuration with Hilt -->
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="androidx.work.WorkManagerInitializer"
        android:value="androidx.startup"
        tools:node="remove" />
</provider>
```

### Provide WorkManager in Hilt

```kotlin
// di/WorkerModule.kt
import android.content.Context
import androidx.work.WorkManager
import dagger.Module
import dagger.Provides
import dagger.hilt.InstallIn
import dagger.hilt.android.qualifiers.ApplicationContext
import dagger.hilt.components.SingletonComponent
import javax.inject.Singleton

@Module
@InstallIn(SingletonComponent::class)
object WorkerModule {

    @Provides
    @Singleton
    fun provideWorkManager(
        @ApplicationContext context: Context
    ): WorkManager {
        return WorkManager.getInstance(context)
    }
}
```

---

## 9. ViewModel

```kotlin
// SyncViewModel.kt
import androidx.lifecycle.ViewModel
import androidx.lifecycle.viewModelScope
import androidx.work.*
import dagger.hilt.android.lifecycle.HiltViewModel
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.StateFlow
import kotlinx.coroutines.launch
import java.util.concurrent.TimeUnit
import javax.inject.Inject

data class SyncUiState(
    val isSyncing: Boolean = false,
    val lastSyncMessage: String = "Not synced yet",
    val workStatus: String = "IDLE"
)

@HiltViewModel
class SyncViewModel @Inject constructor(
    private val workManager: WorkManager
) : ViewModel() {

    private val _uiState = MutableStateFlow(SyncUiState())
    val uiState: StateFlow<SyncUiState> = _uiState

    // Observe work status as a Flow
    val syncWorkInfo = workManager
        .getWorkInfosForUniqueWorkFlow(SyncPostsWorker.WORK_NAME)

    // ─── One-time sync ───────────────────────────────────────
    fun syncNow() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .build()

        val request = OneTimeWorkRequestBuilder<SyncPostsWorker>()
            .setConstraints(constraints)
            .addTag(SyncPostsWorker.WORK_NAME)
            .build()

        workManager.enqueueUniqueWork(
            SyncPostsWorker.WORK_NAME,
            ExistingWorkPolicy.REPLACE,
            request
        )
    }

    // ─── Periodic sync ───────────────────────────────────────
    fun schedulePeriodicSync() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .build()

        val request = PeriodicWorkRequestBuilder<SyncPostsWorker>(6, TimeUnit.HOURS)
            .setConstraints(constraints)
            .build()

        workManager.enqueueUniquePeriodicWork(
            SyncPostsWorker.WORK_NAME,
            ExistingPeriodicWorkPolicy.KEEP,
            request
        )
    }

    // ─── Cancel ──────────────────────────────────────────────
    fun cancelSync() {
        workManager.cancelUniqueWork(SyncPostsWorker.WORK_NAME)
    }
}
```

---

## 10. Compose UI

```kotlin
// SyncScreen.kt
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.hilt.navigation.compose.hiltViewModel
import androidx.work.WorkInfo

@Composable
fun SyncScreen(
    viewModel: SyncViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsState()
    val workInfoList by viewModel.syncWorkInfo.collectAsState(initial = emptyList())

    // Derive current status from WorkInfo
    val workStatus = workInfoList.firstOrNull()?.state
    val isSyncing = workStatus == WorkInfo.State.RUNNING

    Column(
        modifier = Modifier
            .fillMaxSize()
            .padding(24.dp),
        verticalArrangement = Arrangement.spacedBy(16.dp),
        horizontalAlignment = Alignment.CenterHorizontally
    ) {

        Text(
            text = "Background Sync",
            style = MaterialTheme.typography.headlineMedium
        )

        // ─── Status Card ─────────────────────────────────────
        Card(modifier = Modifier.fillMaxWidth()) {
            Column(modifier = Modifier.padding(16.dp)) {
                Text(
                    text = "Current Status",
                    style = MaterialTheme.typography.titleSmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant
                )
                Spacer(modifier = Modifier.height(4.dp))
                Text(
                    text = when (workStatus) {
                        WorkInfo.State.RUNNING   -> "⏳ Syncing..."
                        WorkInfo.State.SUCCEEDED -> "✅ Sync complete"
                        WorkInfo.State.FAILED    -> "❌ Sync failed"
                        WorkInfo.State.ENQUEUED  -> "🕐 Waiting to sync"
                        WorkInfo.State.CANCELLED -> "🚫 Sync cancelled"
                        else                     -> "💤 Idle"
                    },
                    style = MaterialTheme.typography.bodyLarge
                )
            }
        }

        // ─── Sync Now ────────────────────────────────────────
        Button(
            onClick = { viewModel.syncNow() },
            modifier = Modifier.fillMaxWidth(),
            enabled = !isSyncing
        ) {
            if (isSyncing) {
                CircularProgressIndicator(
                    modifier = Modifier.size(18.dp),
                    strokeWidth = 2.dp,
                    color = MaterialTheme.colorScheme.onPrimary
                )
                Spacer(modifier = Modifier.width(8.dp))
            }
            Text(if (isSyncing) "Syncing..." else "Sync Now")
        }

        // ─── Schedule Periodic ───────────────────────────────
        OutlinedButton(
            onClick = { viewModel.schedulePeriodicSync() },
            modifier = Modifier.fillMaxWidth()
        ) {
            Text("Schedule Periodic Sync (every 6h)")
        }

        // ─── Cancel ──────────────────────────────────────────
        OutlinedButton(
            onClick = { viewModel.cancelSync() },
            modifier = Modifier.fillMaxWidth(),
            colors = ButtonDefaults.outlinedButtonColors(
                contentColor = MaterialTheme.colorScheme.error
            ),
            enabled = isSyncing || workStatus == WorkInfo.State.ENQUEUED
        ) {
            Text("Cancel Sync")
        }
    }
}
```

---

## 11. Observing Work Status

WorkManager exposes work status as a **Flow** so Compose can observe it live:

```kotlin
// Observe by unique work name
workManager.getWorkInfosForUniqueWorkFlow("sync_posts_work")

// Observe by tag
workManager.getWorkInfosByTagFlow("upload_work")

// Observe by work ID
workManager.getWorkInfoByIdFlow(workRequest.id)
```

### WorkInfo States

|State|Meaning|
|---|---|
|`ENQUEUED`|Scheduled, waiting for constraints to be met|
|`RUNNING`|Currently executing|
|`SUCCEEDED`|Finished successfully|
|`FAILED`|Failed permanently (returned `Result.failure()`)|
|`CANCELLED`|Manually cancelled|
|`BLOCKED`|In a chain, waiting for previous work to finish|

---

## 12. Full Flow Summary

```
User taps "Sync Now"
          ↓
SyncViewModel.syncNow()
          ↓
WorkManager.enqueueUniqueWork(...)
          ↓
WorkManager checks Constraints (network connected?)
          ↓
Constraints met → SyncPostsWorker.doWork() runs in background
          ↓
repository.syncPosts() → API → Room
          ↓
Result.success() returned
          ↓
WorkInfo.State changes to SUCCEEDED
          ↓
Flow emits new WorkInfo
          ↓
Compose re-renders → "✅ Sync complete" ✅
```

---

## 13. Project Structure

```
app/
├── di/
│   ├── NetworkModule.kt
│   ├── DatabaseModule.kt
│   ├── DataStoreModule.kt
│   └── WorkerModule.kt             ← provides WorkManager instance
│
├── data/
│   ├── local/
│   ├── remote/
│   └── repository/
│       └── PostRepository.kt
│
├── worker/                         ← all Worker classes live here
│   ├── SyncPostsWorker.kt
│   └── UploadImageWorker.kt
│
├── ui/
│   ├── screen/
│   │   └── SyncScreen.kt
│   └── viewmodel/
│       └── SyncViewModel.kt
│
├── MyApp.kt                        ← Configuration.Provider for Hilt + WorkManager
└── MainActivity.kt
```

---

## 14. Quick Reference

|Question|Answer|
|---|---|
|What is WorkManager for?|Background tasks that must complete even if app is closed or device restarts|
|What is `CoroutineWorker`?|A Worker that supports `suspend` functions — use this for all Kotlin projects|
|What is `doWork()`?|The function where your background logic goes — must return a `Result`|
|What are the 3 results?|`Result.success()`, `Result.failure()`, `Result.retry()`|
|What is a `Constraint`?|A condition that must be met before work runs (e.g. network connected)|
|What is `OneTimeWorkRequest`?|Run a task once|
|What is `PeriodicWorkRequest`?|Run a task repeatedly — minimum interval is 15 minutes|
|What is `enqueueUniqueWork`?|Schedule work with a unique name — prevents duplicate tasks|
|What is `@HiltWorker`?|Enables Hilt injection into a Worker class|
|How do I observe work in Compose?|`workManager.getWorkInfosForUniqueWorkFlow(name).collectAsState()`|
|When NOT to use WorkManager?|For immediate in-app work — use Coroutines instead|

---

## When to Use What

```
Task needs to run while app is open only
    → Coroutines + viewModelScope

Task must complete even if app is closed
    → WorkManager (OneTimeWorkRequest)

Task must run on a schedule (every few hours)
    → WorkManager (PeriodicWorkRequest)

Task needs conditions (Wi-Fi, battery, etc.)
    → WorkManager + Constraints

Task shows ongoing progress (music, location)
    → Foreground Service
```

---

> ✅ **You're ready!** WorkManager is the final piece of the modern Android data stack. Use **Retrofit** for remote data, **Room** for local storage, **DataStore** for settings, **Hilt** for wiring, and **WorkManager** for guaranteed background work.