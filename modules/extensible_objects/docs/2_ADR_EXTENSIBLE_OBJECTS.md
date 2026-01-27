# ADR — Extensible Objects (Serverless Object Storage)

## Status
Proposed

## Context
The Serverless Runtime requires a mechanism for tenants and users to define, store, and manage custom data objects at runtime without requiring platform rebuilds or traditional database provisioning.

This ADR defines the design of **Extensible Objects**: serverless objects with dynamic schemas deployed at runtime, with indexed search, lifecycle hooks, and data APIs.

### Related Requirements
- **BR-136 (P1)**: The system MUST provide a mechanism to define extensible object schemas in GTS format, implement storage and retrieval APIs with appropriate indexing and search capability, and hook functions for lifecycle events (create, update, delete).
- **BR-208 (P2)**: The system SHOULD provide a change log for extensible objects at both the user and system level to support auditability and compliance requirements.
- **BR-002 (P0)**: Tenant and user registries — Extensible Objects inherit tenant isolation requirements.
- **BR-017 (P0)**: Access control and separation of duties.
- **BR-018 (P0)**: Data protection and privacy controls.

## Goals
- Enable applications to define custom object base schemas at runtime using GTS format.
- Enable tenants and users to define and extend custom object schemas at runtime using GTS format.
- Allow runtime deployment/registration of object schemas without rebuilding or compiling Rust code per schema.
- Provide CRUD APIs for object instances with tenant isolation.
- Support indexed fields and real sql fields for efficient querying and search.
- Enable lifecycle hook functions (on create, update, delete) for custom and extensible business logic.
- Maintain auditability via change logs at user and system levels.
- Integrate with the existing Serverless Runtime invocation model.

## Non-goals
- Replacing the platform's core relational data stores.
- Providing a full-featured relational database or SQL interface.
- Real-time synchronization or subscriptions (future capability).
- Cross-tenant object sharing (objects are strictly tenant-isolated).

## Terminology
- **Extensible Object Type**: A GTS schema definition that describes the structure of an extensible object.
- **Extensible Object Instance**: A persisted record conforming to an Extensible Object Type.
- **Base Type**: An application-defined object type with core fields.
- **Type Extension**: Runtime-defined additional fields added to a base type by tenants/users (like Jira custom fields).
- **Lifecycle Hook**: A serverless function invoked automatically on object lifecycle events.
- **Object Method**: A serverless function attached to an object type, callable via the API.
- **Change Log**: An append-only audit record of mutations to extensible object instances.

## Extensible Object Type Definition

### Schema Format
Extensible Object Types are defined as GTS-identified JSON Schemas, following the same conventions as function/workflow definitions.
Schema definitions are deployed and stored at runtime; the runtime MUST validate object instances against their JSON Schemas without compiling Rust code per schema.

**Schema ID (type):** `gts.x.core.faas.extensible_object.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.extensible_object.v1~",
  "title": "Extensible Object Type Definition",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "id": {
      "type": "string",
      "description": "Well-known type identifier of this extensible object type.",
      "examples": ["gts.x.core.faas.extensible_object.v1~vendor.app.namespace.object_name.v1~"]
    },

    "title": { "type": "string" },
    "description": { "type": "string" },

    "schema": {
      "description": "JSON Schema defining the object's data structure. All instances MUST conform to this schema.",
      "oneOf": [
        {"$ref": "https://json-schema.org/draft/2020-12/schema"},
        {"type": "object", "additionalProperties": true}
      ]
    },

    "indexes": {
      "type": "array",
      "description": "Fields to index for efficient querying. Each index specifies a field path and index type.",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "field": {
            "type": "string",
            "description": "JSON path to the field to index (e.g., 'email', 'metadata.category')."
          },
          "type": {
            "type": "string",
            "enum": ["exact", "prefix", "fulltext", "numeric", "datetime"],
            "default": "exact",
            "description": "Index type determining supported query operations."
          },
          "unique": {
            "type": "boolean",
            "default": false,
            "description": "If true, the indexed field must be unique across all instances within the tenant scope."
          }
        },
        "required": ["field"]
      }
    },

    "references": {
      "type": "object",
      "description": "Foreign key relationships to other Extensible Object types. Keys are field names in the schema that hold object IDs.",
      "additionalProperties": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "target_type_id": {
            "type": "string",
            "description": "GTS type ID of the referenced Extensible Object type."
          },
          "on_delete": {
            "type": "string",
            "enum": ["restrict", "cascade", "set_null"],
            "default": "restrict",
            "description": "Behavior when the referenced object is deleted."
          },
          "eager_load": {
            "type": "boolean",
            "default": false,
            "description": "If true, referenced objects are included in query results by default."
          }
        },
        "required": ["target_type_id"]
      }
    },

    "methods": {
      "type": "object",
      "description": "Serverless functions attached to this object type, callable via objects.call API.",
      "additionalProperties": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "function_id": {
            "type": "string",
            "description": "GTS function ID to invoke."
          },
          "description": {
            "type": "string"
          }
        },
        "required": ["function_id"]
      }
    },

    "extensible": {
      "type": "boolean",
      "default": false,
      "description": "If true, tenants/users can add custom fields to this type at runtime (like Jira custom fields). Extended fields become real SQL columns and are queryable/sortable."
    },

    "hooks": {
      "type": "object",
      "description": "Lifecycle hook function references. Each hook is a GTS function ID invoked on the corresponding event.",
      "additionalProperties": false,
      "properties": {
        "on_create": {
          "type": "string",
          "description": "Function ID invoked after an object is created. Receives the created object."
        },
        "on_update": {
          "type": "string",
          "description": "Function ID invoked after an object is updated. Receives the previous and updated object."
        },
        "on_delete": {
          "type": "string",
          "description": "Function ID invoked after an object is deleted. Receives the deleted object."
        },
        "on_before_create": {
          "type": "string",
          "description": "Function ID invoked before an object is created. Can validate or transform the object. Returning an error prevents creation."
        },
        "on_before_update": {
          "type": "string",
          "description": "Function ID invoked before an object is updated. Can validate or transform the object. Returning an error prevents update."
        },
        "on_before_delete": {
          "type": "string",
          "description": "Function ID invoked before an object is deleted. Returning an error prevents deletion."
        }
      }
    },

    "traits": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "change_log": {
          "type": "boolean",
          "default": true,
          "description": "If true, all mutations are recorded in the change log for auditability."
        },
        "soft_delete": {
          "type": "boolean",
          "default": false,
          "description": "If true, deleted objects are marked as deleted rather than physically removed."
        },
        "versioning": {
          "type": "boolean",
          "default": false,
          "description": "If true, previous versions of objects are retained on update."
        },
        "retention_days": {
          "type": "integer",
          "minimum": 1,
          "description": "Optional retention period in days for deleted objects and change log entries."
        },
        "max_instances": {
          "type": "integer",
          "minimum": 1,
          "description": "Optional maximum number of instances allowed for this object type within a tenant."
        },
        "auto_fields": {
          "type": "object",
          "description": "System-managed fields automatically populated on create/update.",
          "additionalProperties": false,
          "properties": {
            "created_at": {"type": "boolean", "default": true},
            "created_by": {"type": "boolean", "default": true},
            "updated_at": {"type": "boolean", "default": true},
            "updated_by": {"type": "boolean", "default": true}
          }
        },
        "max_before_hook_timeout_ms": {
          "type": "integer",
          "minimum": 100,
          "maximum": 30000,
          "default": 5000,
          "description": "Maximum execution time for before_* hooks in milliseconds."
        },
        "max_after_hook_timeout_ms": {
          "type": "integer",
          "minimum": 100,
          "maximum": 60000,
          "default": 30000,
          "description": "Maximum execution time for on_* hooks in milliseconds."
        }
      }
    }
  },
  "required": ["id", "schema"]
}
```

