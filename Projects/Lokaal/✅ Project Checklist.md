
> Tags: `Firebase Auth` `Firestore` `Paging 3` `CameraX` `Maps SDK` `Unit Testing` `UI` `Navigation`

---

## Step 1 — Project & Firebase Setup

- [x] Create a new Android project in Android Studio
- [x] Create a new Firebase project at console.firebase.google.com `Firebase Auth`
- [x] Add Android app in Firebase Console with your package name `Firebase Auth`
- [x] Download google-services.json and place it in the app/ folder `Firebase Auth`
- [x] Enable Email/Password authentication in Firebase Console `Firebase Auth`
- [x] Add google-services plugin in project-level build.gradle.kts `Firebase Auth`
- [x] Add google-services plugin in app-level build.gradle.kts `Firebase Auth`
- [x] Add Firebase BOM, firebase-auth-ktx, firebase-firestore-ktx dependencies `Firebase Auth`
- [x] Add Hilt, Room, Navigation 3, Coroutines dependencies
	- [x] Sync Gradle and confirm no errors

---

## Step 2 — Hilt Base Setup

- [x] Create MyApp class extending Application
- [x] Annotate MyApp with @HiltAndroidApp
- [x] Register MyApp in AndroidManifest.xml with android:name
- [x] Annotate MainActivity with @AndroidEntryPoint
- [x] Create AppModule annotated with @Module @InstallIn(SingletonComponent::class)
- [x] Create FirebaseModule providing FirebaseAuth and FirebaseFirestore as @Singleton `Firebase Auth`

---

## Step 3 — Auth Repository & ViewModel

- [x] Create AuthRepository interface with signUp, signIn, signOut, getCurrentUser, observeAuthState `Firebase Auth`
- [x] Create AuthRepositoryImpl with @Inject constructor taking FirebaseAuth `Firebase Auth`
- [x] Implement signUp using createUserWithEmailAndPassword().await() `Firebase Auth`
- [x] Implement signIn using signInWithEmailAndPassword().await() `Firebase Auth`
- [x] Implement observeAuthState() returning Flow<FirebaseUser?> using callbackFlow `Firebase Auth`
- [x] Wrap all Firebase calls in try/catch returning `Result<T>` `Firebase Auth`
- [x] Provide AuthRepository in AppModule binding AuthRepositoryImpl `Firebase Auth`
- [x] Create AuthUiState sealed interface with Idle, Loading, Success, Error `Firebase Auth`
- [x] Create AuthViewModel annotated with @HiltViewModel `Firebase Auth`
- [x] Implement signIn() function in ViewModel emitting Loading then Success or Error `Firebase Auth`
- [x] Implement signUp() function in ViewModel emitting Loading then Success or Error `Firebase Auth`
- [x] Create toAuthMessage() extension function mapping Firebase exceptions to user-friendly strings `Firebase Auth`
- [x] Add email and password validation functions in a utils file

---

## Step 4 — Auth Screens

- [ ] Create SignInScreen composable with email and password fields `UI`
- [ ] Add show/hide password toggle on password field `UI`
- [ ] Add Sign In button that calls viewModel.signIn() `UI`
- [ ] Add 'Don't have an account? Sign Up' text button `UI`
- [ ] Create SignUpScreen composable with email, password, confirm password fields `UI`
- [ ] Add Sign Up button that calls viewModel.signUp() `UI`
- [ ] Show inline error messages below each field on validation failure `UI`
- [ ] Show loading indicator while auth is in progress `UI`
- [ ] Show snackbar or error text when Firebase returns an error `UI`

---

## Step 5 — Navigation & Auth Guard

- [ ] Create Route sealed interface with Auth, SignIn, SignUp, Feed, Map, Camera, CreateMoment, Profile `Navigation`
- [ ] Create NavRoot with NavHost and bottom navigation bar `Navigation`
- [ ] Add bottom nav with 3 tabs: Feed, Map, Profile `Navigation`
- [ ] Observe auth state in NavRoot using collectAsStateWithLifecycle `Navigation`
- [ ] Redirect to SignIn screen if user is not authenticated `Navigation`
- [ ] Redirect to Feed screen if user is already authenticated `Navigation`
- [ ] Add FAB on Feed screen navigating to Camera screen `Navigation`

---

## Step 6 — Firestore Data Model

- [ ] Create Moment data class with id, userId, caption, photoUrl, latitude, longitude, locationName, timestamp `Firestore`
- [ ] Create UserProfile data class with uid, displayName, email, momentsCount `Firestore`
- [ ] Create MomentRepository interface with getMoments(), createMoment(), getUserMoments() `Firestore`
- [ ] Create MomentRepositoryImpl with @Inject constructor taking FirebaseFirestore and FirebaseAuth `Firestore`
- [ ] On sign up — create a Firestore user document in users/{uid} `Firestore`
- [ ] Implement createMoment() saving a document to moments/ collection `Firestore`
- [ ] Provide MomentRepository in AppModule `Firestore`
- [ ] Set Firestore security rules — authenticated read, owner-only write `Firestore`

