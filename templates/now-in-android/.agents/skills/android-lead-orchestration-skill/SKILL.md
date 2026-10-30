---
name: android-lead-orchestration-skill
description: >
  Lead orchestration workflow for the Android multi-repo ecosystem. Use when: 
  planning and delegating Android scopes, registering ACCs, managing gate sequencing,
  enforcing backlog state, issuing final APPROVED/REJECTED/BLOCKED verdicts, 
  or supervising the agent pipeline from intent to delivery.
---

# Android Lead Orchestration

> This skill is used exclusively by **LeadAndroidAgent**.
> Always load `android-arch-layers-skill` first (replaces the old android-architecture-core-skill index).

---

## 1. Orchestration State Machine

Every governed scope must pass through this pipeline in order:

```
IntentDefined
    ↓ (optional HistoricalContext requested by Lead)
    ↓ (ACC or ACC-lite registered)
ArchitectValidated     ← AndroidArchitectAgent: ACCEPTABLE required
    ↓ (acc_id APPROVED)
ACCLocked
    ↓ (implementation starts)
Implemented            ← AndroidProgrammerAgent delivers
    ↓ (tests submitted)
ChaosValidated         ← AndroidTestReviewerAgent: ACCEPTABLE required
    ↓ (lead reviews)
LeadFinalized          ← LeadAndroidAgent: APPROVED | REJECTED | BLOCKED
```

No step may be skipped. Returning to a prior step requires explicit state rollback logged in `lead-backlog.md`.

HistoricalContext is optional:
- Triggered only by Lead
- Produced by `GitForensicsAgent`
- Used to reduce ambiguity before architecture review
- Never treated as a formal gate
- Never allowed to replace architecture, implementation, or test review

---

## 2. ACC Reference Contract

For STRUCTURAL scopes, the following must be present before implementation proceeds:

```
acc_id:           AND-ACC-YYYY-NNN
acc_scope:        STRUCTURAL
acc_storage_path: .github/governance/android/acc-registry-android.yaml
acc_status:       APPROVED
```

For LOCAL scopes, an ACC-lite must be declared in the implementation plan.

---

## 3. ACC Registration Protocol

### STRUCTURAL scope
1. Lead or Architect assigns `acc_id: AND-ACC-YYYY-NNN`
2. Entry added to `.github/governance/android/acc-registry-android.yaml` with `status: PENDING`
3. AndroidArchitectAgent reviews and issues verdict
4. Lead updates `status: APPROVED` before delegating to programmer
5. acc_id referenced in all subsequent agent inputs

### LOCAL scope
1. ACC-lite declared in implementation plan
2. No registry entry required
3. Programmer references ACC-lite in implementation
4. Test reviewer validates ACC-lite scope is genuinely LOCAL

---

## 4. Delegation Contracts

### Delegating to GitForensicsAgent (optional HistoricalContext)

Use only when one or more of these triggers apply:
- Legacy-heavy area or unclear historical rationale
- High churn, repeated hotfixes, or rollback-prone area
- Suspected similar precedent in prior branches or releases
- Ambiguity around an existing restriction, workaround, or dependency choice
- Need to compare two refs before architecture handoff

Provide:
```
- Target repo, module, file, or folder scope
- Commit window or base/head refs
- The specific question to answer: patterns, chronology, churn, rename signals, or likely rationale
```

Expect: neutral evidence report only.
HistoricalContext does not produce a verdict and does not block by itself.

### Delegating to AndroidArchitectAgent

Provide:
```
- acc_id (if known) or scope classification request
- Change description: modules affected, dependency changes, DI impact
- Gradle topology changes (if any)
- Local-vs-remote toggle change (yes/no)
- Context: which repo, module names, and existing analogous patterns
```

Expect: Architecture verdict (ACCEPTABLE/INSUFFICIENT) + ACC or ACC-lite reference.

### Delegating to AndroidProgrammerAgent via sdd-apply

When ACC is APPROVED, delegate implementation through the `sdd-apply` skill.

Provide to sdd-apply:
```
- change_name: <scope slug, e.g. biometric-android>
- artifact_store_mode: engram
- acc_id: AND-ACC-YYYY-NNN (STRUCTURAL) or ACC-lite scope (LOCAL)
- acc_scope: STRUCTURAL | LOCAL
- acc_storage_path: .github/governance/android/acc-registry-android.yaml (if STRUCTURAL)
- Target repo and Gradle module(s)
- Layer(s) affected: domain | infrastructure | presentation | app
- Feature description and expected behavior
- Existing analogous implementations to mirror
- Test requirements
- Platform constraints: android-arch-layers-skill invariants apply
```

sdd-apply will:
1. Retrieve ACC context from Engram: `mem_search("Android acc [acc_id]")`
2. Write all tasks to Engram under `sdd/{change-name}/tasks`
3. Implement code, save progress under `sdd/{change-name}/apply-progress`

Require: Implementation + tests + Fake for every new domain interface.

### Delegating to AndroidTestReviewerAgent via sdd-verify

After sdd-apply completes, delegate quality review through `sdd-verify`.

