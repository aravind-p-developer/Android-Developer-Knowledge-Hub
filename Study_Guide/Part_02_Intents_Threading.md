# Android Fundamentals Master Guide
# Part 2: Intents, Threading & Services (Chapters 6–8)

> **Series:** Android Fundamentals Master Guide  
> **Part:** 2 of N  
> **Chapters:** 6 – Intents & Intent Filters | 7 – Threading | 8 – Services & Process Lifecycle  
> **Level:** Beginner → Advanced  
> **Last Updated:** June 2026

---

## Table of Contents

- [Chapter 6: Intents & Intent Filters](#chapter-6-intents--intent-filters)
- [Chapter 7: Threading — UI Thread, Looper, Handler, Coroutines](#chapter-7-threading--ui-thread-looper-handler-coroutines)
- [Chapter 8: Services & Process Lifecycle](#chapter-8-services--process-lifecycle)

---

# Chapter 6: Intents & Intent Filters

---

## Concept

An **Intent** is Android's fundamental inter-component message-passing mechanism. It is an abstract description of an operation to be performed — a data structure that carries both the *what* (action, data) and optionally the *who* (target component). Intents are how Android decouples components from each other and from the broader ecosystem: an Activity does not need to know that a browser exists — it simply declares the intent to open a URL, and the system resolves the appropriate handler.

There are two fundamental kinds of Intents:

| Kind | Resolution | Use Case |
|------|-----------|----------|
| **Explicit** | Caller specifies the exact target component class | Navigating between your own Activities/Services |
| **Implicit** | Caller declares an action; system resolves the best handler | Open URL, share content, take photo, pick file |

An Intent object is a bundle of information. Its primary fields are:

| Field | Description | Example |
|-------|-------------|---------|
| `action` | String constant describing the operation | `Intent.ACTION_VIEW` |
| `data` | URI identifying the data to act on | `Uri.parse("https://…")` |
| `type` | MIME type of the data | `"image/*"` |
| `category` | Metadata string refining intent semantics | `Intent.CATEGORY_DEFAULT` |
| `extras` | `Bundle` of key-value pairs for additional data | `putExtra("key", value)` |
| `flags` | Integer bitmask controlling launch behaviour | `FLAG_ACTIVITY_NEW_TASK` |
| `component` | Explicit target `ComponentName` (makes it explicit) | `ComponentName(pkg, cls)` |

An **IntentFilter** is the manifest-side declaration that tells the system which implicit Intents a component is willing to handle. It is a whitelist: a component only receives an implicit Intent if the Intent passes all three of the IntentFilter's tests — action, category, and data.

---

## Why It Exists

Android's security model and component architecture demand loose coupling. A single application cannot anticipate every possible way the user might want to complete a task (share a photo, choose a ringtone, open a PDF). If components were tightly coupled by class name, every app would need to ship its own browser, image viewer, and file picker. This would waste space and fragment the user experience.

Intents solve this through **late binding**: the association between a caller's intent and the concrete handler is made at runtime by the `ActivityManager`, not at compile time. This enables:

1. **Reuse of system components** — one Camera app serves every third-party app.
2. **User choice** — the user picks their preferred browser, mail client, or sharing target.
3. **Sandboxed IPC** — the system intermediates all cross-process component invocations, applying permission checks before dispatch.
4. **Deferred execution** — `PendingIntent` allows a trusted third party (the system, a notification, a widget) to fire an Intent on behalf of your app at a future time, with your app's identity and permissions.

---

## Internal Working

### Explicit Intent Resolution

```
startActivity(Intent(this, DetailActivity::class.java))
         │
         ▼
  ActivityManagerService (AMS) in system_server
         │
         ├─► Validates the component exists in the package's manifest
         ├─► Checks android:exported if caller is a different app (API 31+)
         ├─► Applies permission checks (android:permission attribute)
         └─► Instructs Zygote / Process manager to start/resume the Activity
```

AMS is the central authority. The call crosses a Binder IPC boundary (your app → system_server). AMS then issues another Binder call back into your app's process to deliver `onCreate()` / `onNewIntent()`.

### Implicit Intent Resolution (Intent Matching)

The system evaluates every installed component's `<intent-filter>` declarations:

```
Intent (ACTION_VIEW, data=http://…, category=DEFAULT)
         │
         ▼
  PackageManagerService scans installed manifests
         │
         ├─► Action test:   Intent.action ∈ filter's <action> list?
         ├─► Category test: ALL Intent categories ⊆ filter's <category> list?
         │                  (CATEGORY_DEFAULT implicitly required for startActivity)
         └─► Data test:     URI scheme/host/path AND MIME type match?
                │
                ├─► Zero matches  → ActivityNotFoundException
                ├─► One match     → Launch it
                └─► Many matches  → Disambiguation dialog (chooser)
```

**Key subtlety — CATEGORY_DEFAULT:** When you call `startActivity()` with an implicit Intent, the system automatically adds `CATEGORY_DEFAULT` to the Intent. Therefore, every `<intent-filter>` that is meant to respond to implicit `startActivity()` calls **must** include `<category android:name="android.intent.category.DEFAULT"/>`. Without it, the filter will never match.

### Data Matching Rules

The data test is the most complex. The filter must match:
- **scheme** (e.g., `http`, `content`, `file`)
- **authority** (`host` + optional `port`)
- **path** (`pathPrefix`, `path`, `pathPattern`)
- **mimeType**

A filter can declare data or type or both. The Intent must be consistent with *all* declared constraints. Partial declaration means the constraint is unchecked.

### PendingIntent Internals

A `PendingIntent` is a token referencing a frozen `Intent` stored in system_server. The system keeps a reference to the original app's identity. When a notification or widget triggers the `PendingIntent`, AMS fires the stored Intent with the originating app's permissions — not the executor's. Android 12+ requires either `FLAG_MUTABLE` or `FLAG_IMMUTABLE` to be explicitly set.

---

## Lifecycle / Flow (ASCII Diagrams)

### Explicit Intent Navigation Flow

```
App Process A                       system_server                    App Process A
──────────────                     ──────────────                   ──────────────
startActivity(intent)
      │
      └──── Binder IPC ──────────► AMS.startActivity()
                                         │
                                   Validate component
                                   Check permissions
                                   Check android:exported
                                         │
                                   Resume / create task
                                         │
                                   Binder IPC ◄──────────────────── ActivityThread
                                         │
                                         └─── scheduleTransaction()
                                                    │
                                              ┌─────▼──────┐
                                              │  onCreate  │
                                              │  onStart   │
                                              │  onResume  │
                                              └────────────┘
```

### Implicit Intent Resolution Flow

```
Caller                    PackageManager              System
──────                   ───────────────            ───────
createIntent(ACTION_SEND)
      │
      ▼
  resolveActivity(pm)  ──► scan all <intent-filter>
      │                          │
      │                    action match?
      │                    category match?
      │                    data match?
      │                          │
      │                   ┌──────┴──────┐
      │                 0 match       1+ match
      │                   │              │
      │               return null   1 match → launch directly
      │                              N match → chooser dialog
      │
  if null → don't call startActivity (avoids crash)
```

### Activity Result API Flow

```
Caller Activity                        Target Activity
──────────────                        ───────────────
registerForActivityResult(
  ActivityResultContracts.StartActivityForResult()
) { result →
    // handle result
}
      │
      ▼
launcher.launch(intent) ──────────────► onCreate()
                                              │
                                        user interaction
                                              │
                                    setResult(RESULT_OK, data)
                                         finish()
                                              │
                         ◄────────────────────┘
result callback fires on
calling Activity's main thread
```

---

## Real World Example

A social media app wants to let users share a post image. It does not know (or care) which app the user prefers — Gmail, WhatsApp, Telegram, or a custom SMS app. Using an implicit Intent with `ACTION_SEND`, the system presents all candidates. This is a textbook application of the Open/Closed Principle at the OS level: the social media app is open for extension (new sharing targets) without modification.

Another example: a file manager app declares an `<intent-filter>` for `ACTION_VIEW` with `mimeType="application/pdf"`. When any app in the system tries to open a PDF, the file manager can surface as a handler — even if the PDF viewer was installed years after the file manager.

---

## Common Mistakes

### 1. Not Checking `resolveActivity()` Before Starting Implicit Intents

```kotlin
// ❌ WRONG — crashes with ActivityNotFoundException if no handler exists
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("geo:0,0?q=Eiffel+Tower"))
startActivity(intent)

// ✅ CORRECT
val intent = Intent(Intent.ACTION_VIEW, Uri.parse("geo:0,0?q=Eiffel+Tower"))
if (intent.resolveActivity(packageManager) != null) {
    startActivity(intent)
} else {
    Toast.makeText(this, "No Maps app found", Toast.LENGTH_SHORT).show()
}
```

### 2. Missing `<queries>` in Manifest (Android 11+ / API 30+)

Android 11 introduced **Package Visibility** restrictions. Apps can no longer query all installed packages by default. `resolveActivity()` returns `null` on API 30+ unless you declare the intent filter you are querying in `<queries>`.

```xml
<!-- ❌ Will return null on API 30+ without this -->
<!-- ✅ Add to AndroidManifest.xml -->
<queries>
    <intent>
        <action android:name="android.intent.action.VIEW" />
        <data android:scheme="geo" />
    </intent>
    <intent>
        <action android:name="android.media.action.IMAGE_CAPTURE" />
    </intent>
</queries>
```

### 3. Forgetting `android:exported` (API 31+)

Any Activity, Service, or BroadcastReceiver with an `<intent-filter>` must explicitly declare `android:exported="true"` or `"false"` on API 31+. Omitting this causes an `IllegalStateException` at install time.

### 4. Passing Non-Parcelable Objects via Extras

Passing large objects as `Serializable` is slow (uses Java reflection). Passing objects that aren't serializable at all will throw a `RuntimeException` when the Bundle is unparcelled.

### 5. Using `startActivityForResult()` (Deprecated)

The legacy `startActivityForResult()` is fragile — the `requestCode` must be managed manually, and the callback in `onActivityResult()` requires boilerplate switch logic. The modern `Activity Result API` is composable, testable, and lifecycle-safe.

---

## Memory Leak / Performance Concerns

### Implicit Intents and Security

Implicit Intents sent via `sendBroadcast()` are inherently insecure — any app can register a matching `BroadcastReceiver` and intercept the Intent. Use `LocalBroadcastManager` (deprecated but still common in legacy code) or the `permissions` parameter in `sendBroadcast()`.

### PendingIntent Leaks

A `PendingIntent` holds a reference to a `Context`. If you store a `PendingIntent` created with an `Activity` context in a notification and the Activity is destroyed, the notification keeps the Activity alive. Always use `applicationContext` when creating `PendingIntent`s for long-lived objects like notifications.

```kotlin
// ❌ Activity context — can leak
val pi = PendingIntent.getActivity(this, 0, intent, PendingIntent.FLAG_IMMUTABLE)

// ✅ Application context
val pi = PendingIntent.getActivity(
    applicationContext, 0, intent, PendingIntent.FLAG_IMMUTABLE
)
```

### Extra Bundle Size

`Intent` extras are transported via Binder transactions. Binder's transaction buffer is 1 MB shared across all transactions in the process. Putting large bitmaps or data structures in extras can cause `TransactionTooLargeException`. Prefer passing an ID and loading data from a repository/database in the target component.

---

## Interview Questions

### Beginner

**Q1: What is the difference between an explicit and an implicit Intent?**

> An explicit Intent specifies the exact target component by class name (e.g., `Intent(this, DetailActivity::class.java)`). The system routes it directly without any resolution step. An implicit Intent only declares an *action* and optional *data* (e.g., `Intent(Intent.ACTION_VIEW, uri)`). The system resolves the best handler at runtime by comparing the Intent against all installed components' `<intent-filter>` declarations.

**Q2: What is `CATEGORY_DEFAULT` and why is it required?**

> When you call `startActivity()` with an implicit Intent, Android automatically adds `CATEGORY_DEFAULT` to the Intent object. An `<intent-filter>` must therefore include `<category android:name="android.intent.category.DEFAULT"/>` to be considered as a candidate for `startActivity()`. Without this category in the filter, the component will never match implicit Intents launched via `startActivity()`.

**Q3: What is a PendingIntent?**

> A `PendingIntent` is a token referencing an Intent stored inside `system_server`. It allows another application or the system itself (e.g., the notification system, an alarm) to execute the wrapped Intent on behalf of your application — with your app's permissions and identity — at a future time, even if your app is no longer running.

---

### Intermediate

**Q4: How does the Intent resolution process work internally?**

> The `PackageManagerService` scans the manifest-declared `<intent-filter>` entries of all installed packages. For an implicit Intent, it applies three tests in sequence: (1) **Action test** — the Intent's action must appear in the filter's `<action>` list; (2) **Category test** — every category in the Intent must appear in the filter's `<category>` list; (3) **Data test** — the URI scheme, authority, path, and MIME type must all match the filter's `<data>` constraints. A component is a candidate only if it passes all three tests.

**Q5: What changed with Package Visibility in Android 11 (API 30)?**

> Android 11 introduced restrictions on which packages an app can see. By default, an app can only see packages it directly interacts with (same UID, declared dependencies, etc.). Calling `resolveActivity()` or `queryIntentActivities()` for arbitrary Intents returns `null` / an empty list unless the app declares matching `<queries>` elements in its manifest, specifying the packages, authorities, or intent filters it needs to query.

**Q6: Compare `Parcelable` and `Serializable` for passing objects in Intents.**

> `Serializable` uses Java reflection to traverse the object graph at runtime — it is slow, produces more garbage, and includes class metadata in the byte stream, making the payload larger. `Parcelable` is Android's custom binary serialization protocol. Objects implement it manually (or via the `@Parcelize` Kotlin plugin), writing/reading their fields directly with typed methods. This is 10–100x faster than `Serializable` and produces a smaller, tighter payload. For Intents, always prefer `Parcelable` for objects passed as extras.

**Q7: What is the Activity Result API and why was it introduced?**

> The Activity Result API (`registerForActivityResult`) was introduced in AndroidX Activity 1.2 to replace `startActivityForResult()` + `onActivityResult()`. The legacy API had no lifecycle awareness (you could register a callback after the Activity was already destroyed), required manual `requestCode` management, and was difficult to test. The new API uses an `ActivityResultLauncher` registered during `onCreate` (before the Activity is STARTED), is lifecycle-aware, is type-safe via `ActivityResultContract` subtypes, and is easily mockable in unit tests.

---

### Advanced

**Q8: Explain the security implications of implicit Intents for broadcast.**

> Implicit broadcasts are sent to all registered receivers matching the filter — across all apps. A malicious app can register a receiver for your custom implicit broadcast and intercept sensitive data. Mitigations: (1) Use `LocalBroadcastManager` or `Flow`/`EventBus` for in-process communication. (2) Pass a `receiverPermission` string to `sendBroadcast()` — only apps holding that permission can receive it. (3) Use `sendOrderedBroadcast()` with a result receiver to detect interception. (4) For API 26+ background receivers, most implicit broadcasts are blocked anyway; use explicit Intents or `JobScheduler`.

**Q9: How does `FLAG_ACTIVITY_NEW_TASK` interact with task affinity, and when does it create a new task vs. reusing an existing one?**

> `FLAG_ACTIVITY_NEW_TASK` instructs AMS to start the Activity in a task whose *task affinity* matches the Activity's own `android:taskAffinity` attribute (defaults to the app's package name). If a task with that affinity already exists in the recents stack, the system brings that task to the foreground and delivers the Intent via `onNewIntent()` — it does **not** always create a brand-new task. A truly new task is created only if no matching-affinity task exists. Combining `FLAG_ACTIVITY_NEW_TASK` with `FLAG_ACTIVITY_CLEAR_TOP` clears all Activities on top of the matching Activity in the existing task, delivering the Intent freshly.

**Q10: How does a PendingIntent survive app process death, and what are `FLAG_UPDATE_CURRENT` and `FLAG_IMMUTABLE`?**

> The `PendingIntent` token is stored in `system_server`, not in your app's process. When your process dies, the token remains valid. When the system fires it, it re-launches your app (if necessary) to handle the Intent. `FLAG_UPDATE_CURRENT` tells the system: if a `PendingIntent` with the same operation, action, data, and request code already exists, *replace its extras* with the new Intent's extras while keeping the token. `FLAG_IMMUTABLE` (mandatory from API 31) prevents the receiving component from modifying the Intent's extras before it is dispatched — critical for security (prevents extra injection attacks). `FLAG_MUTABLE` is only needed when the system must fill in part of the Intent (e.g., inline reply in notifications).

---

## Scenario Questions

**Scenario 1:** Your app shares images via `ACTION_SEND`. On Android 11 devices, `resolveActivity()` returns `null` and the share sheet never appears. What is wrong and how do you fix it?

> **Answer:** Android 11's Package Visibility restrictions prevent `resolveActivity()` from seeing sharing targets without a `<queries>` declaration. The fix is to add an intent query in the manifest. However, for the share sheet specifically, `createChooser()` is always the right API. The intent passed to `createChooser()` itself is resolved by the system chooser process, which has visibility into all packages regardless of the caller's `<queries>`. Switch from manual `resolveActivity()` + `startActivity()` to `startActivity(Intent.createChooser(sendIntent, "Share via"))`. For cases where you genuinely need `resolveActivity()`, add the appropriate `<queries>` entry.

**Scenario 2:** You launch a `CameraActivity` using the modern Activity Result API to capture a photo. On some low-RAM devices, the camera returns `RESULT_CANCELED` instead of `RESULT_OK`. The user did take the photo. Why?

> **Answer:** On low-RAM devices, the system may kill your app's process while the camera app is in the foreground (your app is in the background with no visible UI). When the camera returns, the system recreates your Activity from scratch using the saved instance state. However, if the `ActivityResultLauncher` is not registered during `onCreate()` before the Activity reaches the `STARTED` state, the result delivery window is missed and the Activity receives `RESULT_CANCELED`. The fix: always register `ActivityResultLauncher` as a class-level field using `registerForActivityResult()` called unconditionally in `onCreate()`, not lazily inside a click listener.

**Scenario 3:** You receive a `TransactionTooLargeException` when passing a `Bitmap` via an Intent extra. What is the correct architectural solution?

> **Answer:** Binder's transaction buffer (1 MB, shared across all concurrent transactions in the process) cannot accommodate large bitmaps. The correct solution is the **ID pattern**: save the bitmap to a temporary file or database in the source component, pass only the file URI or database row ID via the Intent extra, and load the bitmap from storage in the destination component. For sharing bitmaps with *other* apps, use a `FileProvider` and pass a `content://` URI with `FLAG_GRANT_READ_URI_PERMISSION` — do not pass bitmap bytes directly.

---

## Code Examples (Production-Quality Kotlin)

### Example 1: Explicit Intent Navigation with Parcelable Data

```kotlin
// --- Data class using @Parcelize ---
// Requires: plugins { id("kotlin-parcelize") } in build.gradle

import android.os.Parcelable
import kotlinx.parcelize.Parcelize

@Parcelize
data class Product(
    val id: String,
    val name: String,
    val priceInCents: Long,
    val imageUrl: String
) : Parcelable

// --- Navigating to ProductDetailActivity ---

class ProductListActivity : AppCompatActivity() {

    companion object {
        // Typed factory method — the canonical pattern for Activity extras
        fun createIntent(context: Context, product: Product): Intent =
            Intent(context, ProductDetailActivity::class.java).apply {
                putExtra(ProductDetailActivity.EXTRA_PRODUCT, product)
            }
    }

    private fun openProductDetail(product: Product) {
        startActivity(createIntent(this, product))
    }
}

// --- Receiving the Parcelable in the target Activity ---

class ProductDetailActivity : AppCompatActivity() {

    companion object {
        const val EXTRA_PRODUCT = "extra_product"
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_product_detail)

        // API 33+: use the typed getParcelableExtra overload
        val product: Product? = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
            intent.getParcelableExtra(EXTRA_PRODUCT, Product::class.java)
        } else {
            @Suppress("DEPRECATION")
            intent.getParcelableExtra(EXTRA_PRODUCT)
        }

        product?.let { render(it) } ?: finish() // guard against missing data
    }

    private fun render(product: Product) {
        // Bind data to views
    }
}
```

### Example 2: Implicit Intents — URL, Share, Camera

```kotlin
class ImplicitIntentExamples(private val context: Context) {

    // ─── Open a URL in a browser ───────────────────────────────────────────
    fun openUrl(url: String) {
        val uri = Uri.parse(url)
        val intent = Intent(Intent.ACTION_VIEW, uri)
        if (intent.resolveActivity(context.packageManager) != null) {
            context.startActivity(intent)
        } else {
            Log.w("Intent", "No browser available to handle $url")
        }
    }

    // ─── Share plain text ──────────────────────────────────────────────────
    fun shareText(text: String, chooserTitle: String = "Share via") {
        val sendIntent = Intent(Intent.ACTION_SEND).apply {
            type = "text/plain"
            putExtra(Intent.EXTRA_TEXT, text)
            putExtra(Intent.EXTRA_SUBJECT, "Check this out!")
        }
        // createChooser always shows the system share sheet —
        // no resolveActivity() check needed here.
        context.startActivity(Intent.createChooser(sendIntent, chooserTitle))
    }

    // ─── Share an image via FileProvider ──────────────────────────────────
    fun shareImage(imageFile: File) {
        val uri = FileProvider.getUriForFile(
            context,
            "${context.packageName}.provider",
            imageFile
        )
        val sendIntent = Intent(Intent.ACTION_SEND).apply {
            type = "image/jpeg"
            putExtra(Intent.EXTRA_STREAM, uri)
            addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
        }
        context.startActivity(Intent.createChooser(sendIntent, "Share Image"))
    }

    // ─── Open device dialer ───────────────────────────────────────────────
    fun dialPhone(phoneNumber: String) {
        val intent = Intent(Intent.ACTION_DIAL, Uri.parse("tel:$phoneNumber"))
        // ACTION_DIAL only opens the dialer pre-filled; no CALL_PHONE permission needed
        if (intent.resolveActivity(context.packageManager) != null) {
            context.startActivity(intent)
        }
    }
}
```

### Example 3: Activity Result API (Modern Replacement for `startActivityForResult`)

```kotlin
class ProfileActivity : AppCompatActivity() {

    // ─── Register launcher at class level, before onCreate completes ───────
    // This is the critical requirement: registration must happen before STARTED state.
    private val pickImageLauncher = registerForActivityResult(
        ActivityResultContracts.PickVisualMedia()
    ) { uri: Uri? ->
        if (uri != null) {
            // Persist read permission across app restarts (required for content URIs)
            contentResolver.takePersistableUriPermission(
                uri, Intent.FLAG_GRANT_READ_URI_PERMISSION
            )
            viewModel.onImageSelected(uri)
        } else {
            Log.d("ProfileActivity", "User cancelled image picker")
        }
    }

    // ─── Capture a photo from camera ──────────────────────────────────────
    private lateinit var photoUri: Uri

    private val cameraLauncher = registerForActivityResult(
        ActivityResultContracts.TakePicture()
    ) { success: Boolean ->
        if (success) {
            viewModel.onPhotoTaken(photoUri)
        }
    }

    // ─── Request a permission ──────────────────────────────────────────────
    private val permissionLauncher = registerForActivityResult(
        ActivityResultContracts.RequestPermission()
    ) { isGranted: Boolean ->
        if (isGranted) {
            launchCamera()
        } else {
            showPermissionRationale()
        }
    }

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_profile)
        // Launchers are already registered; just wire up UI actions
    }

    fun onPickImageClicked() {
        // Photo Picker API — no permission required on Android 13+
        pickImageLauncher.launch(
            PickVisualMediaRequest(ActivityResultContracts.PickVisualMedia.ImageOnly)
        )
    }

    fun onTakePhotoClicked() {
        when {
            ContextCompat.checkSelfPermission(
                this, Manifest.permission.CAMERA
            ) == PackageManager.PERMISSION_GRANTED -> launchCamera()

            shouldShowRequestPermissionRationale(Manifest.permission.CAMERA) ->
                showPermissionRationale()

            else -> permissionLauncher.launch(Manifest.permission.CAMERA)
        }
    }

    private fun launchCamera() {
        photoUri = createTempImageUri()
        cameraLauncher.launch(photoUri)
    }

    private fun createTempImageUri(): Uri {
        val imageFile = File(cacheDir, "captured_${System.currentTimeMillis()}.jpg")
        return FileProvider.getUriForFile(this, "$packageName.provider", imageFile)
    }

    private fun showPermissionRationale() {
        // Show rationale UI
    }
}
```

### Example 4: Receiving Shared Files (as a Share Target)

```xml
<!-- AndroidManifest.xml -->
<activity
    android:name=".ShareReceiverActivity"
    android:exported="true"
    android:label="My App">
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="image/*" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.SEND" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="text/plain" />
    </intent-filter>
    <intent-filter>
        <action android:name="android.intent.action.SEND_MULTIPLE" />
        <category android:name="android.intent.category.DEFAULT" />
        <data android:mimeType="image/*" />
    </intent-filter>
</activity>
```

```kotlin
class ShareReceiverActivity : AppCompatActivity() {

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_share_receiver)

        handleIncomingIntent(intent)
    }

    private fun handleIncomingIntent(intent: Intent) {
        when {
            intent.action == Intent.ACTION_SEND && intent.type == "text/plain" -> {
                val sharedText = intent.getStringExtra(Intent.EXTRA_TEXT)
                sharedText?.let { handleIncomingText(it) }
            }

            intent.action == Intent.ACTION_SEND && intent.type?.startsWith("image/") == true -> {
                val imageUri: Uri? = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                    intent.getParcelableExtra(Intent.EXTRA_STREAM, Uri::class.java)
                } else {
                    @Suppress("DEPRECATION")
                    intent.getParcelableExtra(Intent.EXTRA_STREAM)
                }
                imageUri?.let { handleIncomingImage(it) }
            }

            intent.action == Intent.ACTION_SEND_MULTIPLE &&
                    intent.type?.startsWith("image/") == true -> {
                val imageUris: List<Uri>? = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                    intent.getParcelableArrayListExtra(Intent.EXTRA_STREAM, Uri::class.java)
                } else {
                    @Suppress("DEPRECATION")
                    intent.getParcelableArrayListExtra(Intent.EXTRA_STREAM)
                }
                imageUris?.let { handleIncomingImages(it) }
            }

            else -> finish() // unsupported type — close gracefully
        }
    }

    private fun handleIncomingText(text: String) {
        // Populate a compose/post field with the shared text
    }

    private fun handleIncomingImage(uri: Uri) {
        // Read the image — the URI is a content:// URI granted to us temporarily.
        // Use ContentResolver to open an InputStream.
        contentResolver.openInputStream(uri)?.use { stream ->
            // Process stream
        }
    }

    private fun handleIncomingImages(uris: List<Uri>) {
        uris.forEach { handleIncomingImage(it) }
    }
}
```

### Example 5: PendingIntent for Notifications (Production Pattern)

```kotlin
class NotificationHelper(private val context: Context) {

    companion object {
        const val CHANNEL_ID = "order_updates"
        const val NOTIFICATION_ID = 1001
    }

    fun createChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "Order Updates",
                NotificationManager.IMPORTANCE_DEFAULT
            ).apply {
                description = "Notifications about your order status"
                enableVibration(true)
            }
            val manager = context.getSystemService(NotificationManager::class.java)
            manager.createNotificationChannel(channel)
        }
    }

    fun showOrderNotification(orderId: String, message: String) {
        // Always use applicationContext for PendingIntents in notifications
        val deepLinkIntent = Intent(context, OrderDetailActivity::class.java).apply {
            putExtra(OrderDetailActivity.EXTRA_ORDER_ID, orderId)
            // Ensures back-stack is correct when launched from notification
            flags = Intent.FLAG_ACTIVITY_NEW_TASK or Intent.FLAG_ACTIVITY_CLEAR_TOP
        }

        val pendingIntent = PendingIntent.getActivity(
            context,
            orderId.hashCode(), // unique requestCode per order
            deepLinkIntent,
            PendingIntent.FLAG_UPDATE_CURRENT or PendingIntent.FLAG_IMMUTABLE
        )

        val notification = NotificationCompat.Builder(context, CHANNEL_ID)
            .setSmallIcon(R.drawable.ic_order)
            .setContentTitle("Order #$orderId")
            .setContentText(message)
            .setContentIntent(pendingIntent)
            .setAutoCancel(true) // dismiss on tap
            .build()

        NotificationManagerCompat.from(context).notify(NOTIFICATION_ID, notification)
    }
}
```

---

## Best Practices

1. **Always check `resolveActivity()` before starting implicit Intents.** Wrap it in an extension function for reuse.
2. **Add `<queries>` in the manifest** for all implicit Intents you plan to resolve (API 30+).
3. **Declare `android:exported`** explicitly for all components with `<intent-filter>` (API 31+).
4. **Use `@Parcelize`** for all data classes passed between components — never use raw `Serializable`.
5. **Pass IDs, not objects** across process boundaries. Load data from a shared repository in the destination.
6. **Use `registerForActivityResult()`** instead of `startActivityForResult()`. Register launchers as class-level fields, initialized before `onCreate()` completes.
7. **Use `applicationContext`** when creating `PendingIntent`s tied to long-lived objects (notifications, widgets, alarms).
8. **Use `FLAG_IMMUTABLE`** for `PendingIntent`s whenever the system does not need to modify the Intent's extras.
9. **Use `createChooser()`** for share Intents — it always shows the system chooser and handles the API 30+ visibility restrictions for the share sheet.
10. **Use `FileProvider`** for sharing files — never expose raw `file://` URIs to other apps (blocked since API 24 via `FileUriExposedException`).

---

## Legacy vs Modern

| Concern | Legacy | Modern |
|---------|--------|--------|
| Launching Activity for result | `startActivityForResult()` + `onActivityResult()` | `registerForActivityResult()` with `ActivityResultContract` |
| Picking images | `ACTION_PICK` / `ACTION_GET_CONTENT` with permissions | `PickVisualMedia` contract (no permission needed on API 33+) |
| Parcelable boilerplate | Manual `writeToParcel()` / `createFromParcel()` | `@Parcelize` annotation (kotlin-parcelize plugin) |
| Passing objects between apps | `Serializable` | `Parcelable` with `@Parcelize` |
| Sharing content | `file://` URIs | `FileProvider` with `content://` URIs + `FLAG_GRANT_READ_URI_PERMISSION` |
| In-process event broadcasting | `LocalBroadcastManager` (deprecated) | `SharedFlow` / `StateFlow` / `EventBus` |

---

## Revision Notes

- **Intent = message envelope; IntentFilter = whitelist on the receiver side.**
- Implicit intents go through `PackageManagerService` for resolution; explicit intents bypass this.
- `CATEGORY_DEFAULT` is auto-added by `startActivity()` — your filter must include it.
- On API 30+, add `<queries>` or `resolveActivity()` returns null.
- On API 31+, all exported components with filters must explicitly set `android:exported`.
- Binder limit ≈ 1 MB — never pass large bitmaps/data via extras; use IDs + repository.
- `PendingIntent` lives in system_server, survives process death; use `FLAG_IMMUTABLE` + `applicationContext`.
- `registerForActivityResult` launchers must be registered *before* the Activity reaches STARTED state.

---

## Key Takeaways

> **Intent is Android's universal connector** — the lingua franca between components, apps, and the system.

- Two types: **Explicit** (direct, by class name) and **Implicit** (action-based, resolved at runtime).
- The system resolves implicit Intents using a three-part filter test: action → category → data.
- `CATEGORY_DEFAULT` is the invisible handshake between `startActivity()` and `<intent-filter>`.
- Prefer `Parcelable` (@Parcelize) over `Serializable` for object transport.
- The Activity Result API is the modern, lifecycle-safe, testable replacement for `startActivityForResult`.
- `PendingIntent` is a time-capsule: it stores your app's Intent and identity in system_server for later execution.
- Security: use `FileProvider` for files, `FLAG_IMMUTABLE` for PendingIntents, and `<queries>` for visibility.

---
---

# Chapter 7: Threading — UI Thread, Looper, Handler, Coroutines

---

## Concept

Android is a **single-threaded UI framework**. All UI rendering, touch event dispatch, and lifecycle callbacks are executed on a single thread called the **Main Thread** (also called the **UI Thread**). This is not a limitation but a deliberate design choice: UI toolkits that allow concurrent access to view hierarchies require pervasive locking, which is expensive, error-prone, and leads to deadlocks. Android instead enforces a contract: *only the main thread may touch the UI*.

This single-thread model immediately creates a problem: any blocking operation on the main thread (network call, database query, file I/O) stalls UI rendering and event processing. Android's answer is a rich threading infrastructure:

| Mechanism | Era | Purpose |
|-----------|-----|---------|
| `Thread` + `runOnUiThread()` | API 1+ | Raw threads; no lifecycle awareness |
| `Looper` + `Handler` | API 1+ | Message queue per thread; foundation for all Android async |
| `HandlerThread` | API 1+ | Thread with a built-in `Looper`; managed message queue |
| `AsyncTask` | API 3–29 | ⚠️ **Deprecated API 30** — simple background + UI thread callback |
| `Kotlin Coroutines` | 2018+ | Structured concurrency; lifecycle-aware; production standard |
| `WorkManager` | 2018+ | Guaranteed, deferrable background work; survives process death |

### What is ANR?

**ANR (Application Not Responding)** is triggered by the system when:
- An Activity does not respond to a user input event within **5 seconds**.
- A `BroadcastReceiver` does not finish `onReceive()` within **10 seconds**.
- A `Service` does not respond within **20 seconds** (for foreground services with `startForeground` on Android 11+: stricter limits apply).

The system displays an ANR dialog offering the user the option to "Wait" or "Close App". ANRs are fatal to user experience and are tracked by Google Play's Android Vitals. All blocking I/O **must** happen off the main thread.

---

## Why It Exists

The Looper/Handler system predates Kotlin and coroutines by over a decade. It exists because:

1. **Java's raw `Thread` API has no concept of a message queue.** Posting work to a specific thread required custom synchronization. `Looper` provides a standardized, efficient, lock-free message queue per thread.
2. **UI updates must be posted back to the main thread.** Background work threads need a safe mechanism to schedule UI updates. `Handler(Looper.getMainLooper())` provides exactly this.
3. **Android's own framework** — `View.post()`, `Activity.runOnUiThread()`, `ViewRootImpl`, `WifiStateMachine`, `BluetoothService` — are all built on top of `Looper` and `Handler`. Understanding them means understanding the entire Android runtime.

Kotlin Coroutines exist because:
1. **`AsyncTask` was fundamentally broken** — it leaked Contexts, was not cancellable cleanly, and mixed concerns.
2. **Callbacks (Retrofit, RxJava)** lead to "callback hell" and are hard to reason about for error handling.
3. **Structured concurrency** allows async code to look sequential, integrates with lifecycle automatically via `lifecycleScope` / `viewModelScope`, and handles cancellation/error propagation correctly.

---

## Internal Working

### The Looper / MessageQueue / Handler Architecture

Every Android thread that processes messages has a `Looper`. The Looper owns a `MessageQueue` — a priority queue (ordered by delivery time) of `Message` objects. The Looper's `loop()` method runs an infinite `for` loop, dequeuing the next due `Message` and dispatching it to the `Message`'s `target` — which is always a `Handler`.

```
Thread
│
├── Looper
│     └── MessageQueue ←──────────────────────────────────────┐
│           │                                                   │
│           │  dequeue next Message                            │
│           ▼                                                   │
│       Message { target: Handler, callback: Runnable, what: Int, obj: Any }
│                │
│                └── Handler.dispatchMessage()
│                         │
│                    ┌────▼─────┐
│                    │ Runnable │ ── if msg.callback != null
│                    │handleMsg │ ── else Handler.handleMessage()
│                    └──────────┘
```

### Main Thread Looper

The main thread's `Looper` is prepared by `ActivityThread.main()` before any application code runs. It loops forever, processing messages that encode everything from touch events (`MotionEvent`) to lifecycle transactions (`H.LAUNCH_ACTIVITY`, `H.RESUME_ACTIVITY`) to `Choreographer` frame callbacks (VSync signals for rendering).

```
ActivityThread.main()
    │
    Looper.prepareMainLooper()
    ActivityThread at = new ActivityThread()
    at.attach(false)
    │
    Looper.loop()  ←──── runs forever; all app code happens inside here
           │
     ┌─────▼─────────────────────────────────────────────────────┐
     │  H.handleMessage()  ← all lifecycle callbacks dispatched here │
     └───────────────────────────────────────────────────────────┘
```

### Handler Message Delivery

A `Handler` is always associated with one specific `Looper` (and therefore one thread). You can post `Runnable`s or send `Message`s via:
- `handler.post(runnable)` — executes immediately (as soon as the queue processes it)
- `handler.postDelayed(runnable, delayMs)` — scheduled after a delay
- `handler.sendMessage(msg)` — dispatches to `handleMessage()`
- `handler.sendMessageAtTime(msg, uptimeMs)` — absolute delivery time

### Coroutine Internals (Simplified)

A coroutine is a **suspendable computation**. Under the hood, the Kotlin compiler transforms `suspend` functions into a state machine. Each suspension point (`suspend` keyword, `delay()`, `withContext()`) becomes a state in this machine. The coroutine is not a thread — it is a lightweight object that can be **paused** (at a suspension point) and **resumed** later on an appropriate thread, as determined by its `CoroutineDispatcher`.

```
lifecycleScope.launch(Dispatchers.Main) {   // coroutine starts on Main thread
    showLoading(true)                        // runs on Main

    val data = withContext(Dispatchers.IO) { // suspends; resumes on IO thread pool
        repository.fetchData()               // blocking call on IO thread
    }                                        // suspends; resumes on Main thread

    showLoading(false)                       // runs on Main
    renderData(data)                         // runs on Main
}
```

The `withContext(Dispatchers.IO)` does **not** create a new coroutine — it switches the coroutine's dispatcher (thread) for the duration of the block, then automatically switches back to the parent dispatcher. This is **structured concurrency**: the inner block's lifetime is bounded by the outer coroutine.

### Dispatchers

| Dispatcher | Backed By | Use Case |
|-----------|-----------|----------|
| `Dispatchers.Main` | Main thread Looper | UI updates, LiveData observation |
| `Dispatchers.IO` | Shared thread pool (≤64 threads) | Network, disk, database |
| `Dispatchers.Default` | CPU-bound thread pool (= CPU cores) | Sorting, JSON parsing, crypto |
| `Dispatchers.Unconfined` | Caller's thread until suspension | Testing, rare edge cases |

---

## Lifecycle / Flow (ASCII Diagrams)

### Looper / Handler Flow

```
Background Thread                         Main Thread
──────────────────                       ──────────────
  Looper.prepare()
  val handler = Handler(looper)
  Looper.loop()
       │
       │  (waiting for messages)
       │
       │                                 handler.post {
       │◄───── Message enqueued ─────────   textView.text = "Done"
       │                                 }
       │
  MessageQueue.next()  ← unblocks
  dispatchMessage()
  handler.handleMessage()
       │
  runnable executes
  on this thread
```

### Coroutine Dispatcher Flow

```
Main Thread                     IO Thread Pool
─────────────                  ──────────────
launch(Dispatchers.Main)
  │
  showLoading(true)
  │
  withContext(IO) ─────────────► fetchData()  [blocks IO thread, not main]
  │  (suspended)                      │
  │                              data returned
  │◄────────────────────────────────  │
  │  (resumed on Main)
  showLoading(false)
  renderData(data)
  │
  coroutine completes
```

### HandlerThread Lifecycle

```
HandlerThread lifecycle:
  start()
    │
    run() → Looper.prepare() → Looper.loop()
                │
         onLooperPrepared()  ← override to create Handler here
                │
         Thread is ready for messages
                │
         ... processing messages ...
                │
         quitSafely()
                │
         MessageQueue drains pending messages
                │
         Looper exits loop()
                │
         Thread.run() returns → Thread dies
```

---

## Real World Example

A messaging app needs to:
1. Load the conversation list from a local database (disk I/O — must be off main thread).
2. Decrypt each message (CPU-bound — use `Dispatchers.Default`).
3. Render the decrypted list in a `RecyclerView` (UI — must be on main thread).

```kotlin
viewModelScope.launch {
    val encrypted = withContext(Dispatchers.IO) { db.conversationDao().getAll() }
    val decrypted = withContext(Dispatchers.Default) { encrypted.map { decrypt(it) } }
    _conversations.value = decrypted // StateFlow, observed by UI on Main
}
```

All three dispatching decisions are explicit, readable, and correct. If the ViewModel is cleared (user navigates away), `viewModelScope` cancels automatically — no memory leak, no stale UI update.

---

## Common Mistakes

### 1. Blocking the Main Thread

```kotlin
// ❌ WRONG — freezes UI, causes ANR
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        val data = URL("https://api.example.com/data").readText() // NetworkOnMainThreadException
        textView.text = data
    }
}

// ✅ CORRECT — use coroutines
lifecycleScope.launch {
    val data = withContext(Dispatchers.IO) {
        URL("https://api.example.com/data").readText()
    }
    textView.text = data
}
```

### 2. Updating UI from a Background Thread

```kotlin
// ❌ WRONG — CalledFromWrongThreadException
Thread {
    val bitmap = loadBitmap()
    imageView.setImageBitmap(bitmap) // crash!
}.start()

// ✅ CORRECT
Thread {
    val bitmap = loadBitmap()
    runOnUiThread { imageView.setImageBitmap(bitmap) }
}.start()

// ✅ BETTER — use coroutines (no explicit thread switching boilerplate)
lifecycleScope.launch {
    val bitmap = withContext(Dispatchers.IO) { loadBitmap() }
    imageView.setImageBitmap(bitmap)
}
```

### 3. Leaking `Handler` with Inner Class Reference

```kotlin
// ❌ WRONG — anonymous Handler holds implicit reference to outer Activity
class MyActivity : AppCompatActivity() {
    private val handler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            textView.text = "Done" // leaks Activity if delayed message fires after finish()
        }
    }
}

// ✅ CORRECT — use WeakReference pattern or, better, use coroutines
```

### 4. Using `AsyncTask` for Anything New

> ⚠️ `AsyncTask` is **deprecated as of API 30 (Android 11)**. Do not use it in new code. Replace with Kotlin Coroutines + `viewModelScope` / `lifecycleScope`.

### 5. Not Quitting HandlerThread

```kotlin
// ❌ WRONG — HandlerThread keeps running after Activity is destroyed
class MyActivity : AppCompatActivity() {
    private val workerThread = HandlerThread("Worker").also { it.start() }
}

// ✅ CORRECT — quit in onDestroy
override fun onDestroy() {
    super.onDestroy()
    workerThread.quitSafely()
}
```

### 6. Using `GlobalScope`

```kotlin
// ❌ WRONG — GlobalScope is not lifecycle-aware; coroutine outlives the Activity
GlobalScope.launch {
    val data = repository.fetch()
    withContext(Dispatchers.Main) { textView.text = data } // Activity may be dead!
}

// ✅ CORRECT
lifecycleScope.launch {
    val data = withContext(Dispatchers.IO) { repository.fetch() }
    textView.text = data // safe: coroutine is cancelled when lifecycle is destroyed
}
```

---

## Memory Leak / Performance Concerns

### Handler Memory Leaks

A `Handler` posted with `postDelayed()` holds a strong reference to its `Runnable`. If the `Runnable` captures an `Activity` reference (even implicitly through an anonymous class or lambda), the Activity cannot be garbage-collected for the duration of the delay. The fix: always remove pending callbacks in `onDestroy()`.

```kotlin
private val handler = Handler(Looper.getMainLooper())
private val myRunnable = Runnable { /* ... */ }

override fun onDestroy() {
    super.onDestroy()
    handler.removeCallbacks(myRunnable) // critical — removes from MessageQueue
}
```

### Thread Pool Saturation (Dispatchers.IO)

`Dispatchers.IO` is bounded to 64 threads by default. If 64 coroutines all block on slow I/O simultaneously, new coroutines queue up — effectively creating a thread pool starvation scenario. Use `limitedParallelism()` for specific tasks with known concurrency requirements, or prefer truly non-blocking I/O (Retrofit with `suspend` functions, Room coroutine queries) that don't hold a thread while waiting.

### `AsyncTask` Memory Leak (Legacy Reference)

`AsyncTask`'s most notorious flaw: if declared as an inner class of an Activity, it holds an implicit reference to the Activity. If the Activity is destroyed (rotation) while the task is in `doInBackground()`, the Activity instance is kept alive until the task completes. This is why every production codebase that used `AsyncTask` had either static inner classes with `WeakReference` wrappers or extracted the logic to a separate non-Activity class.

---

## Interview Questions

### Beginner

**Q1: What is the Main Thread / UI Thread, and why can't we do network operations on it?**

> The Main Thread is the single thread on which Android runs all UI rendering, touch event dispatch, and lifecycle callbacks. It is implemented as a `Looper` thread with the main `MessageQueue`. Network operations (and disk I/O) can block for arbitrary durations — milliseconds to seconds. If the main thread is blocked for more than 5 seconds without processing input events, Android's `ActivityManager` detects this and triggers an ANR (Application Not Responding) dialog. Beyond ANR risk, blocking the main thread prevents frame rendering: at 60 fps, each frame has 16ms. Any operation taking longer than 16ms on the main thread causes a dropped frame and visible jank.

**Q2: What is a Looper, and what is its relationship to a Handler?**

> A `Looper` is an object that turns any thread into a message-processing loop. It manages a `MessageQueue` and runs a blocking loop (`Looper.loop()`) that continuously dequeues `Message` objects and dispatches them. A `Handler` is the API through which code *enqueues* messages into a specific thread's `Looper`. Every `Handler` is bound to exactly one `Looper` at construction time. Posting a `Runnable` to a `Handler` schedules it to run on the `Handler`'s associated thread.

**Q3: What is `viewModelScope` and why should you use it over `GlobalScope`?**

> `viewModelScope` is a `CoroutineScope` that is tied to a `ViewModel`'s lifecycle. It is automatically cancelled when the `ViewModel` is cleared (i.e., when the associated Activity/Fragment is permanently destroyed). This means all coroutines launched in `viewModelScope` are automatically cancelled when they are no longer needed — no manual cancellation, no memory leaks. `GlobalScope` has no lifecycle awareness and lives for the entire app process lifetime. A coroutine in `GlobalScope` continues running after its associated UI is gone, can cause stale UI updates (updating a dead Activity's views), and prevents garbage collection of any objects it captures.

---

### Intermediate

**Q4: Explain `Handler`, `Looper`, and `MessageQueue` and how they work together.**

> The `MessageQueue` is a priority queue (ordered by delivery time) of `Message` objects, owned by a `Looper`. The `Looper.loop()` method runs an infinite blocking loop that calls `MessageQueue.next()` — this blocks when the queue is empty and returns the next due `Message`. The `Message` has a `target` field pointing to the `Handler` that should receive it. The `Looper` calls `msg.target.dispatchMessage(msg)`, which eventually calls `handleMessage()` or runs the Runnable callback. The entire mechanism is **thread-safe**: enqueuing to a `MessageQueue` from any thread is synchronized internally, while dequeuing and dispatch always happen on the owning thread.

**Q5: What is `HandlerThread` and when would you use it over a plain `Thread`?**

> `HandlerThread` is a `Thread` subclass that calls `Looper.prepare()` and `Looper.loop()` inside its `run()` method, providing a ready-to-use `Looper`. You use it when you want a background thread that processes work *serially* (one task at a time, in order), rather than spawning a new thread for each task. Classic use cases: a serial I/O queue, background audio processing, a serial command dispatcher. Compared to a plain `Thread`, `HandlerThread` gives you a `Handler` to post work to, automatic message ordering, and `quit()`/`quitSafely()` for clean shutdown.

**Q6: Explain `withContext()` vs `launch()` in Kotlin Coroutines.**

> `launch()` starts a new coroutine concurrently — it returns a `Job` immediately, and the new coroutine runs independently (though still within the structured concurrency scope). It is fire-and-forget; you don't get a return value from `launch`. `withContext()` is a *suspending* function — it suspends the current coroutine, switches its `CoroutineDispatcher` (thread) to the specified dispatcher, executes the block, then *switches back* to the original dispatcher and returns the result. `withContext()` does not create a new concurrent coroutine; it sequentially delegates the block to a different thread, waits for it, and returns. Use `launch()` for concurrent fire-and-forget; use `withContext()` for thread-switching within a sequential flow.

---

### Advanced

**Q7: How does Kotlin coroutine cancellation work? What is a `CancellationException`?**

> Cancellation in coroutines is **cooperative**. When you cancel a coroutine (`job.cancel()`), the `Job` is marked as cancelled. The cancellation propagates: all child coroutines receive a `CancellationException`. However, the coroutine itself must *check* for cancellation — this happens automatically at every **suspension point** (every call to a `suspend` function like `delay()`, `withContext()`, `yield()`). If a coroutine never suspends (a tight CPU loop), it will not be cancelled until it calls a suspension point. To make CPU-bound loops cancellable, periodically call `yield()` or check `isActive`. `CancellationException` is special: it is **not** re-thrown after the coroutine's scope catches it, and it does not represent a failure — it is the normal signal for structured cancellation and is never propagated to parent coroutines as an error.

**Q8: What is `StrictMode` and how does it help with threading issues?**

> `StrictMode` is an Android developer tool that detects accidental resource misuse in debug builds. Specifically, `StrictMode.ThreadPolicy` can flag: (1) disk reads on the main thread, (2) disk writes on the main thread, (3) network access on the main thread, (4) unbuffered I/O on the main thread. When a violation is detected, `StrictMode` can log it, flash the screen, display a dialog, or crash the app (in debug). `StrictMode.VmPolicy` detects: leaked `Closeable`/`SQLiteCursor` objects, leaked Activities, and unclosed `ContentProvider` clients. You should enable `StrictMode` in `Application.onCreate()` for debug builds:
> ```kotlin
> if (BuildConfig.DEBUG) {
>     StrictMode.setThreadPolicy(StrictMode.ThreadPolicy.Builder()
>         .detectAll().penaltyLog().build())
>     StrictMode.setVmPolicy(StrictMode.VmPolicy.Builder()
>         .detectAll().penaltyLog().build())
> }
> ```

**Q9: Explain `Dispatchers.IO` vs `Dispatchers.Default` and when each is appropriate.**

> `Dispatchers.Default` uses a thread pool sized to the number of CPU cores (minimum 2). It is designed for CPU-bound work: sorting, image processing, JSON deserialization, encryption. These tasks keep the CPU busy and benefit from up to N threads for N cores. `Dispatchers.IO` uses a pool of up to 64 threads (or `Runtime.getRuntime().availableProcessors()`, whichever is larger). Its larger pool exists because I/O operations typically spend most of their time *waiting* (for disk, network, database) rather than consuming CPU. Many I/O coroutines can run concurrently on many threads without contending for CPU. Using `Dispatchers.IO` for CPU-bound work wastes memory (64 threads) and can lead to cache thrashing. Using `Dispatchers.Default` for I/O work limits concurrency unnecessarily. The key heuristic: if the operation *blocks a thread*, use `IO`; if it uses the CPU, use `Default`.

---

## Scenario Questions

**Scenario 1:** A junior developer reports that their image loading coroutine works fine in isolation but causes visible UI jank in a `RecyclerView` list. They are loading 30 thumbnails simultaneously. What is the likely cause and fix?

> **Answer:** Launching 30 independent `Dispatchers.IO` coroutines simultaneously saturates the IO thread pool with blocking decodes. The real issue is the coroutines are decoding images (CPU-bound work) on `Dispatchers.IO` instead of `Dispatchers.Default`, and there's no concurrency control. Even with the correct dispatcher, 30 concurrent CPU operations will starve each other. The fix involves two changes: (1) use `Dispatchers.Default` for the decode step (CPU-bound). (2) Use `limitedParallelism(4)` on `Dispatchers.Default` to decode at most 4 images simultaneously. Better still: use Coil or Glide, which implement their own request queuing, memory caching, disk caching, and concurrency management — don't reinvent image loading.

**Scenario 2:** Your ViewModel fetches data with `viewModelScope.launch`. The user rotates the device, the Activity is recreated, the ViewModel survives — but the UI never updates. Why?

> **Answer:** The likely cause is that the coroutine updates a `LiveData` object using `postValue()` or `setValue()`, but the new Activity's Fragment/Observer hasn't observed it yet, or it observed it but the data was emitted *before* the new observer subscribed. Check: (1) Is the `LiveData` observed in `onStart()`/`onViewCreated()`? (2) Is the `LiveData` a `MutableLiveData` holding its last value (it should re-deliver the last value to new observers)? If using `StateFlow`, confirm the Activity re-collects in `repeatOnLifecycle(Lifecycle.State.STARTED)` — plain `lifecycleScope.launch { flow.collect {} }` does not re-activate on re-creation. The correct pattern for `StateFlow` collection in fragments is `viewLifecycleOwner.lifecycleScope.launch { viewLifecycleOwner.lifecycle.repeatOnLifecycle(STARTED) { viewModel.uiState.collect { render(it) } } }`.

**Scenario 3:** You have a long-running operation in a `Handler.postDelayed()` call. The user presses back and the Activity is destroyed. Memory Analyzer Tool shows the Activity is not garbage collected. Explain why and how to fix it.

> **Answer:** The `postDelayed()` call enqueues a `Message` in the main `MessageQueue` with a scheduled delivery time in the future. The `Message`'s `callback` field holds a reference to the `Runnable`. If the `Runnable` is an anonymous class or lambda that captures `this` (the Activity), the `MessageQueue` → `Message` → `Runnable` → `Activity` reference chain keeps the Activity alive. The main thread's `Looper` (and its `MessageQueue`) is a GC root — it lives for the process lifetime. Fix: call `handler.removeCallbacks(myRunnable)` in `Activity.onDestroy()` to remove the pending message from the queue, breaking the reference chain.

---

## Code Examples (Production-Quality Kotlin)

### Example 1: Handler + Looper Pattern (Foundation Knowledge)

```kotlin
// Understanding the raw Handler/Looper API is essential for debugging
// even if you use coroutines in production.

class HandlerLooperDemo {

    // Posting to the Main Thread from any background thread
    private val mainHandler = Handler(Looper.getMainLooper())

    fun postToMainThread(action: () -> Unit) {
        mainHandler.post(action)
    }

    fun postDelayedToMainThread(delayMs: Long, action: () -> Unit): Runnable {
        val runnable = Runnable(action)
        mainHandler.postDelayed(runnable, delayMs)
        return runnable // caller must keep reference to cancel
    }

    fun cancelDelayed(runnable: Runnable) {
        mainHandler.removeCallbacks(runnable)
    }

    // Custom Handler subclass with WeakReference — prevents Activity leaks
    class SafeHandler(
        looper: Looper,
        activity: WeakReference<MainActivity>
    ) : Handler(looper) {
        private val activityRef = activity

        override fun handleMessage(msg: Message) {
            val activity = activityRef.get() ?: return // Activity is gone
            when (msg.what) {
                MSG_UPDATE_UI -> activity.updateUi(msg.obj as String)
                MSG_SHOW_ERROR -> activity.showError(msg.obj as String)
            }
        }

        companion object {
            const val MSG_UPDATE_UI = 1
            const val MSG_SHOW_ERROR = 2
        }
    }
}
```

### Example 2: HandlerThread — Serial Background Worker

```kotlin
/**
 * A serial background worker backed by a HandlerThread.
 * Useful for ordered I/O tasks (e.g., sequential file writes,
 * Bluetooth command queues, serial sensor reads).
 */
class SerialBackgroundWorker(name: String) {

    private val handlerThread = HandlerThread(name)
    private val handler: Handler

    init {
        handlerThread.start()
        // After start(), looper is available on the HandlerThread
        handler = Handler(handlerThread.looper)
    }

    /**
     * Submits a task to run on the worker thread.
     * Tasks are executed serially in submission order.
     */
    fun submit(task: () -> Unit) {
        handler.post(task)
    }

    /**
     * Submits a task and returns its result via callback on the main thread.
     */
    fun <T> submitWithResult(task: () -> T, onResult: (T) -> Unit) {
        handler.post {
            val result = task()
            Handler(Looper.getMainLooper()).post { onResult(result) }
        }
    }

    /**
     * Shuts down the worker thread gracefully.
     * Pending tasks are processed before the thread exits.
     * Call this in onDestroy().
     */
    fun shutdown() {
        handlerThread.quitSafely()
    }
}

// Usage in an Activity
class MyActivity : AppCompatActivity() {

    private val worker = SerialBackgroundWorker("FileWorker")

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        worker.submit {
            // Runs on HandlerThread — safe for disk I/O
            writeDataToFile("important_data.txt", "content")
        }

        worker.submitWithResult(
            task = { readFromDatabase() },
            onResult = { data -> textView.text = data } // runs on Main
        )
    }

    override fun onDestroy() {
        super.onDestroy()
        worker.shutdown() // critical — prevents thread leak
    }

    private fun writeDataToFile(filename: String, content: String) { /* ... */ }
    private fun readFromDatabase(): String = "data"
}
```

### Example 3: AsyncTask — Legacy Reference (Deprecated ⚠️)

> ⚠️ **DEPRECATED in API 30 (Android 11).** This example is provided for reference when maintaining legacy codebases. **Do not use in new code.**

```kotlin
/**
 * ⚠️ DEPRECATED — AsyncTask was deprecated in Android API 30.
 * This code is shown for LEGACY REFERENCE ONLY.
 * Migrate to Kotlin Coroutines (see Example 4).
 *
 * Known issues:
 * - Not lifecycle-aware (continues after Activity is destroyed)
 * - Can leak Activity if declared as inner class
 * - Not easily restartable or composable
 * - Serial execution by default (executeOnExecutor for parallel)
 */
@Deprecated("Use Kotlin Coroutines instead. See Example 4.")
class FetchDataTask(
    private val activityRef: WeakReference<MainActivity>
) : AsyncTask<String, Int, Result<String>>() {

    override fun onPreExecute() {
        // Runs on UI thread before background work
        activityRef.get()?.showLoading(true)
    }

    override fun doInBackground(vararg params: String): Result<String> {
        // Runs on background thread — must NOT touch UI
        return try {
            val url = params[0]
            val data = fetchFromNetwork(url)
            publishProgress(50) // triggers onProgressUpdate
            val processed = processData(data)
            publishProgress(100)
            Result.success(processed)
        } catch (e: Exception) {
            Result.failure(e)
        }
    }

    override fun onProgressUpdate(vararg values: Int) {
        // Runs on UI thread
        activityRef.get()?.updateProgress(values[0])
    }

    override fun onPostExecute(result: Result<String>) {
        // Runs on UI thread after doInBackground completes
        val activity = activityRef.get() ?: return
        activity.showLoading(false)
        result.fold(
            onSuccess = { activity.showData(it) },
            onFailure = { activity.showError(it.message ?: "Unknown error") }
        )
    }

    private fun fetchFromNetwork(url: String): String = "" // placeholder
    private fun processData(data: String): String = data
}
```

### Example 4: Modern Coroutine Replacement for AsyncTask

```kotlin
// ✅ MODERN — Kotlin Coroutines with full lifecycle awareness

class DataViewModel(
    private val repository: DataRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<DataUiState>(DataUiState.Idle)
    val uiState: StateFlow<DataUiState> = _uiState.asStateFlow()

    fun fetchData(url: String) {
        viewModelScope.launch {
            _uiState.value = DataUiState.Loading

            _uiState.value = try {
                val data = withContext(Dispatchers.IO) {
                    repository.fetch(url)
                }
                DataUiState.Success(data)
            } catch (e: CancellationException) {
                throw e // always re-throw CancellationException
            } catch (e: Exception) {
                DataUiState.Error(e.message ?: "Unknown error")
            }
        }
    }

    // If the ViewModel is cleared, viewModelScope cancels all coroutines automatically
    // No manual cleanup needed — this is structured concurrency.
}

sealed class DataUiState {
    object Idle : DataUiState()
    object Loading : DataUiState()
    data class Success(val data: String) : DataUiState()
    data class Error(val message: String) : DataUiState()
}

// Collecting in Fragment — safe and lifecycle-aware
class DataFragment : Fragment(R.layout.fragment_data) {

    private val viewModel: DataViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewLifecycleOwner.lifecycleScope.launch {
            // repeatOnLifecycle suspends collection when lifecycle < STARTED
            // and resumes when it returns to STARTED — prevents updates to
            // invisible UI and stops collecting after view destruction.
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    renderState(state)
                }
            }
        }

        binding.fetchButton.setOnClickListener {
            viewModel.fetchData("https://api.example.com/data")
        }
    }

    private fun renderState(state: DataUiState) {
        when (state) {
            is DataUiState.Idle -> { /* show empty state */ }
            is DataUiState.Loading -> { /* show progress */ }
            is DataUiState.Success -> { /* show data */ }
            is DataUiState.Error -> { /* show error */ }
        }
    }
}
```

### Example 5: StrictMode Setup (Debug Builds)

```kotlin
class MyApplication : Application() {

    override fun onCreate() {
        super.onCreate()

        if (BuildConfig.DEBUG) {
            StrictMode.setThreadPolicy(
                StrictMode.ThreadPolicy.Builder()
                    .detectDiskReads()
                    .detectDiskWrites()
                    .detectNetwork()
                    .detectCustomSlowCalls()
                    .penaltyLog()         // log to Logcat
                    // .penaltyCrash()   // uncomment to crash on violation
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

---

## Best Practices

1. **Never block the main thread.** Not for 100ms. Not for "just a quick DB read." Period.
2. **Use `viewModelScope` in ViewModels, `lifecycleScope` in Activities/Fragments.** Never `GlobalScope` in production.
3. **Always `throw` `CancellationException`.** If you catch `Exception` in a coroutine, check for and re-throw `CancellationException` to keep cancellation cooperative.
4. **Collect `StateFlow`/`Flow` with `repeatOnLifecycle(STARTED)`.** Plain `lifecycleScope.launch { flow.collect {} }` keeps collecting even when the UI is not visible (e.g., home button pressed).
5. **Use `Dispatchers.IO` for blocking I/O, `Dispatchers.Default` for CPU work.** Never guess — profile with Android Studio's CPU Profiler.
6. **Always call `handlerThread.quitSafely()` in `onDestroy()`.** A running `HandlerThread` is a leaked thread.
7. **Remove pending `Handler` callbacks in `onDestroy()`.** Use `handler.removeCallbacksAndMessages(null)` to remove everything at once.
8. **Enable `StrictMode` in debug builds.** It catches threading violations at development time, not in production.
9. **Do not use `AsyncTask` in new code.** Migrate existing `AsyncTask` usage to coroutines.
10. **Use `limitedParallelism()` for bounded concurrent I/O.** Prevents thread pool saturation for bulk operations.

---

## Legacy vs Modern

| Concern | Legacy | Modern |
|---------|--------|--------|
| Background work | `Thread` + `runOnUiThread()` | Kotlin Coroutines + `Dispatchers` |
| Background work | `AsyncTask` (**Deprecated API 30**) | `viewModelScope.launch` + `withContext` |
| Serial background queue | `HandlerThread` | `Dispatchers.IO` with single-thread `newSingleThreadContext()` or `HandlerThread` (still valid) |
| Result delivery | `onPostExecute()` callback | `StateFlow` / `LiveData` |
| UI state observation | `AsyncTask` callbacks | `StateFlow` + `repeatOnLifecycle` |
| Guaranteed background work | `AlarmManager` + `Service` | `WorkManager` |
| Progress reporting | `publishProgress()` | `MutableStateFlow` update in loop |
| Lifecycle-aware coroutine scope | N/A | `viewModelScope`, `lifecycleScope` |

---

## Revision Notes

- **Main thread = single thread for all UI + lifecycle.** Blocking it > 5s = ANR.
- **Looper** owns a `MessageQueue` and processes it in a loop. **Handler** posts to a Looper.
- **`Handler(Looper.getMainLooper())`** is the universal "post to UI thread" mechanism, used internally by `runOnUiThread()`, `View.post()`, etc.
- **HandlerThread** = Thread + Looper + automatic `quitSafely()` management pattern.
- **`AsyncTask` is DEAD (API 30+).** Know it for interviews about legacy code; never write it new.
- **Coroutines** are lightweight, structured, and cancellable. Key mental model: `launch` = concurrent, `withContext` = sequential thread switch.
- **Dispatchers:** Main (UI), IO (blocking I/O), Default (CPU), Unconfined (testing).
- **`repeatOnLifecycle(STARTED)`** is the correct way to collect flows in Fragment/Activity.
- **`CancellationException` must always be re-thrown** inside coroutine catch blocks.

---

## Key Takeaways

> **Threading is the single biggest source of bugs and ANRs in Android apps.** Mastering it separates junior from senior developers.

- The main thread is sacred — keep it free of blocking operations at all times.
- `Looper` + `Handler` is the foundation of the entire Android async system; everything else is built on top.
- `HandlerThread` is still relevant for serial background work (Bluetooth stacks, audio engines, etc.).
- `AsyncTask` is deprecated — migrate all usage to coroutines.
- Kotlin Coroutines with `viewModelScope`/`lifecycleScope` are the production standard.
- `withContext(Dispatchers.IO)` for I/O, `withContext(Dispatchers.Default)` for CPU — always be explicit.
- Use `repeatOnLifecycle(STARTED)` for `StateFlow`/`Flow` collection in UI.
- Memory leaks from threading come from captured references in `Runnable`s, `Handler` subclasses, and `AsyncTask` inner classes.

---
---

# Chapter 8: Services & Process Lifecycle

---

## Concept

A **Service** is an Android application component designed to perform operations in the background without providing a user interface. Unlike an Activity, a Service has no visual representation and can continue running even when the user switches to another application.

> ⚠️ **Critical Misconception:** A Service does **not** run in a separate thread by default. It runs on the **main thread** of the process that hosts it. This is the single most common misunderstanding about Services. Long-running or blocking operations inside a Service *must* be explicitly moved to a background thread using coroutines or a `HandlerThread`.

### Types of Services

| Type | Start Mechanism | Lifecycle | Use Case |
|------|----------------|-----------|----------|
| **Started** | `startService()` / `startForegroundService()` | Independent of caller | Music playback, file download |
| **Bound** | `bindService()` | Tied to bound clients | IPC, local data access |
| **Foreground** | `startForegroundService()` + `startForeground()` | Independent; higher priority | Navigation, media, fitness |
| **WorkManager** (recommended) | `WorkManager.enqueue()` | System-managed | Deferred, guaranteed tasks |

### `onStartCommand()` Return Values

When a Started Service is killed by the system (low memory), its return value from `onStartCommand()` determines restart behaviour:

| Value | Restart Behaviour | Intent on Restart |
|-------|------------------|------------------|
| `START_STICKY` | Recreated automatically | `null` intent |
| `START_NOT_STICKY` | **Not** recreated automatically | N/A |
| `START_REDELIVER_INTENT` | Recreated automatically | Last received intent re-delivered |
| `START_STICKY_COMPATIBILITY` | Same as `START_STICKY` but no recreation guarantee | `null` intent |

**Choosing the right value:**
- `START_STICKY` → Music player (restart is desired even without the original intent)
- `START_NOT_STICKY` → One-off tasks; don't restart if killed mid-task
- `START_REDELIVER_INTENT` → File download (must know the original URL to resume)

---

## Why It Exists

Services exist to solve a fundamental tension in Android's process lifecycle model: Activities are ephemeral — they are destroyed and recreated constantly as the user navigates. But many operations (playing music, tracking location, syncing data, maintaining a WebSocket connection) need to outlive any single Activity.

Services provide:
1. **Process persistence** — a running Service raises the process priority, making it less likely to be killed.
2. **IPC foundation** — Bound Services (via `IBinder`) are the backbone of cross-process communication, used internally by `LocationManager`, `AudioManager`, `WindowManager`, and every other system service.
3. **Background execution contract** — starting with API 26, Android restricts background execution aggressively. A **Foreground Service** with a visible notification is the explicit, user-visible contract for long-running background work.

---

## Internal Working

### Service Hosting and Thread Model

A Service declared in the manifest without `android:process` runs in the **same process** as the declaring application. It receives lifecycle callbacks (`onCreate`, `onStartCommand`, `onBind`, `onDestroy`) on the **main thread** via the `ActivityThread`'s main `MessageQueue`, just like Activity callbacks.

### Started Service Flow

```
Client Process                         system_server (AMS)              Service Process
──────────────                        ──────────────────               ───────────────
context.startService(intent)
  │
  └──── Binder IPC ─────────────────► AMS.startService()
                                             │
                                       Service exists?
                                         ├─ No  → spawn process, call Service.onCreate()
                                         └─ Yes → skip onCreate()
                                             │
                                       Binder IPC ─────────────────────► Service.onStartCommand(intent)
                                                                                │
                                                                          returns START_STICKY
                                                                          (runs until stopSelf() or stopService())
```

### Bound Service — Binder IPC

The `IBinder` returned by `onBind()` is a handle to the Service's internal state. For local (same-process) binding, this is a direct Java object reference. For remote (cross-process) binding, the Binder framework serializes method calls across processes using Binder's shared memory mechanism:

```
Client Activity                  Binder Driver (kernel)           Remote Service
──────────────                  ─────────────────────            ─────────────
bindService(intent, conn, flags)
      │
  ServiceConnection.onServiceConnected(name, binder)
      │
  val proxy = IMyService.Stub.asInterface(binder)
      │
  proxy.doSomething(arg)  ───────── Binder IPC ───────────────► IMyService.Stub.doSomething(arg)
                                                                  (runs on Binder thread pool)
```

### Foreground Service Requirements (API 26+)

```
startForegroundService(intent)  ← can be called from any component
      │
      ▼
  Service.onCreate()
  Service.onStartCommand()
      │
      ▼
  MUST call startForeground(id, notification)
  within 5 seconds  ← if not called, ANR + ForegroundServiceDidNotStartInTimeException
      │
      ▼
  Persistent notification visible to user
      │
      ▼
  Process priority elevated to FOREGROUND
  System will not kill this service under memory pressure
```

### Process Lifecycle Priority

Android's **Low Memory Killer** (LMK) kills processes in reverse priority order when memory is needed. The priority is determined by the most important component running in the process:

```
Priority (highest to lowest, hardest to kill first):
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. FOREGROUND PROCESS
   ├── Has a running Activity in resumed state (onResume)
   ├── Has a Service bound to the foreground Activity
   ├── Has a Foreground Service (startForeground)
   └── Has a BroadcastReceiver executing onReceive()

2. VISIBLE PROCESS
   ├── Has a paused Activity (not in foreground but visible)
   └── Has a Service bound to a Visible Activity

3. SERVICE PROCESS
   └── Has a started Service (not foreground)
       → Will be killed if memory is needed for 30+ minutes

4. CACHED (BACKGROUND) PROCESS
   ├── Has only stopped Activities
   └── Killed first under memory pressure; exact order by LRU + memory

5. EMPTY PROCESS
   └── No active components; kept briefly for startup performance
```

A **Foreground Service** elevates the process to FOREGROUND priority. A regular **Started Service** only elevates to SERVICE priority — it will be killed in memory pressure scenarios, just less aggressively than background processes.

---

## Lifecycle / Flow (ASCII Diagrams)

### Started Service Lifecycle

```
                    ┌─────────────────────────────┐
                    │      Service Created         │
                    │         onCreate()           │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │      onStartCommand()        │◄──── startService() called again
                    │  (called on each startService)│     (service already running)
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │    Service Running           │
                    │   (doing background work)    │
                    └──────────────┬──────────────┘
                                   │
                    ┌──────────────▼──────────────┐
                    │        onDestroy()           │◄──── stopSelf() or stopService()
                    └──────────────┬──────────────┘      or system kills it
                                   │
                              Service ends
```

### Bound Service Lifecycle

```
Client                                    Service
──────                                   ───────
bindService(intent, conn, BIND_AUTO_CREATE)
      │
      │                            ──── onCreate() (if not already running)
      │                            ──── onBind(intent)
      │                                     │ returns IBinder
      │                            ◄─────────┘
  onServiceConnected(binder)

  ... use service via binder ...

  unbindService(conn)
      │
      │                            ──── onUnbind(intent)
      │                                     │ returns true → onRebind enabled
      │                            ──── onDestroy() (if no other clients and not started)

  (new client binds while still alive)
      │
      │                            ──── onRebind(intent)  [only if onUnbind returned true]
```

### Foreground Service Lifecycle

```
Component                         Service                        Notification Bar
──────────                       ─────────                      ────────────────
startForegroundService(intent)
      │
      │                    onCreate()
      │                    onStartCommand()
      │                          │
      │                    startForeground(id, notification) ──────► [Persistent notification]
      │                          │
      │                    (doing work in background thread)
      │                          │
      │                    stopForeground(true/false) ──────────────► [Notification removed]
      │                    stopSelf()
      │                          │
      │                    onDestroy()
```

---

## Real World Example

A **music player app** uses a Foreground Service to continue playback when the user navigates away:
- The Service is Started from `MainActivity` via `startForegroundService()`.
- It immediately calls `startForeground()` with a `MediaStyle` notification showing track info and play/pause controls (backed by `PendingIntent`s).
- The Service manages a `MediaPlayer` or `ExoPlayer` instance on a background thread (or uses ExoPlayer's built-in thread management).
- The `MainActivity` binds to the Service to display the current track and seek bar.
- When the user swipes away the notification, the Service calls `stopSelf()` and music stops.

This pattern separates concerns clearly: the Activity handles UI, the Service handles background playback, and the notification is the persistent user-facing control surface.

---

## Common Mistakes

### 1. Doing Blocking Work on the Service's Main Thread

```kotlin
// ❌ WRONG — Service runs on main thread; this causes ANR
class DownloadService : Service() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val url = intent?.getStringExtra("url") ?: return START_NOT_STICKY
        val data = URL(url).readBytes() // blocks main thread — ANR!
        saveToDisk(data)
        stopSelf(startId)
        return START_NOT_STICKY
    }
}

