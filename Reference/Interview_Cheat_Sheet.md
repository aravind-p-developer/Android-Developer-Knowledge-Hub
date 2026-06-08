# Android Developer Interview Cheat Sheet

> **One-stop reference for Android developer interviews — Junior to Senior level**  
> All answers are production-accurate and reflect modern Jetpack / Kotlin-first practices.

---

## Table of Contents

1. [Activity & Lifecycle](#1-activity--lifecycle)
2. [Fragments](#2-fragments)
3. [Launch Modes & Back Stack](#3-launch-modes--back-stack)
4. [Intents & PendingIntent](#4-intents--pendingintent)
5. [Threading & Coroutines](#5-threading--coroutines)
6. [Services & Background Work](#6-services--background-work)
7. [Broadcast Receivers](#7-broadcast-receivers)
8. [MVVM & ViewModel](#8-mvvm--viewmodel)
9. [Memory & Performance](#9-memory--performance)
10. [Navigation Component](#10-navigation-component)
11. [Dependency Injection with Hilt](#11-dependency-injection-with-hilt)
12. [Data Persistence](#12-data-persistence)

---

## 1. Activity & Lifecycle

---

### Q1. What are the Activity lifecycle methods and when is each called?

**Answer:**

| Method | Called When |
|---|---|
| `onCreate()` | Activity is first created. Do: inflate layout, init ViewModel, restore state. |
| `onStart()` | Activity becomes visible (not yet interactive). |
| `onResume()` | Activity is in the foreground and fully interactive. |
| `onPause()` | Another activity comes partially in front. Must be lightweight — the new activity won't resume until `onPause()` returns. |
| `onStop()` | Activity is completely hidden (e.g., user presses Home). |
| `onRestart()` | Activity returns to started state after being stopped. |
| `onDestroy()` | Activity is being destroyed — either by `finish()` or by the system. |

**Key pairs:**
- `onCreate` ↔ `onDestroy` — full lifetime
- `onStart` ↔ `onStop` — visible lifetime
- `onResume` ↔ `onPause` — foreground lifetime

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    // Called once per activity creation
    // savedInstanceState is null on first launch
}

override fun onPause() {
    super.onPause()
    // MUST be fast — blocks next activity from resuming
    // Save lightweight data here, NOT disk I/O
}
```

---

### Q2. What happens during an orientation change?

**Answer:**

By default, a configuration change (including orientation) causes the system to **destroy and recreate** the Activity. The sequence is:

```
onPause() → onStop() → onDestroy() → [new instance] → onCreate() → onStart() → onResume()
```

**What survives automatically:** Nothing in the Activity itself.

**What you must save manually:** UI state that the user entered (e.g., text in an EditText). Use `onSaveInstanceState(Bundle)`.

**What survives automatically via ViewModel:** Business logic data, network results, LiveData/StateFlow values — because ViewModel is NOT destroyed during configuration changes.

**How to prevent recreation entirely (not recommended):**
```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".MyActivity"
    android:configChanges="orientation|screenSize|keyboardHidden" />
```
Then handle it in `onConfigurationChanged()`. Only do this when you have a specific reason (e.g., a game or camera app).

---

### Q3. Difference between `onSaveInstanceState` and ViewModel?

**Answer:**

| | `onSaveInstanceState` (Bundle) | ViewModel |
|---|---|---|
| **Survives config change** | ✅ Yes | ✅ Yes |
| **Survives process death** | ✅ Yes (system preserves Bundle) | ❌ No (ViewModel is destroyed) |
| **Storage limit** | ~1 MB (Binder transaction limit) | No practical limit (heap memory) |
| **Data type** | Primitives, Parcelable, String | Any Kotlin/Java object |
| **Async operations** | ❌ Cannot hold coroutines | ✅ `viewModelScope` persists across config changes |
| **Best for** | Saving UI state (selected item ID, scroll pos) | Holding business data, loading states |

🎯 **Interview Trap:** "Does ViewModel survive process death?" **No.** Use `SavedStateHandle` inside ViewModel to survive process death:

```kotlin
@HiltViewModel
class MyViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    
    // This value survives both config changes AND process death
    var searchQuery: String
        get() = savedStateHandle["query"] ?: ""
        set(value) { savedStateHandle["query"] = value }
}
```

---

### Q4. What is `onRestoreInstanceState`?

**Answer:**

`onRestoreInstanceState(Bundle)` is called **after `onStart()`** (only when there is a saved state to restore). It receives the same `Bundle` that was passed to `onSaveInstanceState()`.

**vs. `onCreate(Bundle?)`:**
- `onCreate()` — `savedInstanceState` is `null` on first launch, non-null on recreation
- `onRestoreInstanceState()` — only called when state exists; `Bundle` is **always non-null** here

```kotlin
override fun onRestoreInstanceState(savedInstanceState: Bundle) {
    super.onRestoreInstanceState(savedInstanceState)
    val scrollY = savedInstanceState.getInt("scroll_position", 0)
    recyclerView.scrollBy(0, scrollY)
}
```

**Best practice:** Restore state in `onCreate()` (check for null) for most cases. Use `onRestoreInstanceState()` only when restoration must happen after views are fully ready.

---

### Q5. When is `onDestroy()` called but `onStop()` is not?

**Answer:**

This happens when `finish()` is called from within `onCreate()`:

```kotlin
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    if (!userIsLoggedIn()) {
        finish()  // Activity never became visible
        return    // onStop() is NOT called — onDestroy() is called directly
    }
}
```

Since the activity never reached `onStart()` or `onResume()`, there is nothing to "stop" — the lifecycle goes directly from `onCreate()` to `onDestroy()`.

---

## 2. Fragments

---

### Q6. Difference between `add()` and `replace()` in FragmentTransaction?

**Answer:**

| | `add()` | `replace()` |
|---|---|---|
| **Effect** | Adds a new fragment on top of existing ones in the container | Removes existing fragments from the container, adds the new one |
| **Previous fragment** | Still alive and attached (invisible but active) | Destroyed (unless added to back stack) |
| **Back stack behavior** | On back press, newly added fragment is removed | On back press, new fragment is removed and previous is restored |
| **Use case** | Overlaying fragments (e.g., dialog-like overlays) | Normal screen navigation |
| **Memory** | Higher — multiple fragments alive simultaneously | Lower — only active fragment in memory |

```kotlin
// replace() — preferred for navigation
supportFragmentManager.commit {
    replace(R.id.container, DetailFragment())
    addToBackStack("detail")
}

// add() — use when you want the previous fragment to stay alive
supportFragmentManager.commit {
    add(R.id.container, OverlayFragment())
    addToBackStack("overlay")
}
```

⚠️ **Common Mistake:** Using `add()` everywhere leads to multiple fragments all receiving lifecycle events simultaneously, causing duplicate network calls and UI updates.

---

### Q7. What is `addToBackStack()`?

**Answer:**

`addToBackStack(name: String?)` tells the `FragmentManager` to add this transaction to the back stack so that the user can press Back to reverse it.

- **With** `addToBackStack`: Back press pops the fragment, revealing the previous one
- **Without** `addToBackStack`: Back press exits the activity (or pops to the previous activity back stack entry)

The `name` parameter is used to pop back to a specific transaction:

```kotlin
// Transaction 1
commit { replace(R.id.container, AFragment()); addToBackStack("A") }
// Transaction 2
commit { replace(R.id.container, BFragment()); addToBackStack("B") }
// Transaction 3
commit { replace(R.id.container, CFragment()); addToBackStack("C") }

// Pop all the way back to A (removes B and C from back stack)
supportFragmentManager.popBackStack("A", 0)

// Pop A and everything above it
supportFragmentManager.popBackStack("A", FragmentManager.POP_BACK_STACK_INCLUSIVE)
```

---

### Q8. Fragment lifecycle vs Activity lifecycle — what is different?

**Answer:**

Fragment has **additional lifecycle methods** that Activity does not:

```
onAttach()         ← Fragment is attached to its host Activity
onCreate()
onCreateView()     ← Inflate the fragment's layout (returns View)
onViewCreated()    ← View is ready; set up observers, click listeners HERE
onViewStateRestored()
onStart()
onResume()
--- (Fragment is active) ---
onPause()
onStop()
onSaveInstanceState()
onDestroyView()    ← View is destroyed, but fragment instance may survive!
onDestroy()
onDetach()         ← Fragment is detached from Activity
```

🎯 **Key Interview Point:** `onDestroyView()` is called when the fragment goes on the back stack. The fragment **instance** survives, but its **view** is destroyed. This is why `viewLifecycleOwner` exists.

---

### Q9. What is `viewLifecycleOwner` and why must you use it?

**Answer:**

`viewLifecycleOwner` is a `LifecycleOwner` tied to the fragment's **view** lifecycle (from `onCreateView` to `onDestroyView`), as opposed to `this` (the fragment instance) which lives longer.

**The problem with using `this`:**

```kotlin
// ❌ MEMORY LEAK — observer is attached to fragment instance lifecycle
// When fragment goes to back stack, view is destroyed but observer keeps updating the dead view
viewModel.data.observe(this) { data ->
    textView.text = data // textView is now null or stale!
}

// ✅ CORRECT — observer is removed when the view is destroyed
viewModel.data.observe(viewLifecycleOwner) { data ->
    textView.text = data
}
```

**Rule:** Always use `viewLifecycleOwner` for LiveData/Flow observers set up in `onViewCreated()`.

---

### Q10. How do fragments communicate with each other?

**Answer:**

**Modern approach (recommended):** Shared ViewModel

```kotlin
// Both fragments share the same ViewModel scoped to the Activity
class SharedViewModel : ViewModel() {
    val selectedItem = MutableStateFlow<Item?>(null)
}

// Fragment A — sends data
class FragmentA : Fragment() {
    private val viewModel: SharedViewModel by activityViewModels()
    
    fun onItemClicked(item: Item) {
        viewModel.selectedItem.value = item
    }
}

// Fragment B — receives data
class FragmentB : Fragment() {
    private val viewModel: SharedViewModel by activityViewModels()
    
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            viewModel.selectedItem.collect { item ->
                item?.let { updateUI(it) }
            }
        }
    }
}
```

**Other mechanisms:**
- **Fragment Result API** (for one-shot results between fragments on the same back stack):

```kotlin
// Sender (child/other fragment)
setFragmentResult("requestKey", bundleOf("selected_id" to itemId))

// Receiver (parent fragment or activity)
setFragmentResultListener("requestKey") { key, bundle ->
    val id = bundle.getInt("selected_id")
}
```

🔴 **Legacy (avoid):** Interface callbacks between fragment and activity.  
🔴 **Legacy (avoid):** Direct fragment-to-fragment method calls via `findFragmentById()`.

---

### Q11. Child Fragment Manager vs Support Fragment Manager?

**Answer:**

| | `childFragmentManager` | `supportFragmentManager` (parentFragmentManager) |
|---|---|---|
| **Scope** | Manages fragments nested inside *this* fragment | Manages fragments inside the Activity |
| **Back stack** | Back stack scoped to the parent fragment | Back stack scoped to the Activity |
| **Use case** | ViewPager2, nested tabs, dialog from within a fragment | Top-level navigation |

```kotlin
// Inside a Fragment — use childFragmentManager for nesting
childFragmentManager.commit {
    replace(R.id.childContainer, NestedFragment())
}

// Inside a Fragment — use parentFragmentManager or requireActivity().supportFragmentManager for activity-level navigation
parentFragmentManager.commit {
    replace(R.id.activityContainer, AnotherTopLevelFragment())
}
```

⚠️ **Common Mistake:** Using `requireActivity().supportFragmentManager` inside a fragment when you meant to use `childFragmentManager`. This causes nested fragments to be added to the wrong back stack.

---

## 3. Launch Modes & Back Stack

---

### Q12. Explain all 4 launch modes with real-world examples.

**Answer:**

**`standard` (default)**
- A new instance is **always** created, every time.
- Multiple instances of the same Activity can coexist in the same task.
- Use case: Any regular screen (product detail, article view).

```
Task: [A] → [A] → [B] → startActivity(A) → [A] → [A] → [B] → [A]
```

---

**`singleTop`**
- If an instance of this Activity is already at the **top** of the current task, a new instance is **NOT** created. Instead, `onNewIntent()` is called on the existing instance.
- If it's not at the top, a new instance is created normally.
- Use case: Notification tap that brings user to an already-open screen (search results, notifications list).

```kotlin
override fun onNewIntent(intent: Intent) {
    super.onNewIntent(intent)
    setIntent(intent) // Update the intent
    handleNotificationData(intent)
}
```

```
[A] → [B] → startActivity(B) → [A] → [B] (reused, onNewIntent called)
[A] → [B] → startActivity(A) → [A] → [B] → [A] (A is NOT at top, new instance created)
```

---

**`singleTask`**
- Only **one instance** of this Activity can exist in the entire task (or its own task if `taskAffinity` is set).
- If it already exists in the task, all Activities on top of it are **popped off** the back stack, and `onNewIntent()` is called.
- Use case: Main/Home screen of an app. App entry point from notifications or deep links.

```
[A] → [B] → [C] → startActivity(A with singleTask) → [A] (B and C are destroyed)
```

---

**`singleInstance`**
- Like `singleTask`, but the Activity is placed in its **own isolated task** — no other activities can be in the same task.
- Use case: Alarm clock screen, incoming call screen, system-level overlays.

```
Task 1: [A] → [B]
startActivity(C with singleInstance)
Task 2: [C]  ← C lives alone in its own task
```

---

### Q13. When would you use `singleTask`?

**Answer:**

Use `singleTask` for your app's **main entry point** or **hub screen** — a screen you want to always be at the root and unique in the task:

1. **Home screen navigation** — When a notification brings the user into the app, you want to clear the current navigation stack and land on Home.
2. **Authentication screen** — After login, clear the login stack so back doesn't go back to login.
3. **Deep link entry points** — An external deep link should land on a clean state.

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".HomeActivity"
    android:launchMode="singleTask" />
```

🎯 **Interview Alert:** Interviewers love to ask: "What happens if you start a `singleTask` activity that's already in the back stack?" Answer: All activities above it are destroyed, it receives `onNewIntent()`, and it comes to the foreground.

---

### Q14. What is Task Affinity?

**Answer:**

`taskAffinity` is a string that tells Android which **task** an Activity "prefers" to belong to. By default, all activities in an app share the same task affinity (the package name).

- Used in combination with `singleTask` or `FLAG_ACTIVITY_NEW_TASK` to control which task an Activity is launched into.
- Setting a **different** affinity allows an Activity to be launched into a separate task.

```xml
<activity
    android:name=".SharingActivity"
    android:taskAffinity="com.myapp.sharing"
    android:launchMode="singleTask" />
```

**Use case:** A "Share to App" screen that should always open in its own task, separate from the main app navigation.

---

## 4. Intents & PendingIntent

---

### Q15. Explicit vs Implicit Intents?

**Answer:**

**Explicit Intent** — You specify the exact target component (class):

```kotlin
// You know exactly where you're going
val intent = Intent(this, DetailActivity::class.java)
intent.putExtra("id", itemId)
startActivity(intent)
```

**Implicit Intent** — You specify an **action** and let the system find a component that can handle it:

```kotlin
// You don't know (or care) which app handles this
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("https://www.example.com"))
startActivity(intent) // Browser app or chooser dialog opens

val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "text/plain"
    putExtra(Intent.EXTRA_TEXT, "Check this out!")
}
startActivity(Intent.createChooser(shareIntent, "Share via"))
```

**Intent Resolution:** For implicit intents, Android matches `action`, `data` (URI/MIME type), and `category` against registered `<intent-filter>` entries in manifests.

---

### Q16. What is PendingIntent and when to use it?

**Answer:**

A `PendingIntent` is a **token** you give to another application (system, notification manager, alarm manager) that grants it the ability to perform an action **on your app's behalf**, at a future time, even when your app is not running.

**When to use:**
- **Notifications** — Tapping a notification should open your app
- **AlarmManager** — Trigger an action at a scheduled time
- **Widgets** — Button clicks on home screen widgets
- **Bubbles** — Chat bubble click behavior

```kotlin
// Creating a PendingIntent for a notification action
val openIntent = Intent(context, MainActivity::class.java).apply {
    flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TASK
}

val pendingIntent = PendingIntent.getActivity(
    context,
    REQUEST_CODE,
    openIntent,
    PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE // API 31+ requires mutability flag
)

NotificationCompat.Builder(context, CHANNEL_ID)
    .setContentTitle("New Message")
    .setContentIntent(pendingIntent)
    .build()
```

🎯 **API 31+ change:** `PendingIntent.FLAG_MUTABLE` or `PendingIntent.FLAG_IMMUTABLE` is **required** from Android 12 (API 31). Use `FLAG_IMMUTABLE` by default unless you need to modify the intent later (e.g., inline reply actions).

---

### Q17. How do you check if an implicit intent can be handled?

**Answer:**

Always check before calling `startActivity()` with an implicit intent, to avoid an `ActivityNotFoundException` crash:

```kotlin
// Method 1: resolveActivity (deprecated in API 30 without QUERY_ALL_PACKAGES)
val canHandle = intent.resolveActivity(packageManager) != null

// Method 2: queryIntentActivities (preferred from API 30+)
fun canHandleIntent(intent: Intent): Boolean {
    return if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        packageManager.queryIntentActivities(intent, PackageManager.ResolveInfoFlags.of(0)).isNotEmpty()
    } else {
        @Suppress("DEPRECATION")
        packageManager.queryIntentActivities(intent, 0).isNotEmpty()
    }
}

// Usage
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("geo:0,0?q=New+York"))
if (canHandleIntent(intent)) {
    startActivity(intent)
} else {
    Toast.makeText(this, "No maps app found", Toast.LENGTH_SHORT).show()
}
```

From **API 30+**, you also need to declare the packages you intend to query in `AndroidManifest.xml`:

```xml
<queries>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="geo" />
    </intent>
</queries>
```

---

### Q18. `startActivityForResult` vs Activity Result API?

**Answer:**

🔴 **Legacy:** `startActivityForResult()` + `onActivityResult()` — deprecated since Activity 1.2.0 / Fragment 1.3.0.

🟢 **Modern:** `ActivityResultLauncher` from the Activity Result API:

```kotlin
// Register BEFORE onStart (ideally as a field or in onCreate before fragment transactions)
private val pickImageLauncher = registerForActivityResult(
    ActivityResultContracts.GetContent()
) { uri: Uri? ->
    uri?.let { imageView.setImageURI(it) }
}

// Launch it from a button click
binding.btnPickImage.setOnClickListener {
    pickImageLauncher.launch("image/*")
}
```

**Built-in contracts:**
- `StartActivityForResult()` — generic, for custom intents
- `GetContent()` — file picker
- `TakePicture()` — camera
- `RequestPermission()` / `RequestMultiplePermissions()` — runtime permissions
- `OpenDocument()` — document picker

---

## 5. Threading & Coroutines

---

### Q19. What is ANR and how do you prevent it?

**Answer:**

**ANR (Application Not Responding)** is triggered by the Android system when:
- An **Activity or BroadcastReceiver** blocks the **main thread** for more than **5 seconds**
- A **Service** does not respond within **20 seconds** (foreground) / **200 seconds** (background)
- An **input event** (key, touch) is not handled within **5 seconds**

**How to prevent:**
1. **Never do I/O on the main thread** — use coroutines with `Dispatchers.IO`
2. **Never do heavy computation on the main thread** — use `Dispatchers.Default`
3. **Keep BroadcastReceiver.onReceive() fast** — delegate to `goAsync()` or WorkManager
4. **Use StrictMode** during development to detect violations early

```kotlin
// ❌ ANR risk
fun loadData() {
    val data = File("bigfile.json").readText() // Blocking I/O on main thread
    updateUI(data)
}

// ✅ Correct
fun loadData() {
    viewModelScope.launch {
        val data = withContext(Dispatchers.IO) {
            File("bigfile.json").readText()
        }
        updateUI(data) // Back on main thread
    }
}
```

---

### Q20. Explain Looper, Handler, and MessageQueue.

**Answer:**

These three work together to implement Android's **message-passing concurrency model**:

- **`MessageQueue`** — A queue that holds `Message` objects and `Runnable`s to be processed
- **`Looper`** — Runs an infinite loop on a thread, continuously pulling items from the `MessageQueue` and dispatching them
- **`Handler`** — The interface you use to **post messages/runnables** to a `Looper`'s queue. A Handler is always associated with one specific thread's Looper

```
Thread
  └── Looper  (runs the event loop)
        └── MessageQueue  (ordered queue of work)
              └── Messages / Runnables  ← Handler posts to here
```

```kotlin
// The main thread has a Looper set up automatically by the system
// You can post work to it from any thread:
val mainHandler = Handler(Looper.getMainLooper())
mainHandler.post {
    textView.text = "Updated from background thread"
}

// Creating a background thread with a Looper:
val handlerThread = HandlerThread("WorkerThread").also { it.start() }
val workerHandler = Handler(handlerThread.looper)
workerHandler.post {
    // This runs on the background HandlerThread
    processData()
}
```

🟢 **Modern alternative:** Use `lifecycleScope.launch { withContext(Dispatchers.IO) { } }` instead of raw Handlers for new code.

---

### Q21. Difference between Coroutine Dispatchers?

**Answer:**

| Dispatcher | Thread Pool | Use For |
|---|---|---|
| `Dispatchers.Main` | Main (UI) thread only | UI updates, observing LiveData/Flow, calling suspend funs that don't block |
| `Dispatchers.IO` | 64 threads (dynamic) | File I/O, network calls, database queries |
| `Dispatchers.Default` | CPU core count | CPU-intensive work: sorting, JSON parsing, image processing |
| `Dispatchers.Unconfined` | Caller's thread initially | Testing; avoid in production |

```kotlin
viewModelScope.launch {
    // Currently on Main
    _uiState.value = UiState.Loading

    val result = withContext(Dispatchers.IO) {
        // Now on IO thread pool
        repository.fetchData()
    }
    // Back on Main automatically after withContext
    _uiState.value = UiState.Success(result)
}
```

---

### Q22. Why was AsyncTask deprecated?

**Answer:**

`AsyncTask` was deprecated in **API 30** for several reasons:

1. **Memory leaks** — AsyncTask often held a reference to the Activity/Fragment, preventing GC when the Activity was destroyed
2. **Lifecycle unawareness** — Results could arrive after the Activity was destroyed, causing crashes or updating destroyed views
3. **Subtle behavior** — Serial execution by default (not parallel as many assumed), inconsistent across API versions
4. **Error handling** — No built-in mechanism for exception propagation
5. **Testing** — Difficult to test
6. **Replacement exists** — Kotlin Coroutines solve all of these problems cleanly

🟢 **Replace with:**
```kotlin
// AsyncTask.execute() → viewModelScope.launch { withContext(Dispatchers.IO) {} }
// doInBackground() → withContext(Dispatchers.IO) { }
// onPostExecute() → resume on Main after withContext
```

---

### Q23. `viewModelScope` vs `lifecycleScope` vs `GlobalScope`?

**Answer:**

| Scope | Tied To | Cancelled When | Use Case |
|---|---|---|---|
| `viewModelScope` | `ViewModel` | ViewModel is cleared (Activity/Fragment destroyed) | Most ViewModel async work |
| `lifecycleScope` | `Activity` / `Fragment` | Lifecycle reaches `DESTROYED` | UI-driven coroutines, collecting flows in UI |
| `GlobalScope` | Application process | App process is killed | ⚠️ Avoid — not lifecycle aware, can cause leaks |

```kotlin
// ViewModel — use viewModelScope
class MyViewModel : ViewModel() {
    fun loadData() {
        viewModelScope.launch {
            val data = repository.getData()
            _state.value = data
        }
    }
}

// Fragment — use viewLifecycleOwner.lifecycleScope for collecting flows
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    viewLifecycleOwner.lifecycleScope.launch {
        repeatOnLifecycle(Lifecycle.State.STARTED) {
            viewModel.state.collect { state ->
                render(state)
            }
        }
    }
}
```

🎯 **`repeatOnLifecycle`** is preferred over `launchWhenStarted` (which is now deprecated) because it cancels and restarts the collection block automatically when the lifecycle state changes.

---

## 6. Services & Background Work

---

### Q24. What are the types of Services in Android?

**Answer:**

**1. Started Service** (🔴 rarely recommended now)
- Started with `startService()`
- Runs independently of the component that started it
- Must stop itself (`stopSelf()`) or be stopped explicitly

**2. Foreground Service** (🟢 when you need long-running user-visible work)
- A Started Service with a persistent notification
- Less likely to be killed by the system
- Required for audio playback, location tracking, file uploads

**3. Bound Service**
- Components bind to it with `bindService()` for IPC
- Lives only as long as at least one client is bound
- Use for: providing an API to other components/apps

**4. IntentService** (🔴 Deprecated API 30)
- Handled requests on a worker thread, auto-stopped when done
- Replaced by `WorkManager` or coroutines

---

### Q25. Does Service run on the main thread?

**Answer:**

**Yes.** By default, a `Service` runs on the **main (UI) thread** of the process. If you do blocking work in `onStartCommand()` or service callbacks, you will cause ANR.

You must explicitly move work off the main thread:

```kotlin
class MyService : Service() {
    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.IO)

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        serviceScope.launch {
            doHeavyWork()
            stopSelf(startId)
        }
        return START_NOT_STICKY
    }

    override fun onDestroy() {
        super.onDestroy()
        serviceScope.cancel() // Cancel all coroutines when service is destroyed
    }
}
```

---

### Q26. What is `START_STICKY` vs `START_NOT_STICKY` vs `START_REDELIVER_INTENT`?

**Answer:**

These return values from `onStartCommand()` tell the system what to do if the service is killed:

| Return Value | Behavior |
|---|---|
| `START_STICKY` | Service is recreated after being killed, but `intent` is `null`. Use for services that should keep running (e.g., music player service). |
| `START_NOT_STICKY` | Service is **NOT** recreated after being killed. Use for services that can be restarted on demand (e.g., a sync triggered by data change). |
| `START_REDELIVER_INTENT` | Service is recreated AND the last intent is redelivered. Use when it's critical that the work completes (e.g., file upload). |

```kotlin
override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
    // intent can be null if START_STICKY and service was killed
    intent ?: return START_STICKY
    
    processCommand(intent)
    return START_NOT_STICKY
}
```

---

### Q27. When to use a Foreground Service?

**Answer:**

Use a Foreground Service when:
- Work is **user-initiated**, actively **in progress**, and the user **knows about it**
- The operation must survive the app being backgrounded
- Examples: GPS navigation, music playback, ongoing file download, active VoIP call

```kotlin
class LocationTrackingService : Service() {

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = createNotification()
        startForeground(NOTIFICATION_ID, notification) // Must call within 5s of start
        
        startTracking()
        return START_STICKY
    }

    private fun createNotification(): Notification {
        val channel = NotificationChannel(CHANNEL_ID, "Tracking", NotificationManager.IMPORTANCE_LOW)
        getSystemService(NotificationManager::class.java).createNotificationChannel(channel)
        
        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle("Location active")
            .setSmallIcon(R.drawable.ic_location)
            .build()
    }
}
```

**Manifest declaration (API 34+ requires `foregroundServiceType`):**
```xml
<service
    android:name=".LocationTrackingService"
    android:foregroundServiceType="location" />
```

---

### Q28. WorkManager vs Service — when to use which?

**Answer:**

| | `WorkManager` | Foreground Service |
|---|---|---|
| **Work type** | Deferrable, guaranteed background work | User-visible, immediate, ongoing work |
| **Survives process death** | ✅ Yes (persisted to database) | ❌ No (if process dies, service dies) |
| **User visible** | No notification required | Persistent notification required |
| **Scheduling** | Constraints (network, charging), delays, chains | N/A |
| **Examples** | Sync, log upload, image compression | Music playback, navigation, download |

**WorkManager example:**
```kotlin
val uploadWork = OneTimeWorkRequestBuilder<UploadWorker>()
    .setConstraints(Constraints.Builder()
        .setRequiredNetworkType(NetworkType.CONNECTED)
        .build())
    .setBackoffCriteria(BackoffPolicy.LINEAR, 30, TimeUnit.SECONDS)
    .build()

WorkManager.getInstance(context).enqueue(uploadWork)
```

---

### Q29. What is a Bound Service?

**Answer:**

A Bound Service allows other components to **bind** to it and interact with it through a defined interface. It lives as long as at least one component is bound to it.

```kotlin
class MusicService : Service() {
    private val binder = MusicBinder()

    inner class MusicBinder : Binder() {
        fun getService(): MusicService = this@MusicService
    }

    override fun onBind(intent: Intent): IBinder = binder

    fun play(song: Song) { /* ... */ }
    fun pause() { /* ... */ }
}

// In Activity
class PlayerActivity : AppCompatActivity() {
    private var musicService: MusicService? = null
    
    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, service: IBinder) {
            val binder = service as MusicService.MusicBinder
            musicService = binder.getService()
        }
        override fun onServiceDisconnected(name: ComponentName) {
            musicService = null
        }
    }

    override fun onStart() {
        super.onStart()
        bindService(Intent(this, MusicService::class.java), connection, Context.BIND_AUTO_CREATE)
    }

    override fun onStop() {
        super.onStop()
        unbindService(connection)
    }
}
```

---

## 7. Broadcast Receivers

---

### Q30. Static vs Dynamic Registration?

**Answer:**

**Static (Manifest) Registration:**
```xml
<receiver android:name=".BootReceiver" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.BOOT_COMPLETED" />
    </intent-filter>
</receiver>
```
- App does not need to be running to receive broadcasts
- ⚠️ **API 26+:** Most implicit broadcasts can no longer be received via static registration (background execution limits)

**Dynamic (Code) Registration:**
```kotlin
class MyActivity : AppCompatActivity() {
    private val receiver = NetworkChangeReceiver()

    override fun onResume() {
        super.onResume()
        val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
        registerReceiver(receiver, filter)
    }

    override fun onPause() {
        super.onPause()
        unregisterReceiver(receiver) // MUST unregister to avoid leaks
    }
}
```
- Only receives broadcasts while registered
- Not subject to API 26 background limits (if registered in a running component)
- Must manually unregister (Activity/Fragment lifecycle appropriate)

---

### Q31. Normal vs Ordered Broadcasts?

**Answer:**

**Normal Broadcast** (`sendBroadcast()`):
- Delivered to all matching receivers **simultaneously** (in undefined order)
- Receivers cannot pass results to each other

**Ordered Broadcast** (`sendOrderedBroadcast()`):
- Delivered to receivers **one at a time**, in order of `android:priority`
- Each receiver can pass data to the next, or **abort** the broadcast
- Use case: SMS interception (pre-API 19), priority-based event handling

```kotlin
// Sending an ordered broadcast
val intent = Intent("com.example.ORDERED_ACTION")
sendOrderedBroadcast(intent, null)

// Receiving — high priority receiver can abort
class HighPriorityReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        // Process and optionally abort
        if (shouldAbort()) {
            abortBroadcast() // Lower priority receivers won't get it
        }
        resultData = "processed by high priority"
    }
}
```

---

### Q32. Why is LocalBroadcastManager deprecated?

**Answer:**

`LocalBroadcastManager` was deprecated in **`androidx.localbroadcastmanager:localbroadcastmanager:1.1.0`** (2020).

**Reason:** It was a workaround for in-process communication. Kotlin's `SharedFlow`/`StateFlow` (or even `LiveData`) are better alternatives — they are lifecycle-aware, type-safe, coroutine-native, and don't require string-based action routing.

```kotlin
// ❌ Legacy LocalBroadcastManager
LocalBroadcastManager.getInstance(context)
    .sendBroadcast(Intent("DATA_UPDATED"))

// ✅ Modern replacement — SharedFlow in a shared ViewModel or singleton
object EventBus {
    private val _events = MutableSharedFlow<AppEvent>()
    val events: SharedFlow<AppEvent> = _events
    
    suspend fun emit(event: AppEvent) = _events.emit(event)
}
```

---

### Q33. What are the API 26 background broadcast limitations?

**Answer:**

Starting with **Android 8.0 (API 26)**, apps targeting API 26+ **cannot register static (manifest) receivers** for most implicit broadcasts. This was done to reduce background battery drain.

**Exceptions (still allowed via static registration):**
- `BOOT_COMPLETED`
- `LOCKED_BOOT_COMPLETED`
- `ACTION_PACKAGE_ADDED` (for your own package)
- SMS/MMS intents (for default SMS apps)
- Bluetooth and push notification intents

**How to handle restricted broadcasts:**
1. Use **dynamic registration** in a running component
2. Use **WorkManager** to schedule deferred work
3. Use **Firebase Cloud Messaging (FCM)** for push-triggered work

---

## 8. MVVM & ViewModel

---

### Q34. Explain MVVM layers and their responsibilities.

**Answer:**

```
UI Layer (View)          ← observes state from ViewModel
    │
    ▼
ViewModel Layer          ← holds/exposes UI state, handles UI events, calls domain/data layer
    │
    ▼
Domain Layer (optional)  ← UseCases: single-purpose business logic operations
    │
    ▼
Data Layer               ← Repository: single source of truth, abstracts data sources
    │
    ├── Remote: Retrofit/API
    └── Local: Room/DataStore
```

| Layer | Knows About | Does NOT Know About |
|---|---|---|
| View | ViewModel | Repository, database, network |
| ViewModel | Repository / UseCase | Activity/Fragment context, Views |
| Repository | Data sources | ViewModels, UI |
| Data Sources | Their own API | Everything above |

---

### Q35. How does ViewModel survive configuration changes?

**Answer:**

The `ViewModel` is stored in a `ViewModelStore`, which is attached to the `Activity`'s `NonConfigurationInstance` — a special mechanism in `ComponentActivity` that allows data to survive the destroy-recreate cycle of a configuration change.

1. On configuration change, `onRetainNonConfigurationInstance()` is called (internally by `ComponentActivity`)
2. The `ViewModelStore` is saved into this non-configuration instance
3. The Activity is destroyed, then a new instance is created
4. `getLastNonConfigurationInstance()` retrieves the `ViewModelStore`
5. `ViewModelProvider` finds the existing ViewModel in the store — same instance returned

```kotlin
// This is the SAME ViewModel instance across rotation
class MyActivity : AppCompatActivity() {
    private val viewModel: MyViewModel by viewModels()
    // viewModels() delegate uses ViewModelProvider internally
}
```

---

### Q36. Does ViewModel survive process death?

**Answer:**

**No.** When the Android system kills the process (due to low memory), all in-memory data is lost, including ViewModels. When the user returns to the app, the Activity is recreated from scratch with `savedInstanceState`.

**Solution: `SavedStateHandle`**

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val repository: SearchRepository
) : ViewModel() {
    
    // Automatically persisted across process death
    private val _query = savedStateHandle.getStateFlow("query", "")
    val query: StateFlow<String> = _query
    
    fun onQueryChanged(q: String) {
        savedStateHandle["query"] = q
    }
}
```

`SavedStateHandle` uses `onSaveInstanceState` internally — so it has the same **1 MB Bundle limit** and only supports `Parcelable`-compatible types.

---

### Q37. LiveData vs StateFlow?

**Answer:**

| | `LiveData` | `StateFlow` |
|---|---|---|
| **Library** | `androidx.lifecycle` | `kotlinx.coroutines` |
| **Lifecycle awareness** | Built-in (auto stops in background) | Requires `repeatOnLifecycle` |
| **Initial value** | Optional (can be null) | **Required** |
| **Kotlin idiomatic** | ❌ Java-origin | ✅ Kotlin-first |
| **Transformation** | `map`, `switchMap` (limited) | All Flow operators |
| **Testing** | Requires `InstantTaskExecutorRule` | `Turbine` library |
| **Nullability** | Nullable by default | Can be non-null |

🟢 **Recommendation:** Use `StateFlow` (or `MutableStateFlow`) in new code. Use `LiveData` only if working with an older codebase or needing `MediatorLiveData`.

```kotlin
// ViewModel with StateFlow
private val _uiState = MutableStateFlow<UiState>(UiState.Idle)
val uiState: StateFlow<UiState> = _uiState.asStateFlow()

// Collecting in Fragment
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state -> render(state) }
    }
}
```

---

### Q38. What is SingleLiveEvent / Channel for one-time events?

**Answer:**

`StateFlow` and `LiveData` hold the **last emitted value** and replay it to new collectors. This is wrong for one-time events like navigation, showing a Toast, or a Snackbar.

**The problem:**

```kotlin
// ❌ Navigation event stored in StateFlow will re-navigate on rotation!
val navigateToDetail = MutableStateFlow<String?>(null)
```

**Solutions:**

**1. `Channel` with `receiveAsFlow()` (recommended):**
```kotlin
// ViewModel
private val _events = Channel<UiEvent>(Channel.BUFFERED)
val events = _events.receiveAsFlow()

fun onButtonClick() {
    viewModelScope.launch { _events.send(UiEvent.NavigateToDetail(id)) }
}

// Fragment
viewLifecycleOwner.lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.events.collect { event ->
            when (event) {
                is UiEvent.NavigateToDetail -> findNavController().navigate(...)
                is UiEvent.ShowToast -> Toast.makeText(...)
            }
        }
    }
}
```

**2. `SharedFlow` with `replay = 0`:**
```kotlin
private val _events = MutableSharedFlow<UiEvent>()
val events: SharedFlow<UiEvent> = _events
```

---

### Q39. How to share a ViewModel between fragments?

**Answer:**

Scope the ViewModel to the **Activity** using `activityViewModels()`:

```kotlin
// Both fragments use the exact same ViewModel instance
class FragmentA : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
}

