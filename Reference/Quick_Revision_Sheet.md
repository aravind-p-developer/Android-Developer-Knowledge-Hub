# Android Quick Revision Sheet

> **Rapid-fire reference — tables, bullet points, and key facts for last-minute review**  
> Optimized for scanning. Every row matters.

---

## Table of Contents

1. [Activity Lifecycle States](#1-activity-lifecycle-states)
2. [Fragment Lifecycle States](#2-fragment-lifecycle-states)
3. [Service Types Comparison](#3-service-types-comparison)
4. [Launch Modes Comparison](#4-launch-modes-comparison)
5. [Intent Types Comparison](#5-intent-types-comparison)
6. [Threading Options Comparison](#6-threading-options-comparison)
7. [Data Persistence Options](#7-data-persistence-options)
8. [Common Memory Leaks — Causes & Fixes](#8-common-memory-leaks--causes--fixes)
9. [API Level Changes Timeline](#9-api-level-changes-timeline)
10. [Key Annotations Quick Reference](#10-key-annotations-quick-reference)
11. [Common ANR Causes & Solutions](#11-common-anr-causes--solutions)
12. [Modern vs Legacy Comparison](#12-modern-vs-legacy-comparison)
13. [ViewModel Survival Matrix](#13-viewmodel-survival-matrix)
14. [Coroutine Scope Quick Reference](#14-coroutine-scope-quick-reference)
15. [Architecture Layer Rules](#15-architecture-layer-rules)

---

## 1. Activity Lifecycle States

| Method | Trigger | Do Here | UI Visible? |
|---|---|---|---|
| `onCreate()` | First creation | Inflate layout, init ViewModel, restore state | ❌ |
| `onStart()` | Becoming visible | Register lightweight listeners | ✅ (not interactive) |
| `onResume()` | Fully interactive | Start animations, camera, sensors | ✅ |
| `onPause()` | Partially obscured | Pause camera/sensor; MUST be fast | ✅ (partially) |
| `onStop()` | Fully hidden | Save data to disk, unregister heavy listeners | ❌ |
| `onRestart()` | Returning from stopped | Rarely overridden | ❌ |
| `onDestroy()` | Being destroyed | Cancel coroutines, final cleanup | ❌ |
| `onSaveInstanceState()` | Before stop (config change) | Save lightweight UI state to Bundle | ❌ |
| `onRestoreInstanceState()` | After onStart (if state exists) | Restore UI state from Bundle | ✅ |

**Lifecycle Pairs (always come in pairs):**
```
onCreate    ↔  onDestroy     (FULL lifetime)
onStart     ↔  onStop        (VISIBLE lifetime)
onResume    ↔  onPause       (FOREGROUND/interactive lifetime)
```

**Configuration Change Sequence:**
```
onPause → onStop → onSaveInstanceState → onDestroy → [new instance] → onCreate → onStart → onRestoreInstanceState → onResume
```

**Back Press Sequence (API < 33):**
```
onPause → onStop → onDestroy
```

---

## 2. Fragment Lifecycle States

| Method | Notes |
|---|---|
| `onAttach()` | Fragment attached to host Activity. `requireActivity()` is safe from here. |
| `onCreate()` | Fragment instance created. NO view yet. Initialize non-view resources. |
| `onCreateView()` | Inflate and return the fragment's view. |
| `onViewCreated()` | View ready. **Set up click listeners, observers HERE.** |
| `onViewStateRestored()` | View state restored from saved state. |
| `onStart()` | Fragment visible. |
| `onResume()` | Fragment interactive. |
| `onPause()` | Fragment partially hidden. |
| `onStop()` | Fragment fully hidden. |
| `onSaveInstanceState()` | Save instance state here. |
| `onDestroyView()` | **View destroyed** (back stack). Fragment instance SURVIVES. Null out view references! |
| `onDestroy()` | Fragment instance destroyed. |
| `onDetach()` | Fragment detached from Activity. |

**Key Rules:**
- ✅ Always use `viewLifecycleOwner` for observers in `onViewCreated()`
- ✅ `childFragmentManager` for nested fragments; `parentFragmentManager` for Activity-level
- ⚠️ `onDestroyView()` is called when fragment goes on back stack — view is gone but fragment is NOT
- ✅ `by viewModels()` — ViewModel scoped to this Fragment
- ✅ `by activityViewModels()` — ViewModel scoped to the Activity (shared with other fragments)

---

## 3. Service Types Comparison

| Type | Started By | Lives Until | Notification | Killed by System? | Use Case |
|---|---|---|---|---|---|
| **Started Service** | `startService()` | `stopSelf()` or `stopService()` | No | Yes (low memory) | Simple background tasks |
| **Foreground Service** | `startService()` + `startForeground()` | `stopSelf()` | ✅ Required | Rarely | Music, navigation, ongoing download |
| **Bound Service** | `bindService()` | All clients unbound | No | Yes (if not foreground) | IPC, media controller |
| **IntentService** 🔴 | `startService()` | Work complete | No | Yes | **Deprecated** — use WorkManager |
| **WorkManager** | `WorkManager.enqueue()` | Work complete | No | Work is rescheduled | Deferred, guaranteed, constrained work |

**Service runs on:** Main thread by default — always move work off thread!

```kotlin
// Service return values (onStartCommand)
START_STICKY           // Recreated with null intent after kill
START_NOT_STICKY       // NOT recreated after kill
START_REDELIVER_INTENT // Recreated AND last intent redelivered
```

**Foreground Service Types (API 34+ required in manifest):**
```
camera | connectedDevice | dataSync | health | location |
mediaPlayback | mediaProjection | microphone | phoneCall | remoteMessaging | shortService | specialUse | systemExempted
```

---

## 4. Launch Modes Comparison

| Mode | New Instance Created? | Multiple Instances? | `onNewIntent()` Called? | Task Behavior |
|---|---|---|---|---|
| `standard` | Always | ✅ Yes | ❌ No | Added to calling task |
| `singleTop` | Only if NOT at top | ✅ Yes (if not at top) | ✅ Yes (if at top) | Added to calling task |
| `singleTask` | Only if no instance exists | ❌ No | ✅ Yes (if exists) | Activities above it are cleared |
| `singleInstance` | Only if no instance exists | ❌ No | ✅ Yes (if exists) | **Isolated task — no other activities** |

**Intent Flags (programmatic equivalents):**
```kotlin
FLAG_ACTIVITY_NEW_TASK         // Similar to singleTask
FLAG_ACTIVITY_CLEAR_TOP        // Clear activities above target
FLAG_ACTIVITY_SINGLE_TOP       // Same as singleTop launch mode
FLAG_ACTIVITY_CLEAR_TASK       // Clear entire task before starting
FLAG_ACTIVITY_NO_HISTORY       // Don't add to back stack
FLAG_ACTIVITY_REORDER_TO_FRONT // Bring existing to front without clearing
```

**Real-World Mapping:**
```
Home Screen Activity    → singleTask
Notification target     → singleTop  (re-use if already open)
Alarm/Call screen       → singleInstance
Normal screens          → standard
```

---

## 5. Intent Types Comparison

| Type | Target Known? | Component Specified? | Resolved By | Risk |
|---|---|---|---|---|
| **Explicit** | ✅ Yes | Class reference | Direct resolution | None |
| **Implicit** | ❌ No | Action/data/category | Intent filter matching | ActivityNotFoundException |

**Implicit Intent must-check:**
```kotlin
// Always verify an implicit intent can be resolved
if (packageManager.queryIntentActivities(intent, 0).isNotEmpty()) {
    startActivity(intent)
}
```

**PendingIntent Flags (API 31+):**
```kotlin
PendingIntent.FLAG_IMMUTABLE      // Intent cannot be modified (use by default)
PendingIntent.FLAG_MUTABLE        // Intent can be modified (inline reply, bubbles)
PendingIntent.FLAG_UPDATE_CURRENT // Replace extras of existing PendingIntent
PendingIntent.FLAG_ONE_SHOT       // Can only be used once
PendingIntent.FLAG_CANCEL_CURRENT // Cancel existing, create new
```

**Common Implicit Intent Actions:**
```kotlin
ACTION_VIEW        // View data (URL, geo, email)
ACTION_SEND        // Share data
ACTION_CALL        // Make a phone call
ACTION_DIAL        // Open dialer with number
ACTION_PICK        // Pick from gallery
ACTION_GET_CONTENT // Pick file/content
ACTION_SENDTO      // Send to specific address (SMS)
```

---

## 6. Threading Options Comparison

| Mechanism | Thread | Lifecycle Aware | Error Handling | Status |
|---|---|---|---|---|
| `AsyncTask` | Background | ❌ No | Poor | 🔴 **Deprecated API 30** |
| `Handler` / `Looper` | Any | ❌ No | Manual | 🟡 Use when needed for message passing |
| `HandlerThread` | Named bg thread | ❌ No | Manual | 🟡 Use for dedicated bg thread |
| `Thread` | New thread | ❌ No | Manual | 🔴 Avoid — no pooling |
| `ExecutorService` | Thread pool | ❌ No | `Future.get()` | 🟡 Java interop |
| `Coroutines` | Any (Dispatcher) | ✅ With scope | `try/catch`, `CoroutineExceptionHandler` | 🟢 **Preferred** |
| `WorkManager` | Background | ✅ System-managed | Retry policy | 🟢 For deferred guaranteed work |
| `RxJava` | Any (Scheduler) | Manual | `onError` | 🟡 Valid but complex |

**Dispatcher Selection:**
```
Dispatchers.Main    → UI work, LiveData, StateFlow emission
Dispatchers.IO      → Network, File, Database (SharedPreferences, Room, Retrofit)
Dispatchers.Default → CPU work: JSON parsing, sorting, encryption
```

**Coroutine Builders:**
```kotlin
launch { }          // Fire-and-forget; returns Job
async { }           // Returns Deferred<T>; use await() to get result
runBlocking { }     // Blocks current thread; TESTING ONLY
withContext(D) { }  // Switch dispatcher; suspending; returns result
```

---

## 7. Data Persistence Options

| Option | Type | Thread Safe | Max Size | Survives Process Death | Query Support | Use Case |
|---|---|---|---|---|---|---|
| `SharedPreferences` | Key-value | ⚠️ Partially | RAM limited | ✅ | ❌ | Simple flags (legacy) |
| `Preferences DataStore` | Key-value (typed) | ✅ | RAM limited | ✅ | ❌ | Settings, user prefs |
| `Proto DataStore` | Typed schema | ✅ | RAM limited | ✅ | ❌ | Complex typed prefs |
| `Room` | Relational SQL | ✅ | Storage limited | ✅ | ✅ | Structured app data |
| `SQLite` (raw) | Relational SQL | ❌ Manual | Storage limited | ✅ | ✅ | Avoid; use Room |
| `Internal Files` | Raw bytes | ❌ Manual | Storage limited | ✅ | ❌ | Blobs, logs |
| `External Files` | Raw bytes | ❌ Manual | Storage limited | ✅ | ❌ | User files (with permission) |
| `EncryptedSharedPreferences` | Key-value | ⚠️ | RAM limited | ✅ | ❌ | Sensitive settings/tokens |
| `EncryptedFile` | Bytes | ❌ Manual | Storage limited | ✅ | ❌ | Sensitive file data |

**Room Quick Reference:**
```kotlin
@Entity(tableName = "users")
data class User(@PrimaryKey val id: Int, val name: String, val email: String)

@Dao
interface UserDao {
    @Query("SELECT * FROM users WHERE id = :id")
    fun getUserById(id: Int): Flow<User>
    
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insert(user: User)
    
    @Update
    suspend fun update(user: User)
    
    @Delete
    suspend fun delete(user: User)
}

@Database(entities = [User::class], version = 1, exportSchema = false)
abstract class AppDatabase : RoomDatabase() {
    abstract fun userDao(): UserDao
}
```

---

## 8. Common Memory Leaks — Causes & Fixes

| Leak Scenario | Root Cause | Fix |
|---|---|---|
| `companion object { var ctx: Context }` | Static reference to Activity context | Use `applicationContext`; never store Activity statically |
| Non-static inner `Handler` class | Inner class holds implicit Activity reference | Use `WeakReference<Activity>` in handler; or use coroutines |
| `LiveData.observe(this)` in Fragment | Observer stays alive after `onDestroyView()` | Use `viewLifecycleOwner` instead of `this` |
| `registerReceiver` without `unregisterReceiver` | Registered receiver never removed | Unregister in matching lifecycle method |
| Singleton holding Activity context | Singleton lives forever; Activity never GC'd | Inject `applicationContext`; use Hilt's `@Singleton` |
| `handler.postDelayed(runnable, delay)` | Runnable captures outer class | Call `handler.removeCallbacks(runnable)` in `onDestroy()` |
| `addObserver` without `removeObserver` | Observer added in onCreate but never removed | Use lifecycle-aware observers; or remove in `onDestroy()` |
| Bitmap stored in static field | Large bitmaps preventing GC | Use Glide/Coil; avoid static bitmap caches |
| Cursor not closed | SQL cursors remain open | Always close in `finally` block or use Room |
| Window leaks (Dialog shown after Activity finishes) | Dialog references destroyed Activity | Dismiss dialogs in `onStop()`/`onDestroy()` |

**LeakCanary setup (debug only):**
```kotlin
// build.gradle.kts
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
// Zero configuration required — auto-hooks into lifecycle
```

---

## 9. API Level Changes Timeline

| Android Version | API | Key Developer-Affecting Changes |
|---|---|---|
| **5.0 Lollipop** | 21 | ART replaces Dalvik, Material Design, JobScheduler, `Camera2` API |
| **6.0 Marshmallow** | 23 | **Runtime permissions**, Doze mode, App standby |
| **7.0 Nougat** | 24 | Multi-window, `FileProvider` required for file sharing, background data limit |
| **8.0 Oreo** | 26 | **Background service limits**, **Broadcast limits**, Notification channels required, `startForegroundService()` |
| **9.0 Pie** | 28 | Background activity start restrictions, non-SDK API restrictions, `http://` blocked by default |
| **10 Q** | 29 | **Scoped Storage** (opt-in), `ACCESS_BACKGROUND_LOCATION` permission, dark theme |
| **11 R** | 30 | **Scoped Storage enforced**, package visibility (`<queries>`), one-time permissions, `AsyncTask` deprecated |
| **12 S** | 31 | **`PendingIntent` mutability flag required**, Splash Screen API, exact alarm permission, `AppSearch` |
| **12L** | 32 | Large screen and tablet optimizations |
| **13 T** | 33 | `POST_NOTIFICATIONS` permission required, granular media permissions (`READ_MEDIA_IMAGES`), Per-app language |
| **14 U** | 34 | `foregroundServiceType` required in manifest, Photo picker default, Predictive back gesture |
| **15 V** | 35 | Edge-to-edge enforcement, PDF APIs, Health Connect integration |

**Critical thresholds to memorize:**
```
API 23 → Runtime permissions
API 26 → Notification channels; background service restrictions
API 29 → Scoped storage begins
API 30 → Scoped storage enforced; package visibility
API 31 → PendingIntent mutability required
API 33 → Notification permission required (POST_NOTIFICATIONS)
API 34 → foregroundServiceType required
```

---

## 10. Key Annotations Quick Reference

### Hilt
```kotlin
@HiltAndroidApp          // Application class — entry point for Hilt
@AndroidEntryPoint       // Activity, Fragment, Service, View, BroadcastReceiver
@HiltViewModel           // ViewModel — enables @Inject constructor with Hilt
@Inject                  // Constructor/field/method injection
@Module                  // Hilt module — contains @Provides or @Binds
@InstallIn(X::class)     // Which Hilt component this module is installed in
@Provides                // Factory method for a dependency
@Binds                   // Bind interface to implementation (abstract fun)
@Singleton               // One instance per application lifetime
@ActivityScoped          // One instance per Activity
@ViewModelScoped         // One instance per ViewModel
@Qualifier               // Custom qualifier to distinguish same-type bindings
@Named("name")           // String-based qualifier
```

### Room
```kotlin
@Entity(tableName = "x") // Marks a class as a database table
@PrimaryKey              // Marks the primary key field
@PrimaryKey(autoGenerate = true) // Auto-increment integer PK
@ColumnInfo(name = "x")  // Custom column name
@Ignore                  // Exclude field from database
@Dao                     // Marks an interface as a Data Access Object
@Query("SQL")            // Raw SQL query
@Insert                  // Insert operation
@Update                  // Update operation
@Delete                  // Delete operation
@Database(entities=[X::class], version=1) // Marks the Room database class
@TypeConverter           // Method to convert non-primitive types
@Embedded                // Embed a nested object inline
@Relation                // One-to-many / many-to-many relationship
@Transaction             // Wraps multiple DAO operations atomically
@ForeignKey              // Declare foreign key constraint
```

### Jetpack Compose
```kotlin
@Composable              // Marks a composable function
@Preview                 // Render composable in Android Studio preview
@Stable                  // Tells Compose the type is stable (smart recomposition)
@Immutable               // Type is completely immutable
@Remember                // Persist state across recompositions (used with remember{})
@SideEffect              // Run non-composable side effects
@DrawableRes             // Parameter expects a drawable resource ID
@StringRes               // Parameter expects a string resource ID
@ColorRes                // Parameter expects a color resource ID
```

### Kotlin / KSP
```kotlin
@Parcelize               // Auto-generate Parcelable (kotlin-parcelize plugin)
@IgnoredOnParcel         // Exclude field from Parcelable implementation
@JvmStatic               // Generate static method in Java interop
@JvmField                // Expose Kotlin property as Java field
@JvmOverloads            // Generate Java overloads for default param functions
@SerializedName("key")   // Gson JSON key mapping
@Json(name = "key")      // Moshi JSON key mapping
```

### Lifecycle
```kotlin
@OnLifecycleEvent        // 🔴 Deprecated — use DefaultLifecycleObserver
```

---

## 11. Common ANR Causes & Solutions

| Cause | Thread | Threshold | Solution |
|---|---|---|---|
| Network call on main thread | Main | 5 sec | `withContext(Dispatchers.IO)` |
| Database query on main thread | Main | 5 sec | `withContext(Dispatchers.IO)` / Room suspend |
| SharedPreferences `commit()` on main thread | Main | 5 sec | Use `apply()` or DataStore |
| Heavy computation in `onCreate()` | Main | 5 sec | `withContext(Dispatchers.Default)` |
| Deadlock (two threads blocking each other) | Any | 5 sec | Review locking strategy; use coroutines |
| `BroadcastReceiver.onReceive()` doing I/O | Main | 10 sec | Use `goAsync()` or delegate to WorkManager |
| `Service.onStartCommand()` doing I/O | Main | 20 sec | Move work to coroutine scope in service |
| Waiting on `Semaphore`/`Lock` on main thread | Main | 5 sec | Do synchronization on background thread |
| `RecyclerView` binding doing I/O | Main | 5 sec | Pre-load data; use DiffUtil on Dispatchers.Default |

**StrictMode for development:**
```kotlin
if (BuildConfig.DEBUG) {
    StrictMode.setThreadPolicy(
        StrictMode.ThreadPolicy.Builder()
            .detectDiskReads()
            .detectDiskWrites()
            .detectNetwork()
            .penaltyLog()
            .penaltyDialog()
            .build()
    )
    StrictMode.setVmPolicy(
        StrictMode.VmPolicy.Builder()
            .detectLeakedSqlLiteObjects()
            .detectLeakedClosableObjects()
            .detectActivityLeaks()
            .penaltyLog()
            .build()
    )
}
```

**Profile with:**
```
Android Studio → Profiler → CPU → Record method trace
adb shell am bug-report        → Full bug report with ANR info
adb shell dumpsys activity top → Current top activity state
```

---

## 12. Modern vs Legacy Comparison

| Feature | 🔴 Legacy | 🟢 Modern |
|---|---|---|
| **Async work** | `AsyncTask` | Kotlin Coroutines + `viewModelScope` |
| **Background tasks** | `Service`, `IntentService` | `WorkManager`, Foreground Service |
| **UI state** | `LiveData` | `StateFlow` / `SharedFlow` |
| **Preferences** | `SharedPreferences` | `Preferences DataStore` |
| **File sharing** | Direct file path | `FileProvider` |
| **Runtime permissions** | `onRequestPermissionsResult()` | `ActivityResultContracts.RequestPermission` |
| **Activity result** | `startActivityForResult()` + `onActivityResult()` | `registerForActivityResult()` |
| **Inter-fragment comm.** | Interface callbacks | Shared ViewModel / Fragment Result API |
| **In-app events** | `LocalBroadcastManager` | `SharedFlow` / `Channel` |
| **DI** | Manual / Dagger | **Hilt** |
| **UI toolkit** | XML Views + `ViewBinding` | **Jetpack Compose** |
| **Navigation** | Manual `FragmentManager` | **Navigation Component** |
| **Image loading** | Manual `BitmapFactory` | **Glide** / **Coil** |
| **Networking** | `HttpURLConnection` | **Retrofit** + OkHttp |
| **Serialization** | `Gson` | **Kotlin Serialization** / Moshi |
| **DB access** | Raw SQLite | **Room** |
| **View binding** | `findViewById()` | `ViewBinding` / Compose |
| **Testing** | Instrumented only | Unit + Instrumented, Turbine, Robolectric |

---

## 13. ViewModel Survival Matrix

| Event | Activity | ViewModel | `onSaveInstanceState` Bundle |
|---|---|---|---|
| Orientation change | Destroyed + Recreated | ✅ **Survives** | ✅ Saved & Restored |
| Language change | Destroyed + Recreated | ✅ **Survives** | ✅ Saved & Restored |
| Dark mode change | Destroyed + Recreated | ✅ **Survives** | ✅ Saved & Restored |
| User presses Back | Destroyed | ❌ **Destroyed** | ❌ Not restored |
| User presses Home | Stopped (not destroyed) | ✅ **Survives** | N/A |
| App removed from recents | Destroyed | ❌ **Destroyed** | ❌ Not restored |
| Process death (low memory) | Destroyed | ❌ **Destroyed** | ✅ Restored via `savedStateHandle` |
| `finish()` called | Destroyed | ❌ **Destroyed** | ❌ Not restored |

**SavedStateHandle limits:**
```
Max Bundle size: ~1 MB (Binder transaction limit)
Supported types: Parcelable, Serializable, primitives, String, arrays of these
```

---

## 14. Coroutine Scope Quick Reference

| Scope | Created By | Cancelled When | Cancel on Exception? |
|---|---|---|---|
| `viewModelScope` | `ViewModel` (auto) | ViewModel cleared | ✅ (SupervisorJob) |
| `lifecycleScope` | Activity/Fragment (auto) | `DESTROYED` state | ✅ (SupervisorJob) |
| `rememberCoroutineScope()` | Compose | Composable leaves composition | ✅ |
| `GlobalScope` | Manual | Never | ⚠️ Avoid |
| Custom Scope | `CoroutineScope(Job() + Dispatcher)` | Manual `cancel()` | Depends on Job type |

**Flow collection in UI (correct pattern):**
```kotlin
// Fragment — the ONLY correct way to collect flows from UI
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        // Collection starts when STARTED, stops when below STARTED
        // Restarts automatically when lifecycle returns to STARTED
        viewModel.uiState.collect { render(it) }
    }
}
```

**Exception handling:**
```kotlin
// CoroutineExceptionHandler for top-level exceptions
val handler = CoroutineExceptionHandler { _, exception ->
    Log.e("TAG", "Caught: $exception")
}

viewModelScope.launch(handler) {
    riskyOperation()
}

// try/catch inside coroutine (preferred for specific handling)
viewModelScope.launch {
    try {
        val result = repository.fetchData()
        _state.value = Success(result)
    } catch (e: IOException) {
        _state.value = Error(e.message)
    }
}
```

---

## 15. Architecture Layer Rules

```
┌─────────────────────────────────────────────────┐
│                   UI LAYER                       │
│  Activity / Fragment / Composable / View         │
│  ✅ Observes state from ViewModel                │
│  ✅ Sends user events to ViewModel               │
│  ❌ No business logic                           │
│  ❌ No direct data access                       │
└──────────────────┬──────────────────────────────┘
                   │ observes / calls
┌──────────────────▼──────────────────────────────┐
│                VIEWMODEL LAYER                   │
│  ViewModel (one per screen or shared)            │
│  ✅ Holds and exposes UI state (StateFlow)       │
│  ✅ Handles UI events, calls UseCases/Repos      │
│  ✅ viewModelScope for coroutines                │
│  ❌ No Android framework (Context, View)        │
│  ❌ No direct database/network calls            │
└──────────────────┬──────────────────────────────┘
                   │ calls
┌──────────────────▼──────────────────────────────┐
│              DOMAIN LAYER (optional)             │
│  UseCases / Interactors                          │
│  ✅ Single-purpose business logic                │
│  ✅ Callable from multiple ViewModels            │
│  ❌ No Android framework                        │
└──────────────────┬──────────────────────────────┘
                   │ calls
┌──────────────────▼──────────────────────────────┐
│                 DATA LAYER                       │
│  Repository (single source of truth)             │
│  ✅ Abstracts remote vs local data sources       │
│  ✅ Caching strategy lives here                  │
│  Remote: Retrofit / API Service                  │
│  Local: Room DAO / DataStore                     │
└─────────────────────────────────────────────────┘
```

**One-Liner Rules:**
- ViewModel never holds `Context` → use `AndroidViewModel` only as last resort (prefer Hilt abstraction)
- Repository never exposes `LiveData` → use `Flow` and let ViewModel convert
- Activity/Fragment never directly calls a Repository
- ViewModel is never scoped to a View (always to a Lifecycle)
- One ViewModel per screen; share via `activityViewModels()` or NavGraph scope

---

## 16. Broadcast Restrictions Quick Reference

| Broadcast | API 26+ Static OK? | Notes |
|---|---|---|
| `BOOT_COMPLETED` | ✅ Yes | Must have `RECEIVE_BOOT_COMPLETED` permission |
| `LOCKED_BOOT_COMPLETED` | ✅ Yes | Direct boot support |
| `ACTION_MY_PACKAGE_REPLACED` | ✅ Yes | Your app's own update |
| `CONNECTIVITY_CHANGE` | ❌ No | Use `NetworkCallback` via `ConnectivityManager` |
| `NEW_PICTURE` / `NEW_VIDEO` | ❌ No | Use `ContentObserver` |
| `BATTERY_CHANGED` | ❌ No (no static) | Can only be received dynamically |
| `SMS_RECEIVED` | ✅ Yes (SMS apps) | Only for default SMS app |
| Custom app broadcasts | ✅ Yes | Your own explicit broadcasts |

---

## 17. Navigation Component Quick Reference

| Concept | Description |
|---|---|
| `NavGraph` | XML file defining destinations and actions |
| `NavHostFragment` | Container in layout that swaps destinations |
| `NavController` | Performs navigation; tracks back stack |
| `SafeArgs` | Gradle plugin for type-safe argument passing |
| `NavDeepLink` | URL pattern mapped to a destination |
| `NavigationUI` | Integrates NavController with Toolbar/BottomNav |

**Common operations:**
```kotlin
// Navigate to destination
navController.navigate(R.id.action_home_to_detail)

// Navigate with Safe Args
navController.navigate(HomeDirections.actionHomeToDetail(itemId = 42))

// Navigate with deep link
navController.navigate(Uri.parse("myapp://detail/42"))

// Go back
navController.popBackStack()
navController.navigateUp()

// Pop to specific destination (keep it)
navController.popBackStack(R.id.homeFragment, inclusive = false)

// Pop to specific destination (remove it too)
navController.popBackStack(R.id.homeFragment, inclusive = true)

// Check if can go up
if (navController.previousBackStackEntry != null) { /* can go back */ }
```

**BottomNav with multiple back stacks (Navigation 2.4+):**
```kotlin
NavigationUI.setupWithNavController(bottomNavigationView, navController)
// Each bottom nav tab maintains its own back stack automatically
```

---

*End of Quick Revision Sheet*

---

> 🎯 **Key things interviewers ACTUALLY test:**  
> 1. `onSaveInstanceState` vs ViewModel survival matrix  
> 2. `viewLifecycleOwner` in fragments  
> 3. `repeatOnLifecycle` for Flow collection  
> 4. Process death + `SavedStateHandle`  
> 5. Why `singleTask` clears the back stack  
> 6. Service runs on main thread  
> 7. API 26 broadcast/service restrictions  
> 8. `@Binds` vs `@Provides`  
> 9. `Channel` for one-time events  
> 10. `START_STICKY` vs `START_NOT_STICKY`

---

*Android Fundamentals: The Complete Developer & Interview Handbook | Appendix B*