---

## Step 7 — CameraX Screen

- [ ] Add CameraX dependencies in libs.versions.toml `CameraX`
- [ ] Add CAMERA permission in AndroidManifest.xml `CameraX`
- [ ] Request camera permission at runtime before opening camera screen `CameraX`
- [ ] Create CameraScreen composable with a PreviewView using AndroidView `CameraX`
- [ ] Set up CameraX ProcessCameraProvider and bind Preview use case `CameraX`
- [ ] Add ImageCapture use case bound to the same lifecycle `CameraX`
- [ ] Add capture button that calls imageCapture.takePicture() `CameraX`
- [ ] Save captured photo to a temp file in app cache directory `CameraX`
- [ ] Navigate to CreateMoment screen passing the photo file path `CameraX`
- [ ] Show camera permission denied message if permission rejected `CameraX`

---

## Step 8 — Create Moment Screen

- [ ] Create CreateMomentScreen composable receiving photo file path `UI`
- [ ] Display the captured photo using AsyncImage (Coil) `UI`
- [ ] Add caption text field `UI`
- [ ] Add location permission request `Maps SDK`
- [ ] Get current location using FusedLocationProviderClient `Maps SDK`
- [ ] Show current location name as subtitle below the photo `Maps SDK`
- [ ] Upload photo file to Firebase Storage and get download URL `Firestore`
- [ ] Save moment document to Firestore with photoUrl and location `Firestore`
- [ ] Show loading state while uploading `UI`
- [ ] Navigate back to Feed after successful post `Navigation`

---

## Step 9 — Paging 3 Feed Screen

- [ ] Add Paging 3 dependency in libs.versions.toml `Paging 3`
- [ ] Create MomentsPagingSource extending PagingSource<QuerySnapshot, Moment> `Paging 3`
- [ ] Implement load() fetching 10 moments at a time from Firestore `Paging 3`
- [ ] Use Firestore document snapshot as the next page key `Paging 3`
- [ ] Create Pager with PagingConfig(pageSize=10) in MomentRepositoryImpl `Paging 3`
- [ ] Expose Flow<PagingData<Moment>> from the repository `Paging 3`
- [ ] Collect paging data in FeedViewModel using .cachedIn(viewModelScope) `Paging 3`
- [ ] Create FeedScreen using LazyColumn with collectAsLazyPagingItems() `Paging 3`
- [ ] Add loading footer item when more pages are loading `Paging 3`
- [ ] Add error item with retry button when page load fails `Paging 3`
- [ ] Create MomentCard composable showing photo, caption, location, time, author `UI`

---

## Step 10 — Map Screen

- [ ] Add Maps Compose dependency in libs.versions.toml `Maps SDK`
- [ ] Get a Google Maps API key from Google Cloud Console `Maps SDK`
- [ ] Add API key in local.properties and reference in AndroidManifest.xml `Maps SDK`
- [ ] Add ACCESS_FINE_LOCATION and ACCESS_COARSE_LOCATION permissions in manifest `Maps SDK`
- [ ] Request location permission at runtime on Map screen `Maps SDK`
- [ ] Create MapScreen composable with GoogleMap from Maps Compose `Maps SDK`
- [ ] Fetch all moments from Firestore in MapViewModel `Maps SDK`
- [ ] Place a Marker for each moment at its lat/lng coordinates `Maps SDK`
- [ ] Center camera on user's current location on first load `Maps SDK`
- [ ] Show moment preview card at bottom when user taps a marker `Maps SDK`

---

## Step 11 — Profile Screen

- [ ] Create ProfileScreen composable showing display name and email `UI`
- [ ] Fetch current user's moments from Firestore using userId filter `Firestore`
- [ ] Display user's moments in a 3-column LazyVerticalGrid `UI`
- [ ] Show moments count in the profile header `UI`
- [ ] Add Sign Out button that calls authViewModel.signOut() `Firebase Auth`
- [ ] Navigate to SignIn screen after sign out `Navigation`

---

## Step 12 — Unit Tests

- [ ] Add JUnit, Coroutines Test, and Turbine dependencies `Unit Testing`
- [ ] Create FakeAuthRepository implementing AuthRepository with configurable success/failure `Unit Testing`
- [ ] Create FakeMomentRepository implementing MomentRepository `Unit Testing`
- [ ] Write test: signIn success → AuthUiState becomes Success `Unit Testing`
- [ ] Write test: signIn failure → AuthUiState becomes Error with message `Unit Testing`
- [ ] Write test: signUp with invalid email → validation error, Firebase not called `Unit Testing`
- [ ] Write test: signUp with short password → validation error `Unit Testing`
- [ ] Write test: FeedViewModel initial state is Loading `Unit Testing`
- [ ] Write test: createMoment success → navigateBack emits true `Unit Testing`
- [ ] Run all tests and confirm they pass `Unit Testing`