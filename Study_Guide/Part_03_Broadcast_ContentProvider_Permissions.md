# Android Fundamentals Master Guide
# Part 3: Broadcast Receivers · Content Providers · Runtime Permissions

> **Chapters 9 – 11** | Production-Quality Reference for Developers & Interview Candidates

---

## Table of Contents

- [Chapter 9: Broadcast Receivers](#chapter-9-broadcast-receivers)
- [Chapter 10: Content Providers & ContentResolver](#chapter-10-content-providers--contentresolver)
- [Chapter 11: Runtime Permissions](#chapter-11-runtime-permissions)

---

# Chapter 9: Broadcast Receivers

---

## Concept

A **BroadcastReceiver** is one of Android's four core application components. It is a component that **listens for and reacts to system-wide or application-wide broadcast messages** (called `Intent`s). These broadcasts can originate from:

- **The Android system itself** — battery low, SMS received, device booted, network connectivity changed, airplane mode toggled, time zone changed, etc.
- **Other applications** — any app can send a custom broadcast.
- **Your own application** — for internal decoupled communication (though modern alternatives are preferred).

A BroadcastReceiver does not have a user interface. It is a purely logical gateway: it wakes up, handles the event, and exits. This ephemeral nature makes it fundamentally different from Activities or Services.

```
BroadcastReceiver = Event-Driven Listener + Stateless Handler + Zero UI
```

---

## Why It Exists

Android is a multi-application, multi-process environment. Consider these problems:

1. **System events need to reach interested apps without tight coupling.** When battery drops below 15%, Android should not need to know which apps care — it just announces the fact and all registered listeners respond.
2. **Inter-process communication (IPC) must be decoupled.** An SMS app should not need to call your OTP-reading app directly.
3. **App components need a low-overhead trigger mechanism.** Starting a full Service for a momentary event (like screen turning on) is wasteful. BroadcastReceiver provides a short-lived handler.

BroadcastReceiver solves the **pub-sub problem** at the OS level using `Intent`s as messages and the Android framework as the message broker.

---

## Internal Working

### The Broadcast Delivery Pipeline

When `sendBroadcast(intent)` is called:

1. The calling process makes a **Binder IPC call** to **ActivityManagerService (AMS)** in the `system_server` process.
2. AMS looks up all registered receivers for the matching `Intent` action, data, category, and scheme.
3. For **static receivers** (declared in `AndroidManifest.xml`): AMS may need to start the target process if it isn't already running.
4. For **dynamic receivers** (registered via `registerReceiver()`): the record already exists in AMS's in-memory list.
5. AMS dispatches the `Intent` to each matching receiver by calling `ActivityThread.handleReceiver()` on the target process's main thread.
6. `onReceive()` is called on the **main (UI) thread**.
7. The receiver process signals AMS when `onReceive()` returns. From this point, the process is eligible for background killing.

### BroadcastQueue

AMS maintains two queues internally:
- **Foreground BroadcastQueue** — for broadcasts sent with `Intent.FLAG_RECEIVER_FOREGROUND`
- **Background BroadcastQueue** — for all others (lower priority, longer timeout)

Each queue is processed sequentially for **ordered broadcasts** and in parallel for **normal broadcasts**.

### Static vs. Dynamic Registration Internals

| Aspect | Static (Manifest) | Dynamic (Runtime) |
|---|---|---|
| Registered by | `PackageManagerService` at install | `ActivityManagerService` at runtime via `registerReceiver()` |
| Stored in | PM database + AMS cache | AMS `ReceiverList` in-memory map |
| Process lifecycle | AMS can start the process cold | Tied to registering component's lifecycle |
| API 26+ implicit | Most blocked | Allowed |

---

## Lifecycle / Flow (ASCII Diagrams)

### Normal Broadcast Flow

```
App / System
    |
    |  sendBroadcast(intent)
    v
ActivityManagerService (system_server)
    |
    |  Looks up matching receivers
    |
    +-----> Receiver A (onReceive called on Main Thread)
    |
    +-----> Receiver B (onReceive called on Main Thread)
    |
    +-----> Receiver C (onReceive called on Main Thread)
    |
   All receive concurrently (no guaranteed order, no result passing)
```

### Ordered Broadcast Flow

```
App / System
    |
    |  sendOrderedBroadcast(intent, permission)
    v
ActivityManagerService (system_server)
    |
    |  Sort receivers by android:priority (highest first)
    |
    v
Receiver A (priority=100) ──> onReceive()
    |  [can modify result, or call abortBroadcast()]
    |  [if aborted: STOP. No further delivery]
    v
Receiver B (priority=50) ──> onReceive()
    |  [can read/modify result set by A]
    v
Receiver C (priority=0) ──> onReceive()
    |
    v
Result Receiver (final callback, always receives even if aborted)
```

### Dynamic Registration Lifecycle

```
Activity/Fragment onCreate()
        |
        v
    register:
    registerReceiver(receiver, intentFilter)
        |
        | [App is alive, receiver is active]
        |
    onReceive() called zero or more times
        |
        v
Activity/Fragment onDestroy() / onStop() / onPause()
        |
        v
    unregister:
    unregisterReceiver(receiver)
        |
        v
    [Receiver is dead. No more callbacks.]
```

### goAsync() Flow (Brief Extension)

```
onReceive() called
    |
    +---> pendingResult = goAsync()
    |         |
    |         | [Main thread returns. AMS sees onReceive() "done".]
    |         | [Process gets a brief window — ~10 more seconds]
    |         v
    |     Background thread does work
    |         |
    |         v
    |     pendingResult.finish()
    |         |
    |         v
    |     [Now truly done. Process eligible for kill]
```

---

## Real World Example

### Scenario: Spam SMS Interception (Antivirus / SMS Filter App)

A security app registers a high-priority ordered broadcast receiver for incoming SMS. It intercepts the message, checks it against a spam database, and if spam, aborts the broadcast so the default SMS app never sees it.

```kotlin
// AndroidManifest.xml
// <uses-permission android:name="android.permission.RECEIVE_SMS" />
// <receiver
//     android:name=".SmsFilterReceiver"
//     android:exported="true"
//     android:permission="android.permission.BROADCAST_SMS">
//     <intent-filter android:priority="1000">
//         <action android:name="android.provider.Telephony.SMS_RECEIVED" />
//     </intent-filter>
// </receiver>
```

```kotlin
class SmsFilterReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        if (intent.action != Telephony.Sms.Intents.SMS_RECEIVED_ACTION) return

        val messages = Telephony.Sms.Intents.getMessagesFromIntent(intent)
        val fullBody = messages.joinToString(separator = "") { it.messageBody }
        val sender = messages.firstOrNull()?.originatingAddress ?: return

        if (SpamDatabase.isSpam(sender, fullBody)) {
            // Block delivery to lower-priority receivers (e.g., default SMS app)
            abortBroadcast()
            // Optionally notify the user via a notification
        }
    }
}
```

### Scenario: Network Change Listener (Dynamic Registration)

```kotlin
class NetworkAwareActivity : AppCompatActivity() {

    private val networkReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            val isConnected = !intent.getBooleanExtra(
                ConnectivityManager.EXTRA_NO_CONNECTIVITY, false
            )
            updateUI(isConnected)
        }
    }

    override fun onStart() {
        super.onStart()
        val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
        registerReceiver(networkReceiver, filter)
    }

    override fun onStop() {
        super.onStop()
        unregisterReceiver(networkReceiver)
    }

    private fun updateUI(connected: Boolean) {
        // Update UI based on connectivity state
    }
}
```

> **Note:** `ConnectivityManager.CONNECTIVITY_ACTION` is deprecated. Modern code should use `ConnectivityManager.registerNetworkCallback()` instead.

---

## Common Mistakes

### 1. Doing Heavy Work in `onReceive()`
`onReceive()` runs on the main thread with a hard timeout of **10 seconds**. Doing network calls, database queries, or file I/O here causes ANR.

```kotlin
// ❌ WRONG - Network call on main thread
override fun onReceive(context: Context, intent: Intent) {
    val result = URL("https://api.example.com/data").readText() // ANR!
    processResult(result)
}

// ✅ CORRECT - Delegate to WorkManager
override fun onReceive(context: Context, intent: Intent) {
    val workRequest = OneTimeWorkRequestBuilder<SyncWorker>().build()
    WorkManager.getInstance(context).enqueue(workRequest)
}
```

### 2. Forgetting to Unregister Dynamic Receivers

```kotlin
// ❌ WRONG - Registered but never unregistered → memory leak
class LeakyActivity : AppCompatActivity() {
    private val receiver = MyReceiver()
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        registerReceiver(receiver, IntentFilter("MY_ACTION"))
        // No unregisterReceiver → Activity leaks!
    }
}
```

### 3. Assuming Static Receivers Work for All Broadcasts on API 26+

Most **implicit** broadcasts are blocked for static receivers on API 26+. Developers who migrate from older APIs often miss this.

```kotlin
// ❌ WRONG on API 26+ (manifest receiver for implicit broadcast)
// <receiver android:name=".ConnectivityReceiver">
//     <intent-filter>
//         <action android:name="android.net.conn.CONNECTIVITY_CHANGE"/> <!-- BLOCKED -->
//     </intent-filter>
// </receiver>

// ✅ CORRECT - Register dynamically at runtime
registerReceiver(receiver, IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION))
```

### 4. Missing `android:exported` on API 31+

Android 12 (API 31) requires every receiver with an `<intent-filter>` to explicitly declare `android:exported`.

```xml
<!-- ❌ WRONG on API 31+ - will crash at install -->
<receiver android:name=".MyReceiver">
    <intent-filter>
        <action android:name="com.example.MY_ACTION"/>
    </intent-filter>
</receiver>

<!-- ✅ CORRECT -->
<receiver
    android:name=".MyReceiver"
    android:exported="false">
    <intent-filter>
        <action android:name="com.example.MY_ACTION"/>
    </intent-filter>
</receiver>
```

### 5. Using `LocalBroadcastManager` (Deprecated)

```kotlin
// ❌ DEPRECATED
LocalBroadcastManager.getInstance(context).sendBroadcast(intent)

// ✅ MODERN REPLACEMENT
// Use StateFlow, SharedFlow, or LiveData for in-process events
class EventBus {
    private val _events = MutableSharedFlow<AppEvent>()
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()
    suspend fun emit(event: AppEvent) = _events.emit(event)
}
```

---

## Memory Leak / Performance Concerns

### Memory Leak: Unregistered Dynamic Receiver

Every call to `registerReceiver()` registers a `ReceiverList` entry in **AMS**. If `unregisterReceiver()` is never called:
- The `BroadcastReceiver` object is held in AMS memory (another process).
- The registering `Context` (e.g., `Activity`) is also held via the `ReceiverList`.
- This prevents GC of the Activity → **classic memory leak**.

**Detection:** LeakCanary will flag this as a GC root leak.

**Prevention Rule:** Always pair registration/unregistration in **matching lifecycle methods**:

| Register in | Unregister in |
|---|---|
| `onCreate()` | `onDestroy()` |
| `onStart()` | `onStop()` |
| `onResume()` | `onPause()` |

### Performance: Process Wake-ups from Static Receivers

Every static receiver that matches an implicit broadcast causes Android to **cold-start** your process if it's not running. This:
- Increases RAM usage across all apps.
- Slows down broadcast delivery.
- Drains battery.

This is precisely why Android 8.0 introduced strict limits on implicit static receivers.

### Performance: Too Many Ordered Broadcasts

Ordered broadcasts are delivered serially. If 10 receivers each take 1 second, that's 10 seconds total. Use normal broadcasts whenever result passing is not needed.

### goAsync() Caution

`goAsync()` buys you roughly **10 additional seconds** before AMS marks your process as ANR-eligible. It is **not** a replacement for WorkManager. Use it only for very short, unavoidable async tasks (e.g., quick SharedPreferences read).

```kotlin
class AsyncReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val pendingResult = goAsync()
        CoroutineScope(Dispatchers.IO).launch {
            try {
                // Short async work — NOT network calls or long operations
                val prefs = context.getSharedPreferences("prefs", Context.MODE_PRIVATE)
                prefs.edit().putBoolean("received", true).apply()
            } finally {
                pendingResult.finish() // MUST always call finish()
            }
        }
    }
}
```

---

## Interview Questions

### Beginner

**Q1: What is a BroadcastReceiver and when would you use one?**

A BroadcastReceiver is an Android component that listens for system-wide or app-wide `Intent` broadcasts. You'd use it to react to system events (device boot, battery low, network change) or to receive custom events sent by other app components. It has no UI and is ephemeral — it only lives for the duration of `onReceive()`.

---

**Q2: What is the difference between static and dynamic registration of a BroadcastReceiver?**

| Aspect | Static (Manifest) | Dynamic (Runtime) |
|---|---|---|
| Declaration | `AndroidManifest.xml` | `registerReceiver()` in code |
| Lifecycle | Survives beyond component | Tied to component lifecycle |
| API 26+ implicit | Mostly blocked | Works fine |
| Use case | Boot completed, package installs | Connectivity changes, screen on/off |

Static receivers are declared at install time and can start your process from a cold state. Dynamic receivers require your app to be running and must be explicitly unregistered.

---

**Q3: What thread does `onReceive()` run on? What is the timeout?**

`onReceive()` runs on the **main (UI) thread**. The timeout is **10 seconds** — exceeding it triggers an ANR (Application Not Responding) dialog. You must never perform network calls, disk I/O, or long computations directly in `onReceive()`.

---

### Intermediate

**Q4: Explain the difference between normal broadcasts and ordered broadcasts.**

- **Normal broadcast** (`sendBroadcast()`): All matching receivers receive the Intent simultaneously (or in an undefined order). Receivers cannot pass data to each other and cannot abort the broadcast. It is asynchronous.
- **Ordered broadcast** (`sendOrderedBroadcast()`): Receivers are delivered the broadcast one at a time, in order of their declared `android:priority`. Each receiver can read/modify the result (via `setResultCode`, `setResultData`, `setResultExtras`) or abort propagation entirely via `abortBroadcast()`.

---

**Q5: What are Android 8.0's (API 26) background execution limits for BroadcastReceivers?**

Android 8.0 blocked most **implicit** broadcasts from being delivered to **statically** registered receivers. An implicit broadcast is one that doesn't target a specific app. The rationale is to prevent apps from launching in the background in response to common system events (wasting RAM and battery).

**Exceptions** — a small list of implicit broadcasts are still delivered to static receivers (e.g., `ACTION_BOOT_COMPLETED`, `ACTION_MY_PACKAGE_REPLACED`).

**Workaround:** Register receivers **dynamically** at runtime, or use **WorkManager** with constraints for deferred background work.

---

**Q6: How do you safely return data from an ordered broadcast receiver back to the sender?**

The sender passes a `resultReceiver` (a `BroadcastReceiver`) as the final parameter of `sendOrderedBroadcast()`. Intermediate receivers modify the result bundle using:

```kotlin
setResultCode(Activity.RESULT_OK)
setResultData("processed_data")
val extras = Bundle()
extras.putString("key", "value")
setResultExtras(extras)
```

The final result receiver then reads these values. This is the canonical way to have a chain of processors modify data and report back to the initiator.

---

**Q7: How would you prevent a memory leak from a dynamically registered BroadcastReceiver?**

Always unregister the receiver in a lifecycle callback **symmetric** to where you registered it. Using Kotlin's `LifecycleObserver` or registering in `onStart()`/`onStop()` pairs is the safest approach. In Jetpack Compose or ViewModel-tied scenarios, use `DisposableEffect` or structured coroutines with `SharedFlow` to avoid the receiver entirely.

---

### Advanced

**Q8: Explain the `android:exported` attribute and its security implications.**

`android:exported="true"` means any app on the device can send broadcasts to your receiver. `android:exported="false"` restricts delivery to your own application only (same UID).

**Security implications:**
- An exported receiver with no `android:permission` declared can be triggered by any malicious app, potentially causing unauthorized behavior (e.g., triggering a download, modifying state).
- Always validate the action and data inside `onReceive()`.
- Use `android:permission` to require callers to hold a specific permission:
  ```xml
  <receiver android:name=".SecureReceiver"
            android:exported="true"
            android:permission="com.example.RECEIVE_SECURE_BROADCAST">
  ```
- On API 31+, omitting `android:exported` on a receiver with an intent filter **throws an exception at install time**.

---

**Q9: What is the security problem with `sendBroadcast()` and how do you mitigate it?**

By default, `sendBroadcast(intent)` sends to **all** receivers matching the Intent, including those in malicious apps. Mitigations:
1. **Require a permission:** `sendBroadcast(intent, "com.example.CUSTOM_PERMISSION")` — only receivers holding this permission get the broadcast.
2. **Explicit Intent:** `intent.setPackage(packageName)` — limits delivery to one specific package.
3. **`android:exported="false"`** on the receiver — prevents external delivery entirely.
4. **Avoid sensitive data in broadcast Extras** if using implicit broadcasts.

---

**Q10: When should you use `goAsync()` vs. WorkManager?**

| | `goAsync()` | WorkManager |
|---|---|---|
| Max time | ~10 extra seconds | Minutes to hours |
| Guaranteed execution | No (process can be killed) | Yes (survives reboots) |
| Use case | Very brief async (quick DB write) | Network sync, file processing |
| Complexity | Low | Medium |

Use `goAsync()` only for work that is a few seconds at most and doesn't need guaranteed completion. For anything that could fail, needs retry logic, or runs long, delegate to WorkManager.

---

## Scenario Questions

**Scenario 1:** *Your app registers a BroadcastReceiver in `Activity.onCreate()` to listen for USB connection events. After some time in production, users report the app is slow and crashes with OOM. What is likely wrong?*

The receiver is registered in `onCreate()` but never unregistered (or unregistered in `onDestroy()` while it should have been in `onStop()`). Each time the Activity is re-created (e.g., screen rotation), a new receiver instance is registered without the old one being released. This causes multiple receivers to pile up in AMS memory, each holding a reference to its Activity instance, causing a memory leak that eventually triggers OOM.

**Fix:** Call `unregisterReceiver()` in `onDestroy()` if you registered in `onCreate()`, or use `onStart()`/`onStop()` pairing.

---

**Scenario 2:** *You need your app to sync data whenever the network becomes available, even if the app is in the background. How do you implement this on API 26+?*

Since API 26 blocks implicit static receivers for connectivity changes, you cannot use a manifest-registered receiver for `CONNECTIVITY_CHANGE`. The correct approach:

1. Schedule a periodic `WorkManager` task with a `NetworkType.CONNECTED` constraint.
2. WorkManager will run your `Worker` whenever the network is available, even after a reboot.

```kotlin
val constraints = Constraints.Builder()
    .setRequiredNetworkType(NetworkType.CONNECTED)
    .build()

val syncWork = PeriodicWorkRequestBuilder<DataSyncWorker>(1, TimeUnit.HOURS)
    .setConstraints(constraints)
    .build()

WorkManager.getInstance(context).enqueueUniquePeriodicWork(
    "data_sync",
    ExistingPeriodicWorkPolicy.KEEP,
    syncWork
)
```

---

**Scenario 3:** *An SMS verification app needs to auto-read OTPs on Android 10+. How should it be implemented?*

On Android 10+, reading SMS directly requires `READ_SMS` permission (very restricted by Google Play). Instead:
1. Use the **SMS User Consent API** (`SmsRetriever`) — no permission needed.
2. Register a `BroadcastReceiver` for `SmsRetriever.SMS_RETRIEVED_ACTION`.
3. The API presents a system dialog asking the user to confirm sharing the specific OTP message with your app.

```kotlin
class OtpReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        if (SmsRetriever.SMS_RETRIEVED_ACTION == intent.action) {
            val extras = intent.extras ?: return
            val status = extras.get(SmsRetriever.EXTRA_STATUS) as Status
            when (status.statusCode) {
                CommonStatusCodes.SUCCESS -> {
                    val message = extras.getString(SmsRetriever.EXTRA_SMS_MESSAGE)
                    val otp = extractOtp(message)
                    // Populate OTP field
                }
                CommonStatusCodes.TIMEOUT -> {
                    // Handle timeout
                }
            }
        }
    }

    private fun extractOtp(message: String?): String? {
        return message?.let { Regex("\\d{6}").find(it)?.value }
    }
}
```

---

## Code Examples

### Complete Production BroadcastReceiver: Boot Completed Launcher

```kotlin
// AndroidManifest.xml
// <uses-permission android:name="android.permission.RECEIVE_BOOT_COMPLETED"/>
// <receiver
//     android:name=".BootReceiver"
//     android:exported="false">
//     <intent-filter>
//         <action android:name="android.intent.action.BOOT_COMPLETED"/>
//         <action android:name="android.intent.action.MY_PACKAGE_REPLACED"/>
//     </intent-filter>
// </receiver>

class BootReceiver : BroadcastReceiver() {

    override fun onReceive(context: Context, intent: Intent) {
        when (intent.action) {
            Intent.ACTION_BOOT_COMPLETED,
            Intent.ACTION_MY_PACKAGE_REPLACED -> rescheduleAlarms(context)
        }
    }

    private fun rescheduleAlarms(context: Context) {
        // Re-schedule WorkManager tasks lost after reboot
        val workRequest = PeriodicWorkRequestBuilder<MaintenanceWorker>(
            24, TimeUnit.HOURS
        ).build()

        WorkManager.getInstance(context).enqueueUniquePeriodicWork(
            "maintenance",
            ExistingPeriodicWorkPolicy.KEEP,
            workRequest
        )
    }
}
```

### Ordered Broadcast: Custom Processing Chain

```kotlin
// Sending side
fun sendProcessingBroadcast(context: Context, rawData: String) {
    val intent = Intent("com.example.PROCESS_DATA").apply {
        putExtra("raw_data", rawData)
    }
    context.sendOrderedBroadcast(
        intent,
        null, // receiver permission
        object : BroadcastReceiver() {
            // Final result receiver — always called last
            override fun onReceive(context: Context, intent: Intent) {
                val processedData = resultData
                val resultCode = resultCode
                // Handle final result
                Log.d("BroadcastChain", "Final result: $processedData, code: $resultCode")
            }
        },
        null, // scheduler
        Activity.RESULT_OK, // initial result code
        rawData, // initial result data
        null // initial result extras
    )
}

// High-priority receiver (100) — Sanitizer
class SanitizerReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val raw = resultData ?: return
        val sanitized = raw.replace(Regex("[^A-Za-z0-9]"), "")
        setResultData(sanitized) // Pass sanitized data to next receiver
    }
}

// Lower-priority receiver (50) — Formatter
class FormatterReceiver : BroadcastReceiver() {
    override fun onReceive(context: Context, intent: Intent) {
        val sanitized = resultData ?: return
        val formatted = sanitized.uppercase()
        setResultData(formatted)
    }
}
```

### Modern Replacement: SharedFlow Event Bus

```kotlin
// In a shared singleton (e.g., Hilt @Singleton)
class AppEventBus @Inject constructor() {
    private val _events = MutableSharedFlow<AppEvent>(
        extraBufferCapacity = 64,
        onBufferOverflow = BufferOverflow.DROP_OLDEST
    )
    val events: SharedFlow<AppEvent> = _events.asSharedFlow()

    fun tryEmit(event: AppEvent) {
        _events.tryEmit(event)
    }

    suspend fun emit(event: AppEvent) {
        _events.emit(event)
    }
}

sealed class AppEvent {
    data class NetworkChanged(val isConnected: Boolean) : AppEvent()
    data class DataSynced(val count: Int) : AppEvent()
    object UserLoggedOut : AppEvent()
}

// In ViewModel
@HiltViewModel
class MainViewModel @Inject constructor(
    private val eventBus: AppEventBus
) : ViewModel() {
    init {
        viewModelScope.launch {
            eventBus.events.collect { event ->
                when (event) {
                    is AppEvent.NetworkChanged -> handleNetworkChange(event.isConnected)
                    is AppEvent.DataSynced -> handleSync(event.count)
                    AppEvent.UserLoggedOut -> handleLogout()
                }
            }
        }
    }
}
```

---

## Best Practices

1. **Never block `onReceive()`** — It runs on the main thread. Delegate all work to WorkManager, Services, or coroutines.
2. **Always unregister dynamic receivers** — Use lifecycle-aware registration or Kotlin's `use` pattern.
3. **Declare `android:exported` explicitly** — Required on API 31+. Default to `false` unless external access is needed.
4. **Validate incoming broadcasts** — Check the action string; don't blindly trust the Intent.
5. **Use permissions for sensitive broadcasts** — Both `sendBroadcast(intent, permission)` and `android:permission` on the receiver.
6. **Prefer WorkManager over static receivers** — For background triggers and scheduled tasks.
7. **Prefer StateFlow/SharedFlow over LocalBroadcastManager** — For in-process communication.
8. **Use `setPackage()` for targeted broadcasts** — Prevents broadcast interception by other apps.
9. **Keep `onReceive()` logic minimal** — Parse data, decide action, delegate execution.
10. **Use `Intent.FLAG_RECEIVER_FOREGROUND`** — For time-sensitive broadcasts that must be processed promptly.

---

## Legacy vs Modern

| Feature | Legacy Approach | Modern Approach |
|---|---|---|
| In-process events | `LocalBroadcastManager` | `StateFlow` / `SharedFlow` / `LiveData` |
| Background triggers | Static manifest receiver | `WorkManager` |
| Network monitoring | `CONNECTIVITY_CHANGE` broadcast | `ConnectivityManager.registerNetworkCallback()` |
| SMS reading | `READ_SMS` permission | `SmsRetriever` API / SMS User Consent |
| Sticky broadcasts | `sendStickyBroadcast()` | `SharedPreferences` / `DataStore` / `StateFlow` |
| Battery monitoring | `ACTION_BATTERY_CHANGED` | `BatteryManager` API / `WorkManager` constraints |
| Result passing | Ordered broadcasts | Coroutine channels / Flow |
| Security | Hope for the best | `android:exported`, permissions, `setPackage()` |

---

## Revision Notes

- `BroadcastReceiver.onReceive()` → **main thread**, **10-second timeout**
- `sendBroadcast()` → async, all receivers simultaneously
- `sendOrderedBroadcast()` → serial, priority-based, result passable, abortable
- Static = manifest; Dynamic = `registerReceiver()` / `unregisterReceiver()`
- API 26+: most implicit static receivers **blocked**
- API 31+: `android:exported` **required** when intent-filter present
- `LocalBroadcastManager` → **DEPRECATED** (use `SharedFlow`)
- `sendStickyBroadcast()` → **DEPRECATED**
- `goAsync()` → extends receiver for ~10 seconds; not a replacement for WorkManager
- `abortBroadcast()` → only works in **ordered** broadcasts

---

## Key Takeaways

> BroadcastReceiver is the event-driven backbone of Android's decoupled component model, but its power comes with strict constraints. The **main thread limitation** and the **API 26 background restrictions** are the two most critical facts. In modern Android development, BroadcastReceiver is increasingly **replaced** by WorkManager (for background), `ConnectivityManager` callbacks (for network), and `SharedFlow`/`StateFlow` (for in-process events). Understanding when NOT to use a BroadcastReceiver is as important as knowing how to use one.

**The 5 things every interview candidate must know:**
1. `onReceive()` runs on the **main thread** — no blocking I/O
2. Dynamic receivers must be **unregistered** — or memory leaks
3. API 26 blocks **most implicit static receivers**
4. API 31 requires **`android:exported`** — no exceptions
5. `LocalBroadcastManager` is **deprecated** — use `SharedFlow`

---
---

# Chapter 10: Content Providers & ContentResolver

---

## Concept

A **ContentProvider** is Android's standardized mechanism for **sharing structured data** between application processes. It abstracts an underlying data store (typically a SQLite database, but can be files, in-memory data, or a network resource) behind a **URI-based REST-like interface**.

A **ContentResolver** is the **client-side counterpart**: a proxy object that routes CRUD operations (Create, Read, Update, Delete) to the correct ContentProvider on behalf of the caller.

```
ContentProvider = Data Server (URI-addressed, process-separated)
ContentResolver = Universal Client Proxy (routes by URI authority)
```

### URI Anatomy

```
content://com.example.app.provider/users/42
   ^           ^                    ^     ^
scheme      authority           path  id (row)

content://com.android.contacts/contacts
content://media/external/images/media/5
```

- **scheme**: Always `content://`
- **authority**: Uniquely identifies the provider (reverse-domain convention)
- **path**: Identifies the resource type (table/collection)
- **id**: Optional — identifies a specific row

---

## Why It Exists

Android enforces **process isolation**: apps cannot directly access each other's files or databases. Without a ContentProvider, there is no standard, safe mechanism to:

1. Let the OS **Contacts** database be queried by a dialer, a messaging app, and your own app — all simultaneously.
2. Let a **camera app** share a photo file with a social media app without embedding the full file in an Intent.
3. Let a **file manager** expose documents to document-reading apps.

ContentProvider solves these by:
- Providing a **uniform data access contract** (URI + CRUD methods).
- Enforcing **permission boundaries** (read/write permissions per provider).
- Supporting **change notifications** (ContentObserver).
- Handling **cross-process data access** via Binder automatically.

---

## Internal Working

### Binder-Based IPC

When your app calls `contentResolver.query(uri, ...)`:

1. `ContentResolver` resolves the URI's **authority** to the correct `ContentProvider` by querying `PackageManagerService`.
2. If the provider is in another process, `ContentResolver` makes a **Binder IPC** call to `ContentProvider` via `IContentProvider` (the AIDL interface).
3. The `ContentProvider.query()` method executes **on a Binder thread** (not the main thread of the provider's process).
4. For query results, a **`Cursor`** backed by a `CursorWindow` (shared memory via `MemoryFile`) is returned — the actual data crosses processes via **shared memory**, not Binder directly.

```
Caller Process               Provider Process
─────────────────            ─────────────────
ContentResolver               ContentProvider
      |                              |
      | Binder IPC call              |
      +----> IContentProvider ──────>|
      |                              | Binder thread pool
      |                              | (NOT main thread)
      |                              |
      |      Cursor (CursorWindow)   |
      |<──────────────── Shared Mem ─+
```

### ContentProvider Thread Safety

`ContentProvider` methods (`query`, `insert`, `update`, `delete`) are called on **Binder thread pool threads** — meaning they can be called **concurrently from multiple processes**. Your implementation must be **thread-safe**. SQLite's `SQLiteDatabase` handles its own locking, but if you use in-memory data structures, synchronize them explicitly.

### UriMatcher

`UriMatcher` is a utility class that matches incoming URIs to integer codes, enabling clean dispatch logic:

```kotlin
companion object {
    const val USERS = 1
    const val USER_ID = 2
    const val POSTS = 3

    val uriMatcher = UriMatcher(UriMatcher.NO_MATCH).apply {
        addURI("com.example.provider", "users", USERS)
        addURI("com.example.provider", "users/#", USER_ID)  // # = number wildcard
        addURI("com.example.provider", "posts", POSTS)
    }
}
```

---

## Lifecycle / Flow (ASCII Diagrams)

### ContentProvider Initialization

```
App Process Starts
        |
        v
ContentProvider.onCreate() called
[Runs on MAIN thread, before Application.onCreate()]
        |
        v
Provider is ready to handle requests
        |
  ┌─────┴──────┐
  | query()    |  ← Binder thread
  | insert()   |  ← Binder thread
  | update()   |  ← Binder thread
  | delete()   |  ← Binder thread
  | getType()  |  ← Binder thread
  └────────────┘
```

### Query Flow (Client → Provider)

```
Caller App
    |
    | contentResolver.query(uri, projection, selection, selectionArgs, sortOrder)
    v
ContentResolver
    |
    | Resolve authority → PackageManager → Find Provider
    v
IContentProvider (Binder stub)
    |                              Provider App
    | ────── Binder IPC ─────────> ContentProvider.query()
    |                                     |
    |                                     v
    |                               SQLiteDatabase.query()
    |                                     |
    |                                     v
    |                               Cursor (backed by CursorWindow)
    |                                     |
    | <──── Shared Memory (CursorWindow) ─+
    v
MatrixCursor / SQLiteCursor in Caller Process
    |
    v
Caller processes Cursor, then cursor.close()
```

### ContentObserver Flow

```
Provider modifies data → calls notifyChange(uri, null)
        |
        v
ContentResolver delivers notification to all observers
        |
        v
ContentObserver.onChange(selfChange, uri) called
        |
        v
Observer re-queries or updates UI
```

---

## Real World Example

### Scenario 1: Querying the Contacts Database

```kotlin
class ContactsRepository(private val context: Context) {

    fun getAllContacts(): List<Contact> {
        val contacts = mutableListOf<Contact>()

        // Define which columns we need (projection = SELECT columns)
        val projection = arrayOf(
            ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME,
            ContactsContract.CommonDataKinds.Phone.NUMBER,
            ContactsContract.CommonDataKinds.Phone.CONTACT_ID
        )

        val cursor: Cursor? = context.contentResolver.query(
            ContactsContract.CommonDataKinds.Phone.CONTENT_URI, // URI
            projection,                                          // columns
            null,                                               // selection (WHERE)
            null,                                               // selection args
            "${ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME} ASC" // ORDER BY
        )

        cursor?.use { c ->
            val nameIndex = c.getColumnIndexOrThrow(
                ContactsContract.CommonDataKinds.Phone.DISPLAY_NAME
            )
            val numberIndex = c.getColumnIndexOrThrow(
                ContactsContract.CommonDataKinds.Phone.NUMBER
            )

            while (c.moveToNext()) {
                contacts.add(
                    Contact(
                        name = c.getString(nameIndex),
                        phone = c.getString(numberIndex)
                    )
                )
            }
        }
        return contacts
    }
}

data class Contact(val name: String, val phone: String)
```

### Scenario 2: Sharing Files with FileProvider

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths"/>
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths xmlns:android="http://schemas.android.com/apk/res/android">
    <cache-path name="shared_images" path="images/"/>
    <external-cache-path name="external_images" path="images/"/>
</paths>
```

```kotlin
fun shareImage(context: Context, imageFile: File) {
    val uri = FileProvider.getUriForFile(
        context,
        "${context.packageName}.fileprovider",
        imageFile
    )

    val shareIntent = Intent(Intent.ACTION_SEND).apply {
        type = "image/jpeg"
        putExtra(Intent.EXTRA_STREAM, uri)
        addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION) // Temporary permission grant
    }

    context.startActivity(Intent.createChooser(shareIntent, "Share Image"))
}
```

---

## Common Mistakes

### 1. Not Closing the Cursor

```kotlin
// ❌ WRONG - Cursor never closed → resource leak
val cursor = contentResolver.query(uri, null, null, null, null)
if (cursor != null && cursor.moveToFirst()) {
    val name = cursor.getString(0)
    // Forgot to close!
}

// ✅ CORRECT - Use cursor.use{} which calls close() automatically
val name = contentResolver.query(uri, null, null, null, null)?.use { cursor ->
    if (cursor.moveToFirst()) cursor.getString(0) else null
}
```

### 2. Querying ContentProvider on the Main Thread

```kotlin
// ❌ WRONG - Query blocks main thread
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    val contacts = contactsRepository.getAllContacts() // Blocks UI!
}

// ✅ CORRECT - Query on background thread via coroutines
lifecycleScope.launch {
    val contacts = withContext(Dispatchers.IO) {
        contactsRepository.getAllContacts()
    }
    adapter.submitList(contacts)
}
```

### 3. Using Raw File URIs Instead of FileProvider (API 24+)

```kotlin
// ❌ WRONG on API 24+ - FileUriExposedException thrown
val uri = Uri.fromFile(File(context.cacheDir, "photo.jpg"))
intent.putExtra(MediaStore.EXTRA_OUTPUT, uri) // Crash on API 24+

// ✅ CORRECT
val uri = FileProvider.getUriForFile(context, "${packageName}.fileprovider", file)
intent.putExtra(MediaStore.EXTRA_OUTPUT, uri)
intent.addFlags(Intent.FLAG_GRANT_WRITE_URI_PERMISSION)
```

### 4. Selecting All Columns When You Need Just a Few

```kotlin
// ❌ WRONG - Fetches ALL columns, wastes memory and bandwidth
val cursor = contentResolver.query(uri, null, null, null, null)

// ✅ CORRECT - Specify only needed columns
val projection = arrayOf(ContactsContract.Contacts._ID, ContactsContract.Contacts.DISPLAY_NAME)
val cursor = contentResolver.query(uri, projection, null, null, null)
```

### 5. Not Using Selection Arguments (SQL Injection Risk)

```kotlin
// ❌ WRONG - SQL injection vulnerable
val name = userInput
val cursor = contentResolver.query(uri, null, "name = '$name'", null, null)

// ✅ CORRECT - Parameterized query
val cursor = contentResolver.query(
    uri, null,
    "name = ?",         // selection with placeholder
    arrayOf(userInput), // selectionArgs - safely escaped by framework
    null
)
```

---

## Memory Leak / Performance Concerns

### Cursor Leak

A `Cursor` holds a reference to a `CursorWindow` (up to 2MB of shared memory). If never closed, this shared memory is never released, and the associated `SQLiteDatabase` connection may remain open.

**Detection:** StrictMode will log unclosed Cursors. Use `cursor.use {}` idiom always.

### ContentObserver Leak

```kotlin
// ❌ WRONG - Observer never unregistered
class LeakyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        contentResolver.registerContentObserver(
            ContactsContract.Contacts.CONTENT_URI, true,
            object : ContentObserver(Handler(Looper.getMainLooper())) {
                override fun onChange(selfChange: Boolean) {
                    refreshContacts()
                }
            }
        )
        // Never unregistered → Activity leak
    }
}

