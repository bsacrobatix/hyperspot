# Review: Serverless Runtime Proposals — Consistency and Completeness

## Status
Review Completed: 2026-01-26

## Executive Summary

The Serverless Runtime proposals (PRD + 3 ADRs) provide a solid foundation with good alignment between business requirements and technical design. The GTS schema-based approach is consistent, tenant isolation is well-emphasized, and the unified invocation API is well-designed.

However, there are **critical gaps** where P0/P1 requirements from the PRD are not addressed in any ADR, and several **consistency issues** where concepts are mentioned but not fully specified or aligned across documents.

**Overall Assessment**: The proposals are ~80% complete and implementation-ready pending resolution of the gaps identified below.

---

## Critical Gaps (Missing Requirements)

### 1. Scheduling (BR-023, BR-110) - P0/P1 ⚠️

**Issue**: The PRD extensively covers schedule-based triggers, missed schedule handling, and schedule-level input parameters, but **none of the ADRs address the scheduling mechanism**.

**Requirements Not Addressed**:
- BR-023 (P0): Schedule lifecycle (create, update, pause/resume, delete)
- BR-110 (P1): Schedule-level input parameters and overrides
- Missed schedule handling policy
- Recurring execution patterns (cron expressions, intervals)

**Recommendation**: Create `6_ADR_SCHEDULING.md` or add a scheduling section to `2_ADR_FUNCTION_ENTRYPOINT.md`.

**Required Design Elements**:
```json
// Schedule schema
{
  "schedule_id": "...",
  "function_id": "gts.x.core.faas.func.v1~...",
  "cron": "0 0 * * *",
  "enabled": true,
  "params": {...},
  "missed_schedule_policy": "skip | run_immediately | run_once_catchup"
}

// Schedule API
POST /api/serverless-runtime/v1/schedules
GET /api/serverless-runtime/v1/schedules/{schedule_id}
PUT /api/serverless-runtime/v1/schedules/{schedule_id}
POST /api/serverless-runtime/v1/schedules/{schedule_id}:pause
POST /api/serverless-runtime/v1/schedules/{schedule_id}:resume
DELETE /api/serverless-runtime/v1/schedules/{schedule_id}
```

---

### 2. Hot Module Reloading (BR-137) - P1 ⚠️

**Issue**: The PRD requires loading native modules at runtime and invoking them via the common calling mechanism. This is not addressed in any ADR.

**Requirements Not Addressed**:
- BR-137 (P1): Hot (re)loading native modules for development or worker-process cases
- How native Rust modules register with the runtime
- How Starlark code invokes native modules
- Versioning and compatibility for native modules

**Recommendation**: Add a section to `3_ADR_STARLARK_RUNTIME.md` explaining the hot module mechanism.

**Required Design Elements**:
- Module registration API (how native modules expose functions)
- Dynamic loading mechanism (dlopen, plugin architecture)
- Module namespace and versioning
- Starlark invocation syntax (e.g., `r_native_invoke_v1(module, function, params)`)
- Security and isolation considerations for native code

---

### 3. Static Traits for Dynamic Functions (BR-138) - P1 ⚠️

**Issue**: The PRD requires application developers to define reusable function interfaces (traits) that bind to dynamic functions at runtime. Not addressed.

**Requirements Not Addressed**:
- BR-138 (P1): Mechanism to define a reusable function interface
- How dynamic implementations bind to traits
- Validation rules for trait compliance
- Use cases (e.g., plugin systems, extensibility points)

**Recommendation**: Add a section to `2_ADR_FUNCTION_ENTRYPOINT.md` defining function traits.

**Conceptual Design**:
```json
// Trait definition (interface)
{
  "$id": "gts://gts.x.core.faas.trait.v1~vendor.app.plugin.payment_processor.v1~",
  "params": {"type": "object", "properties": {...}},
  "returns": {"type": "object", "properties": {...}},
  "errors": [...]
}

// Implementation binding
{
  "function_id": "gts.x.core.faas.func.v1~vendor.stripe.process_payment.v1~",
  "implements": ["gts.x.core.faas.trait.v1~vendor.app.plugin.payment_processor.v1~"]
}

// Runtime validation: params/returns/errors MUST be compatible with trait
```

---

### 4. Child Workflows (BR-104) - P1 ⚠️

