# Android Interview Question Bank (Part 1: Questions 1-150)

---

## 🏛️ Section 1: Android Architecture & Core Components (Questions 1-30)

### 1. What are the four main app components in Android?
**Answer:** Activity (UI screen), Service (background operation), BroadcastReceiver (system-wide event listener), and ContentProvider (structured data sharing).

### 2. What is the role of `AndroidManifest.xml`?
**Answer:** It declares application metadata, permissions, package identity, hardware prerequisites, and lists all component entry points (Activities, Services, etc.).

### 3. Explain the contents of an APK file.
**Answer:** An APK contains `classes.dex` (compiled code), `resources.arsc` (compiled resources), `res/` (uncompiled raw layouts/assets), `assets/` (raw binary data), `lib/` (architecture-specific `.so` files), and cryptographic signatures inside `META-INF/`.

### 4. What is the Zygote process?
**Answer:** A pre-warmed template process containing pre-loaded JVM resources and system classes. Android forks Zygote using copy-on-write to start new applications rapidly.

### 5. What is the role of the System Server?
**Answer:** It runs inside a separate process and manages core system services like `ActivityManagerService` (AMS), `WindowManagerService` (WMS), and `PackageManagerService` (PMS).

### 6. What is the difference between Dalvik and ART?
**Answer:** Dalvik compiled dex bytecode dynamically at runtime via Just-In-Time (JIT) compilation. ART compiles code during installation or idle time using Ahead-Of-Time (AOT) compilation, reducing CPU utilization and startup delays.

### 7. How does the Low Memory Killer (LMK) prioritize processes?
**Answer:** LMK uses an Out-Of-Memory (OOM) adjustment score (`adj`). Foreground processes have the lowest score (highest priority), while empty/cached background processes have the highest score (first to be killed).

### 8. Explain the concept of Linux sandboxing in Android.
**Answer:** Android assigns a unique User ID (UID) to each application. The app process runs inside an isolated environment where it cannot read or write to other apps' directories or memory without permissions.

### 9. What is a Binder transaction limit?
**Answer:** It is a 1MB process-wide memory limit shared by all active Binder IPC transactions in a process. Exceeding this throws a `TransactionTooLargeException`.

### 10. What is the Hardware Abstraction Layer (HAL)?
**Answer:** A layer that provides standard interfaces exposing hardware capabilities (like camera, Bluetooth, sensors) to the higher-level Java API Framework without exposing implementation details.

### 11. Explain Zygote's Copy-On-Write (COW) optimization.
**Answer:** When Zygote forks an app process, they share the same physical memory pages. Pages are only duplicated when a process attempts to modify them, optimizing overall system RAM.

### 12. What is a DEX file?
**Answer:** Dalvik Executable file format. It consolidates multiple Java class files into a single optimized binary to reduce duplicate constant pool data and fit mobile resource constraints.

### 13. What does `ActivityThread.main()` do?
**Answer:** It serves as the entrance for an Android process, configuring the Main Looper, setting up the IPC binder attachment, and instantiating the default `Application` context.

### 14. What are process adjustment values (`adj` scores)?
**Answer:** Values ranging from negative (system/foreground) to 900+ (cached/empty) assigned to processes, indicating their relative importance to the OS.

### 15. What is the role of `PackageManagerService` (PMS)?
**Answer:** It parses APKs during installation, verifies application signatures, tracks permissions, and resolves implicit intent filters.

### 16. What is an OOM Adjustment (OOM_ADJ) collision?
**Answer:** When background services run concurrently, dropping their priority values and causing the LMK to kill processes simultaneously.

### 17. What is R8?
**Answer:** The modern code shrinker and optimizer that compiles Java bytecode into optimized DEX format, replacing ProGuard.

### 18. What is the difference between cold start, warm start, and hot start?
**Answer:** Cold start builds the process from scratch. Warm start restarts a killed Activity within a cached process. Hot start brings an active, paused Activity from background memory to focus.

### 19. What is the role of `ActivityManagerService` (AMS)?
**Answer:** It manages the lifecycle of all active components, keeps track of the back stack, handles process fork requests, and calculates priority adjustments.

### 20. Why do we need `FullBackupContent` configurations?
**Answer:** To restrict the system from uploading sensitive databases or shared preferences to Google Cloud backups.

### 21. How do you define a custom permission in the manifest?
**Answer:** Use the `<permission>` tag with specific security levels (normal, dangerous, signature).

### 22. What is the significance of the `tools:node="merge"` attribute?
**Answer:** It instructs the manifest merger tool to combine matching tags from dependent libraries rather than overriding them.

### 23. What is an out-of-process component?
**Answer:** A component explicitly set to run in a separate process in the manifest using the `android:process` tag.

### 24. What are native libraries (.so files) in an APK?
**Answer:** Compiled C/C++ binaries targeting specific CPU architectures (ARM/x86) to perform high-performance calculations or run legacy systems.

### 25. Explain the importance of `android:allowBackup`.
**Answer:** If set to true, it permits extracting application private directories via `adb backup`, presenting a security vulnerability if left active in production.

### 26. What is the purpose of `<queries>` in Android 11+?
**Answer:** To declare packages our app needs to interact with. Without it, `PackageManager` queries return filtered results.

### 27. What is an Empty Process?
**Answer:** A process with no active components, kept in memory only as a cache to speed up warm starts.

### 28. What is the role of the System Apps layer?
**Answer:** It contains the core system application components ( Dialer, Browser, Settings) that execute using system-level privileges.

### 29. What is the significance of `ZygoteInit`?
**Answer:** It is the initialization script running inside Zygote that pre-loads framework classes, system resources, and configures the socket connection.

### 30. How does a Signature Permission work?
**Answer:** The permission is only granted if the requesting app is signed with the identical cryptographic certificate as the app defining the permission.

---

## 🔗 Section 2: Application Class & Context (Questions 31-50)

### 31. What is the `Application` class?
**Answer:** The base class for maintaining global state, instantiated before any other class when the process starts.

### 32. What is `Context`?
**Answer:** An interface containing information about the application's environment, system services, directories, and assets.

### 33. What is `ContextImpl`?
**Answer:** The direct concrete subclass of `Context` provided by the OS that implements standard filesystem access, asset management, and service linking.

### 34. What is the Decorator pattern in the Context hierarchy?
**Answer:** `ContextWrapper` delegates all execution to a base `ContextImpl` instance, allowing components like `Application` and `Service` to extend it without direct inheritance constraints.

### 35. Explain the difference between Application Context and Activity Context.
**Answer:** Application Context is a process-level singleton not bound to a theme. Activity Context is short-lived, theme-aware, and bound to a window layout.

### 36. Why is storing an Activity Context in a singleton class a bad practice?
**Answer:** It creates a memory leak. The singleton holds a reference to the Activity after it is destroyed, preventing garbage collection.

### 37. What is `ContextThemeWrapper`?
**Answer:** A context wrapper class that adds theme configurations to a base context, inherited by Activities.

### 38. What does `attachBaseContext()` do?
**Answer:** Binds a concrete `ContextImpl` instance to a `ContextWrapper` subclass (e.g., inside Application/Activity).

### 39. Can you display an AlertDialog using the Application Context?
**Answer:** No. Doing so throws a `BadTokenException` because the Application Context does not contain a valid window token.

### 40. Why is `onTerminate()` not reliable in the Application class?
**Answer:** It is only executed in emulation environments; on production devices, the kernel terminates processes directly.

### 41. What is `onTrimMemory(level)`?
**Answer:** A callback allowing apps to release memory when the system runs low on RAM.

### 42. What is the difference between `TRIM_MEMORY_RUNNING_CRITICAL` and `TRIM_MEMORY_UI_HIDDEN`?
**Answer:** `RUNNING_CRITICAL` means the device is out of memory and the app must release resources. `UI_HIDDEN` means the user closed the app's UI, making it safe to clean up UI assets.

### 43. Explain the Jetpack App Startup Library.
**Answer:** A library providing a unified, thread-pool backed dependency graph to initialize libraries in parallel.

### 44. What is a custom Initializer in App Startup?
**Answer:** A class implementing `Initializer<T>` that defines how to instantiate a library and lists its dependencies.

### 45. What is the hazard of running databases in `Application.onCreate()`?
**Answer:** Doing so synchronously blocks the main thread, causing slow app cold starts and potential ANRs.

### 46. What is the scope of a ContextThemeWrapper?
**Answer:** It exists for the lifetime of the UI component hosting it, preserving visual theme overrides.

### 47. Why does using Application Context to inflate views result in missing styles?
**Answer:** Application Context does not parse the theme attributes associated with UI styles, falling back to default themes.

### 48. What is the relationship between `LoadedApk` and Application instantiation?
**Answer:** `LoadedApk` calls reflection to load the manifest class string and instantiates the `Application` object during startup.

### 49. When does the system call `onLowMemory()`?
**Answer:** A legacy callback executed when the system is low on RAM, equivalent to `onTrimMemory(TRIM_MEMORY_COMPLETE)`.

### 50. Is `onConfigurationChanged()` called on the Application class?
**Answer:** Yes, it is invoked when system-wide changes (language, font size) occur, allowing the app to adjust global configurations.

---

## ⏱️ Section 3: Activity Lifecycle & Launch Modes (Questions 51-90)

### 51. Detail the lifecycle callbacks in order from launch to visibility.
**Answer:** `onCreate()` $\rightarrow$ `onStart()` $\rightarrow$ `onResume()`.

### 52. What happens to Activity A when Activity B is launched?
**Answer:** A's `onPause()` is executed, B's `onCreate()`, `onStart()`, and `onResume()` run, and then A's `onStop()` is executed.

### 53. What is the difference between `onStart()` and `onResume()`?
**Answer:** `onStart()` makes the Activity visible. `onResume()` brings it to the foreground, making it interactive.

### 54. When is `onPause()` executed instead of `onStop()`?
**Answer:** When the Activity is partially covered (e.g., by a dialog or transparent Activity) but remains visible.

### 55. What is the difference between `onSaveInstanceState()` and `onRestoreInstanceState()`?
**Answer:** `onSaveInstanceState()` saves transient UI state to a Bundle before stop. `onRestoreInstanceState()` restores it after start, but only if a saved state exists.

### 56. Does `onRestoreInstanceState()` run on initial app launch?
**Answer:** No. It only executes when recreating the Activity from a saved state.

### 57. What are the four default launch modes?
**Answer:** `standard`, `singleTop`, `singleTask`, and `singleInstance`.

### 58. How does `singleTop` behave?
**Answer:** If the Activity is already at the top of the back stack, it is reused, executing its `onNewIntent()` method.

### 59. Explain the behavior of `singleTask`.
**Answer:** It checks if the Activity exists in the task. If found, it clears all Activities above it, bringing it to the top.

### 60. How does `singleInstance` isolate an Activity?
**Answer:** It runs the Activity as the sole member of a separate task, routing all future launches to a new task.

### 61. What is the default launch mode?
**Answer:** `standard`.

### 62. What is `onNewIntent()`?
**Answer:** A callback invoked when an existing Activity instance is reused due to launch modes or intent flags.

### 63. Why must you call `setIntent(intent)` inside `onNewIntent()`?
**Answer:** If omitted, subsequent calls to `getIntent()` return the original launch intent instead of the new one.

### 64. What is Task Affinity?
**Answer:** A value defining which task an Activity prefers to join. It defaults to the app package name.

### 65. What does `finishAffinity()` do?
**Answer:** Closes the current Activity and all other Activities sharing the same task affinity.

### 66. How does `FLAG_ACTIVITY_NEW_TASK` work?
**Answer:** Starts the Activity in a new task. (Equivalent to `singleTask` launch mode).

### 67. Explain `FLAG_ACTIVITY_CLEAR_TOP`.
**Answer:** If the Activity is already running, it closes all Activities on top of it, delivering the intent.

### 68. What is `FLAG_ACTIVITY_CLEAR_TASK`?
**Answer:** It clears the entire task history before launching the new Activity, used for user logouts.

### 69. What is `FLAG_ACTIVITY_REORDER_TO_FRONT`?
**Answer:** Moves an existing Activity to the top of the stack without destroying intervening Activities.

### 70. How does a configuration change affect the Activity lifecycle?
**Answer:** The system destroys the Activity (`onPause` $\rightarrow$ `onStop` $\rightarrow$ `onDestroy`) and recreates it.

### 71. How does `ViewModel` survive configuration changes?
**Answer:** The `ViewModelStore` is retained across recreation inside `NonConfigurationInstances`.

### 72. What is the limit of data saved in `onSaveInstanceState()`?
**Answer:** The Binder transaction buffer is limited to 1MB process-wide. Keep bundle sizes under 50KB.

### 73. Explain `singleInstancePerTask` (API 31+).
**Answer:** The Activity can only run as the root activity of a task. Only one instance can exist per task.

### 74. What is the role of `TransactionExecutor` in lifecycles?
**Answer:** A framework component that executes lifecycle transition items sequentially.