// ✅ CORRECT
class SafeActivity : AppCompatActivity() {
    private lateinit var observer: ContentObserver

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        observer = object : ContentObserver(Handler(Looper.getMainLooper())) {
            override fun onChange(selfChange: Boolean) = refreshContacts()
        }
        contentResolver.registerContentObserver(
            ContactsContract.Contacts.CONTENT_URI, true, observer
        )
    }

    override fun onDestroy() {
        super.onDestroy()
        contentResolver.unregisterContentObserver(observer)
    }
}
```

### Performance: Avoid ContentProvider for Internal App Data

ContentProvider adds Binder IPC overhead even for in-process access. For data internal to your app, use **Room** directly. ContentProvider is appropriate only when:
- Another app needs to access your data.
- You're exposing data to the Android system (e.g., search suggestions, contacts sync).

### Large Cursor Sets

Avoid loading millions of rows into a Cursor. Use:
- **Projection** to limit columns.
- **Selection** to filter rows.
- **Paging** (limit/offset in `sortOrder`): `"name ASC LIMIT 100 OFFSET 0"`.
- **`CursorAdapter` with `RecyclerView`** for lazy loading.

---

## Interview Questions

### Beginner

**Q1: What is a ContentProvider and what problem does it solve?**

A ContentProvider is an Android component that provides a standardized, URI-based interface for sharing structured data between applications. It solves the problem of cross-process data access: apps run in separate processes with isolated memory, so without a ContentProvider, there's no safe, standardized way for one app to read or write another app's data. Examples include the system Contacts database, MediaStore (photos/videos), and Calendar data — all accessed via ContentProvider.

---

**Q2: What is ContentResolver and how does it relate to ContentProvider?**

`ContentResolver` is the client-side interface that your app uses to communicate with any ContentProvider. It is obtained via `context.contentResolver`. When you call `contentResolver.query(uri, ...)`, the resolver looks up which ContentProvider handles that URI authority and routes your call there — potentially across process boundaries. You never interact with `ContentProvider` directly from client code; you always go through `ContentResolver`.

---

**Q3: Explain the URI format used by ContentProviders.**

```
content://authority/path/id
```
- **content://** — scheme indicating this is a ContentProvider URI
- **authority** — uniquely identifies the provider (e.g., `com.android.contacts`)
- **path** — the resource type (e.g., `/contacts`, `/phones`)
- **id** (optional) — a specific row ID

Example: `content://com.android.contacts/contacts/42` refers to contact with ID 42.

