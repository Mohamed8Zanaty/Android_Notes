
> Tags: `Hilt` `Room` `DataStore` `WorkManager` `UI`
---
## Step 1 — Project Setup

- [x] Create a new Android project in Android Studio
- [x] Add Hilt dependency in libs.versions.toml `Hilt`
- [x] Add Hilt Gradle plugin in build.gradle (project level) `Hilt`
- [x] Add kapt or KSP plugin in build.gradle (app level) `Hilt`
- [x] Add Room dependency in libs.versions.toml `Room`
- [ ] Add DataStore dependency in libs.versions.toml `DataStore`
- [ ] Add WorkManager dependency in libs.versions.toml `WorkManager`
- [ ] Sync Gradle and confirm no errors

---

## Step 2 — Hilt Base Setup

- [ ] Create MyApp class that extends Application `Hilt`
- [ ] Annotate MyApp with @HiltAndroidApp `Hilt`
- [ ] Register MyApp in AndroidManifest.xml with android:name `Hilt`
- [ ] Annotate MainActivity with @AndroidEntryPoint `Hilt`
- [ ] Create AppModule object annotated with @Module and @InstallIn(SingletonComponent::class) `Hilt`

---

## Step 3 — Data Models

- [ ] Create Expense data class with id, amount, category, note, date fields `Room`
- [ ] Annotate Expense with @Entity(tableName = "expenses") `Room`
- [ ] Annotate id field with @PrimaryKey(autoGenerate = true) `Room`
- [ ] Create Category enum with Food, Transport, Shopping, Bills, Other `UI`
- [ ] Add @TypeConverter for Category enum so Room can store it `Room`

---

## Step 4 — Room Database

- [ ] Create ExpenseDao interface `Room`
- [ ] Add insertExpense suspend function to DAO `Room`
- [ ] Add deleteExpense suspend function to DAO `Room`
- [ ] Add getAllExpenses function that returns `Flow<List<Expense>>` `Room`
- [ ] Add getTotalSpent function that returns `Flow<Double>` `Room`
- [ ] Add getExpensesByCategory function that returns `Flow<List<Expense>>` `Room`
- [ ] Create AppDatabase abstract class annotated with @Database `Room`
- [ ] Register TypeConverters in AppDatabase `Room`
- [ ] Provide AppDatabase as @Singleton in AppModule using @Provides `Hilt`
- [ ] Provide ExpenseDao from AppDatabase in AppModule `Hilt`

---

## Step 5 — DataStore Setup

- [ ] Create a PreferencesKeys object with keys for BUDGET and CURRENCY `DataStore`
- [ ] Create UserPreferencesRepository interface with getMonthlyBudget() and saveBudget() `DataStore`
- [ ] Create UserPreferencesRepositoryImpl that holds a `DataStore<Preferences>` `DataStore`
- [ ] Implement getMonthlyBudget() returning `Flow<Double>` from DataStore `DataStore`
- [ ] Implement saveBudget() using dataStore.edit { } `DataStore`
- [ ] Provide DataStore instance in AppModule using @Provides @Singleton `Hilt`
- [ ] Provide UserPreferencesRepository in AppModule `Hilt`

---

## Step 6 — Repository Layer

- [ ] Create ExpenseRepository interface `Hilt`
- [ ] Add getAllExpenses(), insertExpense(), deleteExpense(), getTotalSpent() to interface `Room`
- [ ] Create ExpenseRepositoryImpl that takes ExpenseDao via @Inject constructor `Hilt`
- [ ] Implement all functions by delegating to the DAO `Room`
- [ ] Provide ExpenseRepository in AppModule binding ExpenseRepositoryImpl `Hilt`

---

## Step 7 — Navigation Setup

- [ ] Create Route sealed interface with Dashboard, AddExpense, Transactions, Stats, Settings objects `UI`
- [ ] Create NavRoot composable with NavHost `UI`
- [ ] Add bottom navigation bar with 4 tabs (Dashboard, Transactions, Stats, Settings) `UI`
- [ ] Add each screen as an empty composable in NavHost `UI`
- [ ] Test navigation between all 4 tabs works `UI`

---

## Step 8 — Dashboard Screen

- [ ] Create DashboardUiState with totalSpent, monthlyBudget, recentExpenses fields `UI`
- [ ] Create DashboardViewModel annotated with @HiltViewModel `Hilt`
- [ ] Inject ExpenseRepository and UserPreferencesRepository in DashboardViewModel `Hilt`
- [ ] Combine totalSpent flow and budget flow into a single UiState flow `UI`
- [ ] Create DashboardScreen composable that receives state and callbacks `UI`
- [ ] Add a summary card showing spent / budget / remaining `UI`
- [ ] Add a LazyColumn showing last 5 recent expenses `UI`
- [ ] Add a FAB button that navigates to AddExpense screen `UI`

---

## Step 9 — Add Expense Screen

- [ ] Create AddExpenseViewModel annotated with @HiltViewModel `Hilt`
- [ ] Inject ExpenseRepository in AddExpenseViewModel `Hilt`
- [ ] Create AddExpenseScreen composable with amount TextField `UI`
- [ ] Add note TextField to the form `UI`
- [ ] Add category selector (row of chips for each Category) `UI`
- [ ] Add Save button that calls viewModel.addExpense() `UI`
- [ ] Navigate back to Dashboard after saving `UI`

---

## Step 10 — Transactions Screen

- [ ] Create TransactionsViewModel annotated with @HiltViewModel `Hilt`
- [ ] Add selectedCategory state in ViewModel (null = show all) `UI`
- [ ] Create TransactionsScreen with a LazyColumn of all expenses `UI`
- [ ] Add category filter chips row at the top `UI`
- [ ] Add swipe-to-delete or delete icon on each expense item `UI`

---

## Step 11 — Settings Screen

- [ ] Create SettingsViewModel annotated with @HiltViewModel `Hilt`
- [ ] Inject UserPreferencesRepository in SettingsViewModel `Hilt`
- [ ] Collect budget flow in ViewModel and expose it as state `DataStore`
- [ ] Create SettingsScreen with a TextField to enter monthly budget `UI`
- [ ] Add Save button that calls viewModel.saveBudget(amount) `DataStore`
- [ ] Add a toggle for enabling/disabling budget notifications `DataStore`
- [ ] Save notification toggle preference in DataStore `DataStore`

---

## Step 12 — Stats Screen

- [ ] Add a DAO query that returns total spent grouped by category `Room`
- [ ] Create StatsViewModel annotated with @HiltViewModel `Hilt`
- [ ] Create StatsScreen composable that receives category totals `UI`
- [ ] Draw a simple bar chart using Canvas in Compose `UI`
- [ ] Show each category name and amount below its bar `UI`

---

## Step 13 — WorkManager Notification

- [ ] Create a notification channel in MyApp onCreate() `WorkManager`
- [ ] Add POST_NOTIFICATIONS permission in AndroidManifest.xml `WorkManager`
- [ ] Create BudgetCheckWorker class that extends CoroutineWorker `WorkManager`
- [ ] Inject ExpenseDao and DataStore into the Worker via HiltWorker `Hilt`
- [ ] Inside doWork() read totalSpent and monthlyBudget `WorkManager`
- [ ] If spent >= 80% of budget send a notification `WorkManager`
- [ ] Schedule BudgetCheckWorker as PeriodicWorkRequest every 24 hours `WorkManager`
- [ ] Enqueue the worker in MyApp onCreate() with KEEP existing policy `WorkManager`