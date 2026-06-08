# Part 5: Memory, Performance & Advanced Android

> **Android Fundamentals Master Guide — Part 5**
> Chapters 17–20 | Memory Leaks · GC · ANR · Performance · Hilt DI · Jetpack Compose

---

## Table of Contents

| Chapter | Topic |
|---------|-------|
| [Chapter 17](#chapter-17-memory-leaks--garbage-collection) | Memory Leaks & Garbage Collection |
| [Chapter 18](#chapter-18-anr-performance--best-practices) | ANR, Performance & Best Practices |
| [Chapter 19](#chapter-19-dependency-injection-with-hilt) | Dependency Injection with Hilt |
| [Chapter 20](#chapter-20-jetpack-compose-modern-ui-vs-xml-views) | Jetpack Compose (Modern UI) vs XML Views |

---

# Chapter 17: Memory Leaks & Garbage Collection

---

## Concept

A **memory leak** occurs when an object that is no longer needed by the application is still reachable through a chain of strong references, preventing the Garbage Collector (GC) from reclaiming its memory. Over time, leaked objects accumulate on the heap, starving the app of memory, triggering frequent GCs, degrading performance, and ultimately causing an `OutOfMemoryError` crash.

In Android, this problem is especially severe because **Activities, Fragments, and Views are large objects** with well-defined lifecycles. If any longer-lived component holds a reference to one of these, the entire view hierarchy, bitmaps, and associated resources remain in memory long after the UI has been destroyed.

**Key distinction:**
- **Memory Leak** → Object is alive but should be dead (GC cannot collect it).
- **OOM** → No memory left to allocate new objects (often caused by accumulated leaks).
- **Memory Pressure** → System is low on RAM; Android may kill background processes.

---

## Why It Exists

Android manages memory through the **Java Virtual Machine (JVM)** / **Android Runtime (ART)** with automatic garbage collection. Developers coming from languages like C++ are not used to thinking about manual memory management, but Android introduces unique challenges:

1. **Component lifecycle mismatch** — Activities are created/destroyed frequently (rotation, back-stack), but many constructs (Singletons, static fields, Handler messages) survive beyond a single Activity instance.
2. **Context proliferation** — `Context` is everywhere in Android APIs. Using the wrong context (Activity vs Application) in long-lived objects is a classic source of leaks.
3. **Async operations** — Network calls, database queries, and background threads often complete after the originating Activity has been destroyed. If they hold a reference back to it, the Activity leaks.
4. **Framework callbacks** — Registering listeners (Sensors, Location, Broadcasts) with system services requires explicit unregistration; the system holds a strong reference to your listener.

---

## Internal Working

### ART Garbage Collection

Android Runtime (ART, replacing Dalvik from API 21+) uses a **generational, concurrent GC**:

```
┌─────────────────────────────────────────────────────────────┐
│                        JVM/ART HEAP                         │
│                                                             │
│  ┌──────────────────────────────┐   ┌────────────────────┐  │
│  │       Young Generation       │   │  Old Generation    │  │
│  │  ┌────────┐ ┌────┐ ┌────┐   │   │  (Tenured Space)   │  │
│  │  │  Eden  │ │ S0 │ │ S1 │   │──▶│  Long-lived objs   │  │
│  │  │(alloc) │ │    │ │    │   │   │  Large objects      │  │
│  │  └────────┘ └────┘ └────┘   │   └────────────────────┘  │
│  │   Minor GC happens here      │   Major/Full GC here      │
│  └──────────────────────────────┘                           │
│                                                             │
│  ┌─────────────────────────────────────────────────────────┐ │
│  │  Metaspace (was PermGen pre-Java 8)                     │ │
│  │  Class metadata, method bytecode, interned Strings      │ │
│  └─────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────┘
```

**GC Process:**
1. **Minor GC (Young Generation)** — Fast, stops-the-world briefly; collects short-lived objects (Eden → Survivor spaces). Objects that survive multiple GC cycles are promoted to Old Generation.
2. **Major GC (Old Generation)** — Slower; collects long-lived objects. Triggered when Old Gen is full.
3. **Full GC** — Collects entire heap. Causes noticeable pauses (jank). ART's concurrent GC tries to minimise STW pauses.

**ART GC Improvements over Dalvik:**
- Concurrent copying GC (moves live objects, compacts heap — reduces fragmentation)
- Generational concurrent copying (API 29+ / Android 10+)
- Reduced GC pause times via background GC threads

### Reference Types in Java/Kotlin

| Reference Type | GC Behavior | Use Case |
|----------------|-------------|----------|
| **Strong** (`val obj = Obj()`) | Never collected while reachable | Default — all normal references |
| **SoftReference** | Collected under memory pressure | Memory-sensitive caches |
| **WeakReference** | Collected at next GC if no strong refs | Caches, breaking cycles (Handler) |
| **PhantomReference** | Collected after finalization | Pre-cleanup hooks (advanced) |

---

## Lifecycle / Flow (ASCII Diagrams)

### GC Root Chain — What Causes a Leak

```
  GC Roots (never collected)
  ├── Static fields
  ├── Local variables on active thread stacks
  ├── JNI references
  └── System class loader

  Example: Activity Leak via Static Field
  ┌─────────────────────────────────────────────┐
  │ Static Field (GC Root)                      │
  │    │                                         │
  │    ▼                                         │
  │ MySingleton.instance                         │
  │    │                                         │
  │    ▼                                         │
  │ MySingleton.context ──────────────────────▶ MainActivity
  │                                              (should be dead,
  │                                               but GC cannot
  │                                               collect it!)
  └─────────────────────────────────────────────┘
```

### Memory Leak Detection Flow

```
App Running
    │
    ▼
Object Created (e.g., Activity)
    │
    ▼
Activity Destroyed (onDestroy called)
    │
    ├──── GC triggered?
    │         │
    │         ▼
    │    Is Activity still reachable?
    │         │
    │    YES  │   NO
    │         │    └──── Memory reclaimed ✓
    │         ▼
    │    MEMORY LEAK! 💥
    │    (LeakCanary reports chain)
    │
    ▼
Profiler shows heap growing over time
```

### onTrimMemory Levels

```
System Memory Pressure Scale:

LOW PRESSURE ──────────────────────────────── HIGH PRESSURE
    │                                               │
    ▼                                               ▼
TRIM_MEMORY_UI_HIDDEN          TRIM_MEMORY_COMPLETE
(20) - App moved to bg         (80) - Critical! Release all
                                       caches immediately
    TRIM_MEMORY_RUNNING_MODERATE  TRIM_MEMORY_RUNNING_CRITICAL
    (5) - Device is low           (15) - Very low, kill bg procs
```

---

## Real World Example

**Scenario:** A news app streams articles. The `ArticleActivity` registers a `NetworkCallback` to track connectivity changes. On device rotation, the old `ArticleActivity` is destroyed, a new one created, but the old one is never garbage collected because `ConnectivityManager` holds a reference to the registered `NetworkCallback` which holds a reference back to the Activity.

---

## Common Mistakes

### 1. Static Reference to Activity

```kotlin
// ❌ WRONG — Classic leak: static field holds Activity reference
class DatabaseManager private constructor() {
    companion object {
        // This Activity reference will NEVER be GC'd
        var currentActivity: Activity? = null
        val instance = DatabaseManager()
    }
}

// Usage:
DatabaseManager.currentActivity = this // In onCreate — LEAKS!
```

```kotlin
// ✅ CORRECT — Use Application context for singletons
class DatabaseManager private constructor(context: Context) {
    companion object {
        @Volatile private var instance: DatabaseManager? = null
        fun getInstance(context: Context) =
            instance ?: synchronized(this) {
                instance ?: DatabaseManager(context.applicationContext).also { instance = it }
            }
    }
    // context is now ApplicationContext — lives as long as the app
}
```

---

### 2. Non-Static Inner Class / Anonymous Listener

```kotlin
// ❌ WRONG — Inner class holds implicit reference to outer Activity
class MainActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        // This Runnable captures `this` (MainActivity) implicitly!
        handler.postDelayed({
            updateUI() // If Activity is destroyed, this still runs
        }, 10_000)
    }
}
```

```kotlin
// ✅ CORRECT — WeakReference pattern for Handler
class MainActivity : AppCompatActivity() {
    private lateinit var handler: Handler
    private lateinit var updateTask: Runnable

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val weakActivity = WeakReference(this)
        handler = Handler(Looper.getMainLooper())
        updateTask = Runnable {
            weakActivity.get()?.updateUI() // Safe: only runs if Activity alive
        }
        handler.postDelayed(updateTask, 10_000)
    }

    override fun onDestroy() {
        super.onDestroy()
        handler.removeCallbacks(updateTask) // Cancel pending messages
    }
}
```

---

### 3. RxJava Subscription Not Disposed

```kotlin
// ❌ WRONG — Subscription outlives Activity
class ProfileActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        userRepository.getUserStream()
            .subscribeOn(Schedulers.io())
            .observeOn(AndroidSchedulers.mainThread())
            .subscribe { user -> updateProfile(user) } // Leaked!
    }
}
```

```kotlin
// ✅ CORRECT — CompositeDisposable pattern
class ProfileActivity : AppCompatActivity() {
    private val disposables = CompositeDisposable()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        disposables.add(
            userRepository.getUserStream()
                .subscribeOn(Schedulers.io())
                .observeOn(AndroidSchedulers.mainThread())
                .subscribe(
                    { user -> updateProfile(user) },
                    { error -> handleError(error) }
                )
        )
    }

    override fun onDestroy() {
        super.onDestroy()
        disposables.clear() // Dispose all subscriptions
    }
}
```

---

### 4. Coroutine Leak from GlobalScope

```kotlin
// ❌ WRONG — GlobalScope ignores lifecycle; runs forever
class SearchActivity : AppCompatActivity() {
    fun onSearchClick() {
        GlobalScope.launch { // This scope is never cancelled
            val results = repository.search(query)
            withContext(Dispatchers.Main) {
                showResults(results) // Activity may be dead!
            }
        }
    }
}
```

```kotlin
// ✅ CORRECT — lifecycleScope is cancelled when Activity is destroyed
class SearchActivity : AppCompatActivity() {
    fun onSearchClick() {
        lifecycleScope.launch {
            try {
                val results = repository.search(query)
                showResults(results) // Safe: only runs if Activity alive
            } catch (e: CancellationException) {
                // Scope was cancelled (Activity destroyed) — expected
            }
        }
    }
}
```

---

### 5. Unregistered BroadcastReceiver

```kotlin
// ❌ WRONG — Receiver registered but never unregistered
class NetworkActivity : AppCompatActivity() {
    private val networkReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            updateNetworkUI()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        registerReceiver(networkReceiver,
            IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION))
        // No corresponding unregisterReceiver call! Activity leaks.
    }
}
```

```kotlin
// ✅ CORRECT — Register/unregister symmetrically
class NetworkActivity : AppCompatActivity() {
    private val networkReceiver = object : BroadcastReceiver() {
        override fun onReceive(context: Context, intent: Intent) {
            updateNetworkUI()
        }
    }

    override fun onStart() {
        super.onStart()
        registerReceiver(networkReceiver,
            IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION))
    }

    override fun onStop() {
        super.onStop()
        unregisterReceiver(networkReceiver)
    }
}
```

---

## Memory Leak / Performance Concerns

| Leak Source | Impact | Detection | Fix |
|-------------|--------|-----------|-----|
| Static Activity ref | Entire Activity + View tree leaked | LeakCanary | Use `applicationContext`; nullify in `onDestroy` |
| Non-static inner class | Outer class (Activity) kept alive | Android Profiler heap dump | Use `WeakReference` or static nested class |
| Handler messages | Activity kept alive until message fires | LeakCanary | Remove callbacks in `onDestroy` |
| Unclosed Cursor | Memory + SQLite connection leak | StrictMode | Use `use {}` block (auto-close) |
| Bitmap not recycled | Large native memory held (pre-API 26) | Memory Profiler | Use Glide/Coil; call `recycle()` when done |
| RxJava not disposed | Subscribers run on dead Activity | LeakCanary | `CompositeDisposable.clear()` in `onDestroy` |
| Coroutine GlobalScope | Unbound coroutines, phantom updates | — | Use `lifecycleScope` / `viewModelScope` |
| EventBus not unregistered | Bus holds subscriber reference | — | Unregister in `onStop` / `onDestroy` |

---

## Code Examples (Production-Quality Kotlin)

### LeakCanary Setup

```kotlin
// app/build.gradle.kts
dependencies {
    debugImplementation("com.squareup.leakcanary:leakcanary-android:2.14")
    // No release dependency needed — LeakCanary auto-installs in debug builds only
}
```

### Implementing onTrimMemory

```kotlin
class MyApplication : Application() {
    private val imageCache = LruCache<String, Bitmap>(calculateCacheSize())

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        when {
            level >= ComponentCallbacks2.TRIM_MEMORY_COMPLETE -> {
                // Extreme: release everything
                imageCache.evictAll()
                Log.w("Memory", "TRIM_MEMORY_COMPLETE — releasing all caches")
            }
            level >= ComponentCallbacks2.TRIM_MEMORY_MODERATE -> {
                // Moderate: release half the cache
                imageCache.trimToSize(imageCache.size() / 2)
            }
            level >= ComponentCallbacks2.TRIM_MEMORY_BACKGROUND -> {
                // Background: trim to 25%
                imageCache.trimToSize(imageCache.maxSize() / 4)
            }
            level >= ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN -> {
                // UI hidden: a good time to free non-critical UI resources
                Log.d("Memory", "UI hidden — freeing UI caches")
            }
        }
    }

    private fun calculateCacheSize(): Int {
        val maxMemory = (Runtime.getRuntime().maxMemory() / 1024).toInt()
        return maxMemory / 8 // Use 1/8th of available memory
    }
}
```

### Proper Bitmap Handling

```kotlin
// ❌ DEPRECATED (API < 26): Manual bitmap recycling needed
// fun loadBitmap(resId: Int): Bitmap {
//     val bitmap = BitmapFactory.decodeResource(resources, resId)
//     // Must call bitmap.recycle() when done!
//     return bitmap
// }

// ✅ MODERN: Use Glide — handles lifecycle, LRU cache, downsampling
class ArticleFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        Glide.with(this) // Tied to Fragment lifecycle automatically
            .load(article.imageUrl)
            .apply(
                RequestOptions()
                    .diskCacheStrategy(DiskCacheStrategy.AUTOMATIC)
                    .override(Target.SIZE_ORIGINAL)
                    .placeholder(R.drawable.placeholder)
                    .error(R.drawable.error_image)
                    .downsample(DownsampleStrategy.CENTER_INSIDE)
            )
            .into(binding.articleImage)
    }
}
```

### WeakReference Pattern — Complete Example

```kotlin
/**
 * A utility class that holds a weak reference to a callback target,
 * preventing memory leaks when used with long-running operations.
 */
class WeakCallback<T>(target: T) {
    private val weakRef = WeakReference(target)
    val target: T? get() = weakRef.get()
    val isAlive: Boolean get() = weakRef.get() != null
}

// ViewModel using WeakCallback to avoid holding Fragment reference
class DownloadViewModel : ViewModel() {
    private var progressCallback: WeakCallback<ProgressListener>? = null

    interface ProgressListener {
        fun onProgress(percent: Int)
        fun onComplete(filePath: String)
    }

    fun registerProgressListener(listener: ProgressListener) {
        progressCallback = WeakCallback(listener)
    }

    fun startDownload(url: String) {
        viewModelScope.launch(Dispatchers.IO) {
            // Simulate download with progress
            for (progress in 0..100 step 10) {
                delay(500)
                withContext(Dispatchers.Main) {
                    progressCallback?.target?.onProgress(progress)
                }
            }
            withContext(Dispatchers.Main) {
                progressCallback?.target?.onComplete("/sdcard/file.zip")
            }
        }
    }

    fun unregisterProgressListener() {
        progressCallback = null
    }
}
```

### Safe Cursor Usage with `use {}`

```kotlin
// ❌ WRONG — Cursor may not be closed on exception
fun getContactName(id: Long): String? {
    val cursor = contentResolver.query(
        ContactsContract.Contacts.CONTENT_URI,
        arrayOf(ContactsContract.Contacts.DISPLAY_NAME),
        "${ContactsContract.Contacts._ID} = ?",
        arrayOf(id.toString()),
        null
    )
    val name = if (cursor != null && cursor.moveToFirst()) {
        cursor.getString(0)
    } else null
    cursor?.close() // Might be skipped if exception thrown above
    return name
}

// ✅ CORRECT — use{} ensures close() is always called
fun getContactName(id: Long): String? {
    return contentResolver.query(
        ContactsContract.Contacts.CONTENT_URI,
        arrayOf(ContactsContract.Contacts.DISPLAY_NAME),
        "${ContactsContract.Contacts._ID} = ?",
        arrayOf(id.toString()),
        null
    )?.use { cursor ->
        if (cursor.moveToFirst()) cursor.getString(0) else null
    }
}
```

---

## Best Practices

1. **Always use `applicationContext` in Singletons** — Never store `Activity` context in any object that outlives the Activity.
2. **Pair every register with an unregister** — `registerReceiver`/`unregisterReceiver`, `addObserver`/`removeObserver`, `subscribe`/`dispose`.
3. **Use `lifecycleScope` and `viewModelScope`** — These are automatically cancelled at the appropriate lifecycle event.
4. **Integrate LeakCanary in every debug build** — It provides exact reference chains for every detected leak.
5. **Use `use {}` for Closeable resources** — Cursors, Streams, Database connections.
6. **Prefer Glide/Coil over manual Bitmap management** — They handle lifecycle, caching, and memory limits automatically.
7. **Never call `System.gc()`** — It's a suggestion, not a guarantee; it disrupts GC heuristics.
8. **Avoid static references to Views** — Views hold a reference to their Context (Activity).
9. **Use `viewModelScope` for async work** — ViewModel survives rotation; its scope is cancelled only when ViewModel is cleared.
10. **Implement `onTrimMemory()`** — Proactively release caches when the system signals memory pressure.

---

## Legacy vs Modern

| Aspect | Legacy (Pre-2019) | Modern |
|--------|-------------------|--------|
| Background tasks | `AsyncTask` ⚠️ **Deprecated API 30** | Coroutines + `lifecycleScope`/`viewModelScope` |
| Image loading | Manual `Bitmap` management | Glide / Coil (lifecycle-aware) |
| Leak detection | Manual heap dumps via DDMS | LeakCanary (automatic notification) |
| Memory callbacks | `onLowMemory()` | `onTrimMemory(level)` (granular) |
| GC runtime | Dalvik (Android < 5.0) | ART with concurrent/generational GC |
| Thread-to-Activity refs | `Handler` with anonymous Runnable | `lifecycleScope.launch {}` |

> ⚠️ **Deprecated:** `AsyncTask` is deprecated in API level 30. Migrate to Kotlin Coroutines or `java.util.concurrent`.

---

## Interview Questions

### Beginner

**Q1: What is a memory leak in Android?**
> A memory leak occurs when an object that is no longer needed is still referenced by a GC root, preventing the garbage collector from reclaiming its memory. In Android, the most common form is an Activity or Fragment reference being held by a longer-lived object (Singleton, static field, background thread), causing the Activity to remain in memory even after `onDestroy()` is called.

**Q2: What is the difference between a memory leak and an OOM error?**
> A memory leak is the *cause*; OOM (OutOfMemoryError) is often the *consequence* of accumulated leaks. A leak means an object survives longer than intended. OOM means the heap has no more space to allocate new objects. You can have an OOM without a leak (e.g., loading a very large Bitmap), and you can have leaks that don't cause OOM if the app is restarted before memory is exhausted.

**Q3: What is `WeakReference` and when do you use it?**
> A `WeakReference` wraps an object so the GC can collect it if there are no strong references pointing to it. The `WeakReference` itself is not a GC root. You use it when you want to reference an object that you don't want to *own* — for example, caching an Activity reference in a background task. `weakRef.get()` returns `null` after GC collects the target.

---

### Intermediate

**Q4: Why is a non-static inner class a source of memory leaks?**
> In Kotlin/Java, a non-static inner class holds an implicit reference to its enclosing outer class instance. This means if you create an inner class `Runnable`, `Listener`, or `Callback` inside an Activity, the inner class object holds a strong reference to the Activity. If the inner class is posted to a Handler or passed to a system service, the Activity cannot be GC'd until the callback fires (or never, if it's registered indefinitely).

**Q5: Explain the difference between `SoftReference` and `WeakReference`.**
> Both allow GC to collect the referent, but with different urgency. A **WeakReference** is collected at the *next* GC cycle when there are no strong references. A **SoftReference** is more lenient — the GC *may* keep the object alive if there's sufficient memory, but *will* collect it under memory pressure. `SoftReference` is suitable for memory-sensitive image caches; `WeakReference` is preferred for preventing leaks in callbacks and listeners.

**Q6: How does ART's generational GC work?**
> ART uses a two-generation model. New objects are allocated in the **Young Generation** (Eden space). When Eden fills, a **Minor GC** runs, copying surviving objects to Survivor spaces (S0/S1) and eventually promoting long-lived objects to the **Old Generation**. A **Major GC** periodically collects the Old Generation. ART (API 29+) adds **concurrent copying GC**, which compacts the heap while the app runs, reducing memory fragmentation and pause times compared to Dalvik.

---

### Advanced

**Q7: How would you diagnose a memory leak in a production app?**
> **Development:** Integrate LeakCanary — it automatically detects and reports leaked objects with the full reference chain. Use Android Studio's Memory Profiler to take heap dumps and inspect the retained object graph.
> **Production:** Use Firebase Performance Monitoring to track memory trends. Instrument `onTrimMemory()` to log memory events. Use the `Debug.MemoryInfo` API to report native/Java heap usage to your analytics backend. For post-mortem analysis, analyze ANR/crash reports which often include OOM stack traces.

**Q8: What is the "Leaking Activity" pattern in Hilt/ViewModel and how do you avoid it?**
> With Hilt, if a `@Singleton`-scoped dependency stores a reference to an `Activity`-scoped resource, the Activity leaks because the Singleton lives for the entire app lifetime. Solution: Use the correct Hilt scope (`@ActivityScoped`, `@ViewModelScoped`), use `applicationContext` for singletons, and never inject `Activity` context into a `SingletonComponent`. For cases where you need a ViewModel to survive configuration changes, inject `Application` through `@HiltAndroidApp` using `ApplicationContext`.

**Q9: How do coroutine scopes prevent memory leaks?**
> `lifecycleScope` is a `CoroutineScope` extension property on `LifecycleOwner` (Activity/Fragment) that is tied to the `Lifecycle`. When the `Lifecycle` reaches `DESTROYED` state, the scope is automatically cancelled via a `LifecycleEventObserver`. All coroutines launched within this scope receive a `CancellationException` and clean up. Similarly, `viewModelScope` is cancelled in `ViewModel.onCleared()`. Using these structured scopes means coroutines can never outlive their owner, eliminating the entire class of "coroutine reference to dead Activity" leaks.

---

## Scenario Questions

**Scenario 1:** Your app's memory usage grows steadily after the user navigates between Activities multiple times. Opening the Memory Profiler shows the heap never decreases after each rotation. What are the likely causes and how would you diagnose?

> **Approach:** This is a classic Activity leak on rotation. I would:
> 1. Add LeakCanary to the debug build and reproduce the navigation pattern.
> 2. LeakCanary will report the reference chain holding the Activity.
> 3. Common suspects: static fields, Singleton holding Activity context, ViewModel accidentally scoped to Activity, Hilt dependency with wrong scope.
> 4. Take a heap dump from Android Profiler → filter by "Activity" type → any instance count > 1 for the same Activity class indicates a leak.
> 5. Fix by ensuring Singletons use `applicationContext`, ViewModels use `viewModelScope`, and no static fields hold Activity/View/Context references.

**Scenario 2:** A user reports the app crashes with OOM after loading many high-resolution images in a RecyclerView gallery. How would you fix this?

> **Approach:**
> 1. Replace manual `BitmapFactory.decodeFile()` calls with Glide or Coil, which handle lifecycle-aware loading, LRU caching, and automatic downsampling.
> 2. Use `BitmapFactory.Options.inSampleSize` to load images at a fraction of their original size if loading manually.
> 3. Call `Glide.with(fragment).clear(imageView)` when views are recycled if using a custom adapter.
> 4. Set a Glide memory cache size appropriate for the device: `Glide.get(context).setMemoryCategory(MemoryCategory.HIGH)` for gallery-heavy apps.
> 5. Implement `RecyclerView.RecycledViewPool` to share recycled views across nested lists.
> 6. Ensure `RecyclerView.Adapter` uses `DiffUtil` to avoid unnecessary rebinds that trigger image reloads.

---

## Revision Notes

- **ART** replaced Dalvik as of Android 5.0 (API 21). Dalvik used JIT compilation; ART uses AOT (Ahead-Of-Time) + JIT.
- **GC Roots:** The GC starts from roots (static fields, active threads, JNI refs) and traces all reachable objects. Everything NOT reachable is garbage.
- **`System.gc()`** is merely a *hint* to the VM — never rely on it. The GC runs on its own schedule.
- **LeakCanary** works by installing a `WeakReference` watch on Activities after `onDestroy()`. If after a GC the reference isn't cleared, it reports a leak.
- **`onTrimMemory()`** supersedes the older **`onLowMemory()`** callback, providing 7 granular levels of memory pressure.
- **`applicationContext` vs `activityContext`:** Use `applicationContext` for singletons, database helpers, and anything that outlives a single Activity. Use `activityContext` only for UI operations (dialogs, LayoutInflater, window decorations).

---

## Key Takeaways

- 🧠 A memory leak = object alive but should be dead. GC cannot help if strong references exist.
- 🏗️ ART uses generational GC: Young Gen (fast Minor GC) → Old Gen (slower Major GC).
- 💀 The #1 Android leak: holding `Activity` or `View` reference in a longer-lived object.
- 🔗 Reference strength: Strong > Soft > Weak > Phantom. Use `WeakReference` to break retain cycles.
- 🛡️ `lifecycleScope` and `viewModelScope` eliminate entire categories of async leaks via structured concurrency.
- 🔍 LeakCanary is the gold standard for development-time leak detection.
- ♻️ Always pair register/subscribe with unregister/dispose in symmetric lifecycle methods.
- 📱 Implement `onTrimMemory()` to proactively release caches before the system kills your process.

---

---

# Chapter 18: ANR, Performance & Best Practices

---

## Concept

**ANR (Application Not Responding)** is Android's mechanism to kill apps that block the main thread (UI thread) for too long, causing the UI to become unresponsive. The system shows a dialog: *"App isn't responding. Do you want to close it?"*

Beyond ANRs, **performance** encompasses smooth animations (60+ fps), fast app startup, efficient battery usage, and optimal memory consumption. A performant Android app feels instant, never janks, and doesn't drain the battery.

**Key Metrics:**
- **Jank** — Frame drop below 60fps (each frame > 16ms) causes visual stuttering.
- **ANR threshold** — Activity: 5 seconds, BroadcastReceiver: 10 seconds, Service: 20 seconds.
- **Cold Start target** — < 500ms (excellent), < 1s (good), > 2s (poor).

---

## Why It Exists

Android's UI is rendered on a single **Main Thread** (also called the UI thread). This design simplifies UI programming by eliminating multi-threading complexities from view rendering. However, it means **any blocking operation on the main thread** — file I/O, network call, database query, heavy computation — directly delays every frame render and user input response.

ANR exists because:
1. Users expect instant UI response.
2. The system needs a mechanism to recover from frozen apps.
3. It provides a signal to developers that their threading model is wrong.

---

## Internal Working

### ANR Detection Mechanism

Android's `ActivityManagerService` (AMS) uses **watchdog timers**:

```
User touches screen
      │
      ▼
Input Event dispatched to app's main thread
      │
      ├── Main thread free? ────▶ Process event immediately ✓
      │
      └── Main thread BLOCKED?
                │
                ▼
         Watchdog timer starts (5s)
                │
                ├── Thread unblocked before 5s? ─▶ No ANR ✓
                │
                └── Still blocked at 5s?
                          │
                          ▼
                    ANR triggered! 💥
                    - SIGQUIT sent to app
                    - Trace dumped to /data/anr/traces.txt
                    - Dialog shown to user
```

### Choreographer & Frame Rendering

```
VSYNC Signal (every 16.67ms @ 60Hz)
      │
      ▼
  Choreographer.doFrame()
      │
      ├── Input callbacks (touch events)
      ├── Animation callbacks
      └── Traversal callbacks
               │
               ├── measure()
               ├── layout()
               └── draw()
                     │
                     ▼
               Rendered to display ✓ (if < 16ms)
               OR
               Frame SKIPPED 💥 (if > 16ms → jank!)
```

### Cold vs Warm vs Hot Start

```
COLD START (app not in memory):
[Launch Intent] → [Zygote fork] → [Application.onCreate()] → [Activity.onCreate()] → [First frame] → [Interactive]
 ↑                                  ↑
 slowest                    heavy work here = slow start

WARM START (app in background, Activity destroyed):
[Launch Intent] → [Activity.onCreate()] → [First frame] → [Interactive]
 (skips Application.onCreate)

HOT START (Activity in back stack):
[Launch Intent] → [Activity.onStart/onResume()] → [Interactive]
 (fastest — no inflation needed)
```

---

## Lifecycle / Flow (ASCII Diagrams)

### StrictMode Flow

```
Application.onCreate() [DEBUG ONLY]
       │
       ▼
StrictMode.setThreadPolicy(...)
StrictMode.setVmPolicy(...)
       │
       ▼
App Runs → Violation Detected:
       │
       ├── Disk read on main thread? → Penalty!
       ├── Network on main thread?   → Penalty!
       ├── Unclosed Closeable?       → Penalty!
       └── Leaked Activity?          → Penalty!

Penalty options:
  LOG → Print to logcat
  DIALOG → Show dialog
  DEATH → Crash the app (best for CI)
  DEATH_ON_NETWORK → Crash only on network violations
```

### Overdraw Visualization

```
Without optimization:          With optimization:
┌────────────────┐             ┌────────────────┐
│ Root BG (draw 1)│            │ Root BG (draw 1)│
│ ┌────────────┐ │             │ ┌────────────┐ │
│ │ Layout BG  │ │             │ │  No BG     │ │ ← Background removed
│ │ (draw 2)   │ │             │ │ transparent│ │   (already covered)
│ │ ┌────────┐ │ │             │ │ ┌────────┐ │ │
│ │ │Card BG │ │ │             │ │ │Card BG │ │ │
│ │ │(draw 3)│ │ │             │ │ │(draw 2)│ │ │
│ │ └────────┘ │ │             │ │ └────────┘ │ │
│ └────────────┘ │             │ └────────────┘ │
└────────────────┘             └────────────────┘
 3x overdraw (RED) 🔴           2x overdraw (GREEN) 🟢
```

---

## Real World Example

**Scenario:** An e-commerce app loads product categories on `Activity.onCreate()` by querying a local Room database synchronously on the main thread. On older devices, this causes an ANR because the database query takes 6+ seconds when the catalogue has 50,000 products.

**Fix:** Move the database query to a coroutine with `Dispatchers.IO`; the main thread remains free to handle input events while data loads in the background.

---

## Common Mistakes

### 1. Network/Disk I/O on Main Thread

```kotlin
// ❌ WRONG — Network on main thread → crash on Android 3.0+ (NetworkOnMainThreadException)
class ProductActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val products = httpClient.get("https://api.example.com/products") // CRASH!
        displayProducts(products)
    }
}
```

```kotlin
// ✅ CORRECT — I/O on Dispatchers.IO, result delivered to Main
class ProductActivity : AppCompatActivity() {
    private val viewModel: ProductViewModel by viewModels()

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_product)

        viewModel.products.observe(this) { products ->
            displayProducts(products) // Called on main thread safely
        }

        viewModel.loadProducts()
    }
}

class ProductViewModel(private val repository: ProductRepository) : ViewModel() {
    private val _products = MutableLiveData<List<Product>>()
    val products: LiveData<List<Product>> = _products

    fun loadProducts() {
        viewModelScope.launch {
            val result = withContext(Dispatchers.IO) {
                repository.fetchProducts() // I/O on background thread
            }
            _products.value = result // Delivered to main thread
        }
    }
}
```

---

### 2. Not Using DiffUtil in RecyclerView

```kotlin
// ❌ WRONG — notifyDataSetChanged() redraws EVERYTHING
class ProductAdapter : RecyclerView.Adapter<ProductViewHolder>() {
    private var products = listOf<Product>()

    fun updateProducts(newProducts: List<Product>) {
        products = newProducts
        notifyDataSetChanged() // Full rebind + redraw — no animations, poor perf
    }
}
```

```kotlin
// ✅ CORRECT — DiffUtil calculates minimal diff; only changed items are redrawn
class ProductAdapter : ListAdapter<Product, ProductViewHolder>(ProductDiffCallback()) {

    override fun onCreateViewHolder(parent: ViewGroup, viewType: Int): ProductViewHolder {
        val binding = ItemProductBinding.inflate(
            LayoutInflater.from(parent.context), parent, false
        )
        return ProductViewHolder(binding)
    }

    override fun onBindViewHolder(holder: ProductViewHolder, position: Int) {
        holder.bind(getItem(position))
    }
}

class ProductDiffCallback : DiffUtil.ItemCallback<Product>() {
    override fun areItemsTheSame(oldItem: Product, newItem: Product): Boolean {
        return oldItem.id == newItem.id // Stable identity check
    }

    override fun areContentsTheSame(oldItem: Product, newItem: Product): Boolean {
        return oldItem == newItem // Structural equality (data class)
    }
}
```

---

### 3. Deep View Hierarchy (Nested LinearLayouts)

```xml
<!-- ❌ WRONG — Deep nesting causes O(n²) measure passes -->
<LinearLayout orientation="vertical">
    <LinearLayout orientation="horizontal">
        <LinearLayout orientation="vertical">
            <LinearLayout orientation="horizontal">
                <!-- Content buried 4 levels deep! -->
                <TextView />
                <ImageView />
            </LinearLayout>
        </LinearLayout>
    </LinearLayout>
</LinearLayout>
```

```xml
<!-- ✅ CORRECT — ConstraintLayout achieves same in 1 level -->
<androidx.constraintlayout.widget.ConstraintLayout>
    <TextView
        android:id="@+id/title"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintStart_toStartOf="parent" />
    <ImageView
        android:id="@+id/icon"
        app:layout_constraintTop_toTopOf="parent"
        app:layout_constraintEnd_toEndOf="parent" />
</androidx.constraintlayout.widget.ConstraintLayout>
```

---

## Memory Leak / Performance Concerns

| Issue | Effect | Tool to Detect | Fix |
|-------|--------|----------------|-----|
| Main thread I/O | ANR (5s timeout) | StrictMode, Profiler | Coroutines `Dispatchers.IO` |
| Overdraw | Jank (>16ms/frame) | GPU Overdraw Debug | Remove redundant backgrounds |
| Deep view hierarchy | Slow measure/layout | Layout Inspector | ConstraintLayout, flat hierarchy |
| `notifyDataSetChanged()` | Full RecyclerView redraw | Frame Profiler | DiffUtil + ListAdapter |
| Cold start > 1s | Poor user experience | `adb shell am start -W` | Lazy init, App Startup library |
| Synchronous SharedPreferences | Main thread block | StrictMode | DataStore (async) |
| Unrestricted background work | Battery drain | Battery Historian | WorkManager with constraints |

---

## Code Examples (Production-Quality Kotlin)

### StrictMode Setup

```kotlin
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectAll()           // Detect disk, network, resource mismatch
                    .penaltyLog()          // Log to logcat
                    .penaltyDeath()        // Crash on violation (catches issues early)
                    .build()
            )
            StrictMode.setVmPolicy(
                StrictMode.VmPolicy.Builder()
                    .detectLeakedSqlLiteObjects()
                    .detectLeakedClosableObjects()
                    .detectActivityLeaks()
                    .detectLeakedRegistrationObjects()
                    .penaltyLog()
                    .build()
            )
        }
    }
}
```

### App Startup Optimization with Lazy Initialization

```kotlin
// ❌ WRONG — All SDKs initialized synchronously in onCreate()
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // These all run on main thread, blocking first frame!
        Analytics.init(this)
        CrashReporter.init(this)
        ImageLoader.init(this)
        NetworkClient.init(this)
        Database.init(this)
    }
}
```

```kotlin
// ✅ CORRECT — Defer non-critical initialization using App Startup library
// Also initializes in dependency order with minimal overhead

// Initializer for Analytics (non-critical — defer until first use)
class AnalyticsInitializer : Initializer<Analytics> {
    override fun create(context: Context): Analytics {
        return Analytics.Builder(context)
            .setTrackingEnabled(true)
            .build()
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// Initializer for Network (needed at startup — run eagerly)
class NetworkInitializer : Initializer<OkHttpClient> {
    override fun create(context: Context): OkHttpClient {
        return OkHttpClient.Builder()
            .connectTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    override fun dependencies(): List<Class<out Initializer<*>>> = emptyList()
}

// AndroidManifest.xml:
// <provider
//     android:name="androidx.startup.InitializationProvider"
//     android:authorities="${applicationId}.androidx-startup"
//     android:exported="false"
//     tools:node="merge">
//     <meta-data android:name="com.example.NetworkInitializer"
//                android:value="androidx.startup.InitializationProvider" />
// </provider>
```

### Measuring App Startup Time

```kotlin
// Macrobenchmark — measures cold/warm/hot start in a separate test module
// :benchmark module, build.gradle.kts
// plugins { id("androidx.benchmark.macro") }

@RunWith(AndroidJUnit4::class)
class StartupBenchmark {

    @get:Rule
    val benchmarkRule = MacrobenchmarkRule()

    @Test
    fun measureColdStartup() {
        benchmarkRule.measureRepeated(
            packageName = "com.example.myapp",
            metrics = listOf(StartupTimingMetric()),
            iterations = 5,
            startupMode = StartupMode.COLD
        ) {
            pressHome()
            startActivityAndWait()
        }
    }

    @Test
    fun measureWarmStartup() {
        benchmarkRule.measureRepeated(
            packageName = "com.example.myapp",
            metrics = listOf(StartupTimingMetric()),
            iterations = 5,
            startupMode = StartupMode.WARM
        ) {
            startActivityAndWait()
        }
    }
}
```

### RecyclerView Performance — Complete Setup

```kotlin
class ProductListFragment : Fragment() {
    private val viewModel: ProductViewModel by viewModels()
    private lateinit var adapter: ProductAdapter
    private var _binding: FragmentProductListBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        adapter = ProductAdapter { product -> onProductClicked(product) }

        binding.recyclerView.apply {
            this.adapter = this@ProductListFragment.adapter
            layoutManager = LinearLayoutManager(requireContext())
            setHasFixedSize(true) // Optimization: tells RV that size won't change
            // RecycledViewPool for sharing across multiple lists:
            setRecycledViewPool(RecycledViewPool().apply {
                setMaxRecycledViews(ProductAdapter.TYPE_PRODUCT, 20)
            })
            // Prefetch items off-screen for smoother scroll
            (layoutManager as LinearLayoutManager).initialPrefetchItemCount = 5
        }

        viewModel.products
            .flowWithLifecycle(lifecycle, Lifecycle.State.STARTED)
            .onEach { adapter.submitList(it) } // submitList uses DiffUtil automatically
            .launchIn(lifecycleScope)
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null // Prevent View binding memory leak
    }
}
```

### Profiling with Perfetto Trace

```kotlin
// Add custom trace sections to measure specific code paths
fun expensiveOperation() {
    // API 29+ (Perfetto-aware)
    val trace = perfetto.Trace.beginSection("ExpensiveOperation")
    try {
        // ... work
        processLargeDataset()
    } finally {
        perfetto.Trace.endSection()
    }
}

// Older API: android.os.Trace
fun anotherOperation() {
    android.os.Trace.beginSection("AnotherOperation")
    try {
        // ... work
    } finally {
        android.os.Trace.endSection()
    }
}
```

### WorkManager for Battery-Efficient Background Work

```kotlin
// Define the work
class SyncWorker(
    context: Context,
    workerParams: WorkerParameters
) : CoroutineWorker(context, workerParams) {

    override suspend fun doWork(): Result {
        return try {
            withContext(Dispatchers.IO) {
                syncRepository.syncAll()
            }
            Result.success()
        } catch (e: IOException) {
            if (runAttemptCount < 3) {
                Result.retry() // Retry with exponential backoff
            } else {
                Result.failure(
                    workDataOf("error" to e.message)
                )
            }
        }
    }
}

// Schedule the work with constraints
class SyncScheduler @Inject constructor(
    private val workManager: WorkManager
) {
    fun schedulePeriodic() {
        val constraints = Constraints.Builder()
            .setRequiredNetworkType(NetworkType.CONNECTED)
            .setRequiresBatteryNotLow(true)
            .setRequiresStorageNotLow(true)
            .build()

        val syncRequest = PeriodicWorkRequestBuilder<SyncWorker>(
            repeatInterval = 6,
            repeatIntervalTimeUnit = TimeUnit.HOURS
        )
            .setConstraints(constraints)
            .setBackoffCriteria(BackoffPolicy.EXPONENTIAL, 30, TimeUnit.MINUTES)
            .addTag("sync_work")
            .build()

        workManager.enqueueUniquePeriodicWork(
            "periodic_sync",
            ExistingPeriodicWorkPolicy.UPDATE,
            syncRequest
        )
    }
}
```

### R8 / ProGuard Configuration

```proguard
# Keep Hilt-generated code
-keep class dagger.hilt.** { *; }
-keep @dagger.hilt.android.HiltAndroidApp class * { *; }
-keep @dagger.hilt.InstallIn class * { *; }

# Keep data classes used with Gson/Moshi serialization
-keep class com.example.myapp.data.model.** { *; }

# Keep Room entities
-keep @androidx.room.Entity class * { *; }

# OkHttp
-dontwarn okhttp3.**
-dontwarn okio.**
-keepnames class okhttp3.internal.publicsuffix.PublicSuffixDatabase

# Coroutines
-keepnames class kotlinx.coroutines.internal.MainDispatcherFactory {}
-keepnames class kotlinx.coroutines.CoroutineExceptionHandler {}
```

---

## Best Practices

1. **Never perform I/O on the Main Thread** — Use coroutines with `Dispatchers.IO` or `viewModelScope`.
2. **Enable StrictMode in debug builds** — Catch violations before they reach production.
3. **Target 60fps (16ms/frame)** — Use Android Profiler's Frame Rendering view to identify slow frames.
4. **Use `ConstraintLayout` for complex layouts** — Single-level hierarchy reduces measure passes.
5. **Always use `DiffUtil`/`ListAdapter`** — Never call `notifyDataSetChanged()` in production code.
6. **Measure before optimizing** — Use Profiler and Macrobenchmark before making premature optimizations.
7. **Use `ViewStub` for conditional views** — Views that aren't always shown shouldn't be inflated at startup.
8. **Enable R8 (shrink + obfuscate)** in release builds — Reduces APK size and improves load time.
9. **Use `DataStore` instead of `SharedPreferences`** — DataStore is async and safe on Main thread.
10. **Constrain WorkManager with network + battery** — Background sync should not drain the battery.

---

## Legacy vs Modern

| Concern | Legacy | Modern |
|---------|--------|--------|
| Background tasks | `IntentService` ⚠️ **Deprecated API 30** | `CoroutineWorker` + WorkManager |
| Scheduling | `AlarmManager` + `Service` | WorkManager (handles Doze, battery) |
| Shared preferences | `SharedPreferences` (synchronous) | `DataStore<Preferences>` (async Flow) |
| List performance | `ListView` + `BaseAdapter` | `RecyclerView` + `ListAdapter` + `DiffUtil` |
| Startup measurement | `adb shell am start -W` | Macrobenchmark library |
| Tracing | DDMS Systrace | Perfetto + Android Profiler |
| Bitmap loading | `BitmapFactory` + manual caching | Glide / Coil |
| Battery scheduling | `JobScheduler` (API 21) | WorkManager (wraps JobScheduler + more) |

> ⚠️ **Deprecated:** `IntentService` is deprecated in API level 30. Migrate to `WorkManager` or `CoroutineWorker`.
> ⚠️ **Deprecated:** `AsyncTask` is deprecated in API level 30.

---

## Interview Questions

### Beginner

**Q1: What is ANR and when does it occur?**
> ANR (Application Not Responding) occurs when the Android system detects that the main (UI) thread is blocked for too long: 5 seconds for user input events (Activity), 10 seconds for a `BroadcastReceiver`, and 20 seconds for a `Service` start/bind. The system shows a dialog asking the user to wait or force-close the app. Traces are written to `/data/anr/traces.txt`.

**Q2: What is the Main Thread in Android?**
> The Main Thread (UI Thread) is the single thread responsible for all UI operations: drawing views, handling touch events, running `Activity` lifecycle callbacks, and executing `Handler` messages. It's created when the app process starts. Only this thread can safely update the UI (calling `View.setText()`, `invalidate()`, etc. from a background thread will throw `CalledFromWrongThreadException`).

**Q3: What is overdraw and how do you reduce it?**
> Overdraw occurs when the GPU renders the same pixel multiple times in a single frame — e.g., a background drawn by the Window, then by a layout, then by a card, then by text. Each layer adds rendering time. You can visualize overdraw in Developer Options → Debug GPU Overdraw. Reduce it by removing redundant backgrounds (`android:background="@null"`), using transparent themes for Activities that have a full-screen custom background, and flattening view hierarchies.

---

### Intermediate

**Q4: What is StrictMode and how do you configure it?**
> `StrictMode` is a developer tool that detects policy violations at runtime. The **ThreadPolicy** detects disk reads/writes and network access on the main thread. The **VmPolicy** detects leaked SQLite objects, `Closeable` objects, Activities, and registered objects. You configure it in `Application.onCreate()`, gated behind `BuildConfig.DEBUG`. Violations can be logged, shown in a dialog, or cause the app to crash (`penaltyDeath()`). It should never be enabled in release builds.

**Q5: Explain cold start vs warm start vs hot start.**
> **Cold start:** The app process doesn't exist. Android forks a new process from Zygote, initializes the JVM, runs `Application.onCreate()`, creates the first `Activity`, inflates layout, draws first frame. Slowest.
> **Warm start:** The app process is alive but the Activity was destroyed (back pressed or killed). The process exists, so Zygote fork is skipped. `Application.onCreate()` is skipped. `Activity.onCreate()` runs. Faster than cold.
> **Hot start:** Both process and Activity exist in memory. `onStart()`/`onResume()` called. Fastest.

**Q6: How does `DiffUtil` work internally?**
> `DiffUtil` implements the **Myers diff algorithm** — an efficient O(N+M) algorithm that finds the minimum edit distance between two lists (insertions, deletions, moves). When you call `ListAdapter.submitList(newList)`, DiffUtil runs the comparison on a background thread, then dispatches the minimal set of `RecyclerView.Adapter.notify*()` calls on the main thread. This produces smooth item animations and avoids the full rebind of `notifyDataSetChanged()`.

---

### Advanced

**Q7: How would you investigate an ANR reported from a production device?**
> ANR traces are stored in `/data/anr/traces.txt` (requires root or `adb bugreport`). The trace shows a thread dump at the moment of ANR. I look for:
> 1. **Main thread state** — Is it BLOCKED (waiting on a lock), WAITING (idle, blocked by a condition), or RUNNABLE (CPU-bound)?
> 2. **Lock contention** — Is another thread holding a monitor that main thread is waiting on?
> 3. **Deadlock** — Thread A holds lock X, waits for Y; Thread B holds Y, waits for X.
> For production, I integrate **Firebase Crashlytics** which auto-captures ANR reports with the full stack trace. I also use Firebase Performance to monitor startup time and identify slow screens.

**Q8: What is Perfetto and how does it improve on Systrace?**
> Perfetto is the modern system profiling framework replacing Systrace (both trace the same events, but Perfetto's trace format is richer and its UI more powerful). Perfetto traces can capture: CPU scheduling (which thread runs on which CPU core), Choreographer frame timing, `binder` calls, `atrace` custom events, memory allocations. You analyze traces in `ui.perfetto.dev`. Unlike Systrace (which was a Python script), Perfetto runs as a system daemon and supports streaming, filtering, and much larger traces.

**Q9: Explain how R8 differs from ProGuard and what "full-mode" R8 enables.**
> **ProGuard:** Separate tool. Performs shrinking (removing unused code) and obfuscation (renaming classes/methods). Does not perform significant optimization beyond dead code removal.
> **R8:** Google's replacement for ProGuard, integrated into the Android Gradle Plugin. R8 performs shrinking, obfuscation, **and optimization** in a single pass. "Full mode" R8 (enabled with `android.enableR8.fullMode=true` in `gradle.properties`) additionally performs: class merging, field propagation, method inlining, constant propagation, dead branch elimination, and more aggressive optimizations that ProGuard never did. Full mode requires more complete keep rules but produces significantly smaller, faster code.

---

## Scenario Questions

**Scenario 1:** Your app works fine on high-end devices but users on mid-range devices complain of ANRs when opening the product detail screen. The screen loads product details, reviews, and related items. How would you diagnose and fix?

> **Approach:**
> 1. Enable StrictMode in a debug build on a mid-range device — identify if any main-thread I/O is occurring.
> 2. Use Android Profiler (CPU) with "Record Java/Kotlin method trace" to identify what's taking long during the screen open.
> 3. Likely causes: Room database query on main thread, JSON parsing on main thread, large layout inflation.
> 4. Fix: Move all data loading to `viewModelScope.launch(Dispatchers.IO)`. Observe results as `StateFlow` / `LiveData`.
> 5. Optimize layout: use `ConstraintLayout`, defer loading reviews and related items (load product details first, then paginate reviews).
> 6. Use `ViewStub` for the "Related Items" section — inflate it only when data arrives.

**Scenario 2:** The QA team reports that scrolling in a RecyclerView-based product grid is janky on all devices. How would you profile and fix?

> **Approach:**
> 1. Enable GPU Rendering Profile (Developer Options → Profile GPU rendering → On screen as bars) to visualize per-frame render time.
> 2. Use Android Profiler → CPU → System Trace to see which work exceeds 16ms.
> 3. Common causes of RecyclerView jank:
>    - Heavy `onBindViewHolder()` — parsing dates, formatting strings, creating new objects.
>    - Image loading without placeholders → layout shifts.
>    - `notifyDataSetChanged()` instead of DiffUtil.
>    - Nested RecyclerViews without shared `RecycledViewPool`.
> 4. Fixes: Pre-compute display data in ViewModel, use Glide with placeholder, use `ListAdapter` + `DiffUtil`, set `setHasFixedSize(true)`, share `RecycledViewPool` for similar inner lists.

---

## Revision Notes

- **ANR threshold:** Activity input: 5s, BroadcastReceiver: 10s, Service start: 20s.
- **16ms budget** per frame at 60fps. Modern devices target 120fps (8.3ms).
- **StrictMode** is debug-only. Never enable in release builds.
- **`adb shell am start -W com.example.app/.MainActivity`** prints `TotalTime` — the time from intent dispatch to first frame fully drawn.
- **Choreographer** is the heartbeat of Android's UI rendering — it synchronizes drawing with VSYNC signals.
- **WorkManager** is the recommended solution for guaranteed background work. It respects Doze mode, battery optimization, and works across all API levels 14+.
- **DataStore** replaces SharedPreferences. It's built on Kotlin coroutines and Flow, ensuring all reads/writes are off the main thread.
- **R8** is enabled by default in release builds via `minifyEnabled = true` in `build.gradle.kts`.

---

## Key Takeaways

- 🚫 The Main Thread is sacred — no I/O, no heavy computation, ever.
- ⏱️ ANR = 5s blocked input on Activity, 10s on BroadcastReceiver, 20s on Service start.
- 🔍 StrictMode + Profiler + Perfetto are your performance debugging trilogy.
- 🎨 Overdraw = pixel drawn multiple times per frame → reduce with flat hierarchies, no redundant BGs.
- ♻️ `DiffUtil` + `ListAdapter` is the correct, production-grade way to update RecyclerView.
- 🚀 Cold start < 1s is the target — defer non-critical SDK init; use App Startup library.
- 🔋 Use `WorkManager` for background tasks — it handles Doze, battery, and guaranteed execution.
- 📦 Enable R8 in release — smaller APK, faster load, obfuscated code.

---

---

# Chapter 19: Dependency Injection with Hilt

---

## Concept

**Dependency Injection (DI)** is a design pattern where an object receives its dependencies from an external source rather than creating them internally. Instead of writing `val repo = UserRepository(api, db)` inside a ViewModel, DI provides the `UserRepository` to the ViewModel automatically.

**Hilt** is Jetpack's recommended DI library, built on top of **Dagger 2**. It provides compile-time correctness (DI graph verified at build time), standard Android component integration, and simplified configuration compared to raw Dagger.

**Core terminology:**
- **Dependency** — An object that another object needs to function.
- **Injection** — The act of providing a dependency from outside.
- **Container / Component** — The DI framework's registry that knows how to create all objects.
- **Scope** — Defines the lifetime of a dependency instance (how long the same instance is reused).
- **Binding** — A rule that tells the container how to provide a specific type.

---

## Why It Exists

Consider this without DI:

```kotlin
// Tightly coupled — hard to test, hard to change
class LoginViewModel : ViewModel() {
    private val userRepository = UserRepository(
        RetrofitClient.create(ApiService::class.java),
        AppDatabase.getInstance(context).userDao()
    ) // What if we need to swap to a mock in tests? Impossible!
}
```

DI solves:
1. **Testability** — Inject mock implementations for unit testing.
2. **Reusability** — Same dependency can be shared across multiple consumers.
3. **Separation of Concerns** — Classes don't know how their dependencies are created.
4. **Single Responsibility** — Construction logic lives in one place (the module), not scattered across every class.
5. **Compile-time safety** — Hilt verifies the entire dependency graph at compile time; missing dependencies are build errors, not runtime crashes.

---

## Internal Working

Hilt is built on **Dagger 2** (annotation processing / KSP):

1. You annotate classes with `@Inject`, `@Module`, `@Provides`, etc.
2. **KSP/KAPT** processes these annotations at compile time.
3. Dagger generates **factory classes** and **component implementations** in Java.
4. Hilt generates **standard component classes** tied to Android lifecycles (`SingletonComponent`, `ActivityComponent`, etc.).
5. At runtime, when a component is created (e.g., Activity starts), Hilt instantiates the generated component which knows how to provide all registered types.

### Hilt Component Hierarchy

```
SingletonComponent (Application lifetime)
        │
        ├── ServiceComponent (Service lifetime)
        │
        └── ActivityRetainedComponent (Survives config change)
                   │
                   ├── ViewModelComponent (ViewModel lifetime)
                   │
                   └── ActivityComponent (Activity lifetime)
                              │
                              ├── ViewComponent (View lifetime)
                              │
                              └── FragmentComponent (Fragment lifetime)
                                         │
                                         └── ViewWithFragmentComponent
```

### Binding Resolution Flow

```
@AndroidEntryPoint Activity needs: LoginViewModel
      │
      ▼
Hilt checks: ActivityRetainedComponent can provide LoginViewModel?
      │
      ├── YES: LoginViewModel has @HiltViewModel + @Inject constructor
      │         │
      │         ▼
      │    LoginViewModel needs: UserRepository
      │         │
      │         └── UserRepository has @Inject constructor
      │                   │
      │                   ▼
      │              UserRepository needs: ApiService + UserDao
      │                   │
      │                   ├── ApiService: @Provides in NetworkModule (SingletonComponent)
      │                   └── UserDao: @Provides in DatabaseModule (SingletonComponent)
      │
      ▼
All deps resolved at COMPILE TIME ✓ — runtime injection succeeds
```

---

## Lifecycle / Flow (ASCII Diagrams)

### Hilt Injection Points

```
Application (annotated @HiltAndroidApp)
    │ created
    ▼
SingletonComponent created → @Singleton dependencies instantiated
    │
    ▼
Activity created (@AndroidEntryPoint)
    │
    ├── ActivityRetainedComponent created (survives rotation)
    │       └── ViewModelComponent created
    │               └── @HiltViewModel ViewModel injected ✓
    │
    └── ActivityComponent created
            └── @ActivityScoped deps injected into Activity ✓
    │
    ▼
Fragment created (@AndroidEntryPoint)
    │
    └── FragmentComponent created
            └── @FragmentScoped deps injected into Fragment ✓
    │
    ▼
Fragment destroyed → FragmentComponent destroyed
Activity destroyed → ActivityComponent destroyed
        (ActivityRetainedComponent survives rotation!)
App killed → SingletonComponent destroyed
```

### Scope vs Lifetime

```
┌──────────────────────────────────────────────────────────────┐
│  Scope          │ Component                │ Lifetime         │
├─────────────────┼──────────────────────────┼─────────────────┤
│ @Singleton      │ SingletonComponent       │ App lifetime     │
│ @ActivityScoped │ ActivityComponent        │ Activity lifetime│
│ @FragmentScoped │ FragmentComponent        │ Fragment lifetime│
│ @ViewModelScoped│ ViewModelComponent       │ ViewModel lifetime│
│ @ServiceScoped  │ ServiceComponent         │ Service lifetime │
│ (unscoped)      │ New instance every time  │ No caching       │
└──────────────────────────────────────────────────────────────┘
```

---

## Real World Example

**Scenario:** A social app has a `ProfileFragment` that displays user profile data. It uses a `ProfileViewModel` that depends on `UserRepository`. `UserRepository` depends on `UserRemoteDataSource` (Retrofit API) and `UserLocalDataSource` (Room DAO). All of this is wired together by Hilt with zero boilerplate in the Fragment or ViewModel.

---

## Common Mistakes

### 1. Injecting Activity Context into Singleton

```kotlin
// ❌ WRONG — Activity context in @Singleton → Activity leaks!
@Module
@InstallIn(SingletonComponent::class)
object WrongModule {
    @Provides
    @Singleton
    fun provideAnalytics(activity: Activity): Analytics { // WRONG! Activity != Application
        return Analytics(activity)
    }
}
```

```kotlin
// ✅ CORRECT — Use @ApplicationContext for Singleton deps
@Module
@InstallIn(SingletonComponent::class)
object AnalyticsModule {
    @Provides
    @Singleton
    fun provideAnalytics(@ApplicationContext context: Context): Analytics {
        return Analytics(context) // Application context — safe for singletons
    }
}
```

---

### 2. Using @Provides Instead of @Binds for Interfaces

```kotlin
// ❌ SUBOPTIMAL — @Provides for a binding that @Binds can handle
@Module
@InstallIn(SingletonComponent::class)
object RepositoryModule {
    @Provides
    fun provideUserRepository(impl: UserRepositoryImpl): UserRepository {
        return impl // Hilt still creates impl then wraps — wasteful
    }
}
```

```kotlin
// ✅ CORRECT — @Binds is more efficient (no extra object creation, abstract function)
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    abstract fun bindUserRepository(impl: UserRepositoryImpl): UserRepository
}
```

---

### 3. Using @HiltViewModel Without Proper Scoping

```kotlin
// ❌ WRONG — @Singleton ViewModel outlives all Activities
@Singleton // WRONG — ViewModel should NOT be singleton
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val searchRepository: SearchRepository
) : ViewModel()
// This ViewModel is never cleared! Memory leak!
```

```kotlin
// ✅ CORRECT — @HiltViewModel is @ViewModelScoped by default
@HiltViewModel // No additional scope annotation needed
class SearchViewModel @Inject constructor(
    private val searchRepository: SearchRepository,
    private val savedStateHandle: SavedStateHandle // Auto-injected by Hilt
) : ViewModel() {
    // ViewModelComponent manages this — cleared when ViewModel.onCleared() is called
}
```

---

## Code Examples (Production-Quality Kotlin)

### Complete Hilt Setup

```kotlin
// Step 1: Application — annotate with @HiltAndroidApp
@HiltAndroidApp
class MyApplication : Application() {
    override fun onCreate() {
        super.onCreate()
        // Hilt generates a Hilt_MyApplication base class
        // SingletonComponent is created here
    }
}
```

```kotlin
// Step 2: Network Module
@Module
@InstallIn(SingletonComponent::class) // Lives as long as the app
object NetworkModule {

    @Provides
    @Singleton
    fun provideOkHttpClient(): OkHttpClient {
        return OkHttpClient.Builder()
            .addInterceptor(HttpLoggingInterceptor().apply {
                level = if (BuildConfig.DEBUG)
                    HttpLoggingInterceptor.Level.BODY
                else
                    HttpLoggingInterceptor.Level.NONE
            })
            .connectTimeout(30, TimeUnit.SECONDS)
            .readTimeout(30, TimeUnit.SECONDS)
            .build()
    }

    @Provides
    @Singleton
    fun provideRetrofit(okHttpClient: OkHttpClient): Retrofit {
        return Retrofit.Builder()
            .baseUrl(BuildConfig.API_BASE_URL)
            .client(okHttpClient)
            .addConverterFactory(MoshiConverterFactory.create())
            .build()
    }

    @Provides
    @Singleton
    fun provideApiService(retrofit: Retrofit): ApiService {
        return retrofit.create(ApiService::class.java)
    }
}
```

```kotlin
// Step 3: Database Module
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database"
        )
            .fallbackToDestructiveMigration()
            .build()
    }

    @Provides
    fun provideUserDao(database: AppDatabase): UserDao {
        return database.userDao() // Unscoped — same DB, different DAO instance is fine
    }
}
```

```kotlin
// Step 4: Repository Module with @Binds
@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {

    @Binds
    @Singleton
    abstract fun bindUserRepository(
        impl: UserRepositoryImpl // Hilt knows how to create this via @Inject constructor
    ): UserRepository // Binds the interface to the implementation
}
```

```kotlin
// Step 5: Repository Implementation
class UserRepositoryImpl @Inject constructor(
    private val apiService: ApiService,     // Provided by NetworkModule
    private val userDao: UserDao,            // Provided by DatabaseModule
    @IoDispatcher private val ioDispatcher: Dispatcher // Custom qualifier (see below)
) : UserRepository {

    override fun getUserProfile(userId: String): Flow<User> = flow {
        // Emit cached data first
        userDao.getUserFlow(userId).collect { cached ->
            if (cached != null) emit(cached)
        }
    }.combine(
        flow {
            val remote = apiService.getUser(userId)
            userDao.insertUser(remote.toEntity())
            emit(remote.toDomain())
        }
    ) { _, fresh -> fresh }
        .flowOn(ioDispatcher)
}
```

```kotlin
// Step 6: Custom Qualifier for Dispatchers
@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class IoDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class MainDispatcher

@Qualifier
@Retention(AnnotationRetention.BINARY)
annotation class DefaultDispatcher

@Module
@InstallIn(SingletonComponent::class)
object DispatcherModule {
    @Provides
    @IoDispatcher
    fun provideIoDispatcher(): CoroutineDispatcher = Dispatchers.IO

    @Provides
    @MainDispatcher
    fun provideMainDispatcher(): CoroutineDispatcher = Dispatchers.Main

    @Provides
    @DefaultDispatcher
    fun provideDefaultDispatcher(): CoroutineDispatcher = Dispatchers.Default
}
```

```kotlin
// Step 7: ViewModel with @HiltViewModel
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val savedStateHandle: SavedStateHandle // Hilt injects this automatically
) : ViewModel() {

    private val userId: String = savedStateHandle.get<String>("userId")
        ?: throw IllegalArgumentException("userId required in SavedStateHandle")

    private val _uiState = MutableStateFlow<ProfileUiState>(ProfileUiState.Loading)
    val uiState: StateFlow<ProfileUiState> = _uiState.asStateFlow()

    init {
        loadProfile()
    }

    private fun loadProfile() {
        viewModelScope.launch {
            userRepository.getUserProfile(userId)
                .catch { e -> _uiState.value = ProfileUiState.Error(e.message ?: "Unknown error") }
                .collect { user -> _uiState.value = ProfileUiState.Success(user) }
        }
    }
}

sealed class ProfileUiState {
    object Loading : ProfileUiState()
    data class Success(val user: User) : ProfileUiState()
    data class Error(val message: String) : ProfileUiState()
}
```

```kotlin
// Step 8: Fragment — @AndroidEntryPoint
@AndroidEntryPoint
class ProfileFragment : Fragment() {
    private val viewModel: ProfileViewModel by viewModels()
    private var _binding: FragmentProfileBinding? = null
    private val binding get() = _binding!!

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.lifecycle.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is ProfileUiState.Loading -> showLoading()
                        is ProfileUiState.Success -> showProfile(state.user)
                        is ProfileUiState.Error -> showError(state.message)
                    }
                }
            }
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

### Hilt Testing — Replacing Modules

```kotlin
// Test module that replaces NetworkModule in tests
@TestInstallIn(
    components = [SingletonComponent::class],
    replaces = [NetworkModule::class] // Replaces the real module
)
@Module
object FakeNetworkModule {
    @Provides
    @Singleton
    fun provideApiService(): ApiService = FakeApiService() // In-memory fake
}

// Test class using HiltRule
@HiltAndroidTest
class ProfileViewModelTest {

    @get:Rule
    val hiltRule = HiltAndroidRule(this)

    @Inject
    lateinit var userRepository: UserRepository

    @Before
    fun setUp() {
        hiltRule.inject()
    }

    @Test
    fun testLoadProfile() = runTest {
        // Inject the real ViewModel with fake dependencies
        val viewModel = ProfileViewModel(userRepository, SavedStateHandle(mapOf("userId" to "123")))
        viewModel.uiState.test {
            assertEquals(ProfileUiState.Loading, awaitItem())
            val success = awaitItem() as ProfileUiState.Success
            assertEquals("John Doe", success.user.name)
        }
    }
}
```

### Lazy Injection

```kotlin
// Defer injection until first use — useful for expensive dependencies
@AndroidEntryPoint
class MainActivity : AppCompatActivity() {

    @Inject
    lateinit var analyticsService: AnalyticsService // Injected at Activity creation

    @Inject
    lateinit var heavyReportingService: Lazy<HeavyReportingService> // NOT injected yet

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        analyticsService.trackScreenView("Main")

        // heavyReportingService.get() only called when actually needed
    }

    fun onReportButtonClick() {
        heavyReportingService.get().generateReport() // Created on first access
    }
}
```

### @EntryPoint for Non-Hilt Classes

```kotlin
// ContentProvider cannot use @AndroidEntryPoint — use @EntryPoint instead
@EntryPoint
@InstallIn(SingletonComponent::class)
interface ContentProviderEntryPoint {
    fun userRepository(): UserRepository
}

class MyContentProvider : ContentProvider() {
    private lateinit var userRepository: UserRepository

    override fun onCreate(): Boolean {
        val entryPoint = EntryPointAccessors.fromApplication(
            context!!.applicationContext,
            ContentProviderEntryPoint::class.java
        )
        userRepository = entryPoint.userRepository()
        return true
    }
}
```

---

## Best Practices

1. **`@Singleton` only for truly app-wide resources** — Database, Retrofit, OkHttpClient. Not for per-screen state.
2. **Use `@Binds` over `@Provides` for interface bindings** — More efficient (no concrete object creation).
3. **Inject `@ApplicationContext`** into `@Singleton` dependencies, never `Activity` context.
4. **Use custom `@Qualifier` for same-type disambiguation** — Especially for `CoroutineDispatcher`, `OkHttpClient`, `String` base URLs.
5. **`@HiltViewModel` + `SavedStateHandle`** — Always use for state-safe ViewModels.
6. **Replace modules in tests with `@TestInstallIn`** — Never modify production modules for testing.
7. **Keep `@Module` classes focused** — One module per concern (NetworkModule, DatabaseModule, RepositoryModule).
8. **Scope dependencies at the correct level** — Over-scoping (making everything `@Singleton`) causes memory waste; under-scoping (no scope) creates unnecessary instances.
9. **Prefer constructor injection over field injection** — More testable, no lateinit var, works with non-Android classes.
10. **Use `Lazy<T>` for expensive dependencies** used infrequently to defer initialization cost.

---

## Legacy vs Modern

| Aspect | Legacy | Modern |
|--------|--------|--------|
| Manual DI | Factory pattern, `ServiceLocator` | Hilt (compile-time safe, auto-wired) |
| DI Framework | Dagger 2 (complex setup) | Hilt (standard Android components) |
| ViewModel creation | `ViewModelProvider.Factory` (boilerplate) | `@HiltViewModel` + `by viewModels()` |
| Testing | Manual factory swapping | `@TestInstallIn` module replacement |
| Koin (alternative) | Runtime DI, no compile-time check | Hilt (compile-time, Jetpack-recommended) |
| Entry points | `Dagger.builder()` patterns | `@AndroidEntryPoint`, `@EntryPoint` |

---

## Interview Questions

### Beginner

**Q1: What is dependency injection and why do we use it?**
> DI is a design pattern where a class receives its dependencies from an external source rather than instantiating them itself. We use it because: (1) it improves testability — we can inject mock dependencies in tests; (2) it reduces tight coupling — classes don't know how their dependencies are built; (3) it centralizes object construction — all wiring is in modules, not scattered across the codebase.

**Q2: What is the difference between `@Inject` constructor and `@Provides`?**
> `@Inject constructor` tells Hilt that it can create this class by using its constructor. All constructor parameters must also be injectable. This works for classes *you own* (your own classes).
> `@Provides` is used in a `@Module` to tell Hilt how to create a dependency you *don't own* — e.g., a Retrofit instance, a third-party SDK class, or a class that requires complex construction logic that can't be expressed with just an `@Inject constructor`.

**Q3: What does `@HiltAndroidApp` do?**
> `@HiltAndroidApp` is a required annotation on your `Application` class. It triggers Hilt's code generation, creating the `SingletonComponent` which is the root of the dependency graph. Hilt generates a `Hilt_MyApplication` base class that sets up the component and handles injection. Without this annotation, no Hilt injection works.

---

### Intermediate

**Q4: What is the difference between `@Singleton`, `@ActivityScoped`, and unscoped bindings?**
> - **`@Singleton`**: Hilt creates exactly **one instance** for the app's entire lifetime (SingletonComponent). The same instance is returned every time it's requested.
> - **`@ActivityScoped`**: Hilt creates **one instance per Activity lifetime**. Every injection into the same Activity instance gets the same object. When the Activity is destroyed, the instance is discarded.
> - **Unscoped (no annotation)**: Hilt creates a **new instance every time** the dependency is requested. No caching.

**Q5: What is `@Binds` and when should you use it instead of `@Provides`?**
> `@Binds` is used to bind an interface to its implementation when the implementation is injectable via `@Inject constructor`. It must be in an `abstract` function within an `abstract` class (or interface) `@Module`. Hilt/Dagger uses it directly without instantiating a wrapper — it's more efficient than `@Provides` (which would create a concrete function that gets called at runtime). Use `@Binds` whenever you have `interface → implementation` wiring.

**Q6: Explain how `SavedStateHandle` is automatically injected into a `@HiltViewModel`.**
> When Hilt creates a `@HiltViewModel`, it uses a custom `AbstractSavedStateViewModelFactory` under the hood. This factory receives the `SavedStateRegistryOwner` (the Activity or Fragment) and creates a `SavedStateHandle` instance backed by that owner's saved state registry. Hilt's generated code passes this `SavedStateHandle` as a constructor argument when instantiating the ViewModel. The developer doesn't need to write any factory boilerplate — just declare `SavedStateHandle` as a constructor parameter.

---

### Advanced

**Q7: How does Hilt's compile-time verification work? What happens if a dependency is missing?**
> Hilt processes annotations using KAPT (or KSP). During compilation, Dagger traverses the entire dependency graph starting from each `@InstallIn` component. For every requested type, it checks if there's a binding (`@Provides`, `@Binds`, or `@Inject constructor`) in the same or a parent component. If any type in the graph is missing a binding, KAPT throws a **compilation error** with a clear message: *"[Dagger/MissingBinding] Type X cannot be provided without an @Inject constructor or an @Provides-annotated method."* This means you can never have a runtime DI failure from a missing dependency — it's always caught at build time.

**Q8: How would you provide two different `OkHttpClient` instances (one with auth, one without) using Hilt?**
> Use `@Qualifier` annotations to distinguish the two bindings:
> ```kotlin
> @Qualifier @Retention(AnnotationRetention.BINARY)
> annotation class AuthenticatedClient
>
> @Qualifier @Retention(AnnotationRetention.BINARY)
> annotation class UnauthenticatedClient
>
> @Module @InstallIn(SingletonComponent::class)
> object NetworkModule {
>     @Provides @Singleton @AuthenticatedClient
>     fun provideAuthClient(authInterceptor: AuthInterceptor): OkHttpClient =
>         OkHttpClient.Builder().addInterceptor(authInterceptor).build()
>
>     @Provides @Singleton @UnauthenticatedClient
>     fun providePublicClient(): OkHttpClient =
>         OkHttpClient.Builder().build()
> }
>
> // Usage:
> class ApiService @Inject constructor(
>     @AuthenticatedClient private val authClient: OkHttpClient,
>     @UnauthenticatedClient private val publicClient: OkHttpClient
> )
> ```

**Q9: What is the difference between Hilt and Koin? Which would you choose and why?**
> | Aspect | Hilt | Koin |
> |--------|------|------|
> | Type | Compile-time DI (Dagger-based) | Runtime Service Locator |
> | Safety | Compile-time verification | Runtime failures possible |
> | Setup | More verbose (annotations, modules) | Simple DSL (`single {}`, `viewModel {}`) |
> | Performance | No reflection overhead | Uses reflection at startup |
> | Android integration | Official Jetpack library | Third-party, but well-maintained |
> | Testing | `@TestInstallIn` | Override modules in test `startKoin` |
> | Build time | Slower (KAPT) | Faster |
>
> **Choice:** For production apps, I choose **Hilt** — compile-time safety is non-negotiable at scale. Missing bindings are build errors, not user-facing crashes. For very small apps or rapid prototypes, Koin's simplicity is appealing.

---

## Scenario Questions

**Scenario 1:** You have a `UserRepository` that needs different implementations in debug and release builds — `FakeUserRepository` for debug (returns hardcoded data) and `RealUserRepository` for release. How do you achieve this with Hilt?

> **Approach:** Use separate source sets and `@Binds`:
> ```
> src/debug/java/.../DebugRepositoryModule.kt  → @Binds FakeUserRepository as UserRepository
> src/release/java/.../ReleaseRepositoryModule.kt → @Binds RealUserRepository as UserRepository
> ```
> Both modules have the same `@Module` name and `@InstallIn(SingletonComponent::class)`. Gradle includes the correct one based on the build variant. The ViewModel and Fragment code never change — they always inject `UserRepository`.

**Scenario 2:** You have a class that is neither an Activity, Fragment, ViewModel, nor Service, but it needs a Hilt-injected dependency. How do you inject into it?

> Use `@EntryPoint`:
> ```kotlin
> @EntryPoint
> @InstallIn(SingletonComponent::class)
> interface MyHelperEntryPoint {
>     fun analyticsService(): AnalyticsService
> }
>
> class MyHelper(private val context: Context) {
>     private val analyticsService: AnalyticsService by lazy {
>         EntryPointAccessors.fromApplication(
>             context.applicationContext,
>             MyHelperEntryPoint::class.java
>         ).analyticsService()
>     }
> }
> ```
> This is the correct pattern for `ContentProvider`, legacy classes, or any class not supported by `@AndroidEntryPoint`.

---

## Revision Notes

- **Hilt = Dagger 2 + Android lifecycle awareness.** All of Dagger 2's power, simplified.
- **Component hierarchy:** `SingletonComponent` → `ActivityRetainedComponent` → `ActivityComponent` → `FragmentComponent`.
- **`@AndroidEntryPoint`** triggers field injection in Activity/Fragment. Generates `Hilt_XxxActivity` base class.
- **`@HiltAndroidApp`** is required on the `Application` class — it's the root of the DI graph.
- **`@Inject` constructor** is preferred over field injection for non-Android classes (testable without DI framework).
- **`@Singleton`** should be used sparingly — only for truly shared, expensive-to-create objects.
- **`@Binds`** vs **`@Provides`:** Use `@Binds` for interface→impl binding; `@Provides` for third-party or complex construction.
- **Testing:** `@TestInstallIn` replaces production modules in tests. `HiltAndroidRule` sets up Hilt in test.

---

## Key Takeaways

- 💉 DI = objects receive their dependencies from outside, not create them internally.
- 🏗️ Hilt is built on Dagger 2 — all DI graph validation happens at **compile time**.
- 🌲 Component hierarchy mirrors Android lifecycle: `Singleton → Activity → Fragment`.
- 🔒 `@Singleton` = one instance for app lifetime. `@ActivityScoped` = one per Activity. Unscoped = new each time.
- 🎯 `@Binds` is more efficient than `@Provides` for interface→implementation binding.
- 🧪 Replace real modules with `@TestInstallIn` — never modify production code for tests.
- 💾 `SavedStateHandle` is auto-injected into `@HiltViewModel` — no factory boilerplate.
- 🚀 Hilt eliminates thousands of lines of DI boilerplate compared to raw Dagger 2.

---

---

# Chapter 20: Jetpack Compose (Modern UI) vs XML Views

---

## Concept

**Jetpack Compose** is Android's modern, fully declarative UI toolkit written in Kotlin. Released stable in 2021, it replaces the traditional XML-based view system with composable functions that describe the UI as a function of state.

**XML Views** (the traditional system) uses an **imperative** paradigm: you define a static layout in XML, inflate it to a `View` tree at runtime, then manually update individual views when data changes.

**Declarative vs Imperative:**
- **Declarative (Compose):** You describe *what* the UI should look like for a given state. When state changes, Compose automatically re-renders only what changed.
- **Imperative (XML):** You describe *how* to update the UI. When data changes, you imperatively call `textView.text = "new value"`, `imageView.visibility = View.GONE`, etc.

```
Imperative (XML):            Declarative (Compose):
State changes               State changes
    │                           │
    ▼                           ▼
Developer writes             Compose automatically
update code:                 re-executes composables
textView.text = x            that depend on state
button.isEnabled = y         ↓
imageView.visibility = z     Only changed UI updated
    (manual, error-prone)    (automatic, correct)
```

---

## Why It Exists

The XML View system was introduced in Android 1.0 (2008). Over 15 years, it accumulated:
1. **Massive boilerplate** — `findViewById`, ViewBinding, data binding, custom adapters.
2. **State management complexity** — Manual syncing of data to views causes bugs (stale UI, null pointer on views that aren't inflated yet).
3. **Performance costs** — XML inflation (parsing XML, creating View objects, measuring, laying out) is slow and happens on the main thread.
4. **Limited Kotlin integration** — The View system was designed for Java; Kotlin features (lambdas, extension functions, coroutines) are bolted on.
5. **Testing difficulty** — Instrumented tests required; hard to test UI logic in isolation.

Compose was designed from scratch to solve all these problems while being idiomatic Kotlin.

---

## Internal Working

### How Compose Renders UI

Compose has three phases per frame:

```
1. COMPOSITION Phase:
   @Composable functions execute
   → Build in-memory UI tree (Slot Table / Gap Buffer)
   → Compose remembers which composable produced which UI node

2. LAYOUT Phase:
   Each UI node measures itself (constraints from parent)
   → Each node determines its position
   → Single pass (vs multiple passes in XML's measure/layout)

3. DRAWING Phase:
   Each node is drawn onto the Canvas
   → Hardware-accelerated via DisplayList (same as XML)
```

### Recomposition

Compose's **smart recomposition** is central to its performance model:

```
State changes (e.g., MutableState<String> updated)
        │
        ▼
Compose INVALIDATES only composables that READ that state
        │
        ▼
Only those composables RECOMPOSE (re-execute)
        │
        ├── If output is SAME as before → skipped (stable optimization)
        └── If output CHANGED → layout + draw updated
```

### Slot Table (Internal Data Structure)

Compose uses a **Slot Table** (a gap buffer) to store the composition's state. Each composable's position in the call tree is tracked by the slot table. `remember {}` stores values in this table keyed by position. This is why `remember {}` state survives recompositions but not Activity restarts (for that, use `rememberSaveable`).

---

## Lifecycle / Flow (ASCII Diagrams)

### Composable Lifecycle

```
Initial Composition:
onCompose() → Enter Composition → Remember state initialized
                                                 │
                                                 ▼
                                    State changes or parent recomposes
                                                 │
                                                 ▼
Recomposition: ← ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ Re-execute composable ─ ─ ─ ─┐
                                       (skipped if stable & unchanged)│
                                                 │                     │
                                                 ▼                     │
                              Composable removed from tree              │
                                                 │                     │
                                        onDispose() called             │
                                        (DisposableEffect cleanup)     │
                                                 │                     │
                                        Leave Composition ─ ─ ─ ─ ─ ─ ┘
```

### Side Effects Flow

```
@Composable fun MyScreen(userId: String) {

    LaunchedEffect(userId) {          ← Runs coroutine when userId changes
        loadUser(userId)               (also on first composition)
    }                                  Cancels previous coroutine if userId changes

    DisposableEffect(sensor) {         ← Setup on enter, cleanup on leave
        sensor.register()
        onDispose {
            sensor.unregister()        ← Called when leaving composition
        }
    }

    SideEffect {                       ← Runs after every successful recompose
        analytics.track(currentScreen) (not a coroutine; synchronous)
    }
}
```

### Compose vs XML State Flow

```
XML Views:
ViewModel (LiveData/StateFlow)
    │
    ▼
Observer in Fragment
    │
    ▼
Manually update: textView.text = it.name
                 imageView.isVisible = it.isLoggedIn
                 (every field, manually, error-prone)

Compose:
ViewModel (StateFlow<UiState>)
    │
    ▼
val state by viewModel.uiState.collectAsState()
    │
    ▼
UI auto-rendered:
MyScreen(state)  ← re-executes automatically when state changes
(only changed composables recompose — correct & efficient)
```

---

## Real World Example

**Scenario:** A weather app displays current temperature, weather icon, and a 7-day forecast list. In XML, you'd inflate the layout, find views, set text/image, create a RecyclerView adapter. In Compose, you write a single `WeatherScreen` composable that reads a `WeatherUiState` and renders itself. When temperature updates, only the temperature `Text` recomposes — not the entire screen.

---

## Common Mistakes

### 1. Performing Side Effects Directly in Composable Body

```kotlin
// ❌ WRONG — Side effects in composable body run on EVERY recomposition
@Composable
fun UserScreen(userId: String) {
    analytics.logScreenView("UserScreen") // Called on EVERY recompose! Wrong!
    val viewModel: UserViewModel = hiltViewModel()
    // ...
}
```

```kotlin
// ✅ CORRECT — Use SideEffect for synchronous, LaunchedEffect for async
@Composable
fun UserScreen(userId: String) {
    val viewModel: UserViewModel = hiltViewModel()

    // Runs ONCE per composition entry (or when userId changes)
    LaunchedEffect(userId) {
        viewModel.loadUser(userId)
    }

    // Runs after every SUCCESSFUL recomposition (not cancelled)
    SideEffect {
        analytics.logScreenView("UserScreen")
    }
}
```

---

### 2. Using `remember` Without `rememberSaveable` for State That Survives Rotation

```kotlin
// ❌ WRONG — remember{} is reset on Activity recreation (rotation)
@Composable
fun SearchScreen() {
    var searchQuery by remember { mutableStateOf("") } // Lost on rotation!
    // ...
}
```

```kotlin
// ✅ CORRECT — rememberSaveable persists across config changes
@Composable
fun SearchScreen() {
    var searchQuery by rememberSaveable { mutableStateOf("") } // Survives rotation ✓
    // For complex types, provide a custom Saver:
    // var complex by rememberSaveable(stateSaver = ComplexSaver) { mutableStateOf(Complex()) }
}
```

---

### 3. Creating State in Composable Instead of ViewModel (State Hoisting Violation)

```kotlin
// ❌ WRONG — State buried inside composable; not testable, not reusable
@Composable
fun LoginForm() {
    var email by remember { mutableStateOf("") }
    var password by remember { mutableStateOf("") }
    var isLoading by remember { mutableStateOf(false) }

    // How do you test this? How do you share this state? You can't!
    Button(onClick = {
        isLoading = true
        // Direct API call from composable — WRONG!
        loginApi.login(email, password)
    }) { Text("Login") }
}
```

```kotlin
// ✅ CORRECT — State hoisted to ViewModel; composable is stateless
@Composable
fun LoginScreen(viewModel: LoginViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsState()

    LoginForm(
        email = uiState.email,
        password = uiState.password,
        isLoading = uiState.isLoading,
        onEmailChange = viewModel::onEmailChange,
        onPasswordChange = viewModel::onPasswordChange,
        onLoginClick = viewModel::onLoginClick
    )
}

// Stateless composable — easy to test and preview
@Composable
fun LoginForm(
    email: String,
    password: String,
    isLoading: Boolean,
    onEmailChange: (String) -> Unit,
    onPasswordChange: (String) -> Unit,
    onLoginClick: () -> Unit
) {
    // UI only — no state, no business logic
}
```

---

## Code Examples (Production-Quality Kotlin)

### Complete Compose Screen with ViewModel

```kotlin
// Data / State
data class ProductListUiState(
    val products: List<Product> = emptyList(),
    val isLoading: Boolean = false,
    val errorMessage: String? = null,
    val searchQuery: String = ""
)

// ViewModel
@HiltViewModel
class ProductListViewModel @Inject constructor(
    private val productRepository: ProductRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow(ProductListUiState(isLoading = true))
    val uiState: StateFlow<ProductListUiState> = _uiState.asStateFlow()

    init {
        loadProducts()
    }

    private fun loadProducts() {
        viewModelScope.launch {
            productRepository.getProducts()
                .catch { e ->
                    _uiState.update { it.copy(isLoading = false, errorMessage = e.message) }
                }
                .collect { products ->
                    _uiState.update { it.copy(products = products, isLoading = false) }
                }
        }
    }

    fun onSearchQueryChange(query: String) {
        _uiState.update { it.copy(searchQuery = query) }
    }

    fun onRetry() {
        _uiState.update { it.copy(isLoading = true, errorMessage = null) }
        loadProducts()
    }
}

// Composable Screen
@Composable
fun ProductListScreen(
    onProductClick: (Product) -> Unit,
    viewModel: ProductListViewModel = hiltViewModel()
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    Scaffold(
        topBar = {
            TopAppBar(
                title = { Text("Products") },
                colors = TopAppBarDefaults.topAppBarColors(
                    containerColor = MaterialTheme.colorScheme.primaryContainer
                )
            )
        }
    ) { paddingValues ->
        ProductListContent(
            uiState = uiState,
            onProductClick = onProductClick,
            onSearchQueryChange = viewModel::onSearchQueryChange,
            onRetry = viewModel::onRetry,
            modifier = Modifier.padding(paddingValues)
        )
    }
}

// Stateless content composable — fully testable via Preview
@Composable
fun ProductListContent(
    uiState: ProductListUiState,
    onProductClick: (Product) -> Unit,
    onSearchQueryChange: (String) -> Unit,
    onRetry: () -> Unit,
    modifier: Modifier = Modifier
) {
    Column(modifier = modifier.fillMaxSize()) {
        // Search Bar
        SearchBar(
            query = uiState.searchQuery,
            onQueryChange = onSearchQueryChange,
            modifier = Modifier
                .fillMaxWidth()
                .padding(horizontal = 16.dp, vertical = 8.dp)
        )

        when {
            uiState.isLoading -> LoadingIndicator(modifier = Modifier.fillMaxSize())

            uiState.errorMessage != null -> ErrorState(
                message = uiState.errorMessage,
                onRetry = onRetry,
                modifier = Modifier.fillMaxSize()
            )

            uiState.products.isEmpty() -> EmptyState(modifier = Modifier.fillMaxSize())

            else -> ProductList(
                products = uiState.products,
                onProductClick = onProductClick
            )
        }
    }
}

@Composable
fun ProductList(
    products: List<Product>,
    onProductClick: (Product) -> Unit,
    modifier: Modifier = Modifier
) {
    LazyColumn(
        modifier = modifier,
        contentPadding = PaddingValues(horizontal = 16.dp, vertical = 8.dp),
        verticalArrangement = Arrangement.spacedBy(8.dp)
    ) {
        items(
            items = products,
            key = { it.id } // Stable key for optimized recomposition
        ) { product ->
            ProductCard(
                product = product,
                onClick = { onProductClick(product) }
            )
        }
    }
}

@Composable
fun ProductCard(
    product: Product,
    onClick: () -> Unit,
    modifier: Modifier = Modifier
) {
    Card(
        onClick = onClick,
        modifier = modifier.fillMaxWidth(),
        elevation = CardDefaults.cardElevation(defaultElevation = 2.dp)
    ) {
        Row(
            modifier = Modifier
                .padding(12.dp)
                .fillMaxWidth(),
            horizontalArrangement = Arrangement.SpaceBetween,
            verticalAlignment = Alignment.CenterVertically
        ) {
            Column(modifier = Modifier.weight(1f)) {
                Text(
                    text = product.name,
                    style = MaterialTheme.typography.titleMedium,
                    maxLines = 1,
                    overflow = TextOverflow.Ellipsis
                )
                Text(
                    text = product.description,
                    style = MaterialTheme.typography.bodySmall,
                    color = MaterialTheme.colorScheme.onSurfaceVariant,
                    maxLines = 2,
                    overflow = TextOverflow.Ellipsis
                )
            }
            Text(
                text = "$${product.price}",
                style = MaterialTheme.typography.titleMedium,
                color = MaterialTheme.colorScheme.primary,
                modifier = Modifier.padding(start = 8.dp)
            )
        }
    }
}

@Preview(showBackground = true)
@Composable
fun ProductCardPreview() {
    MaterialTheme {
        ProductCard(
            product = Product(id = "1", name = "Wireless Headphones",
                description = "Premium noise-cancelling audio", price = 99.99),
            onClick = {}
        )
    }
}
```

### Compose Navigation

```kotlin
// Navigation Graph
@Composable
fun AppNavigation(navController: NavHostController = rememberNavController()) {
    NavHost(
        navController = navController,
        startDestination = "product_list"
    ) {
        composable("product_list") {
            ProductListScreen(
                onProductClick = { product ->
                    navController.navigate("product_detail/${product.id}")
                }
            )
        }

        composable(
            route = "product_detail/{productId}",
            arguments = listOf(navArgument("productId") { type = NavType.StringType })
        ) { backStackEntry ->
            val productId = backStackEntry.arguments?.getString("productId") ?: return@composable
            ProductDetailScreen(
                productId = productId,
                onBack = { navController.popBackStack() }
            )
        }

        composable("settings") {
            SettingsScreen()
        }
    }
}
```

### Animation in Compose

```kotlin
// AnimatedVisibility — show/hide with animation
@Composable
fun FabSection(isExpanded: Boolean, onFabClick: () -> Unit) {
    Column(horizontalAlignment = Alignment.End) {
        // Sub-actions only visible when expanded
        AnimatedVisibility(
            visible = isExpanded,
            enter = fadeIn() + slideInVertically(),
            exit = fadeOut() + slideOutVertically()
        ) {
            Column {
                ExtendedFloatingActionButton(
                    text = { Text("Share") },
                    icon = { Icon(Icons.Default.Share, null) },
                    onClick = {}
                )
                Spacer(modifier = Modifier.height(8.dp))
                ExtendedFloatingActionButton(
                    text = { Text("Add to Cart") },
                    icon = { Icon(Icons.Default.ShoppingCart, null) },
                    onClick = {}
                )
                Spacer(modifier = Modifier.height(8.dp))
            }
        }

        // Main FAB
        FloatingActionButton(onClick = onFabClick) {
            val rotation by animateFloatAsState(
                targetValue = if (isExpanded) 45f else 0f,
                animationSpec = tween(durationMillis = 300),
                label = "fab_rotation"
            )
            Icon(
                imageVector = Icons.Default.Add,
                contentDescription = "Expand",
                modifier = Modifier.rotate(rotation)
            )
        }
    }
}
```

### Compose + XML Interoperability

```kotlin
// 1. Use Compose inside an XML layout (ComposeView)
// res/layout/activity_main.xml:
// <androidx.compose.ui.platform.ComposeView
//     android:id="@+id/compose_view"
//     android:layout_width="match_parent"
//     android:layout_height="match_parent" />

class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val composeView = findViewById<ComposeView>(R.id.compose_view)
        composeView.apply {
            setViewCompositionStrategy(ViewCompositionStrategy.DisposeOnViewTreeLifecycleDestroyed)
            setContent {
                MaterialTheme {
                    ProductListScreen(onProductClick = { /* navigate */ })
                }
            }
        }
    }
}

// 2. Use Android View inside Compose (AndroidView)
@Composable
fun MapView(modifier: Modifier = Modifier) {
    val context = LocalContext.current

    AndroidView(
        factory = { ctx ->
            MapView(ctx).apply {
                // Initialize the MapView (traditional View)
                onCreate(null)
                getMapAsync { googleMap ->
                    googleMap.moveCamera(CameraUpdateFactory.newLatLng(LatLng(0.0, 0.0)))
                }
            }
        },
        update = { mapView ->
            // Called on recompositions — update the view here
            mapView.onResume()
        },
        modifier = modifier
    )
}
```

### Custom Theming with Material 3

```kotlin
// Theme Definition
private val LightColorScheme = lightColorScheme(
    primary = Color(0xFF1565C0),
    onPrimary = Color.White,
    primaryContainer = Color(0xFFD1E4FF),
    secondary = Color(0xFF415F91),
    background = Color(0xFFF8F9FF),
    surface = Color(0xFFF8F9FF),
    error = Color(0xFFBA1A1A)
)

private val DarkColorScheme = darkColorScheme(
    primary = Color(0xFFA0C4FF),
    onPrimary = Color(0xFF00315E),
    primaryContainer = Color(0xFF004885),
    secondary = Color(0xFFAAC7FF),
    background = Color(0xFF111318),
    surface = Color(0xFF111318),
    error = Color(0xFFFFB4AB)
)

@Composable
fun AppTheme(
    darkTheme: Boolean = isSystemInDarkTheme(),
    dynamicColor: Boolean = true, // Material You (API 31+)
    content: @Composable () -> Unit
) {
    val colorScheme = when {
        dynamicColor && Build.VERSION.SDK_INT >= Build.VERSION_CODES.S -> {
            val context = LocalContext.current
            if (darkTheme) dynamicDarkColorScheme(context)
            else dynamicLightColorScheme(context)
        }
        darkTheme -> DarkColorScheme
        else -> LightColorScheme
    }

    MaterialTheme(
        colorScheme = colorScheme,
        typography = AppTypography, // Your custom typography
        content = content
    )
}
```

---

## Best Practices

### Compose Best Practices

1. **Hoist state to the ViewModel** — Composables should be as stateless as possible.
2. **Use `collectAsStateWithLifecycle()`** instead of `collectAsState()` — lifecycle-aware, stops collection when UI is not visible (reduces unnecessary recompositions).
3. **Always provide a `modifier` parameter** in reusable composables — allows callers to customize layout.
4. **Use `key()` in `LazyColumn` items** — provides stable identity for animated insertions/deletions.
5. **Avoid expensive operations in composable body** — use `remember {}` to cache computations.
6. **Use `@Stable` and `@Immutable`** annotations on data classes to help Compose's skip optimization.
7. **Prefer `LazyColumn`/`LazyRow`** over `Column`/`Row` in `verticalScroll()` for lists.
8. **Use `derivedStateOf`** when a state is computed from other states to avoid unnecessary recompositions.

### XML Views Best Practices

1. **ViewBinding** over `findViewById` — type-safe, null-safe.
2. **ConstraintLayout** over nested LinearLayouts.
3. **RecyclerView + DiffUtil** for any list.
4. **ViewStub** for conditionally visible heavy layouts.
5. **Data Binding** for two-way binding (though Compose eliminates this need).

---

## Legacy vs Modern

| Aspect | XML Views (Legacy) | Jetpack Compose (Modern) |
|--------|---------------------|--------------------------|
| Language | XML + Java/Kotlin | Kotlin only |
| Paradigm | Imperative | Declarative |
| UI updates | Manual (`view.text = x`) | Automatic (recomposition) |
| Lists | RecyclerView + Adapter + DiffUtil | `LazyColumn` / `LazyRow` |
| Animation | Animator/Transition frameworks | Built-in `animate*AsState`, `AnimatedVisibility` |
| Theming | `styles.xml` + `themes.xml` | `MaterialTheme` composable |
| Testing | Espresso (instrumented) | Compose UI testing + Preview |
| Boilerplate | High (LayoutInflater, ViewHolder, Adapter, DiffUtil) | Low |
| Custom views | Extend `View`, override `onDraw` | Custom `Modifier`, `Canvas` API |
| Navigation | Fragment backstack, Navigation Component | Navigation Compose |
| Learning curve | Low (existing Android devs) | Medium (new paradigm) |
| Interop | Host Compose in XML (`ComposeView`) | Host XML View in Compose (`AndroidView`) |
| Lifecycle | Fragment/Activity lifecycle callbacks | Side effects (`LaunchedEffect`, `DisposableEffect`) |
| Stable since | API 1 (2008) | Compose 1.0 — August 2021 |

---

## Interview Questions

### Beginner

**Q1: What is Jetpack Compose and how is it different from XML views?**
> Jetpack Compose is Android's modern, declarative UI toolkit built entirely in Kotlin. Unlike XML views (which are imperative — you manually update views when data changes), Compose is declarative — you describe what the UI should look like for a given state, and Compose handles updating the UI automatically when state changes. Compose eliminates the need for XML layout files, `findViewById`, ViewHolder patterns, and manual view updates.

**Q2: What is recomposition in Compose?**
> Recomposition is the process of Compose re-executing composable functions when the state they read changes. Unlike traditional view systems where you redraw the entire screen, Compose's smart recomposition only re-executes composables whose inputs have changed. Composables that haven't changed are skipped entirely (if Compose can determine their output is stable). This makes Compose efficient even with complex UI.

**Q3: What is `remember {}` and when do you use it?**
> `remember {}` is a Compose API that stores a value across recompositions. Without `remember`, a variable in a composable would be re-initialized on every recomposition. `remember { mutableStateOf("") }` creates a `MutableState` object once and keeps it alive across recompositions. It does NOT survive Activity recreation (rotation). For state that must survive rotation, use `rememberSaveable {}`.

---

### Intermediate

**Q4: What are `LaunchedEffect`, `DisposableEffect`, and `SideEffect`? When do you use each?**
> - **`LaunchedEffect(key)`**: Launches a coroutine in the composition scope. Runs when the composable enters composition and is re-launched when `key` changes. Use for: fetching data when screen opens, reacting to parameter changes asynchronously.
> - **`DisposableEffect(key)`**: Runs setup code on enter and cleanup code (`onDispose`) on exit. Not a coroutine. Use for: registering/unregistering listeners, sensor subscriptions, lifecycle observers.
> - **`SideEffect`**: Runs synchronously after every successful recomposition. Use for: sending analytics events, syncing Compose state to non-Compose code (e.g., updating a legacy SDK).

**Q5: Explain state hoisting in Compose.**
> State hoisting is the pattern of moving state from a composable *up* to its caller, making the composable stateless. Instead of `var text by remember { mutableStateOf("") }` inside the composable, the composable receives `text: String` and `onTextChange: (String) -> Unit` as parameters. Benefits: (1) the composable is reusable (same component can be controlled by different state sources), (2) it's testable (you can pass any state), (3) it's previewable, (4) a single source of truth for state. In production, state is typically hoisted all the way to the ViewModel.

**Q6: What is `collectAsStateWithLifecycle()` and why is it preferred over `collectAsState()`?**
> Both convert a `Flow` to Compose `State`. However, `collectAsState()` always collects the flow, even when the composable is in the background (Activity paused). `collectAsStateWithLifecycle()` (from `lifecycle-runtime-compose`) is lifecycle-aware — it pauses collection when the lifecycle drops below `Lifecycle.State.STARTED` and resumes when it returns. This prevents unnecessary recompositions and battery drain when the UI isn't visible, and prevents state from being processed when the UI is stopped.

---

### Advanced

**Q7: How does Compose handle state stability and skipping recomposition?**
> Compose can skip recomposing a function if all its parameters are **stable** and haven't changed. A type is stable if:
> 1. Primitive types (Boolean, Int, String, etc.) — always stable.
> 2. Classes annotated with `@Stable` or `@Immutable`.
> 3. Kotlin data classes with all-stable properties.
> Compose uses the Compose Compiler Plugin to analyze stability at compile time. If all parameters of a composable are stable and unchanged, the composable is **skipped** during recomposition. Unstable types (like `List<T>`, mutable classes without `@Stable`) prevent skipping. Fix: use `kotlinx.collections.immutable.ImmutableList`, annotate with `@Immutable`, or use a `@Stable` wrapper.

**Q8: How does Navigation Compose handle deep links and back stack management?**
> Navigation Compose uses a `NavController` backed by a `NavBackStack`. Each `composable()` destination in `NavHost` has a route (a string pattern, e.g., `"product/{id}"`). Deep links are registered with `deepLinks = listOf(navDeepLink { uriPattern = "https://example.com/product/{id}" })`. When a deep link fires, Navigation matches the URI pattern to a route and navigates there, creating the backstack correctly. `popBackStack()` removes the current destination. `navigate("route") { popUpTo("home") { inclusive = false } }` clears intermediate destinations. Unlike fragment transactions, Navigation Compose manages the backstack declaratively — each destination is a snapshot of the NavBackStack entry.

**Q9: How would you implement a custom `Modifier` in Compose?**
> Compose `Modifier` is a chain of elements. You implement a custom modifier using `Modifier.composed {}` (for stateful modifiers) or by creating a `ModifierElement` (for stateless, performance-critical modifiers). Example:
> ```kotlin
> // Stateless custom modifier
> fun Modifier.coloredBorder(color: Color, width: Dp): Modifier = this.then(
>     Modifier.border(width, color, RoundedCornerShape(4.dp))
> )
>
> // Stateful custom modifier with composed
> fun Modifier.pressScale(): Modifier = composed {
>     var isPressed by remember { mutableStateOf(false) }
>     val scale by animateFloatAsState(if (isPressed) 0.95f else 1f, label = "scale")
>     this
>         .graphicsLayer { scaleX = scale; scaleY = scale }
>         .pointerInput(Unit) {
>             detectTapGestures(
>                 onPress = {
>                     isPressed = true
>                     tryAwaitRelease()
>                     isPressed = false
>                 }
>             )
>         }
> }
> ```

---

## Scenario Questions

**Scenario 1:** Your team has a large existing XML-based app with 50+ screens. The product team wants to adopt Compose. How would you approach the migration?

> **Strategy: Gradual Migration (recommended by Google)**
> 1. **Don't rewrite everything at once** — start with new screens in Compose.
> 2. Enable Compose in the module's `build.gradle.kts` (`buildFeatures { compose = true }`).
> 3. For existing screens, use **`ComposeView`** within existing XML layouts to replace individual components (buttons, cards, complex sections) with Compose incrementally.
> 4. Share `ViewModel` — both XML and Compose screens can use the same ViewModel with `viewModels()` and `hiltViewModel()`.
> 5. Migrate the design system first — implement `MaterialTheme`, colors, typography.
> 6. Migrate simpler screens first to build team competency.
> 7. For Navigation, first migrate within existing screens, then migrate the nav graph to Navigation Compose.
> 8. Set a policy: all new screens must be Compose. Existing screens are migrated opportunistically.

**Scenario 2:** Your Compose list screen is janky — scrolling isn't smooth. The list has 500 items, each with an image, two text fields, and a price calculation. How do you diagnose and fix?

> **Approach:**
> 1. Enable Compose's **recomposition count tracing**: add `RecomposeHighlighter` modifier to see which composables recompose on scroll.
> 2. Check for **missing `key` parameter** in `items()` — without keys, Compose can't optimize insertions/deletions.
> 3. Check for **unstable types** in item composable parameters — if `Product` uses `List<String>` (unstable), every item recomposes on every scroll tick. Fix: annotate `Product` with `@Immutable` or use `kotlinx.collections.immutable`.
> 4. Check **image loading** — use Coil's `AsyncImage` composable which handles async loading with placeholders.
> 5. Move **price calculation** out of the composable into a `remember(product.price) { formatPrice(it) }` or precompute in ViewModel.
> 6. Use **`LazyColumn`** (not `Column` in `verticalScroll`) — ensure images are loading lazily.
> 7. Profile with **Android Studio's Compose Tracing** (API 30+): enables named frames in Perfetto showing which composable caused jank.

---

## Revision Notes

- **Compose = declarative + reactive.** UI = f(state). When state changes, UI updates automatically.
- **Three phases:** Composition → Layout → Drawing. Single-pass layout (vs multiple passes in XML).
- **`remember {}`** survives recompositions; **`rememberSaveable {}`** also survives configuration changes.
- **State hoisting:** move state up to callers; keep composables stateless for testability and reusability.
- **`LaunchedEffect(key)`** for async coroutines; **`DisposableEffect`** for register/unregister; **`SideEffect`** for sync post-recompose work.
- **`LazyColumn`/`LazyRow`** are Compose's RecyclerView equivalents — only compose visible items.
- **`collectAsStateWithLifecycle()`** is preferred over `collectAsState()` for lifecycle-aware collection.
- **`@Stable`/`@Immutable`** annotations enable Compose's skip optimization for data classes.
- **Interop:** `ComposeView` in XML; `AndroidView` in Compose. Both work in either direction.
- **Navigation Compose:** `rememberNavController()` → `NavHost` → `composable("route")` — no Fragment transactions.

---

## Key Takeaways

- 🎨 Compose is **declarative** — describe the UI for a given state; framework handles updates.
- 🔄 **Recomposition** is Compose's mechanism to update UI — only changed composables re-run.
- 💾 `remember {}` → survives recompose. `rememberSaveable {}` → also survives rotation.
- 🏋️ **State hoisting** = move state up; keep composables stateless, reusable, testable.
- 🚀 `LazyColumn`/`LazyRow` = RecyclerView equivalent — compose only visible items.
- ⚡ Use `collectAsStateWithLifecycle()` for lifecycle-safe `Flow` collection in Compose.
- 🧩 **Interop is seamless** — `ComposeView` in XML, `AndroidView` in Compose.
- 📐 `Modifier` is the primary way to style and position composables — always pass it through.
- 🔬 `@Stable`/`@Immutable` annotations unlock smart recomposition skipping — critical for performance.
- 🗺️ **Navigation Compose** replaces Fragment transactions — fully declarative, type-safe routes.

---

---

# Part 5 — Summary & Cross-Chapter Connections

```
Memory Management (Ch.17)
        │
        ├──▶ Coroutine scopes prevent leaks (Ch.17 + Ch.19)
        │    └── lifecycleScope / viewModelScope tied to lifecycle
        │
        ├──▶ Performance (Ch.18): GC pauses cause jank (frame drops)
        │    └── Fewer leaks = fewer GCs = smoother UI
        │
        └──▶ Hilt (Ch.19): wrong scope = memory leak
             └── @Singleton holding Activity context = Classic leak

ANR & Performance (Ch.18)
        │
        ├──▶ Never block Main Thread: DI + Coroutines ensure async work (Ch.19)
        │
        └──▶ Compose (Ch.20): better perf than XML for dynamic UI
             └── Single-pass layout, smart recomposition, no inflation cost

Hilt DI (Ch.19)
        │
        ├──▶ Powers Compose ViewModels: @HiltViewModel (Ch.20)
        │
        └──▶ Scope correctness = Memory safety (Ch.17)

Jetpack Compose (Ch.20)
        │
        ├──▶ State from ViewModel (Hilt Ch.19 + Ch.20)
        │
        └──▶ lifecycleScope / LaunchedEffect = safe async (Ch.17 + Ch.18)
```

---

## Master Checklist — Part 5

- [ ] Can you explain ART's generational GC and why leaks prevent collection?
- [ ] Can you identify all 12 common Android memory leak sources?
- [ ] Can you implement proper Handler/coroutine patterns to avoid leaks?
- [ ] Can you explain ANR thresholds (5s/10s/20s) and how to prevent them?
- [ ] Can you configure StrictMode for debug builds?
- [ ] Can you profile a janky RecyclerView and identify the root cause?
- [ ] Can you set up a complete Hilt DI graph (Application → Module → Repository → ViewModel)?
- [ ] Can you explain the difference between `@Singleton`, `@ActivityScoped`, and unscoped bindings?
- [ ] Can you explain Compose's three phases (Composition, Layout, Drawing)?
- [ ] Can you implement state hoisting correctly in a Compose screen?
- [ ] Can you use `LaunchedEffect`, `DisposableEffect`, and `SideEffect` correctly?
- [ ] Can you integrate Compose into an existing XML project using `ComposeView`?
- [ ] Can you explain why `collectAsStateWithLifecycle()` is preferred over `collectAsState()`?
- [ ] Can you describe the `@Stable`/`@Immutable` optimization and when to apply it?

---

*Part 5 — Chapters 17–20 | Android Fundamentals Master Guide*
*Written for: Senior-level Android interview preparation*
*All Kotlin code examples are production-quality and target API 26+ unless noted.*
*Deprecated APIs are explicitly marked with ⚠️.*
