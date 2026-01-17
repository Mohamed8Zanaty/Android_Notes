
## Download Screen
```kotlin
package com.example.test.ui.theme.screen  
  
import android.Manifest  
import android.content.pm.PackageManager  
import android.net.Uri  
import android.os.Build  
import android.widget.Toast  
import androidx.activity.compose.rememberLauncherForActivityResult  
import androidx.activity.result.contract.ActivityResultContracts  
import androidx.compose.foundation.layout.*  
import androidx.compose.material3.*  
import androidx.compose.runtime.*  
import androidx.compose.ui.Modifier  
import androidx.compose.ui.platform.LocalContext  
import androidx.compose.ui.text.input.TextFieldValue  
import androidx.compose.ui.unit.dp  
import androidx.core.content.ContextCompat  
import androidx.lifecycle.viewmodel.compose.viewModel  
import com.example.test.DownloadViewModel  
import androidx.core.net.toUri  
  
@OptIn(ExperimentalMaterial3Api::class)  
@Composable  
fun DownloadScreen(viewModel: DownloadViewModel = viewModel()) {  
    val state by viewModel.state  
    var url by remember { mutableStateOf(TextFieldValue("")) }  
    val context = LocalContext.current  
    var pendingUri by remember { mutableStateOf<Uri?>(null) }  
    // for storage permission  
    val permissionLauncher = rememberLauncherForActivityResult(  
        ActivityResultContracts.RequestPermission()  
    ) { granted ->  
        if (granted) {  
            pendingUri?.let {  
                viewModel.startDownload(context, it)  
                pendingUri = null  
            }  
        } else {  
            Toast.makeText(context, "Permission denied. Can't save to public downloads.", Toast.LENGTH_SHORT).show()  
            pendingUri = null  
        }  
    }  
    Scaffold(  
        topBar = { TopAppBar(title = { Text("Mini IDM Downloader") }) }  
    ) { padding ->  
        Column(  
            modifier = Modifier  
                .padding(padding)  
                .padding(16.dp)  
        ) {  
            OutlinedTextField(  
                value = url,  
                onValueChange = { url = it },  
                label = { Text("File URL") },  
                modifier = Modifier.fillMaxWidth()  
            )  
  
            Spacer(modifier = Modifier.height(16.dp))  
  
            if (!state.isDownloading && !state.isCompleted) {  
                Button(  
                    onClick = {  
                        if(Build.VERSION.SDK_INT < Build.VERSION_CODES.Q) {  
                            val permission = Manifest.permission.WRITE_EXTERNAL_STORAGE  
                            if (ContextCompat.checkSelfPermission(context, permission) == PackageManager.PERMISSION_GRANTED) {  
                                viewModel.startDownload(context, url.text.toUri())  
                            } else {  
                                pendingUri = url.text.toUri()  
                                permissionLauncher.launch(permission)  
                            }  
                        } else {  
                            // Modern Android: just enqueue  
                            viewModel.startDownload(context, url.text.toUri())  
                        }  
                        },  
                    modifier = Modifier.fillMaxWidth()  
                ) {  
                    Text("Start Download")  
                }  
            }  
  
            if (state.isDownloading) {  
                LinearProgressIndicator(  
                    progress = state.progress / 100f,  
                    modifier = Modifier.fillMaxWidth()  
                )  
                Spacer(modifier = Modifier.height(8.dp))  
                Text("Downloading: ${state.progress}%")  
                Spacer(modifier = Modifier.height(8.dp))  
                Button(  
                    onClick = { viewModel.cancelDownload() },  
                    modifier = Modifier.fillMaxWidth(),  
                    colors = ButtonDefaults.buttonColors(containerColor = MaterialTheme.colorScheme.error)  
                ) {  
                    Text("Cancel Download")  
                }  
            }  
  
            if (state.isCompleted) {  
                Text(  
                    text = state.message,  
                    style = MaterialTheme.typography.bodyLarge,  
                    color = MaterialTheme.colorScheme.primary  
                )  
                Spacer(modifier = Modifier.height(8.dp))  
                Button(onClick = {  
                    url = TextFieldValue("")  
                    // reset state manually  
                    viewModel.resetDownload()  
                }) {  
                    Text("Download Another File")  
                }  
            }  
  
            Spacer(modifier = Modifier.height(16.dp))  
            Text(text = state.message)  
        }  
    }}
```

