# Cloud Resources Example

This example demonstrates how to design safe MCP tools for a cloud infrastructure domain. **No real cloud provider APIs are integrated here.** The tool signatures and patterns are illustrative and generalize to real systems (e.g., AWS, GCP, Azure) when you add provider-specific clients and validation.

## Domain Overview

A cloud resources domain allows agents to:
- List and inspect instances, volumes, and other resources (read)
- Create or update resources (write)
- Terminate or delete resources (destructive)

This domain is well-suited to scope design because listing and creating resources are fundamentally different from terminating them. Unbounded or mistaken destructive calls can take down production workloads.

## Tool Design

### Read Tool: List Instances

```python
def list_instances(
    environment: str,
    resource_type: str = "compute",
    max_results: int = 50
) -> dict:
    """
    List cloud instances (VMs) in an environment.
    
    Scope: read
    Idempotent: Yes
    Side effects: None
    
    Args:
        environment: Target environment ("development" | "staging" | "production")
        resource_type: Resource category ("compute" | "database" | "storage")
        max_results: Maximum number of results to return (1-100, default 50)
    
    Returns:
        {
            "environment": str,
            "resources": [
                {
                    "resource_id": str,
                    "resource_type": str,
                    "name": str,
                    "status": "running" | "stopped" | "terminated",
                    "created_at": str (ISO 8601),
                    "tags": dict
                }
            ],
            "total_count": int
        }
    
    Raises:
        ValidationError: If environment or max_results is invalid
    """
    if environment not in ("development", "staging", "production"):
        raise ValidationError(f"Invalid environment: {environment}")
    
    if not 1 <= max_results <= 100:
        raise ValidationError(f"max_results must be 1-100, got {max_results}")
    
    resources = db.resources.find(
        {"environment": environment, "resource_type": resource_type}
    ).limit(max_results)
    
    resource_list = [
        {
            "resource_id": r["resource_id"],
            "resource_type": r["resource_type"],
            "name": r["name"],
            "status": r["status"],
            "created_at": r["created_at"].isoformat(),
            "tags": r.get("tags", {})
        }
        for r in resources
    ]
    
    total = db.resources.count_documents(
        {"environment": environment, "resource_type": resource_type}
    )
    
    return {
        "environment": environment,
        "resources": resource_list,
        "total_count": min(total, max_results)
    }
```

**Why this design is safe**:
- No state changes; read-only
- Bounded result set (max_results) to avoid unbounded agent loops
- Validates environment and numeric bounds
- Returns structured data; no internal IDs or secrets
- Requires only read scope

### Write Tool: Create Instance

```python
def create_instance(
    environment: str,
    instance_type: str,
    name: str,
    idempotency_key: str | None = None
) -> dict:
    """
    Create a new compute instance in an environment.
    
    Scope: write
    Idempotent: Yes (if idempotency_key provided and instance exists)
    Side effects: Provisions VM, allocates resources
    
    Args:
        environment: Target environment ("development" | "staging" | "production")
        instance_type: Instance size (e.g., "small", "medium", "large")
        name: Human-readable instance name
        idempotency_key: Optional UUID to prevent duplicate creation
    
    Returns:
        {
            "resource_id": str,
            "environment": str,
            "instance_type": str,
            "name": str,
            "status": "provisioning",
            "created_at": str (ISO 8601)
        }
    
    Raises:
        ValidationError: If inputs are invalid
        QuotaExceededError: If environment instance limit reached
    """
    if environment not in ("development", "staging", "production"):
        raise ValidationError(f"Invalid environment: {environment}")
    
    allowed_types = {"small", "medium", "large"}
    if instance_type not in allowed_types:
        raise ValidationError(f"instance_type must be one of {allowed_types}")
    
    if not name or len(name) > 64:
        raise ValidationError("name must be 1-64 characters")
    
    # Idempotency: return existing instance if key matches
    if idempotency_key:
        existing = db.resources.find_one({"idempotency_key": idempotency_key})
        if existing:
            return format_instance(existing)
    
    # Check quota
    current_count = db.resources.count_documents(
        {"environment": environment, "resource_type": "compute", "status": {"$ne": "terminated"}}
    )
    if current_count >= get_environment_quota(environment):
        raise QuotaExceededError(f"Instance quota exceeded for {environment}")
    
    # Create instance
    resource_id = generate_uuid()
    now = datetime.utcnow()
    instance = {
        "resource_id": resource_id,
        "environment": environment,
        "resource_type": "compute",
        "instance_type": instance_type,
        "name": name,
        "status": "provisioning",
        "created_at": now,
        "idempotency_key": idempotency_key
    }
    db.resources.insert_one(instance)
    
    # Trigger provisioning (simulated)
    provision_instance(resource_id, environment, instance_type)
    
    return format_instance(instance)
```

