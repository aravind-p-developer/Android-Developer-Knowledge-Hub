# Android Fundamentals Master Guide
# Part 4: Architecture & Persistence (Chapters 12–16)

> **Coverage:** MVVM Architecture · ViewModel · SavedStateHandle & Process Death · Navigation Component · Data Persistence
> **Skill Level:** Beginner → Advanced
> **Last Updated:** June 2026

---

## Table of Contents

- [Chapter 12 — MVVM Architecture](#chapter-12--mvvm-architecture)
- [Chapter 13 — ViewModel](#chapter-13--viewmodel)
- [Chapter 14 — SavedStateHandle & Process Death](#chapter-14--savedstatehandle--process-death)
- [Chapter 15 — Navigation Component](#chapter-15--navigation-component)
- [Chapter 16 — Data Persistence](#chapter-16--data-persistence)

---

# Chapter 12 — MVVM Architecture

---

## Concept

**MVVM (Model–View–ViewModel)** is an architectural pattern that separates an application into three distinct layers, each with clearly defined responsibilities:

| Layer | Role | Android Equivalent |
|-------|------|-------------------|
| **Model** | Data, business logic, domain rules | Data classes, Repository, Room, Retrofit |
| **View** | Passive UI — renders state, emits events | Activity, Fragment, Composable |
| **ViewModel** | UI state holder, bridges Model and View | `androidx.lifecycle.ViewModel` |

The fundamental contract of MVVM is **separation of concerns**:

- The **View** knows nothing about the Model
- The **ViewModel** knows nothing about the Android UI framework (no `Context`, no `View` references)
- The **Model** knows nothing about the ViewModel or View

```
┌─────────────────────────────────────────────────┐
│                    MVVM Layers                  │
│                                                 │
│  ┌──────────┐   observes    ┌───────────────┐   │
│  │          │◄──────────────│               │   │
│  │   VIEW   │               │  VIEW MODEL   │   │
│  │          │──── events ──►│               │   │
│  │ Activity │               │  UI State     │   │
│  │ Fragment │               │  StateFlow    │   │
│  │Composable│               │  LiveData     │   │
│  └──────────┘               └───────┬───────┘   │
│                                     │           │
│                               calls │           │
│                                     ▼           │
│                          ┌──────────────────┐   │
│                          │      MODEL       │   │
│                          │                  │   │
│                          │   Repository     │   │
│                          │   Room Database  │   │
│                          │   Retrofit API   │   │
│                          │   DataStore      │   │
│                          └──────────────────┘   │
└─────────────────────────────────────────────────┘
```

---

## Why It Exists

Before MVVM, Android developers stuffed everything into Activities and Fragments — the "God Activity" anti-pattern. This created several critical problems:

| Problem | Consequence |
|---------|-------------|
| Business logic in Activity | Cannot unit-test without Android emulator |
| UI state re-fetched on rotation | Network calls on every config change |
| Direct database calls in UI layer | Slow UI thread, ANR risk |
| Fragment-to-fragment communication via casts | Tight coupling, crash-prone |
| No separation of concerns | Impossible to maintain or scale |

MVVM solves all of these by introducing:

1. **Testability** — ViewModel has no Android dependencies; unit-testable with JUnit
2. **Lifecycle awareness** — ViewModel survives configuration changes; no re-fetching
3. **Reactive data flow** — UI automatically updates when state changes
4. **Single Responsibility** — each layer has one job

---

## Internal Working

### The Repository Pattern

The Repository acts as a **single entry point** for all data operations. It abstracts the data sources from the ViewModel:

```
ViewModel
   │
   ▼
Repository (single entry point)
   ├──► Remote: Retrofit API calls
   ├──► Local: Room Database
   └──► Cache: In-memory, DataStore
```

The Repository decides where data comes from. The ViewModel simply asks "give me users" — it doesn't know or care whether data comes from cache, database, or network.

### Unidirectional Data Flow (UDF)

MVVM enforces **Unidirectional Data Flow** — data flows in one direction, events flow in the opposite direction:

```
┌─────────────────────────────────────────────────────┐
│              Unidirectional Data Flow               │
│                                                     │
│  ┌──────────┐                                       │
│  │          │──── User Events ──────────────────►   │
│  │  VIEW    │    (click, scroll, input)              │
│  │          │◄─── UI State ─────────────────────    │
│  └──────────┘    (loading, success, error)          │
│        ▲                     │                      │
│        │                     │                      │
│        │              ┌──────▼──────┐               │
│        │              │  VIEW MODEL │               │
│        │              │             │               │
│     emits             │  processes  │               │
│     state             │  events     │               │
│        │              └──────┬──────┘               │
│        │                     │                      │
│        │              ┌──────▼──────┐               │
│        │              │ REPOSITORY  │               │
│        │              │             │               │
│        └──────────────│  Room/API   │               │
│                       └─────────────┘               │
└─────────────────────────────────────────────────────┘
```

### Single Source of Truth (SSOT)

Room Database is the **Single Source of Truth**. Network responses are never exposed directly to the UI. Instead:

1. Network response → saved to Room
2. Room emits `Flow<List<T>>` → ViewModel collects
3. ViewModel exposes state → View observes

This means the UI always reflects what's in the database, ensuring consistency across restarts.

### State Exposure Pattern

```kotlin
// WRONG: Mutable state exposed directly
val users = MutableStateFlow<List<User>>(emptyList()) // ❌ Anyone can write

// CORRECT: Immutable public state backed by private mutable
private val _uiState = MutableStateFlow<UsersUiState>(UsersUiState.Loading)
val uiState: StateFlow<UsersUiState> = _uiState.asStateFlow() // ✅ Read-only
```

---

## Lifecycle / Flow (ASCII Diagrams)

### MVVM Lifecycle with Configuration Change

```
Activity Created
      │
      ▼
ViewModelProvider.get(UserViewModel::class.java)
      │
      ├── (first time) ──► Creates UserViewModel
      │
      └── (after rotation) ─► Returns SAME instance from ViewModelStore
                                        │
                                        ▼
                               UserViewModel alive
                               (data preserved)
```

### Data Flow Timeline

```
T=0  User opens screen
      │
      ▼
T=1  View subscribes to uiState
      │
      ▼
T=2  ViewModel.init{ } → Repository.getUsers()
      │
      ▼
T=3  Repository checks Room (empty) → triggers API call
      │
      ▼
T=4  API response → Repository saves to Room
      │
      ▼
T=5  Room Flow emits List<User>
      │
      ▼
T=6  ViewModel maps to UiState.Success
      │
      ▼
T=7  View receives new state → renders RecyclerView

T=8  User rotates device
      │
      ▼
T=9  Activity destroyed → ViewModel SURVIVES
      │
      ▼
T=10 New Activity created → subscribes to SAME ViewModel
      │
      ▼
T=11 StateFlow emits last value immediately (no network call)
```

---

## Real World Example

### User List Screen — Full MVVM Stack

**1. Domain Model**

```kotlin
data class User(
    val id: Int,
    val name: String,
    val email: String,
    val avatarUrl: String
)
```

**2. UI State**

```kotlin
sealed interface UsersUiState {
    data object Loading : UsersUiState
    data class Success(val users: List<User>) : UsersUiState
    data class Error(val message: String) : UsersUiState
}
```

**3. Repository Interface + Implementation**

```kotlin
interface UserRepository {
    fun getUsers(): Flow<List<User>>
    suspend fun refreshUsers()
}

class UserRepositoryImpl @Inject constructor(
    private val userDao: UserDao,
    private val apiService: UserApiService
) : UserRepository {

    // Room is the Single Source of Truth
    override fun getUsers(): Flow<List<User>> =
        userDao.getAllUsers().map { entities -> entities.map { it.toDomain() } }

    override suspend fun refreshUsers() {
        val remoteUsers = apiService.fetchUsers()
        userDao.insertAll(remoteUsers.map { it.toEntity() })
    }
}
```

**4. ViewModel**

```kotlin
@HiltViewModel
class UserListViewModel @Inject constructor(
    private val userRepository: UserRepository
) : ViewModel() {

    private val _uiState = MutableStateFlow<UsersUiState>(UsersUiState.Loading)
    val uiState: StateFlow<UsersUiState> = _uiState.asStateFlow()

    // One-time events (navigation, snackbars)
    private val _events = Channel<UserEvent>(Channel.BUFFERED)
    val events: Flow<UserEvent> = _events.receiveAsFlow()

    init {
        observeUsers()
        refreshUsers()
    }

    private fun observeUsers() {
        userRepository.getUsers()
            .onEach { users ->
                _uiState.value = if (users.isEmpty()) UsersUiState.Loading
                                 else UsersUiState.Success(users)
            }
            .catch { e -> _uiState.value = UsersUiState.Error(e.message ?: "Unknown error") }
            .launchIn(viewModelScope)
    }

    fun refreshUsers() {
        viewModelScope.launch {
            runCatching { userRepository.refreshUsers() }
                .onFailure { e ->
                    if (_uiState.value !is UsersUiState.Success) {
                        _uiState.value = UsersUiState.Error(e.message ?: "Refresh failed")
                    }
                    _events.send(UserEvent.ShowSnackbar("Refresh failed: ${e.message}"))
                }
        }
    }

    fun onUserClicked(user: User) {
        viewModelScope.launch {
            _events.send(UserEvent.NavigateToDetail(user.id))
        }
    }
}

sealed interface UserEvent {
    data class NavigateToDetail(val userId: Int) : UserEvent
    data class ShowSnackbar(val message: String) : UserEvent
}
```

**5. Fragment (View)**

```kotlin
@AndroidEntryPoint
class UserListFragment : Fragment(R.layout.fragment_user_list) {

    private val viewModel: UserListViewModel by viewModels()
    private lateinit var adapter: UserAdapter

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        super.onViewCreated(view, savedInstanceState)

        adapter = UserAdapter { user -> viewModel.onUserClicked(user) }
        binding.recyclerView.adapter = adapter

        // Observe UI state
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.uiState.collect { state ->
                    when (state) {
                        is UsersUiState.Loading -> showLoading()
                        is UsersUiState.Success -> showUsers(state.users)
                        is UsersUiState.Error   -> showError(state.message)
                    }
                }
            }
        }

        // Observe one-time events
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.events.collect { event ->
                    when (event) {
                        is UserEvent.NavigateToDetail ->
                            findNavController().navigate(
                                UserListFragmentDirections.actionToDetail(event.userId)
                            )
                        is UserEvent.ShowSnackbar ->
                            Snackbar.make(requireView(), event.message, Snackbar.LENGTH_SHORT).show()
                    }
                }
            }
        }

        binding.swipeRefresh.setOnRefreshListener { viewModel.refreshUsers() }
    }

    private fun showLoading() { binding.progressBar.isVisible = true }
    private fun showUsers(users: List<User>) {
        binding.progressBar.isVisible = false
        adapter.submitList(users)
    }
    private fun showError(message: String) {
        binding.progressBar.isVisible = false
        binding.errorText.text = message
    }
}
```

**6. Jetpack Compose Integration**

```kotlin
@Composable
fun UserListScreen(
    viewModel: UserListViewModel = hiltViewModel(),
    onNavigateToDetail: (Int) -> Unit
) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()

    // Collect one-time events
    val snackbarHostState = remember { SnackbarHostState() }
    val context = LocalContext.current

    LaunchedEffect(Unit) {
        viewModel.events.collect { event ->
            when (event) {
                is UserEvent.NavigateToDetail -> onNavigateToDetail(event.userId)
                is UserEvent.ShowSnackbar -> snackbarHostState.showSnackbar(event.message)
            }
        }
    }

    Scaffold(snackbarHost = { SnackbarHost(snackbarHostState) }) { padding ->
        when (val state = uiState) {
            is UsersUiState.Loading -> CircularProgressIndicator(modifier = Modifier.fillMaxSize())
            is UsersUiState.Success -> UserList(users = state.users, modifier = Modifier.padding(padding))
            is UsersUiState.Error   -> ErrorMessage(message = state.message)
        }
    }
}
```

---

## Common Mistakes

| Mistake | Why It's Wrong | Fix |
|---------|---------------|-----|
| Exposing `MutableStateFlow` publicly | Anyone can write to it; breaks encapsulation | Back with `asStateFlow()` |
| Calling `lifecycleScope.launch` without `repeatOnLifecycle` | Collects in background when UI is invisible | Always use `repeatOnLifecycle(STARTED)` |
| Using `this` instead of `viewLifecycleOwner` for LiveData | Fragment's view destroyed before fragment | Always `viewLifecycleOwner.lifecycleScope` |
| Doing network calls in ViewModel constructor with no error handling | Crashes on first launch | Use `init{}` with proper `runCatching` |
| One giant UiState data class with 30 fields | Hard to reason about, unnecessary recompositions | Decompose into smaller states |
| Holding Context in ViewModel | Memory leak — ViewModel outlives Activity | Use `AndroidViewModel` or Hilt's `@ApplicationContext` |
| Using `SingleLiveEvent` for navigation | Not Compose-friendly, subtle bugs | Use `Channel<Event>.receiveAsFlow()` |
| ViewModel directly accesses Database | Breaks Repository abstraction | Always go through Repository |

---

## Memory Leak / Performance Concerns

### 1. Context Leak via ViewModel

```kotlin
// ❌ DANGEROUS: Activity context leaked into ViewModel
class BadViewModel(private val context: Context) : ViewModel() {
    fun doSomething() { context.startActivity(...) } // crash after rotation!
}

// ✅ CORRECT: Application context is safe (lives as long as the app)
@HiltViewModel
class GoodViewModel @Inject constructor(
    @ApplicationContext private val context: Context
) : ViewModel()

// ✅ ALSO CORRECT: Use AndroidViewModel
class GoodViewModel2(application: Application) : AndroidViewModel(application) {
    fun doSomething() { getApplication<Application>().startActivity(...) }
}
```

### 2. Coroutine Scope Misuse

```kotlin
// ❌ WRONG: Global scope — not cancelled when ViewModel is cleared
class BadViewModel : ViewModel() {
    init {
        GlobalScope.launch { /* never cancelled! */ }
    }
}

// ✅ CORRECT: viewModelScope is cancelled in onCleared()
class GoodViewModel : ViewModel() {
    init {
        viewModelScope.launch { /* auto-cancelled on ViewModel death */ }
    }
}
```

### 3. Collecting Flow Without Lifecycle Awareness

```kotlin
// ❌ WRONG: Collects even when Fragment is in background
lifecycleScope.launch {
    viewModel.state.collect { render(it) }
}

// ✅ CORRECT: Pauses collection when lifecycle is below STARTED
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.state.collect { render(it) }
    }
}
```

### 4. Heavy Objects in UiState

```kotlin
// ❌ WRONG: Bitmap in UiState causes excessive memory pressure
data class UiState(val profileBitmap: Bitmap)

// ✅ CORRECT: Store URL, load image with Coil/Glide
data class UiState(val profileImageUrl: String)
```

---

## Interview Questions

### Beginner

**Q1: What are the three layers of MVVM and their responsibilities?**

> **Model** — data classes, repositories, business logic; knows nothing about UI.
> **View** — Activity/Fragment/Composable; renders state, forwards user events; knows nothing about Model.
> **ViewModel** — holds and manages UI state; exposes StateFlow/LiveData; calls Repository; has no Android UI dependencies.

**Q2: Why should you use `viewLifecycleOwner` instead of `this` when observing LiveData in a Fragment?**

> A Fragment instance can outlive its view (e.g., when added to the back stack). If you observe LiveData with `this` (the Fragment's lifecycle), the observer stays active even after the view is destroyed, potentially updating a null or recycled view. `viewLifecycleOwner` is tied to the view's lifecycle — it's destroyed and recreated with the view, preventing stale updates.

**Q3: What is the Repository pattern and why use it?**

> The Repository is a single class that abstracts all data sources (network, database, cache) from the ViewModel. The ViewModel always asks the Repository for data without knowing where it comes from. Benefits: testability (mock the Repository in tests), single place to define caching logic, easy to swap data sources.

---

### Intermediate

**Q4: What is Unidirectional Data Flow and how does MVVM enforce it?**

> UDF means data flows in one direction (ViewModel → View as state) and events flow in the opposite direction (View → ViewModel as user actions). MVVM enforces this by:
> 1. Making `StateFlow`/`LiveData` read-only from the View's perspective
> 2. The View never directly modifies state — it calls ViewModel functions
> 3. The ViewModel is the single place that transforms events into state changes

**Q5: How do you handle one-time UI events (navigation, Snackbars) in MVVM?**

> Using `Channel<Event>.receiveAsFlow()` in the ViewModel. A `Channel` with `BUFFERED` capacity queues events that arrive before the collector is active. The View collects this Flow inside `repeatOnLifecycle(STARTED)`. This avoids the problems of `SingleLiveEvent` (not multicast-safe) and SharedFlow (can miss events or replay unintentionally).

**Q6: What is Single Source of Truth and how does Room serve as SSOT?**

> SSOT means there is one authoritative source for any piece of data. With Room as SSOT: (1) API data is always written to Room first, never directly to the UI; (2) The UI observes a Room `Flow<List<T>>`, which emits whenever data changes; (3) There's no split-brain between "what the server returned" and "what's in the database" — they're always the same.

---

### Advanced

**Q7: Walk through the full lifecycle of a StateFlow from ViewModel to Compose UI, including what `collectAsStateWithLifecycle` does differently from `collectAsState`.**

> `collectAsState()` starts collecting when the Composable enters the composition and **never pauses** — even when the app is backgrounded, it keeps the coroutine alive and processes emissions. This wastes CPU and battery.
>
> `collectAsStateWithLifecycle()` (from `lifecycle-runtime-compose`) ties collection to the nearest `LifecycleOwner`. It automatically **suspends collection** when the lifecycle drops below `Lifecycle.State.STARTED` (app backgrounded) and **resumes** when it returns to STARTED. This mirrors `repeatOnLifecycle()` behavior in Fragment-based UI.

**Q8: How would you share a ViewModel between two Fragments without using `activityViewModels()`?**

> Use a Navigation graph-scoped ViewModel. Both fragments that need to share state are in the same nested navigation graph. Each Fragment accesses the ViewModel scoped to the NavBackStackEntry of that graph:
> ```kotlin
> val parentEntry = findNavController().getBackStackEntry(R.id.nested_graph)
> val sharedViewModel: SharedViewModel by viewModels({ parentEntry })
> ```
> This is more precise than `activityViewModels()` because the ViewModel is destroyed when the nested graph is popped, not when the entire Activity finishes.

---

## Scenario Questions

**Scenario 1:** Your app fetches a list of products. The user searches, and results must filter instantly. How would you model this in MVVM?

> Keep a `searchQuery: StateFlow<String>` in the ViewModel. Combine it with the product list from the repository using `combine()`:
> ```kotlin
> val filteredProducts = combine(
>     repository.getProducts(),
>     _searchQuery
> ) { products, query ->
>     products.filter { it.name.contains(query, ignoreCase = true) }
> }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())
> ```
> The UI only needs to call `viewModel.onSearchQueryChanged(text)`.

**Scenario 2:** Two fragments need to communicate (Fragment A selects an item, Fragment B displays details). How do you implement this without direct Fragment-to-Fragment communication?

> Use a **shared ViewModel** scoped to the parent Activity or a Navigation graph. Fragment A calls `sharedViewModel.selectUser(user)`. Fragment B observes `sharedViewModel.selectedUser`. No direct reference between fragments — they communicate through the ViewModel state.

**Scenario 3:** The PM says the app must work offline. Data must be available even without internet. How does MVVM architecture support this?

> With Room as SSOT and an offline-first Repository:
> 1. Always read from Room (always works offline)
> 2. Trigger a refresh from network in the background
> 3. If network fails, Room still has cached data — UI shows stale data with a "last updated" timestamp
> 4. Use `networkBoundResource()` pattern: emit local data immediately, then fetch remote and update DB

---

## Code Examples (Production-Quality Kotlin)

### networkBoundResource — Offline-First Pattern

```kotlin
fun <ResultType, RequestType> networkBoundResource(
    query: () -> Flow<ResultType>,
    fetch: suspend () -> RequestType,
    saveFetchResult: suspend (RequestType) -> Unit,
    shouldFetch: (ResultType) -> Boolean = { true }
): Flow<Resource<ResultType>> = flow {

    emit(Resource.Loading())

    val data = query().first()

    val flow = if (shouldFetch(data)) {
        emit(Resource.Loading(data))
        try {
            saveFetchResult(fetch())
            query().map { Resource.Success(it) }
        } catch (t: Throwable) {
            query().map { Resource.Error(t, it) }
        }
    } else {
        query().map { Resource.Success(it) }
    }

    emitAll(flow)
}

// Usage in Repository
fun getProducts(): Flow<Resource<List<Product>>> = networkBoundResource(
    query = { productDao.getAllProducts() },
    fetch = { apiService.fetchProducts() },
    saveFetchResult = { productDao.insertAll(it.map { dto -> dto.toEntity() }) },
    shouldFetch = { cachedProducts -> cachedProducts.isEmpty() || isCacheStale() }
)
```

### StateFlow with SharingStarted.WhileSubscribed

```kotlin
// Efficient: upstream only active when someone is collecting
val users: StateFlow<List<User>> = userRepository.getUsers()
    .stateIn(
        scope = viewModelScope,
        started = SharingStarted.WhileSubscribed(stopTimeoutMillis = 5_000L),
        initialValue = emptyList()
    )
// 5-second timeout: keeps upstream alive briefly after config change
// so rotation doesn't cancel and restart the upstream query
```

---

## Best Practices

1. **Define a `UiState` sealed interface or data class** for each screen — one object describes the complete screen state
2. **Use `SharingStarted.WhileSubscribed(5000)`** to avoid restarting upstream on config changes
3. **Expose only `StateFlow`/`Flow` from ViewModel** — never `MutableStateFlow` or `MutableLiveData`
4. **Use `Channel` for one-time events**, not `SharedFlow` (replay=0 can drop events if no collector)
5. **All network calls in Repository**, never in ViewModel directly
6. **Inject Repository via constructor** (with Hilt), not via `getInstance()` singletons
7. **Avoid `AndoirdViewModel` when possible** — prefer injecting `@ApplicationContext` via Hilt
8. **Test ViewModel in isolation** with `kotlinx-coroutines-test` and `Turbine` for Flow testing

---

## Legacy vs Modern

| Aspect | Legacy | Modern |
|--------|--------|--------|
| UI state | `LiveData<T>` | `StateFlow<T>` |
| One-time events | `SingleLiveEvent` | `Channel<Event>.receiveAsFlow()` |
| Observe in Fragment | `observe(this, { })` | `repeatOnLifecycle(STARTED)` |
| DI for ViewModel | `ViewModelFactory` boilerplate | `@HiltViewModel` + `@Inject` |
| Compose state | N/A | `collectAsStateWithLifecycle()` |
| Coroutine scope | Manual `Job` + `cancel()` | `viewModelScope` (built-in) |
| Error handling | try-catch everywhere | `runCatching`, `catch{}` operator |

---

## Revision Notes

- MVVM = Model (data) + View (passive UI) + ViewModel (state + logic bridge)
- ViewModel survives config changes; View is recreated but reattaches to same ViewModel
- Repository = single source of truth abstractor; Room = SSOT for data
- UDF: events go UP (View→VM), state goes DOWN (VM→View)
- Never expose mutable state; never hold Context/View in ViewModel
- `viewLifecycleOwner` in Fragment, not `this`
- `repeatOnLifecycle(STARTED)` for safe Flow collection
- One-time events via `Channel`, not `SharedFlow` or `SingleLiveEvent`

---

## Key Takeaways

> 🏗️ MVVM enforces separation of concerns — Model doesn't know about UI, View doesn't know about data sources, ViewModel knows only about data and state.

> 🔄 Unidirectional Data Flow makes state changes predictable and debuggable — follow the flow and you can always find where state changed.

> 📦 Repository pattern is non-negotiable in production apps — it's the single entry point for all data, enabling offline-first architecture and easy testing.

> ⚡ `SharingStarted.WhileSubscribed(5000)` is the production-grade starting strategy — it handles rotation efficiently without unnecessary network calls.

> 🚫 The three cardinal sins of MVVM: (1) holding Context in ViewModel, (2) collecting Flow without `repeatOnLifecycle`, (3) exposing mutable state publicly.

---
---

# Chapter 13 — ViewModel

---

## Concept

`ViewModel` is a Jetpack Architecture Component that holds and manages **UI-related data** in a lifecycle-conscious way. Its defining characteristic: it **survives configuration changes** (screen rotation, language change, split-screen) while the Activity or Fragment is destroyed and recreated.

```kotlin
class MyViewModel : ViewModel() {
    // Data here survives rotation
    private val _count = MutableStateFlow(0)
    val count: StateFlow<Int> = _count.asStateFlow()

    fun increment() { _count.value++ }

    override fun onCleared() {
        super.onCleared()
        // Called ONLY when Activity/Fragment is truly finished
        // Clean up non-coroutine resources here
    }
}
```

**Key guarantees:**
- Same ViewModel instance returned after configuration change
- `onCleared()` called only on true destruction (user pressing Back, `finish()`)
- `viewModelScope` automatically cancelled when ViewModel is cleared

---

## Why It Exists

Before ViewModel, developers handled configuration changes with:

```kotlin
// OLD way — extremely fragile
override fun onSaveInstanceState(outState: Bundle) {
    outState.putParcelableArrayList("users", ArrayList(userList)) // size limit!
    outState.putSerializable("selectedUser", selectedUser) // slow!
}

override fun onRestoreInstanceState(savedInstanceState: Bundle) {
    userList = savedInstanceState.getParcelableArrayList("users") ?: emptyList()
}
```

Problems with this approach:
1. **Bundle size limit** (~1MB) — large lists crash with `TransactionTooLargeException`
2. **No coroutine scope** — async operations cancelled on rotation, restarted unnecessarily
3. **Repeated network calls** — data re-fetched on every rotation
4. **Manual serialization** — error-prone, boilerplate-heavy
5. **Not testable** — logic embedded in lifecycle callbacks

ViewModel solves all of this by keeping data in memory, outside the Activity/Fragment lifecycle.

---

## Internal Working

### ViewModelStore and NonConfigurationInstances

This is the core mechanism. When an Activity undergoes a configuration change:

```
Activity.onRetainNonConfigurationInstance()
         │
         └──► Returns NonConfigurationInstances object
                    │
                    └──► Contains ViewModelStore
                              │
                              └──► HashMap<String, ViewModel>
                                        │
                                        └──► "MyViewModel" → instance
```

When the new Activity is created:

```
new Activity.getLastNonConfigurationInstance()
         │
         └──► Retrieves NonConfigurationInstances
                    │
                    └──► ViewModelStore (same HashMap!)
                              │
                              └──► ViewModelProvider.get("MyViewModel")
                                        │
                                        └──► Returns SAME ViewModel instance
```

The `ViewModelStore` itself lives in `ComponentActivity` which implements `ViewModelStoreOwner`. The `NonConfigurationInstances` object is passed by the Android framework to the new Activity instance — this is the secret handoff.

### ViewModelProvider Internals

```kotlin
// What happens when you call:
val vm: MyViewModel by viewModels()

// Expands to:
ViewModelProvider(this, defaultViewModelProviderFactory).get(MyViewModel::class.java)

// ViewModelProvider.get():
fun <T : ViewModel> get(modelClass: Class<T>): T {
    val key = "androidx.lifecycle.ViewModelProvider.DefaultKey:${modelClass.canonicalName}"
    var viewModel = store[key]
    if (viewModel == null || !modelClass.isInstance(viewModel)) {
        viewModel = factory.create(modelClass)
        store[key] = viewModel
    }
    return modelClass.cast(viewModel)!!
}
```

### Fragment ViewModel Scoping

Fragments have their own `ViewModelStore`, managed by the Fragment itself. The `FragmentManager` retains `ViewModelStore` for fragments that survive configuration changes (via `setRetainInstance` — now deprecated — or the new Fragment backstack mechanism).

```
FragmentActivity
     │
     ├── ViewModelStore (Activity-scoped)
     │        └── activityViewModels() ← shared across all fragments
     │
     └── FragmentManager
              │
              ├── FragmentA
              │      └── ViewModelStore (Fragment-scoped)
              │               └── viewModels() ← private to FragmentA
              │
              └── FragmentB
                     └── ViewModelStore (Fragment-scoped)
                              └── viewModels() ← private to FragmentB
```

---

## Lifecycle / Flow (ASCII Diagrams)

### ViewModel vs Activity Lifecycle

```
Activity Lifecycle          ViewModel Lifecycle
────────────────────        ────────────────────────────────────────────
onCreate()          ──────► ViewModel CREATED (first time)
onStart()
onResume()
                  [User rotates device]
onPause()
onStop()
onDestroy()         ─────── ViewModel SURVIVES (NonConfigurationInstances)
                  ▲
                  │ config change ─ ViewModel NOT destroyed
onCreate()          ──────► ViewModel RETRIEVED from store
onStart()
onResume()
                  [User presses Back / finish()]
onPause()
onStop()
onDestroy()         ──────► onCleared() called ──► ViewModel DESTROYED
```

### viewModelScope Lifecycle

```
ViewModel created
     │
     ▼
viewModelScope created (SupervisorJob + Dispatchers.Main.immediate)
     │
     ├── launch { networkCall() }  ─┐
     ├── launch { dbObserver() }   ─┤  All coroutines running
     └── launch { timer() }        ─┘
                                    │
                   [onCleared() called]
                                    │
                                    ▼
                        viewModelScope.cancel()
                        All child coroutines cancelled
```

### Custom Factory Flow

```
@HiltViewModel annotation
         │
         ▼
Hilt generates HiltViewModelFactory
         │
         ▼
ViewModelProvider(owner, hiltViewModelFactory).get(MyViewModel::class.java)
         │
         ▼
HiltViewModelFactory.create(MyViewModel::class.java)
         │
         ▼
Dagger graph satisfies @Inject constructor parameters
         │
         ▼
MyViewModel instance created and stored in ViewModelStore
```

---

## Real World Example

### Timer ViewModel (Demonstrates viewModelScope + onCleared)

```kotlin
@HiltViewModel
class TimerViewModel @Inject constructor() : ViewModel() {

    private val _elapsedSeconds = MutableStateFlow(0L)
    val elapsedSeconds: StateFlow<Long> = _elapsedSeconds.asStateFlow()

    private val _isRunning = MutableStateFlow(false)
    val isRunning: StateFlow<Boolean> = _isRunning.asStateFlow()

    private var timerJob: Job? = null

    fun startTimer() {
        if (timerJob?.isActive == true) return
        _isRunning.value = true
        timerJob = viewModelScope.launch {
            while (isActive) {
                delay(1_000)
                _elapsedSeconds.update { it + 1 }
            }
        }
    }

    fun pauseTimer() {
        timerJob?.cancel()
        _isRunning.value = false
    }

    fun resetTimer() {
        timerJob?.cancel()
        _isRunning.value = false
        _elapsedSeconds.value = 0
    }

    override fun onCleared() {
        super.onCleared()
        // timerJob is already cancelled via viewModelScope, but explicit cleanup
        // is good practice for clarity
        timerJob?.cancel()
    }
}
```

### Shared ViewModel Between Fragments

```kotlin
// Shared ViewModel scoped to Activity
@HiltViewModel
class CartViewModel @Inject constructor(
    private val cartRepository: CartRepository
) : ViewModel() {

    private val _cartItems = MutableStateFlow<List<CartItem>>(emptyList())
    val cartItems: StateFlow<List<CartItem>> = _cartItems.asStateFlow()

    val cartCount: StateFlow<Int> = cartItems
        .map { it.sumOf { item -> item.quantity } }
        .stateIn(viewModelScope, SharingStarted.Eagerly, 0)

    fun addToCart(product: Product) {
        viewModelScope.launch {
            cartRepository.addItem(product)
            _cartItems.value = cartRepository.getCartItems()
        }
    }
}

// ProductFragment — adds items
@AndroidEntryPoint
class ProductFragment : Fragment() {
    private val cartViewModel: CartViewModel by activityViewModels()

    fun onAddToCartClicked(product: Product) {
        cartViewModel.addToCart(product)
    }
}

// CartFragment — displays items
@AndroidEntryPoint
class CartFragment : Fragment() {
    private val cartViewModel: CartViewModel by activityViewModels()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                cartViewModel.cartItems.collect { items ->
                    adapter.submitList(items)
                }
            }
        }
    }
}
```

### Custom ViewModelFactory (Without Hilt)

```kotlin
class UserDetailViewModel(
    private val userId: Int,
    private val userRepository: UserRepository
) : ViewModel() {

    val user: StateFlow<User?> = userRepository.getUserById(userId)
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)

    class Factory(
        private val userId: Int,
        private val userRepository: UserRepository
    ) : ViewModelProvider.Factory {
        @Suppress("UNCHECKED_CAST")
        override fun <T : ViewModel> create(modelClass: Class<T>): T {
            require(modelClass.isAssignableFrom(UserDetailViewModel::class.java))
            return UserDetailViewModel(userId, userRepository) as T
        }
    }
}

// Usage
val viewModel: UserDetailViewModel by viewModels {
    UserDetailViewModel.Factory(userId = args.userId, userRepository = repository)
}
```

---

## Common Mistakes

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| `new ViewModel()` directly | New instance every time, defeats purpose | Always use `ViewModelProvider` / delegates |
| Passing Fragment/Activity to ViewModel constructor | Immediate memory leak | Never; use `@ApplicationContext` or Hilt |
| Using `lifecycleScope` instead of `viewModelScope` | Coroutine cancelled on config change | Always `viewModelScope` in ViewModel |
| Accessing ViewModel after `onDestroy()` | May get cleared/wrong state | Only access inside lifecycle callbacks |
| `activityViewModels()` for everything | ViewModel never cleared, stale state across screens | Use `viewModels()` unless truly shared |
| Creating ViewModel in `onCreateView()` | Works, but `onViewCreated()` is the contract | Initialize UI observers in `onViewCreated()` |
| Heavy init work in `ViewModel.init{}` with no error handling | Uncaught exception crashes coroutine silently | Always `runCatching` or `.catch{}` |

---

## Memory Leak / Performance Concerns

### 1. Never Store View References

```kotlin
// ❌ CRASH + MEMORY LEAK
class BadViewModel : ViewModel() {
    var textView: TextView? = null // View holds Activity context!

    fun updateText(text: String) {
        textView?.text = text // ViewModel outlives the Activity!
    }
}

// ✅ CORRECT: Expose state, let View update itself
class GoodViewModel : ViewModel() {
    private val _text = MutableStateFlow("")
    val text: StateFlow<String> = _text.asStateFlow()

    fun updateText(text: String) { _text.value = text }
}
```

### 2. AndroidViewModel vs Regular ViewModel

```kotlin
// AndroidViewModel gives Application context — safe because Application lives as long as the app
class FileViewModel(application: Application) : AndroidViewModel(application) {
    fun readFile(): String {
        val file = File(getApplication<Application>().filesDir, "data.txt")
        return file.readText()
    }
}
// ⚠️ Prefer Hilt with @ApplicationContext injection over AndroidViewModel when possible
// AndroidViewModel makes unit testing harder (requires robolectric or instrumented test)
```

### 3. Coroutine Exception Propagation

```kotlin
// ❌ Uncaught exception kills the entire viewModelScope
viewModelScope.launch {
    repository.fetchUsers() // throws IOException — kills scope!
}

// ✅ Use SupervisorScope or catch exceptions
viewModelScope.launch {
    runCatching { repository.fetchUsers() }
        .onSuccess { _uiState.value = UiState.Success(it) }
        .onFailure { _uiState.value = UiState.Error(it.message ?: "Error") }
}
```

---

## Interview Questions

### Beginner

**Q1: What is a ViewModel and why does it survive rotation?**

> A ViewModel is a Jetpack class that holds UI state outside the Activity/Fragment lifecycle. It survives rotation because Android stores the `ViewModelStore` (a HashMap of ViewModels) in `NonConfigurationInstances` — a mechanism that transfers data from the destroyed Activity to the recreated one. The ViewModel itself is never destroyed during a config change, only during true destruction (Back press, `finish()`).

**Q2: What is `viewModelScope`?**

> `viewModelScope` is a `CoroutineScope` that is automatically tied to the ViewModel's lifecycle. It uses `Dispatchers.Main.immediate` and a `SupervisorJob`. When `onCleared()` is called (ViewModel destroyed), `viewModelScope` is automatically cancelled, stopping all coroutines started within it.

**Q3: How do you get a ViewModel instance?**

```kotlin
// In Activity:
val viewModel: MyViewModel by viewModels()

// In Fragment (own ViewModel):
val viewModel: MyViewModel by viewModels()

// In Fragment (shared with Activity):
val viewModel: MyViewModel by activityViewModels()

// With custom factory:
val viewModel: MyViewModel by viewModels { MyViewModel.Factory(arg1, arg2) }
```

---

### Intermediate

**Q4: Explain the difference between `viewModels()` and `activityViewModels()` in terms of ViewModel scope and lifecycle.**

> `viewModels()` creates a ViewModel scoped to the Fragment's `ViewModelStore`. It's created when first accessed and destroyed when the Fragment is permanently destroyed (not just back-stacked). `activityViewModels()` scopes the ViewModel to the Activity's `ViewModelStore`, keeping it alive as long as the Activity is alive — even when all fragments are replaced. Use `viewModels()` for screen-specific state; `activityViewModels()` only for truly shared state between fragments.

**Q5: Why can't you just pass a `userId` directly to the ViewModel constructor without a Factory?**

> By default, `ViewModelProvider` uses a `DefaultViewModelProviderFactory` that calls the ViewModel's no-arg constructor via reflection. If your ViewModel has constructor parameters, the default factory throws `InstantiationException`. You must provide a `ViewModelProvider.Factory` (or use Hilt's `@HiltViewModel` with `@AssistedInject` for runtime parameters) so that `ViewModelProvider` knows how to create the instance.

---

### Advanced

**Q6: Describe exactly how `NonConfigurationInstances` enables ViewModel survival through configuration changes.**

> When `ComponentActivity.onRetainNonConfigurationInstance()` is called during a config change, it packages the `ViewModelStore` into a `NonConfigurationInstances` object and returns it. The Android framework keeps this object alive in the `ActivityClientRecord` on the `ActivityThread` — it's not serialized, it's a direct object reference kept in memory. When the new Activity instance is created, `getLastNonConfigurationInstance()` retrieves this object. `ComponentActivity`'s `ensureViewModelStore()` extracts the `ViewModelStore` from it. Subsequent `ViewModelProvider.get()` calls find existing ViewModel instances in this store.

**Q7: How does `viewModelScope` work internally, and what is `SupervisorJob`'s role?**

> `viewModelScope` is a `CloseableCoroutineScope` created lazily as:
> ```kotlin
> CoroutineScope(SupervisorJob() + Dispatchers.Main.immediate)
> ```
> `SupervisorJob` means child coroutines are independent — if one fails, it doesn't cancel siblings. This is crucial: if a network call coroutine crashes, other ViewModel coroutines keep running. When `onCleared()` is called, `viewModelScope.cancel()` is invoked, which cancels the `SupervisorJob` and propagates cancellation to all children.

---

## Scenario Questions

**Scenario 1:** Your ViewModel fetches data in `init{}`. The user rotates the device. What happens?

> The ViewModel instance survives — `init{}` is NOT called again. The existing `StateFlow` already holds the fetched data. The new Fragment/Activity subscribes to the existing `StateFlow` and immediately receives the last emitted value. No new network call is made. This is exactly the optimization ViewModel provides.

**Scenario 2:** You need to pass a `productId: Int` to a ViewModel when using Hilt. How?

> Use Hilt's Navigation argument injection (for Navigation Component) or `SavedStateHandle`:
> ```kotlin
> @HiltViewModel
> class ProductDetailViewModel @Inject constructor(
>     savedStateHandle: SavedStateHandle,
>     private val repository: ProductRepository
> ) : ViewModel() {
>     private val productId: Int = checkNotNull(savedStateHandle["productId"])
>     val product = repository.getProductById(productId)
>         .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)
> }
> ```
> Hilt's Navigation integration automatically populates `SavedStateHandle` from navigation arguments.

---

## Code Examples (Production-Quality Kotlin)

### Pagination with ViewModel and Paging 3

```kotlin
@HiltViewModel
class NewsViewModel @Inject constructor(
    private val newsRepository: NewsRepository
) : ViewModel() {

    val newsFlow: Flow<PagingData<Article>> = Pager(
        config = PagingConfig(
            pageSize = 20,
            enablePlaceholders = false,
            prefetchDistance = 3
        )
    ) {
        newsRepository.getNewsPagingSource()
    }.flow
        .cachedIn(viewModelScope) // Cache PagingData in ViewModel scope — survives rotation!
}

// Fragment
class NewsFragment : Fragment() {
    private val viewModel: NewsViewModel by viewModels()
    private val adapter = NewsAdapter()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.recyclerView.adapter = adapter.withLoadStateHeaderAndFooter(
            header = NewsLoadStateAdapter { adapter.retry() },
            footer = NewsLoadStateAdapter { adapter.retry() }
        )

        viewLifecycleOwner.lifecycleScope.launch {
            viewLifecycleOwner.repeatOnLifecycle(Lifecycle.State.STARTED) {
                viewModel.newsFlow.collectLatest { pagingData ->
                    adapter.submitData(pagingData)
                }
            }
        }
    }
}
```

### Multiple StateFlows Combined

```kotlin
@HiltViewModel
class DashboardViewModel @Inject constructor(
    private val userRepository: UserRepository,
    private val statsRepository: StatsRepository
) : ViewModel() {

    private val _user = userRepository.getCurrentUser()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)

    private val _stats = statsRepository.getDashboardStats()
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), null)

    val uiState: StateFlow<DashboardUiState> = combine(_user, _stats) { user, stats ->
        when {
            user == null || stats == null -> DashboardUiState.Loading
            else -> DashboardUiState.Success(user, stats)
        }
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), DashboardUiState.Loading)
}
```

---

## Best Practices

1. **Use `by viewModels()` delegate** — it's lazy, property-caching, and scope-correct
2. **Always use `viewModelScope`** for coroutines in ViewModel
3. **`onCleared()` for non-coroutine cleanup** — `MediaPlayer`, `WebSocket`, `BluetoothGatt` etc.
4. **Use `SharingStarted.WhileSubscribed(5_000L)`** for StateFlows backed by DB/network
5. **Prefer `@HiltViewModel`** over manual factory — eliminates boilerplate
6. **Don't put navigation logic in ViewModel** — emit events, let View navigate
7. **Unit-test ViewModels** with `TestCoroutineScheduler` and `Turbine`
8. **Use `activityViewModels()` sparingly** — prefer Navigation-graph-scoped ViewModels

---

## Legacy vs Modern

| Aspect | Legacy | Modern |
|--------|--------|--------|
| Getting ViewModel | `ViewModelProviders.of(this)` ⚠️ deprecated | `by viewModels()` delegate |
| DI | Manual `ViewModelProvider.Factory` | `@HiltViewModel` + `@Inject` |
| Coroutine scope | Manual `Job()` in `init`, cancel in `onCleared()` | `viewModelScope` (built-in) |
| State type | `LiveData<T>` | `StateFlow<T>` |
| `setRetainInstance(true)` | Old fragment retention ⚠️ deprecated | ViewModel + `ViewModelStore` |
| Cross-fragment comms | `interface` callbacks | Shared ViewModel |

> ⚠️ `ViewModelProviders.of()` was deprecated in lifecycle 2.2.0. Use `ViewModelProvider(this)` or property delegates.

---

## Revision Notes

- ViewModel survives config changes via `NonConfigurationInstances` → `ViewModelStore`
- ViewModel is destroyed only on `finish()`, Back press, or explicit `clear()` call
- `viewModelScope` = `SupervisorJob + Dispatchers.Main.immediate`; cancelled in `onCleared()`
- Never hold Context, View, Activity, or Fragment references in ViewModel
- `viewModels()` = Fragment/Activity scoped; `activityViewModels()` = Activity scoped
- Custom constructor params require `ViewModelProvider.Factory` or `@HiltViewModel`
- `ViewModelStore` is a `HashMap<String, ViewModel>` keyed by class canonical name

---

## Key Takeaways

> 🔄 ViewModel survives configuration changes by living outside the Activity lifecycle — in `NonConfigurationInstances`. This is a framework mechanism, not magic.

> 🧹 `onCleared()` is your cleanup hook — use it for non-coroutine resources. Coroutines in `viewModelScope` are auto-cancelled.

> 🚫 Never hold UI references in ViewModel — expose state, let the View observe and update itself.

> 📡 `viewModelScope` + `StateFlow` + `SharingStarted.WhileSubscribed(5000)` is the golden combination for production ViewModel state management.

---
---

# Chapter 14 — SavedStateHandle & Process Death

---

## Concept

`SavedStateHandle` is a key-value map that survives **both** configuration changes AND **process death**. It bridges two separate survival mechanisms:

| Scenario | ViewModel survives? | SavedStateHandle survives? |
|----------|--------------------|-----------------------------|
| Screen rotation | ✅ Yes | ✅ Yes |
| Language change | ✅ Yes | ✅ Yes |
| App backgrounded (memory pressure → process killed) | ❌ No | ✅ Yes |
| User presses Back | ❌ No | ❌ No (intentional) |
| User force-kills app | ❌ No | ❌ No |

The key insight: **Process death is silent**. Android kills your process without warning when memory is needed. The next time the user returns to the app, they expect to see the same screen with the same state — even though your ViewModel was destroyed.

```kotlin
@HiltViewModel
class SearchViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    // Automatically persists across process death!
    var searchQuery: String
        get() = savedStateHandle["search_query"] ?: ""
        set(value) { savedStateHandle["search_query"] = value }

    // Or as a StateFlow
    val searchQueryFlow: StateFlow<String> =
        savedStateHandle.getStateFlow("search_query", initialValue = "")
}
```

---

## Why It Exists

### The Gap Between ViewModel and onSaveInstanceState

Before `SavedStateHandle`:

```
Config Change:    ViewModel alive ──────────────────────────────────►
                                  Config change → same ViewModel ✅

Process Death:    ViewModel dies ─────────────────────────────────────►
                                  Process killed → ViewModel recreated ❌
                                  (data gone!)

onSaveInstanceState: saves data → Bundle → survives process death ✅
                                         but: size limit, manual serialization ❌
```

`SavedStateHandle` combines both: ViewModel's convenience with `onSaveInstanceState`'s durability. It uses the same `Bundle` mechanism internally but packages it cleanly into the ViewModel.

### What Process Death Actually Means

```
User: Opens app to SearchScreen
      Types "android tutorial" in search box
      Presses Home (app backgrounded)
      [Hours later, system is low on memory]
      System: *silently kills your process*
      User: Taps app icon again
      Expected: SearchScreen with "android tutorial" still typed
      Without SavedStateHandle: Empty search box ← BAD UX
      With SavedStateHandle: "android tutorial" restored ← CORRECT
```

---

## Internal Working

### SavedStateRegistry

`SavedStateHandle` is backed by `SavedStateRegistry`, which is part of `LifecycleOwner` (Activity/Fragment). Here's the complete chain:

```
Activity.onCreate(savedInstanceState)
         │
         ▼
ComponentActivity → implements SavedStateRegistryOwner
         │
         ▼
SavedStateRegistryController.performRestore(savedInstanceState)
         │  Deserializes Bundle into SavedStateRegistry
         ▼
SavedStateRegistry (per LifecycleOwner)
         │
         ▼
AbstractSavedStateViewModelFactory
         │  Creates SavedStateHandle from registry
         ▼
SavedStateHandle (injected into ViewModel)
         │
         ▼
ViewModel stores/reads key-value pairs
         │
         ▼
Activity.onSaveInstanceState(outState)
         │
         ▼
SavedStateRegistryController.performSave(outState)
         │  Serializes SavedStateHandle data into Bundle
         ▼
Bundle stored by Android framework
```

### Hilt Integration

When using `@HiltViewModel`, Hilt's `HiltViewModelFactory` wraps `AbstractSavedStateViewModelFactory`. The `SavedStateHandle` is automatically created and injected — you just declare it in the constructor.

### Supported Types

```kotlin
// Only these types can be stored in SavedStateHandle:
// (because internally it's a Bundle)

savedStateHandle["key"] = 42                     // Int ✅
savedStateHandle["key"] = "string"               // String ✅
savedStateHandle["key"] = true                   // Boolean ✅
savedStateHandle["key"] = 3.14f                  // Float ✅
savedStateHandle["key"] = myParcelable           // Parcelable ✅
savedStateHandle["key"] = mySerializable         // Serializable ✅
savedStateHandle["key"] = intArrayOf(1, 2, 3)    // Primitive arrays ✅
savedStateHandle["key"] = ArrayList<String>()    // ArrayList ✅

// NOT supported:
savedStateHandle["key"] = myLambda              // Lambda ❌
savedStateHandle["key"] = myArbitraryObject     // Non-Parcelable object ❌
savedStateHandle["key"] = largeList            // Risk: Bundle size limit ❌
```

### Reading as StateFlow (Modern API)

```kotlin
// Creates a StateFlow backed by SavedStateHandle
val filterFlow: StateFlow<FilterOptions> =
    savedStateHandle.getStateFlow("filter", FilterOptions.default())

// Writing updates both the StateFlow AND persists to the Bundle
savedStateHandle["filter"] = FilterOptions(category = "tech")
```

---

## Lifecycle / Flow (ASCII Diagrams)

### Normal Config Change Flow

```
User opens app
     │
     ▼
Activity.onCreate(savedInstanceState = null)
     │
     ▼
SavedStateHandle created (empty)
ViewModel created with empty SavedStateHandle
     │
     ▼
User interacts → savedStateHandle["query"] = "android"
     │
     ▼
User rotates device
     │
     ▼
Activity.onSaveInstanceState() called
SavedStateHandle serialized → Bundle (contains "query"="android")
     │
     ▼
Activity destroyed → ViewModel SURVIVES (config change)
     │
     ▼
New Activity.onCreate(savedInstanceState = bundle)
     │
     ▼
SavedStateHandle RESTORED from bundle (has "query"="android")
Same ViewModel retrieved (already has this SavedStateHandle)
     │
     ▼
UI restored: search box shows "android" ✅
```

### Process Death Flow

```
User opens app → interacts → savedStateHandle["query"] = "android"
     │
     ▼
User presses Home → app backgrounded
     │
     ▼
System needs memory → kills process silently
     │
     ▼
ViewModel DESTROYED (process dead)
BUT: Bundle was already serialized in onSaveInstanceState!
     │
     ▼
User taps app icon → NEW process starts
     │
     ▼
Activity.onCreate(savedInstanceState = bundle) // bundle from before death!
     │
     ▼
SavedStateHandle RESTORED from bundle
NEW ViewModel created with RESTORED SavedStateHandle
     │
     ▼
savedStateHandle["query"] == "android" ✅
UI shows previous search ✅
```

---

## Real World Example

### Full Screen State Preservation

```kotlin
@HiltViewModel
class ProductListViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val productRepository: ProductRepository
) : ViewModel() {

    companion object {
        private const val KEY_SEARCH_QUERY = "search_query"
        private const val KEY_SELECTED_CATEGORY = "selected_category"
        private const val KEY_SORT_ORDER = "sort_order"
    }

    // All UI state that should survive process death
    val searchQuery: StateFlow<String> =
        savedStateHandle.getStateFlow(KEY_SEARCH_QUERY, "")

    val selectedCategory: StateFlow<String> =
        savedStateHandle.getStateFlow(KEY_SELECTED_CATEGORY, "All")

    val sortOrder: StateFlow<SortOrder> =
        savedStateHandle.getStateFlow(KEY_SORT_ORDER, SortOrder.NAME_ASC)

    // Derived state combining all filters
    val products: StateFlow<List<Product>> = combine(
        searchQuery,
        selectedCategory,
        sortOrder
    ) { query, category, sort ->
        Triple(query, category, sort)
    }.flatMapLatest { (query, category, sort) ->
        productRepository.getProducts(query, category, sort)
    }.stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onSearchQueryChanged(query: String) {
        savedStateHandle[KEY_SEARCH_QUERY] = query
    }

    fun onCategorySelected(category: String) {
        savedStateHandle[KEY_SELECTED_CATEGORY] = category
    }

    fun onSortOrderChanged(sortOrder: SortOrder) {
        savedStateHandle[KEY_SORT_ORDER] = sortOrder
    }
}

// SortOrder must be Serializable or Parcelable
enum class SortOrder : Serializable {
    NAME_ASC, NAME_DESC, PRICE_ASC, PRICE_DESC
}
```

### Parcelable Custom Object

```kotlin
@Parcelize
data class FormState(
    val name: String = "",
    val email: String = "",
    val phoneNumber: String = "",
    val agreedToTerms: Boolean = false
) : Parcelable

@HiltViewModel
class RegistrationViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle
) : ViewModel() {

    private companion object { const val KEY_FORM = "form_state" }

    val formState: StateFlow<FormState> =
        savedStateHandle.getStateFlow(KEY_FORM, FormState())

    fun updateName(name: String) = updateForm { it.copy(name = name) }
    fun updateEmail(email: String) = updateForm { it.copy(email = email) }
    fun updatePhone(phone: String) = updateForm { it.copy(phoneNumber = phone) }
    fun toggleTerms() = updateForm { it.copy(agreedToTerms = !it.agreedToTerms) }

    private fun updateForm(transform: (FormState) -> FormState) {
        val current = savedStateHandle.get<FormState>(KEY_FORM) ?: FormState()
        savedStateHandle[KEY_FORM] = transform(current)
    }
}
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| Storing large Parcelable lists in SavedStateHandle | `TransactionTooLargeException` — Bundle limit ~1MB | Store only IDs or small primitives; rebuild list from Room |
| Using SavedStateHandle as primary data store | Not intended for large datasets | Use Room/DataStore for data; SavedStateHandle for UI state only |
| Not using `getStateFlow()` | Must manually bridge to StateFlow | Use `getStateFlow(key, default)` for reactive reads |
| Storing non-Parcelable/Serializable types | `ClassCastException` at runtime | Use `@Parcelize` or primitive types |
| Forgetting SavedStateHandle for critical UI state | UI state lost on process death | Identify ALL user-input that should survive; use SavedStateHandle |
| Using SavedStateHandle for auth tokens/sensitive data | Bundle not encrypted | Use EncryptedSharedPreferences or EncryptedDataStore |

---

## Memory Leak / Performance Concerns

1. **Bundle size limit**: The Bundle used by `onSaveInstanceState()` has a ~1MB limit. Exceeding it causes `TransactionTooLargeException`. Only store primitive UI state (query strings, selected IDs, scroll position offsets).

2. **Avoid serialization of entire data models**: Store `userId: Int`, not `user: User`. Reconstruct the full object from Room/network.

3. **No heavy objects**: Never store Bitmaps, file descriptors, or large byte arrays.

```kotlin
// ❌ TOO MUCH DATA — risk of TransactionTooLargeException
savedStateHandle["users"] = ArrayList(users) // 500 users × Parcelable overhead!

// ✅ CORRECT — store only the ID, fetch user from Room
savedStateHandle["selected_user_id"] = selectedUser.id
```

---

## Interview Questions

### Beginner

**Q1: What is the difference between ViewModel and SavedStateHandle in terms of what they survive?**

> ViewModel survives configuration changes (rotation) but is destroyed when the process is killed. `SavedStateHandle` survives both — configuration changes AND process death — because it uses the same `Bundle` serialization mechanism as `onSaveInstanceState()`. The combination of ViewModel + `SavedStateHandle` gives you the best of both worlds: in-memory efficiency for config changes, Bundle persistence for process death.

**Q2: What types can you store in SavedStateHandle?**

> Only types that can be put in a `Bundle`: primitives (Int, String, Boolean, Float, Long), `Parcelable` objects (including `@Parcelize` data classes), `Serializable` objects, and primitive arrays. Non-serializable arbitrary objects cannot be stored.

---

### Intermediate

**Q3: How does `savedStateHandle.getStateFlow(key, default)` work?**

> `getStateFlow()` creates a `StateFlow` backed by the SavedStateHandle entry. Reading from the StateFlow reads from the SavedStateHandle. Writing to `savedStateHandle[key] = value` updates both the backing storage (the Bundle) AND emits a new value from the StateFlow. This makes it reactive — any collector of the StateFlow automatically sees updates when you write to the SavedStateHandle.

**Q4: When would you use SavedStateHandle vs Room database for persistence?**

> `SavedStateHandle` is for **transient UI state** — data that the user entered or navigated to that should survive process death to restore UX. It's limited to ~1MB, stores only Parcelable/primitives, and is cleared when the user explicitly finishes the screen.
>
> **Room** is for **persistent application data** — data that should exist even after the user closes the app completely, or that needs to be queried/filtered. Room handles large datasets, complex queries, migrations, and typed access.
>
> Example: `searchQuery: String` → SavedStateHandle. `List<Products>` → Room.

---

### Advanced

**Q5: Walk through the internal mechanism by which SavedStateHandle survives process death.**

> 1. `ComponentActivity` implements `SavedStateRegistryOwner` and holds a `SavedStateRegistryController`
> 2. When the ViewModel is created via `AbstractSavedStateViewModelFactory`, it receives a `SavedStateHandle` initialized from the current `SavedStateRegistry`'s restored bundle
> 3. Whenever `savedStateHandle[key] = value` is called, the `SavedStateHandle` sets the value in its internal `MutableMap` and notifies the `SavedStateRegistry`
> 4. When `Activity.onSaveInstanceState(outState)` is called, the `SavedStateRegistryController.performSave(outState)` collects all registered providers (including the ViewModel's SavedStateHandle data) into the Bundle
> 5. Android framework stores this Bundle in the Activity's stack entry
> 6. On process death + restore, `Activity.onCreate(savedInstanceState)` receives this Bundle, `performRestore()` deserializes it into the `SavedStateRegistry`, and the new ViewModel is created with a restored `SavedStateHandle`

---

## Scenario Questions

**Scenario 1:** Your app shows a multi-step form (3 steps). The user fills in step 1 and 2, then gets a phone call. The system kills your app. When they return, how do you restore their form state?

> Use a single `@Parcelize` data class capturing all form state, stored in `SavedStateHandle`. The ViewModel updates the single Parcelable on every field change. When restored, the ViewModel gets the complete form state back and emits it to the UI immediately.

**Scenario 2:** You have a list of 500 items cached in the ViewModel. Process death occurs. How do you restore without hitting Bundle size limits?

> Don't store the 500 items in `SavedStateHandle`. Instead: (1) Store the items in Room (persists permanently), (2) Store only the current filter/sort/scroll position in `SavedStateHandle`, (3) After process death, the ViewModel recreates from Room + `SavedStateHandle` — Room provides the 500 items, `SavedStateHandle` provides the current UI position. The user sees the same state without Bundle size issues.

---

## Code Examples (Production-Quality Kotlin)

### Combine SavedStateHandle with Room for Robust Restoration

```kotlin
@HiltViewModel
class ArticleListViewModel @Inject constructor(
    private val savedStateHandle: SavedStateHandle,
    private val articleRepository: ArticleRepository
) : ViewModel() {

    // Surviving state: scroll position and filter
    val selectedTag: StateFlow<String> =
        savedStateHandle.getStateFlow("selected_tag", "All")

    val scrollPosition: Int
        get() = savedStateHandle["scroll_position"] ?: 0

    // Article data comes from Room (not SavedStateHandle!)
    val articles: StateFlow<List<Article>> = selectedTag
        .flatMapLatest { tag ->
            if (tag == "All") articleRepository.getAllArticles()
            else articleRepository.getArticlesByTag(tag)
        }
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), emptyList())

    fun onTagSelected(tag: String) {
        savedStateHandle["selected_tag"] = tag
    }

    fun onScrollPositionChanged(position: Int) {
        savedStateHandle["scroll_position"] = position
    }
}
```

### Testing SavedStateHandle

```kotlin
class ArticleListViewModelTest {

    @get:Rule val coroutineRule = MainCoroutineRule()

    private lateinit var savedStateHandle: SavedStateHandle
    private lateinit var viewModel: ArticleListViewModel

    @Before
    fun setup() {
        savedStateHandle = SavedStateHandle(mapOf("selected_tag" to "Kotlin"))
        viewModel = ArticleListViewModel(
            savedStateHandle = savedStateHandle,
            articleRepository = FakeArticleRepository()
        )
    }

    @Test
    fun `initial state uses SavedStateHandle value after process death`() = runTest {
        val tag = viewModel.selectedTag.value
        assertEquals("Kotlin", tag) // Restored from SavedStateHandle ✅
    }

    @Test
    fun `selecting tag updates SavedStateHandle`() = runTest {
        viewModel.onTagSelected("Android")
        assertEquals("Android", savedStateHandle.get<String>("selected_tag"))
    }
}
```

---

## Best Practices

1. **Only store UI-critical, small, Parcelable state** — scroll position, selected tab, search query, form fields
2. **Use `@Parcelize`** for custom state objects — reduces boilerplate
3. **Use `getStateFlow()`** for reactive reads — don't manually create StateFlow from SavedStateHandle
4. **Combine SavedStateHandle + Room** — SavedStateHandle for UI state, Room for actual data
5. **Define constant keys** — avoid string typos with `companion object { const val KEY_X = "x" }`
6. **Never store sensitive data** — Bundle content is not encrypted; use EncryptedSharedPreferences
7. **Test with pre-populated SavedStateHandle** — `SavedStateHandle(mapOf("key" to value))`

---

## Legacy vs Modern

| Aspect | Legacy | Modern |
|--------|--------|--------|
| Process death survival | Manual `onSaveInstanceState()` + `Bundle` | `SavedStateHandle` in ViewModel |
| Reading state | `bundle.getString("key")` in `onCreate()` | `savedStateHandle.getStateFlow("key", default)` |
| Writing state | Override `onSaveInstanceState()` | `savedStateHandle["key"] = value` |
| Reactive reads | Poll in `onCreate()` | `StateFlow` via `getStateFlow()` |
| DI for SavedStateHandle | Manual factory | Hilt auto-injects into `@HiltViewModel` |

---

## Revision Notes

- ViewModel: config changes ✅, process death ❌
- SavedStateHandle: config changes ✅, process death ✅, user Back ❌
- Internally uses `Bundle` via `SavedStateRegistry`
- Only Parcelable/Serializable/primitive types
- `getStateFlow(key, default)` for reactive access
- Store small UI state only; large data → Room
- Hilt auto-provides SavedStateHandle to `@HiltViewModel`

---

## Key Takeaways

> 💀 Process death is silent and unpredictable — assume it can happen at any time the app is backgrounded and design accordingly.

> 🔑 SavedStateHandle = `onSaveInstanceState()` Bundle + ViewModel convenience — same mechanism, better API.

> 📐 Rule of thumb: If the user typed it, selected it, or scrolled to it → SavedStateHandle. If the app fetched it → Room.

> 🧪 Always test process death scenarios: enable "Don't keep activities" in Developer Options and use "Background process limit: No background processes" to simulate.

---
---

# Chapter 15 — Navigation Component

---

## Concept

The **Navigation Component** is a Jetpack framework for implementing in-app navigation. It provides a declarative, graph-based approach to navigation that replaces manual `FragmentTransaction` management.

**Three core pieces:**

| Component | What it is | Where |
|-----------|-----------|-------|
| **NavHostFragment** | Container that swaps destination fragments | Layout XML |
| **NavController** | Navigation API — navigate, back, args | Code |
| **Navigation Graph** | XML declaring all destinations and actions | `res/navigation/` |

```xml
<!-- Navigation Graph: nav_graph.xml -->
<navigation xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_graph"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment">
        <action
            android:id="@+id/action_home_to_detail"
            app:destination="@id/detailFragment" />
    </fragment>

    <fragment
        android:id="@+id/detailFragment"
        android:name="com.example.DetailFragment">
        <argument
            android:name="productId"
            app:argType="integer" />
    </fragment>
</navigation>
```

```xml
<!-- Activity Layout: activity_main.xml -->
<androidx.fragment.app.FragmentContainerView
    android:id="@+id/nav_host_fragment"
    android:name="androidx.navigation.fragment.NavHostFragment"
    app:navGraph="@navigation/nav_graph"
    app:defaultNavHost="true"  <!-- Intercepts system back button -->
    android:layout_width="match_parent"
    android:layout_height="match_parent" />
```

---

## Why It Exists

Before Navigation Component, developers wrote this kind of code for every navigation:

```kotlin
// OLD WAY — Fragile and inconsistent
supportFragmentManager.beginTransaction()
    .replace(R.id.container, DetailFragment.newInstance(productId))
    .addToBackStack("detail")
    .setCustomAnimations(R.anim.slide_in, R.anim.slide_out)
    .commit()

// Problems:
// 1. No type-safe argument passing (easy to get key strings wrong)
// 2. Manual back stack management — easy to create inconsistent states
// 3. Deep links required manual Intent parsing in onCreate()
// 4. No visual tool to see navigation structure
// 5. Fragment transactions committed at wrong time → IllegalStateException
// 6. No automatic BottomNavigation integration with back stacks
```

Navigation Component solves all of this with:
1. **Type-safe arguments** via Safe Args Gradle plugin
2. **Declarative back stack** via `popUpTo` + `popUpToInclusive`
3. **Deep link handling** built-in
4. **Visual editor** in Android Studio
5. **Multiple back stack support** (Navigation 2.4+)
6. **Compose integration** via `NavHost`

---

## Internal Working

### NavController Internals

`NavController` maintains an internal back stack of `NavBackStackEntry` objects. Each entry has:
- The destination it represents
- A `Bundle` of arguments
- Its own `ViewModelStore` (for ViewModel scoped to that entry)
- Its own `SavedStateHandle`
- Its own `Lifecycle`

```
NavBackStack (internal)
│
├── NavBackStackEntry(homeFragment, args={}, viewModelStore)
│
├── NavBackStackEntry(listFragment, args={category="tech"}, viewModelStore)
│
└── NavBackStackEntry(detailFragment, args={id=42}, viewModelStore) ← TOP
```

### Safe Args Code Generation

Safe Args is a Gradle plugin that generates:
- **`XxxFragmentDirections`** — extension functions for each action from Xxx
- **`XxxFragmentArgs`** — type-safe argument accessor for each destination

```
nav_graph.xml
     │
     ▼ Gradle kapt/KSP
HomeFragmentDirections.kt       ← generated
DetailFragmentArgs.kt           ← generated
```

### NavHostFragment and NavController Attachment

```
Activity.setContentView()
     │
     ▼
NavHostFragment.onViewCreated()
     │
     ├── Creates NavController
     ├── Inflates NavGraph
     ├── Creates initial NavBackStackEntry for startDestination
     └── Navigates to startDestination
           │
           ▼
     FragmentTransaction: add(startDestinationFragment)
```

### Navigate() Call Flow

```
navController.navigate(R.id.action_home_to_detail)
     │
     ▼
NavController.navigate(resId, args, navOptions, navigatorExtras)
     │
     ▼
NavGraph.findNode(destinationId) → resolves destination
     │
     ▼
Navigator.navigate() (FragmentNavigator for fragments)
     │
     ▼
FragmentTransaction.replace(navHostContainer, DestinationFragment)
FragmentTransaction.addToBackStack(destinationId)
     │
     ▼
New NavBackStackEntry pushed onto NavController's stack
```

---

## Lifecycle / Flow (ASCII Diagrams)

### Navigation Back Stack State Machine

```
Screen: Home
NavBackStack: [Home]

User navigates to List:
NavBackStack: [Home, List]

User navigates to Detail(id=42):
NavBackStack: [Home, List, Detail(42)]

User presses Back:
Detail popped → Detail.onDestroy() called
NavBackStack: [Home, List]
Detail.NavBackStackEntry ViewModel cleared

popUpTo="home" inclusive=false:
Navigate to Settings → pops until Home (exclusive):
NavBackStack: [Home, Settings]

popUpTo="home" inclusive=true:
Navigate to Login → pops Home too:
NavBackStack: [Login]
```

### BottomNavigation Multiple Back Stacks (Navigation 2.4+)

```
Tab 1 (Home):     [Home]
Tab 2 (Search):   [Search]
Tab 3 (Profile):  [Profile]

                     ↕ Tab switching
User on Tab1:     [Home] ← active
User taps Tab2:   [Home (saved), Search] ← active
User goes deeper: [Home (saved), Search, SearchDetail]
User taps Tab1:   [Home] ← restored, Search back stack saved
User presses Back:[Home] — navigateUp()
```

### Deep Link Flow

```
External Intent (deepLink: myapp://product/42)
     │
     ▼
Activity.onCreate(intent)
     │
     ▼
NavController.handleDeepLink(intent)
     │
     ▼
NavGraph scanned for matching <deepLink> URI pattern
     │
     ▼
Back stack built from root to deep link destination:
[Home, ProductList, ProductDetail(42)]
     │
     ▼
User sees ProductDetail with correct back stack ✅
```

---

## Real World Example

### Complete Navigation Setup with Safe Args

**Step 1: Gradle setup**

```kotlin
// build.gradle.kts (app)
plugins {
    id("androidx.navigation.safeargs.kotlin")
}

dependencies {
    val nav_version = "2.7.7"
    implementation("androidx.navigation:navigation-fragment-ktx:$nav_version")
    implementation("androidx.navigation:navigation-ui-ktx:$nav_version")
}
```

**Step 2: Navigation graph**

```xml
<!-- res/navigation/nav_main.xml -->
<navigation
    xmlns:android="http://schemas.android.com/apk/res/android"
    xmlns:app="http://schemas.android.com/apk/res-auto"
    android:id="@+id/nav_main"
    app:startDestination="@id/homeFragment">

    <fragment
        android:id="@+id/homeFragment"
        android:name="com.example.HomeFragment"
        android:label="Home">
        <action
            android:id="@+id/action_home_to_productList"
            app:destination="@id/productListFragment" />
    </fragment>

    <fragment
        android:id="@+id/productListFragment"
        android:name="com.example.ProductListFragment"
        android:label="Products">
        <argument
            android:name="categoryId"
            app:argType="integer"
            android:defaultValue="-1" />
        <action
            android:id="@+id/action_list_to_detail"
            app:destination="@id/productDetailFragment"
            app:enterAnim="@anim/slide_in_right"
            app:exitAnim="@anim/slide_out_left"
            app:popEnterAnim="@anim/slide_in_left"
            app:popExitAnim="@anim/slide_out_right" />
    </fragment>

    <fragment
        android:id="@+id/productDetailFragment"
        android:name="com.example.ProductDetailFragment"
        android:label="{productName}">
        <argument
            android:name="productId"
            app:argType="integer" />
        <argument
            android:name="productName"
            app:argType="string" />
        <deepLink app:uri="myapp://product/{productId}" />
    </fragment>

    <dialog
        android:id="@+id/confirmationDialog"
        android:name="com.example.ConfirmationDialogFragment">
        <argument
            android:name="message"
            app:argType="string" />
    </dialog>
</navigation>
```

**Step 3: Fragments using Safe Args**

```kotlin
// HomeFragment.kt
class HomeFragment : Fragment(R.layout.fragment_home) {

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        binding.btnViewProducts.setOnClickListener {
            // Type-safe navigation with Safe Args
            val action = HomeFragmentDirections.actionHomeToProductList(categoryId = 1)
            findNavController().navigate(action)
        }
    }
}

// ProductListFragment.kt
class ProductListFragment : Fragment(R.layout.fragment_product_list) {

    // Type-safe argument access
    private val args: ProductListFragmentArgs by navArgs()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val categoryId = args.categoryId // Type-safe Int, not getString()!

        adapter.setOnItemClickListener { product ->
            val action = ProductListFragmentDirections.actionListToDetail(
                productId = product.id,
                productName = product.name
            )
            findNavController().navigate(action)
        }
    }
}

// ProductDetailFragment.kt
class ProductDetailFragment : Fragment(R.layout.fragment_product_detail) {

    private val args: ProductDetailFragmentArgs by navArgs()

    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        val productId = args.productId     // Int ✅
        val productName = args.productName // String ✅
        // No bundle.getInt() / bundle.getString() — compile-time safe!
    }
}
```

**Step 4: Activity with NavController**

```kotlin
@AndroidEntryPoint
class MainActivity : AppCompatActivity(R.layout.activity_main) {

    private lateinit var navController: NavController

    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)

        val navHostFragment = supportFragmentManager
            .findFragmentById(R.id.nav_host_fragment) as NavHostFragment
        navController = navHostFragment.navController

        // Connect ActionBar with NavController (shows title, Up button)
        setupActionBarWithNavController(navController)

        // Connect BottomNavigationView with multiple back stacks
        binding.bottomNav.setupWithNavController(navController)
    }

    override fun onSupportNavigateUp(): Boolean {
        return navController.navigateUp() || super.onSupportNavigateUp()
    }
}
```

### ViewModel Scoped to NavBackStackEntry

```kotlin
// Sharing ViewModel between two fragments in same flow
// (more precise than activityViewModels())