## Permissions
```xml
<uses-permission android:name="android.permission.INTERNET"/>  
  
<uses-permission android:name="android.permission.WRITE_EXTERNAL_STORAGE" android:maxSdkVersion="28" />
```

## Download State
```kotlin
data class DownloadState(  
    val progress: Int = 0,  
    val isDownloading: Boolean = false,  
    val isCompleted: Boolean = false,  
    val message: String = ""  
)
```

## Download View Model

```kotlin
package com.example.test  
  
import android.app.DownloadManager  
import android.content.Context  
import android.net.Uri  
import android.widget.Toast  
import androidx.compose.runtime.mutableStateOf  
import androidx.lifecycle.ViewModel  
import androidx.lifecycle.viewModelScope  
import kotlinx.coroutines.Dispatchers  
import kotlinx.coroutines.Job  
import kotlinx.coroutines.delay  
import kotlinx.coroutines.launch  
import kotlinx.coroutines.withContext  
import java.sql.Time  
import java.util.Date  
  
class DownloadViewModel : ViewModel() {  
  
    val state = mutableStateOf(DownloadState(isDownloading = false, message = ""))  
    private var currentDownloadId: Long? = null  
  
    private var downloadJob: Job? = null  
  
    fun startDownload(context: Context, url: Uri?) {  
        if (url == null) {  
            state.value = state.value.copy(message = "Please enter a valid URL")  
            return  
        }  
        val appCtx = context.applicationContext  
        // if there working download, cancel it  
        downloadJob?.cancel()  
  
        downloadJob = viewModelScope.launch(Dispatchers.IO) {  
            state.value = DownloadState(isDownloading = true, message = "Starting download...")  
            val dm = appCtx.getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager  
            val request = DownloadManager.Request(url)  
                .setTitle(System.currentTimeMillis().toString())  
                .setDescription("Downloading file...")  
                .setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED)  
                .setAllowedOverMetered(true)  
                .setAllowedOverRoaming(true)  
  
            val id = dm.enqueue(request)  
            withContext(Dispatchers.Main) {  
                Toast.makeText(appCtx, "Download started (id = $id)", Toast.LENGTH_SHORT).show()  
                state.value = DownloadState(isDownloading = true, message = "Download started")  
                for (i in 1..100) {  
                    delay(10) // Download Speed  
                    state.value = state.value.copy(progress = i)  
                }  
            }  
        }    }  
  
    fun cancelDownload() {  
        downloadJob?.cancel()  
        state.value = state.value.copy(  
            isDownloading = false,  
            message = "Download canceled"  
        )  
    }  
  
    fun resetDownload() {  
        state.value = DownloadState() // Reset everything to normal  
    }  
}
```

## Main Activity
```kotlin
package com.example.test  
  
import android.os.Bundle  
import androidx.activity.ComponentActivity  
import androidx.activity.compose.setContent  
import androidx.activity.enableEdgeToEdge  
import androidx.compose.foundation.layout.fillMaxSize  
import androidx.compose.foundation.layout.padding  
import androidx.compose.material3.Scaffold  
import androidx.compose.material3.Text  
import androidx.compose.runtime.Composable  
import androidx.compose.ui.Modifier  
import androidx.compose.ui.tooling.preview.Preview  
import com.example.test.ui.theme.TestTheme  
import com.example.test.ui.theme.screen.DownloadScreen  
  
class MainActivity : ComponentActivity() {  
    override fun onCreate(savedInstanceState: Bundle?) {  
        super.onCreate(savedInstanceState)  
        enableEdgeToEdge()  
        setContent {  
            TestTheme {  
                DownloadScreen()  
            }  
        }    }  
}
```