class FragmentB : Fragment() {
    private val sharedViewModel: SharedViewModel by activityViewModels()
}
```

**With Navigation Component** — scope to the nav graph:

```kotlin
// ViewModel scoped to a NavGraph, shared only within that graph
class FragmentA : Fragment() {
    private val sharedViewModel: CheckoutViewModel by navGraphViewModels(R.id.checkout_graph)
}
```

---

## 9. Memory & Performance

---

### Q40. What are the most common causes of memory leaks in Android?

**Answer:**

| Leak Pattern | Cause | Fix |
|---|---|---|
| **Static Activity/View reference** | `companion object { var activity: Activity? }` | Never store Activity in static fields |
| **Inner class Handler** | Non-static inner class Handler holds outer Activity reference | Use `WeakReference<Activity>` or static inner class |
| **Unregistered receiver/listener** | Registered but never unregistered | Unregister in paired lifecycle method |
| **Observer not removed** | LiveData observer with `this` (not `viewLifecycleOwner`) | Use `viewLifecycleOwner` |
| **Context leak in singleton** | Singleton holds `Activity` Context | Use `applicationContext` in singletons |
| **Anonymous Runnable in Handler** | `handler.postDelayed({ ... }, delay)` — lambda captures Activity | Use `removeCallbacks()` in `onDestroy()` |
| **Bitmap not recycled** | Old bitmap handling without pooling | Use Glide/Coil (handle this for you) |

---

### Q41. How to detect memory leaks?

**Answer:**

1. **LeakCanary** — Add as a debug dependency; automatically detects and reports leaks with full stack traces:
```kotlin
debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
// No code needed — auto-detects leaks
```

2. **Android Studio Memory Profiler** — Record allocations, take heap dumps, analyze retained objects

3. **`adb shell dumpsys meminfo <package>`** — Command-line memory analysis

4. **StrictMode** — Detect leaked SQLite objects, closeable objects:
```kotlin
StrictMode.setVmPolicy(
    StrictMode.VmPolicy.Builder()
        .detectLeakedSqlLiteObjects()
        .detectLeakedClosableObjects()
        .penaltyLog()
        .build()
)
```

---

### Q42. What is `WeakReference` and when to use it?

**Answer:**

A `WeakReference<T>` holds a reference to an object that does **not prevent** the Garbage Collector from collecting it. When GC runs and only weak references point to the object, it is collected and `weakRef.get()` returns `null`.

```kotlin
// Classic use case: Handler that must outlive the Activity
class MyHandler(activity: MyActivity) : Handler(Looper.getMainLooper()) {
    private val activityRef = WeakReference(activity)