// ✅ CORRECT — launch coroutine on IO dispatcher
class DownloadService : LifecycleService() {
    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        val url = intent?.getStringExtra("url") ?: return START_NOT_STICKY
        lifecycleScope.launch {
            withContext(Dispatchers.IO) {
                val data = URL(url).readBytes()
                saveToDisk(data)
            }
            stopSelf(startId)
        }
        return START_REDELIVER_INTENT
    }
}
```

### 2. Starting a Foreground Service Without Calling `startForeground()` in Time

On API 26+, calling `startForegroundService()` starts a timer. If `startForeground()` is not called within 5 seconds, the system throws `ForegroundServiceDidNotStartInTimeException` and ANRs the app. Always call `startForeground()` as the *first* thing in `onStartCommand()`.

### 3. Forgetting to Unbind in `onStop()`

```kotlin
// ❌ WRONG — binding in onStart but forgetting to unbind causes ServiceConnection leak
class MyActivity : AppCompatActivity() {
    override fun onStart() {
        super.onStart()
        bindService(intent, connection, Context.BIND_AUTO_CREATE)
    }
    // Missing onStop() with unbindService()!
}

// ✅ CORRECT
override fun onStop() {
    super.onStop()
    unbindService(connection)
}
```

### 4. Using IntentService for New Code

> ⚠️ `IntentService` is **deprecated in API 30**. Use `WorkManager` for deferrable tasks or a `Service` with coroutines for immediate background work.

### 5. Not Requesting `POST_NOTIFICATIONS` Permission (API 33+)

On Android 13+, showing a notification requires `POST_NOTIFICATIONS` permission to be granted at runtime. Foreground Services *also* need this — the notification will not appear, and on Android 13+ the system will throw an exception.

---

## Memory Leak / Performance Concerns

### ServiceConnection Leak

A `ServiceConnection` must be explicitly unbound via `unbindService()`. Forgetting to do so creates a reference from the system to your `ServiceConnection` object (which typically holds an Activity reference), preventing garbage collection of the Activity.

```kotlin
// Always pair bind/unbind in symmetric lifecycle callbacks
// Activity: bind in onStart(), unbind in onStop()
// Fragment: bind in onStart(), unbind in onStop()
// Application: bind in onCreate() (for app-lifetime services)
```

### Foreground Service Notification Overhead

Each foreground service requires a visible notification. If your app runs multiple independent foreground services simultaneously, the notification tray becomes cluttered. Use a **single Foreground Service** with a compound notification (updated as work progresses) rather than multiple services.

### Process Priority Inflation

Running a Foreground Service raises your process to FOREGROUND priority — the same level as the active app. This means your process consumes more RAM without being subject to LMK pressure. Use Foreground Services only for operations the user is actively aware of. For periodic background work with no user awareness, use `WorkManager` — it is system-managed and will not inflate process priority unnecessarily.

### Binder Thread Pool Exhaustion (Remote Services)

Remote Bound Services receive calls on Binder threads (a pool of ~15–16 threads per process). If AIDL methods perform blocking operations on Binder threads, the pool saturates and new calls queue up. AIDL method implementations should be non-blocking; offload work to a `HandlerThread` or coroutine.

---

## Interview Questions

### Beginner

**Q1: Does a Service run in a background thread?**

> **No.** This is the most common misconception. A Service runs on the **main thread** of its host process by default. If you perform blocking operations (network, disk, long computation) directly in `onStartCommand()` or `onBind()`, you will block the main thread and risk an ANR. You must explicitly move work to a background thread using coroutines (`lifecycleScope` in a `LifecycleService`), a `HandlerThread`, or an `Executor`.

**Q2: What is the difference between a Started Service and a Bound Service?**

> A **Started Service** is initiated via `startService()` or `startForegroundService()`. It runs independently of the caller and continues until it calls `stopSelf()` or the caller calls `stopService()`. It provides no direct API for the caller to interact with it after starting. A **Bound Service** is initiated via `bindService()`. It exposes an `IBinder` interface that the caller uses to communicate with the Service. The Service's lifetime is tied to its bound clients — it is destroyed when the last client unbinds (unless it was also started via `startService()`).

**Q3: What are the three return values of `onStartCommand()` and when do you use each?**

> - `START_STICKY`: The system recreates the Service after it is killed, passing a `null` intent. Use for services that should always be running (e.g., a music player that can restart and recover state). 
> - `START_NOT_STICKY`: The system does not recreate the Service. Use for tasks that can be safely abandoned if killed mid-execution and re-initiated explicitly by the user (e.g., a one-shot fetch).
> - `START_REDELIVER_INTENT`: The system recreates the Service and re-delivers the last intent. Use when the Service needs the original intent to continue or retry work (e.g., a file download that needs the URL).

---

### Intermediate

**Q4: What is a Foreground Service, why is it required on API 26+, and what must you do to start one correctly?**

> Android 8.0 (API 26) introduced background execution limits that prevent background services from running arbitrarily. A **Foreground Service** is exempt from these limits because it has a user-visible notification, making the user aware that background work is in progress. To start one correctly: (1) Call `startForegroundService(intent)` (not `startService()`). (2) In `Service.onStartCommand()`, call `startForeground(notificationId, notification)` within **5 seconds**. (3) Create a `NotificationChannel` on API 26+. (4) On API 33+, request `POST_NOTIFICATIONS` permission. (5) On API 29+, declare `android:foregroundServiceType` in the manifest. Failing any of these causes a crash or the notification not showing.

**Q5: Explain the difference between local binding and remote binding (AIDL).**

> **Local binding:** Both client and service are in the same process. `onBind()` returns a concrete `Binder` subclass (e.g., `LocalBinder` that exposes `getService()`). The client receives the actual Java object via `ServiceConnection.onServiceConnected()`. Method calls are direct Java calls — no serialization, no IPC overhead. This is the most common pattern for binding an Activity to an in-app Service. **Remote binding (AIDL):** Client and service are in different processes. `onBind()` returns an `IBinder` generated by the AIDL compiler (`IMyService.Stub`). Method calls cross the process boundary via Binder IPC — arguments are serialized (parcelled), transmitted through the Binder kernel driver, and unparcelled in the remote process. Results return the same way. **Messenger** is a simpler alternative to AIDL for single-threaded, message-based IPC.

**Q6: When and why would you use `BIND_AUTO_CREATE` flag in `bindService()`?**

> `BIND_AUTO_CREATE` tells the system to automatically create the Service if it is not already running when `bindService()` is called. Without this flag, `bindService()` only connects to an already-running Service — it does not start one. The `BIND_AUTO_CREATE` flag is used in most normal binding scenarios. Its complement is `BIND_NOT_FOREGROUND`, which prevents the client from elevating the service's process priority to FOREGROUND — useful when an Activity binds to a background helper service and you don't want that to inflate the service's priority.

---

### Advanced

**Q7: Explain the process lifecycle hierarchy in Android and how Services affect it.**

> Android's Low Memory Killer kills processes in reverse priority order. The priority of a process is determined by its most important currently active component: **Foreground** (active Activity, active Foreground Service, or active BroadcastReceiver) > **Visible** (visible but paused Activity, or Service bound to a Foreground/Visible activity) > **Service** (running Started Service) > **Cached** (stopped Activities only) > **Empty** (no components). A Foreground Service elevates its process to FOREGROUND priority — essentially as important as an app in the foreground — making it virtually immune to LMK. A plain Started Service only elevates to SERVICE priority; the system *may* kill it under memory pressure after it has been running for 30+ minutes, though it tries to restart it per `onStartCommand()`'s return value.

**Q8: How does `WorkManager` differ from a Foreground Service, and when should you use each?**

> **Foreground Service:** Immediate execution, user-visible notification required, survives process death via notification, best for time-sensitive user-initiated tasks (music, navigation, active upload). The user can see and control it. Not guaranteed to survive reboots without explicit re-scheduling. **WorkManager:** For deferrable, guaranteed background tasks. Work is scheduled via `JobScheduler` (or `AlarmManager`/`BroadcastReceiver` on older APIs). It survives process death *and* device reboots (for `PeriodicWorkRequest`). It respects system constraints (network availability, battery charging, storage). It is not immediate and can be deferred minutes or hours. WorkManager does not show a UI to the user by default, but you can combine it with a Foreground Service via `setForeground()` for long-running guaranteed tasks that also need a notification.

**Q9: What is the `onUnbind()` / `onRebind()` lifecycle and when is `onRebind()` called?**

> When all clients unbind from a Bound Service, `onUnbind(intent)` is called. If `onUnbind()` returns `true`, it signals the system that the Service *wants* to be notified when a new client re-binds. If a new client subsequently binds (after all previous clients unbound), `onRebind(intent)` is called instead of `onBind()`. If `onUnbind()` returns `false` (the default), subsequent bindings always call `onBind()`. The primary use case for `onRebind()` is performing initialization work only once (in `onBind()`) but needing to reset state when a new client session begins (in `onRebind()`).

---

## Scenario Questions

**Scenario 1:** Users report that music stops playing when the app is swiped away from Recents. You're using a plain `startService()`. What's wrong and how do you fix it?

> **Answer:** On API 26+, the system kills background services aggressively to conserve battery. A Service started with `startService()` runs at SERVICE process priority and can be killed when the app is swiped away from Recents (which destroys the app's task). The fix: convert to a **Foreground Service**. Call `startForegroundService()` from the client, then call `startForeground(notificationId, mediaNotification)` inside `onStartCommand()`. Use `MediaStyle` notification with `MediaSession` for a proper media control notification. Declare `android:foregroundServiceType="mediaPlayback"` in the manifest. The Foreground Service elevates process priority and the persistent notification informs the user that music is playing — they can dismiss it to stop playback.

**Scenario 2:** You have a Bound Service with an AIDL interface. Clients report occasional `DeadObjectException` when calling methods. What causes this and how do you handle it?

> **Answer:** `DeadObjectException` (a subclass of `RemoteException`) is thrown when the remote service process has died between the client obtaining the `IBinder` and the client calling a method on it. The `IBinder` became stale. Handling: (1) Wrap all remote calls in `try/catch (RemoteException)`. (2) In `ServiceConnection.onServiceDisconnected()`, null out the proxy and show an error state — the system will call this when the service process dies. (3) Re-bind in `onServiceDisconnected()` if the service should always be available. (4) Consider using a `Messenger` instead of AIDL for simpler use cases — it handles `RemoteException` more gracefully via its `Handler` abstraction.

**Scenario 3:** Your `WorkManager` task is being deferred for hours even when the device is plugged in and has Wi-Fi. The constraint is `NetworkType.CONNECTED`. What might be happening?

> **Answer:** Several possibilities: (1) **Battery optimization:** If the app is in Doze mode or Battery Saver, `WorkManager` may defer even CONNECTED-network tasks. Request exemption from battery optimization (with clear justification to users and Play Store). (2) **Expedited Work:** For urgent tasks, use `setExpedited(OutOfQuotaPolicy.RUN_AS_NON_EXPEDITED_WORK_REQUEST)` — this runs the task as an expedited job with higher priority. (3) **WorkManager quota:** On API 31+, apps have an expedited job quota. If exhausted, work is deferred. (4) **Constraint mismatch:** `CONNECTED` is satisfied by metered data too, but if you added `UNMETERED` and the current network is metered (even if Wi-Fi), it won't run. (5) **`PeriodicWork` minimum interval:** Periodic work has a minimum interval of 15 minutes, enforced by the system regardless of what you configure.

---

## Code Examples (Production-Quality Kotlin)

### Example 1: Basic Started Service with Coroutines

```kotlin
// Extend LifecycleService (from lifecycle-service artifact) to get
// a CoroutineScope (lifecycleScope) inside the Service.
// Add to build.gradle: implementation "androidx.lifecycle:lifecycle-service:2.7.0"