### Derived Extensible Object Types
Derived extensible object types MUST reference the base schema with `allOf` and pin fixed values using `const` (object-level or property-level), consistent with function/workflow definitions in `2_ADR_FUNCTION_ENTRYPOINT.md`. At minimum, derived types MUST pin `id` and `schema`; `indexes`, `hooks`, and `traits` SHOULD be pinned when present.

### Example: Customer Preferences Object Type

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
  "allOf": [
    {"$ref": "gts://gts.x.core.faas.extensible_object.v1~"}
  ],
  "type": "object",
  "properties": {
    "id": {"const": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~"},
    "title": {"const": "Customer Preferences"},
    "description": {"const": "Stores customer notification and display preferences."},
    "schema": {
      "const": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "customer_id": {"type": "string"},
          "email_notifications": {"type": "boolean", "default": true},
          "sms_notifications": {"type": "boolean", "default": false},
          "locale": {"type": "string", "default": "en-US"},
          "timezone": {"type": "string", "default": "UTC"},
          "preferred_contact_method": {
            "type": "string",
            "enum": ["email", "sms", "phone"]
          },
          "account_manager_id": {
            "type": "string",
            "description": "Reference to the assigned account manager (User object)."
          }
        },
        "required": ["customer_id"]
      }
    },
    "indexes": {
      "const": [
        {"field": "customer_id", "type": "exact", "unique": true},
        {"field": "locale", "type": "exact"},
        {"field": "account_manager_id", "type": "exact"}
      ]
    },
    "references": {
      "const": {
        "account_manager_id": {
          "target_type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.user.v1~",
          "on_delete": "set_null",
          "eager_load": false
        }
      }
    },
    "methods": {
      "const": {
        "send_welcome_email": {
          "function_id": "gts.x.core.faas.func.v1~vendor.app.crm.send_welcome_email.v1~",
          "description": "Sends a localized welcome email to the customer."
        },
        "reset_to_defaults": {
          "function_id": "gts.x.core.faas.func.v1~vendor.app.crm.reset_preferences.v1~",
          "description": "Resets all preferences to default values."
        }
      }
    },
    "extensible": {"const": true},
    "hooks": {
      "const": {
        "on_update": "gts.x.core.faas.func.v1~vendor.app.crm.on_preferences_updated.v1~"
      }
    },
    "traits": {
      "const": {
        "change_log": true,
        "soft_delete": true,
        "retention_days": 90,
        "auto_fields": {
          "created_at": true,
          "created_by": true,
          "updated_at": true,
          "updated_by": true
        },
        "max_before_hook_timeout_ms": 5000
      }
    }
  },
  "required": ["id", "schema"]
}
```

## Type Management API

The Type Management API provides operations for registering, extending, and managing Extensible Object type definitions at runtime.

### Endpoint
`POST /api/serverless-runtime/v1/types`

### Operations

#### Register Type
Registers a new Extensible Object type definition. The schema is validated and the corresponding database table is created.

```json
{
  "jsonrpc": "2.0",
  "method": "types.register",
  "params": {
    "definition": {
      "id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
      "title": "Customer Preferences",
      "schema": { ... },
      "indexes": [ ... ],
      "references": { ... },
      "traits": { ... }
    }
  },
  "id": "req-001"
}
```

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "status": "active",
    "created_at": "2025-01-26T05:46:00Z",
    "table_name": "do_vendor_app_crm_customer_preferences_v1"
  },
  "id": "req-001"
}
```

#### Get Type
Retrieves a type definition by ID.

```json
{
  "jsonrpc": "2.0",
  "method": "types.get",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~"
  },
  "id": "req-002"
}
```

#### List Types
Lists all registered types for the tenant.

```json
{
  "jsonrpc": "2.0",
  "method": "types.list",
  "params": {
    "filter": "vendor.app.crm.*",
    "include_system": false,
    "limit": 100
  },
  "id": "req-003"
}
```

#### Update Type
Updates a type definition. Only additive changes are allowed (new fields, new indexes). Removing fields or changing field types requires migration.

```json
{
  "jsonrpc": "2.0",
  "method": "types.update",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "add_fields": {
      "phone_notifications": {"type": "boolean", "default": false}
    },
    "add_indexes": [
      {"field": "phone_notifications", "type": "exact"}
    ]
  },
  "id": "req-004"
}
```

#### Delete Type
Deletes a type definition. Fails if instances exist unless `force: true` is specified.

```json
{
  "jsonrpc": "2.0",
  "method": "types.delete",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "force": false
  },
  "id": "req-005"
}
```

### Type Extensions (Custom Fields)

For types with `extensible: true`, tenants can add custom fields at runtime. These become real SQL columns, not JSON blobs.

#### Extend Type
Adds custom fields to an extensible type for a specific tenant.

```json
{
  "jsonrpc": "2.0",
  "method": "types.extend",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "extensions": [
      {
        "field_name": "loyalty_tier",
        "field_type": "string",
        "enum": ["bronze", "silver", "gold", "platinum"],
        "default": "bronze",
        "indexed": true,
        "description": "Customer loyalty program tier"
      },
      {
        "field_name": "referral_code",
        "field_type": "string",
        "max_length": 20,
        "indexed": true,
        "unique": true
      },
      {
        "field_name": "notes",
        "field_type": "text",
        "indexed": false
      }
    ]
  },
  "id": "req-006"
}
```

**Extension Field Types:**
- `string` - VARCHAR with optional `max_length` (default 256)
- `text` - TEXT (unlimited length, not indexable for exact match)
- `integer` - INTEGER
- `number` - DOUBLE PRECISION
- `boolean` - BOOLEAN
- `date` - DATE
- `datetime` - TIMESTAMPTZ
- `reference` - Foreign key to another Extensible Object type

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "type_id": "...",
    "extensions_added": 3,
    "columns_created": [
      "ext_loyalty_tier",
      "ext_referral_code",
      "ext_notes"
    ]
  },
  "id": "req-006"
}
```

#### List Extensions
Lists custom fields added to a type.

```json
{
  "jsonrpc": "2.0",
  "method": "types.list_extensions",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~"
  },
  "id": "req-007"
}
```

#### Remove Extension
Removes a custom field. Data in that field is lost.

```json
{
  "jsonrpc": "2.0",
  "method": "types.remove_extension",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "field_name": "referral_code",
    "confirm_data_loss": true
  },
  "id": "req-008"
}
```

---

## Data API

### Endpoint
`POST /api/serverless-runtime/v1/objects`

The Data API uses JSON-RPC 2.0 for consistency with the function invocation API.

### Operations

#### Create Object
```json
{
  "jsonrpc": "2.0",
  "method": "objects.create",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "data": {
      "customer_id": "cust-12345",
      "email_notifications": true,
      "locale": "en-GB"
    }
  },
  "id": "req-001"
}
```

**Success Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "object_id": "obj-abc123",
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "data": {
      "customer_id": "cust-12345",
      "email_notifications": true,
      "sms_notifications": false,
      "locale": "en-GB",
      "timezone": "UTC",
      "preferred_contact_method": "email",
      "account_manager_id": null
    },
    "extensions": {
      "loyalty_tier": "gold",
      "referral_code": "REF-ABC123"
    },
    "metadata": {
      "created_at": "2025-01-26T05:46:00Z",
      "created_by": "user-xyz",
      "updated_at": "2025-01-26T05:46:00Z",
      "updated_by": "user-xyz",
      "version": 1
    }
  },
  "id": "req-001"
}
```