    override fun handleMessage(msg: Message) {
        val activity = activityRef.get() ?: return // Activity may be GC'd
        activity.updateUI()
    }
}
```

**When to use:** When you need a reference to an object but don't want to prevent its collection (callback receivers, caches, long-lived objects referencing short-lived ones).

---

### Q43. What is `onTrimMemory`?

**Answer:**

`onTrimMemory(level: Int)` is called by the system when it detects the device is running low on memory. Your app should release non-critical cached resources in response.

```kotlin
override fun onTrimMemory(level: Int) {
    super.onTrimMemory(level)
    when (level) {
        ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN -> {
            // App moved to background — release UI-only caches
            imageCache.evictAll()
        }
        ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW,
        ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL -> {
            // System is running very low on memory — release everything possible
            clearAllCaches()
        }
        ComponentCallbacks2.TRIM_MEMORY_COMPLETE -> {
            // App will likely be killed next — release everything
            clearAllCaches()
        }
    }
}
```

---

## 10. Navigation Component

---

### Q44. `NavController` vs `NavHostFragment`?

**Answer:**

- **`NavHostFragment`** — A container `Fragment` in your layout that serves as the navigation "host". It provides the area where destination fragments are swapped in/out.

- **`NavController`** — The brain of Navigation Component. It reads the `NavGraph` (XML or code-built), knows the current destination, and performs navigate/popBack operations.

```xml
<!-- activity_main.xml -->
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/nav_host_fragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    app:navGraph="@navigation/nav_graph"
    app:defaultNavHost="true" />
