# Targeted codebase improvement notes (controller-focused)

This document records a focused audit of controller paths that are high traffic and operationally sensitive. It is intentionally written in a Kubernetes-style engineering note format so future follow-up PRs can pick up items directly.

## Context

During a quick pass across controller code, we looked for patterns that typically cause production pain over time:
- request context not being propagated;
- retry loops without explicit cancellation/time bounds;
- stale compatibility comments that no longer match API reality;
- reconciliation paths doing full work when targeted work is possible.

## How this was reviewed

1. Repo-wide pattern scans (`context.TODO`, `TODO/FIXME/XXX`) to identify hotspots.
2. Manual read-through of selected files in `pkg/controller/*` and `staging/src/k8s.io/controller-manager/*`.
3. Prioritization by expected operator impact and implementation risk.

## Findings and recommended follow-ups

### 1. StatefulSet: incomplete context propagation in object manager writes
**Area:** `pkg/controller/statefulset`

**Observed**
- `StatefulPodControlObjectManager` takes context for `CreatePod`, but not for `UpdatePod`, `DeletePod`, `CreateClaim`, or `UpdateClaim`.
- The concrete implementation uses `context.TODO()` for those write operations.

**Why this matters**
- On shutdown, API-server slowness, or cascading retries, these calls cannot participate in caller cancellation/deadline handling.

**Suggested follow-up PR**
- Thread `ctx context.Context` through all write methods in the interface and implementations.
- Plumb `ctx` from existing higher-level call sites (`UpdateStatefulPod`, etc.).

**Priority**: P0

---

### 2. Dynamic client builder: token/SA calls are not context-bounded; clock injection is underused
**Area:** `staging/src/k8s.io/controller-manager/pkg/clientbuilder`

**Observed**
- Token and SA bootstrap operations call API methods with `context.TODO()`.
- `DynamicControllerClientBuilder` carries an injected clock field, but token-expiry calculations still use `time.Now()` directly.

**Why this matters**
- Under API disruption, token-fetch retries can behave less predictably without explicit cancellation.
- Direct wall-clock calls make skew/rotation behavior harder to test deterministically.

**Suggested follow-up PR**
- Use a context with timeout for token and service-account bootstrap paths.
- Pass injected clock into token source and remove direct `time.Now()` in expiry calculations.

**Priority**: P0

---

### 3. Dynamic client builder: error wrapping can preserve richer error chains
**Area:** `staging/src/k8s.io/controller-manager/pkg/clientbuilder`

**Observed**
- Some wrapping uses `%v` where `%w` would preserve unwrap semantics.

**Why this matters**
- Better `errors.Is` / `errors.As` behavior improves debugging and future refactors.

**Suggested follow-up PR**
- Update wrapped returns to use `%w` where appropriate.
- Add unit assertions for unwrapping behavior where practical.

**Priority**: P1

---

### 4. ResourceQuota: replenishment path still enqueues full recalculation
**Area:** `pkg/controller/resourcequota`

**Observed**
- Replenishment currently enqueues quota recalculation broadly when resources intersect, with a TODO noting lack of targeted behavior.

**Why this matters**
- In large namespaces/high churn, full recalculation can add avoidable controller and API pressure.

**Suggested follow-up PR**
- Carry triggering resource metadata through queue items.
- Allow evaluation path to skip unaffected evaluators/resources.

**Priority**: P1

---

### 5. Deployment rollback helpers: TODO wording appears stale
**Area:** `pkg/controller/deployment`

**Observed**
- Comments still reference removal conditions tied to API versions removed long ago.

**Why this matters**
- Stale TODOs reduce signal quality for maintainers and reviewers.

**Suggested follow-up PR**
- Replace TODO wording with explicit current compatibility rationale (or link to active tracking issue if removal is still planned).

**Priority**: P2

## Proposed execution order

1. **P0**: StatefulSet context propagation.
2. **P0**: Dynamic builder context/time-bound token flow + clock cleanup.
3. **P1**: Error wrapping consistency (`%w`).
4. **P1**: ResourceQuota targeted replenishment design and incremental implementation.
5. **P2**: Rollback TODO refresh.

## Validation guidance for implementation PRs

- Unit tests covering context cancellation for StatefulSet pod/PVC write calls.
- Unit tests for token expiry/rotation math using fake clock.
- Error chain tests (`errors.Is/As`) where wrapping changes are introduced.
- Benchmark or pprof comparison for ResourceQuota sync behavior before/after targeted replenishment.

## Non-goals for this document

- This note does **not** change behavior.
- It is not a KEP; it is a focused engineering backlog artifact to make follow-up PRs faster and better scoped.
