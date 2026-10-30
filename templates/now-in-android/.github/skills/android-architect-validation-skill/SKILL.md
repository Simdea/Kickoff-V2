---
name: android-architect-validation-skill
description: >
  Architecture validation workflow for the Android multi-repo ecosystem. Use when:
  classifying scope as LOCAL or STRUCTURAL, issuing ACCEPTABLE/INSUFFICIENT architecture verdicts,
  generating ACC or ACC-lite entries, performing dependency direction analysis, Hilt graph hygiene review,
  Gradle module topology review, or local-vs-remote toggle impact analysis.
---

# Android Architect Validation

> Load `shared-architect-validation-protocol-skill` first, then `android-arch-layers-skill`.
> This skill adds Android-specific scope classification, Gradle/Hilt checklists, ACC YAML template, and risk escalation.
> Used exclusively by **AndroidArchitectAgent**.

---

## 1. Scope Classification

Before every architectural decision, classify the proposed change:

| Classification | Criterion |
|---------------|-----------|
| **LOCAL** | Change confined to a single Gradle module; no new cross-module dependencies; no DI graph changes outside the module; no settings.gradle changes |
| **STRUCTURAL** | Change affects ≥2 modules; introduces new cross-repo dependency; changes Gradle module topology; toggles local-vs-remote dependency; modifies Hilt component hierarchy; introduces new multi-binding across module boundaries |

**If ambiguous, classify as STRUCTURAL.**

> Cross-platform note: iOS/Java use a 4-tier taxonomy (LOCAL / CROSS-MODULE / CROSS-REPO / SYSTEMIC).
> Android STRUCTURAL maps to all three of those tiers — the ACC requirement threshold is the same: anything beyond LOCAL requires an ACC.

---

## 2. Architecture Verdict Protocol

For every architectural review, produce a verdict in this format:

```
## Architecture Verdict

**Scope:** LOCAL | STRUCTURAL
**ACC required:** YES | NO (ACC-lite for LOCAL scope)
**Verdict:** ACCEPTABLE | INSUFFICIENT
**Risk level:** LOW | MEDIUM | HIGH

### Findings
- <finding 1>
- <finding 2>

### Required ACC fields (if STRUCTURAL)
- acc_id: AND-ACC-<YYYY>-<NNN>
- acc_scope: STRUCTURAL
- acc_storage_path: .github/governance/android/acc-registry-android.yaml

### Conditions for ACCEPTABLE
- <condition if verdict is INSUFFICIENT>
```

---

## 3. ACC Template (STRUCTURAL scope)

When issuing a new ACC for a STRUCTURAL change, register it in `.github/governance/android/acc-registry-android.yaml`:

```yaml
- acc_id: AND-ACC-YYYY-NNN
  title: "<short description>"
  scope: STRUCTURAL
  date: "YYYY-MM-DD"
  status: APPROVED | PENDING | REJECTED
  author: AndroidArchitectAgent
  lead_approval: APPROVED | PENDING
  summary: >
    <What changes. Why. Which modules/repos affected.>
  affected_modules:
    - <repo>/<module>
  dependency_changes:
    - from: <source>
      to: <target>
      type: new | removed | source-changed
  hilt_impact: none | scope-change | new-binding | multi-binding
  local_remote_toggle: false | true
  invariants_verified:
    - domain_no_android_imports: true | false | N/A
    - no_cross_layer_data_leak: true | false | N/A
    - fake_contract_maintained: true | false | N/A
```

---

## 4. ACC-lite Template (LOCAL scope)

For LOCAL scope only — no registry entry required, but document in implementation plan:

```
## ACC-lite
- scope: LOCAL
- module: <repo>/<module>
- summary: <What changes. Why.>
- layer_boundaries_respected: YES | NO
- hilt_impact: none | constructor injection only
- no_settings_gradle_change: YES
```

---

## 5. Architectural Review Checklist

### Layering
- [ ] `domain` module imports no Android framework classes
- [ ] `presentation` does not import `infrastructure` directly  
- [ ] `infrastructure` implements `domain` interfaces — not the reverse
- [ ] `app` module contains DI wiring only — no business logic

### Dependency Direction
- [ ] Inter-module dependencies follow the canonical direction (Section 6 of architecture-core-skill)
- [ ] No circular module dependencies
- [ ] No new inter-repo dependencies without an ACC

### Hilt/DI
- [ ] Component scope is appropriate (`@Singleton` / `@ViewModelScoped` / `@ActivityScoped`)
- [ ] No ServiceLocator usage
- [ ] Fakes provided for new domain interfaces
- [ ] Multi-binding changes documented if cross-module

### Coroutines/Flow
- [ ] No `GlobalScope` usage
- [ ] Dispatchers injected (not hardcoded)
- [ ] ViewModels expose `StateFlow` — not `MutableStateFlow`

### Gradle
- [ ] No new module added without Lead approval
- [ ] Local-vs-remote toggle not changed without ACC
- [ ] `settings.gradle` / `settingsVariables.gradle` changes reviewed

### Testing
- [ ] Fake provided for each new domain interface
- [ ] Coroutine tests use `runTest` or `TestScope`
- [ ] Flow assertions use turbine or `take(n)` — no `Thread.sleep`

---

## 6. Risk Escalation

Escalate to LeadAndroidAgent when:

- Risk level is **HIGH**
- ACC status is **PENDING** (not yet APPROVED)
- Change affects ≥3 repos simultaneously
- Coroutine dispatcher architecture changes at app module level
- Hilt `SingletonComponent` binding changes
- local-vs-remote dependency toggle in `app` or `sdk-android`
- Proposed change violates any non-negotiable invariant

**INSUFFICIENT** verdict blocks implementation. AndroidProgrammerAgent must not proceed until verdict is resolved.

---

## 7. Output Contract

Every response from AndroidArchitectAgent using this skill must include:

1. Scope classification (LOCAL/STRUCTURAL)
2. Architecture verdict (ACCEPTABLE/INSUFFICIENT)
3. ACC or ACC-lite reference
4. Full checklist evaluation summary
5. Open risks or conditions (if INSUFFICIENT)