### 75. Explain the difference between `isFinishing()` and process death.
**Answer:** `isFinishing()` is true when the Activity is closing permanently. Process death terminates the process immediately.

### 76. Why is `repeatOnLifecycle` preferred over `lifecycleScope` for collect flows?
**Answer:** It stops collecting when the Activity goes to the background (`onStop`), saving resources.

### 77. What happens if you block `onPause()`?
**Answer:** It delays the transition to the next Activity, leading to frame drops or ANRs.

### 78. Does a View need an `android:id` to auto-save its state?
**Answer:** Yes. The system uses the ID as the key to map states in the hierarchical Bundle.

### 79. What is the role of `ClientTransaction`?
**Answer:** A Binder container object carrying lifecycle callbacks sent from AMS to the app.

### 80. How do you prevent recreation on orientation change in the manifest?
**Answer:** Declare `android:configChanges="orientation|screenSize"`. (Use with caution).

### 81. What is the lifecycle of an Activity started from a Notification?
**Answer:** It initializes clean: `onCreate()` $\rightarrow$ `onStart()` $\rightarrow$ `onResume()`.

### 82. Explain the purpose of `onWindowFocusChanged()`.
**Answer:** Called when the Activity's window gains or loses focus (e.g., when the keyboard appears).

### 83. Why is `onPostCreate()` used?
**Answer:** A framework hook invoked after `onStart()` that handles final initialization (like drawer layouts).

### 84. What happens when a user locks the device screen?
**Answer:** The active Activity executes `onPause()` and `onStop()`.

### 85. What is the relationship between `ActivityRecord` and `TaskRecord`?
**Answer:** `ActivityRecord` represents an Activity instance. `TaskRecord` groups multiple `ActivityRecord` instances.

### 86. How does `allowTaskReparenting` work?
**Answer:** It allows an Activity to move from the task it started in to the task it has affinity for.

### 87. What does `Activity.isDestroyed()` check?
**Answer:** Returns true if the final `onDestroy()` callback has completed.

### 88. Explain why network calls are banned in `onResume()`.
**Answer:** They block the main thread, causing UI freezes and throwing `NetworkOnMainThreadException`.

### 89. What is the difference between `finish()` and `finishAndRemoveTask()`?
**Answer:** `finish()` closes the Activity. `finishAndRemoveTask()` closes the Activity and removes its task from Recents.

### 90. What is `DefaultLifecycleObserver`?
**Answer:** An interface used to implement lifecycle-aware components without overriding Activity callbacks.

---

## 🧩 Section 4: Fragments (Questions 91-120)

### 91. What is a Fragment?
**Answer:** A modular UI component that runs inside an Activity, sharing its context.

### 92. Why were Fragments introduced?
**Answer:** To support responsive multi-pane layouts on tablet screens and modularize UI logic.

### 93. Detail the Fragment lifecycle from attachment to detachment.
**Answer:** `onAttach` $\rightarrow$ `onCreate` $\rightarrow$ `onCreateView` $\rightarrow$ `onViewCreated` $\rightarrow$ `onStart` $\rightarrow$ `onResume` $\rightarrow$ `onPause` $\rightarrow$ `onStop` $\rightarrow$ `onDestroyView` $\rightarrow$ `onDestroy` $\rightarrow$ `onDetach`.

### 94. What is the difference between `onDestroyView()` and `onDestroy()`?
**Answer:** `onDestroyView()` destroys the View. `onDestroy()` destroys the Fragment instance.

### 95. What is the role of `FragmentManager`?
**Answer:** It executes Fragment transactions, updates the back stack, and syncs states.

### 96. Difference between `supportFragmentManager` and `childFragmentManager`?
**Answer:** `supportFragmentManager` manages fragments added to the Activity. `childFragmentManager` manages nested fragments inside a Fragment.

### 97. Explain the difference between `add()` and `replace()` transactions.
**Answer:** `add()` overlays the container. `replace()` destroys the view of the existing Fragment before adding the new one.

### 98. What does `addToBackStack()` do in a transaction?
**Answer:** Saves the transaction state, allowing the user to reverse it by pressing back.

### 99. Why should you avoid parameterized Fragment constructors?
**Answer:** The system calls the default parameterless constructor via reflection during recreation, losing arguments.

### 100. How do you pass data safely to a Fragment?
**Answer:** Pass arguments as a key-value Bundle using static factory methods.

### 101. Why should you use `viewLifecycleOwner` inside a Fragment?
**Answer:** It represents the lifespan of the View, auto-destroying observers in `onDestroyView()` to prevent duplicates.

### 102. Explain the Fragment Result API.
**Answer:** A decoupled communication framework using key-value results passed between fragments.

### 103. What is the difference between `commit()` and `commitNow()`?
**Answer:** `commit()` runs asynchronously on the main thread queue. `commitNow()` runs synchronously immediately.

### 104. Why does `commit()` throw `IllegalStateException` after `onSaveInstanceState()`?
**Answer:** Because the system cannot save the state of a newly added fragment after the Activity freezes.

### 105. What is `commitAllowingStateLoss()`?
**Answer:** It suppresses the crash that occurs if a commit is executed after state saving, risking state loss on recreation.

### 106. Explain the purpose of `FragmentContainerView`.
**Answer:** The modern container for fragments that resolves layout animation z-ordering issues.

### 107. Why is `onActivityCreated()` deprecated?
**Answer:** It tied view initialization to the Activity's creation state. Use `onViewCreated()` instead.

### 108. Why is `setRetainInstance(true)` deprecated?
**Answer:** It retains fragment instances on rotation, which leaks contexts and conflicts with ViewModels.

### 109. What is the hazard of keeping View Binding references in `onDestroyView()`?
**Answer:** If the Fragment remains in the back stack, the View is destroyed but the binding retains it, leaking memory.

### 110. How do you clean up View Binding in a Fragment?
**Answer:** Set the binding reference to null inside `onDestroyView()`.

### 111. What are headless fragments?
**Answer:** Fragments without a UI (returning null in `onCreateView`), used to encapsulate background tasks.

### 112. Explain static addition of Fragments.
**Answer:** Defining a fragment in layout XML. It is instantiated at inflation and cannot be swapped at runtime.

### 113. What is the difference between `show()` and `hide()` transactions?
**Answer:** They change the visibility of a Fragment's view without destroying it or changing its lifecycle state.

### 114. How do you share a ViewModel between Fragments?
**Answer:** Use the `activityViewModels()` delegate to scope the ViewModel to the host Activity.

### 115. Explain fragment transition animations.
**Answer:** Custom transitions configured inside the transaction block using `setCustomAnimations()`.

### 116. What is the default tag of a Fragment container?
**Answer:** `FragmentContainerView`.

### 117. What is `onViewStateRestored()`?
**Answer:** A callback invoked when all saved states have been restored to the Fragment's view hierarchy.

### 118. Can a Fragment handle hardware back button presses directly?
**Answer:** Not directly, but it can register an `OnBackPressedCallback` with the Activity's dispatcher.

### 119. How do you find a Fragment dynamically?
**Answer:** Use `findFragmentById()` or `findFragmentByTag()` via the `FragmentManager`.

### 120. Can you commit transactions inside lifecycle callbacks?
**Answer:** Yes, but do not do so inside `onDestroy()` or after `onSaveInstanceState()`.

---

## ✉️ Section 5: Intents & Communication (Questions 121-150)

### 121. What is an Intent?
**Answer:** An asynchronous message-passing mechanism used to request actions from other components.

### 122. Difference between Explicit and Implicit Intents?
**Answer:** Explicit Intents name the target class. Implicit Intents declare a general action, allowing the OS to find matching filters.

### 123. What are the key components of an Intent?
**Answer:** Action, Data (URI/MIME), Category, Extras, and Flags.

### 124. What is a PendingIntent?
**Answer:** A wrapper around an Intent that grants another app or the system permission to execute it on our behalf.

### 125. Why is `resolveActivity()` important?
**Answer:** It checks if a receiver exists for an implicit intent before starting it, preventing crashes.

### 126. Explain the security issue of `PendingIntent.FLAG_MUTABLE`.
**Answer:** It allows the receiving app to modify the underlying Intent's extras and actions, potentially hijacking the app.

### 127. What is the Activity Result API?
**Answer:** The modern replacement for `startActivityForResult()`, decoupling result handling from the Activity.

### 128. What is `registerForActivityResult()`?
**Answer:** A contract registration method that sets up callbacks to receive component results.

### 129. How do you pass complex objects via Intents?
**Answer:** By implementing the `Parcelable` interface.

### 130. What is `@Parcelize`?
**Answer:** A Kotlin plugin annotation that auto-generates `Parcelable` implementation boilerplates.

### 131. Explain `FLAG_GRANT_READ_URI_PERMISSION`.
**Answer:** Grants temporary read permission to the target app for a shared content URI.

### 132. What is the difference between Parcelable and Serializable?
**Answer:** `Parcelable` is Android-specific, optimized for IPC, and fast. `Serializable` is standard Java, uses reflection, and is slow.

### 133. What is an Intent Filter?
**Answer:** An XML declaration in the manifest specifying the actions, categories, and data MIME types a component can handle.

### 134. Why is `CATEGORY_DEFAULT` required in intent filters?
**Answer:** `startActivity()` implicitly adds it to all implicit intents; without it, the Activity will not resolve.

### 135. What is the role of `Intent.createChooser()`?
**Answer:** It forces the system chooser dialog to appear, preventing the user from setting a default handler.

### 136. Explain the exported attribute (`android:exported`) in API 31+.
**Answer:** It must be set explicitly for components with intent filters. True allows launch by external apps; false blocks it.

### 137. How do you share files securely between applications?
**Answer:** Using `FileProvider` to generate `content://` URIs instead of `file://` URIs.

### 138. What is the purpose of `intent.setDataAndType()`?
**Answer:** Sets both the URI data and the MIME type simultaneously, clearing any conflicting values.

### 139. Explain the use of `ACTION_SEND`.
**Answer:** An implicit action used to share data (text, images) with other applications.

### 140. How do you read shared data stream in an Activity?
**Answer:** Retrieve the stream URI from `intent.getParcelableExtra(Intent.EXTRA_STREAM)`.

### 141. What is the role of the Binder IPC layer in Intent delivery?
**Answer:** It serializes intent extras and marshals them across process boundaries to the system server.

### 142. How does the system handle multiple matching apps for an implicit intent?
**Answer:** It displays the system resolver dialog, allowing the user to select their preferred app.

### 143. What is `PendingIntent.FLAG_IMMUTABLE`?
**Answer:** A flag making the PendingIntent read-only, preventing other apps from modifying its parameters.

### 144. What is the default category added to standard activities?
**Answer:** `CATEGORY_DEFAULT`.

### 145. What is the purpose of `Intent.FLAG_ACTIVITY_EXCLUDE_FROM_RECENTS`?
**Answer:** Prevents the launched Activity from appearing in the system recents menu.

### 146. How do you launch system settings using an Intent?
**Answer:** Use an implicit intent with actions like `Settings.ACTION_SETTINGS`.

### 147. What is `ActivityResultContract`?
**Answer:** A class defining the input type needed to start an Activity and the output type returned as a result.

### 148. Can you put a Bundle inside Intent extras?
**Answer:** Yes, using `intent.putExtra(name, bundle)`.

### 149. What is `Intent.ACTION_BOOT_COMPLETED`?
**Answer:** A system broadcast action sent when the device finishes booting.

### 150. What is the security risk of launching explicit intents with unvalidated class names?
**Answer:** It exposes the app to component hijacking if class names are injected dynamically from untrusted sources.

---
---

# Android Interview Question Bank (Part 2: Questions 151-300)

---

## 🧵 Section 6: Threading, Loopers & Coroutines (Questions 151-180)

### 151. What is the Main Thread (UI Thread) in Android?
**Answer:** The primary thread created when an application process starts, responsible for rendering the UI, handling user input, and executing component lifecycle callbacks.

### 152. What is an ANR (Application Not Responding) crash?
**Answer:** A system crash triggered when the Main Thread is blocked for more than 5 seconds (by Activities) or 10 seconds (by BroadcastReceivers).

### 153. What is a `Looper`?
**Answer:** A class associated with a thread that executes an infinite loop to dispatch messages and runnables from a `MessageQueue`.

### 154. What is the relationship between `Looper`, `Handler`, and `MessageQueue`?
**Answer:** A `Looper` runs an infinite loop processing messages in a `MessageQueue`. A `Handler` is used to send/post messages or runnables to the `MessageQueue` associated with a specific `Looper`.

### 155. How do you create a custom Thread with a Looper?
**Answer:** By calling `Looper.prepare()` inside the thread's `run()` method, creating a handler, and then calling `Looper.loop()`.

### 156. What is `HandlerThread`?
**Answer:** A subclass of `Thread` with a built-in `Looper`, designed to handle sequential background executions easily.

### 157. Why should you call `quitSafely()` on a `HandlerThread`?
**Answer:** To exit the Looper, releasing thread resources and GC allocations. `quitSafely()` processes all pending messages before terminating.

