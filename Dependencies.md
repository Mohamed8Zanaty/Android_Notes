```kotlin

dependencies {  
  
	// Compose BOM or explicit
	implementation(platform("androidx.compose:compose-bom:2025.05.00")) 
	implementation("androidx.compose.ui:ui")
	implementation("androidx.compose.material:material")
	implementation("androidx.compose.ui:ui-graphics")
	implementation("androidx.compose.foundation:foundation")
	// etc for other compose modules
	
	// Navigation (Compose)
	implementation("androidx.navigation:navigation-compose:2.7.0") 
	
	// Lifecycle / ViewModel
	implementation("androidx.lifecycle:lifecycle-viewmodel-compose:2.6.0")
	implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.6.0")
	
	// Room (local data / SQLite)
	implementation("androidx.room:room-runtime:2.6.0")
	ksp("androidx.room:room-compiler:2.6.0") 
	implementation("androidx.room:room-ktx:2.6.0")
	
	// Retrofit (network)
	implementation("com.squareup.retrofit2:retrofit:2.12.0")
	implementation("com.squareup.retrofit2:converter-moshi:2.12.0") 
	implementation("com.squareup.okhttp3:okhttp:4.12.0") 
	implementation("com.squareup.okhttp3:logging-interceptor:4.12.0")
	
	// Coroutines / Flow
	implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.7.0")
	implementation("org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.0")
	
	// (Optional) JSON serialization (Moshi / kotlinx-serialization / Gson) as needed
	implementation("com.squareup.moshi:moshi:1.15.0")
	ksp("com.squareup.moshi:moshi-kotlin-codegen:1.15.0")
}
```