**Issue**: The PRD requires invoking child workflows from parent workflows. Not explicitly addressed in Starlark runtime.

**Requirements Not Addressed**:
- BR-104 (P1): Invoking child workflows/functions from a parent workflow
- Parent-child relationship tracking
- How child workflow failures affect parents
- Lifecycle management (cancel parent cancels children?)

**Recommendation**: Add to `3_ADR_STARLARK_RUNTIME.md`.

**Required Design Elements**:
```python
# Invoke another function/workflow synchronously
result = r_await(r_invoke_v1(
    "gts.x.core.faas.func.v1~vendor.app.billing.calculate_tax.v1~",
    params={"invoice_total": 100, "region": "US-CA"},
    mode="sync",
    exit_on_error=True
))

# Invoke a child workflow asynchronously
job = r_await(r_invoke_v1(
    "gts.x.core.faas.func.v1~vendor.app.provisioning.provision_tenant.v1~",
    params={"tenant_id": "t-123"},
    mode="async",
    exit_on_error=False
))
job_id = job.job_id

# Poll child workflow status later
status = r_await(r_get_job_status_v1(job_id))
```

**Open Questions**:
- Are child executions tracked in parent execution timeline?
- If parent is canceled, are children automatically canceled?
- If parent is paused, are children paused?

---

### 5. Parallel Execution (BR-105) - P1 ⚠️

**Issue**: The Starlark ADR mentions `r_await_all([p1, p2, ...])` for parallel promise resolution but doesn't detail parallel step execution with concurrency controls.

**Requirements Not Addressed**:
- BR-105 (P1): Parallel execution of independent steps with controllable concurrency caps
- How errors in parallel branches are handled
- Checkpoint/resume semantics for parallel execution
- Compensation for partial parallel execution failures

**Recommendation**: Expand `3_ADR_STARLARK_RUNTIME.md` with parallel execution design.

**Required Design Elements**:
```python
# Simple parallel promise resolution
results = r_await_all([p1, p2, p3])

# Parallel step execution with concurrency control
steps = [
    ("fetch_customer", fetch_customer_step, {"customer_id": "c1"}),
    ("fetch_orders", fetch_orders_step, {"customer_id": "c1"}),
    ("fetch_preferences", fetch_preferences_step, {"customer_id": "c1"}),
]
results = r_await(r_run_steps_parallel_v1(
    steps,
    max_concurrency=2,  # max 2 concurrent
    exit_on_first_error=True
))

# Results: list of StepResult structs
for result in results:
    if result.ok:
        print(result.value)
    else:
        print(result.error)
```

**Semantics**:
- `exit_on_first_error=True`: First error stops remaining steps and triggers compensation
- `exit_on_first_error=False`: All steps complete; caller handles errors
- Compensation for parallel steps: execute in reverse order of completion

---

## Consistency Issues

### 1. Security Context Structure ⚠️

**Issue**: Security context is mentioned throughout but never formally defined.

**Where Mentioned**:
- PRD BR-006: Execution identity context (system/API client/user)
- PRD BR-025: Security context must be available throughout execution
- ADR #2: Mentions security context without defining structure
- ADR #3: Has `ctx` parameter but doesn't define properties
- ADR #4: Shows `actor` in hook input but not linked to `ctx`

**Recommendation**: Add to `2_ADR_FUNCTION_ENTRYPOINT.md` a formal context definition.

**Proposed Structure**:
```json
{
  "ctx": {
    "execution_id": "exec-abc123",
    "correlation_id": "corr-xyz789",
    "trace_id": "trace-123",
    "tenant_id": "tenant-456",
    "actor": {
      "type": "system | api_client | user",
      "id": "actor-id",
      "token": "..." // for long-running credential refresh (BR-014)
    },
    "security": {
      "permissions": ["workflow.execute", "object.read"],
      "scopes": ["tenant:456", "resource:*"]
    },
    "invocation": {
      "mode": "sync | async",
      "idempotency_key": "...",
      "client_request_id": "..."
    }
  }
}
```

**Starlark Access**:
```python
def main(ctx, input):
    tenant_id = ctx.tenant_id
    actor_type = ctx.actor.type
    is_system = actor_type == "system"
```

---

### 2. Caching Semantics - Conflated Concepts ⚠️

