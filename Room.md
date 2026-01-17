## Dependencies

- ### libs.version.toml
```toml
[Versions]

room = "2.7.2"

[libraries]

androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }  
androidx-room-ktx     = { group = "androidx.room", name = "room-ktx",     version.ref = "room" }  
androidx-room-compiler= { group = "androidx.room", name = "room-compiler",  version.ref = "room" }

```

- ### Build.gradel.kts: Project
```kotlin
id("com.google.devtools.ksp") version "2.2.0-2.0.2" apply false
```

- ### Build.gradel.kts: Module:App
 ```kotlin
  plugins {
   id("com.google.devtools.ksp")
}
dependencies {

    implementation("androidx.room:room-runtime:2.7.2")  
	implementation("androidx.room:room-ktx:2.7.2")  
	ksp("androidx.room:room-compiler:2.7.2")
}
```

## Task Entity
```kotlin
import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "task_table")
data class Task(
    @PrimaryKey(autoGenerate = true) val id: Int = 0,
    val title: String,
    val isDone: Boolean = false
)
```

## Task DAO
```kotlin
import androidx.lifecycle.LiveData
import androidx.room.Dao
import androidx.room.Insert
import androidx.room.Query
import androidx.room.Delete

@Dao
interface TaskDao {
    @Insert
	suspend fun insertTask(task: Task)

	@Delete
	suspend fun deleteTask(task: Task)

	@Query("SELECT * FROM task_table ORDER BY id DESC")
	fun getAllTasks(): LiveData<List<Task>>
}
```

## Task Database
```kotlin
import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase

@Database(entities = [Task::class], version = 1, exportSchema = false)
abstract class TaskDatabase : RoomDatabase() {
    abstract fun taskDao(): TaskDao

    companion object {
        @Volatile
        private var INSTANCE: TaskDatabase? = null

        fun getDatabase(context: Context): TaskDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    TaskDatabase::class.java,
                    "task_database"
                ).build()
                INSTANCE = instance
                instance
            }
        }
    }
}

```

## Task Repo
```kotlin
import androidx.lifecycle.LiveData

class TaskRepository(private val taskDao: TaskDao) {
    val allTasks: LiveData<List<Task>> = taskDao.getAllTasks()

    suspend fun insert(task: Task) {
        taskDao.insertTask(task)
    }

    suspend fun delete(task: Task) {
        taskDao.deleteTask(task)
    }
}
```

## Task View Model
```kotlin
import androidx.lifecycle.AndroidViewModel
import androidx.lifecycle.LiveData
import androidx.lifecycle.viewModelScope
import android.app.Application
import kotlinx.coroutines.launch

class TaskViewModel(application: Application) : AndroidViewModel(application) {
    private val repository: TaskRepository
    val allTasks: LiveData<List<Task>>

    init {
        val dao = TaskDatabase.getDatabase(application).taskDao()
        repository = TaskRepository(dao)
        allTasks = repository.allTasks
    }

    fun insert(task: Task) = viewModelScope.launch {
        repository.insert(task)
    }

    fun delete(task: Task) = viewModelScope.launch {
        repository.delete(task)
    }
}
```

## Task Adapter
```kotlin
import android.view.LayoutInflater  
import android.view.View  
import android.view.ViewGroup  
import android.widget.Button  
import android.widget.TextView  
import androidx.recyclerview.widget.RecyclerView  
  
class TaskAdapter(  
    private var tasks: List<Task>,  
    private val onDeleteClick: (Task) -> Unit  
) : RecyclerView.Adapter<TaskAdapter.TaskViewHolder>() {  
  
    class TaskViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {  
        val textViewTask: TextView = itemView.findViewById(R.id.textViewTask)  
        val buttonDelete: Button = itemView.findViewById(R.id.buttonDelete)  
    }  
  
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): TaskViewHolder {  
        val view = LayoutInflater.from(parent.context)  
            .inflate(R.layout.item_task, parent, false)  
        return TaskViewHolder(view)  
    }  
  
    override fun onBindViewHolder(holder: TaskViewHolder, position: Int) {  
        val task = tasks[position]  
        holder.textViewTask.text = task.title  
  
        holder.buttonDelete.setOnClickListener {  
            onDeleteClick(task)  
        }  
    }  
  
    override fun getItemCount(): Int = tasks.size  
  
    fun updateTasks(newTasks: List<Task>) {  
        tasks = newTasks  
        notifyDataSetChanged()  
    }  
}
```

## Main Activity
```kotlin
import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.Toast
import androidx.activity.enableEdgeToEdge
import androidx.activity.viewModels
import androidx.appcompat.app.AppCompatActivity
import androidx.core.view.ViewCompat
import androidx.core.view.WindowInsetsCompat
import androidx.lifecycle.Observer
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView

class MainActivity : AppCompatActivity() {
    private val taskViewModel: TaskViewModel by viewModels()
    private lateinit var adapter: TaskAdapter
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        enableEdgeToEdge()
        setContentView(R.layout.activity_main)
        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main)) { v, insets ->
            val systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom)
            insets
        }

        val editText = findViewById<EditText>(R.id.editTextTask)
        val buttonAdd = findViewById<Button>(R.id.buttonAdd)
        val recyclerView = findViewById<RecyclerView>(R.id.recyclerViewTasks)

        adapter = TaskAdapter(emptyList()) { task ->
            taskViewModel.delete(task)
        }
        recyclerView.layoutManager = LinearLayoutManager(this)
        recyclerView.adapter = adapter

        // observe LiveData
        taskViewModel.allTasks.observe(this, Observer { tasks ->
            adapter.updateTasks(tasks)
        })

        buttonAdd.setOnClickListener {
            val title = editText.text.toString()
            if (title.isNotEmpty()) {
                taskViewModel.insert(Task(title = title))
                editText.text.clear()
            }else{
                Toast.makeText(this,"Task Must Be Filled",Toast.LENGTH_SHORT).show()
            }
        }
    }

}
```

## activity_main.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical"
    android:padding="16dp"
    tools:context=".MainActivity">

    <EditText
        android:id="@+id/editTextTask"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:hint="Enter task"
        android:inputType="text" />

    <Button
        android:id="@+id/buttonAdd"
        android:layout_width="match_parent"
        android:layout_height="wrap_content"
        android:text="Add Task"
        android:layout_marginTop="8dp" />

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerViewTasks"
        android:layout_width="match_parent"
        android:layout_height="0dp"
        android:layout_weight="1"
        android:layout_marginTop="16dp"/>

</LinearLayout>
```

## taskItem.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:layout_width="match_parent"
    android:layout_height="wrap_content"
    android:orientation="horizontal"
    android:padding="12dp">

    <TextView
        android:id="@+id/textViewTask"
        android:layout_width="0dp"
        android:layout_height="wrap_content"
        android:layout_weight="1"
        android:text="Task title"
        android:textSize="18sp" />

    <Button
        android:id="@+id/buttonDelete"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Delete" />
</LinearLayout>
```

