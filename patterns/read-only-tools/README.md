# Read-Only Tools

Read-only tools are the safest category of MCP tools. They retrieve information without modifying system state, making them ideal for agents that need to explore, validate, or gather context before taking action.

## What is a Read-Only Tool?

A read-only tool performs queries or fetches data without creating, updating, or deleting anything. It has no side effects beyond logging and resource consumption. Examples include: fetching user profiles, listing files in a directory, querying database records, or retrieving API status.

## Why Read-Only Tools Are Safest

Read-only tools are the foundation of safe agent behavior because:

1. **No state changes**: Agents can call them repeatedly without risk
2. **Exploration-friendly**: Agents can discover system state before making decisions
3. **Low permission requirements**: They require only `read` scope, reducing attack surface
4. **Easier to audit**: Read operations are logged but don't require rollback planning

When designing agent capabilities, prefer read-only tools whenever possible. Only add write or destructive tools when read-only operations are insufficient.

## When to Use Read-Only Tools

Use read-only tools for:
- Data retrieval and queries
- Status checks and health monitoring
- Validation (checking if a resource exists before writing)
- Discovery (listing available resources)
- Reporting and analytics

Avoid read-only tools for operations that appear read-only but have side effects (e.g., incrementing view counters, triggering webhooks, or warming caches).

## Example: Well-Designed Read-Only Tool

```python
def get_user_profile(user_id: str) -> dict:
    """
    Retrieve a user's profile information.
    
    Scope: read
    Idempotent: Yes
    Side effects: None
    
    Args:
        user_id: Unique identifier for the user (UUID format)
    
    Returns:
        {
            "user_id": str,
            "email": str,
            "created_at": str (ISO 8601),
            "last_login": str (ISO 8601) | None,
            "status": "active" | "suspended" | "deleted"
        }
    
    Raises:
        NotFoundError: If user_id does not exist
        ValidationError: If user_id format is invalid
    """
    # Validate input format
    if not is_valid_uuid(user_id):
        raise ValidationError(f"Invalid user_id format: {user_id}")
    
    # Fetch from database
    user = db.users.find_one({"user_id": user_id})
    
    if not user:
        raise NotFoundError(f"User {user_id} not found")
    
    # Return structured data
    return {
        "user_id": user["user_id"],
        "email": user["email"],
        "created_at": user["created_at"].isoformat(),
        "last_login": user.get("last_login").isoformat() if user.get("last_login") else None,
        "status": user["status"]
    }
```

## Common Mistakes

### Mistake 1: Implicit Side Effects

```python
# BAD: Increments view counter, not truly read-only
def get_user_profile(user_id: str):
    user = db.users.find_one({"user_id": user_id})
    db.users.update_one({"user_id": user_id}, {"$inc": {"views": 1}})  # Side effect!
    return user
```

**Why it's dangerous**: Agents will call read-only tools frequently during exploration. Incrementing counters on every call creates unintended state changes and can degrade performance.

### Mistake 2: Accepting Arbitrary Queries

```python
# BAD: Allows arbitrary SQL queries
def query_database(sql: str):
    return db.execute(sql)  # Dangerous!
```

**Why it's dangerous**: Agents can construct queries that modify data, access unauthorized tables, or cause performance issues. Even with read-only database permissions, this pattern is unsafe.

### Mistake 3: Unstructured Returns

```python
# BAD: Returns raw database objects
def get_user_profile(user_id: str):
    return db.users.find_one({"user_id": user_id})  # May include sensitive fields
```

**Why it's dangerous**: Raw database objects may include internal IDs, timestamps, or sensitive fields that shouldn't be exposed to agents. Always return explicitly structured data.

## Best Practices

1. **Explicit return structures**: Define the exact shape of returned data
2. **Input validation**: Validate all inputs before processing
3. **Clear error messages**: Return specific errors (NotFound, ValidationError) rather than generic failures
4. **No side effects**: Ensure the tool truly has no state-changing operations
5. **Scope requirement**: Always require `read` scope, never `write` or `destructive`
