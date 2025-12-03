// app/build.gradle
plugins {
    id 'com.android.application'
    id 'kotlin-android'
    id 'kotlin-kapt'
}

android {
    compileSdk 34

    defaultConfig {
        applicationId "com.example.attendanceapp"
        minSdk 21
        targetSdk 34
        versionCode 1
        versionName "1.0"
    }

    buildTypes {
        release {
            minifyEnabled false
            proguardFiles getDefaultProguardFile('proguard-android-optimize.txt'), 'proguard-rules.pro'
        }
    }

    compileOptions {
        sourceCompatibility JavaVersion.VERSION_17
        targetCompatibility JavaVersion.VERSION_17
    }
    kotlinOptions {
        jvmTarget = "17"
    }
}

dependencies {
    implementation "org.jetbrains.kotlin:kotlin-stdlib:1.9.0"
    implementation 'androidx.core:core-ktx:1.11.0'
    implementation 'androidx.appcompat:appcompat:1.6.1'
    implementation 'com.google.android.material:material:1.9.0'
    implementation 'androidx.constraintlayout:constraintlayout:2.1.4'
    implementation 'androidx.recyclerview:recyclerview:1.3.0'

    // Room
    implementation "androidx.room:room-runtime:2.6.1"
    kapt "androidx.room:room-compiler:2.6.1"
    implementation "androidx.room:room-ktx:2.6.1"

    // lifecycle
    implementation "androidx.lifecycle:lifecycle-runtime-ktx:2.6.1"

    // Optional: coroutines (used here)
    implementation 'org.jetbrains.kotlinx:kotlinx-coroutines-android:1.7.3'
}
<?xml version="1.0" encoding="utf-8"?>
<manifest package="com.example.attendanceapp"
    xmlns:android="http://schemas.android.com/apk/res/android">

    <application
        android:allowBackup="true"
        android:label="AttendanceApp"
        android:theme="@style/Theme.AttendanceApp">
        <activity android:name=".EditStudentActivity" />
        <activity android:name=".MentorActivity" />
        <activity android:name=".StudentListActivity" />
        <activity android:name=".LoginActivity">
            <intent-filter>
                <action android:name="android.intent.action.MAIN"/>
                <category android:name="android.intent.category.LAUNCHER"/>
            </intent-filter>
        </activity>
    </application>

</manifest>
// Student.kt
package com.example.attendanceapp

import androidx.room.Entity
import androidx.room.PrimaryKey

@Entity(tableName = "students")
data class Student(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    var studentId: String,
    var name: String,
    var present: Boolean = false
)
StudentDao.kt
package com.example.attendanceapp

import androidx.room.*
import kotlinx.coroutines.flow.Flow

@Dao
interface StudentDao {
    @Query("SELECT * FROM students ORDER BY id ASC")
    fun getAll(): Flow<List<Student>>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(student: Student): Long

    @Update
    suspend fun update(student: Student)

    @Delete
    suspend fun delete(student: Student)

    @Query("DELETE FROM students")
    suspend fun deleteAll()
}
// AppDatabase.kt
package com.example.attendanceapp

import android.content.Context
import androidx.room.Database
import androidx.room.Room
import androidx.room.RoomDatabase
import androidx.sqlite.db.SupportSQLiteDatabase
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.launch

@Database(entities = [Student::class], version = 1)
abstract class AppDatabase : RoomDatabase() {
    abstract fun studentDao(): StudentDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase =
            INSTANCE ?: synchronized(this) {
                INSTANCE ?: buildDatabase(context).also { INSTANCE = it }
            }

        private fun buildDatabase(context: Context): AppDatabase {
            return Room.databaseBuilder(context.applicationContext, AppDatabase::class.java, "attendance-db")
                .addCallback(object : Callback() {
                    override fun onCreate(db: SupportSQLiteDatabase) {
                        super.onCreate(db)
                        // pre-populate on first create
                        CoroutineScope(Dispatchers.IO).launch {
                            val dao = getInstance(context).studentDao()
                            val demo = listOf(
                                Student(studentId = "S001", name = "Ananya Sharma"),
                                Student(studentId = "S002", name = "Rahul Kumar"),
                                Student(studentId = "S003", name = "Priya Singh"),
                                Student(studentId = "S004", name = "Amit Gupta"),
                                Student(studentId = "S005", name = "Riya Patel")
                            )
                            demo.forEach { dao.insert(it) }
                        }
                    }
                })
                .fallbackToDestructiveMigration()
                .build()
        }
    }
}
LoginActivity.kt
package com.example.attendanceapp

import android.content.Intent
import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import android.widget.Toast
import androidx.appcompat.app.AppCompatActivity

class LoginActivity : AppCompatActivity() {

