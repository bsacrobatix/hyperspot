# PRD — Extensible Objects (Business Requirements)

## Purpose
Provide a platform capability for defining, storing, and managing extensible objects with dynamic schemas, secure access control, indexed search, and lifecycle hooks.

## Background / Problem Statement
The platform requires a mechanism for applications, tenants, and users to define and manage persistent data objects at runtime without requiring schema changes at the platform level. These objects need to support rich querying, lifecycle automation, and strong access controls.

A key differentiator is **runtime extensibility**: base object types can be extended by tenants with custom fields (similar to Jira custom fields) that become real SQL columns—queryable, sortable, and indexable—not just JSON blobs.

This PRD defines the business requirements for Extensible Objects that support:
- dynamic schema definition using GTS format
- runtime schema extension with typed custom fields
- storage and retrieval APIs with appropriate indexing
- search capability across object instances
- hook functions for lifecycle events (create, update, delete)
- methods attached to object types for custom business logic
- secure access control by specification

## Goals (Business Outcomes)
- Enable flexible, tenant-specific data modeling without platform rebuilds.
- Allow tenants to extend base schemas with custom typed fields at runtime.
- Provide rich querying and search capabilities for custom objects.
- Support automation through lifecycle hooks and object methods.
- Maintain compliance posture with auditability and strict tenant isolation.

## Stakeholders / Users
- **Platform service teams**: define and manage system-level extensible objects.
- **Integration vendors**: provide custom object schemas for integrations.
- **Tenant administrators**: manage object schemas, custom field extensions, access controls, and lifecycle hooks.
- **End users**: interact with extensible objects through applications.

## Scope
### In Scope
- Extensible objects with dynamic schemas deployed at runtime.
- Schema definition in GTS format with JSON Schema.
- Runtime schema extension (custom fields) by tenants.
- Storage in real SQL tables with typed columns (not JSON blobs).
- Storage and retrieval APIs (CRUD + batch operations).
- Indexed search capability with configurable index types.
- Hook functions for lifecycle events (create, update, delete).
- Methods attached to object types (serverless functions).
- Foreign key relationships between object types.
- Secure access control by specification.
- Change log for auditability and compliance.
- Optimistic concurrency control.

### Out of Scope
- Visual schema designer UI (future capability).
- Cross-tenant object sharing (future capability).
- Real-time subscriptions/notifications (future capability).
- Computed/formula fields (future capability).

## Business Requirements

### P0 Requirements (Critical)

### BR-EO-001 (P0): Extensible Object Type Definition
The system MUST provide a mechanism to define extensible object type schemas in GTS format, enabling:
- typed field definitions with JSON Schema
- validation rules
- relationships (foreign keys) between object types
- configurable indexes per field
- declaration of whether a type is extensible by tenants

### BR-EO-002 (P0): Extensible Object Storage and Retrieval
The system MUST implement storage and retrieval APIs for extensible objects, including:
- create, read, update, and delete operations
- batch operations for efficiency (batch create, update, delete)
- optimistic concurrency control (version-based conflict detection)
- storage as real SQL tables with typed columns

### P1 Requirements (Important)

### BR-EO-003 (P1): Runtime Schema Extension (Custom Fields)
The system MUST allow tenants to extend base object types with custom fields at runtime, where:
- custom fields become real SQL columns (not JSON blobs)
- custom fields are queryable, sortable, and indexable
- custom field types include: string, text, integer, number, boolean, date, datetime, reference
- custom fields can have validation constraints (enum, max_length, unique)

### BR-EO-004 (P1): Extensible Object Indexing and Search
The system MUST provide appropriate indexing and search capability for extensible objects, including:
- configurable indexes per field (exact, prefix, fulltext, numeric, datetime)
- unique constraints
- query API supporting filtering, sorting, and pagination
- full-text search where applicable
- queries work on both base fields and custom extension fields

### BR-EO-005 (P1): Extensible Object Lifecycle Hooks
The system MUST support hook functions for lifecycle events, including:
- before_create hooks (synchronous, can block/transform)
- on_create hooks (asynchronous, fire-and-forget)
- before_update hooks (synchronous, can block/transform)
- on_update hooks (asynchronous, fire-and-forget)
- before_delete hooks (synchronous, can block)
- on_delete hooks (asynchronous, fire-and-forget)
- configurable timeouts for hook execution

