# Database Example

This example demonstrates how to design safe MCP tools for a database domain. **No real database driver or connection logic is included.** The tool signatures and patterns are illustrative. The critical principle is to never expose raw SQL or arbitrary queries to agents; use structured, parameterized tools with allowlisted tables and columns.

## Domain Overview

A database domain allows agents to:
- Query data from allowed tables with structured filters (read)
- Insert or update records in allowed tables (write)
- Truncate tables or perform bulk deletes (destructive)

This domain is especially prone to unsafe design (e.g., "execute_sql"). Safe design requires allowlisted tables, explicit column sets, bounded result sets, and strict separation of read, write, and destructive operations.

## Tool Design

### Read Tool: Query Table

```python
# Allowlisted tables and queryable columns (configured per environment)
QUERYABLE_TABLES = {
    "users": ["user_id", "email", "name", "created_at", "status"],
    "orders": ["order_id", "user_id", "total_cents", "created_at", "status"],
}

def query_table(
    table_name: str,
    filters: dict,
    columns: list[str] | None = None,
    limit: int = 100
) -> dict:
    """
    Query rows from an allowed table with structured filters.
    
    Scope: read
    Idempotent: Yes
    Side effects: None
    
    Args:
        table_name: Name of the table (must be in allowlist)
        filters: Key-value equality filters, e.g. {"status": "active", "user_id": "uuid"}
        columns: Subset of allowlisted columns to return (default: all queryable)
        limit: Maximum rows to return (1-1000, default 100)
    
    Returns:
        {
            "table": str,
            "rows": list[dict],
            "count": int,
            "limit": int
        }
    
    Raises:
        ValidationError: If table or columns not allowed, or limit out of range
    """
    if table_name not in QUERYABLE_TABLES:
        raise ValidationError(f"Table not queryable: {table_name}. Allowed: {list(QUERYABLE_TABLES)}")
    
    allowed_cols = set(QUERYABLE_TABLES[table_name])
    if columns is not None:
        invalid = set(columns) - allowed_cols
        if invalid:
            raise ValidationError(f"Columns not allowed: {invalid}")
        selected = columns
    else:
        selected = QUERYABLE_TABLES[table_name]
    
    if not 1 <= limit <= 1000:
        raise ValidationError(f"limit must be 1-1000, got {limit}")
    
    # Build query from filters (no raw SQL from agent)
    query = {}
    for key, value in filters.items():
        if key not in allowed_cols:
            raise ValidationError(f"Filter column not allowed: {key}")
        query[key] = value
    
    projection = {c: 1 for c in selected}
    cursor = db[table_name].find(query, projection).limit(limit)
    rows = [serialize_row(r) for r in cursor]
    
    return {
        "table": table_name,
        "rows": rows,
        "count": len(rows),
        "limit": limit
    }
```

**Why this design is safe**:
- No raw SQL; table and columns are allowlisted
- Filters are key-value only; no arbitrary expressions or subqueries
- Bounded result set (limit) to avoid unbounded reads
- Read-only; no state changes
- Requires only read scope

### Write Tool: Insert Record