**Issue**: Client-side caching and server-side result caching are conflated.

**Current State**:
- ADR #2: `traits.caching.max_age_seconds` described as **client-side** caching (TTL hint)
- PRD BR-118 (P1): Describes **server-side** execution result caching for idempotent operations

**These are different concepts**:
- **Client-side**: HTTP Cache-Control headers for response reuse by clients
- **Server-side**: Idempotency-based result deduplication within the runtime

**Recommendation**: Clarify and separate the two concepts.

**Proposed Clarification**:
```json
"traits": {
  // Client-side caching: TTL hint for clients to reuse results
  "caching": {
    "max_age_seconds": 60,
    "description": "Clients MAY cache results for this duration. Scoped by tenant, caller, headers, and params."
  },

  // Server-side result caching: runtime deduplication via idempotency
  "idempotency": {
    "enabled": true,
    "cache_window_seconds": 300,
    "description": "Runtime caches results by idempotency key for this duration to deduplicate retries."
  }
}
```

---

### 3. Retry Policies - Incomplete ⚠️

**Issue**: Function/workflow-level retry configuration is incomplete.

**Current State**:
- PRD BR-020 (P0): Configurable retry and failure-handling policies
- ADR #2: Shows retry in HTTP helpers (`r_http_get_v1(..., retry=...)`)
- **Missing**: Function/workflow-level retry configuration

**Recommendation**: Add to `2_ADR_FUNCTION_ENTRYPOINT.md` entrypoint traits.

**Proposed Design**:
```json
"traits": {
  "retry": {
    "enabled": true,
    "max_attempts": 3,
    "backoff_strategy": "exponential | linear | constant",
    "initial_backoff_ms": 200,
    "max_backoff_ms": 5000,
    "backoff_multiplier": 2.0,
    "retryable_errors": [
      "gts.x.core.faas.err.v1~x.core._.runtime.v1~x.core._.timeout.v1~",
      "gts.x.core.faas.err.v1~x.core._.http.v1~" // 5xx only
    ],
    "non_retryable_errors": [
      "gts.x.core.faas.err.v1~x.core._.code.v1~",
      "gts.x.core.faas.err.v1~x.core._.http.v1~" // 4xx only
    ]
  }
}
```

**Runtime Behavior**:
- Applies to top-level function/workflow execution (not individual HTTP calls)
- Retries create new execution attempts with same `execution_id` and `idempotency_key`
- All attempts are visible in execution timeline

---

### 4. Compensation Trigger - Ambiguous ⚠️

**Issue**: Compensation trigger conditions are not fully specified.

**Current State**:
- ADR #3: States `r_exit(error)` triggers compensation
- **Unclear**: Do runtime aborts (timeout, OOM) trigger compensation?
- **Unclear**: Does explicit API cancellation trigger compensation?

**Recommendation**: Clarify in `3_ADR_STARLARK_RUNTIME.md`.

**Proposed Clarification**:

> **Compensation is triggered when**:
> 1. Workflow explicitly calls `r_exit(error)` due to business logic failure
> 2. Workflow is canceled via API (`:cancel` endpoint)
> 3. Runtime aborts workflow due to timeout (BR-029 guardrail)
> 4. Runtime aborts workflow due to memory/CPU limit exceeded
> 5. Runtime aborts workflow due to unhandled code error (uncaught exception)
>
> **Compensation is NOT triggered when**:
> 1. Workflow completes successfully (returns normally)
> 2. Workflow is manually paused via API (`:pause` endpoint)
> 3. Workflow is suspended waiting for event/timer (normal wait state)
>
> **Compensation Execution**:
> - Compensations execute in **reverse order** of step completion
> - Compensation failures are logged but do NOT stop remaining compensations
> - Compensation failures are surfaced in workflow final status
> - Compensations have independent retry policy and timeout

---

## Completeness Issues

### 1. Starlark Helper Versioning - Underspecified

**Issue**: ADR #3 states runtime helpers are versioned (`r_http_get_v1`) but doesn't explain versioning policy.

**Recommendation**: Add versioning section to `3_ADR_STARLARK_RUNTIME.md`.

**Proposed Policy**:

