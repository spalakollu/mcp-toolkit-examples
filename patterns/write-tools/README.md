# Write Tools

Write tools modify system state by creating, updating, or appending data. They require stricter controls than read-only tools because incorrect calls can corrupt data, create duplicates, or trigger cascading failures.

## What Are Write Tools?

Write tools perform operations that change system state: creating records, updating fields, uploading files, or modifying configurations. Unlike read-only tools, they require `write` scope and must be designed with idempotency and validation in mind.

## Why Write Tools Require Stricter Controls

Write tools introduce risk because:

1. **State changes are permanent**: Unlike reads, writes modify data that may be difficult to reverse
2. **Cascading effects**: A write operation may trigger downstream processes (notifications, workflows, billing)
3. **Agent retries**: Agents will retry failed operations, potentially creating duplicates
4. **Permission escalation**: Write access is a higher privilege level than read

Every write tool should be idempotent, validate inputs strictly, and require explicit `write` scope.

## Designing Idempotent Write Tools

An idempotent tool produces the same result whether called once or multiple times with the same inputs. This is critical for agents, which will retry operations on errors.

**Idempotent pattern**: Use unique identifiers (UUIDs, composite keys) to detect and handle duplicates.

```python
def create_user(email: str, name: str, user_id: str | None = None) -> dict:
    """
    Create a new user account.
    
    Scope: write
    Idempotent: Yes (if user_id provided)
    Side effects: Creates database record, sends welcome email
    
    Args:
        email: User email address (must be unique)
        name: User display name
        user_id: Optional UUID. If provided and user exists, returns existing user.
    
    Returns:
        {
            "user_id": str,
            "email": str,
            "name": str,
            "created_at": str (ISO 8601)
        }
    """
    # Generate ID if not provided
    if not user_id:
        user_id = generate_uuid()
    
    # Check if user already exists (idempotency check)
    existing = db.users.find_one({"user_id": user_id})
    if existing:
        return {
            "user_id": existing["user_id"],
            "email": existing["email"],
            "name": existing["name"],
            "created_at": existing["created_at"].isoformat()
        }
    
    # Validate email uniqueness
    if db.users.find_one({"email": email}):
        raise ConflictError(f"User with email {email} already exists")
    
    # Create user
    user = {
        "user_id": user_id,
        "email": email,
        "name": name,
        "created_at": datetime.utcnow()
    }
    db.users.insert_one(user)
    
    # Send welcome email (idempotent: checks if already sent)
    send_welcome_email(user_id, email)
    
    return {
        "user_id": user["user_id"],
        "email": user["email"],
        "name": user["name"],
        "created_at": user["created_at"].isoformat()
    }
```

## Example: Well-Designed Write Tool

```python
def update_user_profile(user_id: str, updates: dict) -> dict:
    """
    Update a user's profile fields.
    
    Scope: write
    Idempotent: Yes
    Side effects: Updates database record
    
    Args:
        user_id: Unique identifier for the user
        updates: Dictionary of fields to update. Allowed keys:
            - name: str
            - bio: str
            - avatar_url: str
    
    Returns:
        Updated user profile (same structure as get_user_profile)
    """
    # Validate user_id exists
    user = db.users.find_one({"user_id": user_id})
    if not user:
        raise NotFoundError(f"User {user_id} not found")
    
    # Validate allowed update fields
    allowed_fields = {"name", "bio", "avatar_url"}
    invalid_fields = set(updates.keys()) - allowed_fields
    if invalid_fields:
        raise ValidationError(f"Invalid fields: {invalid_fields}")
    
    # Validate field types
    if "name" in updates and not isinstance(updates["name"], str):
        raise ValidationError("name must be a string")
    
    # Apply updates (idempotent: same updates = same result)
    db.users.update_one(
        {"user_id": user_id},
        {"$set": updates, "$setOnInsert": {"updated_at": datetime.utcnow()}}
    )
    
    # Return updated profile
    updated_user = db.users.find_one({"user_id": user_id})
    return format_user_profile(updated_user)
```

## Example: Poorly-Designed Write Tool

```python
# BAD: Not idempotent, no validation, accepts arbitrary fields
def update_user(user_id: str, **kwargs):
    """
    Update user with any fields.
    """
    db.users.update_one({"user_id": user_id}, {"$set": kwargs})
    return db.users.find_one({"user_id": user_id})
```

**Why this is dangerous**:

1. **Not idempotent**: Multiple calls with the same data may produce different results if timestamps or random values are included
2. **No input validation**: Agents can set internal fields like `is_admin`, `balance`, or `deleted_at`
3. **No existence check**: Fails silently if user doesn't exist
4. **Arbitrary field updates**: No control over what can be modified

**What can go wrong**:
- Agent retries create inconsistent state
- Agent sets `is_admin=True` by including it in kwargs
- Agent updates `deleted_at` thinking it's a normal field
- No error when user_id is invalid

## Scoping Write Tools Correctly

All write tools must require `write` scope. Never allow write operations with `read` scope, even if they seem harmless.

```python
# Tool definition in MCP server
{
    "name": "update_user_profile",
    "description": "Update user profile fields",
    "inputSchema": {
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "updates": {"type": "object"}
        }
    },
    "requiredScopes": ["write"]  # Explicit scope requirement
}
```

## Best Practices

1. **Idempotency**: Design tools to be safely retriable
2. **Input validation**: Validate all inputs, including field names and types
3. **Explicit allowed fields**: Never accept arbitrary field updates
4. **Existence checks**: Verify resources exist before updating
5. **Structured returns**: Return consistent data structures
6. **Scope enforcement**: Require `write` scope for all state-changing operations
