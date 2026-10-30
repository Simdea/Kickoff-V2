---
name: android-implementation-skill
description: >
  Production Android implementation skill for the multi-repo Kotlin/Compose ecosystem.
  Use when: implementing Kotlin features, writing UseCases/Repositories/ViewModels, scaffolding 
  Compose screens, writing Hilt DI modules, implementing Coroutines/Flow pipelines, adding tests,
  or modifying existing code in common/utilities/services/sdk-android/app.
  Always operates within explicit ACC boundaries.
---

# Android Implementation Skill

> Always load `android-architecture-core-skill` before using this skill.  
> Never implement without a valid acc_id (STRUCTURAL) or ACC-lite (LOCAL).

---

## 1. Pre-Implementation Checklist

Before writing any code:

- [ ] acc_id or ACC-lite scope is confirmed
- [ ] Target repo and Gradle module identified
- [ ] `settings.gradle` / `build.gradle` confirmed — no unintended dependency changes
- [ ] At least one analogous implementation in the same module read and understood
- [ ] Hilt module entry point for this module located
- [ ] Navigation and UiState patterns for the target presentation module identified
- [ ] All new domain interfaces have a Fake planned

---

## 2. Domain Layer Implementation

### UseCase Pattern

```kotlin
class <Name>UseCase @Inject constructor(
    private val repository: I<Name>Repository,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) {
    suspend operator fun invoke(params: <Params>): Result<<ReturnType>> =
        withContext(dispatcher) {
            runCatching { repository.<action>(params) }
        }
}
```

Rules:
- No Android framework imports.
- No `Context` dependency.
- Dispatcher injected, not hardcoded.
- Returns `Result<T>` or a domain-specific type — never raw exceptions.
- Corresponding `Fake<Name>Repository` must ship with the implementation.

### Repository Interface Pattern

```kotlin
interface I<Name>Repository {
    suspend fun <action>(params: <Params>): <ReturnType>
    fun <streamAction>(): Flow<<ReturnType>>
}
```

---

## 3. Infrastructure Layer Implementation

### Repository Implementation Pattern

```kotlin
class <Name>RepositoryImpl @Inject constructor(
    private val remoteDataSource: I<Name>RemoteDataSource,
    private val localDataSource: I<Name>LocalDataSource,
    private val mapper: <Name>Mapper
) : I<Name>Repository {

    override suspend fun <action>(params: Params): ReturnType =
        mapper.toDomain(remoteDataSource.fetch(mapper.toDto(params)))
}
```

Rules:
- Maps at every boundary — never expose DTO or Room entity to domain.
- No business logic in repository — only data orchestration.
- `@Inject` constructor — no companion object factories.

---

## 4. Presentation Layer Implementation

### ViewModel Pattern

```kotlin
@HiltViewModel
class <Name>ViewModel @Inject constructor(
    private val use<Action>UseCase: <Action>UseCase,
    private val dispatcher: CoroutineDispatcher = Dispatchers.IO
) : ViewModel() {

    private val _uiState = MutableStateFlow(<Name>UiState.Loading)
    val uiState: StateFlow<<Name>UiState> = _uiState.asStateFlow()

    private val _events = MutableSharedFlow<Name>Event>(replay = 0)
    val events: SharedFlow<<Name>Event> = _events.asSharedFlow()

    fun onIntent(intent: <Name>Intent) {
        viewModelScope.launch(dispatcher) {
            when (intent) {
                is <Name>Intent.<Action> -> handle<Action>(intent)
            }
        }
    }

    private suspend fun handle<Action>(intent: <Name>Intent.<Action>) {
        use<Action>UseCase(intent.params)
            .onSuccess { _uiState.value = <Name>UiState.Success(it) }
            .onFailure { _uiState.value = <Name>UiState.Error(it.message) }
    }
}
```

### UiState Pattern

```kotlin
sealed class <Name>UiState {
    data object Loading : <Name>UiState()
    data class Success(val data: <DomainType>) : <Name>UiState()
    data class Error(val message: String?) : <Name>UiState()
}
```

### Compose Screen Pattern

