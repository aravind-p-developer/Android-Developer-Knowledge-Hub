# Part 1: Foundations

---

# Chapter 1: Android Architecture & Components Overview

## Concept
The Android operating system is a multi-layered, Linux-based software stack designed for mobile devices. To understand Android development, one must first comprehend the architectural layers that support the execution of an application, followed by the core building blocks—the four **App Components**—that form the entry points for any Android application.

### The Android Software Stack
The Android architecture is divided into five distinct layers:
1. **Linux Kernel:** The foundation of the stack. It handles low-level memory management, process management, threading, security, permissions, and hardware drivers (e.g., display, camera, Wi-Fi, audio).
2. **Hardware Abstraction Layer (HAL):** Provides standard interfaces that expose device hardware capabilities to the higher-level Java API framework. HAL consists of multiple library modules, each implementing an interface for a specific type of hardware component (e.g., camera, bluetooth).
3. **Android Runtime (ART) & Native C/C++ Libraries:**
   * **Native Libraries:** Core components such as WebKit, OpenGL ES, SQLite, Media Framework, and Libc run in native code, providing high-performance capabilities.
   * **Android Runtime (ART):** The engine that executes DEX (Dalvik Executable) bytecode. Each app runs in its own process with its own instance of ART. ART handles compilation (AOT & JIT), garbage collection, and debugging support.
4. **Java API Framework:** The entire feature set of the Android OS is made available through APIs written in Java/Kotlin. This includes UI components (Views), Content Providers, resource managers, and system managers (NotificationManager, ActivityManager).
5. **System Apps:** The outermost layer consisting of pre-installed apps (e.g., Contacts, Dialer, Browser) and user-installed apps.

### The 4 Core App Components
An Android application is not structured with a single `main()` entry point like a standard Java/Kotlin console application. Instead, it is composed of four distinct components, each serving a unique purpose and having its own lifecycle managed by the system:
1. **Activity:** Represents a single screen with a User Interface (UI). It facilitates user interaction.
2. **Service:** Runs in the background to perform long-running operations (e.g., playing music, downloading files) without providing a user interface.
3. **BroadcastReceiver:** A component that enables the system or other apps to deliver events to the app (e.g., battery low, boot completed, SMS received).
4. **ContentProvider:** Manages access to a central repository of data, allowing sharing of data securely between apps or within the same app.

Every component must be declared in the application's `AndroidManifest.xml` file, which serves as the manifest of the application's structure, permissions, and configurations.

---

## Why It Exists
Android's architecture and component model was designed to solve several fundamental constraints of mobile devices in the late 2000s:
* **Severe Resource Constraints:** Mobile devices had limited RAM, weak CPUs, and small batteries. Running a monolithic application that handles everything was unacceptable.
* **Component-Based Modularity:** Android wanted to allow apps to leverage components of other apps. For example, if a shopping app needs to take a photo, it does not need to write camera code; it can simply invoke the Camera app's Photo Activity.
* **Sandboxing & Security:** To protect the device, each app runs in its own Linux process with a unique User ID (UID). This creates an isolated sandbox, preventing one app from accessing another app's memory or resources without explicit permission.
* **Process Priority & Resource Reclaiming:** Mobile users constantly switch between apps. The system needs a way to kill background apps to free up memory for the foreground app. The 4-component architecture allows the OS to understand exactly what state each app is in (e.g., is it showing a UI, running a background service, or doing nothing?) and kill processes strategically.

---

## Internal Working

### App Startup Sequence (From Boot to Launch)
When a device is powered on, the following boot sequence takes place:
1. **Linux Kernel Init:** The kernel initializes hardware drivers, sets up memory protections, and starts the `init` process (the root process of user space).
2. **`init` Process:** Starts system daemons and parses the `init.rc` configuration script to launch the **Zygote** process.
3. **Zygote Process:** This is a warm-started process containing a preloaded JVM, core Java libraries, and pre-warmed system resources. Instead of starting a new virtual machine from scratch for every app (which is extremely slow), the OS forks the Zygote process.
4. **System Server:** Zygote forks the `System Server`, which starts core system services like `ActivityManagerService` (AMS), `PackageManagerService` (PMS), and `WindowManagerService` (WMS).
5. **Launcher App:** Once System Server finishes initialization, it broadcasts an intent to start the default launcher app (Home Screen).

When a user clicks an app icon on the Launcher:
1. **Request to AMS:** Launcher calls `startActivity()` which sends an IPC (Binder) request to `ActivityManagerService` (AMS).
2. **Process Verification:** AMS checks if a process for the target app already exists.
3. **Zygote Forking:** If the process does not exist, AMS requests Zygote to fork a new process.
4. **Process Entry (`ActivityThread`):** The new process begins execution in `ActivityThread.main()`. A `Looper` is prepared for the main thread, and `ActivityThread` binds the application by calling `attach()`.
5. **Component Initialization:** AMS instantiates the `Application` class, executes `Application.onCreate()`, and then launches the requested `Activity` component.

```
+------------------+       forks       +------------------+       starts       +-----------------------+
|  Linux Kernel    | ----------------> |  init Process    | -----------------> |    Zygote Process     |
+------------------+                   +------------------+                    +-----------------------+
                                                                                           |
                                                                                           | forks
                                                                                           v
+------------------+       starts      +------------------+       forks        +-----------------------+
|   Launcher App   | <---------------- |  System Server   | <----------------- |  App Process (ART)    |
+------------------+                   +------------------+                    +-----------------------+
        |                                                                                  |
        | Click App Icon                                                                   | runs
        v                                                                                  v
+-----------------------+              Start Application                       +-----------------------+
| ActivityManagerServ.  | ---------------------------------------------------> | ActivityThread.main() |
+-----------------------+                                                      +-----------------------+
```

### APK File Structure
An Android Package (APK) is a ZIP archive containing:
* `classes.dex`: Compiled Java/Kotlin code converted into Dalvik Executable (DEX) bytecode.
* `resources.arsc`: Precompiled resources (layout IDs, strings, colors).
* `res/`: Directory containing uncompiled resources (drawables, raw XML layouts).
* `assets/`: Directory for raw asset files (fonts, databases, JSON files).
* `lib/`: Compiled native libraries (.so files) grouped by CPU architecture (armeabi-v7a, arm64-v8a, x86).
* `AndroidManifest.xml`: Binary-encoded manifest declaring app identity, components, permissions, and hardware requirements.
* `META-INF/`: Cryptographic signatures and certificate data proving APK integrity.

### The Linux Sandbox & Process Priorities
Android runs each app inside a separate process, protected by Linux user-level permissions. Each app is assigned a unique UID. 

Under low memory conditions, Android's **Low Memory Killer (LMK)** daemon kills processes to free up RAM. The priority of processes is defined by the components running inside them:
1. **Foreground Process (Highest Priority - ADJ <= 0):**
   * Process hosting an Activity the user is interacting with (onResume).
   * Process hosting a Service bound to a foreground activity.
   * Process hosting a Foreground Service (showing a notification).
2. **Visible Process (ADJ 100-200):**
   * Process hosting an Activity that is visible but not in focus (onPause, e.g., partially covered by a dialog).
3. **Service Process (ADJ 500):**
   * Process hosting a background Service started with `startService()`.
4. **Background / Cached Process (ADJ 800+):**
   * Process hosting Activities that are completely invisible (onStop). These are kept in a LIFO cache so they can resume quickly, but are the first to be killed when RAM is low.
5. **Empty Process (Lowest Priority):**
   * Process with no active components. Kept only for caching purposes to speed up warm starts. Killed immediately when memory pressure occurs.

---

## Lifecycle / Flow

### Process Lifecycle State Transitions
The diagram below shows how the system transitions an application process based on user interaction and memory availability.

```
       +------------------+
       |   Empty Process  | <---------------+
       +------------------+                 |
                 |                          |
                 | App Launches             |
                 v                          |
       +------------------+                 |
       |  Cached/Bgd Proc | <---------------+
       +------------------+                 |
                 |                          |
                 | Component Starts         |
                 v                          |
       +------------------+                 | LMK Kills Process
       |  Service Process | <---------------+ (Low Memory Killer)
       +------------------+                 |
                 |                          |
                 | Component Visible        |
                 v                          |
       +------------------+                 |
       |  Visible Process | <---------------+
       +------------------+                 |
                 |                          |
                 | Component Gains Focus    |
                 v                          |
       +------------------+                 |
       | Foreground Proc. | ----------------+
       +------------------+
```

---

## Real World Example
Let's look at how the 4 app components work together in a production scenario, such as a **Fitness Tracking Application**:
* **Activity:** `WorkoutActivity` provides the UI showing a map, elapsed time, and heart rate.
* **Service:** `LocationTrackingService` runs as a Foreground Service to collect GPS coordinates in the background while the user runs, even if the screen is off or another app is open.
* **BroadcastReceiver:** `BootReceiver` listens to `ACTION_BOOT_COMPLETED` to reschedule daily workout reminders if the device restarts.
* **ContentProvider:** `HealthDataProvider` shares steps and calories data with Google Fit or Samsung Health securely.

---

## Common Mistakes
1. **Running Heavy Initializations in Manifest-Declared Components:** Instantiating database connections, network clients, or parsing JSON inside the constructor of an Activity or Service blocks the UI thread, causing slow startup times.
2. **Treating App Components as Regular Java Classes:** Never instantiate components manually (e.g., `val activity = MainActivity()`). Components must be instantiated by the OS via Intents so the system can bind their contexts and initialize their lifecycles.
3. **Hardcoding Component Process Names:** Declaring components to run in separate processes (`android:process=":remote"`) without understanding IPC overhead. Multi-process architectures should be used only for massive apps or services requiring isolation (e.g., music playback).
4. **Incorrect `exported` Manifest Flag:** Leaving components exported (`android:exported="true"`) when they are meant only for internal app use. This creates a severe security vulnerability.