class CheckoutStep1Fragment : Fragment() {
    private val checkoutViewModel: CheckoutViewModel by navGraphViewModels(R.id.checkout_graph) {
        defaultViewModelProviderFactory
    }
    // Or with Hilt:
    private val checkoutViewModel: CheckoutViewModel by hiltNavGraphViewModels(R.id.checkout_graph)
}

class CheckoutStep2Fragment : Fragment() {
    private val checkoutViewModel: CheckoutViewModel by hiltNavGraphViewModels(R.id.checkout_graph)
    // Same instance as Step1! Cleared when checkout_graph is popped.
}
```

### Result Passing Between Destinations

```kotlin
// Modern approach: use NavBackStackEntry's SavedStateHandle
// In DetailFragment (setting result before popping)
class DetailFragment : Fragment() {
    fun onConfirmed() {
        // Pass result back to previous destination
        findNavController()
            .previousBackStackEntry
            ?.savedStateHandle
            ?.set("result_key", selectedItem)
        findNavController().popBackStack()
    }
}

// In ListFragment (observing result)
class ListFragment : Fragment() {
    override fun onViewCreated(view: View, savedInstanceState: Bundle?) {
        // Observe result from detail
        val navBackStackEntry = findNavController().getBackStackEntry(R.id.listFragment)
        navBackStackEntry.savedStateHandle
            .getLiveData<Item>("result_key")
            .observe(viewLifecycleOwner) { result ->
                // Handle result from DetailFragment
                viewModel.onItemSelected(result)
            }
    }
}
```

---

## Common Mistakes

| Mistake | Problem | Fix |
|---------|---------|-----|
| `app:defaultNavHost="false"` | System Back button doesn't work with NavController | Always `true` for main NavHost |
| Manual `FragmentTransaction` inside NavGraph destinations | Back stack inconsistency | Use NavController exclusively |
| `findNavController()` in `onCreateView()` | NavController may not be attached yet | Call in `onViewCreated()` |
| Navigating from Background thread | `NavController` not thread-safe | Always navigate on main thread |
| Not using `popUpTo` for login flow | Login remains in back stack after login success | `popUpTo="@id/loginFragment" inclusive=true` |
| Navigating after `onSaveInstanceState()` | `IllegalStateException: Can not perform this action after onSaveInstanceState` | Check `isResumed` or use `commitAllowingStateLoss` pattern; better: use `LifecycleScope` |
| Sharing data via fragment arguments for large objects | Binder size limit; arguments are in Bundle | Use SharedViewModel or Room |

---

## Memory Leak / Performance Concerns

### 1. ViewBinding + Fragment Lifecycle

```kotlin
// ❌ LEAK: ViewBinding held past onDestroyView
class BadFragment : Fragment() {
    private lateinit var binding: FragmentBadBinding

