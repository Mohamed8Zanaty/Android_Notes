# 🚀 The "All-in-One" Compose Navigation Template


```kotlin
import androidx.compose.foundation.layout.*
import androidx.compose.material3.*
import androidx.compose.runtime.*
import androidx.compose.ui.Alignment
import androidx.compose.ui.Modifier
import androidx.compose.ui.unit.dp
import androidx.navigation.compose.NavHost
import androidx.navigation.compose.composable
import androidx.navigation.compose.rememberNavController
import androidx.navigation.toRoute
import kotlinx.serialization.Serializable

// 1. Define Destinations (Type-Safe)
@Serializable object HomeScreenRoute
@Serializable data class DetailsScreenRoute(val itemId: Int, val itemName: String)

@Composable
fun MainAppNavigation() {
    val navController = rememberNavController()

    // 2. The NavHost: The Bridge between routes and UI
    NavHost(
        navController = navController,
        startDestination = HomeScreenRoute
    ) {
        // Home Destination
        composable<HomeScreenRoute> {
            HomeScreen(
                onItemClick = { id, name ->
                    navController.navigate(DetailsScreenRoute(itemId = id, itemName = name))
                }
            )
        }

        // Details Destination (Extracting arguments automatically)
        composable<DetailsScreenRoute> { backStackEntry ->
            val args = backStackEntry.toRoute<DetailsScreenRoute>()
            DetailsScreen(
                id = args.itemId,
                name = args.itemName,
                onBack = { navController.popBackStack() }
            )
        }
    }
}

// 3. Stateless UI Components (State Hoisting)
@Composable
fun HomeScreen(onItemClick: (Int, String) -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Main Dashboard", style = MaterialTheme.typography.headlineMedium)
        Spacer(modifier = Modifier.height(16.dp))
        Button(onClick = { onItemClick(42, "Cyber-Widget") }) {
            Text("Go to Item #42")
        }
    }
}

@Composable
fun DetailsScreen(id: Int, name: String, onBack: () -> Unit) {
    Column(
        modifier = Modifier.fillMaxSize(),
        verticalArrangement = Arrangement.Center,
        horizontalAlignment = Alignment.CenterHorizontally
    ) {
        Text("Details for: $name", style = MaterialTheme.typography.headlineSmall)
        Text("ID: $id", style = MaterialTheme.typography.bodyLarge)
        Spacer(modifier = Modifier.height(16.dp))
        Button(onClick = onBack) {
            Text("Back to Home")
        }
    }
}
```