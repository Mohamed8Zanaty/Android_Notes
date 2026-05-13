### 📍 Build **"Lokaal"** — Location-Based Social Moments App

Think of it as **Instagram meets Google Maps** — users capture moments, pin them to real locations, and discover what's happening nearby.

---

### 🧠 The Idea in One Sentence

> A user signs in, takes a photo, adds a caption, pins it to their current location, and other users nearby can see it on a map and in a feed.

---

### 📱 The 6 Screens

|Screen|What it does|
|---|---|
|🔐 **Auth Screen**|Sign up / Sign in with email|
|🗺️ **Map Screen**|See all moments pinned on a live map|
|📜 **Feed Screen**|Infinite scroll of all moments|
|📸 **Camera Screen**|Take a photo using CameraX|
|✏️ **Create Moment Screen**|Add caption + confirm location before posting|
|👤 **Profile Screen**|Your own moments + sign out|

---

### 🛠️ What Each New Topic Teaches You

#### 🔥 Firebase Auth

kotlin

```kotlin
// Sign up / sign in / sign out
FirebaseAuth.getInstance().createUserWithEmailAndPassword(email, password)
FirebaseAuth.getInstance().signInWithEmailAndPassword(email, password)
```

**What you learn:** Auth state listening, protecting routes, persisting login

---

#### 🔥 Firestore — Real-time Database

kotlin

```kotlin
// Moments stored as documents
firestore.collection("moments")
    .orderBy("timestamp", Query.Direction.DESCENDING)
    .get()
```

**What you learn:** Collections, documents, real-time listeners, queries, security rules

---

#### 📄 Paging 3 — Infinite Scroll Feed

kotlin

```kotlin
// Only loads 10 moments at a time, fetches more as user scrolls
class MomentsPagingSource : PagingSource<QuerySnapshot, Moment>() {
    override suspend fun load(params: LoadParams<QuerySnapshot>): LoadResult<...>
}
```

**What you learn:** `PagingSource`, `Pager`, `LazyPagingItems`, load states

---

#### 📸 CameraX — Photo Capture

kotlin

```kotlin
// Control camera, capture image, save to file
val imageCapture = ImageCapture.Builder().build()
imageCapture.takePicture(outputOptions, executor, callback)
```

**What you learn:** Camera lifecycle, `ImageCapture`, permissions, file handling

---

#### 🗺️ Maps SDK — Location Pins

kotlin

```kotlin
// Show moments as markers on a real map
GoogleMap(
    onMapLoaded = { },
    cameraPositionState = cameraPositionState
) {
    moments.forEach { Marker(state = MarkerState(it.location)) }
}
```

**What you learn:** Maps Compose, markers, camera position, location permissions

---

#### 🧪 Unit Testing

kotlin

```kotlin
// Test ViewModel logic without UI
@Test
fun `when auth fails, uiState should be Error`() = runTest {
    val fakeRepo = FakeAuthRepository(shouldFail = true)
    val viewModel = AuthViewModel(fakeRepo)
    viewModel.signIn("test@test.com", "123456")
    assertIs<AuthUiState.Error>(viewModel.uiState.value)
}
```

**What you learn:** `runTest`, fake repositories, `assertIs`, testing StateFlow

---

### 🔁 How the Tech Fits Together

```
Firebase Auth
    → protects all screens
    → provides userId for Firestore docs

CameraX
    → captures photo → saved to local file

Create Moment Screen
    → uploads photo to Firebase Storage
    → saves moment doc to Firestore
        { userId, caption, photoUrl, location, timestamp }

Firestore
    → Feed Screen reads via Paging 3 (paginated)
    → Map Screen reads all nearby moments

Maps SDK
    → shows moment pins on map
    → tapping a pin opens the moment detail

Unit Tests
    → test AuthViewModel, FeedViewModel
    → use fake repositories
```

---

### ✅ Full Requirements List

```
Auth
[ ] Sign up screen with email + password
[ ] Sign in screen
[ ] Auth state persists between app launches
[ ] Sign out from Profile screen
[ ] Redirect to Auth if not signed in

Camera
[ ] Request camera permission
[ ] Live camera preview using CameraX
[ ] Capture photo and save to local file
[ ] Show captured photo preview before posting

Create Moment
[ ] Show captured photo
[ ] Add caption text field
[ ] Auto-detect current location
[ ] Post → upload photo to Firebase Storage
[ ] Post → save moment to Firestore

Feed
[ ] Infinite scroll using Paging 3
[ ] Show photo, caption, author, location name, time
[ ] Loading + error + empty states
[ ] Pull to refresh

Map
[ ] Show Google Map centered on user location
[ ] Show all moments as markers
[ ] Tap marker → show moment preview card

Profile
[ ] Show current user's moments in a grid
[ ] Show display name + email
[ ] Sign out button

Unit Tests
[ ] AuthViewModel sign in success + failure
[ ] FeedViewModel loading states
[ ] Fake repository for testing
```

---

### 🪜 Order to Build It

|Step|What to build|New skill|
|---|---|---|
|**1**|Firebase project setup + Auth screens|Firebase Auth|
|**2**|Navigation with auth guard|Navigation 3|
|**3**|CameraX screen + photo capture|CameraX|
|**4**|Firestore model + upload moment|Firestore|
|**5**|Feed screen with Paging 3|Paging 3|
|**6**|Map screen with markers|Maps SDK|
|**7**|Profile screen|Firestore queries|
|**8**|Unit tests|Testing|

---

### 🔥 Why This Project is Worth It

- **Firebase** is used in almost every real startup — Auth + Firestore is industry-standard knowledge
- **Paging 3** is the correct way to handle large lists — you'll never load everything at once again
- **CameraX** opens the door to any app involving photos, QR codes, or AR
- **Maps SDK** makes you capable of building delivery, travel, or location-aware apps
- **Unit Testing** makes you a professional developer — testable code = better architecture
- The app is **genuinely impressive** for a portfolio — it has real-time data, a map, camera, and auth

---

Ready to start with **Step 1 — Firebase setup + Auth screens**?

  

Sonnet 4.6

Adaptive

[Claude is AI an](https://support.anthropic.com/en/articles/8525154-claude-is-providing-incorrect-or-misleading-responses-what-s-going-on)