> **Helper Versioning Policy**:
> - New helper versions are introduced as `r_<name>_v2`, `r_<name>_v3`, etc.
> - Old versions remain available **indefinitely** for backward compatibility
> - Deprecated versions emit **warnings** during function validation
> - Function definitions explicitly reference helper versions used (validated during registration)
> - Breaking changes REQUIRE a new version number
> - Non-breaking additions MAY be added to existing versions (with caution)
>
> **Validation**:
> - Runtime validates all `r_*` calls during function registration
> - Unknown helper names/versions cause registration failure
> - Runtime tracks helper dependencies per function for compatibility checking

---

### 2. Replay/Resume Mechanism - Missing Details

**Issue**: ADR #3 describes snapshots conceptually but doesn't specify internals.

**Missing Details**:
- What exactly is captured in a snapshot?
- How are snapshots stored/retrieved?
- How does deterministic replay work with `r_now_v1()` and `r_rand_v1()`?

**Recommendation**: Add snapshot internals section to `3_ADR_STARLARK_RUNTIME.md`.

**Proposed Specification**:

> **Snapshot Contents**:
> - Call stack frames and instruction pointers
> - Local variable bindings (JSON-serializable subset)
> - Completed step IDs and their outputs
> - Registered compensation actions (function refs + inputs)
> - Deterministic value cache (timestamps, random values keyed by label)
> - Promise states (pending, resolved, rejected)
> - Event subscription registrations
>
> **Deterministic Replay**:
> - `r_now_v1()`: Returns cached timestamp from snapshot
> - `r_rand_v1(label)`: Returns cached random value keyed by label
> - HTTP responses: Replayed from recorded responses (not re-executed)
> - External events: Replayed from recorded events
>
> **Storage**:
> - Snapshots stored in `workflow_snapshots` table
> - Compressed and encrypted at rest
> - Retention aligned with execution history retention (BR-107, BR-126)
> - Indexed by `execution_id` and `snapshot_label`

---

### 3. Resource Governance Enforcement - Missing

**Issue**: PRD BR-005 requires CPU/memory/network limits, but ADR #3 doesn't specify enforcement mechanism.

**Recommendation**: Add enforcement section to `3_ADR_STARLARK_RUNTIME.md`.

**Proposed Enforcement**:

> **Resource Limit Enforcement**:
>
> **Memory Limits**:
> - Custom Starlark allocator tracks total allocation per execution
> - If allocation exceeds `default_memory_mb` trait, runtime aborts with `memory_limit` error
> - Snapshot memory counted separately (does not count against execution limit)
>
> **CPU Limits**:
> - Starlark instruction counting via step budget (configurable steps per ms)
> - If execution exceeds `default_cpu` trait, runtime aborts with `cpu_limit` error
> - Compensation steps have independent CPU budget
>
> **Network Limits**:
> - All HTTP calls proxied via outbound API gateway
> - Gateway tracks request count and bytes per execution
> - If limits exceeded, gateway returns `429 Too Many Requests`
> - Runtime converts to `http_rate_limit` error
>
> **Time Limits**:
> - Wall-clock timeout via tokio `timeout()` wrapper
> - If execution exceeds `default_timeout_seconds`, runtime aborts with `timeout` error
> - Long-running workflows: timeout applies per active execution segment (between suspensions)

---

### 4. Durable Object Versioning - Trait Without Mechanism

**Issue**: ADR #4 includes `traits.versioning: boolean` but doesn't specify how versioning works.

**Recommendation**: If versioning is P1, add versioning API to `4_ADR_DURABLE_OBJECTS.md`.

**Proposed API**:
```json
// Get specific version
{
  "method": "objects.get_version",
  "params": {
    "type_id": "...",
    "object_id": "...",
    "version": 3
  }
}

// List versions
{
  "method": "objects.list_versions",
  "params": {
    "type_id": "...",
    "object_id": "..."
  }
}

// Version history response
{
  "versions": [
    {"version": 1, "timestamp": "...", "actor": {...}},
    {"version": 2, "timestamp": "...", "actor": {...}},
    {"version": 3, "timestamp": "...", "actor": {...}}
  ]
}
```

**Storage**:
- Versions stored in `durable_object_versions` table
- Each update creates new version row
- Current version tracked in main `durable_objects` table
- Soft deletes create tombstone version

---

### 5. Quota Enforcement - Requirements Without Design

