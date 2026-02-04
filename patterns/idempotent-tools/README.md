# Idempotent Tools

An idempotent tool produces the same observable result when called once or multiple times with the same inputs. For agents, which retry on failures and timeouts, idempotency prevents duplicate records, double charges, and inconsistent state.

## What Is Idempotency?

A tool is idempotent when:

- Calling it once with inputs X yields result R and leaves the system in state S.
- Calling it again with the same inputs X yields the same result R (or an equivalent one) and leaves the system in the same state S.

Read-only tools are trivially idempotent: they do not change state. Write and destructive tools must be designed so that retries do not create duplicates or apply the same change multiple times.

## Why Idempotency Matters for Agents

Agents retry. When a tool call fails (network error, timeout, 5xx) or the agent is unsure whether it succeeded, it will call the tool again with the same arguments. Without idempotency:

- **Create operations** can insert duplicate rows or resources.
- **Update operations** can double-apply (e.g. increment twice) or overwrite with stale data.
- **Destructive operations** can fail on the second call (e.g. "already deleted") or, worse, have different semantics on retry (e.g. delete the "next" matching resource).

Idempotent design makes retries safe: the agent (or the platform) can retry without special logic, and the system remains consistent.

## When Tools Should Be Idempotent

- **All write tools**: Create, update, and append operations should be idempotent.
- **All destructive tools**: Delete, terminate, and purge should be idempotent (e.g. "already deleted" is success).
- **Read tools**: By definition they do not change state; no extra design needed beyond avoiding hidden side effects.

## Patterns for Idempotency

### 1. Idempotency Key

The caller supplies a unique key (e.g. UUID) for the logical operation. The server stores the key with the result and, on subsequent calls with the same key, returns the stored result without performing the operation again.

```python
def create_order(
    account_id: str,
    product_id: str,
    quantity: int,
    idempotency_key: str | None = None
) -> dict:
    """
    Create an order. If idempotency_key is provided and matches a previous request,
    return the existing order without creating a new one.
    """
    key = idempotency_key or generate_uuid()
    
    existing = db.idempotency.find_one({"key": key})
    if existing:
        return fetch_order_response(existing["order_id"])
    
    order = do_create_order(account_id, product_id, quantity)
    db.idempotency.insert_one({"key": key, "order_id": order["order_id"]})
    
    return fetch_order_response(order["order_id"])
```

**When to use**: One-off create or action operations (orders, payments, notifications). The agent or client generates the key once and reuses it on retry.

### 2. Natural Key or Resource ID

The operation is identified by a stable identifier (resource ID, composite key). If a resource with that identity already exists in the target state, return it; otherwise create or update to that state.

```python
def create_user(user_id: str, email: str, name: str) -> dict:
    """
    Create a user with the given user_id. If user_id already exists, return existing user.
    """
    existing = db.users.find_one({"user_id": user_id})
    if existing:
        return format_user(existing)
    
    user = {"user_id": user_id, "email": email, "name": name, "created_at": datetime.utcnow()}
    db.users.insert_one(user)
    return format_user(user)
```

**When to use**: When the caller can supply or derive a unique ID (e.g. from a workflow or external system). Avoid letting the server generate a new ID on each call if that would create duplicates on retry.

### 3. Set Semantics for Updates

Updates are expressed as "set these fields to these values" rather than "increment" or "append." Same inputs then produce the same final state.

```python
def update_user_profile(user_id: str, name: str | None = None, bio: str | None = None) -> dict:
    """
    Set user profile fields. Omitted fields are unchanged. Same (user_id, name, bio) -> same result.
    """
    updates = {}
    if name is not None:
        updates["name"] = name
    if bio is not None:
        updates["bio"] = bio
    if not updates:
        return get_user(user_id)
    
    db.users.update_one({"user_id": user_id}, {"$set": updates})
    return get_user(user_id)
```

**When to use**: All update operations. Avoid tools that only support "increment by X" or "append X" without an idempotency key or natural key that identifies the logical operation.

### 4. Delete / Destructive: "Already Gone" Is Success

Destructive operations should treat "resource already deleted" or "already in target state" as success and return a consistent response.