---

### Intermediate

**Q4: What is UriMatcher and why is it used?**

`UriMatcher` is a helper class that maps URI patterns to integer constants, enabling clean switch-statement dispatch in ContentProvider methods. Without it, you'd need complex string parsing.

```kotlin
val matcher = UriMatcher(UriMatcher.NO_MATCH)
matcher.addURI("com.example.provider", "items", ITEMS)       // matches /items
matcher.addURI("com.example.provider", "items/#", ITEM_ID)  // matches /items/123
```

Inside `query()`, you call `matcher.match(uri)` and switch on the returned code.

---

**Q5: What thread do ContentProvider methods run on? How does this affect implementation?**

ContentProvider CRUD methods (`query`, `insert`, `update`, `delete`) are called on the **Binder thread pool** — not on the main thread of the provider's hosting process. This means:
1. You must not update the main thread UI directly from these methods.
2. Multiple calls can arrive **concurrently**, so implementations must be **thread-safe**.
3. SQLite's `SQLiteDatabase` handles its own thread safety, but any in-memory data structures you use need explicit synchronization.

However, `ContentProvider.onCreate()` is called on the **main thread**, very early in the app startup sequence — keep it lightweight.

---

**Q6: What is FileProvider and why is it needed on API 24+?**

Before API 24, apps could share files using `file://` URIs. Starting with API 24, Android throws `FileUriExposedException` when a `file://` URI crosses process boundaries, because it bypasses Android's permission system (the receiving app needs read access to the actual file path).

