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
import android.database.Cursor  
import android.net.Uri  
import android.widget.Toast  
import androidx.compose.runtime.mutableStateOf  
import androidx.lifecycle.ViewModel  
import androidx.lifecycle.viewModelScope  
import kotlinx.coroutines.Dispatchers  
import kotlinx.coroutines.Job  
import kotlinx.coroutines.delay  
import kotlinx.coroutines.isActive  
import kotlinx.coroutines.launch  
import kotlinx.coroutines.withContext  
  
class DownloadViewModel : ViewModel() {  
  
    val state = mutableStateOf(DownloadState(isDownloading = false, message = ""))  
    private var currentDownloadId: Long? = null  
    private var downloadMonitorJob: Job? = null  
  
    fun startDownload(context: Context, url: Uri?) {  
        if (url == null) {  
            state.value = state.value.copy(message = "Please enter a valid URL")  
            return  
        }  
        val appCtx = context.applicationContext  
  
        // Cancel any existing download monitoring job  
        downloadMonitorJob?.cancel()  
  
        viewModelScope.launch(Dispatchers.IO) {  
            state.value = DownloadState(isDownloading = true, message = "Starting download...")  
            val dm = appCtx.getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager  
            val request = DownloadManager.Request(url)  
                .setTitle(url.lastPathSegment ?: "Unknown File")  
                .setDescription("Downloading file...")  
                .setNotificationVisibility(DownloadManager.Request.VISIBILITY_VISIBLE_NOTIFY_COMPLETED)  
                .setAllowedOverMetered(true)  
                .setAllowedOverRoaming(true)  
                .setDestinationInExternalPublicDir(android.os.Environment.DIRECTORY_DOWNLOADS, url.lastPathSegment ?: "downloaded_file")  
  
            currentDownloadId = dm.enqueue(request)  
            withContext(Dispatchers.Main) {  
                Toast.makeText(appCtx, "Download started (id = $currentDownloadId)", Toast.LENGTH_SHORT).show()  
                state.value = state.value.copy(isDownloading = true, message = "Download initiated...")  
            }  
            monitorDownloadProgress(appCtx, dm, currentDownloadId!!)  
        }  
    }  
  
    private fun monitorDownloadProgress(context: Context, dm: DownloadManager, downloadId: Long) {  
        downloadMonitorJob = viewModelScope.launch(Dispatchers.IO) {  
            var finishing = false  
            while (isActive && !finishing) {  
                val query = DownloadManager.Query().setFilterById(downloadId)  
                var cursor: Cursor? = null  
                try {  
                    cursor = dm.query(query)  
                    if (cursor != null && cursor.moveToFirst()) {  
                        val statusIndex = cursor.getColumnIndex(DownloadManager.COLUMN_STATUS)  
                        val bytesDownloadedIndex = cursor.getColumnIndex(DownloadManager.COLUMN_BYTES_DOWNLOADED_SO_FAR)  
                        val bytesTotalIndex = cursor.getColumnIndex(DownloadManager.COLUMN_TOTAL_SIZE_BYTES)  
                        val reasonIndex = cursor.getColumnIndex(DownloadManager.COLUMN_REASON)  
  
                        val status = cursor.getInt(statusIndex)  
                        val bytesDownloaded = cursor.getLong(bytesDownloadedIndex)  
                        val bytesTotal = cursor.getLong(bytesTotalIndex)  
                        val reason = cursor.getInt(reasonIndex)  
  
                        withContext(Dispatchers.Main) {  
                            when (status) {  
                                DownloadManager.STATUS_PENDING -> {  
                                    state.value = state.value.copy(message = "Download pending...")  
                                }  
                                DownloadManager.STATUS_RUNNING -> {  
                                    if (bytesTotal > 0) {  
                                        val progress = ((bytesDownloaded * 100) / bytesTotal).toInt()  
                                        state.value = state.value.copy(progress = progress, message = "Downloading: $progress%")  
                                    } else {  
                                        state.value = state.value.copy(message = "Downloading...")  
                                    }  
                                }  
                                DownloadManager.STATUS_PAUSED -> {  
                                    state.value = state.value.copy(message = "Download paused.")  
                                }  
                                DownloadManager.STATUS_SUCCESSFUL -> {  
                                    state.value = state.value.copy(isDownloading = false, isCompleted = true, message = "Download completed!")  
                                    finishing = true  
                                }  
                                DownloadManager.STATUS_FAILED -> {  
                                    state.value = state.value.copy(isDownloading = false, isCompleted = false, message = "Download failed! Reason: $reason")  
                                    finishing = true  
                                }  
                            }  
                        }  
                    } else {  
                        // Download not found or completed.  
                        finishing = true  
                    }  
                } catch (e: Exception) {  
                    withContext(Dispatchers.Main) {  
                        state.value = state.value.copy(isDownloading = false, isCompleted = false, message = "Error monitoring download: ${e.message}")  
                    }  
                    finishing = true  
                } finally {  
                    cursor?.close()  
                }  
                delay(1000) // Update every second  
            }  
        }  
    }  
  
    fun cancelDownload() {  
        currentDownloadId?.let { id ->  
            val dm = Class.forName("android.app.ActivityThread").getMethod("currentApplication").invoke(null) as Context  
            (dm.getSystemService(Context.DOWNLOAD_SERVICE) as DownloadManager).remove(id)  
            currentDownloadId = null  
        }  
        downloadMonitorJob?.cancel()  
        state.value = state.value.copy(  
            isDownloading = false,  
            message = "Download canceled"  
        )  
    }  
  
    fun resetDownload() {  
        state.value = DownloadState() // Reset everything to normal  
        currentDownloadId = null  
        downloadMonitorJob?.cancel()  
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