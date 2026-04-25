```toml
# libs.versions.toml
[versions]
nav3                  = "1.1.1"           # stable  
lifecycleNav3         = "2.11.0-beta01"  # latest with Nav3 ViewModel support  
material3AdaptiveNav3 = "1.3.0-alpha10"  
kotlinxSerialCore     = "1.11.0"

[libraries]
# Core (required)
navigation3-runtime  = { module = "androidx.navigation3:navigation3-runtime",           version.ref = "nav3" }
navigation3-ui       = { module = "androidx.navigation3:navigation3-ui",                version.ref = "nav3" }

# ViewModel scoping per back stack entry (recommended)
lifecycle-viewmodel-nav3 = { module = "androidx.lifecycle:lifecycle-viewmodel-navigation3", version.ref = "lifecycleNav3" }

# Adaptive layouts / multi-pane (optional)
material3-adaptive-nav3  = { module = "androidx.compose.material3.adaptive:adaptive-navigation3", version.ref = "material3AdaptiveNav3" }

# Serialization for persistent back stack (recommended)
kotlinx-serialization-core = { module = "org.jetbrains.kotlinx:kotlinx-serialization-core", version.ref = "kotlinxSerialCore" }

[plugins]
kotlin-serialization = { id = "org.jetbrains.kotlin.plugin.serialization", version = "2.3.21" }
```

```kotlin
plugins {
    alias(libs.plugins.kotlin.serialization)
}

dependencies {
    implementation(libs.navigation3.runtime)
    implementation(libs.navigation3.ui)
    implementation(libs.lifecycle.viewmodel.nav3)
    implementation(libs.material3.adaptive.nav3)      // optional
    implementation(libs.kotlinx.serialization.core)
}
```