    override fun onCreateView(...): View {
        binding = FragmentBadBinding.inflate(inflater)
        return binding.root
    }
    // binding holds reference to destroyed view! Fragment lives on back stack.
}

// ✅ CORRECT: Null binding in onDestroyView
class GoodFragment : Fragment() {
    private var _binding: FragmentGoodBinding? = null
    private val binding get() = _binding!!

    override fun onCreateView(...): View {
        _binding = FragmentGoodBinding.inflate(inflater, container, false)
        return binding.root
    }

    override fun onDestroyView() {
        super.onDestroyView()
        _binding = null // Fragment lives on back stack; view is destroyed
    }
}
```

### 2. NavBackStackEntry ViewModel Lifecycle

When you navigate to a destination, a `NavBackStackEntry` is created. If you scope a ViewModel to it, the ViewModel is cleared when the entry is popped from the stack. This is more memory-efficient than `activityViewModels()` because it's automatically cleaned up when the navigation flow ends.

### 3. Multiple Back Stacks and Memory

With `setupWithNavController()` for BottomNavigation, all tabs maintain their own back stacks in memory. For memory-intensive tabs, consider clearing the back stack when the app goes to background.

---

## Interview Questions

### Beginner

**Q1: What are the three components of Navigation Component and what does each do?**

> - **NavHostFragment**: A special fragment in your Activity layout that acts as a container/host. It swaps in and out the destination fragments as the user navigates.
> - **NavController**: The programmatic API you call in code. `navigate()` to go forward, `popBackStack()` to go back, `navigateUp()` for Up navigation.
> - **Navigation Graph**: An XML resource in `res/navigation/` that declares all destinations (fragments, activities, dialogs) and the actions (connections) between them, including animations and arguments.

**Q2: What is Safe Args and why should you use it?**

> Safe Args is a Gradle plugin that generates type-safe Kotlin classes from your Navigation Graph XML. Instead of passing arguments as raw strings with `bundle.putInt("id", id)`, you use generated `Directions` classes: `HomeFragmentDirections.actionHomeToDetail(id = 42)`. Benefits: compile-time type checking (wrong type → build error, not runtime crash), no string literal keys to mistype, IDE autocomplete, refactor-safe.

**Q3: What is `popUpTo` and `popUpToInclusive` in Navigation?**

> `popUpTo` specifies a destination to pop back to when navigating. All destinations above it in the back stack are removed. `popUpToInclusive = true` also removes the specified destination itself.
>
> Use case: Login flow — after successful login, navigate to Home with `popUpTo = loginFragment, inclusive = true`. This removes Login from the back stack, so pressing Back from Home exits the app rather than returning to Login.

---

### Intermediate

**Q4: What is the difference between `navigateUp()` and `popBackStack()`?**

> `popBackStack()` is purely **chronological** — it pops the most recent destination from the back stack, regardless of the navigation hierarchy. It mirrors the system Back button.
>
> `navigateUp()` is **hierarchical** — it navigates to the "parent" destination as defined by the Up button (ActionBar back arrow). In practice within a single Activity, they often behave the same. The difference matters with deep links: if a user deep-links directly to `ProductDetail`, `navigateUp()` constructs the logical parent path (Home → ProductList → ProductDetail, then up to ProductList), while `popBackStack()` would exit the app (no previous entry).

**Q5: How do multiple back stacks work with `BottomNavigationView` in Navigation 2.4+?**

> Before Navigation 2.4, switching tabs always cleared the current tab's back stack and returned to the tab's root. Navigation 2.4 introduced multiple back stack support. Each tab maintains its own independent `NavBackStack`. When you switch tabs, the current tab's stack is saved and the new tab's stack is restored. `setupWithNavController()` handles this automatically. Each tab's fragment instances and their state are preserved. This matches Material Design guidelines where tabs are peers, not a hierarchy.

---

### Advanced

**Q6: How does Navigation Component handle deep links, and how does it build the correct back stack?**

> When an external deep link arrives via `Intent`, `NavController.handleDeepLink(intent)` is called. It searches the `NavGraph` for a matching `<deepLink>` URI pattern. Once found, it doesn't just navigate directly to that destination — it **synthetically constructs the full back stack** by traversing from the graph's `startDestination` to the deep link destination through the defined graph hierarchy. This means pressing Back from a deep-linked screen gives the user the logical parent screen, not the previous app they came from.
>
> For explicit deep links (in-app PendingIntent), `NavDeepLinkBuilder` constructs the TaskStackBuilder and NavController back stack programmatically.

**Q7: How would you implement a flow where Fragment B needs to return a result to Fragment A using Navigation Component?**

> Use the `SavedStateHandle` of the `NavBackStackEntry`:
> ```kotlin
> // Fragment B (returning result):
> findNavController().previousBackStackEntry?.savedStateHandle?.set("result", myResult)
> findNavController().popBackStack()
>
> // Fragment A (observing result):
> findNavController().currentBackStackEntry?.savedStateHandle
>     ?.getLiveData<MyResult>("result")
>     ?.observe(viewLifecycleOwner) { result -> handleResult(result) }
> ```
> This is the Navigation Component's official pattern — avoids Fragment callbacks or shared ViewModel just for result passing.

---

## Scenario Questions

**Scenario 1:** Your app has a login flow (Login → Register → ForgotPassword). After successful login, you want the user to go to Home and never be able to go back to the auth screens. How do you configure the navigation?

```xml
<action
    android:id="@+id/action_login_to_home"
    app:destination="@id/homeFragment"
    app:popUpTo="@id/nav_graph"
    app:popUpToInclusive="true" />
