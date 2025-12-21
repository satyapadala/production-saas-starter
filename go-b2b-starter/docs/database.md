# Database Guide

The database layer uses PostgreSQL with SQLC for type-safe SQL operations and the Adapter Pattern to keep SQLC isolated from business logic.

## Architecture

The database layer has three components:

**1. Store Interfaces** (`internal/db/adapters/`) - Contracts for database operations
**2. Store Adapters** (`internal/db/postgres/adapter_impl/`) - Implement interfaces using SQLC
**3. SQLC Generated Code** (`internal/db/postgres/sqlc/gen/`) - Auto-generated from SQL queries

### Why Use Adapters?

- External modules depend on **interfaces**, not SQLC directly
- Easy to mock for testing
- Can swap database implementations
- SQLC internals stay contained

## SQLC Workflow

### 1. Write SQL Query

Create queries in `internal/db/postgres/sqlc/query/{domain}.sql`:

```sql
-- name: GetResourceByID :one
SELECT * FROM resources WHERE id = $1;

-- name: CreateResource :one
INSERT INTO resources (name, status)
VALUES ($1, $2)
RETURNING *;

-- name: ListResources :many
SELECT * FROM resources
ORDER BY created_at DESC
LIMIT $1 OFFSET $2;
```

**SQLC Annotations:**
- `:one` - Returns single row
- `:many` - Returns slice of rows
- `:exec` - Returns error only (no data)

### 2. Generate Code

```bash
make sqlc
```

Generates Go code in `internal/db/postgres/sqlc/gen/`.

**Never edit generated files** - they are regenerated on every run.

### 3. Create Store Interface

Define interface in `internal/db/adapters/resource_store.go`:

```go
type ResourceStore interface {
    GetResourceByID(ctx context.Context, id int32) (sqlc.Resource, error)
    CreateResource(ctx context.Context, arg sqlc.CreateResourceParams) (sqlc.Resource, error)
    ListResources(ctx context.Context, arg sqlc.ListResourcesParams) ([]sqlc.Resource, error)
}
```

### 4. Implement Adapter

Create adapter in `internal/db/postgres/adapter_impl/resource_store.go`:

```go
type resourceStore struct {
    store sqlc.Store
}

func NewResourceStore(store sqlc.Store) adapters.ResourceStore {
    return &resourceStore{store: store}
}

func (s *resourceStore) GetResourceByID(ctx context.Context, id int32) (sqlc.Resource, error) {
    return s.store.GetResourceByID(ctx, id)
}
```

### 5. Register in DI

Add to `internal/db/inject.go`:

```go
container.Provide(func(sqlcStore sqlc.Store) adapters.ResourceStore {
    return adapter_impl.NewResourceStore(sqlcStore)
})
```

## Database Migrations

### File Structure

Migrations live in `internal/db/postgres/sqlc/migrations/`:

```
000001_create_schema.up.sql
000001_create_schema.down.sql
000002_add_indexes.up.sql
000002_add_indexes.down.sql
```

### Naming Convention

Format: `{6-digit-number}_{description}.{up|down}.sql`

- `.up.sql` - Apply the migration
- `.down.sql` - Rollback the migration

### Example Migration

**Up migration** (`000005_create_resources.up.sql`):

```sql
CREATE SCHEMA IF NOT EXISTS app;

CREATE TABLE app.resources (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255) NOT NULL,
    status VARCHAR(50) NOT NULL,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_resources_status ON app.resources(status);
```

**Down migration** (`000005_create_resources.down.sql`):

```sql
DROP TABLE IF EXISTS app.resources;
DROP SCHEMA IF EXISTS app;
```

### Running Migrations

```bash
make migrateup      # Apply all pending migrations
make migratedown    # Rollback last migration
```

## Type Conversions

PostgreSQL types need conversion to Go types.

### Nullable Fields

SQLC uses `pgtype` for nullable fields:

```go
// Convert pgtype.Text to string
str := postgres.StringFromPgText(dbRecord.NullableField)

// Convert string to pgtype.Text
pgText := postgres.ToPgText(str)

// Convert pgtype.Int4 to int32
num := postgres.Int32FromPgInt4(dbRecord.NullableInt)
```

