# Audit Logging for MCP Tools

Audit logging records who called which tools, with what arguments, and what happened. It is essential for accountability, debugging agent behavior, and meeting compliance requirements. This document describes what to log, where to log it, and how to avoid common pitfalls.

## Why Audit Logging Matters

When agents call MCP tools:

- **Accountability**: You need to know which agent or session performed an operation, especially for write and destructive calls.
- **Debugging**: When something goes wrong, logs show the sequence of tool calls and arguments that led to the outcome.
- **Security**: Unusual patterns (e.g. many destructive calls, access to unexpected resources) can be detected and investigated.
- **Compliance**: Many policies and regulations require an audit trail of who did what and when.

Logging should happen at the MCP server layer (around `tools/call`), not only inside individual tools. That way every tool invocation is recorded consistently, including failures and permission denials.

## What to Log

Log an entry for each tool invocation (or attempted invocation) with the following.

### Required Fields

| Field | Description |
|-------|-------------|
| `timestamp` | ISO 8601 UTC time of the call |
| `tool_name` | Name of the tool that was called |
| `caller_id` | Identifier for the agent, session, or user (e.g. session ID, user ID, agent ID). Do not log credentials. |
| `scopes` | Scopes the caller had for this request (e.g. `["read", "write"]`) |
| `result` | One of: `success`, `error`, `scope_denied`, `validation_error` |
| `duration_ms` | Optional but useful: time taken to execute the tool |

### Arguments and Results

- **Arguments**: Log the names and types of arguments, and their values in a form safe for storage. Redact or truncate sensitive values (passwords, tokens, bulk content). For example: log `user_id` and `account_id`, but not a full `content` field; log `content_length` or a hash instead.
- **Result shape**: For success, log a summary (e.g. `rows_returned`, `resource_id`, `created`) rather than the full response body. For errors, log the error type and message, not stack traces or internal details, unless they are needed for debugging in a controlled environment.
- **Ids**: Log resource IDs (e.g. `user_id`, `order_id`) so you can correlate with other systems and investigations.

### What Not to Log

- **Secrets**: No API keys, passwords, or tokens. No raw confirmation tokens; log only that a confirmation was present or absent (or a non-reversible hash if you need to detect duplicates).
- **Full request/response bodies**: Avoid logging entire payloads that may contain PII or large content. Log structure and identifiers instead.
- **Unbounded data**: Do not log arbitrarily large strings or lists. Truncate or summarize (e.g. "list of 50 ids" or first N items).

## Where to Log

Log at the **MCP server** when handling `tools/call`:

1. After resolving the tool and checking scopes (so scope denials are logged).
2. After validating arguments (so validation errors are logged).
3. After executing the tool (so success or tool-raised errors are logged).

Do not rely only on each tool to log internally. Centralizing at the server ensures every call is logged with a consistent schema and that denied or invalid calls are still recorded.

```python
# Pseudo-code: server-side audit on tools/call
def handle_tools_call(caller_id: str, scopes: list[str], tool_name: str, arguments: dict):
    start = time.time()
    
    if not tool_has_scope(tool_name, scopes):
        audit_log(
            timestamp=now_iso(),
            tool_name=tool_name,
            caller_id=caller_id,
            scopes=scopes,
            result="scope_denied",
            duration_ms=elapsed_ms(start)
        )
        raise ScopeError(...)
    
    sanitized_args = sanitize_for_audit(arguments)
    
    try:
        result = invoke_tool(tool_name, arguments)
        audit_log(
            timestamp=now_iso(),
            tool_name=tool_name,
            caller_id=caller_id,
            scopes=scopes,
            arguments=sanitized_args,
            result="success",
            result_summary=summarize_result(result),
            duration_ms=elapsed_ms(start)
        )
        return result
    except ValidationError as e:
        audit_log(
            timestamp=now_iso(),
            tool_name=tool_name,
            caller_id=caller_id,
            scopes=scopes,
            arguments=sanitized_args,
            result="validation_error",
            error_type=type(e).__name__,
            error_message=str(e),
            duration_ms=elapsed_ms(start)
        )
        raise
    except Exception as e:
        audit_log(
            timestamp=now_iso(),
            tool_name=tool_name,
            caller_id=caller_id,
            scopes=scopes,
            arguments=sanitized_args,
            result="error",
            error_type=type(e).__name__,
            duration_ms=elapsed_ms(start)
        )
        raise
```