### BR-EO-006 (P1): Extensible Object Methods
The system MUST support attaching serverless functions as methods on object types, enabling:
- custom business logic callable via API
- methods receive object data and caller-provided arguments
- methods execute with caller's security context

### BR-EO-007 (P1): Extensible Object Access Control
The extensible objects MUST apply secure access control by specification, including:
- tenant isolation (mandatory)
- integration with platform Secure ORM
- role-based access control
- field-level access control where needed (future consideration)

### BR-EO-008 (P1): Foreign Key Relationships
The system MUST support foreign key relationships between object types, including:
- reference fields that point to other object types
- configurable on_delete behavior (restrict, cascade, set_null)
- optional eager loading of referenced objects

### P2 Requirements (Nice-to-have)

### BR-EO-101 (P2): Extensible Object Change Log
The system SHOULD provide a change log for extensible objects at both the user and system level to support auditability and compliance requirements, including:
- who made the change (actor type and ID)
- when the change was made (timestamp)
- what was changed (before/after values)
- correlation ID for tracing

### BR-EO-102 (P2): Extensible Object Schema Versioning
The system SHOULD support versioning of extensible object schemas so that:
- schema changes can be applied incrementally
- existing objects remain accessible during migrations
- changes are traceable and can be rolled back where needed

### BR-EO-103 (P2): Extensible Object Import/Export
The system SHOULD support importing and exporting extensible object data to enable backup, migration, and cross-environment management.

### BR-EO-104 (P2): Auto-Populated System Fields
The system SHOULD automatically populate system fields on objects:
- created_at, created_by on create
- updated_at, updated_by on update
- configurable per object type

## Acceptance Criteria (Business-Level)

### Schema and Type Management
- Object types can be defined in GTS format and validated before registration.
- Object types can be registered, retrieved, updated, and deleted via API.
- Tenants can extend object types with custom fields that become real SQL columns.
- Schema changes do not corrupt existing object instances.

### Object Instance Management
- Objects can be created, read, updated, and deleted via APIs.
- Batch operations work correctly with proper error handling.
- Optimistic concurrency prevents lost updates.

### Search and Querying
- Objects can be queried with filters, sorting, and pagination.
- Indexes are automatically maintained for configured fields.
- Queries work on both base fields and custom extension fields.
- Full-text search returns relevant results.

### Lifecycle Hooks and Methods
- Hook functions are invoked at the appropriate lifecycle events.
- Before-hook failures prevent or roll back the triggering operation.
- After-hook failures are logged but don't affect the committed operation.
- Object methods can be invoked via API and execute correctly.

### Access Control
- A tenant can only access its own extensible objects.
- Access control rules are enforced consistently across all APIs.
- Foreign key constraints are enforced within tenant boundaries.

## Non-Functional Business Requirements
- **Availability**: extensible object service availability MUST meet or exceed 99.95% monthly.
- **Query latency**: under normal load, object queries SHOULD respond promptly (target p95 ≤ 100 ms).
- **Write latency**: under normal load, object writes SHOULD complete promptly (target p95 ≤ 50 ms).
- **Compliance**: audit trails and tenant isolation MUST support SOC 2-aligned controls.
- **Scalability**: system SHOULD handle 100+ custom fields per object type without significant performance degradation.

## Assumptions
- Platform identity and authorization are available and can be used to determine user/system context.
- Persistent storage (PostgreSQL) exists to support durability of object data.
- The Serverless Runtime capability exists to execute lifecycle hook functions and methods.
- Secure ORM (SecureConn + SecurityCtx) is available for tenant isolation.

## Risks to be mitigated
- **Schema complexity**: complex schemas may be difficult for tenants to manage.
- **Migration challenges**: schema changes may require careful data migration.
- **Hook performance**: poorly performing hooks could impact object operation latency. Mitigated by configurable timeouts.
- **Index maintenance**: large numbers of indexes may impact write performance.
- **Custom field proliferation**: tenants adding too many custom fields could impact performance. Consider limits.
- **Field type conversion**: changing field types requires data migration and may fail if data is incompatible.