Note: `extensions` contains custom fields added via `types.extend`. These are stored in real SQL columns (`ext_loyalty_tier`, `ext_referral_code`) and are queryable/sortable.

#### Get Object
```json
{
  "jsonrpc": "2.0",
  "method": "objects.get",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "object_id": "obj-abc123"
  },
  "id": "req-002"
}
```

#### Update Object
```json
{
  "jsonrpc": "2.0",
  "method": "objects.update",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "object_id": "obj-abc123",
    "data": {
      "locale": "de-DE",
      "sms_notifications": true
    },
    "merge": true,
    "expected_version": 1
  },
  "id": "req-003"
}
```

- `merge: true` (default) performs a partial update, merging provided fields with existing data.
- `merge: false` replaces the entire object data.
- `expected_version` (optional): Optimistic concurrency control. If provided, the update only succeeds if the object's current version matches. Returns `version_conflict` error otherwise.

#### Delete Object
```json
{
  "jsonrpc": "2.0",
  "method": "objects.delete",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "object_id": "obj-abc123"
  },
  "id": "req-004"
}
```

#### Query Objects
Querying uses the ModKit OData grammar (same semantics as REST endpoints). The `query` object mirrors `$filter`, `$orderby`, and `$select` expressions as strings. Filters and ordering can reference `data.<field>` for object data and `metadata.<field>` for system fields (e.g., `metadata.created_at`).
```json
{
  "jsonrpc": "2.0",
  "method": "objects.query",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "query": {
      "filter": "data.locale eq 'en-US'",
      "orderby": "metadata.created_at desc",
      "select": "object_id,data.customer_id,data.locale,metadata.created_at",
      "limit": 50,
      "cursor": null
    }
  },
  "id": "req-005"
}
```

**Query Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "items": [
      {
        "object_id": "obj-abc123",
        "data": { ... },
        "metadata": { ... }
      }
    ],
    "next_cursor": "cursor-xyz",
    "total_count": 150
  },
  "id": "req-005"
}
```

### OData Query Semantics
- `$filter`, `$orderby`, and `$select` follow ModKit OData conventions (see `docs/ODATA_SELECT.md`).
- Use dot notation for nested fields (e.g., `data.metadata.category`).
- Indexes determine which fields are filterable/orderable.

### Batch Operations

For efficiency, batch operations allow multiple creates, updates, or deletes in a single request.

#### Batch Create
```json
{
  "jsonrpc": "2.0",
  "method": "objects.batch_create",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "items": [
      {"customer_id": "cust-001", "locale": "en-US"},
      {"customer_id": "cust-002", "locale": "en-GB"},
      {"customer_id": "cust-003", "locale": "de-DE"}
    ],
    "on_error": "continue"
  },
  "id": "req-010"
}
```

- `on_error: "continue"` (default) processes all items, returning individual success/error per item.
- `on_error: "abort"` stops on first error and rolls back all changes (transactional).

**Response:**
```json
{
  "jsonrpc": "2.0",
  "result": {
    "succeeded": 2,
    "failed": 1,
    "items": [
      {"index": 0, "object_id": "obj-001", "status": "created"},
      {"index": 1, "object_id": "obj-002", "status": "created"},
      {"index": 2, "error": {"code": "unique_constraint", "field": "customer_id"}}
    ]
  },
  "id": "req-010"
}
```

#### Batch Update
```json
{
  "jsonrpc": "2.0",
  "method": "objects.batch_update",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "items": [
      {"object_id": "obj-001", "data": {"locale": "fr-FR"}, "expected_version": 1},
      {"object_id": "obj-002", "data": {"locale": "es-ES"}, "expected_version": 1}
    ],
    "merge": true,
    "on_error": "continue"
  },
  "id": "req-011"
}
```

#### Batch Delete
```json
{
  "jsonrpc": "2.0",
  "method": "objects.batch_delete",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "object_ids": ["obj-001", "obj-002", "obj-003"],
    "on_error": "continue"
  },
  "id": "req-012"
}
```

### Object Method Calls

For types with defined `methods`, invoke attached serverless functions.

#### Call Method
```json
{
  "jsonrpc": "2.0",
  "method": "objects.call",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "object_id": "obj-abc123",
    "method": "send_welcome_email",
    "args": {
      "template": "onboarding_v2",
      "include_promo": true
    }
  },
  "id": "req-013"
}
```

The method function receives:
```json
{
  "object_id": "obj-abc123",
  "object_data": { ... },
  "method": "send_welcome_email",
  "args": { "template": "onboarding_v2", "include_promo": true },
  "actor": { "type": "user", "id": "user-xyz" }
}
```

## Lifecycle Hooks

### Hook Function Contract
Lifecycle hook functions follow the standard entrypoint contract defined in `2_ADR_FUNCTION_ENTRYPOINT.md`.
Hook entrypoints MUST declare:
- `params` referencing `gts://gts.x.core.faas.extensible_object.hook_input.v1~`
- `returns` referencing `gts://gts.x.core.faas.extensible_object.hook_output.v1~`

#### Hook Input Schema
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.extensible_object.hook_input.v1~",
  "title": "Extensible Object Hook Input",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "event": {
      "type": "string",
      "enum": ["before_create", "on_create", "before_update", "on_update", "before_delete", "on_delete"]
    },
    "type_id": {"type": "string"},
    "object_id": {"type": "string"},
    "data": {"type": "object"},
    "previous_data": {"type": "object", "description": "Only present for update events."},
    "actor": {
      "type": "object",
      "additionalProperties": false,
      "properties": {
        "type": {"type": "string", "enum": ["system", "api_client", "user"]},
        "id": {"type": "string"}
      }
    }
  },
  "required": ["event", "type_id", "data"]
}
```

#### Hook Output Schema (before_* hooks)
```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.extensible_object.hook_output.v1~",
  "title": "Extensible Object Hook Output",
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "allow": {"type": "boolean", "default": true},
    "data": {"type": "object", "description": "Transformed data (optional). If provided, replaces the original data."},
    "error": {"$ref": "gts://gts.x.core.faas.err.v1~", "description": "Error to return if allow is false."}
  }
}
```

### Hook Execution Semantics
- **before_*** hooks are executed synchronously before the operation commits.
- **on_*** hooks are executed asynchronously after the operation commits.
- Hook failures in **before_*** hooks abort the operation and return an error.
- Hook failures in **on_*** hooks are logged but do not affect the committed operation.
- Hooks inherit the security context of the original request.

### Example: Starlark Hook Function

```python
def main(ctx, input):
    if input.event == "on_update":
        customer_id = input.data.get("customer_id")
        previous_locale = input.previous_data.get("locale")
        new_locale = input.data.get("locale")
        
        if previous_locale != new_locale:
            # Notify downstream systems of locale change
            p = r_http_post_v1(
                "https://example.notifications/api/locale-changed",
                body = {
                    "customer_id": customer_id,
                    "old_locale": previous_locale,
                    "new_locale": new_locale,
                },
                timeout_ms = 2000,
                exit_on_http_error = False,
            )
            r_await(p)
    
    return {}