---

## Memory Leak / Performance Concerns
* **Over-allocation of Components in Separate Processes:** Every process has a baseline memory overhead of ~15-30MB due to ART VM initialization. Unnecessarily splitting components into multiple processes wastes resources and drains battery.
* **Large manifest files and resource allocations:** Unused `<queries>` tags or excessively complex Intent Filters slow down the `PackageManagerService` parsing duration during app install and startup.
* **Component Instantiation Overhead:** Starting Services or sending Broadcasts repeatedly in tight loops stresses the Binder IPC system, causing garbage collection pauses and UI stuttering.

---

## Interview Questions

### Beginner
1. **What are the four main components of an Android application?**
   * *Answer:* Activity, Service, BroadcastReceiver, and ContentProvider.
2. **What is the purpose of the `AndroidManifest.xml` file?**
   * *Answer:* It declares the application package, components (Activities, Services, etc.), permissions required, permissions defined, hardware requirements, and launcher entry points.
3. **What is the difference between ART and Dalvik?**
   * *Answer:* Dalvik is the legacy runtime that used Just-In-Time (JIT) compilation (compiling code during execution). ART is the modern runtime introduced in Android 5.0 that uses a combination of Ahead-of-Time (AOT) compilation (compiling dex during installation) and JIT, which improves runtime performance and reduces battery consumption.

### Intermediate
1. **Explain the process lifecycle hierarchy in Android. How does the Low Memory Killer decide which app to kill?**
   * *Answer:* Android groups processes into five categories based on active components: Foreground, Visible, Service, Background/Cached, and Empty. The Low Memory Killer (LMK) uses an Out-Of-Memory (OOM) adjustment value (`adj` score) calculated by `ActivityManagerService`. Processes with higher adj scores (e.g., Background, Empty) are terminated first, while Foreground processes are kept alive as long as possible.
2. **What is the Zygote process and why does it exist?**
   * *Answer:* Zygote is a foundational process started during boot that contains preloaded JVM and framework classes. When a new app process is needed, the OS forks Zygote. This leverages Linux's Copy-On-Write (COW) optimization, enabling rapid process creation and shared memory usage across applications, saving valuable RAM.

### Advanced
1. **Walk through the exact chain of execution when a user taps an app icon on the home screen.**
   * *Answer:* Tapping an icon sends an Intent request to `ActivityManagerService` (AMS) via Binder IPC. AMS verifies the package and process state. If the process is absent, AMS requests the Zygote socket to fork a new process. The newly created process executes `ActivityThread.main()`, which initializes the Main Looper, instantiates the `Application` class, and invokes `onCreate()`. Finally, the system initiates the Main Activity through `TransactionExecutor`.

---

## Scenario Questions
1. **Scenario:** Your app has a background Service running and a user is typing in the foreground Activity of a different app. If the system runs low on memory, will your app's process be killed? Explain the mechanics.
   * *Answer:* Yes. The foreground app has a high priority (ADJ = 0). Your app's process containing only a background Service has a lower priority (ADJ = 500). If RAM is needed for the foreground app, LMK will terminate the Service process.
2. **Scenario:** A developer wants to run a network request immediately when the application starts. They write the networking code in the `init` block of the `Application` class. What will happen?
   * *Answer:* A `NetworkOnMainThreadException` will be thrown, causing the app to crash instantly. The `Application` class's initialization executes on the main thread, and network operations are strictly forbidden on the main thread.

---

## Code Examples

### Standard Production-Ready Manifest Declaration
Below is an example of a secure and modern `AndroidManifest.xml` structure.

```xml
<?xml version="1.0" encoding="utf-8"?>
<manifest xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:tools="http://schemas.android.com/tools"
    package="com.example.fitnessapp">

    <!-- Permissions -->
    <uses-permission android:name="android.permission.INTERNET" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE" />
    <uses-permission android:name="android.permission.FOREGROUND_SERVICE_LOCATION" />
    <uses-permission android:name="android.permission.ACCESS_FINE_LOCATION" />

    <!-- Query limits for Android 11+ -->
    <queries>
        <!-- Package names of apps we need to query -->
        <package android:name="com.google.android.apps.fitness" />
    </queries>

    <application
        android:name=".FitnessApplication"
        android:allowBackup="true"
        android:dataExtractionRules="@xml/data_extraction_rules"
        android:fullBackupContent="@xml/backup_rules"
        android:icon="@mipmap/ic_launcher"
        android:label="@string/app_name"
        android:roundIcon="@mipmap/ic_launcher_round"
        android:supportsRtl="true"
        android:theme="@style/Theme.FitnessApp"
        tools:targetApi="34">

        <!-- Main Launcher Activity -->
        <activity
            android:name=".presentation.MainActivity"
            android:exported="true"
            android:theme="@style/Theme.FitnessApp.Splash">
            <intent-filter>
                <action android:name="android.intent.action.MAIN" />
                <category android:name="android.intent.category.LAUNCHER" />
            </intent-filter>
        </activity>

        <!-- Secure Background Service (Non-exported) -->
        <service
            android:name=".data.service.LocationTrackingService"
            android:exported="false"
            android:foregroundServiceType="location" />

        <!-- Broadcast Receiver for boot completion -->
        <receiver
            android:name=".data.receiver.BootReceiver"
            android:exported="false">
            <intent-filter>
                <action android:name="android.intent.action.BOOT_COMPLETED" />
            </intent-filter>
        </receiver>

        <!-- Secure FileProvider Content Provider -->
        <provider
            android:name="androidx.core.content.FileProvider"
            android:authorities="${applicationId}.fileprovider"
            android:exported="false"
            android:grantUriPermissions="true">
            <meta-data
                android:name="android.support.FILE_PROVIDER_PATHS"
                android:resource="@xml/file_paths" />
        </provider>

    </application>
</manifest>
```

---

## Best Practices
* **Enforce Security Boundaries:** Explicitly set `android:exported="false"` for all components unless they specifically need to be launched by other applications.
* **Keep manifest lean:** Remove redundant elements and avoid declaration of components in separate processes (`android:process`) unless absolutely required.
* **Enable R8 / ProGuard:** Configure code shrinking, resource shrinking, and obfuscation to keep the APK file footprint small and optimize startup.
* **Use explicit intents:** For internal communication, always launch components with explicit intents (using the class name) to block component-hijacking attacks.

---

## Legacy vs Modern
* **Dalvik Runtime vs ART Runtime (Modern):** Dalvik used JIT which caused overhead on every launch. ART (introduced in Lollipop, improved iteratively through Android 15) uses Ahead-Of-Time (AOT) to compile bytecode into machine code during install/idle, and JIT with Profile-Guided Optimization (PGO) to constantly optimize hot code paths.
* **Imperative component configuration vs Declarative Jetpack frameworks:** In legacy code, developers often structured heavy application initialization sequences manually in the manifest/application; modern systems utilize the Jetpack App Startup library to initialize dependencies cleanly.

---

## Revision Notes
* Android stack is structured: Linux Kernel -> HAL -> Native C/C++ & ART -> Java Framework -> Apps.
* The 4 core components are Activities, Services, Broadcast Receivers, and Content Providers.
* App processes are spawned by Zygote using a Linux process fork containing preloaded JVM resources.
* System components are registered in `AndroidManifest.xml`.
* Low Memory Killer handles RAM constraints by prioritizing processes via OOM adjust (`adj`) scores.

---

## Key Takeaways
* Mobile app architecture differs from desktop; there is no single entrance, but multiple component access ports controlled by the OS.
* Understanding process priorities is essential to designing applications that behave gracefully when backgrounded or interrupted.
* Security must be enforced at the component level by explicitly setting the `exported` attribute in the manifest.

---
---

# Chapter 2: Application Class & Context

## Concept
The `Application` class and the `Context` object are the glue that holds an Android application together. Understanding their relationships, lifecycles, and internal implementations is crucial to avoiding memory leaks and writing robust applications.

### The `Application` Class
The `Application` class in Android is the base class for maintaining global application state. It is the very first class instantiated when the application's process starts. It exists as a singleton throughout the entire lifespan of the process.

### What is `Context`?
`Context` is an abstract class whose implementation is provided by the Android system. It is the interface to the application’s environment, allowing access to:
* **System Services:** Access to core OS services (e.g., `LAYOUT_INFLATER_SERVICE`, `CONNECTIVITY_SERVICE`, `NOTIFICATION_SERVICE`).
* **Application Resources:** Access to assets, resources, drawables, strings, and themes.
* **File Directory Access:** Methods to locate private databases, caches, and preference files (`filesDir`, `cacheDir`).
* **Component Instantiation:** Ability to launch Activities, start Services, and register Broadcast Receivers.

---

## Why It Exists
The separation between components and resources in Android requires a system-level broker. That broker is the `Context`.
* **Resource Decoupling:** Activities need to load layouts, styles, and dimensions. Rather than hardcoding references to file paths, the `Context` resolves resource IDs to the actual system assets.
* **Process and Component Boundaries:** An app runs in a sandbox. To interact with the system or other apps, it needs a token that verifies its identity (UID). The `Context` acts as this cryptographic validation token when requesting system actions.
* **Memory Optimization:** Different operations require different lifespans. Storing global resources in a short-lived component context (like an Activity) would result in memory leaks or crashes during configurations changes. The system provides multiple contexts (Application Context vs Activity Context) to separate short-lived UI scopes from global application scopes.

