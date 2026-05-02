## 🧠 The Idea in One Sentence

> A user sets a monthly budget, logs daily expenses, and gets a notification when they've spent 80% of their budget.

---

## 📱 The 5 Screens

|Screen|What it does|
|---|---|
|🏠 **Dashboard**|Shows total spent vs budget, recent transactions|
|➕ **Add Expense**|Form to log a new expense (amount, category, note)|
|📋 **Transactions**|Full list of all expenses, filterable by category|
|📊 **Stats**|Simple chart showing spending by category|
|⚙️ **Settings**|Set monthly budget, currency, notification toggle|

---

## 🛠️ What Each New Topic Teaches You

### 🗡️ Hilt — Dependency Injection

Replaces your `AppContainer` + `MyApp` manual wiring completely. Every ViewModel, Repository, and DAO gets injected automatically.

```kotlin
// Instead of AppContainer.provideViewModel(context)
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val repo: ExpenseRepository
) : ViewModel()
```

**What you'll learn:** `@HiltAndroidApp`, `@Inject`, `@HiltViewModel`, `@Module`, `@Provides`, `@Singleton`

---

### ⏰ WorkManager — Background Notifications

Every night at 8PM, a background worker checks if the user has exceeded 80% of their budget and fires a notification.

```kotlin
class BudgetCheckWorker(ctx: Context, params: WorkerParameters) : CoroutineWorker(ctx, params) {
    override suspend fun doWork(): Result {
        // check budget from Room
        // if spent > 80% → send notification
    }
}
```

**What you'll learn:** `PeriodicWorkRequest`, `WorkManager.enqueue()`, scheduling, notification channels

---

### 💾 DataStore — User Preferences

Stores the user's monthly budget amount, chosen currency, and notification preferences — no database needed for simple settings.

```kotlin
// Saves and reads preferences as a Flow
val monthlyBudget: Flow<Double> = dataStore.data.map { it[BUDGET_KEY] ?: 0.0 }
```

**What you'll learn:** `DataStore<Preferences>`, `Flow`-based reads, `edit {}` writes — and **when to use DataStore vs Room**

---

## ✅ Full Requirements List

- [ ] Dashboard showing spent / budget / remaining
- [ ] Add expense with amount, category, date, note
- [ ] Categories: Food, Transport, Shopping, Bills, Other
- [ ] Full transaction history screen
- [ ] Delete a transaction
- [ ] Filter transactions by category
- [ ] Simple bar or pie chart for spending by category
- [ ] Settings screen to set monthly budget
- [ ] DataStore saves budget + currency preference
- [ ] Notification when spending hits 80% of budget
- [ ] WorkManager runs the budget check daily
- [ ] Hilt wires everything — no manual AppContainer

---

## 🔁 How the Tech Fits Together

```
DataStore          Room DB
(budget limit)   (transactions)
      ↓                ↓
      └──── Repository ────┘
                  ↓
             ViewModel
                  ↓
              UI Screens

WorkManager (runs nightly)
      → reads Room + DataStore
      → fires Notification if over 80%
```

---

## 🪜 Order to Build It

|Step|What to build|New skill|
|---|---|---|
|**1**|Set up Hilt end to end|Hilt|
|**2**|Room DB for expenses + Repository|Room (revision)|
|**3**|Dashboard + Add Expense screens|Compose (revision)|
|**4**|Transactions list + delete + filter|ViewModel/StateFlow (revision)|
|**5**|Settings screen|DataStore|
|**6**|Stats screen with chart|Compose Canvas / Vico chart lib|
|**7**|Budget check notification|WorkManager|

---

## 🔥 Why This Project is Worth It

- **Hilt** is used in every professional Android project — learning it here means you're industry-ready
- **DataStore** teaches you the right tool for simple key-value storage vs Room for complex data
- **WorkManager** is the only correct way to do guaranteed background work on Android
- The app is **genuinely useful** — you'll actually use it
- No external API needed — 100% local, no network issues like last time

---

Every task is checkable and your progress is tracked. The color tags show which technology each task belongs to. Work through it step by step and don't move to the next section until the current one is fully checked. 🚀
