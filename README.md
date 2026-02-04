# MCP Toolkit Examples

A production-focused guide to designing safe, scoped, and well-structured tools for Model Context Protocol (MCP) servers.

## What is MCP?

Model Context Protocol (MCP) is a standardized interface that allows AI agents to interact with external systems through a well-defined set of tools. An MCP server exposes capabilities via `tools/list` and `tools/call` endpoints, enabling agents to perform actions ranging from reading data to executing complex operations. The protocol includes built-in support for permission scopes and audit logging, making it suitable for production deployments where safety and accountability matter.

## The Problem with Agentic AI

When AI agents are given direct access to systems—whether databases, APIs, or infrastructure—naive tool design creates significant risks. An agent that can call `execute_sql("DROP TABLE users")` or `delete_all_files()` without safeguards will eventually cause damage. Agents operate autonomously, make mistakes, and can be manipulated through prompt injection. Without proper scoping, confirmation flows, and idempotency guarantees, agentic systems become unreliable and dangerous.

## Why Naive Tool Design is Dangerous

The most common mistake is exposing tools that mirror raw system APIs without adding agent-specific safety layers. A tool that accepts arbitrary SQL queries, shell commands, or file paths invites disaster. Agents lack the context and judgment that human operators have. They will attempt operations that seem reasonable in isolation but are catastrophic in practice. They will retry failed operations, creating duplicate records or cascading failures. They will not understand the difference between "test" and "production" environments without explicit guidance.

## What This Repository Teaches

This repository provides patterns, examples, and anti-patterns for designing MCP tools that are:

- **Safe**: Tools that cannot cause unintended damage even when called incorrectly
- **Scoped**: Tools that require explicit permission levels (read, write, destructive)
- **Idempotent**: Tools that can be safely retried without side effects
- **Auditable**: Tools that produce clear logs of what was attempted and why
- **Agent-friendly**: Tools that return structured data and clear error messages

## How to Use This Repository

This repository contains **tool design examples only**. It does not include MCP server implementation code. Use it alongside an MCP server template or framework that handles the protocol layer (transport, message serialization, tool registration).

1. Review the patterns in `patterns/` to understand different tool categories
2. Study the examples in `example-domains/` to see how patterns apply in practice
3. Avoid the anti-patterns documented in `anti-patterns/`
4. Design your tools with scopes as outlined in `security/scope-design.md`

Each example includes tool signatures, scope requirements, and explanations of why the design is safe (or unsafe). Adapt these patterns to your domain while maintaining the same safety principles.

## Who This Is For

This repository is for:
- Platform engineers building MCP servers for production use
- AI safety engineers designing agent interaction patterns
- Developers integrating agentic AI into existing systems
- Teams evaluating whether to expose capabilities to agents

It assumes familiarity with Python, REST APIs, and basic security concepts. Prior knowledge of MCP is helpful but not required.

## What This Repository Is NOT

- **Not an MCP server implementation**: This repo focuses on tool design, not server boilerplate
- **Not a tutorial on building agents**: We assume you have an agent that can call MCP tools
- **Not a security audit checklist**: These are design patterns, not compliance requirements
- **Not production-ready code**: Examples are simplified for clarity; adapt them to your needs

## Structure

```
patterns/              # Design patterns for different tool types
example-domains/       # Complete examples in specific domains
anti-patterns/         # What NOT to do
security/              # Scope design and audit logging guidance
```