class FileProcessingService : LifecycleService() {

    companion object {
        const val ACTION_PROCESS = "action.PROCESS_FILE"
        const val EXTRA_FILE_PATH = "extra.FILE_PATH"

        fun createIntent(context: Context, filePath: String): Intent =
            Intent(context, FileProcessingService::class.java).apply {
                action = ACTION_PROCESS
                putExtra(EXTRA_FILE_PATH, filePath)
            }
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        super.onStartCommand(intent, flags, startId)

        val filePath = intent?.getStringExtra(EXTRA_FILE_PATH)
            ?: return START_NOT_STICKY

        lifecycleScope.launch {
            try {
                withContext(Dispatchers.IO) {
                    processFile(filePath)
                }
            } catch (e: Exception) {
                Log.e("FileProcessingService", "Error processing file: $filePath", e)
            } finally {
                // stopSelf with startId ensures the service stops only
                // after all prior commands have been processed.
                stopSelf(startId)
            }
        }

        return START_REDELIVER_INTENT
    }

    private suspend fun processFile(path: String) {
        // Heavy file I/O — already on Dispatchers.IO
        File(path).bufferedReader().use { reader ->
            reader.lineSequence().forEach { line ->
                // process each line
            }
        }
    }
}
```

### Example 2: Foreground Service with Notification

```kotlin
class MediaPlaybackService : LifecycleService() {