### 158. Why was `AsyncTask` deprecated in API 30?
**Answer:** It was not lifecycle-aware, caused memory leaks easily by holding implicit Activity references, had inconsistent behavior across OS versions, and could not be cancelled reliably.

### 159. What are Kotlin Coroutines?
**Answer:** A lightweight concurrency framework that executes asynchronous, non-blocking code sequentially using compiler-transformed state machines.

### 160. What is a Coroutine Dispatcher?
**Answer:** A component that controls which thread or thread pool the coroutine executes on (e.g., `Dispatchers.Main`, `Dispatchers.IO`, `Dispatchers.Default`).

### 161. Difference between `Dispatchers.IO` and `Dispatchers.Default`?
**Answer:** `Dispatchers.IO` is optimized for disk or network I/O tasks and expands its thread pool dynamically. `Dispatchers.Default` is optimized for CPU-heavy tasks (sorting, parsing) and caps threads to the core CPU count.

### 162. Explain `viewModelScope` vs `lifecycleScope`.
**Answer:** `viewModelScope` is bound to the `ViewModel`'s lifecycle and cancels active coroutines when the ViewModel is cleared. `lifecycleScope` is bound to a `LifecycleOwner` (Activity/Fragment) and cancels coroutines when destroyed.

### 163. What is the purpose of `withContext`?
**Answer:** A suspend function that switches the execution context of a coroutine to another dispatcher, returning the result without blocking the parent thread.

### 164. What is a suspend function?
**Answer:** A function marked with `suspend` that can pause coroutine execution without blocking the underlying thread, resumed once the operation completes.

### 165. How does the coroutine compiler transformation work?
**Answer:** The Kotlin compiler rewrites suspend functions to include a `Continuation` parameter, converting the code into a state machine that tracks execution points.

### 166. What is `CoroutineScope`?
**Answer:** A scope that defines the boundary and lifecycle of newly created coroutines, holding a `CoroutineContext`.

### 167. What is `SupervisorJob`?
**Answer:** A job configuration where the failure or cancellation of a child coroutine does not propagate upwards to cancel sibling coroutines.

### 168. Difference between `launch` and `async` builders?
**Answer:** `launch` starts a coroutine that does not return a result ("fire and forget"). `async` starts a coroutine returning a `Deferred<T>` value, accessed via `await()`.

### 169. What is structured concurrency?
**Answer:** A design principle ensuring that child coroutines are bound to a parent scope, preventing leaks by automatically cancelling children if the parent cancels.

### 170. What is the Main Looper?
**Answer:** The primary `Looper` running on the Main Thread, accessed via `Looper.getMainLooper()`.

### 171. Explain the "Message Leak" inside Handlers.
**Answer:** If a Handler posts a delayed Message, the Message retains a reference to the Handler, which implicitly retains the host Activity, leaking it if destroyed.

### 172. How do you resolve Handler memory leaks?
**Answer:** Use static inner classes extending `Handler` and wrap the Activity context in a `WeakReference`.

### 173. What is `Message.obtain()`?
**Answer:** Retrieves an existing `Message` from a global recycled pool, avoiding memory allocation overhead.

### 174. What is `StrictMode`?
**Answer:** A developer configuration tool that detects disk writes, network queries, or resource leaks running on the main thread, enforcing penalties.

### 175. What is Thread starvation?
**Answer:** When a thread pool is saturated with long-running tasks, leaving pending operations blocked.

### 176. Explain `Flow` in Kotlin.
**Answer:** An asynchronous cold data stream that emits multiple values sequentially over time, executing only when collected.

### 177. Difference between Cold Flow and Hot Flow?
**Answer:** Cold Flow starts emitting data only when there is an active collector (e.g., `flow { }`). Hot Flow emits data regardless of collectors (e.g., `SharedFlow`, `StateFlow`).

### 178. What is `StateFlow`?
**Answer:** A hot state-holder flow that emits the current state and subsequent state updates to its collectors, requiring an initial value.

### 179. What is `SharedFlow`?
**Answer:** A highly configurable hot flow designed to emit events (like navigation or toasts) to multiple collectors, without requiring an initial value.

### 180. Explain `runBlocking`.
**Answer:** A bridge builder that starts a coroutine and blocks the current thread until the coroutine and all of its children finish execution. (Avoid in production).

---

## ⚙️ Section 7: Services & Process Lifecycle (Questions 181-210)

### 181. What is a Service?
**Answer:** An app component designed to run in the background to perform long-running operations without displaying a UI.

### 182. Does a Service run on a separate thread by default?
**Answer:** No. A Service runs on the Main/UI thread of the application process by default.

### 183. What is a Foreground Service?
**Answer:** A Service that performs operations visible to the user, requiring a persistent, non-dismissible notification.

### 184. What is a Started Service?
**Answer:** A Service initiated via `startService()`, running indefinitely until it stops itself (`stopSelf()`) or is stopped (`stopService()`).

### 185. What is a Bound Service?
**Answer:** A Service initiated via `bindService()`, acting as a server interface allowing components to bind to it, exchange data, and perform IPC.

### 186. Explain the lifecycle of a Started Service.
**Answer:** `onCreate()` $\rightarrow$ `onStartCommand()` $\rightarrow$ (Running) $\rightarrow$ `onDestroy()`.

### 187. Explain the lifecycle of a Bound Service.
**Answer:** `onCreate()` $\rightarrow$ `onBind()` $\rightarrow$ (Bound Client Interaction) $\rightarrow$ `onUnbind()` $\rightarrow$ `onDestroy()`.

### 188. What is the significance of `START_STICKY`?
**Answer:** If the system kills the Service, it recreates it with a null intent parameter, useful for continuous background tasks (like music playback).

### 189. What is `START_NOT_STICKY`?
**Answer:** Instructs the system not to recreate the Service if killed, preventing redundant tasks.

### 190. What is `START_REDELIVER_INTENT`?
**Answer:** Recreates the Service, delivering the original launch intent, useful for file downloads.

### 191. How does a client bind to a Service locally?
**Answer:** By implementing a custom `Binder` subclass inside the Service, returned to the client inside `onServiceConnected()`.

### 192. Explain remote binding via `Messenger`.
**Answer:** An IPC mechanism wrapping a Handler. Clients send Messages to the Service, which processes them sequentially on a single thread.

### 193. What is AIDL (Android Interface Definition Language)?
**Answer:** An interface language used to define cross-process communication (IPC) contracts, allowing multi-threaded remote binding.

### 194. What is `ServiceConnection`?
**Answer:** An interface used to monitor the state of a bound service, implementing `onServiceConnected()` and `onServiceDisconnected()`.

### 195. What is the Foreground Service 5-second rule?
**Answer:** When starting a foreground service via `startForegroundService()`, the service must execute `startForeground()` and display a notification within 5 seconds, otherwise the OS crashes the app.

### 196. Why was `IntentService` deprecated in API 30?
**Answer:** It processed intents on a single worker thread and could not adapt to modern background execution limits. Use WorkManager or Coroutines.

### 197. Explain the background execution limits introduced in API 26 (Oreo).
**Answer:** Apps running in the background cannot launch background services or register most implicit broadcasts statically.

### 198. What is `WorkManager`?
**Answer:** A Jetpack library designed to run guaranteed, deferrable background work, automatically adapting to system limitations and power states.

### 199. When should you use `WorkManager` instead of a Service?
**Answer:** Use `WorkManager` for tasks that must complete even if the app or device restarts. Use Services for immediate, user-visible background tasks.

### 200. What is `onUnbind()` returning true used for?
**Answer:** It instructs the system to invoke `onRebind()` instead of `onBind()` next time a client binds to the Service.

### 201. What does `onServiceDisconnected()` indicate?
**Answer:** Called when the connection to the service is lost, usually because the service crashed or was killed by the OS.

### 202. Can a Bound Service run as a Foreground Service?
**Answer:** Yes, by calling `startForeground()` from within the Service.

### 203. What is the Binder execution pool?
**Answer:** A pool of system-created threads that processes incoming IPC calls (like ContentProvider queries or AIDL calls).

### 204. How do you stop a Bound Service that was also started via `startService()`?
**Answer:** It must be unbound by all clients AND explicitly stopped via `stopSelf()` or `stopService()`.

### 205. What is a JobScheduler?
**Answer:** A system API (API 21+) used to schedule background tasks based on conditions (network type, charging state). Replaced by WorkManager.

### 206. Explain the role of `ForegroundServiceStartNotAllowedException` in Android 12+.
**Answer:** A crash thrown if background apps try to start foreground services without satisfying specific exception conditions.

### 207. What is an isolated service process?
**Answer:** A service declared in the manifest with `android:isolatedProcess="true"`, running in a sandbox without permissions.

### 208. Explain the difference between `stopSelf()` and `stopSelf(int startId)`.
**Answer:** `stopSelf()` stops the service immediately. `stopSelf(startId)` only stops the service if the `startId` matches the most recent intent request.

### 209. What is a sticky binding?
**Answer:** Binding to a service with the `BIND_AUTO_CREATE` flag, which automatically starts the service if it is not running.

### 210. What is the lifetime of a Service process compared to a Background process?
**Answer:** A Service process has higher priority (ADJ 500) and is kept alive longer by the LMK compared to an invisible Activity process (ADJ 800+).

---

## 📡 Section 8: Broadcast Receivers (Questions 211-230)

### 211. What is a BroadcastReceiver?
**Answer:** A component that listens for system-wide or custom broadcast intents.

### 212. What is the difference between Static and Dynamic registration?
**Answer:** Static registration is defined in `AndroidManifest.xml` and can launch a closed app. Dynamic registration is registered in code and tied to a lifecycle.

### 213. Explain the thread context of `onReceive()`.
**Answer:** `onReceive()` executes on the main thread and must not block for more than 10 seconds.

### 214. What is `goAsync()`?
**Answer:** A method in `onReceive()` that returns a `PendingResult`, granting an extra 10 seconds to process work on a background thread.

### 215. What are Ordered Broadcasts?
**Answer:** Broadcasts delivered sequentially to receivers based on priority, allowing high-priority receivers to intercept or modify the data.

### 216. How do you abort propagation in an ordered broadcast?
**Answer:** Call `abortBroadcast()` inside `onReceive()`.

### 217. Why is `LocalBroadcastManager` deprecated?
**Answer:** It bypassed security layers, was not process-safe, and is replaced by reactive streams (Flow/SharedFlow).

### 218. What is a sticky broadcast?
**Answer:** A legacy broadcast whose intent remains in the system cache after completion, allowing future receivers to read it. (Deprecated).

### 219. How does API 26 restrict Static Broadcast Receivers?
**Answer:** Most implicit system broadcasts are blocked from launching statically registered receivers to save battery.

### 220. What is an explicit broadcast?
**Answer:** A broadcast targeted to a specific package name, bypassing background restrictions.

### 221. How do you enforce security when sending a broadcast?
**Answer:** Pass a custom permission string inside `sendBroadcast(intent, permission)`.

### 222. How does `setResultData()` work in ordered broadcasts?
**Answer:** It allows a receiver to modify the broadcast data payload before it reaches subsequent receivers.

### 223. What lifecycle stage is recommended to unregister dynamic receivers?
**Answer:** Unregister in the matching lifecycle: register in `onStart()`, unregister in `onStop()`; or register in `onResume()`, unregister in `onPause()`.

### 224. Explain the hazard of registering a receiver dynamically in `onCreate()` and not unregistering it.
**Answer:** It creates a memory leak. The system retains a reference to the receiver, leaking the host Activity context.

### 225. What is the difference between `sendBroadcast()` and `sendOrderedBroadcast()`?
**Answer:** `sendBroadcast()` sends intents asynchronously to all receivers simultaneously. `sendOrderedBroadcast()` delivers them sequentially.

### 226. What happens if a BroadcastReceiver runs a network query in `onReceive()`?
**Answer:** It throws `NetworkOnMainThreadException` and crashes the application.

### 227. Can a BroadcastReceiver start a Foreground Service?
**Answer:** Yes, by calling `context.startForegroundService(intent)`.

### 228. How does `android:exported` affect security in Broadcast Receivers?
**Answer:** Setting it to false restricts the receiver to only accept broadcasts sent from within the same application.

### 229. What is `Telephony.Sms.Intents.SMS_RECEIVED_ACTION`?
**Answer:** An ordered system broadcast triggered when a new SMS is received.

### 230. Why is `ACTION_BATTERY_CHANGED` restricted to dynamic registration?
**Answer:** Because it triggers frequently; static registration would constantly wake up background processes, draining the battery.

---

## 🗄️ Section 9: Content Providers (Questions 231-250)

### 231. What is a ContentProvider?
**Answer:** A component that manages access to a structured database, allowing data sharing inside or between apps.

### 232. What is `ContentResolver`?
**Answer:** A client-side interface used to interact with a ContentProvider, executing CRUD operations.

### 233. Detail the ContentProvider URI structure.
**Answer:** `content://authority/path/id` (e.g., `content://com.example.provider/users/1`).

