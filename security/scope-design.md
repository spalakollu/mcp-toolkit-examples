# Scope Design for MCP Tools

Scopes are permission levels that control which tools an agent can call. Proper scope design is fundamental to agent safety: it ensures agents can only perform operations they're authorized for, reducing the risk of accidental or malicious damage.

## Why Scopes Matter for MCP

Without scopes, agents would have unrestricted access to all tools. A misconfigured agent or a prompt injection attack could call destructive tools even when only read access is intended. Scopes provide:

1. **Principle of least privilege**: Agents receive only the minimum permissions needed
2. **Attack surface reduction**: Limiting scopes reduces the impact of agent errors or attacks
3. **Audit clarity**: Scope requirements make it clear what operations require higher privileges
4. **Deployment flexibility**: Different environments (dev, staging, prod) can grant different scopes

## Scope Levels

MCP defines three scope levels, ordered from least to most privileged:

### Read Scope

**Purpose**: Retrieve information without modifying state.

**Allowed operations**:
- Querying databases
- Fetching API data
- Listing resources
- Reading files
- Checking status

**Tool examples**:
- `get_user_profile(user_id)`
- `list_files(directory)`
- `query_database(table, filters)`
- `get_system_status()`

**When to use**: Default scope for exploration and discovery. Most agents should start with read-only access.

### Write Scope

**Purpose**: Create or modify resources, but not delete them.

**Allowed operations**:
- Creating records
- Updating fields
- Uploading files
- Modifying configurations
- Appending data

**Tool examples**:
- `create_user(email, name)`
- `update_user_profile(user_id, updates)`
- `upload_file(path, content)`
- `update_configuration(key, value)`

**When to use**: When agents need to create or modify data. Requires explicit grant beyond read scope.

### Destructive Scope

**Purpose**: Permanently delete or terminate resources.

**Allowed operations**:
- Deleting records
- Terminating resources
- Purging data
- Canceling subscriptions
- Irreversible operations

**Tool examples**:
- `delete_user_account(user_id)`
- `terminate_cloud_instance(instance_id)`
- `purge_logs(older_than)`
- `cancel_subscription(subscription_id)`

**When to use**: Only for operations that cannot be undone. Should require human confirmation or be restricted to specific, high-confidence scenarios.

## Mapping Tools to Scopes

Every tool must declare its required scope. The mapping is straightforward:

| Tool Behavior | Required Scope |
|---------------|----------------|
| Fetches data, no side effects | `read` |
| Creates or updates resources | `write` |
| Deletes or terminates resources | `destructive` |

### Scope Matrix

```
┌─────────────────────┬──────────┬──────────┬──────────────┐
│ Tool Operation      │ Read     │ Write    │ Destructive  │
├─────────────────────┼──────────┼──────────┼──────────────┤
│ get_user_profile    │ ✓        │ ✓        │ ✓            │
│ create_user         │ ✗        │ ✓        │ ✓            │
│ update_user_profile │ ✗        │ ✓        │ ✓            │
│ delete_user_account │ ✗        │ ✗        │ ✓            │
└─────────────────────┴──────────┴──────────┴──────────────┘
```

An agent with `read` scope can only call `get_user_profile`. An agent with `write` scope can call `get_user_profile`, `create_user`, and `update_user_profile`, but not `delete_user_account`.

## How Scopes Interact with Agent Planning

Agents use tool availability to plan their actions. When an agent only has `read` scope:

1. **Discovery phase**: Agent can explore system state safely
2. **Planning phase**: Agent can identify what needs to be created or updated
3. **Execution phase**: Agent requests write scope for specific operations, or reports that write access is needed

This pattern prevents agents from making changes until explicitly authorized.

### Example: Agent Workflow with Scopes

```
Agent (read scope only):
1. Calls get_user_profile(user_id="123") → Success
2. Calls update_user_profile(user_id="123", updates={...}) → Error: Requires write scope
3. Reports to user: "I can read user profiles but need write scope to update them"

User grants write scope:
4. Calls update_user_profile(user_id="123", updates={...}) → Success
5. Calls delete_user_account(user_id="123") → Error: Requires destructive scope
```

## Enforcing Scope in Tool Calls

The MCP server must enforce scopes before executing tools. This enforcement happens at the protocol level, not within individual tools.

### Example: Scope Enforcement

```python
# MCP server implementation (pseudo-code)
def call_tool(tool_name: str, arguments: dict, agent_scopes: list[str]) -> dict:
    tool = get_tool_definition(tool_name)
    required_scopes = tool["requiredScopes"]
    
    # Check if agent has required scopes
    if not all(scope in agent_scopes for scope in required_scopes):
        raise ScopeError(
            f"Tool {tool_name} requires scopes {required_scopes}, "
            f"but agent has {agent_scopes}"
        )
    
    # Execute tool
    return tool["handler"](**arguments)
```

### Tool Definition with Scope

```json
{
    "name": "update_user_profile",
    "description": "Update user profile fields",
    "inputSchema": {
        "type": "object",
        "properties": {
            "user_id": {"type": "string"},
            "updates": {
                "type": "object",
                "properties": {
                    "name": {"type": "string"},
                    "bio": {"type": "string"}
                }
            }
        },
        "required": ["user_id", "updates"]
    },
    "requiredScopes": ["write"]
}
```

## Scope Best Practices

1. **Default to read**: Start agents with read-only access; grant higher scopes only when needed
2. **Explicit declarations**: Every tool must declare its required scope
3. **Server-side enforcement**: Scopes must be enforced by the MCP server, not tool implementations
4. **Scope inheritance**: Higher scopes include lower scopes (write includes read, destructive includes write)
5. **Time-limited grants**: Consider expiring write and destructive scopes after a period
6. **Context-aware scopes**: Some operations may require scopes plus additional context (environment, resource type)

## Common Mistakes

### Mistake 1: Tools Don't Declare Scopes

```python
# BAD: No scope declaration
def delete_user(user_id: str):
    db.users.delete_one({"user_id": user_id})
```

**Fix**: Always declare required scopes in tool metadata.

### Mistake 2: Scope Checks in Tool Implementation

```python
# BAD: Scope check inside tool
def update_user(user_id: str, updates: dict):
    if "write" not in current_scopes:  # Wrong place!
        raise Error("Need write scope")
    # ...
```

**Fix**: Scopes should be enforced by the MCP server before tool execution.

### Mistake 3: Over-Privileged Defaults

```python
# BAD: Granting destructive scope by default
agent_scopes = ["read", "write", "destructive"]  # Too permissive!
```

**Fix**: Start with read-only, grant higher scopes explicitly when needed.

## Summary

Scopes are the foundation of agent safety in MCP. They ensure agents can only perform operations they're authorized for, reducing risk and enabling safe agent deployment. Every tool must declare its scope requirement, and the MCP server must enforce these requirements before execution.