    // demo credentials
    private val leaderUser = "leader"
    private val leaderPass = "leader123"
    private val mgmtUser = "admin"
    private val mgmtPass = "admin123"

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_login)

        val etUser = findViewById<EditText>(R.id.etUser)
        val etPass = findViewById<EditText>(R.id.etPass)
        val btnLogin = findViewById<Button>(R.id.btnLogin)

        btnLogin.setOnClickListener {
            val u = etUser.text.toString().trim()
            val p = etPass.text.toString().trim()
            when {
                u == leaderUser && p == leaderPass -> {
                    val i = Intent(this, StudentListActivity::class.java)
                    i.putExtra("role", "leader")
                    startActivity(i)
                }
                u == mgmtUser && p == mgmtPass -> {
                    val i = Intent(this, MentorActivity::class.java)
                    startActivity(i)
                }
                else -> {
                    Toast.makeText(this, "Invalid credentials (demo: leader/leader123 or admin/admin123)", Toast.LENGTH_LONG).show()
                }
            }
        }
    }
}
// StudentListActivity.kt
package com.example.attendanceapp

import android.os.Bundle
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch
import java.text.SimpleDateFormat
import java.util.*

class StudentListActivity : AppCompatActivity() {

    private lateinit var adapter: StudentAdapter
    private lateinit var dao: StudentDao

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_student_list)

        dao = AppDatabase.getInstance(this).studentDao()

        val tvDate = findViewById<TextView>(R.id.tvDate)
        val sdf = SimpleDateFormat("EEEE, dd MMM yyyy", Locale.getDefault())
        tvDate.text = sdf.format(Date())

        val rv = findViewById<RecyclerView>(R.id.recyclerStudents)
        rv.layoutManager = LinearLayoutManager(this)
        adapter = StudentAdapter { student, action ->
            // action: "present" or "absent"
            lifecycleScope.launch {
                student.present = action == "present"
                dao.update(student)
            }
        }
        rv.adapter = adapter

        lifecycleScope.launch {
            dao.getAll().collectLatest { list ->
                adapter.submitList(list)
            }
        }
    }
}
// MentorActivity.kt
package com.example.attendanceapp

import android.content.Intent
import android.os.Bundle
import android.widget.Button
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import androidx.recyclerview.widget.LinearLayoutManager
import androidx.recyclerview.widget.RecyclerView
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch

class MentorActivity : AppCompatActivity() {
    private lateinit var dao: StudentDao
    private lateinit var adapter: MentorAdapter

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_mentor)

        dao = AppDatabase.getInstance(this).studentDao()

        val btnEdit = findViewById<Button>(R.id.btnEditTop)
        btnEdit.setOnClickListener {
            // For demo: open add-student / quick-edit screen
            val i = Intent(this, EditStudentActivity::class.java)
            startActivity(i)
        }

        val rv = findViewById<RecyclerView>(R.id.recyclerMentor)
        rv.layoutManager = LinearLayoutManager(this)
        adapter = MentorAdapter { student ->
            // open edit
            val i = Intent(this, EditStudentActivity::class.java)
            i.putExtra("edit_id", student.id)
            startActivity(i)
        }
        rv.adapter = adapter

        lifecycleScope.launch {
            dao.getAll().collectLatest { list ->
                adapter.submitList(list)
            }
        }
    }
}
// EditStudentActivity.kt
package com.example.attendanceapp

import android.os.Bundle
import android.widget.Button
import android.widget.EditText
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.lifecycleScope
import kotlinx.coroutines.launch

class EditStudentActivity : AppCompatActivity() {
    private lateinit var dao: StudentDao
    private var editId: Long = 0L

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_edit_student)

        dao = AppDatabase.getInstance(this).studentDao()
        val etName = findViewById<EditText>(R.id.etName)
        val etSid = findViewById<EditText>(R.id.etSid)
        val btnSave = findViewById<Button>(R.id.btnSave)

        editId = intent.getLongExtra("edit_id", 0L)
        if (editId != 0L) {
            lifecycleScope.launch {
                dao.getAll().collect { list ->
                    val s = list.find { it.id == editId }
                    if (s != null) {
                        runOnUiThread {
                            etName.setText(s.name)
                            etSid.setText(s.studentId)
                        }
                    }
                }
            }
        }

        btnSave.setOnClickListener {
            val name = etName.text.toString().trim()
            val sid = etSid.text.toString().trim()
            if (name.isEmpty() || sid.isEmpty()) {
                return@setOnClickListener
            }
            lifecycleScope.launch {
                if (editId == 0L) {
                    dao.insert(Student(studentId = sid, name = name))
                } else {
                    // update
                    val list = dao.getAll()
                    // simple approach: fetch, modify first match
                    val students = dao.getAll() // but getAll returns Flow -> simpler: update by creating Student with id
                    // Instead query Flow then update when found:
                    dao.getAll().collect { ls ->
                        val st = ls.find { it.id == editId }
                        if (st != null) {
                            st.name = name
                            st.studentId = sid
                            dao.update(st)
                        }
                    }
                }
                runOnUiThread { finish() }
            }
        }
    }
}
// StudentAdapter.kt
package com.example.attendanceapp

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.Button
import android.widget.TextView
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView

