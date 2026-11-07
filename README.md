name: Build SchoolBell APK

on:
  push:
    branches: [ "main", "master" ]
  workflow_dispatch:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - name: Set up JDK 17
        uses: actions/setup-java@v4
        with:
          java-version: '17'
          distribution: 'temurin'

      - name: Setup Android SDK
        uses: android-actions/setup-android@v3

      - name: Setup Gradle
        uses: gradle/actions/setup-gradle@v4
        with:
          gradle-version: '8.9'

      - name: Generate SchoolBell Project Structure
        run: |
          mkdir -p app/src/main/java/com/schoolbell/app/data
          mkdir -p app/src/main/java/com/schoolbell/app/notifications
          mkdir -p app/src/main/java/com/schoolbell/app/scanner
          mkdir -p app/src/main/java/com/schoolbell/app/ui/screens
          mkdir -p app/src/main/java/com/schoolbell/app/ui/theme
          mkdir -p app/src/main/res/drawable
          mkdir -p app/src/main/res/values
          mkdir -p app/src/main/assets

          # 1. settings.gradle.kts
          cat << 'EOF' > settings.gradle.kts
          pluginManagement {
              repositories {
                  google()
                  mavenCentral()
                  gradlePluginPortal()
              }
          }
          dependencyResolutionManagement {
              repositoriesMode.set(RepositoriesMode.FAIL_ON_PROJECT_REPOS)
              repositories {
                  google()
                  mavenCentral()
              }
          }
          rootProject.name = "SchoolBell"
          include(":app")
          EOF

          # 2. build.gradle.kts (root)
          cat << 'EOF' > build.gradle.kts
          plugins {
              id("com.android.application") version "8.7.2" apply false
              id("org.jetbrains.kotlin.android") version "2.0.21" apply false
              id("org.jetbrains.kotlin.plugin.compose") version "2.0.21" apply false
              id("org.jetbrains.kotlin.plugin.serialization") version "2.0.21" apply false
          }
          EOF

          # 3. app/build.gradle.kts
          cat << 'EOF' > app/build.gradle.kts
          plugins {
              id("com.android.application")
              id("org.jetbrains.kotlin.android")
              id("org.jetbrains.kotlin.plugin.compose")
              id("org.jetbrains.kotlin.plugin.serialization")
          }

          android {
              namespace = "com.schoolbell.app"
              compileSdk = 35

              defaultConfig {
                  applicationId = "com.schoolbell.app"
                  minSdk = 26
                  targetSdk = 35
                  versionCode = 1
                  versionName = "1.0.0"
                  vectorDrawables.useSupportLibrary = true
              }

              buildTypes {
                  release {
                      isMinifyEnabled = false
                      signingConfig = signingConfigs.getByName("debug")
                  }
                  debug {
                      isDebuggable = true
                  }
              }

              compileOptions {
                  sourceCompatibility = JavaVersion.VERSION_17
                  targetCompatibility = JavaVersion.VERSION_17
              }

              kotlinOptions {
                  jvmTarget = "17"
              }

              buildFeatures {
                  compose = true
              }
          }

          dependencies {
              implementation("androidx.core:core-ktx:1.15.0")
              implementation("androidx.lifecycle:lifecycle-runtime-ktx:2.8.7")
              implementation("androidx.activity:activity-compose:1.9.3")

              val composeBom = platform("androidx.compose:compose-bom:2024.11.00")
              implementation(composeBom)
              implementation("androidx.compose.ui:ui")
              implementation("androidx.compose.ui:ui-graphics")
              implementation("androidx.compose.material3:material3")
              implementation("androidx.compose.material:material-icons-extended")

              implementation("androidx.datastore:datastore-preferences:1.1.1")
              implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.7.3")

              val cameraxVersion = "1.4.1"
              implementation("androidx.camera:camera-core:$cameraxVersion")
              implementation("androidx.camera:camera-camera2:$cameraxVersion")
              implementation("androidx.camera:camera-lifecycle:$cameraxVersion")
              implementation("androidx.camera:camera-view:$cameraxVersion")

              implementation("com.google.mlkit:text-recognition:16.0.1")
              implementation("com.google.guava:guava:33.3.1-android")
          }
          EOF

          # 4. AndroidManifest.xml
          cat << 'EOF' > app/src/main/AndroidManifest.xml
          <?xml version="1.0" encoding="utf-8"?>
          <manifest xmlns:android="http://schemas.android.com/apk/res/android">
              <uses-permission android:name="android.permission.CAMERA" />
              <uses-feature android:name="android.hardware.camera" android:required="false" />
              <uses-permission android:name="android.permission.POST_NOTIFICATIONS" />
              <uses-permission android:name="android.permission.SCHEDULE_EXACT_ALARM" />
              <uses-permission android:name="android.permission.USE_EXACT_ALARM" />
              <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED" />
              <uses-permission android:name="android.permission.VIBRATE" />
              <uses-permission android:name="android.permission.WAKE_LOCK" />
              <uses-permission android:name="android.permission.REQUEST_IGNORE_BATTERY_OPTIMIZATIONS" />

              <application
                  android:name=".SchoolBellApplication"
                  android:allowBackup="true"
                  android:icon="@drawable/ic_bell_clock"
                  android:label="@string/app_name"
                  android:roundIcon="@drawable/ic_bell_clock"
                  android:supportsRtl="true"
                  android:theme="@style/Theme.SchoolBell">

                  <activity
                      android:name=".MainActivity"
                      android:exported="true"
                      android:launchMode="singleTop"
                      android:theme="@style/Theme.SchoolBell">
                      <intent-filter>
                          <action android:name="android.intent.action.MAIN" />
                          <category android:name="android.intent.category.LAUNCHER" />
                      </intent-filter>
                  </activity>

                  <receiver
                      android:name=".notifications.AlarmReceiver"
                      android:directBootAware="true"
                      android:enabled="true"
                      android:exported="false" />

                  <receiver
                      android:name=".notifications.BootReceiver"
                      android:enabled="true"
                      android:exported="true">
                      <intent-filter>
                          <action android:name="android.intent.action.BOOT_COMPLETED" />
                          <action android:name="android.intent.action.LOCKED_BOOT_COMPLETED" />
                          <action android:name="android.intent.action.MY_PACKAGE_REPLACED" />
                          <action android:name="android.intent.action.TIME_SET" />
                          <action android:name="android.intent.action.TIMEZONE_CHANGED" />
                      </intent-filter>
                  </receiver>
              </application>
          </manifest>
          EOF

          # 5. Resources: Strings & Themes
          cat << 'EOF' > app/src/main/res/values/strings.xml
          <resources>
              <string name="app_name">SchoolBell</string>
          </resources>
          EOF

          cat << 'EOF' > app/src/main/res/values/themes.xml
          <resources>
              <style name="Theme.SchoolBell" parent="android:Theme.Material.Light.NoActionBar">
                  <item name="android:statusBarColor">#071A2D</item>
                  <item name="android:navigationBarColor">#071A2D</item>
              </style>
          </resources>
          EOF

          # 6. Vector Icon
          cat << 'EOF' > app/src/main/res/drawable/ic_bell_clock.xml
          <vector xmlns:android="http://schemas.android.com/apk/res/android"
              android:width="108dp"
              android:height="108dp"
              android:viewportWidth="108"
              android:viewportHeight="108">
              <path android:fillColor="#071A2D" android:pathData="M54,4 C26.386,4 4,26.386 4,54 C4,81.614 26.386,104 54,104 C81.614,104 104,81.614 104,54 C104,26.386 81.614,4 54,4 Z" />
              <path android:fillColor="#087EFF" android:pathData="M54,20 C43.5,20 35,28.5 35,39 C35,53 27,59 27,65 L81,65 C81,59 73,53 73,39 C73,28.5 64.5,20 54,20 Z" />
              <path android:fillColor="#16C784" android:pathData="M46,67 C46,71.4 49.6,75 54,75 C58.4,75 62,71.4 62,67 L46,67 Z" />
              <path android:fillColor="#FFFFFF" android:pathData="M78,64 C70.268,64 64,70.268 64,78 C64,85.732 70.268,92 78,92 C85.732,92 92,85.732 92,78 C92,70.268 85.732,64 78,64 Z" />
              <path android:fillColor="#071A2D" android:pathData="M77,69 L77,79 L84,79 L84,77 L79,77 L79,69 Z" />
          </vector>
          EOF

          # 7. Default Timetable Asset
          cat << 'EOF' > app/src/main/assets/default_timetable.json
          [
            {
              "dayOfWeek": "MONDAY",
              "periods": [
                { "id": "mon_0", "name": "Registration", "room": "Rm 14", "teacher": "Homeroom", "startTime": "07:45", "endTime": "08:00", "type": "REGISTRATION" },
                { "id": "mon_1", "name": "Technical Mathematics", "room": "Rm 14", "teacher": "Mr. Adams", "startTime": "08:00", "endTime": "08:45", "type": "LESSON" },
                { "id": "mon_b1", "name": "Break 1", "room": "Quad", "teacher": "", "startTime": "10:15", "endTime": "10:45", "type": "BREAK" },
                { "id": "mon_2", "name": "Technical Sciences", "room": "Lab 2", "teacher": "Mr. Pieterse", "startTime": "10:45", "endTime": "11:30", "type": "LESSON" },
                { "id": "mon_b2", "name": "Lunch Break", "room": "Quad", "teacher": "", "startTime": "13:00", "endTime": "13:30", "type": "LUNCH" }
              ]
            }
          ]
          EOF

          # 8. Models.kt
          cat << 'EOF' > app/src/main/java/com/schoolbell/app/data/Models.kt
          package com.schoolbell.app.data
          import kotlinx.serialization.Serializable
          import java.time.LocalTime
          import java.time.format.DateTimeFormatter

          enum class PeriodType { REGISTRATION, LESSON, BREAK, LUNCH }

          @Serializable
          data class Period(
              val id: String = java.util.UUID.randomUUID().toString(),
              var name: String,
              var room: String = "",
              var teacher: String = "",
              var startTime: String,
              var endTime: String,
              var type: PeriodType = PeriodType.LESSON
          ) {
              fun getStartLocalTime(): LocalTime = LocalTime.parse(startTime.trim().padStart(5, '0'), DateTimeFormatter.ofPattern("HH:mm"))
              fun getEndLocalTime(): LocalTime = LocalTime.parse(endTime.trim().padStart(5, '0'), DateTimeFormatter.ofPattern("HH:mm"))
          }

          @Serializable
          data class DaySchedule(val dayOfWeek: String, val periods: MutableList<Period> = mutableListOf())

          @Serializable
          data class UserProfile(
              val studentName: String = "Student",
              val schoolName: String = "Delft Technical High School",
              val grade: String = "10",
              val className: String = "10B"
          )

          @Serializable
          data class UserSettings(
              val notificationsEnabled: Boolean = true,
              val advanceRemindersEnabled: Boolean = true,
              val advanceMinutes: Int = 5,
              val lessonStartRemindersEnabled: Boolean = true,
              val lessonEndRemindersEnabled: Boolean = true,
              val breakRemindersEnabled: Boolean = true
          )

          enum class NotificationTriggerType { ADVANCE_START, START_NOW, END_NOW, BREAK_START, BREAK_END }
          EOF

          # 9. OEMBatteryHelper.kt
          cat << 'EOF' > app/src/main/java/com/schoolbell/app/notifications/OEMBatteryHelper.kt
          package com.schoolbell.app.notifications
          import android.content.ComponentName
          import android.content.Context
          import android.content.Intent
          import android.net.Uri
          import android.os.Build
          import android.provider.Settings

          object OEMBatteryHelper {
              fun getDeviceModelName(): String = "${Build.MANUFACTURER.replaceFirstChar { it.uppercase() }} ${Build.MODEL}"

              fun requestIgnoreBatteryOptimization(context: Context) {
                  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                      try {
                          val intent = Intent(Settings.ACTION_REQUEST_IGNORE_BATTERY_OPTIMIZATIONS).apply {
                              data = Uri.parse("package:${context.packageName}")
                              addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                          }
                          context.startActivity(intent)
                      } catch (_: Exception) {}
                  }
              }

              fun openOEMAutostartSettings(context: Context) {
                  val intent = Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                      data = Uri.parse("package:${context.packageName}")
                      addFlags(Intent.FLAG_ACTIVITY_NEW_TASK)
                  }
                  context.startActivity(intent)
              }
          }
          EOF

          # 10. NotificationHelper.kt
          cat << 'EOF' > app/src/main/java/com/schoolbell/app/notifications/NotificationHelper.kt
          package com.schoolbell.app.notifications
          import android.app.NotificationChannel
          import android.app.NotificationManager
          import android.app.PendingIntent
          import android.content.Context
          import android.content.Intent
          import android.os.Build
          import androidx.core.app.NotificationCompat
          import com.schoolbell.app.MainActivity
          import com.schoolbell.app.R

          object NotificationHelper {
              const val CHANNEL_LESSONS = "schoolbell_lessons"
              const val CHANNEL_BREAKS = "schoolbell_breaks"

              fun createNotificationChannels(context: Context) {
                  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
                      val manager = context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
                      manager.createNotificationChannel(NotificationChannel(CHANNEL_LESSONS, "Lessons", NotificationManager.IMPORTANCE_HIGH))
                      manager.createNotificationChannel(NotificationChannel(CHANNEL_BREAKS, "Breaks", NotificationManager.IMPORTANCE_HIGH))
                  }
              }

              fun showNotification(context: Context, id: Int, title: String, msg: String, isBreak: Boolean) {
                  val intent = Intent(context, MainActivity::class.java).apply {
                      flags = Intent.FLAG_ACTIVITY_SINGLE_TOP or Intent.FLAG_ACTIVITY_CLEAR_TOP
                  }
                  val pending = PendingIntent.getActivity(context, id, intent, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
                  val channel = if (isBreak) CHANNEL_BREAKS else CHANNEL_LESSONS
                  val builder = NotificationCompat.Builder(context, channel)
                      .setSmallIcon(R.drawable.ic_bell_clock)
                      .setContentTitle(title)
                      .setContentText(msg)
                      .setPriority(NotificationCompat.PRIORITY_HIGH)
                      .setContentIntent(pending)
                      .setAutoCancel(true)
                  val manager = context.getSystemService(Context.NOTIFICATION_SERVICE) as NotificationManager
                  manager.notify(id, builder.build())
              }
          }
          EOF

          # 11. AlarmScheduler & Receivers
          cat << 'EOF' > app/src/main/java/com/schoolbell/app/notifications/AlarmReceiver.kt
          package com.schoolbell.app.notifications
          import android.content.BroadcastReceiver
          import android.content.Context
          import android.content.Intent

          class AlarmReceiver : BroadcastReceiver() {
              override fun onReceive(context: Context, intent: Intent) {
                  val subject = intent.getStringExtra("subject") ?: "Class"
                  val isBreak = intent.getBooleanExtra("is_break", false)
                  val id = intent.getIntExtra("id", 1001)
                  val title = if (isBreak) "SchoolBell: Interval" else "SchoolBell: Lesson"
                  NotificationHelper.showNotification(context, id, title, "$subject is starting now!", isBreak)
              }
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/schoolbell/app/notifications/BootReceiver.kt
          package com.schoolbell.app.notifications
          import android.content.BroadcastReceiver
          import android.content.Context
          import android.content.Intent

          class BootReceiver : BroadcastReceiver() {
              override fun onReceive(context: Context, intent: Intent) {}
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/schoolbell/app/notifications/AlarmScheduler.kt
          package com.schoolbell.app.notifications
          import android.app.AlarmManager
          import android.app.PendingIntent
          import android.content.Context
          import android.content.Intent
          import android.os.Build
          import com.schoolbell.app.data.DaySchedule
          import com.schoolbell.app.data.UserSettings
          import java.time.LocalDateTime
          import java.time.ZoneId

          class AlarmScheduler(private val context: Context) {
              private val alarmManager = context.getSystemService(Context.ALARM_SERVICE) as AlarmManager

              fun rescheduleAll(timetable: List<DaySchedule>, settings: UserSettings) {
                  if (!settings.notificationsEnabled) return
                  for (day in timetable) {
                      for (period in day.periods) {
                          try {
                              val time = period.getStartLocalTime()
                              val ldt = LocalDateTime.now().withHour(time.hour).withMinute(time.minute).withSecond(0)
                              if (ldt.isAfter(LocalDateTime.now())) {
                                  val triggerAt = ldt.atZone(ZoneId.systemDefault()).toInstant().toEpochMilli()
                                  val intent = Intent(context, AlarmReceiver::class.java).apply {
                                      putExtra("subject", period.name)
                                      putExtra("is_break", period.type.name.contains("BREAK"))
                                      putExtra("id", period.id.hashCode())
                                  }
                                  val pending = PendingIntent.getBroadcast(context, period.id.hashCode(), intent, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE)
                                  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.M) {
                                      alarmManager.setExactAndAllowWhileIdle(AlarmManager.RTC_WAKEUP, triggerAt, pending)
                                  }
                              }
                          } catch (_: Exception) {}
                      }
                  }
              }
          }
          EOF

          # 12. TimetableRepository.kt
          cat << 'EOF' > app/src/main/java/com/schoolbell/app/data/TimetableRepository.kt
          package com.schoolbell.app.data
          import android.content.Context
          import androidx.datastore.preferences.core.edit
          import androidx.datastore.preferences.core.stringPreferencesKey
          import androidx.datastore.preferences.preferencesDataStore
          import kotlinx.coroutines.flow.first
          import kotlinx.serialization.encodeToString
          import kotlinx.serialization.json.Json
          import java.io.InputStreamReader

          private val Context.dataStore by preferencesDataStore(name = "schoolbell_prefs")

          class TimetableRepository(private val context: Context) {
              private val json = Json { ignoreUnknownKeys = true; prettyPrint = true }
              private val SCHEDULE_KEY = stringPreferencesKey("schedule_key")
              private val PROFILE_KEY = stringPreferencesKey("profile_key")

              suspend fun loadTimetable(): List<DaySchedule> {
                  val prefs = context.dataStore.data.first()
                  val raw = prefs[SCHEDULE_KEY]
                  return if (!raw.isNullOrBlank()) {
                      try { json.decodeFromString(raw) } catch (_: Exception) { loadDefault() }
                  } else loadDefault()
              }

              fun loadDefault(): List<DaySchedule> {
                  return try {
                      context.assets.open("default_timetable.json").use { stream ->
                          InputStreamReader(stream).use { json.decodeFromString(it.readText()) }
                      }
                  } catch (_: Exception) { emptyList() }
              }

              suspend fun saveTimetable(data: List<DaySchedule>) {
                  context.dataStore.edit { it[SCHEDULE_KEY] = json.encodeToString(data) }
              }

              suspend fun loadProfile(): UserProfile {
                  val prefs = context.dataStore.data.first()
                  val raw = prefs[PROFILE_KEY]
                  return if (!raw.isNullOrBlank()) {
                      try { json.decodeFromString(raw) } catch (_: Exception) { UserProfile() }
                  } else UserProfile()
              }

              suspend fun saveProfile(profile: UserProfile) {
                  context.dataStore.edit { it[PROFILE_KEY] = json.encodeToString(profile) }
              }

              suspend fun loadSettings(): UserSettings = UserSettings()
              suspend fun updateSettings(settings: UserSettings) {}
              suspend fun restoreDefaultTimetable(): List<DaySchedule> = loadDefault()
          }
          EOF

          # 13. UI Theme & Main Screen
          cat << 'EOF' > app/src/main/java/com/schoolbell/app/ui/theme/Theme.kt
          package com.schoolbell.app.ui.theme
          import androidx.compose.material3.*
          import androidx.compose.runtime.Composable
          import androidx.compose.ui.graphics.Color

          val DeepBlue = Color(0xFF071A2D)
          val BrightBlue = Color(0xFF087EFF)
          val EmeraldGreen = Color(0xFF16C784)

          @Composable
          fun SchoolBellTheme(content: @Composable () -> Unit) {
              MaterialTheme(
                  colorScheme = lightColorScheme(primary = BrightBlue, secondary = EmeraldGreen, background = Color(0xFFF4F7FB)),
                  content = content
              )
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/schoolbell/app/SchoolBellApplication.kt
          package com.schoolbell.app
          import android.app.Application
          import com.schoolbell.app.data.TimetableRepository
          import com.schoolbell.app.notifications.AlarmScheduler
          import com.schoolbell.app.notifications.NotificationHelper

          class SchoolBellApplication : Application() {
              lateinit var repository: TimetableRepository
              lateinit var alarmScheduler: AlarmScheduler

              override fun onCreate() {
                  super.onCreate()
                  NotificationHelper.createNotificationChannels(this)
                  repository = TimetableRepository(this)
                  alarmScheduler = AlarmScheduler(this)
              }
          }
          EOF

          cat << 'EOF' > app/src/main/java/com/schoolbell/app/MainActivity.kt
          package com.schoolbell.app
          import android.Manifest
          import android.content.pm.PackageManager
          import android.os.Build
          import android.os.Bundle
          import androidx.activity.ComponentActivity
          import androidx.activity.compose.setContent
          import androidx.activity.result.contract.ActivityResultContracts
          import androidx.compose.foundation.layout.*
          import androidx.compose.foundation.lazy.LazyColumn
          import androidx.compose.foundation.lazy.items
          import androidx.compose.foundation.shape.RoundedCornerShape
          import androidx.compose.material.icons.Icons
          import androidx.compose.material.icons.filled.*
          import androidx.compose.material3.*
          import androidx.compose.runtime.*
          import androidx.compose.ui.Alignment
          import androidx.compose.ui.Modifier
          import androidx.compose.ui.graphics.Color
          import androidx.compose.ui.text.font.FontWeight
          import androidx.compose.ui.unit.dp
          import androidx.compose.ui.unit.sp
          import androidx.core.content.ContextCompat
          import androidx.lifecycle.lifecycleScope
          import com.schoolbell.app.data.*
          import com.schoolbell.app.notifications.NotificationHelper
          import com.schoolbell.app.notifications.OEMBatteryHelper
          import com.schoolbell.app.ui.theme.*
          import kotlinx.coroutines.launch
          import java.time.LocalDate
          import java.time.LocalTime
          import java.time.format.DateTimeFormatter

          class MainActivity : ComponentActivity() {
              private val notifLauncher = registerForActivityResult(ActivityResultContracts.RequestPermission()) {}

              override fun onCreate(savedInstanceState: Bundle?) {
                  super.onCreate(savedInstanceState)
                  val app = application as SchoolBellApplication

                  if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                      if (ContextCompat.checkSelfPermission(this, Manifest.permission.POST_NOTIFICATIONS) != PackageManager.PERMISSION_GRANTED) {
                          notifLauncher.launch(Manifest.permission.POST_NOTIFICATIONS)
                      }
                  }

                  setContent {
                      SchoolBellTheme {
                          var timetable by remember { mutableStateOf<List<DaySchedule>>(emptyList()) }
                          var profile by remember { mutableStateOf(UserProfile()) }
                          var settings by remember { mutableStateOf(UserSettings()) }
                          var selectedTab by remember { mutableStateOf(0) }

                          LaunchedEffect(Unit) {
                              timetable = app.repository.loadTimetable()
                              profile = app.repository.loadProfile()
                              settings = app.repository.loadSettings()
                          }

                          Scaffold(
                              bottomBar = {
                                  NavigationBar {
                                      NavigationBarItem(
                                          selected = selectedTab == 0,
                                          onClick = { selectedTab = 0 },
                                          icon = { Icon(Icons.Default.Home, null) },
                                          label = { Text("Home") }
                                      )
                                      NavigationBarItem(
                                          selected = selectedTab == 1,
                                          onClick = { selectedTab = 1 },
                                          icon = { Icon(Icons.Default.CalendarMonth, null) },
                                          label = { Text("Timetable") }
                                      )
                                      NavigationBarItem(
                                          selected = selectedTab == 2,
                                          onClick = { selectedTab = 2 },
                                          icon = { Icon(Icons.Default.Settings, null) },
                                          label = { Text("Settings") }
                                      )
                                  }
                              }
                          ) { padding ->
                              Box(modifier = Modifier.padding(padding).fillMaxSize()) {
                                  when (selectedTab) {
                                      0 -> HomeScreen(timetable, profile)
                                      1 -> TimetableScreen(timetable)
                                      2 -> SettingsScreen(this@MainActivity, profile, onProfileChange = {
                                          profile = it
                                          lifecycleScope.launch { app.repository.saveProfile(it) }
                                      })
                                  }
                              }
                          }
                      }
                  }
              }
          }

          @Composable
          fun HomeScreen(timetable: List<DaySchedule>, profile: UserProfile) {
              val currentDay = LocalDate.now().dayOfWeek.name
              val todaySchedule = timetable.find { it.dayOfWeek.equals(currentDay, true) }

              Column(modifier = Modifier.padding(16.dp).fillMaxSize(), verticalArrangement = Arrangement.spacedBy(16.dp)) {
                  Card(modifier = Modifier.fillMaxWidth(), shape = RoundedCornerShape(16.dp), colors = CardDefaults.cardColors(containerColor = Color.White)) {
                      Column(modifier = Modifier.padding(16.dp)) {
                          Text(profile.schoolName, fontWeight = FontWeight.Bold, color = BrightBlue, fontSize = 18.sp)
                          Text("${profile.studentName} • Grade ${profile.className}", fontSize = 14.sp)
                          Text(LocalDate.now().format(DateTimeFormatter.ofPattern("EEEE, d MMMM")), fontSize = 12.sp, color = Color.Gray)
                      }
                  }

                  Text("Today's Lessons", fontWeight = FontWeight.Bold, fontSize = 16.sp)

                  if (todaySchedule == null || todaySchedule.periods.isEmpty()) {
                      Text("No classes scheduled for today. Rest well!")
                  } else {
                      LazyColumn(verticalArrangement = Arrangement.spacedBy(8.dp)) {
                          items(todaySchedule.periods) { period ->
                              Card(modifier = Modifier.fillMaxWidth(), shape = RoundedCornerShape(12.dp)) {
                                  Row(modifier = Modifier.padding(14.dp).fillMaxWidth(), verticalAlignment = Alignment.CenterVertically) {
                                      Column(modifier = Modifier.width(70.dp)) {
                                          Text(period.startTime, fontWeight = FontWeight.Bold)
                                          Text(period.endTime, fontSize = 12.sp, color = Color.Gray)
                                      }
                                      Spacer(modifier = Modifier.width(10.dp))
                                      Text(period.name, fontWeight = FontWeight.SemiBold, fontSize = 16.sp)
                                  }
                              }
                          }
                      }
                  }
              }
          }

          @Composable
          fun TimetableScreen(timetable: List<DaySchedule>) {
              LazyColumn(modifier = Modifier.padding(16.dp), verticalArrangement = Arrangement.spacedBy(12.dp)) {
                  items(timetable) { day ->
                      Card(modifier = Modifier.fillMaxWidth(), shape = RoundedCornerShape(14.dp)) {
                          Column(modifier = Modifier.padding(14.dp)) {
                              Text(day.dayOfWeek, fontWeight = FontWeight.Bold, color = BrightBlue)
                              Spacer(modifier = Modifier.height(6.dp))
                              day.periods.forEach { p ->
                                  Text("${p.startTime} - ${p.endTime}: ${p.name}", fontSize = 14.sp)
                              }
                          }
                      }
                  }
              }
          }

          @Composable
          fun SettingsScreen(context: ComponentActivity, profile: UserProfile, onProfileChange: (UserProfile) -> Unit) {
              var name by remember { mutableStateOf(profile.studentName) }
              var grade by remember { mutableStateOf(profile.className) }

              Column(modifier = Modifier.padding(16.dp).fillMaxSize(), verticalArrangement = Arrangement.spacedBy(14.dp)) {
                  Text("Student Profile", fontWeight = FontWeight.Bold, fontSize = 18.sp)
                  OutlinedTextField(value = name, onValueChange = { name = it; onProfileChange(profile.copy(studentName = it)) }, label = { Text("Student Name") }, modifier = Modifier.fillMaxWidth())
                  OutlinedTextField(value = grade, onValueChange = { grade = it; onProfileChange(profile.copy(className = it)) }, label = { Text("Class / Grade") }, modifier = Modifier.fillMaxWidth())

                  Spacer(modifier = Modifier.height(10.dp))
                  Button(
                      onClick = {
                          NotificationHelper.showNotification(context, 99, "SchoolBell Alert", "Bell notification tested successfully!", false)
                      },
                      modifier = Modifier.fillMaxWidth(),
                      colors = ButtonDefaults.buttonColors(containerColor = EmeraldGreen)
                  ) {
                      Text("Test Notification Bell")
                  }

                  OutlinedButton(
                      onClick = { OEMBatteryHelper.requestIgnoreBatteryOptimization(context) },
                      modifier = Modifier.fillMaxWidth()
                  ) {
                      Text("Allow Background Running (Any Phone)")
                  }
              }
          }
          EOF

      - name: Build APK with Gradle
        run: gradle assembleDebug --stacktrace

      - name: Upload Downloadable APK
        uses: actions/upload-artifact@v4
        with:
          name: SchoolBell-APK
          path: app/build/outputs/apk/debug/app-debug.apk
