## What is Firebase Auth?

Firebase Authentication is a service by Google that handles user identity for your app.
It manages sign up, sign in, session persistence, and sign out — without you building any backend.

---

## How it Works

```
Your App
    ↓ sends email + password
Firebase Auth (Google's servers)
    ↓ verifies credentials
    ↓ returns FirebaseUser object
Your App
    ↓ stores auth state automatically
    ↓ user stays logged in between app restarts
```

---

## Core Concepts

### FirebaseAuth
The main entry point. One instance for the whole app.
```kotlin
val auth = FirebaseAuth.getInstance()
```

### FirebaseUser
The currently signed-in user. Null if no one is signed in.
```kotlin
val user: FirebaseUser? = auth.currentUser
// user?.uid       → unique user ID
// user?.email     → email address
// user?.displayName → display name
```

### Auth State
Firebase automatically persists the auth state.
If the user signed in yesterday and opens the app today — they are still signed in.

```kotlin
// Listen to auth state changes
auth.addAuthStateListener { firebaseAuth ->
    val user = firebaseAuth.currentUser
    if (user != null) {
        // user is signed in
    } else {
        // user is signed out
    }
}
```

---

## Step 1 — Create Firebase Project

1. Go to https://console.firebase.google.com
2. Click **Add project**
3. Enter project name → Continue
4. Disable Google Analytics (not needed) → Create project
5. Click **Android** icon to add an Android app
6. Enter your package name (e.g. `com.example.lokaal`)
7. Download `google-services.json`
8. Place it in your `app/` folder

```
your-project/
└── app/
    ├── google-services.json   ← goes here
    └── src/
```

---

## Step 2 — Enable Email/Password Auth

1. In Firebase Console → **Authentication**
2. Click **Get started**
3. Click **Email/Password**
4. Toggle **Enable** → Save

---

## Step 3 — Add Dependencies

In `libs.versions.toml`:
```toml
[versions]
googleServices = "4.4.2"
firebaseBom = "33.1.0"

[libraries]
firebase-bom = { group = "com.google.firebase", name = "firebase-bom", version.ref = "firebaseBom" }
firebase-auth = { group = "com.google.firebase", name = "firebase-auth-ktx" }
firebase-firestore = { group = "com.google.firebase", name = "firebase-firestore-ktx" }

[plugins]
google-services = { id = "com.google.gms.google-services", version.ref = "googleServices" }
```

In `build.gradle.kts` (project level):
```kotlin
plugins {
    alias(libs.plugins.google.services) apply false
}
```

In `build.gradle.kts` (app level):
```kotlin
plugins {
    alias(libs.plugins.google.services)
}

dependencies {
    implementation(platform(libs.firebase.bom))
    implementation(libs.firebase.auth)
    implementation(libs.firebase.firestore)
}
```

---

## Step 4 — Core Auth Operations

### Sign Up
```kotlin
suspend fun signUp(email: String, password: String): FirebaseUser {
    val result = auth.createUserWithEmailAndPassword(email, password).await()
    return result.user!!
}
```

### Sign In
```kotlin
suspend fun signIn(email: String, password: String): FirebaseUser {
    val result = auth.signInWithEmailAndPassword(email, password).await()
    return result.user!!
}
```

### Sign Out
```kotlin
fun signOut() {
    auth.signOut()
}
```

### Check if Signed In
```kotlin
val isSignedIn: Boolean = auth.currentUser != null
val currentUserId: String? = auth.currentUser?.uid
```

### Observe Auth State as Flow
```kotlin
fun observeAuthState(): Flow<FirebaseUser?> = callbackFlow {
    val listener = FirebaseAuth.AuthStateListener { auth ->
        trySend(auth.currentUser)
    }
    auth.addAuthStateListener(listener)
    awaitClose { auth.removeAuthStateListener(listener) }
}
```

---

## Step 5 — Error Handling

Firebase throws `FirebaseAuthException` with specific error codes:

```kotlin
try {
    auth.signInWithEmailAndPassword(email, password).await()
} catch (e: FirebaseAuthInvalidCredentialsException) {
    // wrong password
} catch (e: FirebaseAuthInvalidUserException) {
    // no account with this email
} catch (e: FirebaseAuthUserCollisionException) {
    // email already registered
} catch (e: FirebaseAuthWeakPasswordException) {
    // password too short (min 6 chars)
} catch (e: Exception) {
    // any other error
}
```

### Map to User-Friendly Messages
```kotlin
fun Exception.toAuthMessage(): String {
    return when (this) {
        is FirebaseAuthInvalidCredentialsException -> "Incorrect password"
        is FirebaseAuthInvalidUserException -> "No account found with this email"
        is FirebaseAuthUserCollisionException -> "Email already in use"
        is FirebaseAuthWeakPasswordException -> "Password must be at least 6 characters"
        else -> "Something went wrong. Try again."
    }
}
```

---

