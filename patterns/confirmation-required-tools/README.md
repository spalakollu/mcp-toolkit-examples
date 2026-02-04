# Confirmation-Required Tools

Confirmation-required tools are tools that must not execute a sensitive operation until a human or an explicit confirmation step has approved it. The tool design enforces this at the API level rather than relying on the agent to "ask the user" in natural language.

## What Are Confirmation-Required Tools?

A confirmation-required tool is one that:

- Accepts a parameter or token that represents explicit approval (e.g. `confirm=True`, `confirmation_token`, or an approval request ID)
- Refuses to perform the sensitive operation when that approval is missing or invalid
- Optionally returns a preview or description of what would happen so a human can decide

The key is that execution is gated by the tool implementation and the MCP server, not by the agent's behavior. An agent cannot bypass confirmation by omitting a step in its reasoning.

## When to Require Confirmation

Use confirmation-required design for:

- **Destructive operations**: Delete, terminate, purge, cancel. These are almost always confirmation-required and use `destructive` scope.
- **High-impact write operations**: Bulk updates, schema changes, role or permission changes, or any write that affects many resources or critical data.
- **Production or sensitive environments**: Operations that target production, billing, or compliance-sensitive data even if they are "just" writes.
- **Irreversible or hard-to-reverse actions**: Anything that cannot be undone or would require manual recovery.

Do not require confirmation for routine, low-impact writes (e.g. updating a single non-critical field) unless your policy explicitly does so. Overuse of confirmation degrades usability and can train users to confirm blindly.

## Confirmation Mechanisms

### 1. Explicit Boolean Parameter

The tool accepts a parameter (e.g. `confirm`) that must be set to `True` to execute. Default is `False`. The agent can call with `confirm=False` to get a preview; execution requires a second call with `confirm=True`.

```python
def liquidate_account(account_id: str, confirm: bool = False) -> dict:
    """
    Liquidate all positions and close account. Requires confirm=True to execute.
    Scope: destructive
    """
    if not confirm:
        return {
            "preview": True,
            "account_id": account_id,
            "positions_to_close": get_position_count(account_id),
            "message": "Set confirm=True to execute."
        }
    # ... perform liquidation
```

**Pros**: Simple, no separate approval system. **Cons**: Agent could call with `confirm=True` autonomously; suitable when combined with destructive scope and policy that agents do not receive that scope without human approval.

### 2. Confirmation Token

The tool requires a token that was issued by a separate approval flow (e.g. human clicks "Approve" in a UI, which creates a short-lived token). The agent cannot execute without that token.

```python
def delete_user_account(user_id: str, confirmation_token: str) -> dict:
    """
    Permanently delete user account. Requires valid confirmation_token from approval API.
    Scope: destructive
    """
    if not validate_confirmation_token(confirmation_token, operation="delete_user", target_id=user_id):
        raise AuthorizationError("Invalid or expired confirmation token")
    # ... perform deletion
```

**Pros**: Strong guarantee; execution is impossible until a human (or approved system) has issued the token. **Cons**: Requires an approval UI or API and token lifecycle.

### 3. Approval Request ID

The agent calls a tool to *request* an operation; the tool creates a pending approval and returns an ID. A separate tool (or out-of-band process) allows a human to approve or reject. The agent can later call an "execute approved request" tool with that ID.

```python
def request_account_closure(account_id: str) -> dict:
    """Create a pending closure request. Returns request_id for approval."""
    request_id = create_pending_request("close_account", {"account_id": account_id})
    return {"request_id": request_id, "status": "pending_approval"}

def execute_approved_request(request_id: str) -> dict:
    """Execute a previously approved request. Fails if not approved."""
    request = get_request(request_id)
    if request["status"] != "approved":
        raise AuthorizationError("Request not approved")
    # ... execute based on request type
```

**Pros**: Fits workflows where approval is asynchronous or reviewed by a team. **Cons**: More moving parts; requires clear documentation so agents know to wait for approval before calling execute.

## Design Rules

1. **Enforce in the tool**: Do not document "the agent should ask the user" and leave enforcement to the agent. The tool must refuse to perform the operation when confirmation is missing or invalid.

2. **Default to safe**: The default or missing value for the confirmation parameter should mean "do not execute." Prefer `confirm: bool = False` and require `confirm=True`, or require a token/request ID that cannot be forged by the agent.

3. **Return a clear preview when unconfirmed**: When the user has not confirmed, return a structured preview (what would be affected, counts, impact) and a clear message that confirmation is required. This supports human-in-the-loop without executing.

4. **Scope and confirmation together**: Confirmation-required tools are typically `destructive` scope. Some high-impact write tools may use `write` scope but still require a confirmation parameter or token. Document both the scope and the confirmation requirement in the tool description.

5. **Idempotency**: Design the operation so that if the agent retries after confirmation, the result is safe (idempotent). Avoid requiring multiple confirmations for the same logical operation.

## Example: Confirmation with Preview

```python
def bulk_update_user_status(
    user_ids: list[str],
    new_status: str,
    confirm: bool = False
) -> dict:
    """
    Set status for multiple users. Requires confirm=True.
    Scope: write (high-impact; confirmation required by policy)
    
    Args:
        user_ids: Up to 100 user IDs
        new_status: "active" | "suspended"
        confirm: Must be True to execute
    """
    if len(user_ids) > 100:
        raise ValidationError("Maximum 100 users per request")
    
    if new_status not in ("active", "suspended"):
        raise ValidationError("new_status must be 'active' or 'suspended'")
    
    if not confirm:
        existing = list(db.users.find({"user_id": {"$in": user_ids}}, {"user_id": 1}))
        found_ids = [u["user_id"] for u in existing]
        return {
            "preview": True,
            "user_count": len(found_ids),
            "user_ids": found_ids,
            "new_status": new_status,
            "message": "Set confirm=True to apply."
        }
    
    result = db.users.update_many(
        {"user_id": {"$in": user_ids}},
        {"$set": {"status": new_status, "updated_at": datetime.utcnow()}}
    )
    return {
        "preview": False,
        "matched": result.matched_count,
        "modified": result.modified_count,
        "new_status": new_status
    }
```

## Anti-Pattern: Relying on the Agent to Confirm

```python
# BAD: No enforcement; documentation only
def delete_resource(resource_id: str):
    """
    Delete a resource. The agent should ask the user for confirmation before calling.
    """
    return db.resources.delete_one({"resource_id": resource_id})
```

**Why it's unsafe**: The agent may not ask, may be instructed to skip confirmation, or may be tricked by prompt injection. There is no technical barrier to autonomous execution.

**Correct approach**: Add a confirmation parameter or token that the tool validates. The tool must not perform the operation when confirmation is missing or invalid.

## Summary

Confirmation-required tools gate sensitive operations on explicit approval (boolean, token, or approval request ID) and enforce that gate in the tool implementation. Use them for destructive and high-impact write operations. Default to safe behavior, return useful previews when unconfirmed, and combine with appropriate scopes so that agents cannot perform these operations without both permission and confirmation.