class StudentAdapter(
    private val onAction: (Student, String) -> Unit
) : ListAdapter<Student, StudentAdapter.VH>(DIFF) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val v = LayoutInflater.from(parent.context).inflate(R.layout.item_student, parent, false)
        return VH(v)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        holder.bind(getItem(position))
    }

    inner class VH(view: View) : RecyclerView.ViewHolder(view) {
        private val tvName: TextView = view.findViewById(R.id.tvName)
        private val tvSid: TextView = view.findViewById(R.id.tvSid)
        private val btnPresent: Button = view.findViewById(R.id.btnPresent)
        private val btnAbsent: Button = view.findViewById(R.id.btnAbsent)

        fun bind(s: Student) {
            tvName.text = s.name
            tvSid.text = s.studentId
            updateButtons(s.present)
            btnPresent.setOnClickListener { onAction(s, "present"); updateButtons(true) }
            btnAbsent.setOnClickListener { onAction(s, "absent"); updateButtons(false) }
        }

        private fun updateButtons(isPresent: Boolean) {
            btnPresent.isEnabled = !isPresent
            btnAbsent.isEnabled = isPresent
        }
    }

    companion object {
        val DIFF = object : DiffUtil.ItemCallback<Student>() {
            override fun areItemsTheSame(oldItem: Student, newItem: Student) = oldItem.id == newItem.id
            override fun areContentsTheSame(oldItem: Student, newItem: Student) = oldItem == newItem
        }
    }
}
// MentorAdapter.kt
package com.example.attendanceapp

import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.Button
import android.widget.TextView
import androidx.recyclerview.widget.DiffUtil
import androidx.recyclerview.widget.ListAdapter
import androidx.recyclerview.widget.RecyclerView

class MentorAdapter(private val onEdit: (Student) -> Unit) : ListAdapter<Student, MentorAdapter.VH>(DIFF) {
    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): VH {
        val v = LayoutInflater.from(parent.context).inflate(R.layout.item_mentor, parent, false)
        return VH(v)
    }

    override fun onBindViewHolder(holder: VH, position: Int) {
        holder.bind(getItem(position))
    }

    inner class VH(view: View) : RecyclerView.ViewHolder(view) {
        private val tvName: TextView = view.findViewById(R.id.tvNameM)
        private val tvSid: TextView = view.findViewById(R.id.tvSidM)
        private val tvPresent: TextView = view.findViewById(R.id.tvPresentM)
        private val btnEdit: Button = view.findViewById(R.id.btnEditM)

        fun bind(s: Student) {
            tvName.text = s.name
            tvSid.text = s.studentId
            tvPresent.text = if (s.present) "Present" else "Absent"
            btnEdit.setOnClickListener { onEdit(s) }
        }
    }

    companion object {
        val DIFF = object : DiffUtil.ItemCallback<Student>() {
            override fun areItemsTheSame(oldItem: Student, newItem: Student) = oldItem.id == newItem.id
            override fun areContentsTheSame(oldItem: Student, newItem: Student) = oldItem == newItem
        }
    }
}
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent" android:layout_height="match_parent"
    android:padding="24dp">

    <TextView android:id="@+id/tvTitle"
        android:layout_width="0dp" android:layout_height="wrap_content"
        android:text="Attendance App - Login" android:textSize="20sp"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent"
        app:layout_constraintEnd_toEndOf="parent"/>

    <EditText android:id="@+id/etUser" android:layout_width="0dp" android:layout_height="wrap_content"
        android:hint="Username" app:layout_constraintTop_toBottomOf="@id/tvTitle" app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toEndOf="parent" android:layout_marginTop="16dp"/>

    <EditText android:id="@+id/etPass" android:layout_width="0dp" android:layout_height="wrap_content"
        android:hint="Password" android:inputType="textPassword"
        app:layout_constraintTop_toBottomOf="@id/etUser" app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toEndOf="parent" android:layout_marginTop="8dp"/>

    <Button android:id="@+id/btnLogin" android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:text="Login" app:layout_constraintTop_toBottomOf="@id/etPass" app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toEndOf="parent" android:layout_marginTop="16dp"/>

    <TextView android:id="@+id/tvHint" android:layout_width="0dp" android:layout_height="wrap_content"
        android:text="Demo accounts: leader/leader123, admin/admin123" android:textSize="12sp"
        app:layout_constraintTop_toBottomOf="@id/btnLogin" app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toEndOf="parent" android:layout_marginTop="12dp"/>