---

## Internal Working

### Context Hierarchy
The Android `Context` architecture is implemented via the **Decorator Pattern**.

```
                   +-------------------+
                   |      Context      | (Abstract Base)
                   +-------------------+
                             ^
                             |
                   +-------------------+
                   |    ContextImpl    | (Direct Implementation)
                   +-------------------+
                             ^
                             | (delegates to)
                   +-------------------+
                   |   ContextWrapper  | (Wrapper / Decorator)
                   +-------------------+
                    /        |        \
                   /         |         \
  +-----------------+ +--------------+ +----------------------+
  |   Application   | |   Service    | | ContextThemeWrapper  |
  +-----------------+ +--------------+ +----------------------+
                                                  |
                                                  v
                                         +------------------+
                                         |     Activity     |
                                         +------------------+
```

* **`ContextImpl`:** The core implementation class created by the system. It handles actual communication with the `ActivityManagerService` and accesses the filesystem.
* **`ContextWrapper`:** A class that delegates all of its calls to another `Context` (the base `ContextImpl`).
* **`ContextThemeWrapper`:** A subclass of `ContextWrapper` that adds styling and theme information. Activities inherit from this because they display UI and require themes, while Services and Applications do not need UI themes.

### Application Instantiation
When Zygote forks a process for your app:
1. `ActivityThread` is invoked.
2. The `LoadedApk` class loads the dex files.
3. The method `LoadedApk.makeApplication()` is executed.
4. It uses reflection to instantiate the `Application` subclass specified in `AndroidManifest.xml`.
5. It instantiates a `ContextImpl` object and binds it to the newly created `Application` singleton via `attachBaseContext(context)`.
6. Finally, `Application.onCreate()` is called on the main thread.

---

## Lifecycle / Flow

### Context Comparison Matrix

| Context Type | Lifespan | Theme-Aware? | Safe for Singletons? | Can Show Dialogs? |
| :--- | :--- | :--- | :--- | :--- |
| **Application Context** | Process Lifetime | No | Yes | No (Throws WindowManager$BadTokenException) |
| **Activity Context** | Activity Lifetime | Yes | No (Causes Memory Leak) | Yes |
| **Service Context** | Service Lifetime | No | No | No |

---

## Real World Example
Below is a standard production implementation of a custom `Application` class. It configures dependency injection (Hilt), logging (Timber), database warm-up, and handles memory pressure events.

```kotlin
package com.example.fitnessapp

import android.app.Application
import android.content.ComponentCallbacks2
import android.content.res.Configuration
import android.util.Log
import com.example.fitnessapp.data.local.AppDatabase
import dagger.hilt.android.HiltAndroidApp
import kotlinx.coroutines.CoroutineScope
import kotlinx.coroutines.Dispatchers
import kotlinx.coroutines.SupervisorJob
import kotlinx.coroutines.launch
import javax.inject.Inject

@HiltAndroidApp
class FitnessApplication : Application() {

    private val applicationScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

    @Inject
    lateinit var database: AppDatabase

    override fun onCreate() {
        super.onCreate()
        initializeLogging()
        warmUpDatabase()
    }

    private fun initializeLogging() {
        // e.g. Setup Timber or custom crashlytics reporting
        Log.i("FitnessApplication", "Application Initialized")
    }

    private fun warmUpDatabase() {
        // Safe to offload asynchronous initialization to background thread
        applicationScope.launch {
            database.queryHelper().warmUp()
        }
    }

    override fun onConfigurationChanged(newConfig: Configuration) {
        super.onConfigurationChanged(newConfig)
        Log.d("FitnessApplication", "Configuration changed: ${newConfig.locale}")
    }

    override fun onTrimMemory(level: Int) {
        super.onTrimMemory(level)
        when (level) {
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_MODERATE,
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_LOW,
            ComponentCallbacks2.TRIM_MEMORY_RUNNING_CRITICAL -> {
                // Clear in-memory caches, image libraries (Coil/Glide), or temporary caches
                Log.w("FitnessApplication", "Memory running low. Releasing caches. Level: $level")
            }
            ComponentCallbacks2.TRIM_MEMORY_UI_HIDDEN -> {
                // The user has navigated away from our UI. Release heavy UI resources.
            }
        }
    }

    override fun onLowMemory() {
        super.onLowMemory()
        // Legacy callback, equivalent to TRIM_MEMORY_COMPLETE
    }

    override fun onTerminate() {
        super.onTerminate()
        // WARNING: This is NEVER called on a production device.
        // It is only used in emulation environments for testing.
    }
}
```

---

## Common Mistakes
1. **Leaking Activity Context inside Singletons:** Storing a reference to an Activity context in a static variable or a singleton class. When the Activity is destroyed, it cannot be garbage collected because the singleton holds a reference to it.
2. **Blocking `Application.onCreate()`:** Performing synchronous database queries, REST API initialization, or heavy file I/O inside `onCreate()`. This directly blocks the main thread and delays app launch, causing high cold start times and ANRs.
3. **Attempting to display Dialogs with Application Context:** Passing `applicationContext` to an `AlertDialog.Builder`. Dialogs must attach to a valid Window Token, which only an `Activity` (inheriting from `ContextThemeWrapper`) possesses.
4. **Expecting `onTerminate()` to handle cleanup:** Writing cleanup logic (saving data, closing connections) in `Application.onTerminate()`. Under normal operations, the Android OS kills the entire process directly via the Linux kernel when exiting, skipping this callback.

---

## Memory Leak / Performance Concerns
* **The Singleton Leak:** If a Class `UserManager` needs a Context to read SharedPreferences, passing the Activity context to `UserManager.getInstance(context)` leaks the Activity when it rotates.
  * *Fix:* Use `context.applicationContext` inside the singleton initialization.
* **Unoptimized SDK Initializations:** Third-party libraries (Firebase, Facebook SDK, AdMob) initialized sequentially in `Application.onCreate()` block main thread startup. Use the Jetpack App Startup library to run initializations asynchronously or in parallel.

---

## Interview Questions

### Beginner
1. **What is the difference between Application Context and Activity Context?**
   * *Answer:* Application Context is tied to the lifespan of the application process and is not theme-aware. Activity Context is tied to the lifecycle of the Activity, contains theme and layout parameters, and is safe for UI-specific calls like displaying Dialogs and inflating layouts.
2. **Is `onTerminate()` called when a user closes the app?**
   * *Answer:* No. On real devices, `onTerminate()` is never called. The system kills the process from user space without invoking lifecycle cleanups in this method.

### Intermediate
1. **Why does using the Application Context to inflate a custom View sometimes fail or look incorrect?**
   * *Answer:* The Application Context does not contain the styling and theme parameters specified in the Activity's theme configuration. Inflating views with it will fall back to default system styles, missing custom fonts, colors, and layout constraints.
2. **Explain the purpose of `onTrimMemory(level)` and name three memory levels.**
   * *Answer:* It allows the application to respond to memory pressure signals sent by the system to prevent the process from being killed. Levels include: `TRIM_MEMORY_RUNNING_CRITICAL` (device is low on RAM, app must release caches), `TRIM_MEMORY_UI_HIDDEN` (app is no longer showing UI, release UI resources), and `TRIM_MEMORY_COMPLETE` (process is near the top of the LMK kill list).

### Advanced
1. **How does the Decorator pattern apply to Android's `Context` architecture? Why is it structured this way?**
   * *Answer:* Android uses a base abstract class `Context`. The real logic is inside `ContextImpl`. `ContextWrapper` delegates all method executions to `ContextImpl` while allowing developers to subclass it (such as in `Application`, `Service`, and `ContextThemeWrapper`) without rewriting standard file access or component instantiation logic. This separates implementation details from wrapper contexts.

---

## Scenario Questions
1. **Scenario:** You are using a dependency injection framework (like Hilt) and want to inject a Database instance. The database requires a Context to build. Which Context should you bind and why?
   * *Answer:* You must bind the Application Context (`@ApplicationContext`). The database singleton exists for the lifetime of the process. Binding an Activity Context would cause a massive memory leak of the Activity on configuration changes.
2. **Scenario:** Your app has an onboarding flow. When the user finishes, you need to navigate to the MainActivity. You want to start MainActivity from a background BroadcastReceiver. How do you do it, and what intent flags are required?
   * *Answer:* You must use the Application Context. Since you are launching an Activity from outside an Activity context, you must set `Intent.FLAG_ACTIVITY_NEW_TASK` on the intent. Without this flag, the operation will crash because there is no task back stack to attach the new Activity to.

---

## Code Examples

### Modern Initialization via Jetpack App Startup Library
To keep `Application.onCreate()` clean, we utilize App Startup for non-blocking parallel initialization.

```kotlin
// 1. Define the Initializer for Timber
package com.example.fitnessapp.init

import android.content.Context
import androidx.startup.Initializer
import android.util.Log

class LoggingInitializer : Initializer<Unit> {
    override fun create(context: Context) {
        Log.i("LoggingInitializer", "Timber / Logging library initialized")
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        // No dependencies, can initialize immediately
        return emptyList()
    }
}

// 2. Define the Database Initializer (depends on Logging)
package com.example.fitnessapp.init

import android.content.Context
import androidx.startup.Initializer
import com.example.fitnessapp.data.local.AppDatabase

class DatabaseInitializer : Initializer<AppDatabase> {
    override fun create(context: Context): AppDatabase {
        return AppDatabase.getInstance(context)
    }

    override fun dependencies(): List<Class<out Initializer<*>>> {
        // Enforce order: Logging must initialize first
        return listOf(LoggingInitializer::class.java)
    }
}
```