### 234. What are the abstract CRUD methods of a ContentProvider?
**Answer:** `query()`, `insert()`, `update()`, `delete()`, `getType()`, and `onCreate()`.

### 235. Explain `Cursor`.
**Answer:** A pointer to a result set returned from a database query. It must always be closed to prevent memory leaks.

### 236. What is `ContentObserver`?
**Answer:** A class that listens for data modifications on a specific content URI, notifying components of changes.

### 237. What is `FileProvider`?
**Answer:** A subclass of `ContentProvider` that shares files securely via temporary content URIs instead of file paths.

### 238. Why are `file://` URIs forbidden in modern Android?
**Answer:** Sharing them throws `FileUriExposedException` because they require global directory read/write access, violating security sandbox rules.

### 239. What does `getType()` return?
**Answer:** Returns the MIME type of the data associated with a specific ContentProvider URI.

### 240. Explain the thread safety of ContentProvider methods.
**Answer:** Queries run on binder pool threads, not the main thread. CRUD operations must be thread-safe.

### 241. What is `UriMatcher`?
**Answer:** A utility class that parses URI structures, mapping them to integer codes to dispatch database actions.

### 242. Explain temporary URI permissions.
**Answer:** Temporary read/write access granted to target apps via `FLAG_GRANT_READ_URI_PERMISSION`.

### 243. Why was `CursorLoader` deprecated?
**Answer:** It was tied to the Fragment/Activity lifecycle. It is replaced by ViewModels observing Room databases.

### 244. How does `ContentProvider.onCreate()` behave during startup?
**Answer:** It is called on the main thread during app launch, before the `Application` class executes. Keep its initialization minimal.

### 245. Can you use Room database inside a ContentProvider?
**Answer:** Yes, Room can serve as the data source behind the ContentProvider's CRUD operations.

### 246. What is a Contract Class?
**Answer:** A helper class defining database schemas, column names, authority strings, and URIs (e.g., `ContactsContract`).

### 247. What is `grantUriPermission()`?
**Answer:** A system API used to grant specific package access to a URI dynamically.

### 248. How do you implement a secure private ContentProvider?
**Answer:** Set `android:exported="false"` in the manifest, restricting access to your own application.

### 249. What is the role of `BulkInsert()`?
**Answer:** An optimized transaction method used to insert multiple records into the database in a single batch.

### 250. How does the system resolve authority strings?
**Answer:** The `PackageManager` matches the authority string in the URI to the registered providers list in the manifest.

---

## 🛡️ Section 10: Runtime Permissions (Questions 251-270)

### 251. What are Runtime Permissions?
**Answer:** Permissions requiring user approval during app execution (introduced in API 23), rather than at install time.

### 252. What is the difference between Normal and Dangerous permissions?
**Answer:** Normal permissions (e.g., Internet) are granted automatically at install. Dangerous permissions (e.g., Camera) access sensitive data and require runtime user approval.

### 253. What are Signature Permissions?
**Answer:** Permissions granted automatically only if the requesting app is signed with the same key as the defining app.

### 254. How do you check if a permission is granted?
**Answer:** By calling `ContextCompat.checkSelfPermission()`.

### 255. What does `shouldShowRequestPermissionRationale()` do?
**Answer:** Returns true if the user previously denied the permission request, indicating we should explain why it is needed.

### 256. What is the modern Activity Result API contract for requesting permissions?
**Answer:** `ActivityResultContracts.RequestPermission()` or `RequestMultiplePermissions()`.

### 257. What are Permission Groups?
**Answer:** Groups of related permissions (e.g., Contacts, Storage). Granting one permission in a group dynamically grants the others.

### 258. What happens if a user checks "Don't ask again" and denies a permission?
**Answer:** Subsequent permission requests are denied automatically without prompting the user. The app must guide them to Settings.

### 259. What are One-Time Permissions (API 30+)?
**Answer:** Dynamic permissions granted only for the duration of the current session, revoked once the app closes.

### 260. How does Android 13 (API 33) handle storage permissions?
**Answer:** It replaces the global storage permission with granular media permissions: `READ_MEDIA_IMAGES`, `READ_MEDIA_VIDEO`, and `READ_MEDIA_AUDIO`.

### 261. Explain the Photo Picker API.
**Answer:** A system UI tool allowing users to select specific photos to share with an app without granting it storage permissions.

### 262. What is `MANAGE_EXTERNAL_STORAGE`?
**Answer:** A special signature permission granting broad filesystem access, reserved only for apps like file managers.

### 263. What is the permission requirement for background location access (API 29+)?
**Answer:** Apps must request `ACCESS_BACKGROUND_LOCATION` separately, after foreground location has been granted.

### 264. How do you define a custom permission tree?
**Answer:** Using the `<permission-tree>` manifest tag to define a namespace for dynamic permissions.

### 265. What happens to background tasks if a user revokes permissions in system Settings?
**Answer:** The system terminates the application process immediately to apply the permission changes.

### 266. Can an app declare custom permissions to share data with other apps?
**Answer:** Yes, by declaring a `<permission>` tag in the manifest with specific protection levels.

### 267. What is a system-level runtime permission prompt?
**Answer:** A system dialog launched by calling the Activity Result API contract, which cannot be customized by the developer.

### 268. What is the role of `PackageManager.PERMISSION_GRANTED`?
**Answer:** An integer constant (value 0) returned by the OS verifying the permission is active.

### 269. What is the requirement for Bluetooth permissions in API 31+?
**Answer:** It separates scans and connects, requiring `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT` without needing location access.

### 270. Explain notification runtime permissions in Android 13+.
**Answer:** Apps must request `POST_NOTIFICATIONS` at runtime before showing push notifications.

---

## 🏛️ Section 11: MVVM, ViewModels & LiveData/Flow (Questions 271-290)

### 271. What is MVVM?
**Answer:** Model-View-ViewModel. An architectural pattern that separates the UI (View) from the business logic (ViewModel) and data sources (Model).

### 272. What are the key responsibilities of the ViewModel?
**Answer:** It holds UI state, processes user actions, interacts with the repository, and survives configuration changes.

### 273. How does the View interact with the ViewModel in MVVM?
**Answer:** The View observes state streams (LiveData/StateFlow) exposed by the ViewModel and updates itself.

### 274. What is the role of the Repository pattern?
**Answer:** It abstracts data sources (local database, remote API), providing a clean API to the ViewModel.

### 275. Explain Unidirectional Data Flow (UDF).
**Answer:** A pattern where state flows down (ViewModel $\rightarrow$ View) and events flow up (View $\rightarrow$ ViewModel).

### 276. Why should you expose immutable state from a ViewModel?
**Answer:** To prevent the View from modifying the state directly, maintaining the ViewModel as the single source of truth.

### 277. What is LiveData?
**Answer:** An observable data holder class that is lifecycle-aware, emitting updates only to active observers (in STARTED/RESUMED states).

### 278. What is the difference between LiveData and StateFlow?
**Answer:** LiveData is lifecycle-aware and runs on the main thread. StateFlow is a Kotlin coroutine flow that requires an initial state and is not lifecycle-aware out of the box (requires `repeatOnLifecycle`).

### 279. How do you share a ViewModel between Fragments?
**Answer:** Use the `activityViewModels()` property delegate.

### 280. What is `SavedStateHandle`?
**Answer:** A key-value map injected into the ViewModel constructor that allows data to survive configuration changes and process death.

### 281. Explain the difference between configuration change and process death.
**Answer:** Configuration change recreates the Activity instance but retains the ViewModel. Process death terminates the app process completely, requiring data recovery from saved state bundles.

### 282. How do you handle one-time events (like Toasts or Navigation) in MVVM?
**Answer:** Use a Kotlin coroutine `Channel` or `SharedFlow` to emit events that are consumed once.

### 283. Why should you avoid passing `Context` references into a ViewModel?
**Answer:** It causes memory leaks. The ViewModel outlives the Activity context. If context must be used, inject Application Context via `AndroidViewModel`.

### 284. Explain `viewModelScope.launch`.
**Answer:** Launches a coroutine scoped to the ViewModel, automatically cancelling all child coroutines when `onCleared()` is called.

### 285. What does `onCleared()` do?
**Answer:** A ViewModel lifecycle method invoked when the host Activity or Fragment is finished, used to release resources.

### 286. What is a ViewModel Factory?
**Answer:** A class implementing `ViewModelProvider.Factory` used to pass custom parameters to a ViewModel constructor.

### 287. How does Hilt simplify ViewModel injection?
**Answer:** By using the `@HiltViewModel` annotation, allowing dependencies to be injected automatically without manual factories.

### 288. Explain the Single Source of Truth (SSOT) pattern.
**Answer:** A database (like Room) serves as the primary source of truth. Remote network queries update the database, and the UI observes database changes.

### 289. How does `collectAsState()` work in Compose?
**Answer:** An extension function that collects emissions from a StateFlow and converts them into a Compose `State<T>`, triggering recompositions on updates.

### 290. What is `SingleLiveEvent`?
**Answer:** A custom legacy LiveData subclass that sends updates only once, preventing event re-emission on rotation. Replaced by Channels.

---

## 🧭 Section 12: Navigation & Data Persistence (Questions 291-300)

### 291. What is the Jetpack Navigation Component?
**Answer:** A declarative framework that manages fragment transactions, back stack navigation, animations, and deep linking.

### 292. Explain `NavBackStackEntry`.
**Answer:** Represents a destination in the navigation back stack, holding its own lifecycle, viewmodel store, and saved state registry.

### 293. What is Safe Args?
**Answer:** A Gradle plugin that generates type-safe directions and arguments classes for navigation actions.

### 294. What is the difference between `navigateUp()` and `popBackStack()`?
**Answer:** `navigateUp()` navigates up the hierarchical graph structure. `popBackStack()` pops destinations chronologically.

### 295. Why is SharedPreferences deprecated in favor of DataStore?
**Answer:** SharedPreferences executes synchronous I/O operations on the main thread, lacks type safety, and can block the main thread.

### 296. Explain the difference between Preferences DataStore and Proto DataStore.
**Answer:** Preferences DataStore stores key-value pairs using Flow. Proto DataStore stores typed objects using Protocol Buffers.

### 297. What is Room?
**Answer:** A compile-time safe Object-Relational Mapping (ORM) library built on top of SQLite.

### 298. Explain Room migrations.
**Answer:** Code definitions handling database updates when the schema changes, executed via `Migration` classes.

### 299. What does the `@Dao` annotation define?
**Answer:** Data Access Object. It defines interfaces containing SQL queries mapped to functions.

### 300. How do you query database changes reactively in Room?
**Answer:** By returning a `Flow` or `LiveData` list from a DAO query.

---
---

# Android Interview Question Bank (Part 3: 100 Scenario-Based Questions)

---

## 🚀 Section 1: Startup & Component Lifecycles (Scenarios 1-20)

### 1. Scenario: Your app takes 10 seconds to open, showing a blank screen. What is causing this, and how do you analyze it?
* **Answer:** Synchronous heavy tasks (like DB, network, or SDK setups) are running inside `Application.onCreate()` on the main thread. Analyze by running adb command `adb shell am start -W` and profiling startup using Android Studio CPU Profiler or Perfetto. Resolve by offloading initializations to a coroutine or using the App Startup library.

### 2. Scenario: A user receives a phone call while using your form Activity. Describe the lifecycle callbacks and what you must protect.
* **Answer:** The Activity transitions `onPause() -> onStop()`. If the system runs out of memory, it may kill the process. You must save current form input strings inside `onSaveInstanceState()` using key-value entries to prevent data loss.

### 3. Scenario: The app crashes with `WindowManager$BadTokenException` when showing a dialog. Why?
* **Answer:** The developer passed the `Application` context instead of an `Activity` context to the `AlertDialog.Builder`. Dialogs require a window token, which is only present in Activity Context wrappers.

### 4. Scenario: A custom View inflated with `applicationContext` is missing its custom styling. How do you fix it?
* **Answer:** Pass the active `Activity` context (which contains theme declarations) during View inflation instead of `applicationContext`, which only possesses default system theme fallback rules.

### 5. Scenario: A Singleton class uses a context reference to access databases. When the user rotates their screen, the app leaks memory. Explain the leak and fix.
* **Answer:** The singleton was initialized with an `Activity` context. When the Activity rotated, it was destroyed but couldn't be garbage collected because the singleton held a reference to it. Fix by calling `context.applicationContext` inside the singleton initialization.

### 6. Scenario: Your app needs to run a task only when it returns from the background to the foreground. Which callback handles this?
* **Answer:** Register a `LifecycleObserver` utilizing `ProcessLifecycleOwner.get().lifecycle`. Handle foreground triggers in `onStart()` or `onResume()`.

### 7. Scenario: A developer registers a network BroadcastReceiver in `onCreate()` but does not unregister it. What happens when the Activity rotates?
* **Answer:** The old Activity instance is destroyed but cannot be garbage collected because the system's broadcast dispatcher holds a reference to its receiver, causing a memory leak on every rotation. Unregister the receiver in `onDestroy()`.