```

## Change Log

### Purpose
The change log provides an immutable audit trail of all mutations to extensible object instances, supporting:
- Compliance and auditability (BR-208)
- Debugging and incident investigation
- Event sourcing patterns
- Edit history and undo/redo capabilities

### Change Log Entry Schema
```json
{
  "type": "object",
  "properties": {
    "log_id": {"type": "string"},
    "timestamp": {"type": "string", "format": "date-time"},
    "type_id": {"type": "string"},
    "object_id": {"type": "string"},
    "operation": {"type": "string", "enum": ["create", "update", "delete"]},
    "actor": {
      "type": "object",
      "properties": {
        "type": {"type": "string", "enum": ["system", "api_client", "user"]},
        "id": {"type": "string"}
      }
    },
    "data_before": {"type": "object", "description": "Object state before mutation (null for create)."},
    "data_after": {"type": "object", "description": "Object state after mutation (null for delete)."},
    "correlation_id": {"type": "string"},
    "tenant_id": {"type": "string"}
  }
}
```

### Change Log Query API
```json
{
  "jsonrpc": "2.0",
  "method": "objects.changelog",
  "params": {
    "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
    "object_id": "obj-abc123",
    "from_timestamp": "2025-01-01T00:00:00Z",
    "to_timestamp": "2025-01-26T23:59:59Z",
    "limit": 100
  },
  "id": "req-006"
}
```

## Starlark Runtime Helpers

The Starlark runtime provides helpers for interacting with Extensible Objects from functions and workflows.

### Single Object Operations

#### r_object_create_v1(...)
```
r_object_create_v1(type_id, data) -> Promise<ObjectResult>
```

#### r_object_get_v1(...)
```
r_object_get_v1(type_id, object_id) -> Promise<ObjectResult>
```

#### r_object_update_v1(...)
```
r_object_update_v1(type_id, object_id, data, merge=True, expected_version=None) -> Promise<ObjectResult>
```

#### r_object_delete_v1(...)
```
r_object_delete_v1(type_id, object_id) -> Promise<ObjectResult>
```

#### r_object_query_v1(...)
```
r_object_query_v1(type_id, filter=None, sort=None, limit=50, cursor=None) -> Promise<QueryResult>
```

### Batch Operations

#### r_object_batch_create_v1(...)
```
r_object_batch_create_v1(type_id, items, on_error="continue") -> Promise<BatchResult>
```

#### r_object_batch_update_v1(...)
```
r_object_batch_update_v1(type_id, items, merge=True, on_error="continue") -> Promise<BatchResult>
```
Each item in `items` is a dict with `object_id`, `data`, and optional `expected_version`.

#### r_object_batch_delete_v1(...)
```
r_object_batch_delete_v1(type_id, object_ids, on_error="continue") -> Promise<BatchResult>
```

### Object Method Calls

#### r_object_call_v1(...)
```
r_object_call_v1(type_id, object_id, method, args=None) -> Promise<MethodResult>
```

### Type Management

#### r_type_get_v1(...)
```
r_type_get_v1(type_id) -> Promise<TypeResult>
```

#### r_type_extend_v1(...)
```
r_type_extend_v1(type_id, extensions) -> Promise<ExtendResult>
```

### Result Shapes

#### ObjectResult
```python
struct(
    ok = True,
    value = struct(
        object_id = "obj-abc123",
        data = {...},
        metadata = struct(created_at = "...", updated_at = "...", version = 1)
    )
)
# or on error:
struct(ok = False, error = <FaaS Error envelope>)
```

#### BatchResult
```python
struct(
    ok = True,
    value = struct(
        succeeded = 2,
        failed = 1,
        items = [
            struct(index = 0, object_id = "obj-001", status = "created"),
            struct(index = 1, object_id = "obj-002", status = "created"),
            struct(index = 2, error = struct(code = "unique_constraint", field = "customer_id"))
        ]
    )
)
```

#### MethodResult
```python
struct(
    ok = True,
    value = <return value from method function>
)
```

## Tenant Isolation

- All Extensible Object Types and instances are strictly scoped to a single tenant.
- Object type definitions can be further scoped to user-level ownership within a tenant.
- Queries and operations automatically filter by tenant context from the security context.
- Persistence MUST use `modkit-db` Secure ORM (`SecureConn`) with a request-scoped `SecurityCtx` per operation; unscoped queries MUST be impossible (see `docs/SECURE-ORM.md`).
- Extensible Object storage entities MUST declare scoping via `ScopableEntity`: `tenant_col = tenant_id`, `resource_col = object_id`, optional `owner_col` for user ownership, and `type_col = type_id` for object type scoping.
- Cross-tenant access is not supported.

## Security Considerations

- **Schema validation**: All object data MUST be validated against the type schema before persistence.
- **Index injection**: Filter queries MUST be parameterized to prevent injection attacks.
- **Hook authorization**: Hook functions execute with the caller's security context, not elevated privileges.
- **Sensitive fields**: The schema MAY annotate fields as sensitive (future: redaction in change logs).
- **Rate limiting**: Object operations SHOULD be subject to per-tenant rate limits.
- **Secure ORM enforcement**: Extensible Object data access MUST use `SecureConn` + `SecurityCtx` per operation; raw database access is only allowed for migrations/admin tooling behind the `insecure-escape` feature.

## Error Types

Extensible Object operations return structured errors following the FaaS error envelope pattern.

### Error Schema
**GTS ID:** `gts.x.core.faas.extensible_object.err.v1~`

```json
{
  "$schema": "https://json-schema.org/draft/2020-12/schema",
  "$id": "gts://gts.x.core.faas.extensible_object.err.v1~",
  "type": "object",
  "properties": {
    "code": {
      "type": "string",
      "enum": [
        "not_found",
        "type_not_found",
        "validation_failed",
        "unique_constraint",
        "reference_constraint",
        "version_conflict",
        "hook_failed",
        "hook_timeout",
        "max_instances_exceeded",
        "type_not_extensible",
        "extension_not_found",
        "permission_denied",
        "rate_limited"
      ]
    },
    "message": {"type": "string"},
    "details": {
      "type": "object",
      "properties": {
        "type_id": {"type": "string"},
        "object_id": {"type": "string"},
        "field": {"type": "string"},
        "expected_version": {"type": "integer"},
        "actual_version": {"type": "integer"},
        "hook_name": {"type": "string"},
        "validation_errors": {
          "type": "array",
          "items": {
            "type": "object",
            "properties": {
              "path": {"type": "string"},
              "message": {"type": "string"}
            }
          }
        }
      }
    }
  },
  "required": ["code", "message"]
}
```

### Error Codes

| Code | Description |
|------|-------------|
| `not_found` | Object instance not found |
| `type_not_found` | Extensible Object type not registered |
| `validation_failed` | Object data does not conform to schema |
| `unique_constraint` | Unique index violation |
| `reference_constraint` | Foreign key constraint violation (referenced object doesn't exist or delete blocked) |
| `version_conflict` | Optimistic concurrency conflict (`expected_version` mismatch) |
| `hook_failed` | A `before_*` hook returned `allow: false` or threw an error |
| `hook_timeout` | Hook execution exceeded timeout |
| `max_instances_exceeded` | `traits.max_instances` limit reached |
| `type_not_extensible` | Attempted to extend a type with `extensible: false` |
| `extension_not_found` | Referenced extension field does not exist |
| `permission_denied` | Caller lacks permission for this operation |
| `rate_limited` | Tenant rate limit exceeded |

### Example Error Response
```json
{
  "jsonrpc": "2.0",
  "error": {
    "code": -32000,
    "message": "Extensible Object operation failed",
    "data": {
      "code": "version_conflict",
      "message": "Object was modified by another request",
      "details": {
        "type_id": "gts.x.core.faas.extensible_object.v1~vendor.app.crm.customer_preferences.v1~",
        "object_id": "obj-abc123",
        "expected_version": 1,
        "actual_version": 3
      }
    }
  },
  "id": "req-003"
}