```kotlin
@Composable
fun <Name>Screen(
    uiState: <Name>UiState,
    onIntent: (<Name>Intent) -> Unit
) {
    when (uiState) {
        is <Name>UiState.Loading -> <Name>LoadingContent()
        is <Name>UiState.Success -> <Name>Content(data = uiState.data, onIntent = onIntent)
        is <Name>UiState.Error -> <Name>ErrorContent(message = uiState.message)
    }
}

@Composable
fun <Name>ScreenRoute(viewModel: <Name>ViewModel = hiltViewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    <Name>Screen(uiState = uiState, onIntent = viewModel::onIntent)
}
```

Rules:
- Route composable owns ViewModel; content composable is pure/stateless.
- Never call `suspend` functions directly inside composable body — use `LaunchedEffect`.
- Navigation triggers via `events` SharedFlow collected in `LaunchedEffect`.

---

## 5. Hilt DI Module Pattern

```kotlin
@Module
@InstallIn(SingletonComponent::class)
abstract class <Name>Module {

    @Binds
    @Singleton
    abstract fun bind<Name>Repository(impl: <Name>RepositoryImpl): I<Name>Repository
}
```

Rules:
- `@Binds` preferred over `@Provides` for interface binding.
- DI module lives in the `-application` Gradle module.
- ViewModel bindings handled automatically by `@HiltViewModel`.
- No business logic in DI modules.

---

## 6. Test Implementation Patterns

### UseCase Unit Test

```kotlin
@ExtendWith(CoroutinesTestExtension::class)
class <Name>UseCaseTest {

    private val fakeRepository = Fake<Name>Repository()
    private val useCase = <Name>UseCase(fakeRepository, UnconfinedTestDispatcher())

    @Test
    fun `given valid input, returns success`() = runTest {
        fakeRepository.resultToReturn = Result.success(<domain_object>)
        val result = useCase(<params>)
        assertThat(result.isSuccess).isTrue()
    }
}
```

### ViewModel Unit Test with Turbine

```kotlin
@ExtendWith(CoroutinesTestExtension::class)
class <Name>ViewModelTest {

    private val fakeUseCase = Fake<Name>UseCase()
    private val viewModel = <Name>ViewModel(fakeUseCase, UnconfinedTestDispatcher())

    @Test
    fun `when intent sent, emits success state`() = runTest {
        fakeUseCase.resultToReturn = Result.success(<data>)
        viewModel.uiState.test {
            viewModel.onIntent(<Name>Intent.<Action>(<params>))
            assertThat(awaitItem()).isInstanceOf(<Name>UiState.Success::class.java)
            cancelAndIgnoreRemainingEvents()
        }
    }
}
```

### Fake Pattern

```kotlin
class Fake<Name>Repository : I<Name>Repository {
    var resultToReturn: Result<<ReturnType>> = Result.failure(NotImplementedError())

    override suspend fun <action>(params: Params): <ReturnType> =
        resultToReturn.getOrThrow()
}
```

---

## 7. Mapper Convention

```kotlin
class <Name>Mapper @Inject constructor() {
    fun toDomain(dto: <Name>Dto): <Name>DomainModel = <Name>DomainModel(
        id = dto.id,
        // ... explicit field mapping
    )

    fun toDto(domain: <Name>DomainModel): <Name>Dto = <Name>Dto(
        id = domain.id,
    )
}
```

No extension functions for mappers that cross layer boundaries — use injected mapper classes for testability.

---

## 8. Post-Implementation Checklist

- [ ] All new domain interfaces have a corresponding `Fake` implementation
- [ ] No `GlobalScope`, no `runBlocking` in production code
- [ ] Dispatchers injected — not hardcoded
- [ ] StateFlow exposed — not MutableStateFlow
- [ ] Mapper used at every layer boundary — no DTO leaks
- [ ] All tests use `runTest` / `TestScope` — no `Thread.sleep`
- [ ] Hilt module placed in `-application` Gradle module
- [ ] `domain` module has zero Android framework imports
- [ ] acc_id referenced in PR description

> ACC Input Contract must be confirmed before any implementation step (see pre-implementation checklist).
> Flow tests use turbine library for deterministic assertions.
