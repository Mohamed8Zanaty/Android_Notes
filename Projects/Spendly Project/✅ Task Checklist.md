
> Tags: `Hilt` `Room` `DataStore` `WorkManager` `UI`
---
## Step 1 — Project Setup

- [x] Create a new Android project in Android Studio
- [x] Add Hilt dependency in libs.versions.toml `Hilt`
- [x] Add Hilt Gradle plugin in build.gradle (project level) `Hilt`
- [x] Add kapt or KSP plugin in build.gradle (app level) `Hilt`
- [x] Add Room dependency in libs.versions.toml `Room`
- [x] Add DataStore dependency in libs.versions.toml `DataStore`
- [x] Add WorkManager dependency in libs.versions.toml `WorkManager`
- [x] Sync Gradle and confirm no errors

---

## Step 2 — Hilt Base Setup

- [x] Create MyApp class that extends Application `Hilt`
- [x] Annotate MyApp with @HiltAndroidApp `Hilt`
- [x] Register MyApp in AndroidManifest.xml with android:name `Hilt`
- [x] Annotate MainActivity with @AndroidEntryPoint `Hilt`
- [x] Create AppModule object annotated with @Module and @InstallIn(SingletonComponent::class) `Hilt`

---

## Step 3 — Data Models

- [x] Create Expense data class with id, amount, category, note, date fields `Room`
- [x] Annotate Expense with @Entity(tableName = "expenses") `Room`
- [x] Annotate id field with @PrimaryKey(autoGenerate = true) `Room`
- [x] Create Category enum with Food, Transport, Shopping, Bills, Other `UI`
- [x] Add @TypeConverter for Category enum so Room can store it `Room`

---

## Step 4 — Room Database

- [x] Create ExpenseDao interface `Room`
- [x] Add insertExpense suspend function to DAO `Room`
- [x] Add deleteExpense suspend function to DAO `Room`
- [x] Add getAllExpenses function that returns `Flow<List<Expense>>` `Room`
- [x] Add getTotalSpent function that returns `Flow<Double>` `Room`
- [x] Add getExpensesByCategory function that returns `Flow<List<Expense>>` `Room`
- [x] Create AppDatabase abstract class annotated with @Database `Room`
- [x] Register TypeConverters in AppDatabase `Room`
- [x] Provide AppDatabase as @Singleton in AppModule using @Provides `Hilt`
- [x] Provide ExpenseDao from AppDatabase in AppModule `Hilt`

---

## Step 5 — DataStore Setup

- [x] Create a PreferencesKeys object with keys for BUDGET and CURRENCY `DataStore`
- [x] Create UserPreferencesRepository interface with getMonthlyBudget() and saveBudget() `DataStore`
- [x] Create UserPreferencesRepositoryImpl that holds a `DataStore<Preferences>` `DataStore`
- [x] Implement getMonthlyBudget() returning `Flow<Double>` from DataStore `DataStore`
- [x] Implement saveBudget() using dataStore.edit { } `DataStore`
- [x] Provide DataStore instance in AppModule using @Provides @Singleton `Hilt`
- [x] Provide UserPreferencesRepository in AppModule `Hilt`

---

## Step 6 — Repository Layer

- [x] Create ExpenseRepository interface `Hilt`
- [x] Add getAllExpenses(), insertExpense(), deleteExpense(), getTotalSpent() to interface `Room`
- [x] Create ExpenseRepositoryImpl that takes ExpenseDao via @Inject constructor `Hilt`
- [x] Implement all functions by delegating to the DAO `Room`
- [x] Provide ExpenseRepository in AppModule binding ExpenseRepositoryImpl `Hilt`

---

## Step 7 — Navigation Setup

- [x] Create Route sealed interface with Dashboard, AddExpense, Transactions, Stats, Settings objects `UI`
- [x] Create NavRoot composable with NavHost `UI`
- [x] Add bottom navigation bar with 4 tabs (Dashboard, Transactions, Stats, Settings) `UI`
- [x] Add each screen as an empty composable in NavHost `UI`
- [x] Test navigation between all 4 tabs works `UI`

---

## Step 8 — Dashboard Screen

- [x] Create DashboardUiState with totalSpent, monthlyBudget, recentExpenses fields `UI`
- [x] Create DashboardViewModel annotated with @HiltViewModel `Hilt`
- [x] Inject ExpenseRepository and UserPreferencesRepository in DashboardViewModel `Hilt`
- [x] Combine totalSpent flow and budget flow into a single UiState flow `UI`
- [x] Create DashboardScreen composable that receives state and callbacks `UI`
- [x] Add a summary card showing spent / budget / remaining `UI`
- [x] Add a LazyColumn showing last 5 recent expenses `UI`
- [x] Add a FAB button that navigates to AddExpense screen `UI`

---

## Step 9 — Add Expense Screen

- [x] Create AddExpenseViewModel annotated with @HiltViewModel `Hilt`
- [x] Inject ExpenseRepository in AddExpenseViewModel `Hilt`
- [x] Create AddExpenseScreen composable with amount TextField `UI`
- [x] Add note TextField to the form `UI`
- [x] Add category selector (row of chips for each Category) `UI`
- [x] Add Save button that calls viewModel.addExpense() `UI`
- [x] Navigate back to Dashboard after saving `UI`

---

## Step 10 — Transactions Screen

- [x] Create TransactionsViewModel annotated with @HiltViewModel `Hilt`
- [x] Add selectedCategory state in ViewModel (null = show all) `UI`
- [x] Create TransactionsScreen with a LazyColumn of all expenses `UI`
- [x] Add category filter chips row at the top `UI`
- [x] Add swipe-to-delete or delete icon on each expense item `UI`

---

## Step 11 — Settings Screen

- [x] Create SettingsViewModel annotated with @HiltViewModel `Hilt`
- [x] Inject UserPreferencesRepository in SettingsViewModel `Hilt`
- [x] Collect budget flow in ViewModel and expose it as state `DataStore`
- [x] Create SettingsScreen with a TextField to enter monthly budget `UI`
- [x] Add Save button that calls viewModel.saveBudget(amount) `DataStore`
- [x] Add a toggle for enabling/disabling budget notifications `DataStore`
- [x] Save notification toggle preference in DataStore `DataStore`

---

## Step 12 — Stats Screen

- [x] Add a DAO query that returns total spent grouped by category `Room`
- [x] Create StatsViewModel annotated with @HiltViewModel `Hilt`
- [x] Create StatsScreen composable that receives category totals `UI`
- [x] Draw a simple bar chart using Canvas in Compose `UI`
- [x] Show each category name and amount below its bar `UI`

---

## Step 13 — WorkManager Notification

- [x] Create a notification channel in MyApp onCreate() `WorkManager`
- [x] Add POST_NOTIFICATIONS permission in AndroidManifest.xml `WorkManager`
- [x] Create BudgetCheckWorker class that extends CoroutineWorker `WorkManager`
- [x] Inject ExpenseDao and DataStore into the Worker via HiltWorker `Hilt`
- [x] Inside doWork() read totalSpent and monthlyBudget `WorkManager`
- [x] If spent >= 80% of budget send a notification `WorkManager`
- [x] Schedule BudgetCheckWorker as PeriodicWorkRequest every 24 hours `WorkManager`
- [x] Enqueue the worker in MyApp onCreate() with KEEP existing policy `WorkManager`