```

```kotlin
// Get NavController from a Fragment
val navController = findNavController()

// Get NavController from an Activity
val navController = findNavController(R.id.nav_host_fragment)

// Navigate
navController.navigate(R.id.action_homeFragment_to_detailFragment)
```

---

### Q45. What is Safe Args?

**Answer:**

Safe Args is a Gradle plugin that generates type-safe Kotlin classes for navigating with arguments, eliminating the need for string-keyed Bundle access and preventing type mismatches.

```kotlin
// nav_graph.xml — argument defined in XML
// <argument android:name="itemId" app:argType="integer" />

// Generated by Safe Args — type-safe navigation action
val action = HomeFragmentDirections.actionHomeToDetail(itemId = 42)
findNavController().navigate(action)

// In DetailFragment — type-safe argument retrieval
val args: DetailFragmentArgs by navArgs()
val itemId: Int = args.itemId
```

---

### Q46. `navigateUp()` vs `popBackStack()`?

**Answer:**

| | `navigateUp()` | `popBackStack()` |
|---|---|---|
| **Effect** | Navigates up the app's logical hierarchy (like the Up button/back arrow in the toolbar) | Pops the top of the back stack |
| **Difference** | If back stack is empty, navigates to parent Activity defined in manifest | If back stack is empty, does nothing (returns false) |
| **Use case** | Toolbar Up button | Programmatic back navigation within the same task |

```kotlin
// Toolbar Up button — use navigateUp
toolbar.setNavigationOnClickListener {
    navController.navigateUp()
}