</androidx.constraintlayout.widget.ConstraintLayout>
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout
    xmlns:android="http://schemas.android.com/apk/res/android" xmlns:app="http://schemas.android.com/apk/res-auto"
    android:layout_width="match_parent" android:layout_height="match_parent" android:padding="12dp">

    <TextView android:id="@+id/tvDate" android:layout_width="0dp" android:layout_height="wrap_content"
        android:text="Date" android:textSize="16sp"
        app:layout_constraintTop_toTopOf="parent" app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toEndOf="parent"/>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerStudents"
        android:layout_width="0dp" android:layout_height="0dp"
        app:layout_constraintTop_toBottomOf="@id/tvDate" app:layout_constraintBottom_toBottomOf="parent"
        app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toEndOf="parent" android:layout_marginTop="8dp"/>

</androidx.constraintlayout.widget.ConstraintLayout>
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto" android:layout_width="match_parent" android:layout_height="wrap_content" android:padding="10dp">

    <TextView android:id="@+id/tvName" android:layout_width="0dp" android:layout_height="wrap_content"
        android:text="Name" android:textSize="16sp" app:layout_constraintTop_toTopOf="parent" app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toStartOf="@id/btnPresent" />

    <TextView android:id="@+id/tvSid" android:layout_width="0dp" android:layout_height="wrap_content" android:text="S001" android:textSize="12sp"
        app:layout_constraintTop_toBottomOf="@id/tvName" app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toStartOf="@id/btnPresent" />

    <Button android:id="@+id/btnPresent" android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:text="Present" app:layout_constraintTop_toTopOf="parent" app:layout_constraintEnd_toStartOf="@id/btnAbsent" app:layout_constraintBottom_toBottomOf="parent"/>

    <Button android:id="@+id/btnAbsent" android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:text="Absent" app:layout_constraintTop_toTopOf="parent" app:layout_constraintEnd_toEndOf="parent" app:layout_constraintBottom_toBottomOf="parent"/>

</androidx.constraintlayout.widget.ConstraintLayout>
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto" android:layout_width="match_parent" android:layout_height="match_parent" android:padding="12dp">

    <Button android:id="@+id/btnEditTop" android:layout_width="wrap_content" android:layout_height="wrap_content"
        android:text="Add Student" app:layout_constraintTop_toTopOf="parent" app:layout_constraintEnd_toEndOf="parent"/>

    <androidx.recyclerview.widget.RecyclerView
        android:id="@+id/recyclerMentor"
        android:layout_width="0dp" android:layout_height="0dp"
        app:layout_constraintTop_toBottomOf="@id/btnEditTop" app:layout_constraintStart_toStartOf="parent" app:layout_constraintEnd_toEndOf="parent" app:layout_constraintBottom_toBottomOf="parent" android:layout_marginTop="12dp"/>

</androidx.constraintlayout.widget.ConstraintLayout>
<?xml version="1.0" encoding="utf-8"?>
<androidx.constraintlayout.widget.ConstraintLayout xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto" android:layout_width="match_parent" android:layout_height="wrap_content" android:padding="10dp">

    <TextView android:id="@+id/tvSidM" android:layout_width="wrap_content" android:layout_height="wrap_content" android:text="S001" app:layout_constraintStart_toStartOf="parent" app:layout_constraintTop_toTopOf="parent"/>

    <TextView android:id="@+id/tvNameM" android:layout_width="0dp" android:layout_height="wrap_content" android:text="Name" app:layout_constraintStart_toEndOf="@id/tvSidM" app:layout_constraintTop_toTopOf="parent" app:layout_constraintEnd_toStartOf="@id/tvPresentM"/>

    <TextView android:id="@+id/tvPresentM" android:layout_width="wrap_content" android:layout_height="wrap_content" android:text="Present" app:layout_constraintTop_toTopOf="parent" app:layout_constraintEnd_toStartOf="@id/btnEditM"/>

    <Button android:id="@+id/btnEditM" android:layout_width="wrap_content" android:layout_height="wrap_content" android:text="Edit" app:layout_constraintEnd_toEndOf="parent" app:layout_constraintTop_toTopOf="parent"/>

</androidx.constraintlayout.widget.ConstraintLayout>
<!-- res/values/themes.xml -->
<resources xmlns:tools="http://schemas.android.com/tools">
    <style name="Theme.AttendanceApp" parent="Theme.MaterialComponents.DayNight.DarkActionBar">
        <item name="colorPrimary">@color/purple_500</item>
        <item name="colorPrimaryVariant">@color/purple_700</item>
        <item name="colorOnPrimary">@android:color/white</item>
    </style>
</resources>