`FileProvider` (a subclass of `ContentProvider`) solves this by:
1. Generating a `content://` URI for the file instead of `file://`.
2. Granting **temporary read/write permissions** to the receiving app via `FLAG_GRANT_READ_URI_PERMISSION`.
3. Requiring a `file_paths.xml` configuration that declares which directory paths can be shared.

---

**Q7: What is ContentObserver and how does change notification work?**

`ContentObserver` is a callback mechanism for watching data changes in a ContentProvider. A client registers an observer with `contentResolver.registerContentObserver(uri, notifyForDescendants, observer)`. When the provider calls `context.contentResolver.notifyChange(uri, null)` after modifying data, all registered observers for that URI are notified via `onChange()`.

This powers reactive data display: your RecyclerView can automatically refresh when contacts data changes, without polling.

---

### Advanced

**Q8: How does cross-process Cursor data transfer work without passing all data through Binder?**

Binder has a per-transaction limit of **1MB**. Raw database rows can easily exceed this for large result sets. Android solves this via `CursorWindow`:

- A `CursorWindow` is backed by anonymous **shared memory** (`MemoryFile` / `SharedMemory`).
- The ContentProvider fills the window with row data in its process's shared memory region.
- A **file descriptor** to this shared memory is passed back to the caller via Binder (file descriptors can be duplicated across processes).
- The caller directly reads row data from the shared memory — no per-row Binder call needed.
- Each `CursorWindow` holds up to **2MB** by default. For larger result sets, the Cursor fetches additional windows on demand.