Helper functions in `internal/db/postgres/types_transform.go`.

### JSONB Fields

```go
// Convert map to JSONB
jsonbData := postgres.ToJSONB(map[string]any{"key": "value"})

// Convert JSONB to map
data := postgres.JSONBToMap(dbRecord.Metadata)
```

## Error Handling

The database layer provides specific error types in `internal/db/core/errors.go`:

**Common Errors:**
- `ErrNoRows` - Query returned no results
- `ErrTxClosed` - Transaction already committed/rolled back
- `ErrTimeout` - Operation exceeded timeout
- `ErrPoolClosed` - Connection pool is closed

**Helper Functions:**

```go
if core.IsNoRowsError(err) {
    return domain.ErrResourceNotFound
}

if core.IsConstraintError(err, "unique_name") {
    return domain.ErrResourceAlreadyExists
}

if core.IsTimeoutError(err) {
    return domain.ErrDatabaseTimeout
}
```

## Transactions

Use transactions for multi-step operations that must be atomic.

### Basic Transaction

```go
func (r *repository) CreateWithRelation(ctx context.Context, resource *domain.Resource) error {
    return r.db.WithTx(ctx, func(tx core.Transaction) error {
        // Step 1: Create resource
        created, err := tx.CreateResource(ctx, params)
        if err != nil {
            return err
        }

        // Step 2: Create relation
        _, err = tx.CreateRelation(ctx, relationParams)
        if err != nil {
            return err // Transaction auto-rolls back on error
        }

        return nil // Transaction commits on success
    })
}
```

### Transaction Options

```go
// Read-only transaction
err := r.db.WithTxOptions(ctx, &sql.TxOptions{ReadOnly: true}, func(tx core.Transaction) error {
    // Read operations only
})

// Custom isolation level
err := r.db.WithTxOptions(ctx, &sql.TxOptions{
    Isolation: sql.LevelSerializable,
}, func(tx core.Transaction) error {
    // Operations
})
```

## Best Practices

### Always Use Context

```go
// ✅ Good
func (r *repository) GetResource(ctx context.Context, id int32) (*Resource, error)

// ❌ Bad
func (r *repository) GetResource(id int32) (*Resource, error)
```

### Handle Errors Appropriately

```go
// ✅ Convert database errors to domain errors
resource, err := r.store.GetResourceByID(ctx, id)
if err != nil {
    if core.IsNoRowsError(err) {
        return nil, domain.ErrResourceNotFound
    }
    return nil, fmt.Errorf("failed to get resource: %w", err)
}
```

### Use Prepared Statements

SQLC automatically creates prepared statements. Never concatenate SQL strings.

```go
// ✅ Good (SQLC handles this)
SELECT * FROM resources WHERE name = $1

// ❌ Bad (SQL injection risk)
query := fmt.Sprintf("SELECT * FROM resources WHERE name = '%s'", name)
```

### Indexes for Performance

Add indexes for commonly queried fields:

```sql
-- Foreign keys
CREATE INDEX idx_resources_org_id ON resources(organization_id);

-- Status fields
CREATE INDEX idx_resources_status ON resources(status);

-- Timestamps for sorting
CREATE INDEX idx_resources_created_at ON resources(created_at DESC);

-- Composite indexes for multi-column queries
CREATE INDEX idx_resources_org_status ON resources(organization_id, status);
```

## File Locations

| Component | Path |
|-----------|------|
| Store interfaces | `internal/db/adapters/` |
| Store implementations | `internal/db/postgres/adapter_impl/` |
| SQL queries | `internal/db/postgres/sqlc/query/` |
| Migrations | `internal/db/postgres/sqlc/migrations/` |
| Generated code | `internal/db/postgres/sqlc/gen/` |
| Type helpers | `internal/db/postgres/types_transform.go` |
| Error types | `internal/db/core/errors.go` |
| DI setup | `internal/db/inject.go` |

## Next Steps

- **Using in repositories**: See [Architecture Guide](./architecture.md)
- **Building APIs**: See [API Development Guide](./api-development.md)
- **SQLC documentation**: https://docs.sqlc.dev/
