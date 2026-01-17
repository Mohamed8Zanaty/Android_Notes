
>[!info]+ Material
>## All Materials Links
> - [Database](https://www.youtube.com/playlist?list=PL93xoMrxRJIuicqcd1UpFUYMfWKGp7JmI)
> -  [SQLite](https://www.youtube.com/playlist?list=PLJJcOjd3n1ZfwZKYMu_JSs0xSejPsQlav)

## DB Helper
```Kotlin
package com.example.depistorage

import android.content.ContentValues
import android.content.Context
import android.database.sqlite.SQLiteDatabase
import android.database.sqlite.SQLiteOpenHelper

class TasksDbHelper(ctx: Context) : SQLiteOpenHelper(ctx, DB_NAME, null, DB_VERSION) {

    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL("""
            CREATE TABLE $TABLE_TASKS (
                $COL_ID INTEGER PRIMARY KEY AUTOINCREMENT,
                $COL_TITLE TEXT NOT NULL,
                $COL_DONE INTEGER NOT NULL DEFAULT 0,
                $COL_CREATED_AT INTEGER NOT NULL
            )
        """.trimIndent())
    }

    override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
        db.execSQL("DROP TABLE IF EXISTS $TABLE_TASKS")
        onCreate(db)
    }

    fun insertTask(title: String): Long {
        val cv = ContentValues().apply {
            put(COL_TITLE, title)
            put(COL_DONE, 0)
            put(COL_CREATED_AT, System.currentTimeMillis())
        }
        return writableDatabase.insertOrThrow(TABLE_TASKS, null, cv)
    }
    fun updateNote(note: Note): Int {
        val db = this.writableDatabase
        val values = ContentValues().apply {
            put(COLUMN_TITLE, note.title)
            put(COLUMN_CONTENT, note.content)
            put(COLUMN_UPDATED_AT, System.currentTimeMillis()) // Update timestamp
	    }
	

        // Updating row with the provided ID
        return db.update(
            TABLE_NOTES,
            values,
            "$COLUMN_ID = ?",
            arrayOf(note.id.toString())
        )
    }

    fun getAllTasks(): List<Task> {
        val tasks = mutableListOf<Task>()
        val c = readableDatabase.query(
            TABLE_TASKS,
            arrayOf(COL_ID, COL_TITLE, COL_DONE, COL_CREATED_AT),
            null, null, null, null,
            "$COL_CREATED_AT DESC"
        )
        c.use {
            while (it.moveToNext()) {
                val id = it.getLong(0)
                val title = it.getString(1)
                val done = it.getInt(2) == 1
                val createdAt = it.getLong(3)
                tasks += Task(id, title, done, createdAt)
            }
        }
        return tasks
    }

    fun setDone(id: Long, done: Boolean): Int {
        val cv = ContentValues().apply { put(COL_DONE, if (done) 1 else 0) }
        return writableDatabase.update(TABLE_TASKS, cv, "$COL_ID=?", arrayOf(id.toString()))
    }

    fun deleteTask(id: Long): Int {
        return writableDatabase.delete(TABLE_TASKS, "$COL_ID=?", arrayOf(id.toString()))
    }

    companion object {
        const val DB_NAME = "tasks.db"
        const val DB_VERSION = 1

        const val TABLE_TASKS = "tasks"
        const val COL_ID = "_id"
        const val COL_TITLE = "title"
        const val COL_DONE = "done"
        const val COL_CREATED_AT = "createdAt"
    }
}
```

## Data Class
```kotlin
package com.example.depistorage

data class Task(
    val id: Long = 0,
    val title: String,
    val done: Boolean = false,
    val createdAt: Long = System.currentTimeMillis()
)
```

## Task Adapter
```kotlin
package com.example.depistorage

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.CheckBox
import android.widget.TextView
import androidx.recyclerview.widget.RecyclerView

class TaskAdapter(
    private val tasks: MutableList<Task>,
    private val onToggleDone: (Task) -> Unit,
    private val onDelete: (Task) -> Unit
) : RecyclerView.Adapter<TaskAdapter.TaskViewHolder>() {

    inner class TaskViewHolder(itemView: View) : RecyclerView.ViewHolder(itemView) {
        val cbDone: CheckBox = itemView.findViewById(R.id.cbDone)
        val tvTitle: TextView = itemView.findViewById(R.id.tvTitle)
    }

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): TaskViewHolder {
        val v = LayoutInflater.from(parent.context).inflate(R.layout.item_task, parent, false)
        return TaskViewHolder(v)
    }

    override fun onBindViewHolder(holder: TaskViewHolder, position: Int) {
        val task = tasks[position]
        holder.tvTitle.text = task.title
        holder.cbDone.isChecked = task.done

        holder.cbDone.setOnClickListener {
            onToggleDone(task)
        }

        holder.itemView.setOnLongClickListener {
            onDelete(task)
            true
        }
    }

    override fun getItemCount(): Int = tasks.size

    fun updateData(newTasks: List<Task>) {
        tasks.clear()
        tasks.addAll(newTasks)
        notifyDataSetChanged()
    }
}
```

## Main Activity
```kotlin
package com.example.depistorage

import android.os.Bundle
import android.widget.EditText
import android.widget.Toast
import androidx.appcompat.app.AlertDialog
import androidx.appcompat.app.AppCompatActivity
import androidx.core.view.ViewCompat
import androidx.core.view.WindowInsetsCompat
import androidx.recyclerview.widget.LinearLayoutManager
import com.example.depistorage.databinding.ActivityTaskDbSqliteBinding

class TaskDbSQLite : AppCompatActivity() {

    private lateinit var binding: ActivityTaskDbSqliteBinding
    private lateinit var db: TasksDbHelper
    private lateinit var adapter: TaskAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        binding = ActivityTaskDbSqliteBinding.inflate(layoutInflater)
        setContentView(binding.root)

        db = TasksDbHelper(this)


        ViewCompat.setOnApplyWindowInsetsListener(findViewById(R.id.main)) { v, insets ->
            val systemBars = insets.getInsets(WindowInsetsCompat.Type.systemBars())
            v.setPadding(systemBars.left, systemBars.top, systemBars.right, systemBars.bottom)
            insets
        }

        adapter = TaskAdapter(mutableListOf(), ::toggleDone, ::deleteTask)
        binding.recyclerView.layoutManager = LinearLayoutManager(this)
        binding.recyclerView.adapter = adapter

        binding.fabAdd.setOnClickListener { showAddDialog() }

        loadTasks()
    }

    private fun loadTasks() {
        val tasks = db.getAllTasks()
        adapter.updateData(tasks)
    }

    private fun showAddDialog() {
        val et = EditText(this)
        AlertDialog.Builder(this)
            .setTitle("New Task")
            .setView(et)
            .setPositiveButton("Add") { _, _ ->
                val title = et.text.toString()
                if (title.isNotBlank()) {
                    db.insertTask(title)
                    loadTasks()
                } else {
                    Toast.makeText(this, "Title can't be empty", Toast.LENGTH_SHORT).show()
                }
            }
            .setNegativeButton("Cancel", null)
            .show()
    }

    private fun toggleDone(task: Task) {
        db.setDone(task.id, !task.done)
        loadTasks() // refresh
    }

    private fun deleteTask(task: Task) {
        db.deleteTask(task.id)
        loadTasks() // refresh
    }
}
```


## Main.xml
```xml
<?xml version="1.0" encoding="utf-8"?>
<RelativeLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    android:id="@+id/main"
    android:orientation="vertical"
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    tools:context=".TaskDbSQLite">

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerView"
        android:layout_width="match_parent"
        android:layout_height="match_parent"
        android:padding="8dp"/>

    <com.google.android.material.floatingactionbutton.FloatingActionButton
        android:id="@+id/fabAdd"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:contentDescription="Add"
        android:src="@android:drawable/ic_input_add"
        android:layout_alignParentBottom="true"
        android:layout_alignParentEnd="true"
        android:layout_margin="16dp"/>

</RelativeLayout>
```

## Item
```xml
<?xml version="1.0" encoding="utf-8"?>
<LinearLayout xmlns:android="http://schemas.android.com/apk/res/android"
    android:orientation="horizontal"
    android:padding="12dp"
    android:layout_width="match_parent"
    android:layout_height="wrap_content">

    <CheckBox
        android:id="@+id/cbDone"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content" />

    <TextView
        android:id="@+id/tvTitle"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Sample Task"
        android:textSize="16sp"
        android:layout_marginStart="8dp"/>
</LinearLayout>
 
```
---