```python
def delete_resource(resource_id: str) -> dict:
    """
    Delete a resource. Idempotent: if already deleted, return success and same response shape.
    """
    resource = db.resources.find_one({"resource_id": resource_id})
    
    if not resource:
        return {"resource_id": resource_id, "status": "deleted", "message": "Already deleted"}
    
    db.resources.update_one(
        {"resource_id": resource_id},
        {"$set": {"status": "deleted", "deleted_at": datetime.utcnow()}}
    )
    return {"resource_id": resource_id, "status": "deleted"}
```

**When to use**: All delete, terminate, and purge tools. The second call with the same ID should not fail; it should return the same or equivalent success result.

## Example: Full Idempotent Create

```python
def create_subscription(
    user_id: str,
    plan_id: str,
    idempotency_key: str | None = None
) -> dict:
    """
    Create a subscription. Idempotent via idempotency_key or (user_id, plan_id) when
    an active subscription already exists.
    
    Scope: write
    Idempotent: Yes
    """
    if idempotency_key:
        stored = db.idempotency.find_one({"key": idempotency_key})
        if stored:
            return get_subscription_response(stored["subscription_id"])
    
    existing = db.subscriptions.find_one({
        "user_id": user_id,
        "plan_id": plan_id,
        "status": "active"
    })
    if existing:
        if idempotency_key:
            db.idempotency.insert_one({
                "key": idempotency_key,
                "subscription_id": existing["subscription_id"]
            })
        return get_subscription_response(existing["subscription_id"])
    
    subscription_id = generate_uuid()
    db.subscriptions.insert_one({
        "subscription_id": subscription_id,
        "user_id": user_id,
        "plan_id": plan_id,
        "status": "active",
        "created_at": datetime.utcnow()
    })
    if idempotency_key:
        db.idempotency.insert_one({"key": idempotency_key, "subscription_id": subscription_id})
    
    return get_subscription_response(subscription_id)
```

## Side Effects and Idempotency

Write tools often trigger side effects (email, webhook, billing). Those must be idempotent too, or gated on the same idempotency key or resource state.

- **Send welcome email once**: Check "welcome email sent" flag or idempotency key before sending.
- **Charge once**: Use an idempotency key for the charge; payment provider should deduplicate by that key.
- **Webhook once**: Associate the webhook call with the same idempotency key or resource version so retries do not re-send.

If a side effect cannot be made idempotent, perform it only after the main operation is committed and consider a separate, idempotent "dispatch" step that records that the side effect was triggered.

## Anti-Pattern: Non-Idempotent Create

```python
# BAD: Every call creates a new order
def create_order(account_id: str, product_id: str, quantity: int) -> dict:
    order_id = generate_uuid()  # New ID every time
    db.orders.insert_one({
        "order_id": order_id,
        "account_id": account_id,
        "product_id": product_id,
        "quantity": quantity,
        "created_at": datetime.utcnow()
    })
    return {"order_id": order_id}
```

**Why it's dangerous**: A single logical "place order" from the user can become multiple orders when the agent retries or the client retries. Duplicate orders, duplicate charges, and inconsistent state.

**Correct approach**: Accept an idempotency key (or natural key) and return the existing order when the key has already been used.

## Anti-Pattern: Increment Without Key

```python
# BAD: Each retry increments again
def increment_view_count(resource_id: str) -> dict:
    db.resources.update_one(
        {"resource_id": resource_id},
        {"$inc": {"view_count": 1}}
    )
    return get_resource(resource_id)
```

**Why it's dangerous**: Retries inflate the counter. The same logical "view" is counted multiple times.

**Correct approach**: Either make the operation a no-op for the agent (e.g. "record view" is not an agent tool) or use an idempotency key so "record view for key K" is applied only once.

## Documenting Idempotency

In the tool description and docstring, state clearly:

- Whether the tool is idempotent.
- What identifies the logical operation (idempotency key, resource ID, or "same inputs").
- Any caveats (e.g. "Idempotent for 24 hours" if keys expire).

Example:

```python
def create_export(job_id: str, format: str) -> dict:
    """
    Create an export job.
    
    Scope: write
    Idempotent: Yes. Same (job_id, format) returns existing job if still valid.
    """
```

## Summary

Idempotent tools return the same result for the same inputs across multiple calls, which is essential when agents or clients retry. Use idempotency keys or natural keys for creates, set semantics for updates, and "already gone = success" for destructive operations. Make side effects idempotent or trigger them in a single, keyed step. Document idempotency in the tool contract so callers can rely on safe retries.