<!-- Pops everything including the start destination, leaving only Home -->
```

**Scenario 2:** You have 3 tabs (Home, Search, Profile) with BottomNavigation. The user goes Home → Article → Comments. Switches to Search → Results → Detail. Switches back to Home. What should they see?

> With Navigation 2.4+ and `setupWithNavController()`, the Home tab restores its back stack: [Home → Article → Comments]. The user is exactly where they left off. The Search tab's stack [Search → Results → Detail] is saved in memory, ready to be restored when the user switches back.

---

## Code Examples (Production-Quality Kotlin)

### Compose Navigation (Modern)

```kotlin
// build.gradle.kts
implementation("androidx.navigation:navigation-compose:2.7.7")

// Navigation setup in Composable
@Composable
fun AppNavigation() {
    val navController = rememberNavController()

    NavHost(
        navController = navController,
        startDestination = "home"
    ) {
        composable("home") {
            HomeScreen(
                onNavigateToProducts = { categoryId ->
                    navController.navigate("products/$categoryId")
                }
            )
        }

        composable(
            route = "products/{categoryId}",
            arguments = listOf(navArgument("categoryId") { type = NavType.IntType })
        ) { backStackEntry ->
            val categoryId = backStackEntry.arguments?.getInt("categoryId") ?: -1
            ProductListScreen(
                categoryId = categoryId,
                onProductClicked = { product ->
                    navController.navigate("product/${product.id}/${product.name}")
                }
            )
        }

        composable(
            route = "product/{productId}/{productName}",
            arguments = listOf(
                navArgument("productId") { type = NavType.IntType },
                navArgument("productName") { type = NavType.StringType }
            ),
            deepLinks = listOf(navDeepLink { uriPattern = "myapp://product/{productId}" })
        ) { backStackEntry ->
            ProductDetailScreen(
                productId = backStackEntry.arguments!!.getInt("productId"),
                productName = backStackEntry.arguments!!.getString("productName")!!,
                onNavigateBack = { navController.popBackStack() }
            )
        }
    }
}
```

### Type-Safe Navigation with Kotlin Serialization (Navigation 2.8+)

```kotlin
// Modern type-safe routes using @Serializable
@Serializable
data object HomeRoute