    companion object {
        const val CHANNEL_ID = "media_playback_channel"
        const val NOTIFICATION_ID = 101
        const val ACTION_PLAY = "action.PLAY"
        const val ACTION_PAUSE = "action.PAUSE"
        const val ACTION_STOP = "action.STOP"
        const val EXTRA_TRACK_TITLE = "extra.TRACK_TITLE"
        const val EXTRA_ARTIST = "extra.ARTIST"
    }

    private var currentTrackTitle: String = ""
    private var currentArtist: String = ""
    private var isPlaying: Boolean = false

    override fun onCreate() {
        super.onCreate()
        createNotificationChannel()
    }

    override fun onStartCommand(intent: Intent?, flags: Int, startId: Int): Int {
        super.onStartCommand(intent, flags, startId)

        when (intent?.action) {
            ACTION_PLAY -> {
                currentTrackTitle = intent.getStringExtra(EXTRA_TRACK_TITLE) ?: ""
                currentArtist = intent.getStringExtra(EXTRA_ARTIST) ?: ""
                isPlaying = true
                // Must call startForeground() within 5 seconds of startForegroundService()
                startForeground(NOTIFICATION_ID, buildNotification())
                startPlayback()
            }
            ACTION_PAUSE -> {
                isPlaying = false
                pausePlayback()
                // Update notification to reflect paused state
                updateNotification()
            }
            ACTION_STOP -> {
                stopForeground(STOP_FOREGROUND_REMOVE)
                stopSelf()
            }
        }

        return START_STICKY
    }

