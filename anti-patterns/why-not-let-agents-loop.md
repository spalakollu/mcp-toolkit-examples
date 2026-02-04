# Why Not Let Agents Loop

Agents naturally retry failed operations and iterate to accomplish goals. When tool design or platform policy allows unbounded loops—retrying indefinitely, iterating over unbounded collections, or polling until a condition is met—the result is often resource exhaustion, cascading side effects, or runaway cost. This document explains the risks and how to avoid them.

## What "Letting Agents Loop" Means

In this context, looping refers to agent behavior that repeats tool calls without a well-defined bound:

- **Unbounded retries**: Retrying a failed tool call until it succeeds, with no maximum attempt count
- **Unbounded iteration**: Calling a tool once per item in a collection that can grow (e.g., "process all users," "delete all files in a path")
- **Polling loops**: Calling a read or status tool repeatedly until some condition holds, with no timeout or max iterations
- **Recursive or chained exploration**: Following links or IDs until "everything" is processed, with no depth or breadth limit

The danger is not looping in the abstract; it is allowing loops that have no clear stopping condition under agent control.

## Why Unbounded Loops Are Dangerous

### Resource Exhaustion

An agent that retries a failing write tool indefinitely can generate a large volume of requests. If the tool is eventually successful after a backend recovery, the same logical operation may have been applied many times. Even with idempotent tools, unbounded retries consume CPU, memory, and rate-limit capacity and can degrade or take down dependent systems.

### Cascading Writes

When an agent loops over a collection and performs a write per item (e.g., "update every user," "notify every subscriber"), the number of writes scales with collection size. A bug or mis-specification (wrong filter, wrong environment) can cause a single agent run to affect far more resources than intended. Bounded operations and batch-oriented tools limit blast radius.

### Cost and Side Effects

Each tool call can incur cost (API usage, downstream services) or trigger side effects (emails, webhooks, logs). Unbounded loops multiply these effects. Without limits, one agent task can generate unacceptable cost or noise.

### Stuck or Divergent Behavior

Agents that poll until a condition is met may never see that condition (e.g., bug in the tool, wrong environment, or condition that is never true). Without a timeout or max iterations, the agent can loop until it hits an external limit (e.g., context window, runtime kill). That makes behavior hard to reason about and operate.

## Dangerous Patterns

### Pattern 1: Retry Until Success

Agent logic: "If the tool fails, call it again until it succeeds."

```python
# BAD: Platform or agent retries with no cap
while True:
    result = call_tool("place_order", {"account_id": "123", "symbol": "AAPL", "quantity": 10})
    if result.success:
        break
    # No max attempts, no backoff, no circuit breaker
```

**Why it's dangerous**: Transient failures (e.g., rate limits, timeouts) can lead to many duplicate attempts. Even with idempotency keys, this stresses systems and can create duplicate side effects if idempotency is not applied everywhere.

**Better approach**: Enforce a maximum retry count and backoff at the platform or agent layer. Use idempotency keys for write tools so that a bounded number of retries is safe.

### Pattern 2: Iterate Over "All" Items

Agent logic: "Get a list of items, then for each item call a write tool."

```python
# BAD: Agent fetches "all" and loops
items = call_tool("list_users", {"limit": 10000})  # Or no limit!
for item in items["users"]:
    call_tool("send_notification", {"user_id": item["id"]})  # N writes
```

**Why it's dangerous**: List size can be large or unbounded. One agent run can issue thousands of write calls, causing overload, rate-limit hits, or unintended mass actions.

**Better approach**: Expose batch or bulk tools (e.g., "notify users matching filter" with a server-side cap) or enforce a strict limit on list size and document that agents must not assume "process all" is safe. Prefer server-side iteration with limits.

### Pattern 3: Poll Until Condition

Agent logic: "Keep calling get_status until status is 'completed'."

```python
# BAD: No timeout or max iterations
while True:
    status = call_tool("get_job_status", {"job_id": job_id})
    if status["state"] == "completed":
        break
    time.sleep(1)  # Agent or platform does this
```

**Why it's dangerous**: If the job never completes (bug, wrong ID, wrong environment), the agent loops until something else stops it. This wastes resources and makes failures hard to detect.

**Better approach**: Enforce a maximum number of polls or a total timeout. Return a clear "incomplete" or "timeout" result so the agent (or user) can decide what to do next instead of looping forever.

### Pattern 4: Follow-All Links or IDs

Agent logic: "Get resource A, then get every resource linked from A, then every resource linked from those, until nothing new appears."

**Why it's dangerous**: Graph or tree structures can be large or cyclic. Without depth or breadth limits, one exploration can turn into a huge number of read (or worse, write) calls.

**Better approach**: Provide tools that return a bounded set of links or IDs (e.g., with a max depth or limit). Document that agents must not implement unbounded graph traversal themselves.

## Safeguards and Alternatives

### Bounded Retries

At the platform or agent layer, enforce a maximum number of retries per logical operation (e.g., 3 or 5), with backoff. After the limit, return a clear failure to the agent so it can report to the user or try a different strategy instead of looping forever.

### Batch and Bulk Tools

Prefer tools that operate on a set of items in one call (e.g., "notify users matching filter" with a server-side limit) rather than requiring the agent to call a write tool once per item. This reduces the number of tool calls and keeps iteration and limits under server control.

### Timeouts and Max Iterations

For polling or long-running operations, use timeouts or a maximum number of iterations. Tools can return a state like "still_running" or "timeout" so the agent can stop looping and report or escalate.

### Pagination and List Limits

List tools should return paginated results with a fixed max page size and require an explicit parameter for the next page. Do not expose "return all" for large collections. Document that agents must not assume they can safely "page through everything" without a bounded task (e.g., "first 100 only").

### Human-in-the-Loop for Large Operations

For operations that could affect many resources (e.g., bulk delete, bulk update), require explicit human confirmation or a separate "bulk" workflow that is not driven by an agent loop. The agent can propose the operation and parameters; a human or a separate process approves and runs it.

## Summary

Unbounded agent loops lead to resource exhaustion, cascading writes, cost, and stuck behavior. Safe design requires:

- Bounded retries with backoff and a clear failure after the limit
- Batch or bulk tools instead of "one write per item" in a loop
- Timeouts and max iterations for polling
- Pagination and list limits so agents do not assume "process all"
- Human-in-the-loop or server-side execution for large or destructive bulk operations

Tool design and platform policy should make it easy for agents to do bounded, predictable work and hard to accidentally run unbounded loops.