@Serializable
data class ProductDetailRoute(val productId: Int, val productName: String)

@Composable
fun TypeSafeNavigation() {
    val navController = rememberNavController()

    NavHost(navController, startDestination = HomeRoute) {
        composable<HomeRoute> {
            HomeScreen(
                onProductClicked = { id, name ->
                    navController.navigate(ProductDetailRoute(id, name))
                }
            )
        }

        composable<ProductDetailRoute> { backStackEntry ->
            val route: ProductDetailRoute = backStackEntry.toRoute()
            ProductDetailScreen(productId = route.productId, productName = route.productName)
        }
    }
}
```

---

## Best Practices

1. **Never mix manual `FragmentTransaction` with NavController** — pick one and stick to it
2. **Use Safe Args** — never pass arguments as raw strings
3. **Use `popUpTo` for auth flows** — ensure Login is never on the back stack after login success
4. **Scope ViewModels to NavBackStackEntry** for flow-specific state — not `activityViewModels()`
5. **Use `NavBackStackEntry.savedStateHandle`** for Fragment-to-Fragment result passing
6. **`app:defaultNavHost="true"`** — always, for correct Back button behavior
7. **Use `setupWithNavController()`** for BottomNavigation — automatic multiple back stack support
8. **Handle deep links in NavGraph** — not manually in `onCreate()`

---

## Legacy vs Modern

| Aspect | Legacy | Modern |
|--------|--------|--------|
| Navigation | Manual `FragmentTransaction` | `NavController.navigate()` |
| Type-safe args | `bundle.putInt("id", id)` | Safe Args + `navArgs()` delegate |
| Back stack | Manual `addToBackStack("tag")` | Declarative `popUpTo` in graph |
| Deep links | Manual `Intent` parsing in `onCreate()` | `<deepLink>` in nav graph |
| BottomNav tabs | Single back stack (tabs reset) | Multiple back stacks (Navigation 2.4+) |
| Fragment result | Interface callbacks | `NavBackStackEntry.savedStateHandle` |
| Compose | N/A | `NavHost` + `composable()` |
| Type-safe routes | N/A | `@Serializable` routes (Nav 2.8+) |

---

## Revision Notes

- NavHostFragment = container; NavController = API; NavGraph = map
- Safe Args generates `XxxDirections` (navigate) and `XxxArgs` (receive) classes
- `popUpTo` + `inclusive` control back stack cleanup on navigation
- `navigateUp()` = hierarchical (Up button); `popBackStack()` = chronological (Back button)
- Deep links build synthetic back stacks from graph root to destination
- Navigation 2.4+: `setupWithNavController()` enables multiple back stacks for BottomNav
- NavBackStackEntry has its own `ViewModelStore`, `SavedStateHandle`, and `Lifecycle`
- Result passing: set on `previousBackStackEntry.savedStateHandle`, observe on `currentBackStackEntry.savedStateHandle`
- `findNavController()` safe only after `onViewCreated()`

---

## Key Takeaways

> 🗺️ Navigation Component makes the navigation graph explicit and visual — you can see your entire app's flow at a glance in the Navigation Editor.

> 🔒 Safe Args converts runtime crashes (wrong Bundle key type) into compile-time errors — always use it.

> 📚 `popUpTo` + `popUpToInclusive` are essential for clean back stacks in authentication flows, one-way flows (onboarding), and tab-based navigation.

> 🔄 Navigation 2.4's multiple back stacks fundamentally changed how BottomNavigation works — tabs now maintain state independently, matching user expectations.

> 🧩 NavBackStackEntry-scoped ViewModels are the right tool for multi-step flows — they're automatically cleared when the flow completes.

---
---

# Chapter 16 — Data Persistence

---

## Concept

Android offers multiple data persistence mechanisms, each designed for different use cases:

| Mechanism | Use Case | Data Size | Type Safety | Async | Status |
|-----------|----------|-----------|-------------|-------|--------|
| SharedPreferences | Simple key-value pairs | Small | ❌ | Partial | ⚠️ Legacy |
| DataStore | Key-value or typed proto storage | Small-Medium | ✅ | ✅ Flow | ✅ Modern |
| Internal Files | App-private file storage | Any | N/A | Manual | ✅ |
| External Files | Shared media/documents | Any | N/A | Manual | ✅ |
| SQLite (direct) | Relational data | Large | ❌ | ❌ | ⚠️ Legacy |
| Room | Relational ORM over SQLite | Large | ✅ | ✅ Flow | ✅ Modern |

---

## Why It Exists

Different data has different persistence requirements:

```
User preferences (dark mode, language) ──────► DataStore/SharedPreferences
Authentication token ─────────────────────────► EncryptedSharedPreferences
Cached API data ──────────────────────────────► Room Database
Downloaded files ─────────────────────────────► Internal Storage
User-shared documents ────────────────────────► External/SAF
User photos/videos ───────────────────────────► MediaStore
Complex relational data ──────────────────────► Room Database
```

No single persistence mechanism fits all scenarios. Understanding when to use each is critical for building correct, efficient Android apps.

---

---

## Part A — SharedPreferences (Legacy)

---

### Concept

`SharedPreferences` is an XML-based key-value store for simple primitive data. It has been the standard for small settings storage since Android 1.0.

```kotlin
// Getting an instance
val prefs = context.getSharedPreferences("my_prefs", Context.MODE_PRIVATE)