Register the initializers in the `AndroidManifest.xml` under the `<provider>` section:

```xml
<provider
    android:name="androidx.startup.InitializationProvider"
    android:authorities="${applicationId}.androidx-startup"
    android:exported="false"
    tools:node="merge">
    <meta-data
        android:name="com.example.fitnessapp.init.DatabaseInitializer"
        android:value="androidx.startup" />
</provider>
```

---

## Best Practices
* **Use the narrowest context scope:** Always use the most restricted context available. For UI components, use the Activity context. For persistent singletons, background jobs, or database initialization, use the Application context.
* **Keep `Application.onCreate()` Lightweight:** Offload secondary initialization steps to background threads or use the Jetpack App Startup library.
* **Unregister listeners:** If you register dynamic listeners using context references, always unregister them when the context's lifecycle ends.
* **Never store context instances in static fields:** If context references must be passed around, utilize weak references (`WeakReference<Context>`).

---

## Legacy vs Modern
* **Manual initialization sequences vs Jetpack App Startup (Modern):** In older codebases, developers manually spawned background threads inside `Application.onCreate()` to initialize libraries. The App Startup library provides a unified, thread-pool backed dependency graph initializer.
* **Hilt Injection vs Manual DI (Legacy):** Modern DI leverages `@HiltAndroidApp` to automate the binding of application contexts, removing the need to write static `applicationContext` singletons.

---

## Revision Notes
* `Context` is an abstract interface to the application environment.
* The context hierarchy uses the Decorator pattern (`ContextImpl` -> `ContextWrapper` -> subclasses).
* Application Context lives for the entire process; Activity Context is short-lived and theme-aware.
* Leaking Activity context in singletons is a primary cause of memory leaks.
* `onTrimMemory()` allows releasing resources dynamically during memory pressure.
* `onTerminate()` is never executed on production devices.

---

## Key Takeaways
* Never pass Activity contexts to objects that will outlive the Activity (such as singletons or thread managers).
* Accessing system resources or databases should be parameterized with Application Context.
* Dialogs require a Window Token, which is exclusive to Activity Context.

---
---

# Chapter 3: Activity Lifecycle

## Concept
An **Activity** is the core entry point for a user's interaction with an Android application. The **Activity Lifecycle** is a set of states through which an Activity transitions from the moment it is initialized until it is destroyed by the system. Developers must implement lifecycle callbacks to manage resources, state, and UI binding efficiently.

### The Lifecycle States & Callbacks
The Android framework defines six principal states, with corresponding callback methods:
1. **`onCreate()` (CREATED State):** Called when the system first creates the Activity. Performs basic startup logic that should happen only once (e.g., inflating layout, binding data to lists, initializing viewmodels).
2. **`onStart()` (STARTED State):** Called when the Activity becomes visible to the user. The UI is initialized but not yet interactive.
3. **`onResume()` (RESUMED State):** Called when the Activity enters the foreground and becomes interactive. This is the state where the app interacts with the user.
4. **`onPause()` (PAUSED State):** Called when the Activity loses focus (e.g., partially covered by a dialog, multi-window mode, or a new transparent activity). The Activity is still partially visible.
5. **`onStop()` (STOPPED State):** Called when the Activity is no longer visible to the user. This occurs when another Activity covers the screen completely, or the user navigates home.
6. **`onDestroy()` (DESTROYED State):** Called before the Activity is destroyed. This is the final cleanup point. It can occur because the user finishes the Activity (`finish()`) or the system is temporarily destroying it for a configuration change.

```
+-------------------------------------------------------------+
|                         ONCREATE()                          |
+-------------------------------------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|                          ONSTART()                          |
+-------------------------------------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|                         ONRESUME()                          | <------+
+-------------------------------------------------------------+        |
                               |                                       |
                     Activity  | User opens dialog /                   |
                     loses     | Activity partially covered            | User returns
                     focus     v                                       | to Activity
+-------------------------------------------------------------+        |
|                          ONPAUSE()                          | -------+
+-------------------------------------------------------------+
                               |
                     Activity  | User navigates home /
                     no longer | new Activity opens
                     visible   v
+-------------------------------------------------------------+
|                          ONSTOP()                           | -------> User navigates back
+-------------------------------------------------------------+          (Recreates / starts)
                               |
                               | User finishes or system kills
                               v
+-------------------------------------------------------------+
|                         ONDESTROY()                         |
+-------------------------------------------------------------+
```

### State Preservation
When an Activity is stopped, the OS reserves its view state in memory. However, if the system kills the process under memory pressure, or if a configuration change (e.g., screen rotation) occurs, the Activity instance is destroyed. Android provides two mechanisms for state preservation:
* **`onSaveInstanceState(outState: Bundle)`:** The system calls this method before `onStop()`. Developers can store small, serializable data in the `Bundle` (primitives, strings, parcelables). This Bundle persists even if the app process is terminated by the Low Memory Killer.
* **`onRestoreInstanceState(savedInstanceState: Bundle)`:** Called after `onStart()` ONLY if the Activity is being recreated from a saved state. The system automatically restores the state of Views that have an `android:id` (e.g., scroll positions, input text).

---

## Why It Exists
The Activity lifecycle exists because mobile devices are highly dynamic, resource-constrained environments:
* **Limited Hardware Resources:** The OS must ensure that background apps do not consume CPU, GPU, or battery. Lifecycles allow the system to stop animations, pause location updates, and free up heavy objects when an app is invisible.
* **Graceful Interruption Handling:** A user may receive a phone call, see a low-battery notification, or lock their device while using an app. The lifecycle callbacks tell the developer exactly when to save state and pause active processes.
* **Configuration Adaptation:** Android devices can rotate, change display language, plug into external monitors, or enter multi-window mode. The destroy-recreate cycle is a clean way for the app to reload resources (layouts, dimensions, translations) matching the new device state.

---

## Internal Working
The lifecycle is orchestrated by the **`ActivityManagerService` (AMS)** and the **`ActivityThread`** inside the app process.

### The Lifecycle Loop via Binder IPC
1. When a transition is needed (e.g., Activity A starting Activity B), AMS sends a Binder transaction to the app process's `ActivityThread`.
2. Inside `ActivityThread`, the message is queued in the `Looper`'s `MessageQueue` and picked up on the main thread.
3. The `ActivityThread` delegates the execution to the **`TransactionExecutor`**.
4. The `TransactionExecutor` executes the lifecycle callbacks. It uses a **`ClientTransaction`** containing a list of lifecycle callbacks (e.g., `PauseActivityItem`, `StopActivityItem`) to move the Activity from one state to another.
5. In modern Android, these transitions are managed by the **`LifecycleRegistry`**, which notifies registered `LifecycleObserver` objects.

```
+--------------------------+                   Binder IPC                   +--------------------------+
|  ActivityManagerService  | ---------------------------------------------> |      ActivityThread      |
|          (AMS)           | <--------------------------------------------- |   (App Main Thread)      |
+--------------------------+          IPC Lifecycle Commands                +--------------------------+
                                                                                          |
                                                                                          v
                                                                            +--------------------------+
                                                                            |   TransactionExecutor    |
                                                                            +--------------------------+
                                                                                          |
                                                                                          v
                                                                            +--------------------------+
                                                                            |    LifecycleRegistry     |
                                                                            +--------------------------+
                                                                                          |
                                                                                          v
                                                                            +--------------------------+
                                                                            |     Activity Callbacks   |
                                                                            | (onCreate/onStart/etc.)  |
                                                                            +--------------------------+
```

### Configuration Changes & View State Restoration
When a configuration change occurs:
1. AMS requests the app to save its state. `ActivityThread` executes `onSaveInstanceState()`.
2. The view hierarchy's state is traversed recursively. Every View with a valid ID writes its state into a hierarchical `Bundle`.
3. AMS destroys the Activity instance by executing `onPause()`, `onStop()`, and `onDestroy()`.
4. AMS immediately instantiates a new Activity object.
5. The saved `Bundle` is passed back during instantiation to `onCreate(savedInstanceState)` and `onRestoreInstanceState(savedInstanceState)`.

---

## Lifecycle / Flow

### Normal Navigation vs Configuration Change vs Process Death

```
Activity A Launches
  └─ A.onCreate() -> A.onStart() -> A.onResume() [A is visible & focused]

Activity A Starts Activity B
  ├─ A.onPause()
  ├─ B.onCreate() -> B.onStart() -> B.onResume() [B is visible & focused]
  └─ A.onStop() [A is now invisible]

Activity B Finishes (Back Button Pressed)
  ├─ B.onPause()
  ├─ A.onStart() -> A.onResume() [A resumes focus]
  └─ B.onStop() -> B.onDestroy() [B is completely destroyed]

Screen Rotation (Configuration Change) on A
  ├─ A.onPause()
  ├─ A.onStop()
  ├─ A.onSaveInstanceState(outState: Bundle)
  ├─ A.onDestroy()  [Old instance destroyed]
  ├─ A.onCreate(savedInstanceState: Bundle) [New instance created]
  ├─ A.onStart()
  ├─ A.onRestoreInstanceState(savedInstanceState: Bundle)
  └─ A.onResume()

Process Death under Low Memory
  ├─ User navigates home: A.onPause() -> A.onStop() -> A.onSaveInstanceState()
  ├─ System kills process. (Memory is reclaimed)
  ├─ User returns to app: Process restarts -> Zygote forks
  ├─ A.onCreate(savedInstanceState: Bundle) [Bundle restored]
  └─ A.onStart() -> A.onRestoreInstanceState() -> A.onResume()
```