### 8. Scenario: You need to start MainActivity from a background BroadcastReceiver. What intent parameters are mandatory?
* **Answer:** You must set `Intent.FLAG_ACTIVITY_NEW_TASK` because the BroadcastReceiver runs outside a window task stack context; without this flag, starting the Activity will crash.

### 9. Scenario: An Activity's `onDestroy()` is never called when the user exits via the home button. Is this normal?
* **Answer:** Yes. Navigating home only stopped the Activity (`onPause() -> onStop()`). If the system needs RAM later, it terminates the process directly, skipping `onDestroy()`. Do not write critical save logic in `onDestroy()`.

### 10. Scenario: Views in your layout do not restore their states (scroll position, text inputs) on rotation. What did the developer forget?
* **Answer:** The developer forgot to assign unique `android:id` values to the views in the XML layout. The system requires IDs to map states in the restoration Bundle.

### 11. Scenario: You configure `android:configChanges="orientation|screenSize"` in the manifest. What lifecycle methods are executed on rotation?
* **Answer:** The Activity is NOT destroyed and recreated. No lifecycle callbacks are executed; instead, the system invokes the `onConfigurationChanged()` callback directly, allowing manual layout adjustments.

### 12. Scenario: Your app has a heavy onboarding flow. After the user logs in, they navigate to the Home screen. Clicking back should close the app. What is the solution?
* **Answer:** Start `HomeActivity` with flags `FLAG_ACTIVITY_NEW_TASK` and `FLAG_ACTIVITY_CLEAR_TASK`. This clears the onboarding task history, making `HomeActivity` the sole root.

### 13. Scenario: A Fragment's View is recreated, but it registers duplicate LiveData observers, causing double emissions. Why?
* **Answer:** The developer registered observers using the Fragment instance (`this`) instead of `viewLifecycleOwner`. The Fragment instance survived the view recreation, causing it to retain the old observer when adding a new one.

### 14. Scenario: The app crashes with `IllegalStateException: Can not perform this action after onSaveInstanceState`. How do you fix it?
* **Answer:** A fragment transaction was committed asynchronously after the Activity was backgrounded. Fix by either committing the transaction before backgrounding, using `commitAllowingStateLoss()`, or postponing the transaction until `onResume()`.

### 15. Scenario: A ViewPager has 5 fragment tabs. Swiping between them is laggy due to repeated fragment creation. How do you optimize?
* **Answer:** Set `viewPager.offscreenPageLimit = 4` to keep the views of offscreen fragments active in the memory cache, preventing repeated layout inflation.

### 16. Scenario: You want to send data from a DialogFragment back to its parent Fragment. What is the modern approach?
* **Answer:** Use the Fragment Result API: the parent registers `setFragmentResultListener("requestKey")` and the dialog returns values via `setFragmentResult("requestKey", bundle)`.

### 17. Scenario: A custom library needs initialization inside your app. You want to run it only in a separate background process declared in the manifest. How do you check this?
* **Answer:** Compare the current process name (retrieved via `ActivityManager` or `Application.getProcessName()`) with your main package name inside `Application.onCreate()`, executing the library setup only in the target process.

### 18. Scenario: Your app is killed in the background. When the user returns, the app crashes because a static utility class variable was reset to null. Why?
* **Answer:** Background process death cleared all static variables in the VM. When the app was recreated, the Activity restored its state, but the static utility variables remained null. Fix by initializing the variable inside `onCreate()`.

### 19. Scenario: How do you measure the exact cold startup duration of your app programmatically?
* **Answer:** Use the Jetpack Macrobenchmark library to capture startup metrics, or read the time difference between `Application.onCreate()` and the Activity's `onWindowFocusChanged()` callback.

### 20. Scenario: An Activity needs to lock the screen in portrait mode dynamically. How is this achieved?
* **Answer:** Call `requestedOrientation = ActivityInfo.SCREEN_ORIENTATION_PORTRAIT` inside the Activity.

---

## 🗺️ Section 2: Tasks, Launch Modes, Navigation (Scenarios 21-40)

### 21. Scenario: Your app stack contains A -> B -> C. C launches B with `singleTask`. What is the resulting stack?
* **Answer:** The stack becomes `[A -> B]`. Activity C is destroyed, and the intent is delivered to B via `onNewIntent()`.

### 22. Scenario: Activity A starts B, which is declared as `singleInstance`. B then starts C (standard). What do the tasks look like?
* **Answer:** Task 1 contains A. Task 2 contains B (isolated). Task 3 contains C (since `singleInstance` cannot host other activities in its task, C is pushed into a new task).

### 23. Scenario: A chat app notification should open a specific ChatActivity. If the app is closed, it should open ChatActivity with a back button that goes to MainActivity. How do you handle this?
* **Answer:** Use `TaskStackBuilder` when building the notification's `PendingIntent`. This constructs a synthetic back stack containing `MainActivity` parent and `ChatActivity` child.

### 24. Scenario: A user clicks a link that deep links into your app. What launch configuration prevents duplicating the MainActivity if it is already open?
* **Answer:** Set `launchMode="singleTop"` or launch the intent with flag `FLAG_ACTIVITY_SINGLE_TOP` in combination with `FLAG_ACTIVITY_CLEAR_TOP`.

### 25. Scenario: A payment gateway Activity is declared as `singleInstance`. During checkout, it redirects back to the main app, but the back stack navigation is broken. Why?
* **Answer:** Because `singleInstance` isolated the checkout screen in its own task. Back navigation is now cross-task, which breaks standard chronological transitions. Use `singleTask` or standard navigation flags instead.

### 26. Scenario: You want to clear the entire Activity stack and exit the app on clicking a logout button. How?
* **Answer:** Start the launcher Activity with flags `FLAG_ACTIVITY_NEW_TASK` and `FLAG_ACTIVITY_CLEAR_TASK`, then immediately call `finish()`.

### 27. Scenario: A user has two tasks from your app in the recents screen. Why did this happen?
* **Answer:** One of the Activities was launched into a separate task using custom `taskAffinity` combined with `FLAG_ACTIVITY_NEW_TASK`.

### 28. Scenario: An Activity started from a widget fails to launch. The logs show `AndroidRuntimeException: Calling startActivity() from outside of an Activity context requires the FLAG_ACTIVITY_NEW_TASK flag`. Fix it.
* **Answer:** The widget context is not an Activity context. Set the flag `Intent.FLAG_ACTIVITY_NEW_TASK` on the intent before passing it to the PendingIntent.

### 29. Scenario: How do you verify the current back stack structure in a running device?
* **Answer:** Run the ADB command: `adb shell dumpsys activity activities` and inspect the active task records.

### 30. Scenario: An Activity in your stack is finished, but pressing back still shows it. Why?
* **Answer:** The Activity was not removed from the stack. Verify if `finish()` was executed, or if intent flags like `FLAG_ACTIVITY_NO_HISTORY` should be set to auto-close the screen when navigating away.

### 31. Scenario: You are using Jetpack Navigation. When swapping tabs in a BottomNavigationView, the fragment state is reset. How do you fix it?
* **Answer:** Enable multiple back stacks support in Navigation 2.4+ by calling `setupWithNavController()` on the navigation view, which automatically handles state preservation.

### 32. Scenario: How do you pass type-safe arguments between destinations in Jetpack Navigation?
* **Answer:** Enable the Safe Args plugin in your Gradle build files, and use the generated `Directions` and `Args` classes to pass arguments.

### 33. Scenario: A deep link fails to open your app when clicked. The manifest looks correct. What is missing?
* **Answer:** Ensure the Intent Filter specifies the scheme (`http/https`), host name, and category `DEFAULT` and `BROWSABLE`.

### 34. Scenario: You want to prevent a user from returning to a splash screen when clicking back from MainActivity. How?
* **Answer:** Call `finish()` inside `SplashActivity` immediately after calling `startActivity(MainActivity)`.

### 35. Scenario: Can an Activity have multiple instances in a task stack in standard mode?
* **Answer:** Yes, starting it repeatedly creates new instances (e.g., [A -> A -> A]).

### 36. Scenario: How do you move an existing background Activity to the front of its task without clearing other screens?
* **Answer:** Start the Activity using the flag `Intent.FLAG_ACTIVITY_REORDER_TO_FRONT`.

### 37. Scenario: You want to check if a specific fragment is currently visible inside your container. How?
* **Answer:** Call `parentFragmentManager.findFragmentById(R.id.container)` and verify if it is an instance of the target fragment class.

### 38. Scenario: A developer manually updates the FragmentManager transactions inside a NavHostFragment, breaking navigation. Why?
* **Answer:** Mixing manual fragment transactions with Jetpack NavigationController corrupts the internal back stack. Navigation must only be performed via the NavController.

### 39. Scenario: You need to pass an argument from Fragment A to Fragment B. B is a nested child fragment. What manager do you use?
* **Answer:** Use `childFragmentManager` to initialize and add B, passing the arguments via B's arguments bundle.

### 40. Scenario: An Activity is launched with `singleTask` but it recreates its instance instead of calling `onNewIntent()`. Why?
* **Answer:** Ensure the `android:launchMode="singleTask"` configuration is declared on the correct `<activity>` tag in the manifest.

---

## 🧵 Section 3: Services, Threads, Coroutines (Scenarios 41-60)

### 41. Scenario: A Started Service downloads a large file. The OS kills the service due to low RAM. Once RAM is freed, the service restarts but fails because the input intent is null. How do you fix it?
* **Answer:** The service returned `START_STICKY` in `onStartCommand()`, causing it to restart with a null intent. Return `START_REDELIVER_INTENT` instead, which redelivers the original intent.

### 42. Scenario: Your background service runs location updates. In Android 12, it crashes immediately on start. What is the cause?
* **Answer:** The app attempted to start a background service without showing a notification first, violating Android 12+ background execution limits. Start it as a foreground service via `startForegroundService()` and call `startForeground()` within 5 seconds.

### 43. Scenario: A background Thread keeps executing network updates after the parent Activity is destroyed. What is the hazard and how do you resolve it?
* **Answer:** The thread is leaking the Activity context if it holds a reference to it, wasting battery. Resolve by using Kotlin Coroutines bound to `lifecycleScope`, which automatically cancels execution when the Activity is destroyed.

### 44. Scenario: A coroutine executing disk reads inside `viewModelScope.launch` blocks the UI thread. Why?
* **Answer:** The coroutine was executed on the default dispatcher (`Dispatchers.Main`). Move the disk I/O operation to the I/O thread pool using `withContext(Dispatchers.IO)`.

### 45. Scenario: A child coroutine inside a scope throws an network exception, causing all other tasks in the scope to cancel. How do you prevent this?
* **Answer:** Initialize the scope with a `SupervisorJob` instead of a standard `Job`, preventing child failures from propagating and cancelling sibling coroutines.

### 46. Scenario: How do you run two network requests in parallel and combine their results using coroutines?
* **Answer:** Use `async` builders: `val call1 = async { api.getA() }`, `val call2 = async { api.getB() }`. Combine the results using `val result = call1.await() + call2.await()`.

### 47. Scenario: You want to run a recurring background sync task every 24 hours, even if the device restarts. What API do you use?
* **Answer:** Use `WorkManager` to schedule a `PeriodicWorkRequest` configured with constraints like network connectivity.

### 48. Scenario: An app crashes with `NetworkOnMainThreadException`. How do you identify the offending call?
* **Answer:** Configure `StrictMode` in the debug build to detect disk or network writes on the main thread and print stack traces to Logcat.

### 49. Scenario: A Bound Service is not stopping after calling `stopSelf()`. Why?
* **Answer:** The service is still bound to one or more active clients. A bound service will not stop until all bound clients call `unbindService()`.

### 50. Scenario: You want to send progress updates from a background Service to an Activity. How?
* **Answer:** Bind the service locally and access its public properties, or broadcast updates via custom broadcasts, or post progress values to a shared database observed by the UI.

### 51. Scenario: How do you verify if a Service is currently running on the device?
* **Answer:** Query active services list using `ActivityManager.getRunningServices()`.

### 52. Scenario: A thread starvation issue is freezing your app's networking queue. How do you resolve it?
* **Answer:** Verify that network clients use connection pools and offload calls to `Dispatchers.IO`, which scales its threads up to 64 to prevent starvation.

### 53. Scenario: Why does calling `runBlocking` on the main thread freeze the UI?
* **Answer:** Because `runBlocking` blocks the calling thread until the coroutine finishes execution. Doing so on the main thread blocks UI rendering and event loop handling, causing the app to freeze.

### 54. Scenario: A coroutine is observing database updates. When the user backgrounds the app, the data stream is still active, consuming battery. How do you fix it?
* **Answer:** Collect the Flow inside the Activity/Fragment using `repeatOnLifecycle(Lifecycle.State.STARTED)` to automatically pause collecting when the app is backgrounded.

### 55. Scenario: What is the risk of using `GlobalScope` to launch coroutines?
* **Answer:** `GlobalScope` coroutines are not bound to any lifecycle. They continue running even if the calling component is destroyed, causing memory leaks and wasting resources.