// Reading
val isDarkMode = prefs.getBoolean("dark_mode", false)
val username = prefs.getString("username", null)

// Writing
prefs.edit() {                    // KTX extension
    putBoolean("dark_mode", true)
    putString("username", "alice")
}
// or
prefs.edit()
    .putBoolean("dark_mode", true)
    .apply()  // async (preferred) — vs commit() which is synchronous
```

### Internal Working

SharedPreferences stores data in an XML file at:
`/data/data/<package_name>/shared_prefs/<name>.xml`

```xml
<?xml version='1.0' encoding='utf-8' standalone='yes' ?>
<map>
    <boolean name="dark_mode" value="true" />
    <string name="username">alice</string>
    <int name="launch_count" value="5" />
</map>
```

On first access, the entire XML file is loaded into memory and kept in a `Map<String, Any?>` cache. `apply()` writes to the in-memory map immediately and schedules a background disk write. `commit()` blocks the calling thread until the disk write completes.

### Critical Problems with SharedPreferences

```
1. NOT type-safe: getString() on an Int key returns null silently
2. NOT coroutine-safe: apply() uses its own thread, not integrated with coroutines
3. No Flow support: no reactive observation of changes
4. No encryption: data stored in plain-text XML
5. No migration support: manual key renaming
6. Race conditions: concurrent reads/writes can cause subtle bugs
7. Loads entire file on first access: slow for large files
8. MODE_WORLD_READABLE/WORLD_WRITEABLE: removed in API 17 (security issue)
```

### When to Still Use SharedPreferences

- Maintaining a legacy codebase that already uses it
- Very simple, non-critical data in small apps
- Only if DataStore cannot be introduced (e.g., dependency constraints)

> ⚠️ **DEPRECATED EQUIVALENT**: `PreferenceManager.getDefaultSharedPreferences()` still works but DataStore is the official replacement.

---

## Part B — DataStore (Modern)

---

### Concept

Jetpack DataStore is the official replacement for SharedPreferences. It comes in two flavors:

| Flavor | Storage | Type Safety | Use Case |
|--------|---------|-------------|----------|
| **Preferences DataStore** | Key-value (like SharedPreferences) | Type-safe keys | Simple settings |
| **Proto DataStore** | Protocol Buffers schema | Fully typed | Complex settings |

Both use Kotlin coroutines and Flow for async, reactive data access.

### Preferences DataStore

```kotlin
// 1. Create DataStore (at file scope, never in a class body — singleton)
val Context.dataStore: DataStore<Preferences> by preferencesDataStore(name = "settings")

// 2. Define type-safe keys
object SettingsKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val USERNAME = stringPreferencesKey("username")
    val FONT_SIZE = intPreferencesKey("font_size")
    val LAUNCH_COUNT = intPreferencesKey("launch_count")
}

// 3. Read (returns Flow — always reactive)
val darkModeFlow: Flow<Boolean> = context.dataStore.data
    .catch { exception ->
        if (exception is IOException) emit(emptyPreferences())
        else throw exception
    }
    .map { preferences ->
        preferences[SettingsKeys.DARK_MODE] ?: false
    }

// 4. Write (suspend function — call from coroutine)
suspend fun setDarkMode(enabled: Boolean) {
    context.dataStore.edit { settings ->
        settings[SettingsKeys.DARK_MODE] = enabled
    }
}
```

### Proto DataStore

```protobuf
// settings.proto
syntax = "proto3";

option java_package = "com.example.datastore";
option java_multiple_files = true;

message UserSettings {
  bool dark_mode = 1;
  string language = 2;
  int32 font_size = 3;
}
```

```kotlin
// Serializer
object UserSettingsSerializer : Serializer<UserSettings> {
    override val defaultValue: UserSettings = UserSettings.getDefaultInstance()

    override suspend fun readFrom(input: InputStream): UserSettings {
        try {
            return UserSettings.parseFrom(input)
        } catch (exception: InvalidProtocolBufferException) {
            throw CorruptionException("Cannot read proto.", exception)
        }
    }

    override suspend fun writeTo(t: UserSettings, output: OutputStream) {
        t.writeTo(output)
    }
}

// Create DataStore
val Context.userSettingsDataStore: DataStore<UserSettings> by dataStore(
    fileName = "user_settings.pb",
    serializer = UserSettingsSerializer
)

// Read
val userSettingsFlow: Flow<UserSettings> = context.userSettingsDataStore.data

// Write
suspend fun setDarkMode(enabled: Boolean) {
    context.userSettingsDataStore.updateData { currentSettings ->
        currentSettings.toBuilder()
            .setDarkMode(enabled)
            .build()
    }
}
```

### DataStore in ViewModel with Repository

```kotlin
// Repository
class SettingsRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    val darkMode: Flow<Boolean> = context.dataStore.data
        .catch { if (it is IOException) emit(emptyPreferences()) else throw it }
        .map { it[SettingsKeys.DARK_MODE] ?: false }

    val fontSize: Flow<Int> = context.dataStore.data
        .catch { if (it is IOException) emit(emptyPreferences()) else throw it }
        .map { it[SettingsKeys.FONT_SIZE] ?: 14 }

    suspend fun setDarkMode(enabled: Boolean) {
        context.dataStore.edit { it[SettingsKeys.DARK_MODE] = enabled }
    }

    suspend fun setFontSize(size: Int) {
        context.dataStore.edit { it[SettingsKeys.FONT_SIZE] = size }
    }
}

// ViewModel
@HiltViewModel
class SettingsViewModel @Inject constructor(
    private val settingsRepository: SettingsRepository
) : ViewModel() {

    val darkMode: StateFlow<Boolean> = settingsRepository.darkMode
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), false)

    val fontSize: StateFlow<Int> = settingsRepository.fontSize
        .stateIn(viewModelScope, SharingStarted.WhileSubscribed(5000), 14)

    fun onDarkModeToggled(enabled: Boolean) {
        viewModelScope.launch { settingsRepository.setDarkMode(enabled) }
    }

    fun onFontSizeChanged(size: Int) {
        viewModelScope.launch { settingsRepository.setFontSize(size) }
    }
}
```

---

## Part C — File Storage

---

### Internal Storage

App-private files. Auto-deleted when the app is uninstalled. No permissions needed.

```kotlin
// Writing to internal storage
context.openFileOutput("notes.txt", Context.MODE_PRIVATE).use { fos ->
    fos.write("Hello, internal storage!".toByteArray())
}

// Reading from internal storage
val content = context.openFileInput("notes.txt").bufferedReader().use { it.readText() }

// Using filesDir for direct File access
val file = File(context.filesDir, "data/config.json")
file.parentFile?.mkdirs()
file.writeText(jsonString)

// Cache files (deleted when device needs space)
val cacheFile = File(context.cacheDir, "temp.bin")
```

**Directory structure:**

```
/data/data/<package>/
├── files/          ← context.filesDir
├── cache/          ← context.cacheDir
├── databases/      ← Room/SQLite files
└── shared_prefs/   ← SharedPreferences XML
```

### External Storage + Scoped Storage (API 29+)

```kotlin
// Scoped Storage (API 29+): No permission needed for app-specific external files
val externalFile = File(context.getExternalFilesDir(Environment.DIRECTORY_DOCUMENTS), "report.pdf")

// MediaStore for shared media (API 29+)
val contentValues = ContentValues().apply {
    put(MediaStore.Images.Media.DISPLAY_NAME, "photo.jpg")
    put(MediaStore.Images.Media.MIME_TYPE, "image/jpeg")
    put(MediaStore.Images.Media.RELATIVE_PATH, Environment.DIRECTORY_PICTURES)
}
val imageUri = contentResolver.insert(MediaStore.Images.Media.EXTERNAL_CONTENT_URI, contentValues)
contentResolver.openOutputStream(imageUri!!)?.use { outputStream ->
    bitmap.compress(Bitmap.CompressFormat.JPEG, 90, outputStream)
}

// Writing to external app-specific directory (no permission, deleted on uninstall)
context.getExternalFilesDir(null)?.let { dir ->
    File(dir, "exported_data.csv").writeText(csvContent)
}
```

### FileProvider (Sharing Internal Files)

```xml
<!-- AndroidManifest.xml -->
<provider
    android:name="androidx.core.content.FileProvider"
    android:authorities="${applicationId}.fileprovider"
    android:exported="false"
    android:grantUriPermissions="true">
    <meta-data
        android:name="android.support.FILE_PROVIDER_PATHS"
        android:resource="@xml/file_paths" />
</provider>
```

```xml
<!-- res/xml/file_paths.xml -->
<paths>
    <files-path name="shared_files" path="shared/" />
    <cache-path name="shared_cache" path="cache/" />