---

## Real World Example
In a camera application, managing the lifecycle correctly is critical to prevent resource locking. The camera sensor is a system-wide resource; if our Activity goes invisible, we must release it so other apps can use it.

Below is a production-quality, lifecycle-aware Camera controller.

```kotlin
package com.example.fitnessapp.presentation

import android.content.Context
import android.hardware.camera2.CameraManager
import android.util.Log
import androidx.lifecycle.DefaultLifecycleObserver
import androidx.lifecycle.LifecycleOwner

class LifecycleCameraManager(
    private val context: Context
) : DefaultLifecycleObserver {

    private var cameraManager: CameraManager? = null
    private var isCameraActive = false

    override fun onCreate(owner: LifecycleOwner) {
        super.onCreate(owner)
        // Initialize non-heavy hardware links
        cameraManager = context.getSystemService(Context.CAMERA_SERVICE) as CameraManager
        Log.d("CameraManager", "Camera Service linked in onCreate")
    }

    override fun onStart(owner: LifecycleOwner) {
        super.onStart(owner)
        // Camera resources are heavy, open them when Activity is starting (visible)
        openCamera()
    }

    override fun onStop(owner: LifecycleOwner) {
        super.onStop(owner)
        // Release camera when Activity is no longer visible to user
        releaseCamera()
    }

    override fun onDestroy(owner: LifecycleOwner) {
        super.onDestroy(owner)
        // Clean up references
        cameraManager = null
        Log.d("CameraManager", "Cleaned up references in onDestroy")
    }

    private fun openCamera() {
        if (!isCameraActive) {
            Log.i("CameraManager", "Camera opened successfully")
            isCameraActive = true
        }
    }

    private fun releaseCamera() {
        if (isCameraActive) {
            Log.i("CameraManager", "Camera released successfully")
            isCameraActive = false
        }
    }
}
```

---

## Common Mistakes
1. **Executing Network Requests or Database Queries inside `onResume()` without Checks:** `onResume()` is called every time the Activity gains focus (e.g., when a dialog closes). Querying data here repeatedly creates massive layout Thrashing and network overhead.
2. **Assuming `onDestroy()` is Always Called:** If the system terminates a background process under memory pressure, it does so immediately via SIGKILL. The `onDestroy()` callback is never executed. Do not rely on it for critical persistence (e.g., saving user progress).
3. **Saving Large Data Collections in `onSaveInstanceState()`:** The transaction buffer size limit for standard Binder transactions is **1MB** shared across the entire process. Storing high-res Bitmaps or large databases in the Bundle will throw a `TransactionTooLargeException`.
4. **Registering Observers without Unregistering:** Adding listeners or RxJava subscriptions in `onStart()` or `onResume()` without disposing of them in `onStop()` or `onPause()` causes memory leaks and background task execution in dead components.

---

## Memory Leak / Performance Concerns
* **The Static View Leak:** Storing a View or Activity instance in a static variable. Since the static variable lives for the life of the process, the Activity cannot be GC'd, leaking the entire layout tree and images.
* **Main Thread Blocking during onCreate:** Storing heavy parsing or I/O initialization blocks in `onCreate()` delays view inflation, causing frame drops and potential App Launch ANRs.

---

## Interview Questions

### Beginner
1. **Explain the Activity lifecycle states in chronological order.**
   * *Answer:* The lifecycle states are Non-existent -> CREATED (`onCreate()`) -> STARTED (`onStart()`) -> RESUMED (`onResume()`) -> PAUSED (`onPause()`) -> STOPPED (`onStop()`) -> DESTROYED (`onDestroy()`).
2. **What is the purpose of `onSaveInstanceState()`?**
   * *Answer:* It allows the Activity to save transient UI state (e.g., user input, scroll position) to a key-value Bundle before the Activity is destroyed by the system, ensuring state can be restored if the system recreates the Activity later.

### Intermediate
1. **What is the difference between `onPause()` and `onStop()`?**
   * *Answer:* `onPause()` is called when the Activity is still partially visible (losing focus, e.g., split-screen, transparent activity overlaid). `onStop()` is called when the Activity is completely invisible (e.g., navigated home or covered entirely by a new fullscreen Activity).
2. **Why should you use `repeatOnLifecycle` or `flowWithLifecycle` when observing cold flows in an Activity?**
   * *Answer:* Standard observation with `lifecycleScope.launch` does not stop collector execution when the Activity goes into the background (`onStop()`). `repeatOnLifecycle(Lifecycle.State.STARTED)` automatically launches a coroutine when the lifecycle enters the `STARTED` state and cancels it immediately when the lifecycle moves below `STARTED`, preventing waste of CPU and network resources.

### Advanced
1. **If Activity A starts Activity B, what is the exact chronological sequence of lifecycle callbacks executed across both Activities?**
   * *Answer:* The sequence is:
     1. Activity A's `onPause()` runs.
     2. Activity B's `onCreate()`, `onStart()`, and `onResume()` run. B is now in focus.
     3. Activity A's `onStop()` runs (since A is now fully covered by B).
2. **How does `ViewModel` survive configuration changes while the Activity is destroyed? Detail the internal mechanism.**
   * *Answer:* During configuration changes, the OS calls `Activity.retainNonConfigurationInstances()`. The system stores the `ViewModelStore` (containing all ViewModel instances) in a wrapper called `NonConfigurationInstances`. When the new Activity instance is instantiated, `ActivityThread` passes this wrapper back, and the new Activity attaches to the existing `ViewModelStore` via `ViewModelProvider`, bypassing recreated initializations.

---

## Scenario Questions
1. **Scenario:** The user is filling out a long form in your Activity. They rotate the device. Write the exact callbacks that occur, and explain where you will save and restore the entered data.
   * *Answer:* Callbacks: `onPause() -> onStop() -> onSaveInstanceState(Bundle) -> onDestroy()` -> (Recreation) -> `onCreate(Bundle) -> onStart() -> onRestoreInstanceState(Bundle) -> onResume()`. You will save the form input string in `onSaveInstanceState()` using `outState.putString(KEY_FORM, text)` and retrieve it either in `onCreate(savedInstanceState)` or `onRestoreInstanceState(savedInstanceState)`.
2. **Scenario:** An app needs to track location coordinates. You start requesting location updates in `onResume()`. Where must you stop requesting updates to avoid leaking the location client, and what happens if you fail to do so?
   * *Answer:* You must stop requesting updates in `onPause()`. If you fail to do so, the location manager will continue processing GPS coordinates in the background, wasting battery, violating user privacy policies, and leaking the Activity context if it cannot be GC'd.

---

## Code Examples

### State Preservation Implementation
Below is a production-level Kotlin implementation of an Activity implementing state saving and modern lifecycle-aware observations.

```kotlin
package com.example.fitnessapp.presentation

import android.os.Bundle
import android.widget.EditText
import androidx.activity.viewModels
import androidx.appcompat.app.AppCompatActivity
import androidx.lifecycle.Lifecycle
import androidx.lifecycle.lifecycleScope
import androidx.lifecycle.repeatOnLifecycle
import com.example.fitnessapp.R
import kotlinx.coroutines.flow.collectLatest
import kotlinx.coroutines.launch

class FormActivity : AppCompatActivity() {

    private val viewModel: FormViewModel by viewModels()
    private lateinit var commentInput: EditText

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_form)

        commentInput = findViewById(R.id.commentInput)

        // Bind lifecycle-aware camera observer
        val cameraManager = LifecycleCameraManager(applicationContext)
        lifecycle.addObserver(cameraManager)

        // Restore custom UI state from process death
        if (savedInstanceState != null) {
            val savedComment = savedInstanceState.getString(KEY_COMMENT_TEXT, "")
            commentInput.setText(savedComment)
        }

        // Modern: Observe state flows safely
        lifecycleScope.launch {
            repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collectLatest { state ->
                    // Update UI components
                }
            }
        }
    }

    override fun onSaveInstanceState(outState: Bundle) {
        // Save minimal transient state to Bundle
        outState.putString(KEY_COMMENT_TEXT, commentInput.text.toString())
        super.onSaveInstanceState(outState)
    }

    companion object {
        private const val KEY_COMMENT_TEXT = "key_comment_text"
    }
}
```

---

## Best Practices
* **Keep `onCreate()` Lightweight:** Avoid synchronous heavy loads. Initialize view layouts, link models, and offload calculations.
* **Observe Safely:** Use `repeatOnLifecycle` or `flowWithLifecycle` for collect flows.
* **Bundle Limits:** Keep data in `onSaveInstanceState` under 50KB to protect Binder transaction limits.
* **Decouple Logic:** Implement `DefaultLifecycleObserver` in separate controller classes instead of bloating the Activity file with business logic.

---

## Legacy vs Modern
* **`onActivityCreated()` vs `onViewCreated()` (Modern):** For Fragment lifecycle attachments, `onActivityCreated()` is completely deprecated; initialization should reside in `onViewCreated()` or `onViewStateRestored()`.
* **Manual lifecycle-check variables vs Lifecycle-Aware Components (Modern):** Legacy developers checked `if (!isFinishing)` inside background threads; modern architectures utilize `lifecycleScope` and `DefaultLifecycleObserver` to bind tasks directly to lifecycle states.

---