### 56. Scenario: How do you cancel a coroutine manually?
* **Answer:** Hold a reference to the `Job` returned by the coroutine builder, and call `job.cancel()`.

### 57. Scenario: A coroutine loop does not respond to cancellation. Why?
* **Answer:** The loop is executing CPU-heavy work without cooperative yield points. Call `ensureActive()` or `yield()` inside the loop to check for cancellation state.

### 58. Scenario: You want to execute a task immediately when a Bound Service connection completes. Where do you write it?
* **Answer:** Write the execution code inside `ServiceConnection.onServiceConnected()`.

### 59. Scenario: Can a Service display a Toast?
* **Answer:** Yes, since a Service inherits from `ContextWrapper` and has access to system services, it can show a Toast on the main thread.

### 60. Scenario: What is the risk of starting a background thread inside a statically registered BroadcastReceiver?
* **Answer:** Once `onReceive()` finishes, the receiver's lifecycle ends. The system may kill the host process immediately, terminating the background thread before it completes. Use `WorkManager` instead.

---

## 📡 Section 4: Broadcasts, Content Providers, Security (Scenarios 61-80)

### 61. Scenario: You need to send a broadcast that can only be received by your own application. How do you enforce this?
* **Answer:** Set `android:exported="false"` on the receiver in the manifest, or set the package name on the Intent before calling `sendBroadcast()`.

### 62. Scenario: Your app listens for SMS broadcasts. How do you prevent other apps from reading or intercepting the SMS data?
* **Answer:** Request signature protection level permissions on your receiver, or use the SMS Retriever API for secure user verification.

### 63. Scenario: You want to share a file securely with a camera app to save a photo. What URI scheme do you use?
* **Answer:** Use `FileProvider` to generate a secure `content://` URI and grant temporary write permission via `Intent.FLAG_GRANT_WRITE_URI_PERMISSION`.

### 64. Scenario: A ContentProvider query throws an exception when returning database cursors. How do you resolve this?
* **Answer:** Ensure database helper operations are called within try-catch blocks and verify that cursors are always closed in the `finally` block.

### 65. Scenario: What is the security risk of leaving `android:exported="true"` on a ContentProvider?
* **Answer:** Any application on the device can query, insert, update, or delete records from your private database unless you protect it with custom permissions.

### 66. Scenario: You want to listen for changes to a specific contact URI in the device's Contacts Provider. How?
* **Answer:** Register a `ContentObserver` bound to the contact URI via the `ContentResolver`.

### 67. Scenario: Your app crashes with `FileUriExposedException` when sharing a file. What is the cause and fix?
* **Answer:** The app shared a raw `file://` URI with another app, which is blocked in modern Android versions. Fix by using `FileProvider` to share a secure `content://` URI.

### 68. Scenario: Can multiple applications register the same authority string for their ContentProviders?
* **Answer:** No. Authority strings must be globally unique across all apps on the device; installing an app with a duplicate authority string will fail.

### 69. Scenario: How do you verify if your dynamic BroadcastReceiver is registered before unregistering it?
* **Answer:** Maintain a boolean flag in your Activity/Fragment indicating registration state, updating it on registration and unregistration.

### 70. Scenario: You want to capture a custom broadcast and return data back to the sender. How?
* **Answer:** Send the broadcast as an ordered broadcast using `sendOrderedBroadcast()`, and use `setResultData()` in the receiver to return the data.

### 71. Scenario: What is the threat of an implicit broadcast intent hijack?
* **Answer:** An attacker app registers a high-priority receiver matching your implicit broadcast action string, intercepting your data payload. Resolve by sending explicit broadcasts.

### 72. Scenario: Your ContentProvider runs CRUD operations on a SQLite database. How do you prevent SQL injection attacks?
* **Answer:** Use parameterized queries with selection arguments (`?` placeholders) instead of raw SQL string concatenation.

### 73. Scenario: Why does `ContentProvider.onCreate()` block app startup?
* **Answer:** Because it is executed synchronously on the main thread during process initialization. Keep database helper instantiations lightweight.

### 74. Scenario: You want to grant temporary read access to a content URI to another app, but only for 5 minutes. How?
* **Answer:** Launch the target app's Activity using an intent with flag `FLAG_GRANT_READ_URI_PERMISSION`. The permission is automatically revoked when the target Activity is closed.

### 75. Scenario: What is the difference between `query()` and `query()` with a cancellation signal?
* **Answer:** The query with a `CancellationSignal` parameter allows long-running database queries to be cancelled in real time if the client disconnects.

### 76. Scenario: How do you configure FileProvider paths?
* **Answer:** Define path mapping configurations (like `<files-path>`) in an XML resource file, linked to the `FileProvider` metadata tag in the manifest.

### 77. Scenario: An app needs to read images from the user's gallery in Android 13. Which permissions must it request?
* **Answer:** Request `android.permission.READ_MEDIA_IMAGES` instead of the legacy `READ_EXTERNAL_STORAGE` permission.

### 78. Scenario: You want to show a rationale message to a user before requesting permission. When should it be displayed?
* **Answer:** Show the message when `shouldShowRequestPermissionRationale()` returns true, indicating the user previously denied the permission.

### 79. Scenario: How do you handle cases where a user grants a runtime permission but then revokes it in system settings?
* **Answer:** The system automatically restarts the application process to apply the changes, so you must verify the permission status dynamically before accessing protected features.

### 80. Scenario: An app requests Bluetooth permissions in Android 12, but it crashes on launch. What did the developer forget?
* **Answer:** Declare `BLUETOOTH_SCAN` and `BLUETOOTH_CONNECT` permissions in the manifest, and request them at runtime before starting scans.

---

## 🏛️ Section 5: Architecture, ViewModel, SavedState (Scenarios 81-100)

### 81. Scenario: Your Activity rotates and the ViewModel's constructor is called again. Why?
* **Answer:** The developer instantiated the ViewModel directly using its constructor (e.g., `MyViewModel()`) instead of retrieving it via the `by viewModels()` delegate or `ViewModelProvider`.

### 82. Scenario: You want to retain search parameters when your app is killed in the background. What API should you use inside the ViewModel?
* **Answer:** Inject `SavedStateHandle` into the ViewModel constructor, and save the search parameters inside it using key-value keys.

### 83. Scenario: A ViewModel exposes a StateFlow UI state. The Activity collects it inside `lifecycleScope.launch`. Why does this waste resources?
* **Answer:** Because `lifecycleScope.launch` collects values even when the Activity is in the background, consuming CPU and memory. Collect the flow inside `repeatOnLifecycle(Lifecycle.State.STARTED)` instead.

### 84. Scenario: Explain how you implement the Repository pattern with local database and remote API data sources.
* **Answer:** The Repository abstracts data sources. It first queries local database cache, emits cached data, executes network queries, saves the results to the database, and emits the updated database state.

### 85. Scenario: You want to share user login state across all screens. Where should you store it in MVVM?
* **Answer:** Store it inside a repository class configured as an application-scoped singleton (`@Singleton`), injected into the ViewModels.

### 86. Scenario: An app crashes with `IllegalStateException: Cannot create an instance of MyViewModel`. What did you forget?
* **Answer:** The ViewModel has constructor arguments, but you did not provide a custom `ViewModelProvider.Factory` or annotate it with `@HiltViewModel`.

### 87. Scenario: You want to implement a search query with debounce to avoid spamming network APIs. How?
* **Answer:** Expose a flow from the text input, and use the `debounce(timeout)` operator on the flow before triggering network requests inside the coroutine scope.

### 88. Scenario: What is the risk of using a shared ViewModel between two fragments hosted in different Activities?
* **Answer:** It fails because the fragments have different host Activities. The `activityViewModels()` delegate scopes the ViewModel to its host Activity, so they will receive separate instances.

### 89. Scenario: Your database query in Room runs on the main thread, throwing an error. How do you resolve it?
* **Answer:** Annotate the DAO query function with `suspend` to run it asynchronously within a coroutine scope.

### 90. Scenario: How do you test a repository class that depends on a Room database?
* **Answer:** Use an in-memory database built via `Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)` in your test class.

### 91. Scenario: You want to expose database changes dynamically to your Compose layouts. How?
* **Answer:** Return a `Flow` from the DAO, and collect it inside your composable function using `collectAsStateWithLifecycle()`.

### 92. Scenario: What is the limit of data stored in a Room Entity?
* **Answer:** Room database size is capped by device storage space, but individual entity sizes should be kept minimal to prevent query lag.

### 93. Scenario: A developer manually updates Room schemas without updating version numbers. What happens?
* **Answer:** The database initialization fails and throws an `IllegalStateException` due to schema mismatch.

### 94. Scenario: How do you write a Room database migration to add a new column to an existing table?
* **Answer:** Define a `Migration` object, execute the raw `ALTER TABLE ADD COLUMN` SQL query inside its `migrate()` function, and add it to the database builder via `addMigrations()`.

### 95. Scenario: What is the benefit of Hilt over manual Dependency Injection?
* **Answer:** Hilt automates compile-time safe code generation, manages component lifecycles, and removes boilerplates like manual factories.

### 96. Scenario: You want to inject a Context reference into a helper class using Hilt. How?
* **Answer:** Use the `@ApplicationContext` or `@ActivityContext` annotation to inject the context reference safely.

### 97. Scenario: What is the purpose of `@Provides` annotation in Hilt?
**Answer:** It tells Hilt how to provide instances of classes we do not own (like Room database or Retrofit clients).

### 98. Scenario: Explain the difference between `@Provides` and `@Binds` annotations.
* **Answer:** `@Provides` is a function providing an instance. `@Binds` is an abstract function binding an interface to its concrete implementation, which is more efficient.

### 99. Scenario: Can you observe LiveData inside a Jetpack Compose layout?
* **Answer:** Yes, by calling `liveData.observeAsState()` to convert it into a state object Compose can observe.

### 100. Scenario: A developer implements bidirectional data flow in an Activity, letting the View update the model directly. Why does this break MVVM?
* **Answer:** It violates Unidirectional Data Flow (UDF), creating tight coupling and making state changes hard to track and debug. State changes must flow from the ViewModel down to the View.

---
---

# Android Interview Question Bank (Part 4: 50 Debugging Questions)

---

## 🛠️ Debugging Scenarios (Questions 1-50)

### 1. The Singleton Context Leak
* **Faulty Code:**
```kotlin
class ApiClient private constructor(private val context: Context) {
    companion object {
        private var instance: ApiClient? = null
        fun getInstance(context: Context): ApiClient {
            if (instance == null) {
                instance = ApiClient(context)
            }
            return instance!!
        }
    }
}
```
* **The Bug:** If initialized with an `Activity` context, `instance` retains the Activity indefinitely, causing a memory leak when the Activity rotates.
* **The Fix:**
```kotlin
class ApiClient private constructor(private val context: Context) {
    companion object {
        private var instance: ApiClient? = null
        fun getInstance(context: Context): ApiClient {
            if (instance == null) {
                instance = ApiClient(context.applicationContext)
            }
            return instance!!
        }
    }
}
```

---

### 2. Main Thread Database Query
* **Faulty Code:**
```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun getAllUsers(): List<User>
}
// Called inside Activity
val users = db.userDao().getAllUsers()
```
* **The Bug:** Executing synchronous database queries on the main thread throws an `IllegalStateException` and freezes the UI.
* **The Fix:**
```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    suspend fun getAllUsers(): List<User>
}
// Called inside Activity within coroutine scope
lifecycleScope.launch {
    val users = withContext(Dispatchers.IO) { db.userDao().getAllUsers() }
}
```

---

### 3. Fragment ViewBinding Memory Leak
* **Faulty Code:**
```kotlin
class DetailFragment : Fragment() {
    private lateinit var binding: FragmentDetailBinding

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?
    ): View {
        binding = FragmentDetailBinding.inflate(inflater, container, false)
        return binding.root
    }
}
```
* **The Bug:** The backing properties of ViewBinding retain references to layout views. Since the Fragment instance outlives its view when placed on the back stack, the view hierarchy leaks.
* **The Fix:**
```kotlin
class DetailFragment : Fragment() {
    private var _binding: FragmentDetailBinding? = null
    private val binding get() = _binding!!

    override fun onCreateView(
        inflater: LayoutInflater, container: ViewGroup?, savedInstanceState: Bundle?
    ): View {
        _binding = FragmentDetailBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null
    }
}
```

---

### 4. Bad AlertDialog Window Token
* **Faulty Code:**
```kotlin
val builder = AlertDialog.Builder(applicationContext)
builder.setTitle("Error")
builder.setMessage("Something went wrong.")
builder.show()
```
* **The Bug:** Throws a `WindowManager$BadTokenException` because the application context does not have a window token.
* **The Fix:**
```kotlin
val builder = AlertDialog.Builder(this) // Pass the active Activity Context
builder.setTitle("Error")
builder.setMessage("Something went wrong.")
builder.show()
```

---