    private fun buildNotification(): Notification {
        val openAppIntent = PendingIntent.getActivity(
            this, 0,
            Intent(this, MainActivity::class.java),
            PendingIntent.FLAG_IMMUTABLE or PendingIntent.FLAG_UPDATE_CURRENT
        )

        val pauseIntent = PendingIntent.getService(
            this, 1,
            Intent(this, MediaPlaybackService::class.java).apply { action = ACTION_PAUSE },
            PendingIntent.FLAG_IMMUTABLE
        )

        val stopIntent = PendingIntent.getService(
            this, 2,
            Intent(this, MediaPlaybackService::class.java).apply { action = ACTION_STOP },
            PendingIntent.FLAG_IMMUTABLE
        )

        return NotificationCompat.Builder(this, CHANNEL_ID)
            .setContentTitle(currentTrackTitle)
            .setContentText(currentArtist)
            .setSmallIcon(R.drawable.ic_music_note)
            .setContentIntent(openAppIntent)
            .addAction(R.drawable.ic_pause, "Pause", pauseIntent)
            .addAction(R.drawable.ic_stop, "Stop", stopIntent)
            .setStyle(
                androidx.media.app.NotificationCompat.MediaStyle()
                    .setShowActionsInCompactView(0, 1)
            )
            .setOngoing(true)
            .build()
    }