## Revision Notes
* The six core callbacks are `onCreate`, `onStart`, `onResume`, `onPause`, `onStop`, `onDestroy`.
* Orientation change destroys and recreates the Activity instance.
* `onSaveInstanceState()` saves transient UI state; the Binder transaction limit is 1MB process-wide.
* `ViewModel` persists configuration changes via `NonConfigurationInstances`.
* Collect UI state safely using `repeatOnLifecycle(STARTED)` to pause background resource usage.

---

## Key Takeaways
* Lifecycle callbacks are hook points designed to manage system resources dynamically.
* Protect against background process death by saving data to the save instance Bundle.
* Activities do not control their own lifetime; the OS destroys components to reclaim hardware resources.

---
---

# Chapter 4: Activity Launch Modes & Back Stack

## Concept
In Android, navigation is managed using tasks and stacks. A **Task** is a collection of Activities that users interact with when performing a specific job. These Activities are arranged in a Last-In, First-Out (LIFO) stack called the **Back Stack**. 

The order in which new Activities are placed in the stack and how existing instances are reused are controlled by **Launch Modes** and **Intent Flags**.

### The 4 Standard Launch Modes
Declared inside the `AndroidManifest.xml` via the `android:launchMode` attribute:
1. **`standard` (Default):** The system always creates a new instance of the Activity in the task from which it was started. Multiple instances of the same Activity can exist in the same or different tasks.
2. **`singleTop`:** Reuses the instance if it is already at the **top** of the back stack. If found on top, the system routes the new Intent to the existing instance via its `onNewIntent(intent)` callback, instead of creating a new instance.
3. **`singleTask`:** Creates a new task and instantiates the Activity at the root of the new task. However, if an instance of the Activity already exists in any task, the system routes the Intent to that instance via `onNewIntent()`. Crucially, **it clears all Activities on top of it**, making it the top Activity in the stack.
4. **`singleInstance`:** Similar to `singleTask`, but the system does not launch any other Activities into the task holding the Activity instance. The Activity is always the single and unique member of its task. Any subsequent Activities started from this one open in a separate task.

### Modern Addition: `singleInstancePerTask` (API 31+)
* **`singleInstancePerTask`:** The activity can only run as the root activity of the task (the first activity in the task). Only one instance of this activity can exist in a given task, but it can be instantiated in multiple different tasks.

### Intent Flags (Dynamic Launch Modes)
Launched dynamically during code execution using `Intent.addFlags()`:
* **`FLAG_ACTIVITY_NEW_TASK`:** Starts the Activity in a new task. If a task already exists for this Activity, that task is brought to the foreground with its last state restored. (Equivalent to `singleTask`).
* **`FLAG_ACTIVITY_CLEAR_TOP`:** If the Activity being started is already running in the current task, then instead of launching a new instance, all of the other Activities on top of it are closed, and this Intent is delivered to the old Activity via `onNewIntent()`.
* **`FLAG_ACTIVITY_SINGLE_TOP`:** If the Activity is already running at the top of the history stack, the activity will not be restarted. (Equivalent to `singleTop`).
* **`FLAG_ACTIVITY_REORDER_TO_FRONT`:** Moves an existing instance of the Activity to the front of the back stack without clearing the stack.

---

## Why It Exists
Proper task management is necessary to provide an intuitive UX:
* **Preventing Duplicate Screens:** If a user clicks a notification to open the `HomeActivity`, the system should not create a new instance of `HomeActivity` if they are already looking at it.
* **Task Isolation:** If an app launches a third-party payment screen, that payment screen should run in its own task so that if the user clicks back or switches apps, the payment transaction is handled separately without cluttering the primary app's flow.
* **LIFO Customization:** The standard LIFO back stack is simple but insufficient for complex navigation hierarchies. Intent flags permit modifying the stack history dynamically (e.g., clearing the login back stack once onboarding completes).

---

## Internal Working
The tasks and back stack configurations are tracked by the `ActivityManagerService` using two internal records:
* **`ActivityRecord`:** Represents a single Activity component instance in memory.
* **`TaskRecord`:** Represents a group of ActivityRecords sharing a common workflow.