</paths>
```

```kotlin
// Sharing an internal file via FileProvider
val file = File(context.filesDir, "shared/document.pdf")
val uri = FileProvider.getUriForFile(context, "${context.packageName}.fileprovider", file)
val shareIntent = Intent(Intent.ACTION_SEND).apply {
    type = "application/pdf"
    putExtra(Intent.EXTRA_STREAM, uri)
    addFlags(Intent.FLAG_GRANT_READ_URI_PERMISSION)
}
startActivity(Intent.createChooser(shareIntent, "Share Document"))
```

---

## Part D — SQLite Direct (Legacy)

---

### Concept

`SQLiteOpenHelper` is the traditional way to manage SQLite databases in Android. It's verbose, error-prone, and lacks coroutine/Flow support.

```kotlin
// ⚠️ LEGACY — Use Room instead
class AppDatabaseHelper(context: Context) :
    SQLiteOpenHelper(context, "app.db", null, DATABASE_VERSION) {

    companion object {
        const val DATABASE_VERSION = 1
        const val TABLE_USERS = "users"
    }

    override fun onCreate(db: SQLiteDatabase) {
        db.execSQL("""
            CREATE TABLE $TABLE_USERS (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                name TEXT NOT NULL,
                email TEXT UNIQUE NOT NULL
            )
        """)
    }

    override fun onUpgrade(db: SQLiteDatabase, oldVersion: Int, newVersion: Int) {
        // ⚠️ This drops all data! Real apps use ALTER TABLE
        db.execSQL("DROP TABLE IF EXISTS $TABLE_USERS")
        onCreate(db)
    }

    fun insertUser(name: String, email: String): Long {
        val values = ContentValues().apply {
            put("name", name)
            put("email", email)
        }
        return writableDatabase.insert(TABLE_USERS, null, values)
    }

    fun getUsers(): List<Pair<String, String>> {
        val users = mutableListOf<Pair<String, String>>()
        val cursor = readableDatabase.query(TABLE_USERS, null, null, null, null, null, null)
        cursor.use {
            while (it.moveToNext()) {
                val name = it.getString(it.getColumnIndexOrThrow("name"))
                val email = it.getString(it.getColumnIndexOrThrow("email"))
                users.add(name to email)
            }
        }
        return users
    }
}
```

**Problems with direct SQLite:**
- SQL strings not verified at compile time → runtime errors
- Cursor management is manual → leak risk if not closed
- No coroutine or Flow support → must run on background thread manually
- No type conversion → convert Date/Enum manually
- No migration tooling → schema upgrades are fragile

---

## Part E — Room (Modern ORM)

---

### Concept

Room is a type-safe SQLite abstraction layer. It verifies SQL queries at **compile time**, returns reactive `Flow<T>`, and integrates seamlessly with coroutines.

**Three pillars:**

| Annotation | Role |
|------------|------|
| `@Entity` | Defines a table |
| `@Dao` | Data Access Object — query interface |
| `@Database` | Database definition and configuration |

### Internal Working

```
@Database annotation
      │
      ▼ Room annotation processor (KSP/kapt)
Generated: AppDatabase_Impl.kt
      │
      ├── Opens/creates SQLite file
      ├── Manages SQLiteOpenHelper internally
      └── Creates DAO implementations
            │
            ▼
@Dao interface → Room generates implementation
@Query → SQL verified at compile time via SQLite parser
Flow<T> return → Room sets up SQLite change notifications
```

### Entity (Table Definition)

```kotlin
@Entity(
    tableName = "products",
    indices = [Index(value = ["sku"], unique = true)]
)
data class ProductEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,

    @ColumnInfo(name = "product_name")
    val name: String,

    val price: Double,

    val categoryId: Int,

    @ColumnInfo(name = "created_at")
    val createdAt: Long = System.currentTimeMillis(),

    val imageUrl: String?,

    @ColumnInfo(name = "is_active")
    val isActive: Boolean = true,

    val sku: String
)

// Foreign key relationship
@Entity(
    tableName = "order_items",
    foreignKeys = [
        ForeignKey(
            entity = ProductEntity::class,
            parentColumns = ["id"],
            childColumns = ["product_id"],
            onDelete = ForeignKey.CASCADE  // Auto-delete items when product deleted
        )
    ],
    indices = [Index("product_id")]
)
data class OrderItemEntity(
    @PrimaryKey(autoGenerate = true)
    val id: Int = 0,
    val orderId: Int,
    val productId: Int,
    val quantity: Int,
    val unitPrice: Double
)
```

### TypeConverters

```kotlin
// Convert complex types that Room can't store natively
class Converters {

    @TypeConverter
    fun fromDate(date: Date?): Long? = date?.time

    @TypeConverter
    fun toDate(timestamp: Long?): Date? = timestamp?.let { Date(it) }

    @TypeConverter
    fun fromStringList(list: List<String>?): String? =
        list?.let { Gson().toJson(it) }

    @TypeConverter
    fun toStringList(json: String?): List<String>? =
        json?.let { Gson().fromJson<List<String>>(it, object : TypeToken<List<String>>() {}.type) }

    @TypeConverter
    fun fromStatus(status: OrderStatus): String = status.name

    @TypeConverter
    fun toStatus(value: String): OrderStatus = OrderStatus.valueOf(value)
}

enum class OrderStatus { PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED }
```

### DAO (Data Access Object)

```kotlin
@Dao
interface ProductDao {

    // Reactive query — emits new list whenever products table changes
    @Query("SELECT * FROM products WHERE is_active = 1 ORDER BY product_name ASC")
    fun getAllActiveProducts(): Flow<List<ProductEntity>>

    @Query("SELECT * FROM products WHERE categoryId = :categoryId AND is_active = 1")
    fun getProductsByCategory(categoryId: Int): Flow<List<ProductEntity>>

    @Query("SELECT * FROM products WHERE id = :id")
    suspend fun getProductById(id: Int): ProductEntity?

    @Query("SELECT * FROM products WHERE product_name LIKE '%' || :query || '%'")
    fun searchProducts(query: String): Flow<List<ProductEntity>>

    // Single suspend functions for write operations
    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertProduct(product: ProductEntity): Long

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(products: List<ProductEntity>)

    @Update
    suspend fun updateProduct(product: ProductEntity)

    @Delete
    suspend fun deleteProduct(product: ProductEntity)

    @Query("DELETE FROM products WHERE id = :id")
    suspend fun deleteProductById(id: Int)

    @Query("UPDATE products SET is_active = 0 WHERE id = :id")
    suspend fun softDeleteProduct(id: Int)

    // Transaction: multiple operations atomically
    @Transaction
    suspend fun replaceAllProducts(products: List<ProductEntity>) {
        deleteAllProducts()
        insertAll(products)
    }

    @Query("DELETE FROM products")
    suspend fun deleteAllProducts()

    // Relation query with JOIN
    @Query("""
        SELECT p.*, c.name as category_name 
        FROM products p 
        INNER JOIN categories c ON p.categoryId = c.id
        WHERE p.is_active = 1
    """)
    fun getProductsWithCategory(): Flow<List<ProductWithCategory>>
}
```

### Relation (One-to-Many)

```kotlin
// Entity with One-to-Many relationship
data class CategoryWithProducts(
    @Embedded val category: CategoryEntity,
    @Relation(
        parentColumn = "id",
        entityColumn = "categoryId"
    )
    val products: List<ProductEntity>
)

// In DAO
@Transaction
@Query("SELECT * FROM categories")
fun getCategoriesWithProducts(): Flow<List<CategoryWithProducts>>
```

### Database Definition

```kotlin
@Database(
    entities = [
        ProductEntity::class,
        CategoryEntity::class,
        OrderEntity::class,
        OrderItemEntity::class
    ],
    version = 3,                    // Current schema version
    exportSchema = true             // Exports schema JSON for migration verification
)
@TypeConverters(Converters::class)
abstract class AppDatabase : RoomDatabase() {

    abstract fun productDao(): ProductDao
    abstract fun categoryDao(): CategoryDao
    abstract fun orderDao(): OrderDao

    companion object {
        @Volatile private var INSTANCE: AppDatabase? = null

        fun getInstance(context: Context): AppDatabase {
            return INSTANCE ?: synchronized(this) {
                val instance = Room.databaseBuilder(
                    context.applicationContext,
                    AppDatabase::class.java,
                    "app_database.db"
                )
                    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
                    .addCallback(DatabaseCallback())
                    .build()
                INSTANCE = instance
                instance
            }
        }

        // Migration from version 1 to 2: added imageUrl column
        val MIGRATION_1_2 = object : Migration(1, 2) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE products ADD COLUMN imageUrl TEXT")
            }
        }

        // Migration from version 2 to 3: added sku column with unique index
        val MIGRATION_2_3 = object : Migration(2, 3) {
            override fun migrate(db: SupportSQLiteDatabase) {
                db.execSQL("ALTER TABLE products ADD COLUMN sku TEXT NOT NULL DEFAULT ''")
                db.execSQL("CREATE UNIQUE INDEX index_products_sku ON products(sku)")
            }
        }
    }
}

// Pre-populate DB callback
class DatabaseCallback : RoomDatabase.Callback() {
    override fun onCreate(db: SupportSQLiteDatabase) {
        super.onCreate(db)
        // Pre-populate with initial data if needed
    }
}
```

### Repository with Room

```kotlin
class ProductRepository @Inject constructor(
    private val productDao: ProductDao,
    private val apiService: ProductApiService
) {
    // Room as Single Source of Truth
    fun getAllProducts(): Flow<List<Product>> =
        productDao.getAllActiveProducts()
            .map { entities -> entities.map { it.toDomain() } }

    fun searchProducts(query: String): Flow<List<Product>> =
        productDao.searchProducts(query)
            .map { entities -> entities.map { it.toDomain() } }

    suspend fun refreshProducts() {
        val remoteProducts = apiService.fetchProducts()
        productDao.replaceAllProducts(remoteProducts.map { it.toEntity() })
    }

    suspend fun getProductById(id: Int): Product? =
        productDao.getProductById(id)?.toDomain()
}
```

### Room in-memory Database for Testing

```kotlin
@RunWith(AndroidJUnit4::class)
class ProductDaoTest {

    private lateinit var db: AppDatabase
    private lateinit var productDao: ProductDao

    @Before
    fun createDb() {
        val context = ApplicationProvider.getApplicationContext<Context>()
        db = Room.inMemoryDatabaseBuilder(context, AppDatabase::class.java)
            .allowMainThreadQueries() // For testing only!
            .build()
        productDao = db.productDao()
    }

    @After
    fun closeDb() = db.close()

    @Test
    fun insertAndRetrieveProduct() = runTest {
        val product = ProductEntity(
            name = "Test Product",
            price = 9.99,
            categoryId = 1,
            sku = "SKU-001"
        )

        productDao.insertProduct(product)

        productDao.getAllActiveProducts().first().let { products ->
            assertEquals(1, products.size)
            assertEquals("Test Product", products[0].name)
        }
    }

    @Test
    fun flowEmitsOnUpdate() = runTest {
        val flow = productDao.getAllActiveProducts()

        val initial = flow.first()
        assertTrue(initial.isEmpty())

        productDao.insertProduct(ProductEntity(name = "New", price = 1.0, categoryId = 1, sku = "S1"))

        val updated = flow.first()
        assertEquals(1, updated.size)
    }
}
```

---

## Lifecycle / Flow (ASCII Diagrams)

### Room Data Flow

```
User taps "Refresh"
      │
      ▼
ViewModel.refresh() → viewModelScope.launch
      │
      ▼
Repository.refreshProducts()
      │
      ├──► ApiService.fetchProducts() [IO dispatcher]
      │
      └──► ProductDao.replaceAllProducts(entities)
                │
                ▼
          SQLite write (Room transaction)
                │
                ▼
          Room's InvalidationTracker detects table change
                │
                ▼
          Flow<List<ProductEntity>> emits new data
                │
                ▼
          Repository.map { it.toDomain() }
                │
                ▼
          ViewModel StateFlow receives new list
                │
                ▼
          View renders updated RecyclerView ✅
```

### Room Migration Flow

```
App updated from version 2 to version 3
      │
      ▼
Room.databaseBuilder() called with version=3
      │
      ▼
Room checks stored SQLite user_version pragma
      │  (was 2, now needs 3)
      ▼
addMigrations() list searched for Migration(2,3)
      │
      ▼
MIGRATION_2_3.migrate(db) called
      │  (ALTER TABLE products ADD COLUMN sku...)
      ▼
SQLite user_version updated to 3
      │
      ▼
Database opened successfully ✅

If no migration found:
      │
      ▼
fallbackToDestructiveMigration() → database wiped and recreated
OR
IllegalStateException thrown → app crashes ← default behavior
```

### DataStore vs SharedPreferences Flow

```
SharedPreferences:
Read  → Load entire XML from disk (first access) → return value
Write → apply(): update in-memory Map immediately
               → schedule async disk write on QueuedWork thread

DataStore:
Read  → Suspend: read from proto/preferences file
      → Return Flow<T> (hot, cached, updated on each write)
Write → Suspend: atomic write to disk via File.renameTo()
      → Transactional: all-or-nothing (no partial writes)
      → Flow updates all collectors after write completes
```

---

## Common Mistakes (All Persistence)

### SharedPreferences

| Mistake | Problem | Fix |
|---------|---------|-----|
| `commit()` on main thread | ANR | Use `apply()` |
| Multiple SharedPreferences files | Fragmented data, hard to manage | One file per logical group |
| Storing sensitive data unencrypted | Security vulnerability | Use `EncryptedSharedPreferences` |
| Accessing same key with different types | Silent null return | Define typed constants |

### DataStore

| Mistake | Problem | Fix |
|---------|---------|-----|
| Creating DataStore in a class body | Multiple instances, file corruption | `by preferencesDataStore` at top level |
| Not catching `IOException` in `.data` | Unhandled file corruption | Always add `.catch { if (it is IOException) emit(emptyPreferences()) else throw it }` |
| Calling `dataStore.edit{}` without try-catch | Coroutine crash on disk error | Wrap in `try-catch(IOException)` |
| Using DataStore for large lists | Not designed for large datasets | Use Room |

### Room

| Mistake | Problem | Fix |
|---------|---------|-----|
| Calling DAO on main thread | `IllegalStateException` | DAO suspend functions run on IO thread automatically |
| `allowMainThreadQueries()` in production | ANR risk | Only in tests |
| Not providing migrations | `IllegalStateException` on DB upgrade | Always write migrations |
| Missing `@Transaction` for multiple writes | Partial write on crash | Use `@Transaction` |
| Not closing cursor in raw queries | Cursor leak, memory issues | Use Room's DAO — Room handles cursors |
| Querying huge datasets in UI thread | Slow rendering | Use `Flow` + `stateIn()` or Paging |

---

## Memory Leak / Performance Concerns

### 1. Room Database — Single Instance

```kotlin
// ❌ WRONG: New database instance per call
class ProductRepository(val context: Context) {
    fun getProducts() = Room.databaseBuilder(...).build().productDao().getAll()
    // Creates new DB file handle every time!
}

// ✅ CORRECT: Singleton (handled by Hilt @Singleton)
@Module @InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase =
        Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
            .addMigrations(AppDatabase.MIGRATION_1_2)
            .build()

    @Provides
    fun provideProductDao(db: AppDatabase): ProductDao = db.productDao()
}
```

### 2. DataStore — Single Instance Per File

```kotlin
// ❌ WRONG: Multiple instances can corrupt the file
class SettingsRepository(val context: Context) {
    fun getDarkMode() = context.createDataStore(name = "settings") // ← new instance!
        .data.map { it[DARK_MODE_KEY] }
}

// ✅ CORRECT: Top-level singleton via delegate
val Context.dataStore by preferencesDataStore("settings") // One instance per Context
```

### 3. Large Room Queries — Use Paging

```kotlin
// ❌ WRONG: Loading 10,000 rows into memory
@Query("SELECT * FROM logs")
fun getAllLogs(): Flow<List<LogEntity>> // 10k items in memory!

// ✅ CORRECT: Use Paging 3
@Query("SELECT * FROM logs ORDER BY created_at DESC")
fun getLogsPaged(): PagingSource<Int, LogEntity>

// Repository
fun getLogsPaged(): Flow<PagingData<Log>> = Pager(
    config = PagingConfig(pageSize = 20, enablePlaceholders = false),
    pagingSourceFactory = { logDao.getLogsPaged() }
).flow
```

### 4. Flow Collection Lifecycle

```kotlin
// ❌ WRONG: Room Flow collected without repeatOnLifecycle
lifecycleScope.launch {
    viewModel.products.collect { render(it) }
    // Keeps running when app is backgrounded — waste of resources
}