**Issue**: PRD has extensive quota requirements (BR-013, BR-106, BR-116) but no ADR addresses enforcement.

**Requirements**:
- BR-013 (P0): Per-function/workflow resource quotas
- BR-106 (P1): Per-tenant resource quotas
- BR-116 (P1): Execution rate limits and volume caps

**Recommendation**: Add quota enforcement section to `2_ADR_FUNCTION_ENTRYPOINT.md`.

**Proposed Design**:

> **Quota Configuration**:
> ```json
> {
>   "tenant_id": "tenant-456",
>   "quotas": {
>     "max_concurrent_executions": 100,
>     "max_function_definitions": 1000,
>     "max_executions_per_minute": 1000,
>     "max_executions_per_day": 100000,
>     "execution_history_retention_days": 7,
>     "max_execution_history_size_mb": 1000
>   }
> }
>
> {
>   "function_id": "...",
>   "quotas": {
>     "max_concurrent_executions": 10,
>     "max_memory_mb": 512,
>     "max_cpu": 1.0,
>     "max_timeout_seconds": 300
>   }
> }
> ```
>
> **Enforcement**:
> - Quotas stored in `tenant_quotas` and `function_quotas` tables
> - Runtime checks quotas before starting new executions
> - If quota exceeded, returns `quota_exceeded` error with details
> - Rate limits use token bucket algorithm per tenant/function
> - Concurrent execution count tracked in Redis for fast access

---

### 6. Step-Through Debugging API - Mentioned But Not Specified

**Issue**: ADR #3 mentions breakpoints and step-through in "Why Starlark?" but doesn't define the API.

**Recommendation**: Add debug control API to `2_ADR_FUNCTION_ENTRYPOINT.md` or `3_ADR_STARLARK_RUNTIME.md`.

**Proposed API**:
```
POST /api/serverless-runtime/v1/executions/{id}/breakpoints
  Body: {"locations": [{"line": 12}, {"line": 45}]}

GET /api/serverless-runtime/v1/executions/{id}/breakpoints

DELETE /api/serverless-runtime/v1/executions/{id}/breakpoints/{breakpoint_id}

POST /api/serverless-runtime/v1/executions/{id}:step
  Body: {"action": "step_over | step_into | step_out | continue"}

GET /api/serverless-runtime/v1/executions/{id}/locals
  Response: {"frame": 0, "locals": {"x": 123, "y": "abc"}}

GET /api/serverless-runtime/v1/executions/{id}/stack
  Response: {"frames": [...]}
```

**Semantics**:
- Breakpoints apply to paused/suspended executions only
- When execution hits breakpoint, status becomes `paused` with reason `breakpoint`
- Step commands resume execution for one step then pause again
- Debugging requires special permission (BR-127)

---

### 7. Throttling/Rate Limiting - Underspecified

**Issue**: PRD BR-113, BR-116 require throttling but mechanism is not detailed.

**Current State**:
- ADR #2: Mentions returning `429 Too Many Requests` but doesn't detail mechanism

**Recommendation**: Add rate limiting section to `2_ADR_FUNCTION_ENTRYPOINT.md`.

**Proposed Design**:

> **Rate Limiting Algorithm**:
> - Token bucket algorithm with configurable rate and burst
> - Separate buckets per tenant, per function, and per user
> - Rate limits configured via `tenant_rate_limits` and `function_rate_limits` tables
>
> **Configuration**:
> ```json
> {
>   "tenant_id": "tenant-456",
>   "rate_limits": {
>     "requests_per_second": 100,
>     "burst_size": 200
>   }
> }
> ```
>
> **Enforcement**:
> - Runtime checks rate limit before accepting invocation
> - If limit exceeded, returns `429 Too Many Requests` with:
>   - `Retry-After: <seconds>` header
>   - Error body: `{"error": "rate_limit_exceeded", "retry_after_seconds": 5}`
> - Rate limit state stored in Redis for distributed enforcement
>
> **Retry-After Calculation**:
> - `retry_after = (tokens_needed - tokens_available) / refill_rate`
> - Rounded up to nearest second

---

## Minor Issues

### 1. Cross-References

**Issue**: ADR #4 references external docs that may not exist.

**Examples**:
- `docs/ODATA_SELECT.md` (referenced for query semantics)
- `docs/SECURE-ORM.md` (referenced for SecureConn)