## Step 6 — Auth Guard (Protecting Screens)

Check auth state before showing any screen:

```kotlin
// In NavRoot
val authState by authViewModel.authState.collectAsStateWithLifecycle()

LaunchedEffect(authState) {
    when (authState) {
        is AuthState.Unauthenticated -> navigator.navigate(Route.Auth)
        is AuthState.Authenticated -> navigator.navigate(Route.Feed)
        else -> { }
    }
}
```

---

## Step 7 — Provide Firebase in Hilt

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object FirebaseModule {

    @Provides
    @Singleton
    fun provideFirebaseAuth(): FirebaseAuth {
        return FirebaseAuth.getInstance()
    }

    @Provides
    @Singleton
    fun provideFirebaseFirestore(): FirebaseFirestore {
        return FirebaseFirestore.getInstance()
    }
}
```

---

## Auth Repository Pattern

Always wrap Firebase behind a repository:

```kotlin
// Interface — ViewModel only knows about this
interface AuthRepository {
    suspend fun signUp(email: String, password: String): Result<FirebaseUser>
    suspend fun signIn(email: String, password: String): Result<FirebaseUser>
    fun signOut()
    fun getCurrentUser(): FirebaseUser?
    fun observeAuthState(): Flow<FirebaseUser?>
}

// Implementation — Firebase details hidden here
class AuthRepositoryImpl @Inject constructor(
    private val auth: FirebaseAuth
) : AuthRepository {

    override suspend fun signUp(email: String, password: String): Result<FirebaseUser> {
        return try {
            val result = auth.createUserWithEmailAndPassword(email, password).await()
            Result.success(result.user!!)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    override suspend fun signIn(email: String, password: String): Result<FirebaseUser> {
        return try {
            val result = auth.signInWithEmailAndPassword(email, password).await()
            Result.success(result.user!!)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    override fun signOut() = auth.signOut()

    override fun getCurrentUser(): FirebaseUser? = auth.currentUser

    override fun observeAuthState(): Flow<FirebaseUser?> = callbackFlow {
        val listener = FirebaseAuth.AuthStateListener { trySend(it.currentUser) }
        auth.addAuthStateListener(listener)
        awaitClose { auth.removeAuthStateListener(listener) }
    }
}
```

---

## Validation Rules

Always validate before calling Firebase:

```kotlin
fun validateEmail(email: String): String? {
    if (email.isBlank()) return "Email is required"
    if (!Patterns.EMAIL_ADDRESS.matcher(email).matches()) return "Invalid email format"
    return null
}

fun validatePassword(password: String): String? {
    if (password.isBlank()) return "Password is required"
    if (password.length < 6) return "Password must be at least 6 characters"
    return null
}

fun validateConfirmPassword(password: String, confirm: String): String? {
    if (confirm != password) return "Passwords do not match"
    return null
}
```

---

## `Result<T>` Pattern

Use Kotlin's `Result<T>` to wrap success and failure cleanly:

```kotlin
// In Repository — returns Result
override suspend fun signIn(email: String, password: String): Result<FirebaseUser> {
    return try {
        val result = auth.signInWithEmailAndPassword(email, password).await()
        Result.success(result.user!!)
    } catch (e: Exception) {
        Result.failure(e)
    }
}

// In ViewModel — handles Result
fun signIn(email: String, password: String) {
    viewModelScope.launch {
        _uiState.value = AuthUiState.Loading
        val result = repository.signIn(email, password)
        _uiState.value = result.fold(
            onSuccess = { AuthUiState.Success },
            onFailure = { AuthUiState.Error(it.toAuthMessage()) }
        )
    }
}
```

---

## Security Rules (Firestore)

Set these in Firebase Console → Firestore → Rules:

```javascript
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    // Users can only read/write their own data
    match /users/{userId} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }

    // Anyone signed in can read moments, only owner can write
    match /moments/{momentId} {
      allow read: if request.auth != null;
      allow create: if request.auth != null;
      allow update, delete: if request.auth != null
        && request.auth.uid == resource.data.userId;
    }
  }
}
```

---

## Common Mistakes

| Mistake | Fix |
|---|---|
| Calling Firebase on main thread | Always use `suspend` + `.await()` |
| Not handling auth state on app start | Check `currentUser` in a splash or NavRoot |
| Storing user data only in Firebase Auth | Also create a Firestore user document on sign up |
| Not validating before calling Firebase | Always validate email/password locally first |
| Catching only `Exception` | Catch specific `FirebaseAuthException` subtypes for better messages |

---

## Summary

```
Firebase Console  →  create project, enable Email/Password auth
google-services.json  →  add to app/ folder
Gradle deps  →  firebase-bom + firebase-auth
FirebaseModule  →  provide FirebaseAuth via Hilt
AuthRepository  →  wrap Firebase, return Result<T>
AuthViewModel  →  call repository, emit UiState
NavRoot  →  observe auth state, guard routes
```