**Why this design is safe**:
- Idempotent via idempotency_key
- Validates environment, instance type, and name
- Quota check prevents runaway creation
- Requires write scope
- Returns a clear, structured response

### Destructive Tool: Terminate Instance

```python
def terminate_instance(
    resource_id: str,
    dry_run: bool = True
) -> dict:
    """
    Terminate a compute instance and release its resources.
    
    Scope: destructive
    Idempotent: Yes (returns success if already terminated)
    Side effects: Stops VM, deletes attached ephemeral storage, releases IP
    
    Args:
        resource_id: Unique identifier for the instance
        dry_run: If True, return impact summary without terminating
    
    Returns:
        {
            "resource_id": str,
            "dry_run": bool,
            "environment": str,
            "name": str,
            "terminated_at": str (ISO 8601) | None,
            "message": str (if dry_run)
        }
    
    Raises:
        NotFoundError: If resource_id does not exist
        ValidationError: If resource is not a terminable compute instance
    """
    resource = db.resources.find_one({"resource_id": resource_id})
    
    if not resource:
        raise NotFoundError(f"Resource {resource_id} not found")
    
    if resource["resource_type"] != "compute":
        raise ValidationError(f"Resource {resource_id} is not a compute instance")
    
    if resource.get("status") == "terminated":
        return {
            "resource_id": resource_id,
            "dry_run": False,
            "environment": resource["environment"],
            "name": resource["name"],
            "terminated_at": resource["terminated_at"].isoformat()
        }
    
    if dry_run:
        return {
            "resource_id": resource_id,
            "dry_run": True,
            "environment": resource["environment"],
            "name": resource["name"],
            "terminated_at": None,
            "message": "Dry run. Set dry_run=False to terminate this instance."
        }
    
    # Execute termination
    terminated_at = datetime.utcnow()
    db.resources.update_one(
        {"resource_id": resource_id},
        {"$set": {"status": "terminated", "terminated_at": terminated_at}}
    )
    
    # Release resources (simulated)
    release_instance_resources(resource_id)
    
    return {
        "resource_id": resource_id,
        "dry_run": False,
        "environment": resource["environment"],
        "name": resource["name"],
        "terminated_at": terminated_at.isoformat()
    }
```

**Why this design is safe**:
- dry_run default prevents accidental termination
- Idempotent when instance is already terminated
- Validates resource type and existence
- Requires destructive scope
- Returns clear outcome and, in dry run, an explicit message

## How Scopes Protect Against Catastrophic Actions

### Agent with Read-Only Access

```
Agent scopes: ["read"]

Agent attempts:
1. list_instances(environment="production") -> Success
2. create_instance(environment="production", ...) -> Error: Requires write scope
3. terminate_instance(resource_id="i-123") -> Error: Requires destructive scope

Result: Agent can discover resources but cannot create or terminate.
```

### Agent with Write Access

```
Agent scopes: ["read", "write"]

Agent attempts:
1. list_instances(environment="production") -> Success
2. create_instance(environment="production", ...) -> Success
3. terminate_instance(resource_id="i-123") -> Error: Requires destructive scope

Result: Agent can provision resources but cannot terminate them.
```

### Agent with Destructive Access

```
Agent scopes: ["read", "write", "destructive"]

Agent attempts:
1. list_instances(environment="production") -> Success
2. terminate_instance(resource_id="i-123", dry_run=True) -> Preview returned
3. terminate_instance(resource_id="i-123", dry_run=False) -> Success

Result: Destructive actions require explicit dry_run=False; dry run is the default.
```

## Additional Safeguards for Cloud

- **Environment isolation**: Validate environment on every call so agents cannot accidentally target production when given staging credentials.
- **Bounded listing**: list_instances uses max_results to avoid "list all then loop" anti-patterns.
- **Dry run by default**: Destructive tools default to dry_run=True so agents must explicitly request execution.
- **Quota checks**: Write tools enforce quotas to limit blast radius from misconfigured or looping agents.

## Why This Pattern Generalizes

The same read / write / destructive split applies to:
- Databases: list schemas (read), create table (write), drop table (destructive)
- Storage: list objects (read), upload object (write), delete bucket (destructive)
- Networking: list VPCs (read), create subnet (write), delete VPC (destructive)

Design tools with clear scope requirements, validation, idempotency where appropriate, and safe defaults (e.g., dry run) for destructive operations.

## Summary

The cloud-resources example shows:
- **Read tools** for safe discovery with bounded results
- **Write tools** with validation, idempotency, and quota checks
- **Destructive tools** with dry-run default and destructive scope

Scopes ensure agents can only list, create, or terminate according to the permissions they are granted, reducing the risk of accidental or malicious impact on production infrastructure.