**Recommendation**: Either inline the relevant documentation or note "See external docs: [link]".

---

### 2. Durable Object Hook Error Handling Edge Cases

**Issue**: ADR #4 doesn't specify edge cases for hook failures.

**Missing Specifications**:
- What if `on_before_*` hook returns `{allow: false}` but provides no error?
- What if `on_before_*` hook times out?
- What if multiple hooks are registered (future extensibility)?

**Recommendation**: Clarify edge cases in `4_ADR_DURABLE_OBJECTS.md`.

**Proposed Clarification**:

> **Hook Edge Cases**:
> - If `allow: false` without error, runtime returns generic "operation_denied" error
> - If hook times out (exceeds hook timeout limit), operation aborts with "hook_timeout" error
> - If hook raises unhandled exception, operation aborts with "hook_error" error
> - Multiple hooks per event type are not currently supported; only one hook per event
> - Future: multiple hooks may execute in registration order with short-circuit on first error

---

### 3. Durable Object Index Implementation

**Issue**: ADR #4 defines index types ("exact", "prefix", "fulltext", "numeric", "datetime") but not how they're implemented without knowing the storage backend.

**Recommendation**: Add implementation note to `4_ADR_DURABLE_OBJECTS.md`.

**Proposed Note**:

> **Index Implementation**:
> Storage backend implementations MUST support these index types or emulate them.
>
> **Reference Implementations**:
> - **PostgreSQL**:
>   - `exact`: B-tree index
>   - `prefix`: GIN index with pg_trgm extension
>   - `fulltext`: GIN index with tsvector
>   - `numeric`: B-tree index
>   - `datetime`: B-tree index
> - **SQLite**:
>   - `exact`: Standard index
>   - `prefix`: Manual LIKE query optimization
>   - `fulltext`: FTS5 virtual table
>   - `numeric`: Standard index
>   - `datetime`: Standard index

---

### 4. Error Taxonomy Completeness

**Issue**: ADR #2 defines many error types but some runtime scenarios aren't covered.

**Missing Error Types**:
- Invalid input schema validation failure
- Permission denied
- Quota exceeded
- Idempotency key conflict
- Function/workflow definition not found

**Recommendation**: Add missing error types to `2_ADR_FUNCTION_ENTRYPOINT.md`.

**Proposed Additions**:
```json
gts.x.core.faas.err.v1~x.core._.validation.v1~
gts.x.core.faas.err.v1~x.core._.permission_denied.v1~
gts.x.core.faas.err.v1~x.core._.quota_exceeded.v1~
gts.x.core.faas.err.v1~x.core._.idempotency_conflict.v1~
gts.x.core.faas.err.v1~x.core._.not_found.v1~
gts.x.core.faas.err.v1~x.core._.conflict.v1~
```

---

## Positive Observations ✅

1. **GTS Schema Consistency**: All ADRs use GTS schemas consistently with `$id` ending in `~`
2. **Unified Invocation API**: JSON-RPC 2.0 for both functions and durable objects provides consistency
3. **Tenant Isolation**: Consistently emphasized across all ADRs with clear scoping
4. **Compensation Support**: Well-designed saga pattern with automatic compensation on failure
5. **Observability**: Good coverage of correlation IDs, trace IDs, and execution metrics
6. **Strong Typing**: Schema validation at boundaries is well-specified
7. **Versioning**: Function/workflow definitions are versioned types with clear evolution path
8. **Async Model**: Promise-based async with `r_await` is clean and familiar
9. **Error Handling**: Structured error envelopes with hierarchical error types
10. **Lifecycle Hooks**: Durable object hooks provide good extensibility points

---

## Recommendations Summary

### High Priority (P0/P1 Requirements - Must Address)

1. **Add Scheduling Design** (BR-023, BR-110)
   - Create `6_ADR_SCHEDULING.md` or add section to `2_ADR_FUNCTION_ENTRYPOINT.md`
   - Define schedule schema, API, and missed schedule handling

2. **Address Hot Module Reloading** (BR-137)
   - Add section to `3_ADR_STARLARK_RUNTIME.md`
   - Define native module loading, registration, and invocation

3. **Define Static Traits Mechanism** (BR-138)
   - Add section to `2_ADR_FUNCTION_ENTRYPOINT.md`
   - Define trait schemas and implementation binding