---

**Q9: When should you use ContentProvider for internal app data vs. Room directly?**

| Situation | Use ContentProvider? |
|---|---|
| Data only consumed within your app | ❌ No — use Room directly |
| Data needs to be accessed by other apps | ✅ Yes |
| Android system features (search suggestions, auto-fill, sync adapters) | ✅ Yes |
| Widget needs data from your app | ✅ Yes |
| Exposing contacts-like data to the OS | ✅ Yes |

ContentProvider adds significant complexity (UriMatcher, permissions, thread safety, contract classes) and Binder IPC overhead. For purely internal data, Room with Flow or LiveData is simpler and faster.

---

**Q10: How do you implement temporary URI permissions for file sharing?**

Use `grantUriPermission()` or `Intent.FLAG_GRANT_READ_URI_PERMISSION`:

```kotlin
// Option 1: Intent flags (preferred for share intents)
intent.addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)

// Option 2: Manual grant (for specific package)
context.grantUriPermission(
    "com.example.receiverapp",
    fileUri,
    Intent.FLAG_GRANT_READ_URI_PERMISSION
)
// Revoke explicitly when done
context.revokeUriPermission(fileUri, Intent.FLAG_GRANT_READ_URI_PERMISSION)
```

Permissions granted via Intent flags are automatically revoked when the receiving task stack is cleared.

---

## Scenario Questions

**Scenario 1:** *Your app takes photos using the camera and needs to share them with a social media app on Android 10+. How do you implement this securely?*

Use **FileProvider** to generate a `content://` URI instead of a raw `file://` URI:

1. Save the photo to your app's cache directory.
2. Use `FileProvider.getUriForFile()` to get a `content://` URI.
3. Grant temporary read permission via `FLAG_GRANT_READ_URI_PERMISSION` in the share Intent.
4. The social media app can read the file via ContentResolver without having access to your private file path.

This satisfies Android 7.0+ (no `FileUriExposedException`), scoped storage on Android 10+, and requires no storage permissions.

---

**Scenario 2:** *You are building a custom ContentProvider for your app's notes database. Another app needs to read (but not write) notes. How do you set up permissions?*

In the manifest:

```xml
<permission
    android:name="com.example.notes.READ_NOTES"
    android:protectionLevel="normal"/>

<provider
    android:name=".NotesProvider"
    android:authorities="com.example.notes.provider"
    android:exported="true"
    android:readPermission="com.example.notes.READ_NOTES"
    android:writePermission="com.example.notes.WRITE_NOTES">
</provider>
```

The reading app must declare `<uses-permission android:name="com.example.notes.READ_NOTES"/>`. Without it, any attempt to call `contentResolver.query()` on your provider throws `SecurityException`. Write operations are protected by a separate, more-restricted permission.

---

**Scenario 3:** *A user reports that switching to the Contacts screen takes 3 seconds. You find the query runs on the main thread. How do you fix it?*

Move the ContentProvider query to a background thread using Kotlin coroutines with Room or a repository pattern:

```kotlin
// Repository
class ContactsRepository(private val context: Context) {
    suspend fun getContacts(): List<Contact> = withContext(Dispatchers.IO) {
        // ContentResolver.query() runs on IO thread
        queryContacts(context.contentResolver)
    }
}

// ViewModel
class ContactsViewModel(private val repo: ContactsRepository) : ViewModel() {
    val contacts = MutableStateFlow<List<Contact>>(emptyList())

    init {
        viewModelScope.launch {
            contacts.value = repo.getContacts()
        }
    }
}

// Fragment observes contacts StateFlow and updates RecyclerView
```

This ensures zero blocking on the main thread and the UI remains responsive throughout the query.

---

## Code Examples

### Custom ContentProvider — Full Implementation

```kotlin
class NotesProvider : ContentProvider() {

    private lateinit var database: NotesDatabase

    companion object {
        const val AUTHORITY = "com.example.notes.provider"
        val CONTENT_URI: Uri = Uri.parse("content://$AUTHORITY/notes")

        private const val NOTES = 1
        private const val NOTE_ID = 2

        private val uriMatcher = UriMatcher(UriMatcher.NO_MATCH).apply {
            addURI(AUTHORITY, "notes", NOTES)
            addURI(AUTHORITY, "notes/#", NOTE_ID)
        }
    }

    override fun onCreate(): Boolean {
        // onCreate() is on the main thread — keep lightweight
        database = NotesDatabase.getInstance(context!!)
        return true
    }

    override fun getType(uri: Uri): String? {
        return when (uriMatcher.match(uri)) {
            NOTES -> "vnd.android.cursor.dir/vnd.$AUTHORITY.notes"
            NOTE_ID -> "vnd.android.cursor.item/vnd.$AUTHORITY.notes"
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
    }

    override fun query(
        uri: Uri,
        projection: Array<out String>?,
        selection: String?,
        selectionArgs: Array<out String>?,
        sortOrder: String?
    ): Cursor? {
        val db = database.readableDatabase
        return when (uriMatcher.match(uri)) {
            NOTES -> db.query("notes", projection, selection, selectionArgs, null, null, sortOrder)
            NOTE_ID -> {
                val id = ContentUris.parseId(uri)
                db.query("notes", projection, "_id = ?", arrayOf(id.toString()), null, null, sortOrder)
            }
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }?.also { cursor ->
            cursor.setNotificationUri(context!!.contentResolver, uri)
        }
    }

    override fun insert(uri: Uri, values: ContentValues?): Uri? {
        val db = database.writableDatabase
        return when (uriMatcher.match(uri)) {
            NOTES -> {
                val id = db.insertOrThrow("notes", null, values)
                context!!.contentResolver.notifyChange(uri, null)
                ContentUris.withAppendedId(CONTENT_URI, id)
            }
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
    }

    override fun update(
        uri: Uri,
        values: ContentValues?,
        selection: String?,
        selectionArgs: Array<out String>?
    ): Int {
        val db = database.writableDatabase
        val rowsUpdated = when (uriMatcher.match(uri)) {
            NOTES -> db.update("notes", values, selection, selectionArgs)
            NOTE_ID -> {
                val id = ContentUris.parseId(uri)
                db.update("notes", values, "_id = ?", arrayOf(id.toString()))
            }
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
        if (rowsUpdated > 0) context!!.contentResolver.notifyChange(uri, null)
        return rowsUpdated
    }

    override fun delete(uri: Uri, selection: String?, selectionArgs: Array<out String>?): Int {
        val db = database.writableDatabase
        val rowsDeleted = when (uriMatcher.match(uri)) {
            NOTES -> db.delete("notes", selection, selectionArgs)
            NOTE_ID -> {
                val id = ContentUris.parseId(uri)
                db.delete("notes", "_id = ?", arrayOf(id.toString()))
            }
            else -> throw IllegalArgumentException("Unknown URI: $uri")
        }
        if (rowsDeleted > 0) context!!.contentResolver.notifyChange(uri, null)
        return rowsDeleted
    }
}
```

### ContentObserver with Coroutines

```kotlin
/**
 * Extension function to observe ContentProvider changes as a Flow
 */
fun ContentResolver.observeUri(uri: Uri): Flow<Unit> = callbackFlow {
    val observer = object : ContentObserver(Handler(Looper.getMainLooper())) {
        override fun onChange(selfChange: Boolean) {
            trySend(Unit)
        }
    }
    registerContentObserver(uri, true, observer)
    // Emit once immediately to trigger initial load
    trySend(Unit)
    awaitClose { unregisterContentObserver(observer) }
}

// Usage in ViewModel
class ContactsViewModel(
    private val contentResolver: ContentResolver
) : ViewModel() {

    val contacts: StateFlow<List<Contact>> = contentResolver
        .observeUri(ContactsContract.Contacts.CONTENT_URI)
        .map { queryContacts() }
        .stateIn(
            scope = viewModelScope,
            started = SharingStarted.WhileSubscribed(5_000),
            initialValue = emptyList()
        )

    private suspend fun queryContacts(): List<Contact> = withContext(Dispatchers.IO) {
        val result = mutableListOf<Contact>()
        contentResolver.query(
            ContactsContract.Contacts.CONTENT_URI,
            arrayOf(ContactsContract.Contacts._ID, ContactsContract.Contacts.DISPLAY_NAME),
            null, null,
            "${ContactsContract.Contacts.DISPLAY_NAME} ASC"
        )?.use { cursor ->
            while (cursor.moveToNext()) {
                result.add(
                    Contact(
                        id = cursor.getLong(0),
                        name = cursor.getString(1) ?: ""
                    )
                )
            }
        }
        result
    }
}
```

---

## Best Practices

1. **Always close Cursors** — Use `cursor.use {}` to ensure `close()` is always called.
2. **Never query on the main thread** — Use `Dispatchers.IO` with coroutines.
3. **Use projections** — Never pass `null` projection in production; specify only needed columns.
4. **Parameterize selections** — Use `?` placeholders with `selectionArgs` to prevent SQL injection.
5. **Notify observers after changes** — Call `contentResolver.notifyChange(uri, null)` after insert/update/delete.
6. **Set notification URI on Cursors** — `cursor.setNotificationUri()` enables ContentObserver callbacks.
7. **FileProvider over raw file URIs** — Required on API 24+ and enforces proper permission scoping.
8. **Prefer Room for internal data** — ContentProvider is for cross-app sharing only.
9. **Define Contract classes** — Expose URI and column name constants in a public Contract class for the consuming apps.
10. **Declare proper permissions** — Use `readPermission` and `writePermission` separately for fine-grained access control.

---

## Legacy vs Modern

| Feature | Legacy Approach | Modern Approach |
|---|---|---|
| Async queries | `CursorLoader` (deprecated) | `Coroutines` + `Dispatchers.IO` |
| Observing changes | `CursorLoader` with `LoaderManager` | `ContentObserver` + `callbackFlow` |
| Internal data storage | ContentProvider wrapping SQLite | `Room` + `Flow` / `LiveData` |
| File sharing | `file://` URI | `FileProvider` + `content://` URI |
| Photo/media selection | Storage permission + file picker | `Photo Picker API` (no permission needed) |
| Background data sync | `SyncAdapter` + `ContentProvider` | `WorkManager` |
| Paging large data | Manual offset/limit | `Paging 3` library |

---

## Revision Notes

- ContentProvider = cross-app data sharing via URI-based CRUD interface
- ContentResolver = client proxy; routes calls by URI authority
- URI format: `content://authority/path/id`
- 6 abstract methods: `onCreate`, `query`, `insert`, `update`, `delete`, `getType`
- `onCreate()` → **main thread**; all others → **Binder thread pool**
- UriMatcher maps URI patterns to integer dispatch codes
- `Cursor` backed by `CursorWindow` (shared memory, max 2MB per window)
- Always `cursor.use {}` to prevent resource leaks
- `FileProvider` required for file sharing on API 24+ (no `file://` URIs)
- `notifyChange()` triggers `ContentObserver.onChange()`
- `CursorLoader` → **DEPRECATED** (use coroutines)
- For internal data: use **Room** directly, skip ContentProvider overhead