### 5. Parameterized Fragment Constructor Crash
* **Faulty Code:**
```kotlin
class UserFragment(private val userId: String) : Fragment() {
    // ...
}
// Launching
val frag = UserFragment("123")
```
* **The Bug:** When the OS recreates the Fragment on rotation, it calls the default parameterless constructor via reflection, crashing the app.
* **The Fix:**
```kotlin
class UserFragment : Fragment() {
    companion object {
        private const val ARG_USER_ID = "arg_user_id"
        fun newInstance(userId: String): UserFragment {
            return UserFragment().apply {
                arguments = Bundle().apply { putString(ARG_USER_ID, userId) }
            }
        }
    }
}
```

---

### 6. Unregistered Broadcast Receiver Leak
* **Faulty Code:**
```kotlin
class MyActivity : AppCompatActivity() {
    private val receiver = MyReceiver()

    override fun onStart() {
        super.onStart()
        registerReceiver(receiver, IntentFilter("MY_ACTION"))
    }
}
```
* **The Bug:** The dynamic BroadcastReceiver is registered but never unregistered, causing a memory leak when the Activity is destroyed.
* **The Fix:**
```kotlin
class MyActivity : AppCompatActivity() {
    private val receiver = MyReceiver()

    override fun onStart() {
        super.onStart()
        registerReceiver(receiver, IntentFilter("MY_ACTION"))
    }

    override fun onStop() {
        super.onStop()
        unregisterReceiver(receiver)
    }
}
```

---

### 7. FileUriExposedException on Share
* **Faulty Code:**
```kotlin
val file = File(filesDir, "image.jpg")
val intent = Intent(Intent.ACTION_SEND).apply {
    type = "image/jpg"
    putExtra(Intent.EXTRA_STREAM, Uri.fromFile(file))
}
startActivity(intent)
```
* **The Bug:** Exposes raw `file://` URIs, throwing `FileUriExposedException` in Android 7.0+.
* **The Fix:**
```kotlin
val file = File(filesDir, "image.jpg")
val uri = FileProvider.getUriForFile(this, "$packageName.fileprovider", file)
val intent = Intent(Intent.ACTION_SEND).apply {
    type = "image/jpg"
    putExtra(Intent.EXTRA_STREAM, uri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
startActivity(intent)
```

---

### 8. LiveData Duplicate Observers in Fragment
* **Faulty Code:**
```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    viewModel.data.observe(this) { data ->
        // Update UI
    }
}
```
* **The Bug:** Passing `this` (the Fragment instance) registers duplicate observers when the Fragment's view is recreated (e.g., returning from the back stack), causing duplicate updates.
* **The Fix:**
```kotlin
override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
    super.onViewCreated(view, savedInstanceState)
    viewModel.data.observe(viewLifecycleOwner) { data ->
        // Update UI
    }
}
```

---

### 9. Handler Posting Delayed UI Tasks Leaks Context
* **Faulty Code:**
```kotlin
class MyActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        handler.postDelayed({
            updateUI()
        }, 60000)
    }
}
```
* **The Bug:** The anonymous runnable holds an implicit reference to the Activity. If the user exits the Activity, the handler holds it in queue, leaking the Activity until the runnable executes.
* **The Fix:**
```kotlin
class MyActivity : AppCompatActivity() {
    private val handler = Handler(Looper.getMainLooper())
    private var runnable: Runnable? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        runnable = Runnable { updateUI() }
        handler.postDelayed(runnable!!, 60000)
    }

    override fun onDestroy() {
        super.onDestroy()
        runnable?.let { handler.removeCallbacks(it) }
    }
}
```

---

### 10. Foreground Service Crash (No Notification)
* **Faulty Code:**
```kotlin
class MyService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        // Run heavy task
        return START_STICKY
    }
}
// Started via context.startForegroundService(intent)
```
* **The Bug:** Crashes with an OS exception because the service did not execute `startForeground()` and show a notification within the required 5 seconds.
* **The Fix:**
```kotlin
class MyService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val notification = createNotification()
        startForeground(1, notification) // Must call immediately
        // Run heavy task
        return START_STICKY
    }
}
```

---

### 11. Stale Intent in `onNewIntent`
* **Faulty Code:**
```kotlin
class MyActivity : AppCompatActivity() {
    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        // Read extras
        val data = getIntent().getStringExtra("KEY")
    }
}
```
* **The Bug:** `getIntent()` returns the original intent that launched the Activity, not the new intent containing updated extras.
* **The Fix:**
```kotlin
class MyActivity : AppCompatActivity() {
    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        setIntent(intent) // Update the intent reference first
        val data = getIntent().getStringExtra("KEY")
    }
}
```

---

### 12. Non-Thread-Safe ContentProvider Queries
* **Faulty Code:**
```kotlin
class MyProvider : ContentProvider() {
    private var lastQueryTime = 0L

    override fun query(...): Cursor? {
        lastQueryTime = System.currentTimeMillis() // Modifies class level variable
        return db.query(...)
    }
}
```
* **The Bug:** ContentProvider queries are processed on binder thread pools. Modifying shared class-level variables without synchronization creates race conditions.
* **The Fix:**
```kotlin
class MyProvider : ContentProvider() {
    @Volatile
    private var lastQueryTime = 0L

    override fun query(...): Cursor? {
        synchronized(this) {
            lastQueryTime = System.currentTimeMillis()
        }
        return db.query(...)
    }
}
```

---

### 13. StateFlow Collection Consuming Background Battery
* **Faulty Code:**
```kotlin
lifecycleScope.launch {
    viewModel.uiState.collect { state ->
        // Update UI
    }
}
```
* **The Bug:** The coroutine collects updates even when the app is in the background, consuming CPU resources and battery.
* **The Fix:**
```kotlin
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.uiState.collect { state ->
            // Update UI
        }
    }
}
```

---

### 14. Static Context Variable Leak
* **Faulty Code:**
```kotlin
object ImageCache {
    var cachedContext: Context? = null
}
// Set inside Activity
ImageCache.cachedContext = this
```
* **The Bug:** The static object retains the Activity context, preventing garbage collection and leaking the entire view hierarchy.
* **The Fix:**
```kotlin
object ImageCache {
    private var contextRef: WeakReference<Context>? = null

    fun setContext(context: Context) {
        contextRef = WeakReference(context.applicationContext) // Use WeakReference & App Context
    }
}
```

---

### 15. Shared ViewModel Initialization Fail
* **Faulty Code:**
```kotlin
class FragmentB : Fragment() {
    private val viewModel: SharedViewModel by viewModels() // Wrong delegate
}
```
* **The Bug:** `by viewModels()` scopes the ViewModel to `FragmentB` itself. `FragmentA` will receive a separate instance, preventing communication.
* **The Fix:**
```kotlin
class FragmentB : Fragment() {
    private val viewModel: SharedViewModel by activityViewModels() // Scope to host Activity
}
```

---

### 16. Debounce Network Spam
* **Faulty Code:**
```kotlin
searchQueryFlow.onEach { query ->
    triggerApiCall(query) // Spams network on every keyboard type
}.launchIn(lifecycleScope)
```
* **The Bug:** Every keystroke triggers a network request, spamming the server and lagging the UI.
* **The Fix:**
```kotlin
searchQueryFlow
    .debounce(300) // Wait for user pause
    .distinctUntilChanged() // Ignore duplicate values
    .onEach { query ->
        triggerApiCall(query)
    }.launchIn(lifecycleScope)
```

---

### 17. Safe Args Null Pointer Exception
* **Faulty Code:**
```kotlin
val userId = arguments?.getString("userId")!! // Risk of null crashes
```
* **The Bug:** If arguments are missing or keys are misspelled, accessing it with double bangs throws a NullPointerException.
* **The Fix:**
```kotlin
val args: DetailFragmentArgs by navArgs()
val userId = args.userId // Type-safe and verified at compile time
```

---

### 18. Service on Main Thread ANR
* **Faulty Code:**
```kotlin
class NetworkService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        executeHeavyUpload() // Synchronous loop
        return START_NOT_STICKY
    }
}
```
* **The Bug:** The service runs on the Main Thread, so executing heavy loops blocks UI rendering, causing an ANR.
* **The Fix:**
```kotlin
class NetworkService : Service() {
    private val serviceScope = CoroutineScope(Dispatchers.IO + SupervisorJob())

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        serviceScope.launch {
            executeHeavyUpload()
        }
        return START_NOT_STICKY
    }

    override fun onDestroy() {
        super.onDestroy()
        serviceScope.cancel()
    }
}
```

---

### 19. Room MainThreadAccessException
* **Faulty Code:**
```kotlin
val db = Room.databaseBuilder(applicationContext, AppDatabase::class.java, "db").build()
val data = db.userDao().getData() // Direct access
```
* **The Bug:** Throws `Cannot access database on the main thread since it may potentially lock the UI for a long period of time`.
* **The Fix:**
```kotlin
// DAO function marked as suspend
@Query("SELECT * FROM data")
suspend fun getData(): List<DataEntity>

// Called within Coroutine scope on I/O dispatcher
lifecycleScope.launch {
    val data = withContext(Dispatchers.IO) { db.userDao().getData() }
}
```

---

### 20. Implicit Broadcast Blocked
* **Faulty Code:**
```xml
<receiver android:name=".MyReceiver">
    <intent-filter>
        <action android:name="android.net.conn.CONNECTIVITY_CHANGE" />
    </intent-filter>
</receiver>
```
* **The Bug:** In Android 8.0+, registering connectivity receivers statically in the manifest is blocked, so the receiver will not fire.
* **The Fix:**
```kotlin
// Register dynamically in code
val filter = IntentFilter(ConnectivityManager.CONNECTIVITY_ACTION)
registerReceiver(myReceiver, filter)
```

---

### 21. SharedPreferences Main Thread Block
* **Faulty Code:**
```kotlin
val editor = sharedPrefs.edit()
editor.putString("token", token)
editor.commit() // Synchronous I/O
```
* **The Bug:** `commit()` runs a synchronous disk write on the main thread, which can block the thread and cause UI stuttering or ANRs.
* **The Fix:**
```kotlin
val editor = sharedPrefs.edit()
editor.putString("token", token)
editor.apply() // Asynchronous disk write
```

---

### 22. Infinite Recomposition Loop in Compose
* **Faulty Code:**
```kotlin
@Composable
fun Counter() {
    var count = 0
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```
* **The Bug:** The variable is reset to 0 on every recomposition. The state is lost immediately.
* **The Fix:**
```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }
    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

---

### 23. Stale State Flow Collections
* **Faulty Code:**
```kotlin
viewModelScope.launch {
    repository.observeData().collect { data ->
        _uiState.value = data // Modifies state flow
    }
}
```
* **The Bug:** If the repository emits data constantly, it stays active, wasting CPU cycles even if the UI is not observing it.
* **The Fix:**
```kotlin
val uiState: StateFlow<UiState> = repository.observeData()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(5000), // Stop collecting 5s after UI disconnects
        initialValue = UiState.Loading
    )
```

---

### 24. LiveData postValue Race Condition
* **Faulty Code:**
```kotlin
liveData.postValue("A")
val value = liveData.value // Reads value immediately
```
* **The Bug:** `postValue` executes asynchronously on the main thread loop, so reading `value` immediately returns the old value, not "A".
* **The Fix:**
```kotlin
// If on main thread, set value directly
liveData.value = "A"
val value = liveData.value
```

---

### 25. Custom View Binding Leak
* **Faulty Code:**
```kotlin
class CustomView @JvmOverloads constructor(
    context: Context, attrs: AttributeSet? = null
) : LinearLayout(context, attrs) {
    private val binding = ViewCustomBinding.inflate(LayoutInflater.from(context), this)
}
```
* **The Bug:** Custom views do not have a separate lifecycle like Fragments, so ViewBinding leaks are rare unless you store views statically, but using ViewBinding directly in custom views is safe as long as the View itself is garbage collected.
* **The Fix:** (Code is correct, but ensure we do not store static instances of the custom View).

---

### 26. Coroutine Scope Job Cancellation
* **Faulty Code:**
```kotlin
val scope = CoroutineScope(Job())
scope.launch {
    throw Exception("Fail")
}
scope.launch {
    // This will never run because the parent Job was cancelled by the sibling's failure
}
```
* **The Bug:** Sibling coroutines are cancelled when one child fails inside a standard Job scope.
* **The Fix:**
```kotlin
val scope = CoroutineScope(SupervisorJob()) // Use SupervisorJob
scope.launch {
    throw Exception("Fail")
}
scope.launch {
    // Runs successfully
}
```

---

### 27. missing exported flag crash (API 31+)
* **Faulty Code:**
```xml
<activity android:name=".ShareActivity">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
    </intent-filter>
</activity>
```
* **The Bug:** In Android 12+, activities with intent filters that do not explicitly set the exported flag will cause installation to fail with a crash.
* **The Fix:**
```xml
<activity android:name=".ShareActivity" android:exported="true">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
    </intent-filter>