    private fun updateNotification() {
        val notificationManager = getSystemService(NotificationManager::class.java)
        notificationManager.notify(NOTIFICATION_ID, buildNotification())
    }

    private fun createNotificationChannel() {
        if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.O) {
            val channel = NotificationChannel(
                CHANNEL_ID,
                "Media Playback",
                NotificationManager.IMPORTANCE_LOW // LOW to prevent sound on update
            ).apply {
                description = "Shows currently playing media"
            }
            getSystemService(NotificationManager::class.java)
                .createNotificationChannel(channel)
        }
    }

    private fun startPlayback() { /* Initialize MediaPlayer/ExoPlayer */ }
    private fun pausePlayback() { /* Pause MediaPlayer/ExoPlayer */ }

    override fun onDestroy() {
        super.onDestroy()
        // Release MediaPlayer/ExoPlayer resources
    }
}
```

### Example 3: Local Bound Service (IBinder Pattern)

```kotlin
// ─── Service side ─────────────────────────────────────────────────────────

class TimerService : Service() {

    // Binder that exposes this service instance directly to local clients.
    // This pattern only works within the same process.
    inner class LocalBinder : Binder() {
        fun getService(): TimerService = this@TimerService
    }

    private val binder = LocalBinder()

    private var elapsedSeconds: Long = 0L
    private var timerJob: Job? = null
    private val _elapsedFlow = MutableStateFlow(0L)
    val elapsedFlow: StateFlow<Long> = _elapsedFlow.asStateFlow()