### Task Affinity
Every Activity has a **Task Affinity** (`android:taskAffinity` in the manifest). By default, all Activities inside an app share the same affinity (matching the application's package name).
* **Reparenting:** If an Activity is started with `FLAG_ACTIVITY_NEW_TASK` or has `allowTaskReparenting="true"`, it will search for a task with matching affinity. If one exists, it moves into that task.

### The `onNewIntent(intent)` Pipeline
When an Activity is reused due to a launch mode or flag configuration:
1. The Activity is brought to the foreground.
2. The system invokes `onNewIntent(Intent)`.
3. The new Intent parameters are stored. (Note: The Activity must call `setIntent(intent)` inside this callback, otherwise subsequent calls to `intent` will return the *original* Intent that launched the Activity first).
4. The Activity transitions: `onNewIntent() -> onResume()`.

---

## Lifecycle / Flow

### Back Stack State Transitions under Different Launch Modes

#### standard mode (Multiple instances created)
```
Initial Stack: [A]
Start Activity B (standard)  -> Stack: [A -> B]
Start Activity B (standard)  -> Stack: [A -> B -> B]
```

#### singleTop mode (Top instance reused)
```
Initial Stack: [A -> B]
Start Activity B (singleTop) -> Stack: [A -> B] (Runs B.onNewIntent())
Start Activity C (standard)  -> Stack: [A -> B -> C]
Start Activity B (singleTop) -> Stack: [A -> B -> C -> B] (New B created because B was not on top)
```

#### singleTask mode (Clears top activities)
```
Initial Stack: [A -> B -> C]
Start Activity B (singleTask) -> Stack: [A -> B] (Destroys C, calls B.onNewIntent())
```

#### singleInstance mode (Isolated Task)
```
Task 1: [A -> B]
Start Activity C (singleInstance)
Task 1: [A -> B]
Task 2: [C] (C runs isolated in a separate task)
```

---

## Real World Example
Let's analyze two standard navigation flows in production apps:
1. **Handling Push Notifications:** A user receives a chat notification. When clicked, it should open `ChatActivity`. If `ChatActivity` is already open, it should simply load the new message on the existing screen rather than creating another instance.
   * *Solution:* Declare `ChatActivity` as `singleTop` or use `FLAG_ACTIVITY_SINGLE_TOP` with `FLAG_ACTIVITY_CLEAR_TOP`.
2. **Post-Login Clean Up:** A user enters their credentials in `LoginActivity`. Once authenticated, they land on `DashboardActivity`. Pressing the hardware back button should exit the app, NOT show the login screen again.
   * *Solution:* Launch `DashboardActivity` with flags `FLAG_ACTIVITY_NEW_TASK` and `FLAG_ACTIVITY_CLEAR_TASK`.

---

## Common Mistakes
1. **Forgetting `setIntent(intent)` inside `onNewIntent()`:** If you skip calling `setIntent(intent)` in `onNewIntent()`, calls to `getIntent()` anywhere else in the Activity (like `onResume`) will return the original, stale Intent, preventing the screen from loading new parameter data.
2. **Unnecessarily marking everything as `singleTask` in Manifest:** Setting `launchMode="singleTask"` for sub-screens. This destroys the back stack history and frustrates users by unexpectedly clearing previously loaded screens.
3. **Implicitly starting Activity from Service without `FLAG_ACTIVITY_NEW_TASK`:** Starting an Activity using a Context that does not have an active task history (like Application Context or Service Context). The operation will crash with `AndroidRuntimeException`.

---

## Memory Leak / Performance Concerns
* **Back Stack Bloat:** In `standard` mode, starting screens recursively in circles (e.g., A -> B -> A -> B) creates duplicate instances, accumulating megabytes of layout nodes in memory, eventually resulting in an `OutOfMemoryError`.
* **Activity Leak via taskAffinity:** Setting custom affinities for activities can keep their records alive in background task structures even if the main application process transitions, retaining assets and contexts.

---

## Interview Questions

### Beginner
1. **What is the default launch mode of an Activity?**
   * *Answer:* The default launch mode is `standard`. It always creates a new instance.
2. **What is the difference between `standard` and `singleTop`?**
   * *Answer:* `standard` always creates a new instance. `singleTop` reuses the existing instance ONLY if it is already at the top of the stack, executing its `onNewIntent()` method.

### Intermediate
1. **Explain the behavior of `singleTask` launch mode. What happens to the activities on top of it?**
   * *Answer:* `singleTask` checks if an instance of the Activity exists in any active task. If it does, the system brings that task to the foreground, routes the Intent to the instance via `onNewIntent()`, and destroys all Activities currently resting on top of it in that stack (popping them off).
2. **What is Task Affinity and how does it affect launch modes?**
   * *Answer:* Task Affinity defines which task an Activity prefers to run in. By default, it is the app's package name. When an Activity is launched with `FLAG_ACTIVITY_NEW_TASK` or `singleTask`, it uses the affinity to look for an existing task with the same affinity to join.

### Advanced
1. **How do `FLAG_ACTIVITY_CLEAR_TOP` and `FLAG_ACTIVITY_SINGLE_TOP` interact when used together in an intent?**
   * *Answer:* If used together, they clear all activities resting above the target activity in the stack. Then, instead of destroying the target activity and recreating it, the target activity is brought to the top and receives the new intent via `onNewIntent()`. If `FLAG_ACTIVITY_CLEAR_TOP` is used *without* `FLAG_ACTIVITY_SINGLE_TOP`, the target activity itself is also destroyed and recreated.
2. **Explain what happens when an Activity declared as `singleInstance` attempts to launch a standard Activity.**
   * *Answer:* A `singleInstance` Activity cannot share its task with any other component. Therefore, if it launches a `standard` Activity, the system automatically redirects the launch into a separate task. If the launched activity has the same affinity, it joins a new task of that affinity; otherwise, it creates an entirely new task.

---

## Scenario Questions
1. **Scenario:** Your app stack contains A -> B -> C -> D. D launches B with an intent having flags `FLAG_ACTIVITY_CLEAR_TOP` and `FLAG_ACTIVITY_SINGLE_TOP`. Detail the new stack structure and list the lifecycle callbacks executed on B.
   * *Answer:* Stack structure becomes: `[A -> B]`. Activities C and D are destroyed. Lifecycle callbacks on B: `B.onNewIntent() -> B.onResume()`. B is not recreated.
2. **Scenario:** You are launching a MainActivity from a widget notification via a `PendingIntent`. You want to make sure that if the app is already open, it does not launch a duplicate, and if it is closed, it launches clean. How do you construct the PendingIntent?
   * *Answer:* You wrap the intent with `FLAG_ACTIVITY_CLEAR_TOP` and `FLAG_ACTIVITY_SINGLE_TOP`. When creating the `PendingIntent`, you pass the flag `PendingIntent.FLAG_UPDATE_CURRENT` to update the extras on the intent dynamically.

---

## Code Examples

### Reusing Activity Instance via `onNewIntent()`
This example demonstrates how to process incoming intents dynamically without destroying the activity instance.

```kotlin
package com.example.fitnessapp.presentation

import android.content.Intent
import android.os.Bundle
import android.widget.TextView
import androidx.appcompat.app.AppCompatActivity
import com.example.fitnessapp.R

class ChatActivity : AppCompatActivity() {

    private lateinit var chatTitleView: TextView

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_chat)
        chatTitleView = findViewById(R.id.chatTitle)

        // Process original intent
        intent?.let { handleIncomingIntent(it) }
    }

    override fun onNewIntent(intent: Intent) {
        super.onNewIntent(intent)
        // CRITICAL: Always update the activity's intent reference
        setIntent(intent)
        
        // Process new intent extras
        handleIncomingIntent(intent)
    }

    private fun handleIncomingIntent(intent: Intent) {
        val chatId = intent.getStringExtra(EXTRA_CHAT_ID) ?: "Default Room"
        chatTitleView.text = "Chat Room: $chatId"
    }

    companion object {
        const val EXTRA_CHAT_ID = "extra_chat_id"
    }
}
```

---

## Best Practices
* **Keep configuration in code:** Prefer dynamic flags via `Intent.addFlags()` over static manifest launch modes where possible, as it is more flexible.
* **Always set intent references:** Always execute `setIntent(intent)` inside `onNewIntent(intent)` overrides.
* **Test navigation flows:** Run tests using ADB shell command `adb shell dumpsys activity activities` to verify the back stack structures under different scenarios.
* **Use Safe Stack clearing:** To clear everything and navigate home, use `FLAG_ACTIVITY_NEW_TASK` and `FLAG_ACTIVITY_CLEAR_TASK` combined.

---

## Legacy vs Modern
* **Manifest Launch Modes vs Jetpack Navigation (Modern):** Modern applications rarely use manual manifest launch mode attributes. Instead, they specify back-stack pops (`popUpTo`, `popUpToInclusive`) inside the Jetpack Navigation XML graph or Compose Navigation actions.

---

## Revision Notes
* Stacks are LIFO. Tasks hold back stacks.
* `standard` creates new. `singleTop` reuses the top instance.
* `singleTask` clearing pops all activities resting on top of the target.
* `singleInstance` isolates the activity in its own task.
* `setIntent(intent)` must be called in `onNewIntent(intent)`.

---

## Key Takeaways
* Back stack design directly affects app memory and user experience.
* Launch modes define how activities are reused.
* Use intent flags to clear login and onboarding history stacks safely.

---
---

# Chapter 5: Fragments

## Concept
A **Fragment** represents a reusable portion of an application's User Interface (UI) that lives inside an Activity. Introduced in Android 3.0 (API 11) to support multi-pane and responsive designs on larger tablet screens, Fragments modularize UI code, allowing a single Activity to swap different views dynamically.

### Fragment Lifecycle vs. View Lifecycle
Unlike Activities, Fragments have two distinct lifecycles: the **Fragment Instance Lifecycle** and the **Fragment View Lifecycle**. A Fragment's view can be created and destroyed multiple times while the Fragment instance remains alive (e.g., when the Fragment is placed on the back stack).

#### Lifecycle Callbacks:
1. **`onAttach()`:** Called when the Fragment is first associated with its host Activity context.
2. **`onCreate()`:** Called to initialize the Fragment instance. Non-UI configurations should be loaded here.
3. **`onCreateView()`:** Instantiates and returns the Fragment's view hierarchy.
4. **`onViewCreated()`:** Called immediately after `onCreateView()` returns a non-null View. Setup of listeners, adapters, and view binding belongs here.
5. **`onStart()`:** Makes the Fragment visible.
6. **`onResume()`:** Makes the Fragment active and interactive.
7. **`onPause()`:** Called when the Fragment loses focus or is partially covered.
8. **`onStop()`:** Called when the Fragment is no longer visible.
9. **`onDestroyView()`:** Cleans up resources associated with the View. The View is destroyed, but the Fragment instance remains in memory (e.g., if on the back stack).
10. **`onDestroy()`:** Final cleanup of the Fragment instance.
11. **`onDetach()`:** Detached from the host Activity.

```
                  +--------------------------------+
                  |           onAttach()           |
                  +--------------------------------+
                                  |
                                  v
                  +--------------------------------+
                  |           onCreate()           |
                  +--------------------------------+
                                  |
                                  v
                  +--------------------------------+ <---------+
                  |         onCreateView()         |           |
                  +--------------------------------+           |
                                  |                            | Re-creation of View
                                  v                            | (e.g., returning from
                  +--------------------------------+           |  back stack)
                  |         onViewCreated()        |           |
                  +--------------------------------+           |
                                  |                            |
                                  v                            |
                  +--------------------------------+           |
                  |           onStart()            |           |
                  +--------------------------------+           |
                                  |                            |
                                  v                            |
                  +--------------------------------+           |
                  |           onResume()           |           |
                  +--------------------------------+           |
                                  |                            |
                                  v                            |
                  +--------------------------------+           |
                  |            onPause()           |           |
                  +--------------------------------+           |
                                  |                            |
                                  v                            |
                  +--------------------------------+           |
                  |            onStop()            |           |
                  +--------------------------------+           |
                                  |                            |
                                  v                            |
                  +--------------------------------+ ----------+
                  |         onDestroyView()        |
                  +--------------------------------+
                                  |
                                  v
                  +--------------------------------+
                  |           onDestroy()          |
                  +--------------------------------+
                                  |
                                  v
                  +--------------------------------+
                  |           onDetach()           |
                  +--------------------------------+
```

---

## Why It Exists
Fragments were designed to handle screen size diversity and reduce activity bloating:
* **Responsive Multi-Pane Layouts:** On a phone, list and details screens are shown in separate fullscreen windows. On a tablet, they are displayed side-by-side. Fragments allow encapsulation of list and details logic so they can be arranged dynamically based on screen configurations.
* **Granular Lifecycle Optimization:** Reusing layouts, views, and bindings without destroying the parent Activity state.
* **Navigation Orchestration:** Swapping modular views rapidly in single-activity architectures, improving navigation speed and transition animations.

---

## Internal Working
The management of Fragments is handled by the **`FragmentManager`** and implemented via **`FragmentTransaction`** executions.

### FragmentManager & Host Bindings
The `FragmentManager` executes transactions, maintains the fragment back stack, and acts as the bridge between the Fragment lifecycle and the host Activity:
* **`supportFragmentManager`:** Belongs to the Activity. Manages fragments directly added to the Activity.
* **`childFragmentManager`:** Belongs to a parent Fragment. Manages nested fragments inside that Fragment.

### Fragment Transactions: `add()` vs `replace()`
* **`add(containerId, fragment)`:** Places the new Fragment on top of the container. The existing Fragment inside that container remains in the `RESUMED` state and is visible underneath if the new Fragment has transparency.
* **`replace(containerId, fragment)`:** Removes the existing Fragment from the container (moving it through `onPause -> onStop -> onDestroyView -> onDestroy -> onDetach` unless added to the back stack) and adds the new Fragment.
* **`addToBackStack(name)`:** Saves the transaction state. When the user clicks the back button, the transaction is reversed (e.g., B is destroyed and A is recreated).

### Transaction Commits
* **`commit()`:** Schedules the transaction on the main thread's message queue asynchronously.
* **`commitNow()`:** Executes the transaction immediately on the main thread synchronously. (Cannot be added to back stack).
* **`commitAllowingStateLoss()`:** Like `commit()`, but suppresses the crash that occurs if a commit is executed after the Activity has saved its state.

---

## Lifecycle / Flow

### Lifecycle Syncing between Activity and Fragment

```
Activity Lifecycle State           Fragment Lifecycle Callbacks
========================           ============================
A.onCreate() --------------------> F.onAttach()
                                   F.onCreate()
                                   F.onCreateView()
                                   F.onViewCreated()
A.onStart() ---------------------> F.onStart()
A.onResume() --------------------> F.onResume()
A.onPause() ---------------------> F.onPause()
A.onStop() ----------------------> F.onStop()
A.onDestroy() -------------------> F.onDestroyView()
                                   F.onDestroy()
                                   F.onDetach()
```

---

## Real World Example
In a shopping application, we utilize a single Activity layout containing a `FragmentContainerView`. The user browses products in `ProductListFragment`. When they select an item, we replace it with `ProductDetailFragment`.

Below is a production-level transaction swap including argument passing and state restoration.

```kotlin
// ProductListFragment.kt
package com.example.fitnessapp.presentation

import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import android.widget.Button
import androidx.fragment.app.Fragment
import androidx.fragment.app.commit
import com.example.fitnessapp.R

class ProductListFragment : Fragment() {

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View? {
        return inflater.inflate(R.layout.fragment_product_list, container, false)
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        view.findViewById<Button>(R.id.detailButton).setOnClickListener {
            navigateToDetail("prod_9982")
        }
    }

    private fun navigateToDetail(productId: String) {
        val detailFragment = ProductDetailFragment.newInstance(productId)

        parentFragmentManager.commit {
            setCustomAnimations(
                R.anim.slide_in,
                R.anim.fade_out,
                R.anim.fade_in,
                R.anim.slide_out
            )
            replace(R.id.fragment_container, detailFragment)
            addToBackStack("product_detail")
        }
    }
}
```

---

## Common Mistakes
1. **Instantiating Fragments via Constructors with Parameters:** Adding parameters to a Fragment constructor (e.g., `ProductDetailFragment(productId)`). When the system recreates the Fragment during a configuration change or process death, it calls the default parameterless constructor via reflection, crashing the app or losing the variables.
   * *Fix:* Use a `Bundle` passed to `arguments` via static factory methods.
2. **Leaking View Binding References in `onDestroyView()`:** Failing to clear the binding reference. If a Fragment goes to the back stack, its View is destroyed, but the Fragment instance remains alive. If the binding holds view references, the entire layout hierarchy is leaked.
   * *Fix:* Set the binding variable to `null` inside `onDestroyView()`.
3. **Observing LiveData / Flow using `this` (Fragment Instance) instead of `viewLifecycleOwner`:** Using `this` binds observers to the Fragment instance. If the Fragment's view is recreated, a duplicate observer is registered, causing double emissions.
   * *Fix:* Always pass `viewLifecycleOwner` to `observe()`.
4. **Committing Transactions After `onSaveInstanceState()`:** Committing a transaction after the Activity is backgrounded throws an `IllegalStateException` because the system cannot save the state of the newly added fragment.

---

## Memory Leak / Performance Concerns
* **The View Binding Leak:**
  ```kotlin
  private var _binding: FragmentDetailBinding? = null
  private val binding get() = _binding!!
  // Must clean up in onDestroyView()
  override fun onDestroyView() {
      super.onDestroyView()
      _binding = null
  }
  ```
* **Retaining Fragment Instances unnecessarily:** Setting `setRetainInstance(true)` is completely deprecated. It breaks when combined with ViewModel frameworks and retains obsolete activity contexts.

---

## Interview Questions

### Beginner
1. **What is a Fragment and why was it introduced?**
   * *Answer:* A Fragment is a modular portion of an Activity's user interface and behavior. It was introduced in Android 3.0 to support responsive layout designs across mobile screens and tablets.
2. **What is the difference between `supportFragmentManager` and `childFragmentManager`?**
   * *Answer:* `supportFragmentManager` is bound to the host Activity and manages fragments added directly to the Activity. `childFragmentManager` belongs to a Fragment and is used to manage nested fragments inside that Fragment.

### Intermediate
1. **Compare `add()` and `replace()` operations in fragment transactions.**
   * *Answer:* `add()` places the new Fragment on top of the container, keeping the previous Fragment alive in memory and active. `replace()` destroys the view of the existing Fragment in that container (invoking `onDestroyView()`), replacing it with the new Fragment. If the transaction is added to the back stack, the replaced Fragment's instance is retained.
2. **Why is `onActivityCreated()` deprecated, and what should be used instead?**
   * *Answer:* It was deprecated because it tied view initialization to the Activity's creation state, creating confusion. Developers should use `onViewCreated()` for layout initialization, and `onViewStateRestored()` if state recovery logic is required.

### Advanced
1. **Why does calling `commit()` on a fragment transaction throw an `IllegalStateException` after `onSaveInstanceState()`, and how does `commitAllowingStateLoss()` handle this?**
   * *Answer:* The OS calls `onSaveInstanceState()` to freeze the Activity's component state. If a transaction is committed *after* this, the new fragment's state cannot be saved, risking state loss during recreation. To prevent this, the system throws a crash. `commitAllowingStateLoss()` bypasses this check, but it should be used with caution, as the transaction may not survive process death.
2. **Why is using `viewLifecycleOwner` preferred over `this` when registering observers inside a Fragment?**
   * *Answer:* A Fragment's instance outlives its view (such as when added to the back stack). Observing using `this` attaches the observer to the Fragment instance. When the user returns and the View is recreated, a second observer is added, causing duplicate events. `viewLifecycleOwner` represents the lifespan of the View itself, and auto-destroys observers in `onDestroyView()`.

---

## Scenario Questions
1. **Scenario:** You have a DialogFragment showing an options menu. When the user selects an option, you need to return the value back to the parent Fragment. How do you implement this safely without manual interface bindings?
   * *Answer:* Use the Fragment Result API. Set a listener on the parent fragment using `setFragmentResultListener("requestKey")` and set the result in the DialogFragment using `setFragmentResult("requestKey", bundleOf("key" to value))`.
2. **Scenario:** You have a ViewPager displaying three tabs of fragments. When swipe transitions occur, the views are destroyed and recreated, causing lag. How do you optimize this?
   * *Answer:* You should set the ViewPager's offscreen page limit using `viewPager.offscreenPageLimit = 2`. This keeps the adjacent fragments in the `STARTED`/`RESUMED` states, preventing view recreation overhead.

---

## Code Examples

### Modern Fragment implementation with Safe Binding & Results API
Below is a production-level Fragment demonstrating safe View Binding management, argument parsing, and Fragment Result API communication.

```kotlin
// ProductDetailFragment.kt
package com.example.fitnessapp.presentation

import android.os.Bundle
import android.view.LayoutInflater
import android.view.View
import android.view.ViewGroup
import androidx.core.os.bundleOf
import androidx.fragment.app.Fragment
import androidx.fragment.app.setFragmentResult
import com.example.fitnessapp.databinding.FragmentProductDetailBinding

class ProductDetailFragment : Fragment() {

    // Safe ViewBinding pattern in Fragments
    private var _binding: FragmentProductDetailBinding? = null
    private val binding get() = _binding!!

    private var productId: String? = null

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        arguments?.let {
            productId = it.getString(ARG_PRODUCT_ID)
        }
    }

    override fun onCreateView(
        inflater: LayoutInflater,
        container: ViewGroup?,
        savedInstanceState: Bundle?
    ): View {
        _binding = FragmentProductDetailBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)
        
        binding.productIdText.text = "Product: $productId"

        binding.buyButton.setOnClickListener {
            // Send result back to parent fragment
            setFragmentResult(
                REQUEST_KEY_PURCHASE,
                bundleOf(EXTRA_SUCCESS to true)
            )
            parentFragmentManager.popBackStack()
        }
    }

    override fun onDestroyView() {
        super.onDestroyView()
        // CRITICAL: Prevent memory leak of the view binding
        _binding = null
    }

    companion object {
        private const val ARG_PRODUCT_ID = "arg_product_id"
        const val REQUEST_KEY_PURCHASE = "request_key_purchase"
        const val EXTRA_SUCCESS = "extra_success"

        fun newInstance(productId: String): ProductDetailFragment {
            return ProductDetailFragment().apply {
                arguments = bundleOf(ARG_PRODUCT_ID to productId)
            }
        }
    }
}
```

---

## Best Practices
* **Use `FragmentContainerView`:** Always use `FragmentContainerView` as the container for fragments in your layouts instead of `FrameLayout`. It resolves animation z-ordering issues.
* **Always Nullify Bindings:** Always set `_binding = null` inside `onDestroyView()`.
* **Use Safe Args:** Utilize the Safe Args Gradle plugin to pass compile-time safe navigation arguments.
* **Encrypt Fragment Communication:** Use shared ViewModels or the official Fragment Results API for communication rather than custom interface implementations.

---

## Legacy vs Modern
* **`setRetainInstance(true)` vs ViewModel (Modern):** In the past, developers used `setRetainInstance(true)` to keep fragment instances alive during rotation. This is deprecated; you must use Jetpack ViewModel to survive configuration changes.
* **Interface Callbacks vs Fragment Result API (Modern):** Legacy code communicated between fragments via Activity interface casting. The modern approach uses the decoupled `setFragmentResultListener` framework.
* **`<fragment>` tag vs `FragmentContainerView` (Modern):** Statically inflating fragments via the `<fragment>` XML tag makes them irreplaceable at runtime. `FragmentContainerView` supports both static and dynamic operations cleanly.

---

## Revision Notes
* Fragments have two distinct lifecycles: the instance lifecycle and the view lifecycle.
* Clean up binding references in `onDestroyView()`.
* Observe flows/LiveData using `viewLifecycleOwner` to prevent duplicate observers.
* Use `replace()` to swap fragments and clean up memory; use `addToBackStack()` to save state.
* Communicate between fragments using Fragment Results API or shared ViewModels.

---

## Key Takeaways
* Fragment View destruction does not mean the Fragment instance is dead.
* Reflection-based recreation requires all Fragments to have a default parameterless constructor.
* Never commit fragment transactions after state saving to avoid crashes.


