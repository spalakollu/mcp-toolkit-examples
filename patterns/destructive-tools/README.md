# Destructive Tools

Destructive tools permanently delete data, terminate resources, or perform irreversible operations. Agents should almost never call them autonomously. These tools require human confirmation, dry-run capabilities, and explicit `destructive` scope.

## What Are Destructive Tools?

Destructive tools perform operations that cannot be easily undone: deleting database records, terminating cloud instances, canceling subscriptions, or purging logs. Once executed, recovery may be impossible or require manual intervention.

## Why Agents Should Almost Never Call Them Autonomously

Destructive operations are dangerous for autonomous agents because:

1. **Irreversible consequences**: Deletion is permanent; recovery may be impossible
2. **Cascading failures**: Deleting one resource may break dependent systems
3. **Agent errors**: Agents make mistakes; a typo in a resource ID can delete the wrong thing
4. **Prompt injection**: Malicious inputs can trick agents into destructive actions

**Rule**: Destructive tools should require explicit human confirmation or be restricted to specific, high-confidence scenarios with extensive safeguards.

## Human-in-the-Loop Patterns

### Pattern 1: Confirmation Required

The tool requires a confirmation token that must be obtained from a human operator.

```python
def delete_user_account(user_id: str, confirmation_token: str) -> dict:
    """
    Permanently delete a user account and all associated data.
    
    Scope: destructive
    Idempotent: Yes (returns success if already deleted)
    Side effects: Deletes user record, anonymizes logs, cancels subscriptions
    
    Args:
        user_id: Unique identifier for the user
        confirmation_token: Token obtained from human operator via separate API
    
    Returns:
        {
            "user_id": str,
            "deleted_at": str (ISO 8601),
            "data_retention_days": int
        }
    """
    # Validate confirmation token
    if not validate_confirmation_token(confirmation_token, operation="delete_user"):
        raise AuthorizationError("Invalid or expired confirmation token")
    
    # Check if already deleted
    user = db.users.find_one({"user_id": user_id})
    if not user:
        raise NotFoundError(f"User {user_id} not found")
    
    if user.get("deleted_at"):
        return {
            "user_id": user_id,
            "deleted_at": user["deleted_at"].isoformat(),
            "data_retention_days": 30
        }
    
    # Perform deletion
    deleted_at = datetime.utcnow()
    db.users.update_one(
        {"user_id": user_id},
        {
            "$set": {
                "deleted_at": deleted_at,
                "status": "deleted"
            }
        }
    )
    
    # Cascade deletions (idempotent)
    cancel_user_subscriptions(user_id)
    anonymize_user_logs(user_id)
    
    return {
        "user_id": user_id,
        "deleted_at": deleted_at.isoformat(),
        "data_retention_days": 30
    }
```

### Pattern 2: Dry-Run First

The tool supports a dry-run mode that shows what would be deleted without actually deleting it.

```python
def delete_cloud_resources(resource_ids: list[str], dry_run: bool = True) -> dict:
    """
    Delete cloud resources (VMs, databases, storage).
    
    Scope: destructive
    Idempotent: Yes
    Side effects: Terminates resources, deletes data
    
    Args:
        resource_ids: List of resource identifiers to delete
        dry_run: If True, return what would be deleted without deleting
    
    Returns:
        {
            "dry_run": bool,
            "resources_to_delete": [
                {
                    "resource_id": str,
                    "resource_type": str,
                    "estimated_cost_saved": float
                }
            ],
            "total_cost_saved": float
        }
    """
    resources_to_delete = []
    total_cost = 0.0
    
    for resource_id in resource_ids:
        resource = cloud_api.get_resource(resource_id)
        if not resource:
            continue  # Already deleted or doesn't exist
        
        cost = calculate_monthly_cost(resource)
        resources_to_delete.append({
            "resource_id": resource_id,
            "resource_type": resource["type"],
            "estimated_cost_saved": cost
        })
        total_cost += cost
    
    # If dry run, return preview only
    if dry_run:
        return {
            "dry_run": True,
            "resources_to_delete": resources_to_delete,
            "total_cost_saved": total_cost,
            "message": "This is a dry run. Set dry_run=False to execute."
        }
    
    # Actual deletion (requires destructive scope)
    deleted = []
    for resource_info in resources_to_delete:
        cloud_api.delete_resource(resource_info["resource_id"])
        deleted.append(resource_info["resource_id"])
    
    return {
        "dry_run": False,
        "resources_deleted": deleted,
        "total_cost_saved": total_cost
    }
```

## Example: Confirmed Execution

A tool that requires explicit confirmation for each destructive operation.

```python
def liquidate_trading_account(account_id: str, confirm: bool = False) -> dict:
    """
    Liquidate all positions in a trading account and close it.
    
    Scope: destructive
    Idempotent: Yes
    Side effects: Sells all positions, transfers funds, closes account
    
    Args:
        account_id: Trading account identifier
        confirm: Must be explicitly set to True to execute
    
    Returns:
        {
            "account_id": str,
            "positions_closed": int,
            "funds_transferred": float,
            "executed_at": str (ISO 8601)
        }
    """
    if not confirm:
        # Return preview without executing
        positions = get_account_positions(account_id)
        total_value = sum(p["value"] for p in positions)
        
        return {
            "account_id": account_id,
            "preview": True,
            "positions_to_close": len(positions),
            "estimated_transfer": total_value,
            "message": "Set confirm=True to execute liquidation"
        }
    
    # Execute liquidation
    account = db.accounts.find_one({"account_id": account_id})
    if not account:
        raise NotFoundError(f"Account {account_id} not found")
    
    if account.get("status") == "closed":
        return {
            "account_id": account_id,
            "positions_closed": 0,
            "funds_transferred": 0.0,
            "executed_at": account["closed_at"].isoformat()
        }
    
    # Close all positions
    positions = get_account_positions(account_id)
    closed_count = 0
    for position in positions:
        close_position(position["position_id"])
        closed_count += 1
    
    # Transfer funds
    balance = get_account_balance(account_id)
    transfer_funds(account_id, balance, destination="user_wallet")
    
    # Mark account as closed
    db.accounts.update_one(
        {"account_id": account_id},
        {"$set": {"status": "closed", "closed_at": datetime.utcnow()}}
    )
    
    return {
        "account_id": account_id,
        "positions_closed": closed_count,
        "funds_transferred": balance,
        "executed_at": datetime.utcnow().isoformat()
    }
```

## Required Scope Explanation

Destructive tools must require `destructive` scope, which is the highest permission level. This scope should be:

1. **Explicitly granted**: Never granted by default, even to trusted agents
2. **Audited**: All destructive operations must be logged with full context
3. **Time-limited**: Consider expiring destructive scope after a short period
4. **Contextual**: Some destructive operations may require additional context (environment, resource type)

```python
# Tool definition in MCP server
{
    "name": "delete_user_account",
    "description": "Permanently delete a user account",
    "inputSchema": {
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "confirmation_token": {"type": "string"}
        },
        "required": ["user_id", "confirmation_token"]
    },
    "requiredScopes": ["destructive"],  # Highest permission level
    "requiresConfirmation": True  # Metadata flag
}
```

## Best Practices

1. **Always require confirmation**: Never allow autonomous destructive operations
2. **Support dry-run mode**: Let agents preview what would be deleted
3. **Idempotent design**: Multiple calls with same inputs should be safe
4. **Explicit scope**: Require `destructive` scope, never `write` or `read`
5. **Comprehensive logging**: Log who requested, when, why, and what was deleted
6. **Recovery information**: Return data retention periods or recovery options when possible