    // CoroutineScope for the service's lifetime
    private val serviceScope = CoroutineScope(SupervisorJob() + Dispatchers.Default)

    override fun onBind(intent: Intent): IBinder = binder

    fun startTimer() {
        timerJob?.cancel()
        timerJob = serviceScope.launch {
            while (isActive) {
                delay(1_000L)
                elapsedSeconds++
                _elapsedFlow.value = elapsedSeconds
            }
        }
    }

    fun stopTimer() {
        timerJob?.cancel()
    }

    fun resetTimer() {
        timerJob?.cancel()
        elapsedSeconds = 0L
        _elapsedFlow.value = 0L
    }

    override fun onDestroy() {
        super.onDestroy()
        serviceScope.cancel() // cancel all coroutines when service dies
    }
}

// ─── Activity side ────────────────────────────────────────────────────────

class TimerActivity : AppCompatActivity() {

    private var timerService: TimerService? = null
    private var isBound = false

    private val serviceConnection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, binder: IBinder) {
            val localBinder = binder as TimerService.LocalBinder
            timerService = localBinder.getService()
            isBound = true

            // Observe timer updates from the bound service
            lifecycleScope.launch {
                repeatOnLifecycle(Lifecycle.State.STARTED) {
                    timerService?.elapsedFlow?.collect { seconds ->
                        binding.timerText.text = formatSeconds(seconds)
                    }
                }
            }
        }

        override fun onServiceDisconnected(name: ComponentName) {
            // Called when the service process crashes — not on unbind
            timerService = null
            isBound = false
        }
    }

    override fun onStart() {
        super.onStart()
        Intent(this, TimerService::class.java).also { intent ->
            bindService(intent, serviceConnection, Context.BIND_AUTO_CREATE)
        }
    }

    override fun onStop() {
        super.onStop()
        if (isBound) {
            unbindService(serviceConnection)
            isBound = false
        }
    }

    private fun formatSeconds(seconds: Long): String {
        val mins = seconds / 60
        val secs = seconds % 60
        return "%02d:%02d".format(mins, secs)
    }
}
```

### Example 4: Messenger-Based Remote Service (One-Way IPC)

```kotlin
// ─── Remote Service (in a separate process or app) ─────────────────────────