// ✅ CORRECT
lifecycleScope.launch {
    repeatOnLifecycle(Lifecycle.State.STARTED) {
        viewModel.products.collect { render(it) }
    }
}
```

---

## Interview Questions

### Beginner

**Q1: When would you use SharedPreferences vs Room?**

> **SharedPreferences/DataStore**: For small, simple key-value data — user settings (dark mode, language, font size), auth tokens, onboarding completion flags. Limited to primitive types and small data.
>
> **Room**: For structured, relational data that needs querying, filtering, or joining — product catalogs, user profiles, transaction history, cached API responses. Handles large datasets with type-safe queries.

**Q2: What is the difference between `apply()` and `commit()` in SharedPreferences?**

> `apply()` — writes changes to the in-memory cache immediately (so reads are updated right away) and schedules the disk write asynchronously. Non-blocking, preferred.
>
> `commit()` — performs the disk write synchronously on the calling thread and returns `true`/`false` indicating success. Blocks the thread. Only use when you absolutely need to know if the write succeeded before proceeding (rare).

**Q3: What are the three annotations in Room and what does each do?**

> - `@Entity` — marks a data class as a database table. Each property becomes a column.
> - `@Dao` — marks an interface as a Data Access Object. Room generates the implementation with SQL.
> - `@Database` — marks an abstract class as the database. Lists all entities and the version number. Room generates the concrete database class.

---

### Intermediate

**Q4: Why does Room return `Flow<List<T>>` from `@Query` methods, and how does it know when to emit new data?**

> Room uses SQLite's `sqlite3_update_hook` (via its `InvalidationTracker`) to detect when tables are modified. When you return `Flow<List<T>>` from a DAO, Room registers the affected tables with the `InvalidationTracker`. Whenever a write occurs to any of those tables (INSERT, UPDATE, DELETE), the `InvalidationTracker` notifies the Flow, which re-executes the query and emits the new result. This gives you automatic, reactive UI updates without any polling.

**Q5: What happens if you upgrade your Room database version without providing a migration?**

> By default, Room throws `IllegalStateException: Room cannot verify the integrity of the database` and the app crashes. This forces developers to handle schema changes explicitly.
>
> If you call `.fallbackToDestructiveMigration()`, Room will drop and recreate the database, losing all user data. Only use this during development.
>
> The correct approach: always write `Migration(fromVersion, toVersion)` objects with the necessary SQL (`ALTER TABLE`, `CREATE TABLE`, etc.) and register them with `addMigrations()`.

**Q6: What is the difference between Preferences DataStore and Proto DataStore?**

> **Preferences DataStore** works like SharedPreferences with type-safe keys (`intPreferencesKey()`, `stringPreferencesKey()` etc.). It's simpler to set up but doesn't enforce a schema — any key can be added at any time.
>
> **Proto DataStore** uses Protocol Buffers (`.proto` schema file). The schema is strictly defined and compiled — you get fully typed Kotlin objects, versioning support, and efficient binary serialization (more compact than XML). Proto DataStore is better for complex settings with many fields or when data evolution/versioning matters.

---

### Advanced

**Q7: Explain how Room's `@Transaction` annotation works and why it's important for `@Relation` queries.**

> `@Transaction` wraps the entire function in a single SQLite transaction. For write operations, this ensures atomicity — if multiple INSERTs/DELETEs fail midway, all changes are rolled back, preventing data corruption.
>
> For `@Relation` queries (like `CategoryWithProducts`), `@Transaction` is required because Room executes two queries: one for the parent entity and one for the children. Without `@Transaction`, another write could occur between the two reads, resulting in an inconsistent joined result. `@Transaction` ensures both queries see the same database snapshot.

**Q8: How would you implement encrypted local storage for sensitive data (auth tokens, health records) in Android?**

> Two options:
>
> 1. **EncryptedSharedPreferences** (Jetpack Security):
> ```kotlin
> val masterKey = MasterKey.Builder(context)
>     .setKeyScheme(MasterKey.KeyScheme.AES256_GCM)
>     .build()
>
> val encryptedPrefs = EncryptedSharedPreferences.create(
>     context,
>     "secret_prefs",
>     masterKey,
>     EncryptedSharedPreferences.PrefKeyEncryptionScheme.AES256_SIV,
>     EncryptedSharedPreferences.PrefValueEncryptionScheme.AES256_GCM
> )
> ```
>
> 2. **SQLCipher with Room** — for large encrypted relational data:
> ```kotlin
> Room.databaseBuilder(context, AppDatabase::class.java, "encrypted.db")
>     .openHelperFactory(SupportFactory(SQLiteDatabase.getBytes("passphrase".toCharArray())))
>     .build()
> ```
>
> Keys should be stored in Android Keystore (via `MasterKey`), not hardcoded in source code.

---

## Scenario Questions

**Scenario 1:** The app must work offline. Products are fetched from API and cached. Explain your complete data persistence architecture.

> 1. **Room** as Single Source of Truth — stores all products as `ProductEntity`
> 2. **Repository** implements `networkBoundResource` pattern:
>    - Immediately emit cached Room data (works offline from first access)
>    - Trigger network refresh in background
>    - On success: update Room → Room Flow emits → UI updates automatically
>    - On failure: emit `Resource.Error` but still show cached data
> 3. **DataStore** stores last-refresh timestamp — Repository skips network call if refreshed < 5 min ago
> 4. **ViewModel** exposes `StateFlow<Resource<List<Product>>>` combining fresh + cached states
> 5. UI shows stale indicator ("Last updated: 2 hours ago") when offline

**Scenario 2:** Your Room database needs to add a `description` column to an existing `products` table in version 2 without losing user data.

```kotlin
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        // ALTER TABLE to add new nullable column with default
        db.execSQL(
            "ALTER TABLE products ADD COLUMN description TEXT NOT NULL DEFAULT ''"
        )
    }
}

// Register in database builder:
Room.databaseBuilder(context, AppDatabase::class.java, "app.db")
    .addMigrations(MIGRATION_1_2)
    .build()
```

---

## Code Examples (Production-Quality Kotlin)

### Complete Room + Repository + Hilt Setup

```kotlin
// Hilt DI Module
@Module
@InstallIn(SingletonComponent::class)
object DatabaseModule {

    @Provides
    @Singleton
    fun provideDatabase(@ApplicationContext context: Context): AppDatabase {
        return Room.databaseBuilder(
            context,
            AppDatabase::class.java,
            "app_database.db"
        )
            .addMigrations(AppDatabase.MIGRATION_1_2, AppDatabase.MIGRATION_2_3)
            .build()
    }

    @Provides
    fun provideProductDao(db: AppDatabase): ProductDao = db.productDao()

    @Provides
    fun provideCategoryDao(db: AppDatabase): CategoryDao = db.categoryDao()
}

@Module
@InstallIn(SingletonComponent::class)
abstract class RepositoryModule {
    @Binds
    @Singleton
    abstract fun bindProductRepository(impl: ProductRepositoryImpl): ProductRepository
}
```

### DataStore Singleton — Best Practice

```kotlin
// AppDataStore.kt (top-level, not in a class)
private val Context.appDataStore: DataStore<Preferences> by preferencesDataStore(
    name = "app_settings"
)

object AppPreferenceKeys {
    val DARK_MODE = booleanPreferencesKey("dark_mode")
    val FONT_SIZE = intPreferencesKey("font_size")
    val LANGUAGE = stringPreferencesKey("language")
    val ONBOARDING_COMPLETE = booleanPreferencesKey("onboarding_complete")
    val LAST_SYNC_TIMESTAMP = longPreferencesKey("last_sync_timestamp")
}

class AppSettingsRepository @Inject constructor(
    @ApplicationContext private val context: Context
) {
    private val dataStore = context.appDataStore

    val settings: Flow<AppSettings> = dataStore.data
        .catch { exception ->
            if (exception is IOException) emit(emptyPreferences())
            else throw exception
        }
        .map { prefs ->
            AppSettings(
                isDarkMode = prefs[AppPreferenceKeys.DARK_MODE] ?: false,
                fontSize = prefs[AppPreferenceKeys.FONT_SIZE] ?: 14,
                language = prefs[AppPreferenceKeys.LANGUAGE] ?: "en",
                onboardingComplete = prefs[AppPreferenceKeys.ONBOARDING_COMPLETE] ?: false,
                lastSyncTimestamp = prefs[AppPreferenceKeys.LAST_SYNC_TIMESTAMP] ?: 0L
            )
        }

    suspend fun updateDarkMode(enabled: Boolean) {
        dataStore.edit { it[AppPreferenceKeys.DARK_MODE] = enabled }
    }

    suspend fun markOnboardingComplete() {
        dataStore.edit { it[AppPreferenceKeys.ONBOARDING_COMPLETE] = true }
    }

    suspend fun updateLastSyncTimestamp() {
        dataStore.edit { it[AppPreferenceKeys.LAST_SYNC_TIMESTAMP] = System.currentTimeMillis() }
    }
}

data class AppSettings(
    val isDarkMode: Boolean,
    val fontSize: Int,
    val language: String,
    val onboardingComplete: Boolean,
    val lastSyncTimestamp: Long
)
```

### Full Room Entity + DAO with Paging 3

```kotlin
// Entity
@Entity(tableName = "news_articles")
data class NewsArticleEntity(
    @PrimaryKey val id: String,
    val title: String,
    val summary: String,
    val imageUrl: String?,
    val publishedAt: Long,
    val sourceId: String,
    val category: String,
    @ColumnInfo(name = "is_bookmarked") val isBookmarked: Boolean = false
)

// DAO
@Dao
interface NewsArticleDao {

    @Query("SELECT * FROM news_articles ORDER BY publishedAt DESC")
    fun getArticlesPaged(): PagingSource<Int, NewsArticleEntity>

    @Query("SELECT * FROM news_articles WHERE category = :category ORDER BY publishedAt DESC")
    fun getArticlesByCategory(category: String): PagingSource<Int, NewsArticleEntity>

    @Query("SELECT * FROM news_articles WHERE is_bookmarked = 1 ORDER BY publishedAt DESC")
    fun getBookmarkedArticles(): Flow<List<NewsArticleEntity>>

    @Query("SELECT * FROM news_articles WHERE id = :id")
    fun getArticleById(id: String): Flow<NewsArticleEntity?>

    @Insert(onConflict = OnConflictStrategy.REPLACE)
    suspend fun insertAll(articles: List<NewsArticleEntity>)

    @Query("UPDATE news_articles SET is_bookmarked = :bookmarked WHERE id = :id")
    suspend fun setBookmarked(id: String, bookmarked: Boolean)

    @Query("DELETE FROM news_articles WHERE is_bookmarked = 0 AND publishedAt < :cutoffTime")
    suspend fun deleteOldUnbookmarkedArticles(cutoffTime: Long)

    @Query("SELECT COUNT(*) FROM news_articles")
    suspend fun getCount(): Int
}
```

---

## Best Practices

### General

1. **Match the tool to the need** — don't use Room for 3 settings, don't use DataStore for 500 records
2. **Always use Repository pattern** — never access DAO/DataStore directly from ViewModel
3. **Inject via Hilt** — database and DataStore as `@Singleton`

### DataStore

4. **Single instance per file** — `by preferencesDataStore` at top level or in Hilt module
5. **Always handle `IOException`** in `.catch{}` operator
6. **Prefer Preferences DataStore** for simple settings; Proto for complex/versioned schemas

### Room

7. **Always write migrations** — never rely on `fallbackToDestructiveMigration()` in production
8. **Return `Flow<T>`** for read queries — reactive, no manual refresh
9. **Use `@Transaction`** for multi-step writes and `@Relation` reads
10. **Export schema** — `exportSchema = true` and commit schema JSON to source control for migration audit
11. **Use TypeConverters** for complex types — don't store JSON strings manually in columns
12. **Test with in-memory database** — `Room.inMemoryDatabaseBuilder()` for unit tests

---

## Legacy vs Modern

| Aspect | Legacy | Modern |
|--------|--------|--------|
| Key-value storage | SharedPreferences (XML, non-reactive) | DataStore (Flow-based, coroutine-safe) |
| Typed settings | `PreferenceManager.getDefaultSharedPreferences()` ⚠️ | Preferences DataStore with typed keys |
| Complex settings | XML-based SharedPreference files | Proto DataStore |
| Relational data | `SQLiteOpenHelper` + `Cursor` | Room (`@Entity`, `@Dao`, `@Database`) |
| Reactive DB | `ContentObserver` + manual polling | Room `Flow<T>` |
| DB on main thread | `allowMainThreadQueries()` or `AsyncTask` | Room suspends automatically, use `Dispatchers.IO` |
| Sharing files | Direct file path | FileProvider URI |
| External media | `READ_EXTERNAL_STORAGE` + direct File access | MediaStore + SAF (Scoped Storage API 29+) |

> ⚠️ `PreferenceFragment` and `PreferenceActivity` are deprecated. Use `PreferenceFragmentCompat` with Jetpack Preference library.
> ⚠️ `MODE_WORLD_READABLE` and `MODE_WORLD_WRITEABLE` removed in API 17.
> ⚠️ Direct external storage access (`Environment.getExternalStorageDirectory()`) deprecated for Scoped Storage on API 29+.

---

## Revision Notes

**SharedPreferences:**
- XML key-value; `apply()` async, `commit()` sync; not type-safe; not Flow-based; ⚠️ legacy

**DataStore:**
- `preferencesDataStore` delegate = singleton; `dataStore.data` = `Flow<Preferences>`; `dataStore.edit{}` = suspend write
- Always `.catch { if (it is IOException) emit(emptyPreferences()) }`
- Proto DataStore = typed schema, binary format

**Files:**
- Internal: `context.filesDir`, no permission, auto-deleted on uninstall
- External: `getExternalFilesDir()` no permission (app-specific); `MediaStore` for shared media
- Scoped Storage (API 29+): no `READ_EXTERNAL_STORAGE` for own files; use `MediaStore`/`SAF`
- `FileProvider` for sharing internal files with other apps

**SQLite (Legacy):**
- `SQLiteOpenHelper`: `onCreate()`, `onUpgrade()`; `Cursor`-based; not type-safe ⚠️

**Room:**
- `@Entity` = table; `@Dao` = interface; `@Database` = config
- SQL verified at compile time; `Flow<T>` for reactive reads; suspend for writes
- Migrations: `Migration(from, to)` + `addMigrations()`; never skip
- `@Transaction` for multi-write and `@Relation` joins
- Test: `inMemoryDatabaseBuilder` + `allowMainThreadQueries()`
- Hilt: `@Singleton` database provided via `@Module`

---

## Key Takeaways

> 🗃️ Use the right persistence layer: DataStore for settings, Room for structured data, Files for raw binary/documents — don't force one tool to do everything.

> ⚗️ Room's compile-time SQL verification and `Flow<T>` reactive queries are its killer features — you catch bugs at build time, not in production.

> 🔄 Always write Room migrations — a missing migration causes an `IllegalStateException` crash on upgrade. Make it a checklist item for every schema change.

> 🚰 DataStore's `Flow`-based API integrates cleanly with ViewModel's `StateFlow` — chain with `.map{}`, `.stateIn()`, and you have a fully reactive settings system.

> 🔒 Never store sensitive data unencrypted — use `EncryptedSharedPreferences` or SQLCipher with Room for health data, auth tokens, and PII.

> 📦 Room as Single Source of Truth + Repository pattern = the foundation of robust offline-first Android apps.

---

# Part 4 — Summary Table

| Chapter | Core Concept | Key Pattern | Modern API | Avoid |
|---------|-------------|------------|------------|-------|
| 12: MVVM | Separation of concerns | Repository + UDF | `StateFlow`, `Channel`, `@HiltViewModel` | God Activity, mutable public state |
| 13: ViewModel | Config change survival | `ViewModelStore` + `NonConfigurationInstances` | `by viewModels()`, `viewModelScope` | Holding Context/View, GlobalScope |
| 14: SavedStateHandle | Process death survival | Bundle + ViewModel bridge | `getStateFlow()`, `@Parcelize` | Large objects, sensitive data in Bundle |
| 15: Navigation | Declarative navigation | NavGraph + NavController | Safe Args, `NavBackStackEntry` ViewModel | Manual FragmentTransactions, no Safe Args |
| 16: Persistence | Data storage | Repository + SSOT | Room, DataStore, MediaStore | SharedPreferences for new code, no migrations |

---

*End of Part 4 — Architecture & Persistence*

*Next: Part 5 — UI Components & RecyclerView*