## Storage Considerations

### General Requirements
- The runtime MUST support pluggable storage backends.
- Storage MUST support atomic operations for create, update, and delete.
- Storage MUST support efficient index-based queries.
- Storage MUST encrypt data at rest (BR-034).
- For SeaORM-backed storage, entities MUST be `Scopable` and all queries MUST originate from `SecureConn`, applying OData filters on top of scoped queries.

### Schema-to-SQL Table Generation

Each Extensible Object Type generates a **real SQL table** with typed columns. Object data is NOT stored as a JSON blob.

#### Table Naming
Tables are named using a deterministic pattern:
```
do_{vendor}_{app}_{namespace}_{object_name}_v{version}
```
Example: `do_vendor_app_crm_customer_preferences_v1`

#### Column Naming Strategy
Following the Baserow pattern, columns use `field_{id}` naming to:
- Allow any user-facing field names (including SQL reserved words)
- Enable field renames without schema migration
- Avoid naming collisions

| User Field Name | Database Column |
|-----------------|-----------------|
| `customer_id` | `field_001` |
| `email_notifications` | `field_002` |
| `locale` | `field_003` |

A metadata table maps field IDs to user-facing names.

For **extension fields** (custom fields added at runtime), columns use `ext_{field_name}` prefix.

#### JSON Schema to SQL Type Mapping

| JSON Schema | SQL Type (PostgreSQL) |
|-------------|----------------------|
| `type: "string"` | `VARCHAR(256)` (or `maxLength` if specified) |
| `type: "string", format: "date-time"` | `TIMESTAMPTZ` |
| `type: "string", format: "date"` | `DATE` |
| `type: "string", format: "uuid"` | `UUID` |
| `type: "string", format: "email"` | `VARCHAR(320)` |
| `type: "string", enum: [...]` | `VARCHAR` + CHECK constraint |
| `type: "integer"` | `INTEGER` |
| `type: "integer", format: "int64"` | `BIGINT` |
| `type: "number"` | `DOUBLE PRECISION` |
| `type: "number", format: "decimal"` | `NUMERIC(precision, scale)` |
| `type: "boolean"` | `BOOLEAN` |
| `type: "array"` | Separate junction table (for references) or `JSONB` (for simple arrays) |
| `type: "object"` | `JSONB` (nested objects stored as JSON) |
| Reference field | `VARCHAR` (object ID) + foreign key constraint |

#### System Columns
Every table includes these system-managed columns:

| Column | Type | Description |
|--------|------|-------------|
| `id` | `VARCHAR(36)` | Primary key (object_id) |
| `tenant_id` | `VARCHAR(36)` | Tenant isolation |
| `version` | `INTEGER` | Optimistic concurrency version |
| `created_at` | `TIMESTAMPTZ` | Auto-populated on create |
| `created_by` | `VARCHAR(36)` | Auto-populated from security context |
| `updated_at` | `TIMESTAMPTZ` | Auto-populated on update |
| `updated_by` | `VARCHAR(36)` | Auto-populated from security context |
| `deleted_at` | `TIMESTAMPTZ` | Soft delete timestamp (if enabled) |

#### Index Generation
Indexes from the type definition are created as database indexes:

| Index Type | SQL Index |
|------------|-----------|
| `exact` | B-tree index |
| `prefix` | B-tree index (for LIKE 'prefix%' queries) |
| `fulltext` | GIN index with `tsvector` |
| `numeric` | B-tree index |
| `datetime` | B-tree index |
| `unique: true` | UNIQUE constraint |

### Field Conversion
When field types change (e.g., string → integer), the system MUST:
1. Check if conversion is safe (e.g., all values are numeric)
2. Create a new column with the target type
3. Migrate data with type conversion
4. Drop the old column
5. Rename the new column

If conversion would lose data, the operation fails with a validation error.

## Future Considerations

