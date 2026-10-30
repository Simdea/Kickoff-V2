---
name: android-arch-layers-skill
description: >
  Layer architecture, dependency direction, Hilt/DI rules, Coroutines/Flow contracts,
  Compose UI contracts, and non-negotiable invariants for the Android ecosystem.
  Use when: validating layer boundaries, checking dependency direction, reviewing DI graph,
  assessing Coroutines/Flow correctness, or enforcing non-negotiable invariants.
  Loaded by: AndroidArchitectAgent, AndroidTestReviewer, LeadAndroidAgent.
---

# Android Architecture — Layers, Invariants & Contracts

> Atomic skill. Covers structural rules only.
> For workspace layout, tech stack and naming conventions → load `android-arch-conventions-skill`.
> For build/test commands → load `android-arch-build-skill`.

---

## 1. Layer Architecture

```
Compose UI  →  ViewModel  →  UseCase  →  Repository (interface)
                                              ↓
                                    RepositoryImpl (infrastructure)
                                              ↓
                                    DataSource (remote/local)
```

### Layer Ownership

| Layer | Gradle Module Suffix | Allowed Dependencies |
|-------|---------------------|---------------------|
| Presentation | `-presentation` | domain contracts, UI toolkit |
| Domain | `-domain` | nothing outside domain |
| Infrastructure | `-infrastructure` | domain contracts, data libraries |
| App/Bootstrap | `-application` | presentation, infrastructure, domain, DI wiring |

**Invariant:** `domain` must never import `infrastructure` or `presentation`.

---

## 2. Dependency Direction (Strict)

```
app  →  sdk-android, services, utilities, common
sdk-android       →  common
services          →  utilities, common
utilities         →  common
common            →  (nothing internal)
```

Reverse dependency direction is **forbidden**. Any inter-repo dependency not listed above requires an ACC.

---

## 3. Hilt/DI Graph Rules

1. Component hierarchy: `SingletonComponent` → `ActivityRetainedComponent` → `ViewModelComponent` → `ActivityComponent`
2. Use `@ViewModelScoped` for ViewModels, `@Singleton` for repositories.
3. No ServiceLocator pattern.
4. No circular DI dependencies.
5. Multi-binding (`@IntoMap`, `@IntoSet`) across module boundaries must be documented in the ACC.
6. Fakes (not Mocks) used to replace injected dependencies in unit tests.

---

## 4. Coroutines + Flow Contracts

| Rule | Detail |
|------|--------|
| No `GlobalScope` | Always use structured concurrency (`viewModelScope`, `lifecycleScope`, coroutine scope passed by constructor) |
| No `runBlocking` in production | Only allowed in tests |
| Dispatcher injection | ViewModels and UseCases receive `CoroutineDispatcher` via constructor (enables test replacement) |
| StateFlow preferred | ViewModels expose `StateFlow<UiState>` — never `MutableStateFlow` publicly |
| SharedFlow for events | One-time UI events use `SharedFlow` with `replay = 0` |
| No Thread.sleep in tests | Use `TestCoroutineScheduler.advanceUntilIdle()` or `runTest` |
| Flow collection in tests | Use `turbine` library or `toList` + `take(n)` |

---

## 5. Compose UI Contracts

| Rule | Detail |
|------|--------|
| State hoisting | Composable functions receive state + callbacks, never business logic |
| UiState is sealed | Each screen has a `sealed class UiState` or `data class UiState` |
| Navigation | Use Jetpack Navigation Compose; no direct Activity/Fragment starts from Composables |
| Side effects | Use `LaunchedEffect`, `SideEffect`, `DisposableEffect` — never call `suspend` directly inside composable body |
| Previews | All composables have `@Preview` functions; no production logic in previews |

---

## 6. Non-Negotiable Invariants

1. No `GlobalScope`.
2. No `runBlocking` in production code.
3. No `lateinit var` for ViewModel state — use `StateFlow`.
4. No cross-layer data class leakage — mappers at every layer boundary.
5. No direct dependency on `android.content.Context` in UseCase or domain classes.
6. Every public domain interface has a Fake.
7. `domain` module has zero Android framework dependencies.
8. Local-vs-remote toggle changes require an ACC.
9. Gradle module topology changes require Lead approval + ACC.