## Log Structure Example

A minimal JSON log entry might look like:

```json
{
  "timestamp": "2025-02-04T14:32:01.123Z",
  "tool_name": "update_user_profile",
  "caller_id": "session:a1b2c3d4",
  "scopes": ["read", "write"],
  "arguments": {
    "user_id": "usr_abc123",
    "updates": {"name": "Alice", "bio": "[REDACTED 12 chars]"}
  },
  "result": "success",
  "result_summary": {"updated": true},
  "duration_ms": 45
}
```

For a scope denial:

```json
{
  "timestamp": "2025-02-04T14:33:00.000Z",
  "tool_name": "delete_user_account",
  "caller_id": "session:a1b2c3d4",
  "scopes": ["read", "write"],
  "arguments": {"user_id": "usr_xyz"},
  "result": "scope_denied",
  "duration_ms": 1
}
```

Define a single schema and use it for all tool calls so that querying and alerting are consistent.

## Sanitization Guidelines

- **Identifiers**: Keep resource IDs, session IDs, and user IDs (they are needed for tracing). Ensure they are not secrets.
- **Short strings**: For non-secret short strings (e.g. name, status), logging a redacted or truncated form is often enough (e.g. first/last character count or "[REDACTED]").
- **Large or sensitive content**: Do not log full file contents, message bodies, or tokens. Log presence and length or a hash if needed for deduplication.
- **Structured input**: For objects, log keys and types; for sensitive values, replace with a placeholder or redaction note.

## Retention and Access

- **Retention**: Define how long audit logs are kept (e.g. 90 days, 1 year) and whether they are archived. Destructive and write operations may require longer retention for compliance.
- **Access**: Restrict read access to audit logs. Only authorized roles (e.g. security, compliance, support) should query them. Log access to the audit log itself where feasible.
- **Integrity**: Prefer append-only storage or integrity checks so logs cannot be altered without detection.

## Correlation with Tools

Tool design can make auditing easier:

- **Stable resource IDs**: Tools that take and return clear resource IDs (e.g. `user_id`, `order_id`) make it easy to correlate log entries with downstream systems.
- **Structured errors**: Tools that raise typed errors (e.g. `ValidationError`, `NotFoundError`) allow the server to log `error_type` and `error_message` consistently.
- **Result summaries**: Tools that return a short, stable result shape (e.g. `{"created": true, "id": "..."}`) allow the server to log a small `result_summary` without logging full response bodies.

## Anti-Patterns

**Logging only successes**: Log every invocation, including scope_denied, validation_error, and tool errors. Missing failures makes it hard to detect abuse or misconfiguration.

**Logging secrets**: Never log passwords, API keys, or confirmation tokens. Sanitize arguments before writing to the audit log.

**Inconsistent schema**: Use one schema for all tools. Ad-hoc or tool-specific log formats make it harder to query and alert.

**Logging inside tools only**: If each tool logs on its own, denied or invalid calls that never reach the tool will not be logged. Centralize at the server.

**High-cardinality or unbounded fields**: Avoid logging full request IDs that change every time (e.g. raw idempotency keys that are random) if they explode storage; or log a short hash. Do not log unbounded lists or strings.

## Summary

Audit logging for MCP should be implemented at the server layer for every tool call. Log timestamp, tool name, caller, scopes, sanitized arguments, result (success or error type), and optionally duration. Redact secrets and large content; keep identifiers and result summaries. Use a consistent schema and retention policy so that audit logs support accountability, debugging, and compliance.