4. **Add Child Workflow Invocation** (BR-104)
   - Add `r_invoke_v1()` helper to `3_ADR_STARLARK_RUNTIME.md`
   - Define parent-child relationship and lifecycle

5. **Expand Parallel Execution Design** (BR-105)
   - Add `r_run_steps_parallel_v1()` to `3_ADR_STARLARK_RUNTIME.md`
   - Define error handling and compensation for parallel steps

### Medium Priority (Consistency - Should Address)

6. **Define Security Context Structure**
   - Add formal `ctx` structure to `2_ADR_FUNCTION_ENTRYPOINT.md`
   - Include actor, permissions, scopes, and token for credential refresh

7. **Clarify Server-Side vs Client-Side Caching**
   - Separate `traits.caching` (client TTL) from `traits.idempotency` (server deduplication)
   - Update `2_ADR_FUNCTION_ENTRYPOINT.md`

8. **Add Function/Workflow-Level Retry Configuration** (BR-020)
   - Add `traits.retry` to entrypoint schema
   - Define retry policy with backoff and retryable errors

9. **Clarify Compensation Trigger Conditions**
   - Update `3_ADR_STARLARK_RUNTIME.md` with explicit trigger/non-trigger conditions
   - Define compensation execution semantics

### Low Priority (Completeness - Nice to Have)

10. **Specify Helper Versioning Policy**
    - Add versioning section to `3_ADR_STARLARK_RUNTIME.md`
    - Define backward compatibility guarantees

11. **Detail Snapshot/Replay Mechanism**
    - Add snapshot internals section to `3_ADR_STARLARK_RUNTIME.md`
    - Explain deterministic replay for `r_now_v1()` and `r_rand_v1()`

12. **Add Resource Governance Enforcement Details**
    - Add enforcement section to `3_ADR_STARLARK_RUNTIME.md`
    - Explain memory, CPU, network, and time limit enforcement

13. **Specify Quota Enforcement Mechanism** (BR-013, BR-106, BR-116)
    - Add quota section to `2_ADR_FUNCTION_ENTRYPOINT.md`
    - Define quota storage, checking, and error handling

14. **Define Debugging Control API** (BR-101, BR-102)
    - Add debug API to `2_ADR_FUNCTION_ENTRYPOINT.md`
    - Define breakpoints, step commands, and local inspection

15. **Expand Rate Limiting Design** (BR-113, BR-116)
    - Add rate limiting section to `2_ADR_FUNCTION_ENTRYPOINT.md`
    - Define token bucket algorithm and Retry-After calculation

16. **Clarify Durable Object Versioning** (if P1)
    - Add versioning API to `4_ADR_DURABLE_OBJECTS.md`
    - Define version storage and retrieval

17. **Document Hook Edge Cases**
    - Add edge case handling to `4_ADR_DURABLE_OBJECTS.md`
    - Clarify timeout, missing error, and multiple hook scenarios

18. **Complete Error Taxonomy**
    - Add missing error types to `2_ADR_FUNCTION_ENTRYPOINT.md`
    - Cover validation, permission, quota, and not_found errors

---

## Implementation Readiness

**Overall Assessment**: ~80% ready for implementation pending resolution of gaps.

**Blocking Issues** (Must Resolve Before Implementation):
- Scheduling design (P0 requirement)
- Security context structure (used everywhere)
- Compensation trigger conditions (affects workflow semantics)

**Important Issues** (Should Resolve Soon):
- Hot module reloading (P1 requirement)
- Static traits (P1 requirement)
- Child workflow invocation (P1 requirement)
- Parallel execution details (P1 requirement)
- Retry policies (P0 requirement)

**Nice-to-Have Issues** (Can Defer):
- Debugging API details
- Rate limiting internals
- Durable object versioning (if not P1)
- Hook edge cases

---

## Next Steps

1. **Prioritize High Priority Gaps**: Focus on scheduling, hot module reloading, static traits, child workflows, and parallel execution
2. **Resolve Consistency Issues**: Define security context, clarify caching, add retry policies
3. **Review and Approve**: Circulate updated ADRs for stakeholder review
4. **Begin Implementation**: Start with core entrypoint and Starlark runtime
5. **Iterate**: Add missing features incrementally based on priority