- **Real-time subscriptions**: WebSocket or SSE notifications for object changes. Consider Electric SQL's durable streams pattern for resumable subscriptions.
- **Cross-object transactions**: Atomic operations across multiple object types within a single request.
- **Computed/stored fields**: Derived fields computed from other fields (like Odoo's `compute` with `store=True`). Could store in database for query performance.
- **Field-level permissions**: Access control at the field level within an object (like Salesforce field-level security).
- **Import/export**: Bulk data import and export capabilities with schema validation.
- **Schema branching**: Git-style schema workflows for staging/testing changes before production (like Xata).
- **Point-in-time recovery**: Ability to restore objects to a previous point in time using change log.

---

## Appendix: Industry Research and Comparison

This section documents research on systems that share the core concept: **schema-driven SQL table generation with runtime extensibility, auto-generated APIs, and support for custom methods/hooks**.

### Core Concept Alignment

The Hyperspot Extensible Objects proposal has a specific architecture:
1. **GTS/JSON Schema defines the type** → converted to **real SQL tables with typed columns** (not `id + JSON blob`)
2. **Runtime schema extension** (like Jira custom fields) where users can add typed fields that are searchable/sortable
3. **Auto-generated CRUD + search APIs** based on the schema
4. **Foreign keys and relationships** as first-class concepts
5. **Methods attached to objects** (ORM-style business logic)
6. **Lifecycle hooks** for custom behavior

---

### Baserow (Most Similar)

**Source**: [Baserow](https://baserow.io/) | [Technical Blog](https://baserow.io/blog/how-baserow-lets-users-generate-django-models) | [GitHub](https://github.com/baserow/baserow)

#### Core Concept: Dynamic Django Models with Real SQL Tables

Baserow generates **real PostgreSQL tables** from user-defined schemas at runtime using Django's `schema_editor`. Each user-created field becomes an actual database column.

#### Architecture

**Metadata Tables**:
- `Table` - one row per user-defined table
- `Field` - one row per field in a table (stores field type, configuration)

**Dynamic Model Generation**:
```python
# Baserow uses Python's type() to generate Django models dynamically
# Then uses schema_editor to create/modify actual database tables

with connection.schema_editor() as schema_editor:
    schema_editor.create_model(DynamicModel)
    schema_editor.add_field(DynamicModel, new_field)
```

**Column Naming**: Database columns are named `field_{id}` to avoid conflicts and allow any user-chosen field names.

**Performance Optimization**: Models are cached in Redis since tables can have 100+ fields and regenerating models is expensive.

#### Field Type System

Each field type defines:
- `get_model_field()` - Returns the Django model field (e.g., `CharField`, `IntegerField`, `ForeignKey`)
- `get_serializer_field()` - Returns the DRF serializer field for API
- Field converters handle type changes (e.g., text → number) with data migration

**Supported Types**: Text, Long Text, Number, Boolean, Date, DateTime, URL, Email, Phone, Rating, Single Select, Multiple Select, Link (foreign key), Formula, Lookup, Rollup, Count, File, Created By, Last Modified By, UUID, Autonumber

#### Auto-Generated API

Every table gets REST endpoints automatically:
```
GET    /api/database/rows/table/{table_id}/
POST   /api/database/rows/table/{table_id}/
GET    /api/database/rows/table/{table_id}/{row_id}/
PATCH  /api/database/rows/table/{table_id}/{row_id}/
DELETE /api/database/rows/table/{table_id}/{row_id}/
```

API docs auto-update when schema changes: `https://api.baserow.io/api/redoc/`

#### Key Patterns to Adopt

1. **Real SQL columns, not JSON blobs**: Each field → real typed column with proper indexes
2. **Field converters for type changes**: Handle data migration when field types change
3. **Metadata-driven model generation**: Store schema in metadata tables, generate models dynamically
4. **Column naming strategy**: Use `field_{id}` pattern to decouple internal names from user-facing names
5. **Model caching**: Cache generated models for performance

---

### Directus

**Source**: [Directus](https://directus.io/) | [GitHub](https://github.com/directus/directus)

#### Core Concept: Database-First with Schema Mirroring

Directus sits on top of **any existing SQL database** and mirrors its schema to generate APIs. Unlike Baserow, Directus can connect to databases with pre-existing schemas.

> "When you create Directus collections, fields, defaults, datatypes... you are actually just creating tables, columns, etc in a custom SQL database."

#### Schema Management

- **No proprietary data model**: Your data is pure SQL, portable, and Directus can be removed without data changes
- **System tables separate from content**: Directus metadata (settings, permissions, revisions) stored separately
- **Works with existing databases**: Install on top of existing schema without migration

#### API Generation

Instant REST and GraphQL APIs from any SQL database:
- Supports PostgreSQL, MySQL, SQLite, OracleDB, CockroachDB, MariaDB, MS-SQL
- Real-time WebSocket support
- Granular permissions at row and field level

#### Key Patterns to Adopt

1. **Database portability**: Data remains in pure SQL format, no lock-in
2. **Support existing schemas**: Can wrap pre-existing database tables
3. **System/content separation**: Platform metadata separate from user data

---

### Salesforce Custom Objects

**Source**: [Salesforce Metadata API](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/customobject.htm) | [Custom Fields](https://developer.salesforce.com/docs/atlas.en-us.api_meta.meta/api_meta/customfield.htm)

#### Core Concept: Enterprise Custom Objects with Metadata API

Salesforce is the enterprise standard for runtime-extensible data models. Custom Objects and Custom Fields are created via Metadata API and become real database entities with full query support.

#### Metadata API for Schema Management

**Create Custom Object**:
```xml
<CustomObject xmlns="http://soap.sforce.com/2006/04/metadata">
    <label>Project</label>
    <pluralLabel>Projects</pluralLabel>
    <nameField>
        <label>Project Name</label>
        <type>Text</type>
    </nameField>
    <deploymentStatus>Deployed</deploymentStatus>
    <sharingModel>ReadWrite</sharingModel>
</CustomObject>
```

**Create Custom Field**:
```xml
<CustomField xmlns="http://soap.sforce.com/2006/04/metadata">
    <fullName>Budget__c</fullName>
    <label>Budget</label>
    <type>Currency</type>
    <precision>18</precision>
    <scale>2</scale>
</CustomField>
```

#### Dynamic Schema Introspection (Apex)

```apex
// Get all fields on an object at runtime
Schema.DescribeSObjectResult objDescribe = Account.sObjectType.getDescribe();
Map<String, Schema.SObjectField> fieldMap = objDescribe.fields.getMap();

for (String fieldName : fieldMap.keySet()) {
    Schema.DescribeFieldResult fieldDescribe = fieldMap.get(fieldName).getDescribe();
    System.debug('Field: ' + fieldName + ', Type: ' + fieldDescribe.getType());
}
```

#### Key Features

- **Custom Objects**: Full database tables with relationships, triggers, validation rules
- **Custom Fields**: Add typed fields to standard or custom objects
- **Field-Level Security**: Control visibility per profile/permission set
- **Validation Rules**: Declarative field validation
- **Triggers**: Before/after hooks in Apex code
- **Formula Fields**: Computed fields based on other fields
- **Roll-Up Summary Fields**: Aggregate child records

#### Key Patterns to Adopt

1. **Naming conventions**: Custom fields end with `__c` to distinguish from standard fields
2. **Validation rules as schema feature**: Declarative validation beyond JSON Schema
3. **Field-level security built-in**: Not just row-level, but column-level permissions
4. **Roll-up summaries**: Aggregate related records declaratively

---

### Jira Custom Fields

**Source**: [Jira Custom Field Types](https://developer.atlassian.com/platform/forge/manifest-reference/modules/jira-custom-field-type/) | [REST API](https://developer.atlassian.com/cloud/jira/platform/rest/v3/api-group-issue-fields/)

#### Core Concept: Extensible Issue Schema with Custom Field Types

Jira exemplifies **runtime schema extension**: issues have base fields (summary, description, status) but administrators add custom fields that become fully searchable, sortable, and reportable.

#### Custom Field Data Types

Built-in types:
- `string` - Text
- `number` - Numeric
- `datetime` - Date with time and timezone
- `date` - Date only
- `user` - Atlassian account reference
- `group` - Group reference
- `object` - Arbitrary JSON with optional schema validation

**Object Type with Schema Validation**:
```javascript
// Apps can store value schemas for object-type fields
{
  "configuration": { ... },
  "schema": {
    "type": "object",
    "properties": {
      "priority": { "type": "integer", "minimum": 1, "maximum": 5 },
      "tags": { "type": "array", "items": { "type": "string" } }
    }
  }
}
```

#### Search Integration

Custom fields are searchable via JQL (Jira Query Language):
```
project = "MYPROJ" AND "Custom Field Name" = "value" ORDER BY "Custom Field Name" DESC
```

**Search Aliases**: For object-type fields, define `searchAlias` properties for fine-grained field searching.

#### Key Patterns to Adopt

1. **Base schema + extensions model**: Core fields are fixed, custom fields extend
2. **Type-aware search**: Custom fields participate in query language
3. **Schema validation for complex types**: JSON Schema for object-type fields
4. **Context-specific configuration**: Fields can have different behavior per project/issue type

---

### Odoo ORM

**Source**: [Odoo ORM API](https://www.odoo.com/documentation/19.0/developer/reference/backend/orm.html)

#### Core Concept: Python ORM with Model Inheritance and Runtime Extension

Odoo's ORM allows **model inheritance** where modules extend existing models with new fields and methods at runtime.

#### Model Inheritance (`_inherit`)

```python
from odoo import models, fields

class ProductExtension(models.Model):
    _inherit = 'product.product'  # Extend existing model

    # Add new fields - creates real database columns
    custom_sku = fields.Char(string='Custom SKU')
    warranty_months = fields.Integer(string='Warranty (Months)')

    # Add new methods
    def calculate_warranty_end(self):
        return self.sale_date + relativedelta(months=self.warranty_months)
```

#### Field Types → SQL Columns

| Odoo Field | SQL Type |
|------------|----------|
| `Char` | `VARCHAR` |
| `Text` | `TEXT` |
| `Integer` | `INTEGER` |
| `Float` | `DOUBLE PRECISION` |
| `Boolean` | `BOOLEAN` |
| `Date` | `DATE` |
| `Datetime` | `TIMESTAMP` |
| `Binary` | `BYTEA` |
| `Many2one` | `INTEGER` (FK) |
| `One2many` | (reverse relation) |
| `Many2many` | (junction table) |

#### Computed Fields

```python
class SaleOrder(models.Model):
    _inherit = 'sale.order'

    total_with_tax = fields.Float(
        string='Total with Tax',
        compute='_compute_total_with_tax',
        store=True  # Stored in database, indexed
    )

    @api.depends('amount_total', 'tax_amount')
    def _compute_total_with_tax(self):
        for order in self:
            order.total_with_tax = order.amount_total + order.tax_amount
```

#### Key Patterns to Adopt

1. **Model inheritance for extension**: New modules extend base schemas without modifying originals
2. **Computed fields with `store=True`**: Calculated fields stored in database for query performance
3. **`@api.depends` for invalidation**: Automatic recomputation when dependencies change
4. **Recordset prefetching**: ORM optimizes field access across multiple records

---

### Payload CMS

**Source**: [Payload CMS](https://payloadcms.com/) | [Collections Docs](https://payloadcms.com/docs/configuration/collections)

#### Core Concept: TypeScript Config → Database Tables + Generated APIs

Payload uses **code-first schema definition** in TypeScript. Collections are TypeScript objects that generate database tables and strongly-typed APIs.

#### Collection Definition

```typescript
import { CollectionConfig } from 'payload/types';

export const Projects: CollectionConfig = {
  slug: 'projects',
  admin: {
    useAsTitle: 'name',
  },
  access: {
    read: () => true,
    create: ({ req: { user } }) => Boolean(user),
    update: ({ req: { user } }) => user?.role === 'admin',
    delete: ({ req: { user } }) => user?.role === 'admin',
  },
  fields: [
    { name: 'name', type: 'text', required: true },
    { name: 'description', type: 'richText' },
    { name: 'budget', type: 'number' },
    { name: 'startDate', type: 'date' },
    { name: 'status', type: 'select', options: ['planning', 'active', 'completed'] },
    { name: 'owner', type: 'relationship', relationTo: 'users' },
    { name: 'tags', type: 'relationship', relationTo: 'tags', hasMany: true },
  ],
  hooks: {
    beforeChange: [validateBudget],
    afterChange: [notifyOwner],
  },
};
```

#### Auto-Generated Artifacts

From the collection config above, Payload generates:
1. **Database table** with typed columns
2. **REST API**: `/api/projects` (CRUD + query)
3. **GraphQL API**: queries and mutations
4. **TypeScript interfaces**: `Project` type for type-safe access
5. **Admin UI**: Form-based editor

#### Hooks System

```typescript
const validateBudget: CollectionBeforeChangeHook = async ({ data, req, operation }) => {
  if (operation === 'create' && data.budget > 1000000) {
    throw new Error('Budget exceeds limit');
  }
  return data;
};

const notifyOwner: CollectionAfterChangeHook = async ({ doc, previousDoc, operation }) => {
  if (operation === 'update' && doc.status !== previousDoc.status) {
    await sendEmail(doc.owner, `Project ${doc.name} status changed to ${doc.status}`);
  }
};
```

#### Key Patterns to Adopt

1. **Config-as-code**: Schema defined in code, not GUI, enabling version control
2. **Field-level access control**: Per-field `access` functions
3. **Generated TypeScript types**: End-to-end type safety
4. **Hooks at collection level**: `beforeChange`, `afterChange`, `beforeDelete`, `afterDelete`

---

### Strapi

**Source**: [Strapi](https://strapi.io/) | [Models Docs](https://docs.strapi.io/cms/backend-customization/models)

#### Core Concept: Content Types → Database Tables + Auto APIs

Strapi generates database tables from content type definitions (stored as JSON/JS files) and auto-generates REST/GraphQL APIs.

#### Content Type Definition

```javascript
// ./src/api/project/content-types/project/schema.json
{
  "kind": "collectionType",
  "collectionName": "projects",
  "info": {
    "singularName": "project",
    "pluralName": "projects",
    "displayName": "Project"
  },
  "attributes": {
    "name": { "type": "string", "required": true },
    "description": { "type": "richtext" },
    "budget": { "type": "decimal" },
    "startDate": { "type": "date" },
    "status": {
      "type": "enumeration",
      "enum": ["planning", "active", "completed"]
    },
    "owner": {
      "type": "relation",
      "relation": "manyToOne",
      "target": "plugin::users-permissions.user"
    },
    "documents": {
      "type": "media",
      "multiple": true
    }
  }
}
```

#### Dynamic Zones (Polymorphic Content)

```javascript
{
  "attributes": {
    "content": {
      "type": "dynamiczone",
      "components": [
        "blocks.hero-section",
        "blocks.feature-grid",
        "blocks.testimonial"
      ]
    }
  }
}
```

#### Schema → Database

Strapi auto-generates database tables from schemas. When schema files change, tables are migrated automatically.

> "Your content blueprint is mapped into tables in the database without worrying about accidental errors in modifying table fields."

#### Key Patterns to Adopt

1. **Schema files as source of truth**: JSON/JS files define structure, database follows
2. **Dynamic zones**: Polymorphic content blocks within a record
3. **Component reuse**: Define reusable field groups across content types

---

### NocoDB

**Source**: [NocoDB](https://nocodb.com/) | [GitHub](https://github.com/nocodb/nocodb)

#### Core Concept: Spreadsheet Interface on Existing SQL Database

NocoDB wraps **existing SQL databases** (MySQL, PostgreSQL, SQLite, SQL Server) with a spreadsheet UI and auto-generated APIs.

#### Key Differentiator: Works with Existing Schemas

Unlike Baserow (which creates its own tables), NocoDB can connect to a database that already has tables and provide:
- Spreadsheet UI for data editing
- REST and GraphQL APIs
- Schema sync when external changes occur

#### API Generation

```
GET    /api/v1/db/data/noco/{project}/{table}
POST   /api/v1/db/data/noco/{project}/{table}
PATCH  /api/v1/db/data/noco/{project}/{table}/{rowId}
DELETE /api/v1/db/data/noco/{project}/{table}/{rowId}
```

#### Field Types

Supports: ID, Links, Lookup, Rollup, SingleLineText, LongText, Attachment, Checkbox, MultiSelect, SingleSelect, Date, Year, Time, PhoneNumber, Email, URL, Number, Decimal, Currency, Percent, Duration, Rating, Formula, Count, DateTime, CreatedTime, LastModifiedTime, CreatedBy, LastModifiedBy, Barcode, QRCode, Button, User

#### Key Patterns to Adopt

1. **Existing database support**: Can wrap pre-existing schemas
2. **Schema sync**: Detect and adapt to external schema changes
3. **Lightweight**: Minimal dependencies, runs as single Node.js app

---

### NocoBase

**Source**: [NocoBase](https://www.nocobase.com/) | [GitHub](https://github.com/nocobase/nocobase)

#### Core Concept: Data Model-First Low-Code Platform

NocoBase uses a **data model-driven approach**: define business objects, relationships, and fields first, then build UIs and workflows on top.

#### Collections and Fields

NocoBase uses "Collections" (tables) and "Fields" (columns) as core abstractions:
- Collections map to database tables
- Fields map to typed columns
- Supports MySQL, PostgreSQL, SQLite as data sources
- Can connect to external databases

#### Plugin Architecture

Everything is a plugin: data sources, field types, actions, UI components, APIs.

> "All functionalities are plugins, similar to WordPress."

#### Key Patterns to Adopt

1. **Data model first**: Define structure before UI
2. **Plugin extensibility**: New field types, actions, etc. via plugins
3. **Multiple data sources**: Connect to external databases

---

### JSON Schema to SQL Libraries

For reference, several libraries convert JSON Schema to SQL DDL:

| Library | Language | Features |
|---------|----------|----------|
| [jsonschema2ddl](https://clarityai-eng.github.io/jsonschema2ddl/) | Python | PostgreSQL/Redshift, handles relationships via `definitions`, type mapping |
| [jsonschema2db](https://github.com/better/jsonschema2db) | Python | Flattens nested JSON into Postgres tables with proper types |
| [json-schema-to-sql](https://github.com/VasilVelikov00/json-schema-to-sql) | JS/TS | Normalizes nested structures into tables with foreign keys |

**Type Mappings** (from jsonschema2ddl):
- `string` → `VARCHAR(256)` (or `maxLength`)
- `string` + `format: date-time` → `TIMESTAMPTZ`
- `string` + `format: date` → `DATE`
- `integer` → `INTEGER`
- `number` → `DOUBLE PRECISION`
- `boolean` → `BOOLEAN`
- `array` → separate table with FK
- `object` → separate table with FK (or inline if simple)

---

### Summary: Feature Comparison Matrix

| Feature | Baserow | Directus | Salesforce | Jira | Odoo | Payload | Strapi | This Proposal |
|---------|---------|----------|------------|------|------|---------|--------|---------------|
| **Schema → Real SQL Columns** | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ (Goal) |
| **Runtime Schema Extension** | ✓ | ✓ | ✓ | ✓ | ✓ | ✗ | ✗ | ✓ (Goal) |
| **Auto-Generated API** | REST | REST + GraphQL | SOAP/REST | REST | XML-RPC/JSON | REST + GraphQL | REST + GraphQL | JSON-RPC 2.0 |
| **Foreign Keys/Relations** | ✓ Link fields | ✓ | ✓ Lookups | ✓ | ✓ Many2one/Many2many | ✓ Relationships | ✓ Relations | ✓ (Planned) |
| **Computed/Formula Fields** | ✓ | ✓ | ✓ | ✓ | ✓ Stored | ✓ Virtual | ✗ | Future |
| **Lifecycle Hooks** | ✗ | ✓ | ✓ Triggers | ✗ | ✓ | ✓ | ✓ | ✓ |
| **Methods on Objects** | ✗ | ✗ | ✓ Apex | ✗ | ✓ | ✓ | ✓ | ✓ (Planned) |
| **Field-Level Permissions** | ✗ | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ | Future |
| **Multi-Tenant Native** | ✓ Workspaces | ✗ | ✓ Orgs | ✓ | ✓ Companies | ✗ | ✗ | ✓ (Secure ORM) |
| **Type-Safe Client Gen** | OpenAPI | OpenAPI | WSDL | OpenAPI | ✗ | TypeScript | TypeScript | GTS |
| **Full-Text Search** | ✗ | ✗ | ✓ SOSL | ✓ JQL | ✗ | ✗ | ✗ | ✓ Index type |

---

### Recommendations Based on Research

#### High Priority (Core Requirements)

1. **Real SQL Columns from JSON Schema**: Follow jsonschema2ddl patterns for type mapping. Don't store as `id + JSONB`—generate actual `VARCHAR`, `INTEGER`, `TIMESTAMP` columns.

2. **Type Management API**: Like Salesforce Metadata API or Baserow's TableHandler:
   ```json
   { "method": "types.register", "params": { "definition": {...} } }
   { "method": "types.extend", "params": { "base_type_id": "...", "extensions": [...] } }
   { "method": "types.list" }
   ```

3. **Field Extension Model** (like Jira): Base type defines core fields, tenants/users can add custom fields that:
   - Become real database columns (not JSONB)
   - Are queryable, sortable, indexable
   - Have proper types (not just strings)

4. **Column Naming Strategy**: Use `field_{id}` pattern (like Baserow) to:
   - Allow any user-facing field names
   - Avoid SQL keyword conflicts
   - Enable field renames without schema migration

5. **Field Converters**: When field types change, handle data migration (like Baserow's converter system).

#### Medium Priority (Enhanced Functionality)

6. **Computed/Stored Fields**: Like Odoo's `compute` with `store=True`—calculated fields that are stored in the database for query performance.

7. **Object Methods**: Allow attaching serverless functions as methods on object types (like Odoo's model methods or Salesforce Apex).

8. **Relationship Resolution**: Like Payload's `relationTo`—declare relationships in schema, auto-resolve in queries.

#### Architecture Decisions

9. **Metadata Tables Pattern**: Store type definitions in metadata tables (`object_types`, `object_fields`), generate models dynamically.

10. **Caching**: Cache generated models/schemas (like Baserow uses Redis) since generation can be expensive for types with many fields.

---

### Naming Recommendation

Given the research, "Extensible Objects" may cause confusion with Cloudflare's actor-based system. Better alternatives:

- **Dynamic Entities** - Emphasizes runtime schema definition (like Salesforce "Custom Objects")
- **Schema Objects** - Emphasizes the GTS/JSON Schema foundation
- **Typed Records** - Emphasizes real SQL types, not document store

The Salesforce terminology of "Custom Objects" and "Custom Fields" is well-understood in enterprise contexts and aligns with the runtime extensibility model.

---

## Decisions (Pending)

1. **Mandatory system columns + scoping model**: Define required columns (e.g., `tenant_id`, `type_id`, `object_id`, `version`, timestamps) for all generated tables and explicitly tie query execution to secure scoping (e.g., Secure ORM).
2. **Optimistic concurrency control**: Specify the mechanism (e.g., `version` field or `ETag`/`If-Match`) and require it in `objects.update`/`objects.delete` to prevent lost updates.
3. **Hook contract requirements**: Make `object_id` mandatory for update/delete events and require `previous_data` for updates; clarify whether hook payloads include metadata fields.
4. **Pagination + cursor semantics**: Define cursor stability, ordering guarantees, and how `orderby` interacts with cursor paging under concurrent writes.
5. **Schema evolution/migrations**: Define migration/backfill behavior when field types or indexes change; specify index rebuild rules and compatibility guarantees.
6. **Post-hook validation + ACL re-checks**: Require JSON Schema validation and field-level access checks after before_* hook data transformations.

## Issues / Gaps

1. **Tenant isolation not enforced in schema spec**: No explicit requirement that generated SQL tables include tenant scope fields, risking cross-tenant access if not mandated.
2. **Update safety**: API examples lack versioning/ETag safeguards for concurrent updates.
3. **Hook ambiguity**: Update/delete hooks may execute without required identifiers, making safe implementation unclear.
4. **Cursor paging ambiguity**: Lack of defined cursor semantics can cause missing/duplicated results under updates.
5. **Schema change risk**: No concrete migration or index-rebuild plan for runtime schema extensions.
6. **Hook security**: Before_* hooks can mutate data without an explicit post-validation/ACL pass.