---

## Key Takeaways

> ContentProvider is Android's **cross-process data sharing standard**, not a general-purpose database wrapper. The most critical practical skill is using `ContentResolver` to query system providers (Contacts, MediaStore) correctly — with proper projections, selection arguments, and **always closing Cursors**. `FileProvider` is a mandatory modern requirement for file sharing. For internal app data, skip ContentProvider entirely and use Room.

**The 5 things every interview candidate must know:**
1. ContentResolver is the **client**; ContentProvider is the **server** — communicate via URI
2. CRUD methods run on **Binder threads** (thread-safe required) — `onCreate()` on main thread
3. **Always close Cursors** — `cursor.use {}` is the safe idiom
4. **FileProvider** is required for file sharing on API 24+ — no raw `file://` URIs
5. ContentProvider is for **cross-app sharing** — use Room for internal data

---
---

# Chapter 11: Runtime Permissions

---

## Concept

**Runtime Permissions** (introduced in Android 6.0 / API 23) is Android's model for granting **dangerous permissions** at runtime — when the app actually needs them — rather than blindly at install time. The user sees a clear, contextual dialog explaining what the app wants to access and can approve or deny.

Permissions fall into three primary protection levels:

| Level | Description | Examples | Grant Mechanism |
|---|---|---|---|
| **Normal** | Low-risk, no privacy concern | `INTERNET`, `ACCESS_NETWORK_STATE`, `VIBRATE` | Auto-granted at install |
| **Dangerous** | Access to sensitive user data | `CAMERA`, `READ_CONTACTS`, `ACCESS_FINE_LOCATION` | Runtime dialog required |
| **Signature** | Only for apps signed with same key | `BIND_DEVICE_ADMIN`, custom app permissions | Auto-granted if same signature |
| **AppOp / Special** | Highly privileged | `MANAGE_EXTERNAL_STORAGE`, `SYSTEM_ALERT_WINDOW` | Settings page required |

---

## Why It Exists

Before API 23, Android used an **install-time permission model**:
- Users had to **accept all permissions at install** or not install the app at all.
- A flashlight app could request `READ_CONTACTS` and `SEND_SMS` — users either accepted or missed out on the app.
- Users had no visibility into which permissions were actually being used.
- **Rogue apps abused this**: requesting broad permissions "just in case."

Runtime permissions solve this by:
1. **Deferring permission grants** until they are contextually needed.
2. **Giving users meaningful choice** with a clear dialog showing what's accessed.
3. **Allowing revocation** — users can revoke permissions at any time in settings.
4. **Improving transparency** — apps must declare their intent and purpose.

---

## Internal Working

### The Permission Lifecycle

```
App declares permission in manifest
        ↓
checkSelfPermission() → PERMISSION_DENIED
        ↓
[optionally] shouldShowRequestPermissionRationale() = true?
        ↓ yes                               ↓ no (first ask or 'don't ask again')
Show rationale UI                     Go straight to request
        ↓
requestPermissions() / ActivityResultContracts.RequestPermission()
        ↓
System shows Permission Dialog
        ↓
    User Grants                         User Denies
        ↓                                   ↓
checkSelfPermission()            shouldShowRequestPermissionRationale()
= PERMISSION_GRANTED                = true  → can ask again with rationale
                                    = false → permanently denied → open Settings
```

### System Permission Check Flow

When your app calls a protected API (e.g., opens the camera):

1. The API method calls `checkCallingOrSelfPermission()` inside the framework or system service.
2. `PackageManagerService` checks the `packages.xml` database for the app's granted permissions.
3. If denied → `SecurityException` is thrown.
4. If granted → the operation proceeds.

### `shouldShowRequestPermissionRationale()` Logic

This method returns:
- **`true`** — The user previously denied the permission (but did NOT check "Don't ask again"). You should explain WHY you need it before requesting again.
- **`false`** — One of three cases:
  - The permission was **never requested** before (first time).
  - The user checked **"Don't ask again"** (permanently denied).
  - The permission is a **device policy** (forced granted or denied).

The **first-time vs. permanently-denied ambiguity** is a known quirk. The best pattern is to track whether you've ever requested the permission in SharedPreferences.

---

## Lifecycle / Flow (ASCII Diagrams)

### Full Permission Request Flow

```
Feature triggered (e.g., user taps "Open Camera")
    |
    v
checkSelfPermission(CAMERA)
    |
    +── GRANTED ──> Proceed with Camera
    |
    +── DENIED ──>
              |
              v
    shouldShowRequestPermissionRationale()?
              |
    YES ──────+──────> Show rationale dialog/snackbar
              |              |
              |              v
              |         User taps "OK"
              |
              v
    requestPermission (via ActivityResultLauncher)
              |
              v
    System Dialog: "Allow Camera?"
              |
    GRANTED ──+──> Proceed with Camera
              |
    DENIED  ──+──>
              |
              v
    shouldShowRequestPermissionRationale()?
              |
    YES ──────+── Show in-app message: "Camera needed for X"
              |
    NO  ──────+── Permanently denied → Open App Settings
                  via ACTION_APPLICATION_DETAILS_SETTINGS
```

### Permission State Machine

```
          ┌─────────────────────────────────────────────────────┐
          │                 NEVER_REQUESTED                     │
          │   (shouldShowRationale = false)                     │
          └──────────────────────┬──────────────────────────────┘
                                 │
                    requestPermission()
                                 │
              ┌──────────────────┴──────────────────┐
              │                                     │
           GRANTED                               DENIED
      (PERMISSION_GRANTED)             (shouldShowRationale = true)
              │                                     │
              │                        requestPermission() again
              │                                     │
              │                    ┌────────────────┴──────────────────┐
              │                    │                                   │
              │                 GRANTED                     PERMANENTLY DENIED
              │                                          (shouldShowRationale = false)
              │                                                        │
              │                                              Must open Settings
              └────────────────────────────────────────────────────────┘
```

---

## Real World Example

### Camera Permission Request (Modern Approach)

```kotlin
class CameraActivity : AppCompatActivity() {

    private val requestCameraPermission = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted ->
        if (isGranted) {
            openCamera()
        } else {
            handlePermissionDenied()
        }
    }

    fun onCameraButtonClicked() {
        when {
            ContextCompat.checkSelfPermission(this, Manifest.permission.CAMERA)
                    == PackageManager.PERMISSION_GRANTED -> {
                openCamera()
            }
            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) -> {
                showCameraRationaleDialog()
            }
            else -> {
                requestCameraPermission.launch(Manifest.permission.CAMERA)
            }
        }
    }

    private fun showCameraRationaleDialog() {
        MaterialAlertDialogBuilder(this)
            .setTitle("Camera Access Required")
            .setMessage("We need camera access to let you take profile photos. Your photos are never uploaded without your consent.")
            .setPositiveButton("Grant") { _, _ ->
                requestCameraPermission.launch(Manifest.permission.CAMERA)
            }
            .setNegativeButton("Not Now") { dialog, _ ->
                dialog.dismiss()
                showLimitedFunctionalityMessage()
            }
            .show()
    }

    private fun handlePermissionDenied() {
        if (!shouldShowRequestPermissionRationale(Manifest.permission.CAMERA)) {
            // Permanently denied → direct to settings
            showSettingsSnackbar()
        } else {
            showLimitedFunctionalityMessage()
        }
    }

    private fun showSettingsSnackbar() {
        Snackbar.make(
            binding.root,
            "Camera permission is required. Please enable it in Settings.",
            Snackbar.LENGTH_LONG
        ).setAction("Settings") {
            openAppSettings()
        }.show()
    }

    private fun openAppSettings() {
        Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
            data = Uri.fromParts("package", packageName, null)
            startActivity(this)
        }
    }

    private fun showLimitedFunctionalityMessage() {
        // Gracefully degrade: show image picker option instead
    }

    private fun openCamera() { /* launch camera */ }
}
```

---

## Common Mistakes

### 1. Using Deprecated `onRequestPermissionsResult()`

```kotlin
// ❌ DEPRECATED - requestPermissions() + onRequestPermissionsResult()
override fun onRequestPermissionsResult(
    requestCode: Int, permissions: Array<String>, grantResults: IntArray
) {
    super.onRequestPermissionsResult(requestCode, permissions, grantResults)
    if (requestCode == REQUEST_CAMERA && grantResults.firstOrNull() == PERMISSION_GRANTED) {
        openCamera()
    }
}

// ✅ MODERN - ActivityResultContracts.RequestPermission()
private val launcher = registerForActivityResult(
    ActivityResultContracts.RequestPermission()
) { granted -> if (granted) openCamera() }
```

### 2. Requesting Permission at App Launch

Users don't understand why a notes app immediately asks for camera access on the first screen. Always request permissions in context (when the user performs the action that requires it).

```kotlin
// ❌ WRONG - Permission at launch without context
override fun onCreate(savedInstanceState: Bundle?) {
    super.onCreate(savedInstanceState)
    requestCameraPermission.launch(Manifest.permission.CAMERA) // Confusing!
}

// ✅ CORRECT - Request when user explicitly tries to use camera
binding.cameraFab.setOnClickListener {
    checkAndRequestCameraPermission() // In context
}
```

### 3. Crashing When Permission Is Denied

```kotlin
// ❌ WRONG - No denial handling
launcher = registerForActivityResult(RequestPermission()) { granted ->
    openCamera() // NullPointerException if camera not available!
}

// ✅ CORRECT - Always handle denial gracefully
launcher = registerForActivityResult(RequestPermission()) { granted ->
    if (granted) openCamera() else showAlternativeFlow()
}
```

### 4. Not Declaring the Permission in Manifest

Runtime request is required even for normal permissions that don't show a dialog. Forgetting the manifest declaration throws `SecurityException`.

```xml
<!-- MUST be in AndroidManifest.xml even if auto-granted -->
<uses-permission android:name="android.permission.CAMERA"/>
<uses-permission android:name="android.permission.ACCESS_FINE_LOCATION"/>
```

### 5. Ignoring Granular Media Permissions on API 33+

```kotlin
// ❌ WRONG on API 33+ - READ_EXTERNAL_STORAGE is deprecated/ineffective
ContextCompat.checkSelfPermission(this, Manifest.permission.READ_EXTERNAL_STORAGE)

// ✅ CORRECT - Use granular permissions on API 33+
val permission = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
    Manifest.permission.READ_MEDIA_IMAGES
} else {
    Manifest.permission.READ_EXTERNAL_STORAGE
}
ContextCompat.checkSelfPermission(this, permission)
```

---

## Memory Leak / Performance Concerns

### Registering ActivityResultLauncher Too Late

