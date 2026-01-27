# Extensible Objects PRD/ADR Review

## Review History
- **2026-01-29 (Initial)**: Initial review of Durable Objects PRD/ADR
- **2026-01-29 (Update)**: Renamed to Extensible Objects; major ADR enhancements; re-reviewed

## Scope
- Reviewed `modules/extensible_objects/docs/1_PRD.md` and `modules/extensible_objects/docs/2_ADR_EXTENSIBLE_OBJECTS.md`
- Checked `docs/` for consistency with the PRD/ADR

---

## Findings Status

### ✅ RESOLVED

#### 1. Tenant isolation now enforced in schema spec
**Original issue**: PRD/ADR require strict tenant isolation but do not define mandatory system fields or how scoping is enforced.

**Resolution**: ADR now includes detailed "Storage Considerations" section with:
- Required system columns: `id`, `tenant_id`, `version`, `created_at`, `created_by`, `updated_at`, `updated_by`, `deleted_at`
- Explicit requirement for `SecureConn` + `SecurityCtx` per operation
- `ScopableEntity` declaration with `tenant_col`, `resource_col`, `owner_col`, `type_col`

**References**: `2_ADR_EXTENSIBLE_OBJECTS.md` §Storage Considerations

#### 2. Optimistic concurrency now specified
**Original issue**: P1 requirement exists, but the ADR/API examples do not define version/etag usage.

**Resolution**: ADR now includes:
- `expected_version` parameter on `objects.update` and `objects.batch_update`
- `version` field in metadata response
- `version_conflict` error type with `expected_version` and `actual_version` in details
- System column `version INTEGER` in all tables

**References**: `2_ADR_EXTENSIBLE_OBJECTS.md` §Update Object, §Error Types, §Storage Considerations

#### 3. Hook contracts now fully specified
**Original issue**: Update/delete hooks may lack `object_id`, and update hooks do not require `previous_data`.

**Resolution**: ADR hook input schema now clearly includes:
- `object_id` as a field (already present)
- `previous_data` documented as "Only present for update events"
- Added `max_before_hook_timeout_ms` and `max_after_hook_timeout_ms` to traits
- Clear execution semantics for before_* (synchronous, can block) vs on_* (async, fire-and-forget)

**References**: `2_ADR_EXTENSIBLE_OBJECTS.md` §Hook Input Schema, §Hook Execution Semantics

#### 4. Secure ORM now explicitly tied to Extensible Objects
**Original issue**: `docs/SECURE-ORM.md` defines strict scoping patterns, but ADR does not explicitly require the storage layer to use Secure ORM.

**Resolution**: ADR now explicitly states:
- "Persistence MUST use `modkit-db` Secure ORM (`SecureConn`) with a request-scoped `SecurityCtx` per operation"
- "Unscoped queries MUST be impossible"
- "Raw database access is only allowed for migrations/admin tooling behind the `insecure-escape` feature"

**References**: `2_ADR_EXTENSIBLE_OBJECTS.md` §Tenant Isolation, §Security Considerations

---

### ⚠️ PARTIALLY RESOLVED

#### 5. Schema evolution strategy
**Original issue**: PRD calls for schema versioning, but ADR does not define migration/backfill rules.

**Current state**: ADR now includes:
- `types.update` API that only allows additive changes (new fields, new indexes)
- Field conversion rules in Storage Considerations (create new column → migrate data → drop old)
- Note that destructive changes require explicit migration

**Remaining gaps**:
- No explicit backfill API for populating new fields on existing records
- No index rebuild behavior defined for large tables
- No rollback mechanism defined

**References**: `2_ADR_EXTENSIBLE_OBJECTS.md` §Type Management API, §Field Conversion

#### 6. Field-level access control
**Original issue**: PRD makes field-level access control a requirement, while ADR positions it as "Future".

**Current state**:
- PRD (BR-EO-007) now says "field-level access control where needed (future consideration)"
- ADR lists it under "Future Considerations"
- Consistent but not implemented

**Decision needed**: Is field-level access control P1 or P2? Current docs treat it as P2/future.

---

### ❌ NOT YET RESOLVED

#### 7. Cursor pagination semantics missing
**Issue**: Cursor stability, ordering guarantees, and concurrency behavior are not defined.

**Impact**: Can lead to missing/duplicate results during pagination if data changes.

**Recommendation**: Add a "Pagination Semantics" subsection defining:
- Cursor encoding (opaque vs. keyset)
- Stability guarantees (snapshot vs. live)
- Behavior when underlying data changes mid-pagination
- Ordering requirements for cursor-based pagination

**References**: `2_ADR_EXTENSIBLE_OBJECTS.md` §Query Objects

#### 8. OData projection mapping not clarified
**Issue**: `docs/ODATA_SELECT.md` defines $select semantics for REST, but the ADR uses JSON-RPC with `query.select` and does not clarify the mapping.

**Recommendation**: Add a note clarifying that `query.filter`, `query.orderby`, `query.select` use the same grammar as OData $filter, $orderby, $select and reference the same validation/parsing code.

**References**: `docs/ODATA_SELECT.md`, `2_ADR_EXTENSIBLE_OBJECTS.md` §OData Query Semantics

---

## Docs Folder Consistency Check

### ❌ Still needs updates

1. **No Extensible Objects coverage in architecture/module docs**
   - `docs/MODULES.md` does not list an Extensible Objects module or its API surface
   - `docs/ARCHITECTURE_MANIFEST.md` mentions GTS extensibility but not the Extensible Objects capability

   **Recommendation**: Add entries for Extensible Objects in both documents.

2. **API surface not documented in central location**
   - The JSON-RPC endpoints (`/api/serverless-runtime/v1/types`, `/api/serverless-runtime/v1/objects`) are defined in the ADR but not in a central API reference.

   **Recommendation**: Add to API documentation or create OpenAPI spec.

---

## New Items from ADR Enhancements

The following new features were added and should be reviewed:

### Type Management API
- `types.register`, `types.get`, `types.list`, `types.update`, `types.delete`
- `types.extend`, `types.list_extensions`, `types.remove_extension`

**Review needed**: Confirm authorization model for type management operations (who can register types? who can extend?)

### Object Methods
- `objects.call` API for invoking attached serverless functions
- Methods defined in type schema with `function_id` reference

**Review needed**: Confirm method execution context and error handling.

### Batch Operations
- `objects.batch_create`, `objects.batch_update`, `objects.batch_delete`
- `on_error: "continue"|"abort"` modes

**Review needed**: Confirm transactional semantics for `abort` mode.

### Foreign Key Relationships
- `references` field in type schema
- `on_delete: restrict|cascade|set_null`
- `eager_load` option

**Review needed**: Confirm cascade delete behavior across tenant boundaries (should be blocked).

### Error Types
- Comprehensive error taxonomy with GTS schema
- 13 error codes defined

**Review needed**: Confirm error codes align with platform conventions.

---

## Summary

| Category | Resolved | Partial | Open |
|----------|----------|---------|------|
| Security/Isolation | 2 | 0 | 0 |
| API Completeness | 2 | 1 | 2 |
| Documentation | 1 | 0 | 2 |
| **Total** | **5** | **1** | **4** |

**Next steps**:
1. Define cursor pagination semantics
2. Clarify OData/JSON-RPC mapping
3. Update `docs/MODULES.md` and `docs/ARCHITECTURE_MANIFEST.md`
4. Review new features (Type Management, Methods, Batch Ops, References, Errors)