// Programmatic navigation — use popBackStack
button.setOnClickListener {
    findNavController().popBackStack()
}

// Pop to a specific destination
findNavController().popBackStack(R.id.homeFragment, inclusive = false)
```

---

## 11. Dependency Injection with Hilt

---

### Q47. What is Dependency Injection and why does it matter?

**Answer:**

Dependency Injection (DI) is a design pattern where a class receives its dependencies from the outside (from a framework or caller) rather than creating them internally.

**Without DI:**
```kotlin
class UserRepository {
    // Tightly coupled — impossible to swap in tests
    private val api = RetrofitFactory.create(UserApi::class.java)
    private val db = Room.databaseBuilder(...).build()
}
```

**With DI (Hilt):**
```kotlin
class UserRepository @Inject constructor(
    private val api: UserApi,       // Provided by Hilt
    private val userDao: UserDao    // Provided by Hilt
)
```

**Benefits:**
- **Testability** — Easily swap real dependencies with fakes/mocks
- **Separation of concerns** — Classes don't know how their dependencies are built
- **Reusability** — Same dependency instance shared across the app
- **Maintainability** — Change implementation in one place

---

### Q48. `@Singleton` vs `@ActivityScoped` vs `@ViewModelScoped`?

**Answer:**

These are Hilt **scoping annotations** that control how long a provided dependency lives:

| Annotation | Scope Component | Lifetime |
|---|---|---|
| `@Singleton` | `SingletonComponent` | App process lifetime |
| `@ActivityRetainedScoped` | `ActivityRetainedComponent` | Survives config changes, destroyed when Activity is destroyed |
| `@ActivityScoped` | `ActivityComponent` | Activity lifetime (destroyed on config change too) |
| `@ViewModelScoped` | `ViewModelComponent` | ViewModel lifetime |
| `@FragmentScoped` | `FragmentComponent` | Fragment lifetime |

```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    @Singleton // One instance for the app's lifetime
    fun provideRetrofit(): Retrofit = Retrofit.Builder()
        .baseUrl(BASE_URL)
        .build()
}
```

---

### Q49. `@Provides` vs `@Binds`?

**Answer:**

Both are used in `@Module` classes to tell Hilt how to create instances:

**`@Provides`** — Used when you **control the construction** (third-party classes, builder patterns):
```kotlin
@Provides
@Singleton
fun provideOkHttpClient(): OkHttpClient {
    return OkHttpClient.Builder()
        .addInterceptor(HttpLoggingInterceptor())
        .build()
}
```

**`@Binds`** — Used to **bind an interface to its implementation**. More efficient (generates less code — no factory class created):
```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    
    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl // Hilt knows how to create this via @Inject constructor
    ): UserRepository // This is the interface type
}
```

**Rule of thumb:** Use `@Binds` when the class has `@Inject constructor`. Use `@Provides` for everything else.

---

### Q50. How does `@HiltViewModel` work?

**Answer:**

`@HiltViewModel` marks a ViewModel for Hilt injection. It causes Hilt to generate a `ViewModelFactory` that knows how to create that ViewModel with its injected dependencies.

```kotlin
@HiltViewModel
class UserViewModel @Inject constructor(
    private val repository: UserRepository,
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {
    // repository is provided by Hilt
    // savedStateHandle is automatically provided by Hilt for @HiltViewModel
}
```

```kotlin
// In Activity — annotate with @AndroidEntryPoint
@AndroidEntryPoint
class UserActivity : AppCompatActivity() {
    private val viewModel: UserViewModel by viewModels()
    // Hilt creates UserViewModel with the injected UserRepository
}
```

The chain: `@HiltAndroidApp` (Application) → `@AndroidEntryPoint` (Activity/Fragment) → `@HiltViewModel` (ViewModel) → `@Inject constructor` dependencies.

---

## 12. Data Persistence

---

### Q51. SharedPreferences vs DataStore vs Room — when to use each?

**Answer:**

| | SharedPreferences | Preferences DataStore | Proto DataStore | Room |
|---|---|---|---|---|
| **Data type** | Key-value (primitives) | Key-value (typed) | Custom Protobuf | Relational (tables) |
| **Thread safety** | ❌ Not safe on main thread | ✅ Coroutine-based | ✅ Coroutine-based | ✅ Suspend functions |
| **Structured data** | ❌ No | ❌ No | ✅ Yes | ✅ Yes (complex queries) |
| **Use case** | Settings, flags (legacy) | User preferences, settings | Complex typed preferences | User data, offline cache |

```kotlin
// Preferences DataStore — modern SharedPreferences replacement
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

object PreferencesKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
}

// Write
suspend fun setDarkMode(enabled: Boolean) {
    context.dataStore.edit { prefs ->
        prefs[PreferencesKeys.DARK_MODE] = enabled
    }
}

// Read (returns Flow)
val darkMode: Flow<Boolean> = context.dataStore.data
    .map { prefs -> prefs[PreferencesKeys.DARK_MODE] ?: false }
```

---

*End of Interview Cheat Sheet*

---

> 💡 **Last-minute tip:** In every interview, after answering a question, briefly mention the modern/recommended approach even if the interviewer asked about a legacy API. It signals that you stay current.

---

*Android Fundamentals: The Complete Developer & Interview Handbook | Appendix A*