class RemoteCalculatorService : Service() {

    companion object {
        const val MSG_CALCULATE = 1
        const val MSG_RESULT = 2
        const val KEY_OPERAND_A = "key_a"
        const val KEY_OPERAND_B = "key_b"
        const val KEY_RESULT = "key_result"
    }

    // Handler that runs on the Main thread of the remote process
    private val incomingHandler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            when (msg.what) {
                MSG_CALCULATE -> {
                    val a = msg.data.getLong(KEY_OPERAND_A)
                    val b = msg.data.getLong(KEY_OPERAND_B)
                    val result = a + b

                    // Reply to the client using the replyTo Messenger
                    val replyMsg = Message.obtain(null, MSG_RESULT).apply {
                        data = Bundle().apply { putLong(KEY_RESULT, result) }
                    }
                    try {
                        msg.replyTo?.send(replyMsg)
                    } catch (e: RemoteException) {
                        Log.e("RemoteCalcService", "Client is dead", e)
                    }
                }
            }
        }
    }

    private val messenger = Messenger(incomingHandler)

    override fun onBind(intent: Intent): IBinder = messenger.binder
}

// ─── Client Activity ──────────────────────────────────────────────────────

class CalculatorClientActivity : AppCompatActivity() {

    private var remoteMessenger: Messenger? = null
    private var isBound = false

    // Messenger to receive results
    private val replyHandler = object : Handler(Looper.getMainLooper()) {
        override fun handleMessage(msg: Message) {
            if (msg.what == RemoteCalculatorService.MSG_RESULT) {
                val result = msg.data.getLong(RemoteCalculatorService.KEY_RESULT)
                binding.resultText.text = "Result: $result"
            }
        }
    }
    private val replyMessenger = Messenger(replyHandler)

    private val connection = object : ServiceConnection {
        override fun onServiceConnected(name: ComponentName, binder: IBinder) {
            remoteMessenger = Messenger(binder)
            isBound = true
        }

        override fun onServiceDisconnected(name: ComponentName) {
            remoteMessenger = null
            isBound = false
        }
    }

    override fun onStart() {
        super.onStart()
        bindService(
            Intent(this, RemoteCalculatorService::class.java),
            connection,
            Context.BIND_AUTO_CREATE
        )
    }

    override fun onStop() {
        super.onStop()
        if (isBound) {
            unbindService(connection)
            isBound = false
        }
    }

    fun calculate(a: Long, b: Long) {
        remoteMessenger?.let { messenger ->
            val msg = Message.obtain(null, RemoteCalculatorService.MSG_CALCULATE).apply {
                data = Bundle().apply {
                    putLong(RemoteCalculatorService.KEY_OPERAND_A, a)
                    putLong(RemoteCalculatorService.KEY_OPERAND_B, b)
                }
                replyTo = replyMessenger // messenger for the result
            }
            try {
                messenger.send(msg)
            } catch (e: RemoteException) {
                Log.e("CalculatorClient", "Remote service unavailable", e)
            }
        }
    }
}
```

### Example 5: WorkManager — Modern Alternative to IntentService

```kotlin
// ─── Work definition ──────────────────────────────────────────────────────

class SyncWorker(
    context: Context,
    params: WorkerParameters
) : CoroutineWorker(context, params) {

    companion object {
        const val KEY_USER_ID = "key_user_id"
        const val KEY_SYNC_RESULT = "key_sync_result"

        // Factory method for type-safe work request creation
        fun buildRequest(userId: String): OneTimeWorkRequest =
            OneTimeWorkRequestBuilder<SyncWorker>()
                .setInputData(workDataOf(KEY_USER_ID to userId))
                .setConstraints(
                    Constraints.Builder()
                        .setRequiredNetworkType(NetworkType.CONNECTED)
                        .setRequiresBatteryNotLow(true)
                        .build()
                )
                .setBackoffCriteria(
                    BackoffPolicy.EXPONENTIAL,
                    WorkRequest.MIN_BACKOFF_MILLIS,
                    TimeUnit.MILLISECONDS
                )
                .addTag("sync_$userId")
                .build()
    }

    override suspend fun doWork(): Result {
        val userId = inputData.getString(KEY_USER_ID)
            ?: return Result.failure()

        return try {
            // CoroutineWorker.doWork() runs on Dispatchers.Default by default.
            // Switch to IO for network/disk work.
            val syncedItemCount = withContext(Dispatchers.IO) {
                syncUserData(userId)
            }
            Result.success(
                workDataOf(KEY_SYNC_RESULT to syncedItemCount)
            )
        } catch (e: Exception) {
            Log.e("SyncWorker", "Sync failed for user $userId", e)
            // Retry up to 3 times (set in backoff criteria above)
            if (runAttemptCount < 3) Result.retry() else Result.failure()
        }
    }

    private suspend fun syncUserData(userId: String): Int {
        // Network call and database write
        return 42 // placeholder
    }
}

// ─── Enqueueing in ViewModel ───────────────────────────────────────────────

class SyncViewModel(application: Application) : AndroidViewModel(application) {

    private val workManager = WorkManager.getInstance(application)

    fun scheduleSyncForUser(userId: String) {
        val syncRequest = SyncWorker.buildRequest(userId)

        // enqueueUniqueWork prevents duplicate sync jobs for the same user
        workManager.enqueueUniqueWork(
            "sync_$userId",
            ExistingWorkPolicy.KEEP, // keep the existing job if already scheduled
            syncRequest
        )
    }

    // Observe work state in the UI
    fun getSyncState(userId: String): LiveData<List<WorkInfo>> =
        workManager.getWorkInfosByTagLiveData("sync_$userId")
}

// ─── Observing in Fragment ─────────────────────────────────────────────────

class SyncFragment : Fragment() {

    private val viewModel: SyncViewModel by viewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        viewModel.getSyncState("user_123").observe(viewLifecycleOwner) { workInfoList ->
            val workInfo = workInfoList.firstOrNull() ?: return@observe
            when (workInfo.state) {
                WorkInfo.State.ENQUEUED -> showStatus("Sync scheduled...")
                WorkInfo.State.RUNNING -> showStatus("Syncing...")
                WorkInfo.State.SUCCEEDED -> {
                    val count = workInfo.outputData.getInt(SyncWorker.KEY_SYNC_RESULT, 0)
                    showStatus("Synced $count items")
                }
                WorkInfo.State.FAILED -> showStatus("Sync failed")
                WorkInfo.State.CANCELLED -> showStatus("Sync cancelled")
                WorkInfo.State.BLOCKED -> showStatus("Waiting for constraints...")
            }
        }
    }

    private fun showStatus(message: String) { /* update UI */ }
}
```

---

## Best Practices

1. **Never do blocking I/O in a Service's `onStartCommand()` without launching a coroutine or thread.** The Service runs on the main thread.
2. **Extend `LifecycleService`** (from `lifecycle-service`) to get a `lifecycleScope` inside Services — the same coroutine lifecycle management available in Activities/Fragments.
3. **Always call `startForeground()` as the first statement in `onStartCommand()`** for Foreground Services — before any async work.
4. **Bind in `onStart()`, unbind in `onStop()`.** This correctly handles cases where the Activity goes to the background and the Service should release resources.
5. **Declare `android:foregroundServiceType`** in the manifest for Foreground Services on API 29+. Required types: `mediaPlayback`, `location`, `camera`, `microphone`, `dataSync`, `phoneCall`, `connectedDevice`, `mediaProjection`.
6. **Use `WorkManager` for deferrable tasks** instead of scheduling them in Services. WorkManager handles retries, constraints, and process-death survival automatically.
7. **Return `START_STICKY` only for services that truly should restart.** Most services should return `START_NOT_STICKY` or `START_REDELIVER_INTENT` to avoid unnecessary resource consumption.
8. **Clean up coroutine scopes in `onDestroy()`.** If you create a manual `CoroutineScope` in a Service, cancel it in `onDestroy()`.
9. **Avoid multiple simultaneous Foreground Services** — use a single service with a unified notification.
10. **Test Service binding thoroughly across configuration changes** — verify that binding is correctly re-established when Activities are recreated.

---

## Legacy vs Modern

| Concern | Legacy | Modern |
|---------|--------|--------|
| Background execution | `Service` (unrestricted) | Restricted; use `WorkManager` or Foreground Service |
| Simple async background | `IntentService` (**Deprecated API 30**) | `WorkManager` + `CoroutineWorker` |
| Long-running user-aware task | `Service` + `startForeground()` | Same, but with `LifecycleService` + `CoroutineScope` |
| IPC simple one-way | `Messenger` | `Messenger` (still valid) |
| IPC bidirectional | `AIDL` | `AIDL` (still valid) |
| Scheduled background work | `AlarmManager` + `BroadcastReceiver` + `Service` | `WorkManager` |
| In-process background | `HandlerThread` | Kotlin Coroutines |
| Thread management in Service | Raw `Thread` | `lifecycleScope.launch(Dispatchers.IO)` |

---

## Revision Notes

- **Service runs on the MAIN THREAD.** This cannot be stated often enough.
- Three lifecycle types: **Started** (independent), **Bound** (client-tied), **Foreground** (user-visible notification, elevated priority).
- `onStartCommand()` return: `START_STICKY` (restart with null), `START_NOT_STICKY` (don't restart), `START_REDELIVER_INTENT` (restart with last intent).
- **API 26+:** Background service restrictions. Use Foreground Service or WorkManager.
- **API 29+:** `android:foregroundServiceType` required in manifest.
- **API 33+:** `POST_NOTIFICATIONS` runtime permission required for any notification.
- **Process priority:** Foreground > Visible > Service > Cached > Empty.
- Foreground Service = FOREGROUND priority; Started Service = SERVICE priority.
- **Bind in `onStart()`, unbind in `onStop()`.** Always.
- `IntentService` deprecated API 30 → use `WorkManager` + `CoroutineWorker`.
- **`WorkManager`** = deferrable, guaranteed, constraint-aware, survives reboots.

---

## Key Takeaways

> **A Service is a component, not a thread.** It gives background work a home in the Android component lifecycle — but threading is your responsibility.

- Services outlive Activities but are still subject to process lifecycle pressure.
- Foreground Services are the explicit, user-visible contract for long-running background work on API 26+.
- The process lifecycle hierarchy determines which processes the LMK kills first — Foreground Services offer the strongest protection short of being the active foreground app.
- Bound Services are the foundation of Android's IPC system — from your app's local bindings to every system service (`LocationManager`, `AudioManager`, etc.).
- WorkManager is the preferred solution for any background work that doesn't require immediate, user-visible execution. It handles the hard problems: retries, constraints, process death, device reboots.
- The migration path from legacy: `IntentService` → `CoroutineWorker`, raw `Service` threads → `LifecycleService` + coroutines, `AlarmManager` + Service → `WorkManager`.

---

## Cross-Chapter Connections

| Topic | Connects To |
|-------|-------------|
| `PendingIntent` (Ch. 6) | Foreground Service notifications (Ch. 8) |
| `Handler` / `Looper` (Ch. 7) | Service main thread model (Ch. 8); `Messenger` IPC (Ch. 8) |
| `Coroutines` / `viewModelScope` (Ch. 7) | `LifecycleService` + `lifecycleScope` (Ch. 8); ViewModel (Part 3) |
| Process lifecycle (Ch. 8) | Activity lifecycle (Part 1); Memory management (Part 4) |
| `WorkManager` (Ch. 8) | Background constraints, `BroadcastReceiver` (next chapters) |

---

*End of Part 2 — Chapters 6, 7, 8*

*Next: Part 3 — Broadcast Receivers, Content Providers, Runtime Permissions (Chapters 9–11)*