```python
INSERTABLE_TABLES = {
    "users": ["email", "name"],
    "orders": ["user_id", "total_cents", "status"],
}

def insert_record(
    table_name: str,
    record: dict,
    idempotency_key: str | None = None
) -> dict:
    """
    Insert a single record into an allowed table.
    
    Scope: write
    Idempotent: Yes (if idempotency_key provided and record exists)
    Side effects: One new row
    
    Args:
        table_name: Table to insert into (must be in allowlist)
        record: Key-value pairs for allowlisted columns only
        idempotency_key: Optional UUID to prevent duplicate inserts
    
    Returns:
        {
            "table": str,
            "id": str (e.g. primary key or generated ID),
            "created_at": str (ISO 8601)
        }
    
    Raises:
        ValidationError: If table or record fields are invalid
        ConflictError: If unique constraint violated (e.g. duplicate email)
    """
    if table_name not in INSERTABLE_TABLES:
        raise ValidationError(f"Table not insertable: {table_name}")
    
    allowed = set(INSERTABLE_TABLES[table_name])
    invalid_keys = set(record.keys()) - allowed
    if invalid_keys:
        raise ValidationError(f"Fields not allowed: {invalid_keys}")
    
    # Idempotency
    if idempotency_key:
        existing = db.idempotency.find_one({"key": idempotency_key, "table": table_name})
        if existing:
            return {
                "table": table_name,
                "id": existing["record_id"],
                "created_at": existing["created_at"].isoformat()
            }
    
    # Generate ID and timestamps server-side
    record_id = generate_uuid()
    now = datetime.utcnow()
    full_record = {
        **record,
        "id": record_id,
        "created_at": now,
        "updated_at": now
    }
    
    try:
        db[table_name].insert_one(full_record)
    except DuplicateKeyError as e:
        raise ConflictError(f"Unique constraint violated: {e}")
    
    if idempotency_key:
        db.idempotency.insert_one({
            "key": idempotency_key,
            "table": table_name,
            "record_id": record_id,
            "created_at": now
        })
    
    return {
        "table": table_name,
        "id": record_id,
        "created_at": now.isoformat()
    }
```

**Why this design is safe**:
- Only allowlisted tables and columns can be written
- Idempotency key prevents duplicate inserts on retries
- IDs and timestamps set server-side; agent cannot inject internal fields
- Single-record scope limits blast radius
- Requires write scope

### Destructive Tool: Truncate Table

```python
TRUNCATABLE_TABLES = {"staging_imports", "audit_temp"}  # Explicit allowlist; exclude critical tables

def truncate_table(table_name: str, confirm: bool = False) -> dict:
    """
    Truncate an allowed table (delete all rows).
    
    Scope: destructive
    Idempotent: Yes (empty table remains empty)
    Side effects: All rows in table removed
    
    Args:
        table_name: Table to truncate (must be in truncation allowlist)
        confirm: Must be True to execute
    
    Returns:
        {
            "table": str,
            "executed": bool,
            "rows_deleted": int,
            "message": str (if not executed)
        }
    
    Raises:
        ValidationError: If table not in truncation allowlist or confirm is False
    """
    if table_name not in TRUNCATABLE_TABLES:
        raise ValidationError(
            f"Table not truncatable: {table_name}. "
            f"Only staging/utility tables are allowed: {TRUNCATABLE_TABLES}"
        )
    
    count = db[table_name].count_documents({})
    
    if not confirm:
        return {
            "table": table_name,
            "executed": False,
            "rows_deleted": 0,
            "message": f"Would delete {count} rows. Set confirm=True to truncate."
        }
    
    db[table_name].delete_many({})
    
    return {
        "table": table_name,
        "executed": True,
        "rows_deleted": count
    }
```

**Why this design is safe**:
- Only explicitly allowlisted tables can be truncated; production tables are excluded
- confirm=True required to execute
- Idempotent for empty table
- Requires destructive scope
- Returns row count for audit

## What Not to Do

Do not expose tools that accept raw SQL or arbitrary table/column names:

```python
# BAD: Never do this
def execute_sql(sql: str):
    return db.execute(sql)

# BAD: Arbitrary table/column access
def query_anything(table: str, where: str):
    return db.execute(f"SELECT * FROM {table} WHERE {where}")
```

Such tools allow agents (or prompt injection) to read arbitrary data, modify or drop tables, and bypass scope design. Use structured, allowlisted tools only.

## How Scopes Protect

- **Read only**: Agent can query allowed tables with bounded results; no inserts or truncation.
- **Read + write**: Agent can query and insert/update via structured tools; still cannot truncate or drop.
- **Destructive**: Agent can truncate only allowlisted staging/utility tables, and only when confirm=True.

## Summary

The database example shows:
- **Read**: Structured query tool with allowlisted tables, columns, and filters and a strict limit
- **Write**: Insert (and optionally update) with allowlisted tables/columns and idempotency
- **Destructive**: Truncate only allowlisted tables with explicit confirmation

Never expose raw SQL or arbitrary schema access. Use allowlists and structured parameters so scopes and audit trails remain meaningful.