`registerForActivityResult()` must be called **before** `onStart()` — ideally in `onCreate()` or as a property initializer. Calling it after the lifecycle has advanced can cause issues, though the framework checks for this.

### Leaking Permission Check Results in Long-Running Objects

If you cache the result of `checkSelfPermission()` in a long-lived object (e.g., a singleton), the cached value can become stale — the user might revoke the permission in settings while your app is backgrounded.

```kotlin
// ❌ WRONG - Cached permission may be stale
class CameraManager {
    private var hasPermission: Boolean = checkPermission() // Cached once!

    fun takePhoto() {
        if (hasPermission) openCamera() // Might fail if revoked!
    }
}

// ✅ CORRECT - Always check live at the point of use
class CameraManager(private val context: Context) {
    fun takePhoto() {
        if (ContextCompat.checkSelfPermission(context, Manifest.permission.CAMERA)
                == PackageManager.PERMISSION_GRANTED) {
            openCamera()
        }
    }
}
```

### StrictMode and Permission Checks

Excessive permission checks on the main thread are acceptable (they're simple bitmask lookups in the framework). However, for UI that needs to know permission state across many components, consolidate the check in a ViewModel and expose it via StateFlow to avoid redundant calls.

---

## Interview Questions

### Beginner

**Q1: What is the difference between normal and dangerous permissions?**

- **Normal permissions** (e.g., `INTERNET`, `VIBRATE`): Low privacy risk. Automatically granted at install if declared in the manifest. No user dialog shown.
- **Dangerous permissions** (e.g., `CAMERA`, `READ_CONTACTS`, `ACCESS_FINE_LOCATION`): Access sensitive user data or hardware. Must be explicitly declared in the manifest AND requested at runtime (API 23+). The user sees a dialog and can approve or deny.

---

**Q2: How do you check if a dangerous permission is granted?**

```kotlin
val granted = ContextCompat.checkSelfPermission(context, Manifest.permission.CAMERA) ==
    PackageManager.PERMISSION_GRANTED
```

`ContextCompat.checkSelfPermission()` is the safe wrapper that handles API level differences. On pre-API 23 devices it always returns `PERMISSION_GRANTED` (since all permissions were granted at install).

---

**Q3: What does `shouldShowRequestPermissionRationale()` return, and when should you use it?**

- Returns `true` when the user has **previously denied** the permission (and did NOT select "Don't ask again"). This signals you should explain WHY you need the permission before requesting again.
- Returns `false` when the permission has **never been requested** OR has been **permanently denied** (selected "Don't ask again") OR is blocked by device policy.

Use it to show an educational dialog/rationale UI before presenting the system dialog a second time.

---

### Intermediate

**Q4: Compare the old `requestPermissions()` approach with the modern `ActivityResultContracts` approach.**

| Aspect | Old API | Modern API |
|---|---|---|
| Request method | `requestPermissions()` | `registerForActivityResult(RequestPermission())` |
| Callback | `onRequestPermissionsResult()` | Lambda in `registerForActivityResult` |
| Type safety | Poor (requestCode int matching) | Strong (typed contract) |
| Testability | Difficult | Easy (inject ActivityResultRegistry) |
| Lifecycle awareness | Manual | Automatic |
| Deprecated? | Not deprecated, but discouraged | Recommended |

The modern approach separates registration (in `onCreate`) from launching (when needed), making code cleaner and testable.

---

**Q5: What changed with media permissions in Android 13 (API 33)?**

Before API 33: `READ_EXTERNAL_STORAGE` allowed access to all media files.

API 33 introduced **granular media permissions**:
- `READ_MEDIA_IMAGES` — only images
- `READ_MEDIA_VIDEO` — only video
- `READ_MEDIA_AUDIO` — only audio

`READ_EXTERNAL_STORAGE` is **deprecated** on API 33+ and no longer grants media access. Apps must request only the specific media types they need.

Additionally, the **Photo Picker API** (available from API 33, backported to API 21 via Google Play System Updates) allows users to select photos without granting any storage permission — the system picker shows the user's gallery and returns the selected URI.

---

**Q6: A user has permanently denied a permission. What is the correct UX response?**

When `shouldShowRequestPermissionRationale()` returns `false` after a denied request (indicating permanent denial), you **must not** show the system dialog again (it won't appear — Android silently denies it). Instead:

1. Show an in-app message explaining that the permission was denied and what functionality is unavailable.
2. Provide a button that opens the app's system settings page so the user can manually grant it.

```kotlin
Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
    data = Uri.fromParts("package", packageName, null)
    startActivity(this)
}
```

Never lock the user out of the app entirely — always provide a graceful degraded experience.

---

**Q7: What are permission groups and how do they affect requesting multiple related permissions?**

Android groups related dangerous permissions into **permission groups** (e.g., `READ_CONTACTS` and `WRITE_CONTACTS` are in the `CONTACTS` group). On Android 6–7, granting one permission in a group auto-granted the others. On API 8+, the behavior was tightened — each permission must be individually requested, though the system dialog may show them together.

**Best practice:** Always request each permission individually and check each one independently. Do not rely on group-level auto-granting behavior in your logic.

---

### Advanced

**Q8: How do you handle requesting multiple permissions simultaneously, and how do you process the results?**

Use `ActivityResultContracts.RequestMultiplePermissions()`:

```kotlin
private val requestMultiplePermissions = registerForActivityResult(
    ActivityResultContracts.RequestMultiplePermissions()
) { permissions: Map<String, Boolean> ->
    val cameraGranted = permissions[Manifest.permission.CAMERA] ?: false
    val audioGranted = permissions[Manifest.permission.RECORD_AUDIO] ?: false

    when {
        cameraGranted && audioGranted -> startVideoCall()
        cameraGranted -> startAudioOnlyCall() // Graceful degradation
        else -> showPermissionDeniedMessage()
    }
}

// Launch
requestMultiplePermissions.launch(arrayOf(
    Manifest.permission.CAMERA,
    Manifest.permission.RECORD_AUDIO
))
```

---

**Q9: Explain how one-time permissions (API 30+) work and how your app should handle them.**

Android 11 (API 30) introduced **one-time permissions** for location, camera, and microphone. When the user selects "Only this time" in the permission dialog, the permission is granted for the current session only. It is automatically revoked when:
- The app goes to the background for a significant period.
- The user leaves the app and returns.

**Your app must handle this gracefully:**
- Re-check permission whenever resuming from background (`onResume()`).
- Do not cache the permission grant result.
- When a one-time permission is revoked mid-session, catch the `SecurityException` and show a permission request dialog again.

```kotlin
override fun onResume() {
    super.onResume()
    // Re-check one-time permissions that may have been revoked
    if (isLocationRequired && !hasLocationPermission()) {
        stopLocationUpdates()
        promptForLocationPermission()
    }
}
```

---

**Q10: What is `MANAGE_EXTERNAL_STORAGE` and when is it appropriate to request it?**

`MANAGE_EXTERNAL_STORAGE` (introduced in API 30 as part of scoped storage enforcement) grants an app unrestricted access to all files on external storage — effectively bypassing scoped storage restrictions. Requesting it requires:

1. Declaring `<uses-permission android:name="android.permission.MANAGE_EXTERNAL_STORAGE"/>` in the manifest.
2. At runtime, directing the user to a **special Settings page** (not a dialog): `Settings.ACTION_MANAGE_APP_ALL_FILES_ACCESS_PERMISSION`.
3. **Google Play severely restricts this permission** — it is only approved for specific app categories: file managers, antivirus apps, backup/restore tools. Using it for general apps results in Play Store rejection.

For most apps that need media access, use **Photo Picker API** (no permission) or granular media permissions (`READ_MEDIA_IMAGES`, etc.).

---

## Scenario Questions

**Scenario 1:** *Your location-based feature works fine on most devices, but on Android 12 devices, users report it stops working after a few minutes of backgrounding. What is happening and how do you fix it?*

This is the **one-time permission** behavior introduced in Android 11+. When users select "Only this time" for location, the permission is automatically revoked when the app goes to the background.

**Fix:**
1. In `onResume()`, re-check location permission.
2. If denied, stop the location updates to prevent `SecurityException`.
3. Show a non-intrusive prompt asking the user to grant the permission again, or suggest they grant "While using the app" permission for continuous use.
4. Educate users about the "While using the app" vs. "Only this time" distinction in your permission rationale UI.

---

**Scenario 2:** *You are building a photo editing app that needs to display the user's gallery photos. How do you handle media permissions across API 21, 29, 32, and 33?*

```kotlin
fun requestMediaPermissions() {
    val permissions = when {
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU -> {
            // API 33+: Granular media permissions
            arrayOf(Manifest.permission.READ_MEDIA_IMAGES)
        }
        Build.VERSION.SDK_INT >= Build.VERSION_CODES.Q -> {
            // API 29-32: READ_EXTERNAL_STORAGE with scoped storage
            arrayOf(Manifest.permission.READ_EXTERNAL_STORAGE)
        }
        else -> {
            // API < 29: READ_EXTERNAL_STORAGE
            arrayOf(Manifest.permission.READ_EXTERNAL_STORAGE)
        }
    }
    
    // Better alternative for API 33+: Use Photo Picker (no permission needed!)
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
        launchPhotoPicker() // Returns selected URIs, no permission required
    } else {
        requestMultiplePermissions.launch(permissions)
    }
}

private fun launchPhotoPicker() {
    val pickMedia = registerForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        uri?.let { displayPhoto(it) }
    }
    pickMedia.launch(PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly))
}
```

---

**Scenario 3:** *You're reviewing a PR where a junior developer added camera permission checking in `Application.onCreate()` and stored the result in a static boolean. What problems do you identify?*

**Problems:**
1. **Stale data:** Permissions can be revoked by the user at any time via Settings. A cached result becomes invalid immediately upon revocation.
2. **Wrong place:** `Application.onCreate()` is far too early — there's no UI context to explain the permission or show a dialog.
3. **Thread safety:** Accessing a static boolean from multiple threads (UI thread + background threads) without synchronization is a data race.
4. **One-time permissions:** On API 30+, one-time location/camera/mic permissions expire automatically. A cached `true` value would be incorrect after expiration.

**Correct approach:** Always call `ContextCompat.checkSelfPermission()` at the **point of use** (when the user triggers the feature), never cache the result long-term, and use `ViewModel` + `StateFlow` to propagate current permission state reactively.

---

## Code Examples

### Production-Ready Permission Manager

```kotlin
/**
 * A reusable, lifecycle-aware permission manager that handles all states:
 * granted, rationale required, permanently denied.
 */
class PermissionManager private constructor(
    private val activity: AppCompatActivity
) {

    companion object {
        fun from(activity: AppCompatActivity) = PermissionManager(activity)
    }

    private val required = mutableListOf<String>()
    private var rationaleTitle: String = "Permission Required"
    private var rationaleMessage: String = "This permission is needed for the feature to work."
    private var onGranted: () -> Unit = {}
    private var onDenied: () -> Unit = {}
    private var onPermanentlyDenied: () -> Unit = { showDefaultSettingsDialog() }

    private val launcher = activity.registerForActivityResult(
        ActivityResultContracts.RequestMultiplePermissions()
    ) { results ->
        val allGranted = results.values.all { it }
        val anyPermanentlyDenied = required.any { permission ->
            !results[permission]!! &&
            !activity.shouldShowRequestPermissionRationale(permission)
        }

        when {
            allGranted -> onGranted()
            anyPermanentlyDenied -> onPermanentlyDenied()
            else -> onDenied()
        }
    }

    fun permissions(vararg perms: String) = apply { required.addAll(perms) }

    fun rationale(title: String, message: String) = apply {
        rationaleTitle = title
        rationaleMessage = message
    }

    fun onGranted(block: () -> Unit) = apply { onGranted = block }
    fun onDenied(block: () -> Unit) = apply { onDenied = block }
    fun onPermanentlyDenied(block: () -> Unit) = apply { onPermanentlyDenied = block }

    fun check() {
        val allGranted = required.all {
            ContextCompat.checkSelfPermission(activity, it) == PackageManager.PERMISSION_GRANTED
        }
        if (allGranted) {
            onGranted()
            return
        }

        val showRationale = required.any {
            activity.shouldShowRequestPermissionRationale(it)
        }

        if (showRationale) {
            showRationaleDialog()
        } else {
            launcher.launch(required.toTypedArray())
        }
    }

    private fun showRationaleDialog() {
        MaterialAlertDialogBuilder(activity)
            .setTitle(rationaleTitle)
            .setMessage(rationaleMessage)
            .setPositiveButton("Continue") { _, _ -> launcher.launch(required.toTypedArray()) }
            .setNegativeButton("Cancel") { _, _ -> onDenied() }
            .show()
    }

    private fun showDefaultSettingsDialog() {
        MaterialAlertDialogBuilder(activity)
            .setTitle("Permission Required")
            .setMessage("Please enable the permission in App Settings.")
            .setPositiveButton("Open Settings") { _, _ ->
                activity.startActivity(
                    Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                        data = Uri.fromParts("package", activity.packageName, null)
                    }
                )
            }
            .setNegativeButton("Cancel", null)
            .show()
    }
}

// Usage
PermissionManager.from(this)
    .permissions(Manifest.permission.CAMERA, Manifest.permission.RECORD_AUDIO)
    .rationale(
        title = "Camera & Microphone Access",
        message = "Video calling requires camera and microphone access to connect you with others."
    )
    .onGranted { startVideoCall() }
    .onDenied { showAudioOnlyOption() }
    .onPermanentlyDenied { showManualSettingsInstruction() }
    .check()
```

### Compose-Friendly Permission Handling

```kotlin
@Composable
fun CameraFeatureScreen(
    viewModel: CameraViewModel = hiltViewModel()
) {
    val cameraPermissionState = rememberPermissionState(Manifest.permission.CAMERA)
    val context = LocalContext.current

    LaunchedEffect(Unit) {
        // Collect one-time events
    }

    when {
        cameraPermissionState.status.isGranted -> {
            CameraPreview()
        }
        cameraPermissionState.status.shouldShowRationale -> {
            RationaleCard(
                message = "Camera access lets you take photos and join video calls.",
                onConfirm = { cameraPermissionState.launchPermissionRequest() }
            )
        }
        else -> {
            PermissionDeniedCard(
                message = "Camera access was denied. Enable it in Settings to use this feature.",
                onOpenSettings = {
                    context.startActivity(
                        Intent(Settings.ACTION_APPLICATION_DETAILS_SETTINGS).apply {
                            data = Uri.fromParts("package", context.packageName, null)
                        }
                    )
                }
            )
        }
    }
}

@Composable
private fun RationaleCard(message: String, onConfirm: () -> Unit) {
    Card(modifier = Modifier.fillMaxWidth().padding(16.dp)) {
        Column(modifier = Modifier.padding(16.dp)) {
            Text(text = "Permission Needed", style = MaterialTheme.typography.titleMedium)
            Spacer(modifier = Modifier.height(8.dp))
            Text(text = message)
            Spacer(modifier = Modifier.height(16.dp))
            Button(onClick = onConfirm, modifier = Modifier.align(Alignment.End)) {
                Text("Grant Permission")
            }
        }
    }
}
```

### Photo Picker (No Permission Required — API 33+)

```kotlin
class PhotoPickerActivity : AppCompatActivity() {

    // Photo Picker: available from API 33+, backported to API 21 via Play Store
    private val pickMedia = registerForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri ->
        if (uri != null) {
            // Persist permission for long-term access
            contentResolver.takePersistableUriPermission(
                uri, Intent.FLAG_GRANT_READ_URI_PERMISSION
            )
            processSelectedPhoto(uri)
        }
    }

    // Multiple photo selection
    private val pickMultipleMedia = registerForActivityResult(
        ActivityResultContracts.PickMultipleVisualMedia(maxItems = 5)
    ) { uris ->
        uris.forEach { uri ->
            contentResolver.takePersistableUriPermission(
                uri, Intent.FLAG_GRANT_READ_URI_PERMISSION
            )
        }
        processSelectedPhotos(uris)
    }

    fun launchSinglePhotoPicker() {
        pickMedia.launch(
            PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
        )
    }

    fun launchMultiplePhotoPicker() {
        pickMultipleMedia.launch(
            PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageAndVideo)
        )
    }

    private fun processSelectedPhoto(uri: Uri) { /* Load with Glide/Coil */ }
    private fun processSelectedPhotos(uris: List<Uri>) { /* Process batch */ }
}
```

---

## Best Practices

1. **Request in context** — Only request permissions when the user triggers the feature that needs them.
2. **Explain before requesting** — Show rationale UI when `shouldShowRequestPermissionRationale()` is `true`.
3. **Graceful degradation** — Always provide an alternative flow when permission is denied. Never crash or block.
4. **Open Settings for permanent denial** — Don't re-request; redirect to `ACTION_APPLICATION_DETAILS_SETTINGS`.
5. **Never cache permission results long-term** — Always check via `checkSelfPermission()` at the point of use.
6. **Re-check in `onResume()`** — Handles one-time permissions and Settings revocations.
7. **Use `ActivityResultContracts`** — Prefer modern approach over deprecated `requestPermissions()`.
8. **Request minimal permissions** — Request only what's needed for the current feature, not everything upfront.
9. **Use Photo Picker over storage permissions** — Simpler, more privacy-respecting, no permission needed.
10. **Handle API level differences** — `READ_MEDIA_IMAGES` on API 33+, `READ_EXTERNAL_STORAGE` on older APIs.
11. **Test permission flows** — Test all states: granted, denied, permanently denied, one-time grant.
12. **Declare `maxSdkVersion`** — For permissions not needed on newer APIs: `android:maxSdkVersion="32"`.

---

## Legacy vs Modern

| Feature | Legacy Approach | Modern Approach |
|---|---|---|
| Permission request | `requestPermissions()` + `onRequestPermissionsResult()` | `registerForActivityResult(RequestPermission())` |
| Multiple permissions | Multiple request codes, complex switch statement | `RequestMultiplePermissions()` contract |
| Media access | `READ_EXTERNAL_STORAGE` | `READ_MEDIA_IMAGES/VIDEO/AUDIO` (API 33+) |
| Photo selection | Storage permission + file picker | **Photo Picker API** (no permission) |
| Location access | `ACCESS_FINE_LOCATION` always | `ACCESS_COARSE_LOCATION` when precision unneeded |
| Background location | Implicit with foreground | `ACCESS_BACKGROUND_LOCATION` separate (API 29+) |
| One-time permissions | Not available | Available for Camera/Mic/Location (API 30+) |
| Bluetooth | `BLUETOOTH` | `BLUETOOTH_SCAN` / `BLUETOOTH_CONNECT` (API 31+) |
| Install-time model | API <23 (all upfront) | API 23+ (runtime, dangerous perms) |

---

## Revision Notes

- API 23+ → dangerous permissions require **runtime request**
- Normal permissions → **auto-granted** at install if in manifest
- `checkSelfPermission()` → check before any sensitive API call
- `shouldShowRequestPermissionRationale()`:
  - `true` → previously denied → show rationale
  - `false` → first time OR permanently denied OR never ask again
- Modern API: `registerForActivityResult(ActivityResultContracts.RequestPermission())`
- Permanently denied → open `ACTION_APPLICATION_DETAILS_SETTINGS`
- One-time permissions (API 30+): Camera, Mic, Location → auto-revoked on background
- Re-check permission in `onResume()` for one-time grant scenarios
- API 33+: `READ_EXTERNAL_STORAGE` → deprecated; use `READ_MEDIA_IMAGES/VIDEO/AUDIO`
- Photo Picker API: **no permission needed** for photo/video selection
- `MANAGE_EXTERNAL_STORAGE` → restricted on Play Store; only for file managers
- API 29+: `ACCESS_BACKGROUND_LOCATION` is a **separate** permission
- API 31+: `BLUETOOTH_SCAN`, `BLUETOOTH_CONNECT` replace monolithic `BLUETOOTH`

---

## Key Takeaways

> Runtime permissions are Android's commitment to **user privacy and informed consent**. The fundamental shift from API 22 to API 23 changed Android from an install-time trust model to a contextual, revocable permission model. As a developer, your responsibility is to ask for **minimal permissions at the right moment**, handle **all denial states gracefully**, and never block the user from your app due to a denied permission. The evolution continues — one-time permissions, granular media permissions, and the Photo Picker API all push toward a future where apps need fewer permissions, not more.

**The 5 things every interview candidate must know:**
1. API 23+: dangerous permissions require **runtime request** — normal = auto-granted
2. `shouldShowRequestPermissionRationale()` = `true` → show rationale; `false` → first-ask or permanently denied
3. Modern API: `registerForActivityResult(ActivityResultContracts.RequestPermission())`
4. Permanently denied → **open Settings** (`ACTION_APPLICATION_DETAILS_SETTINGS`), never re-request
5. API 33+: `READ_EXTERNAL_STORAGE` deprecated → use `READ_MEDIA_IMAGES/VIDEO/AUDIO` or **Photo Picker**

---

# Part 3 Summary

| Chapter | Core Component | Key Class | Modern Alternative / Best Practice |
|---|---|---|---|
| 9 | BroadcastReceiver | `BroadcastReceiver` | `SharedFlow` / `WorkManager` |
| 10 | ContentProvider | `ContentProvider` + `ContentResolver` | `Room` (internal) / `FileProvider` (files) |
| 11 | Runtime Permissions | `ActivityResultContracts` | Photo Picker / Granular media perms |

---

*End of Part 3 — Android Fundamentals Master Guide*
*Next: Part 4 — Services, WorkManager, and Background Execution*