</activity>
```

---

### 28. FileProvider Authority Mismatch
* **Faulty Code:**
```xml
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="com.example.app.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true"/>
```
* **The Bug:** If the authority string passed in `getUriForFile()` doesn't match the manifest, the app crashes with `IllegalArgumentException: Failed to find configured root that contains`.
* **The Fix:**
```kotlin
val uri = FileProvider.getUriForFile(
    context, 
    "com.example.app.fileprovider", // Must match authorities attribute exactly
    file
)
```

---

### 29. PendingIntent Immutable Flag Missing (API 31+)
* **Faulty Code:**
```kotlin
val pendingIntent = PendingIntent.getActivity(
    context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT
)
```
* **The Bug:** In Android 12+, PendingIntents must explicitly set either `FLAG_MUTABLE` or `FLAG_IMMUTABLE`, otherwise a crash is thrown.
* **The Fix:**
```kotlin
val pendingIntent = PendingIntent.getActivity(
    context, 0, intent, PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
)
```

---

### 30. DialogFragment onSaveInstanceState Crash
* **Faulty Code:**
```kotlin
fun showDialog() {
    dialogFragment.show(supportFragmentManager, "tag")
}
// Called inside background callback after Activity is stopped
```
* **The Bug:** Shows the dialog after state saving, throwing `IllegalStateException: Can not perform this action after onSaveInstanceState`.
* **The Fix:**
```kotlin
fun showDialog() {
    supportFragmentManager.beginTransaction()
        .add(dialogFragment, "tag")
        .commitAllowingStateLoss() // Bypasses state check
}
```

---

### 31. WorkManager Worker Thread block
* **Faulty Code:**
```kotlin
class UploadWorker(context: Context, params: WorkerParameters) : Worker(context, params) {
    override fun doWork(): Result {
        executeHeavyUpload() // blocks background thread pool
        return Result.success()
    }
}
```
* **The Bug:** Standard `Worker` runs tasks synchronously on its thread pool, which can block scheduling of other background tasks.
* **The Fix:**
```kotlin
class UploadWorker(context: Context, params: WorkerParameters) : CoroutineWorker(context, params) {
    override suspend fun doWork(): Result = withContext(Dispatchers.IO) {
        executeHeavyUpload() // Runs safely on I/O dispatcher
        Result.success()
    }
}
```

---

### 32. SavedStateHandle non-serializable crash
* **Faulty Code:**
```kotlin
class MyViewModel(private val state: SavedStateHandle) : ViewModel() {
    init {
        state["user"] = UserClass() // Custom complex class
    }
}
```
* **The Bug:** Storing non-serializable objects inside `SavedStateHandle` causes a crash on process death serialization.
* **The Fix:** Ensure the class implements `Parcelable` or `Serializable`.
```kotlin
@Parcelize
data class UserClass(val name: String) : Parcelable
```

---

### 33. ContentObserver Lifecycle Leak
* **Faulty Code:**
```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        contentResolver.registerContentObserver(uri, true, myObserver)
    }
}
```
* **The Bug:** The observer remains registered in the system, leaking the Activity when closed.
* **The Fix:**
```kotlin
class MyActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        contentResolver.registerContentObserver(uri, true, myObserver)
    }

    override fun onDestroy() {
        super.onDestroy()
        contentResolver.unregisterContentObserver(myObserver) // Clean up
    }
}
```

---

### 34. Retrofit Response Body Double Read
* **Faulty Code:**
```kotlin
val response = api.getData()
Log.d("Response", response.body()?.string() ?: "")
val data = gson.fromJson(response.body()?.string(), Data::class.java)
```
* **The Bug:** Calling `response.body()?.string()` reads the buffer into memory and closes the stream, so the second call returns an empty string or throws an exception.
* **The Fix:**
```kotlin
val response = api.getData()
val bodyString = response.body()?.string() ?: "" // Read once
Log.d("Response", bodyString)
val data = gson.fromJson(bodyString, Data::class.java)
```

---

### 35. ViewStub Inflation Null Pointer Exception
* **Faulty Code:**
```kotlin
val viewStub = findViewById<ViewStub>(R.id.view_stub)
viewStub.inflate()
viewStub.setVisibility(View.VISIBLE) // Crashes
```
* **The Bug:** Once `inflate()` is called, the `ViewStub` is removed from its parent container, so subsequent calls on its reference throw a NullPointerException.
* **The Fix:**
```kotlin
val viewStub = findViewById<ViewStub>(R.id.view_stub)
val inflatedView = viewStub.inflate() // Hold reference to the inflated View
inflatedView.visibility = View.VISIBLE
```

---

### 36. Fragment onBackPressed Dispatcher Leak
* **Faulty Code:**
```kotlin
class MyFragment : Fragment() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        requireActivity().onBackPressedDispatcher.addCallback(this) {
            // Handle back press
        }
    }
}
```
* **The Bug:** The callback holds a reference to the Fragment. Passing `this` (the Fragment instance) can cause memory leaks on view destruction.
* **The Fix:**
```kotlin
class MyFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        requireActivity().onBackPressedDispatcher.addCallback(viewLifecycleOwner) { // Pass viewLifecycleOwner
            // Handle back press
        }
    }
}
```

---

### 37. Room Migration IllegalStateException
* **Faulty Code:**
```kotlin
val db = Room.databaseBuilder(context, AppDatabase::class.java, "db")
    .fallbackToDestructiveMigration() // Danger: Destroys user data
    .build()
```
* **The Bug:** Using fallback to destructive migration deletes user databases rather than migrating them safely when the schema updates.
* **The Fix:** Define clean migrations using version updates.
```kotlin
val db = Room.databaseBuilder(context, AppDatabase::class.java, "db")
    .addMigrations(MIGRATION_1_2)
    .build()
```

---

### 38. Compose DisposableEffect Cleanup Missing
* **Faulty Code:**
```kotlin
@Composable
fun SensorScreen(manager: SensorManager) {
    DisposableEffect(Unit) {
        manager.registerListener(listener, sensor, delay)
        onDispose { } // Empty cleanup
    }
}
```
* **The Bug:** The sensor listener remains registered in the system, leaking memory and draining the battery when leaving the screen.
* **The Fix:**
```kotlin
@Composable
fun SensorScreen(manager: SensorManager) {
    DisposableEffect(Unit) {
        manager.registerListener(listener, sensor, delay)
        onDispose {
            manager.unregisterListener(listener) // Unregister sensor
        }
    }
}
```

---

### 39. Hilt Scope Mismatch
* **Faulty Code:**
```kotlin
@Module
@InstallIn(ActivityComponent::class)
object DataModule {
    @Provides
    @Singleton // Scope mismatch
    fun provideData() = DataClass()
}
```
* **The Bug:** Compiling will fail because `@Singleton` scope belongs to `SingletonComponent`, but the module is installed in `ActivityComponent`.
* **The Fix:**
```kotlin
@Module
@InstallIn(ActivityComponent::class)
object DataModule {
    @Provides
    @ActivityScoped // Match scope to component
    fun provideData() = DataClass()
}
```

---

### 40. Activity Result API contract instantiation leak
* **Faulty Code:**
```kotlin
fun triggerRequest() {
    val launcher = registerForActivityResult(ActivityResultContracts.RequestPermission()) { }
    launcher.launch(Manifest.permission.CAMERA)
}
```
* **The Bug:** Throws an `IllegalStateException` because `registerForActivityResult` must be called during initialization (before the Activity transitions to the STARTED state), not inside onClick callbacks.
* **The Fix:**
```kotlin
// Define at class level
private val cameraLauncher = registerForActivityResult(ActivityResultContracts.RequestPermission()) { granted ->
    // Handle result
}

fun triggerRequest() {
    cameraLauncher.launch(Manifest.permission.CAMERA) // Launch in click callback
}
```

---

### 41. RxJava Undisposed Subscription Leak
* **Faulty Code:**
```kotlin
api.getData()
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { data ->
        updateUI(data) // Implicit Activity reference
    }
```
* **The Bug:** The subscription stays active if the network call takes a long time. If the Activity is destroyed, it leaks.
* **The Fix:**
```kotlin
private val disposables = CompositeDisposable()

api.getData()
    .subscribeOn(Schedulers.io())
    .observeOn(AndroidSchedulers.mainThread())
    .subscribe { data -> updateUI(data) }
    .let { disposables.add(it) }

override fun onDestroy() {
    super.onDestroy()
    disposables.clear() // Dispose all active streams
}
```

---

### 42. Glide Image Load Target context leak
* **Faulty Code:**
```kotlin
Glide.with(applicationContext) // Wrong context for views
    .load(url)
    .into(imageView)
```
* **The Bug:** Using `applicationContext` restricts Glide from binding to the Activity's lifecycle, continuing image downloads in the background even if the Activity closes.
* **The Fix:**
```kotlin
Glide.with(this) // Pass Activity Context
    .load(url)
    .into(imageView)
```

---

### 43. Activity Context injected in Singleton Module Hilt
* **Faulty Code:**
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    fun provideClient(context: Context) = MyClient(context) // Activity Context risk
}
```
* **The Bug:** If Hilt binds an Activity context, injecting it into a singleton object creates a permanent memory leak.
* **The Fix:**
```kotlin
@Module
@InstallIn(SingletonComponent::class)
object NetworkModule {
    @Provides
    fun provideClient(@ApplicationContext context: Context) = MyClient(context) // Application Context
}
```

---

### 44. ContentResolver query Projection Leak
* **Faulty Code:**
```kotlin
val cursor = contentResolver.query(uri, null, null, null, null) // Null projection
```
* **The Bug:** Querying with a null projection returns all columns from the provider database, which consumes memory and slows down query performance.
* **The Fix:**
```kotlin
val projection = arrayOf(BaseColumns._ID, ContactsContract.Contacts.DISPLAY_NAME)
val cursor = contentResolver.query(uri, projection, null, null, null) // Select specific columns
```

---

### 45. Broadcast Receiver security leak (manifest)
* **Faulty Code:**
```xml
<receiver android:name=".MyReceiver">
    <intent-filter>
        <action android:name="com.example.CUSTOM_ACTION" />
    </intent-filter>
</receiver>
```
* **The Bug:** The receiver does not declare `android:exported="false"`, allowing external apps to send malicious broadcasts to trigger internal events.
* **The Fix:**
```xml
<receiver android:name=".MyReceiver" android:exported="false">
    <intent-filter>
        <action android:name="com.example.CUSTOM_ACTION" />
    </intent-filter>
</receiver>
```

---

### 46. Room Coroutine Dispatcher redundancy
* **Faulty Code:**
```kotlin
lifecycleScope.launch(Dispatchers.IO) { // Redundant dispatcher switch
    val users = db.userDao().getAllUsers() // userDao function is already suspend
}
```
* **The Bug:** Room DAOs marked with `suspend` handle dispatcher routing internally using a custom thread pool. Manually wrapping them in `Dispatchers.IO` is redundant.
* **The Fix:**
```kotlin
lifecycleScope.launch { // Launch on main thread directly
    val users = db.userDao().getAllUsers() // Async execution is safe
    updateUI(users)
}
```

---

### 47. StateFlow State lost on Recomposition
* **Faulty Code:**
```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state = viewModel.uiState.collectAsState() // Not lifecycle-aware
}
```
* **The Bug:** `collectAsState()` continues collecting updates when the app is backgrounded, wasting battery.
* **The Fix:**
```kotlin
@Composable
fun MyScreen(viewModel: MyViewModel) {
    val state by viewModel.uiState.collectAsStateWithLifecycle() // Lifecycle-aware collection
}
```

---

### 48. Fragment Transaction Commit post-activity stopped
* **Faulty Code:**
```kotlin
override fun onStop() {
    super.onStop()
    parentFragmentManager.commit {
        replace(R.id.container, MyFragment())
    }
}
```
* **The Bug:** Committing transactions inside `onStop` throws `IllegalStateException` because the Activity's state has already been saved.
* **The Fix:**
```kotlin
override fun onStop() {
    super.onStop()
    parentFragmentManager.commitAllowingStateLoss() {
        replace(R.id.container, MyFragment())
    }
}
```

---

### 49. Activity taskAffinity back navigation loop
* **Faulty Code:**
```xml
<activity android:name=".DetailActivity"
    android:taskAffinity="com.example.customtask"
    android:launchMode="singleTask" />
```
* **The Bug:** The details screen runs in a separate task. When the user clicks back, they are returned to the home launcher screen instead of the previous screen in the main app.
* **The Fix:** Remove the custom `taskAffinity` tag to run the details screen in the main task stack context.

---

### 50. Missing Hilt Qualifier
* **Faulty Code:**
```kotlin
class MyClass @Inject constructor(
    private val apiService: ApiService // Interface has 2 implementations (mock and real)
)
```
* **The Bug:** Compiling will fail with `[Dagger/DependencyCycle] ApiService cannot be provided without an @Provides-annotated method` because Hilt doesn't know which implementation to inject.
* **The Fix:** Define qualifiers (`@MockApi` and `@RealApi`) to differentiate bindings.
```kotlin
class MyClass @Inject constructor(
    @RealApi private val apiService: ApiService
)
```