Provide to sdd-verify:
```
- change_name: <same scope slug>
- artifact_store_mode: engram
- acc_id or ACC-lite reference
- acc_scope: STRUCTURAL | LOCAL
- Specific behaviors to probe adversarially
```

sdd-verify retrieves apply-progress and spec from Engram, then delegates to AndroidTestReviewerAgent for chaos review.

Expect: Quality verdict (ACCEPTABLE/INSUFFICIENT) saved to `sdd/{change-name}/verify-report` in Engram.

---

## 5. Final Verdict Protocol

Lead issues the final verdict after all gates pass.

When verdict is APPROVED, invoke `sdd-archive`:
```
- change_name: <scope slug>
- artifact_store_mode: engram
```
sdd-archive reads all SDD artifacts from Engram, produces closure report under `sdd/{change-name}/archive-report`, and records all observation IDs for traceability.

After sdd-archive completes:
1. Write ACC entry to `.github/governance/android/acc-registry-android.yaml`
2. Append event to `.github/backlog/Android/lead-backlog.md`
3. Append to `.github/backlog/Android/decisions.md` for new architectural precedents

Verdict format:
```
## Lead Final Verdict

**Scope:** <description>
**acc_id:** AND-ACC-YYYY-NNN | ACC-lite
**Architecture verdict:** ACCEPTABLE (by AndroidArchitectAgent, <date>)
**Test quality verdict:** ACCEPTABLE (by AndroidTestReviewerAgent, <date>)
**SDD archive:** sdd/{change-name}/archive-report
**Final verdict:** APPROVED | REJECTED | BLOCKED

### Rationale
<Why this scope is approved, rejected, or blocked>

### Conditions (if REJECTED/BLOCKED)
<What must change or be resolved>
```

---

## 6. Blocking Conditions

Issue **BLOCKED** (not REJECTED) when:
- External dependency is unavailable (remote artifact not yet published)
- Another scope must be completed first and is not yet APPROVED
- Governance infrastructure is broken (validator crash, ACC registry corrupted)

Issue **REJECTED** when:
- Architecture verdict is INSUFFICIENT and programmer was given one revision opportunity
- Test quality verdict is INSUFFICIENT after revision
- Non-negotiable invariant was violated and programmer declined to fix

---

## 7. Engram Memory Protocol

Engram replaces ephemeral per-scope pipeline files. Git files are the permanent audit trail.

### Save to Engram on these events

| Event | mem_save content pattern |
|-------|---------------------------|
| Scope registered | `"Android scope [title] acc_id:PENDING status:PENDING [YYYY-MM-DD]"` |
| Architecture verdict received | `"Android acc [acc_id] arch-verdict:ACCEPTABLE scope:[STRUCTURAL/LOCAL] modules:[list]"` |
| Implementation complete | `"Android acc [acc_id] implementation:[modules] status:IMPLEMENTED"` |
| Chaos verdict received | `"Android acc [acc_id] chaos:ACCEPTABLE test-evidence:[summary]"` |
| Lead verdict issued | `"Android acc [acc_id] [APPROVED/REJECTED/BLOCKED] LeadFinalized [date]"` |

### Search Engram on these triggers

| Trigger | mem_search query |
|---------|------------------|
| Start of any task | `"Android active scope"` |
| Before delegating implementation | `"Android acc [acc_id]"` |
| On `/backlog` request | `"Android active"` |
| Resuming interrupted pipeline | `"Android acc [acc_id]"` |

### Mandatory Git persistence after APPROVED

1. Write ACC entry to `.github/governance/android/acc-registry-android.yaml`
2. Append governance event to `.github/backlog/Android/lead-backlog.md`
3. Append to `.github/backlog/Android/decisions.md` only when a new architectural precedent is set

### Fallback rule

If Engram is unavailable, read `lead-backlog.md` and the acc-registry YAML directly.
Never block delivery due to Engram unavailability.

---

## 8. ACC Scope Policy

| Scope | ACC required | Registry entry | Lead sign-off |
|-------|-------------|----------------|---------------|
| STRUCTURAL | AND-ACC-YYYY-NNN | YES | YES |
| LOCAL | ACC-lite | NO | Optional |

**STRUCTURAL triggers:**
- New cross-module dependency
- Gradle module added or removed
- Hilt component hierarchy change
- local-vs-remote toggle changed
- New multi-binding across module boundaries
- `domain` module gains any new external dependency
- Change affects ≥2 repos

If scope is disputed, AndroidArchitectAgent classifies — Lead ratifies.

---

## 9. Weekly Governance Check

Every week, Lead must:
1. Run `python3 .github/policies/android/android-governance-local-gate.py`
2. Review `.github/governance/android/reports/android-weekly-governance-summary.md`
3. Check blocked_pr_rate against threshold (≤ 0.15 to keep Stage active)
4. If threshold exceeded: initiate rollback to warning mode per enforcement rollout doc

---

## 10. Non-Delegable Authorities

Only LeadAndroidAgent may:
- Register a new `AND-ACC-YYYY-NNN` ID
- Change ACC status from PENDING to APPROVED or REJECTED
- Issue BLOCKED verdict
- Promote or rollback governance enforcement stages
- Approve Gradle module topology changes
- Approve changes to `android-architecture-core-skill` invariants
