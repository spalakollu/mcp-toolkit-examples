# Filesystem Example

This example demonstrates how to design safe MCP tools for a filesystem domain. **No real file I/O or OS calls are included.** The tool signatures and patterns are illustrative. The critical principle is to restrict all operations to one or more allowlisted roots and to reject path traversal; agents must not read or write outside those roots.

## Domain Overview

A filesystem domain allows agents to:
- List directory contents and read file contents within allowed roots (read)
- Create or overwrite files within allowed roots (write)
- Delete files or directories within allowed roots (destructive)

Unsafe design (e.g., accepting arbitrary paths or shell commands) can expose sensitive files or allow overwriting or deleting critical paths. Safe design requires a configured root (or roots), canonical path resolution that blocks traversal, and bounded listing and file sizes.

## Path Safety

All tools must:
- Resolve the given path to a single canonical path (resolve `..` and `.` and symlinks if applicable)
- Ensure the result is under an allowlisted root
- Reject the request otherwise

```python
ALLOWED_ROOTS = ["/workspace/agent"]  # Configured per environment

def resolve_within_root(path: str) -> str:
    """Resolve path and ensure it is under an allowed root. Raise ValidationError if not."""
    canonical = os.path.realpath(os.path.normpath(path))
    for root in ALLOWED_ROOTS:
        root_canonical = os.path.realpath(root)
        if canonical == root_canonical or canonical.startswith(root_canonical + os.sep):
            return canonical
    raise ValidationError(f"Path not under allowed roots: {path}")
```

## Tool Design

### Read Tool: List Directory

```python
def list_directory(path: str, max_entries: int = 500) -> dict:
    """
    List entries in a directory under an allowed root.
    
    Scope: read
    Idempotent: Yes
    Side effects: None
    
    Args:
        path: Directory path (must resolve under allowed root)
        max_entries: Maximum number of entries to return (1-1000, default 500)
    
    Returns:
        {
            "path": str (canonical),
            "entries": [
                {"name": str, "type": "file" | "directory", "size_bytes": int | None}
            ],
            "count": int,
            "truncated": bool
        }
    
    Raises:
        ValidationError: If path outside root or max_entries invalid
        NotFoundError: If path does not exist or is not a directory
    """
    canonical = resolve_within_root(path)
    
    if not 1 <= max_entries <= 1000:
        raise ValidationError(f"max_entries must be 1-1000, got {max_entries}")
    
    if not os.path.isdir(canonical):
        raise NotFoundError(f"Not a directory: {path}")
    
    entries = []
    for i, name in enumerate(os.listdir(canonical)):
        if i >= max_entries:
            break
        entry_path = os.path.join(canonical, name)
        if os.path.isfile(entry_path):
            entries.append({
                "name": name,
                "type": "file",
                "size_bytes": os.path.getsize(entry_path)
            })
        else:
            entries.append({"name": name, "type": "directory", "size_bytes": None})
    
    return {
        "path": canonical,
        "entries": entries,
        "count": len(entries),
        "truncated": len(entries) >= max_entries
    }
```

**Why this design is safe**:
- Path is resolved and checked against allowed roots; no traversal outside
- Bounded number of entries to avoid unbounded reads and large responses
- Read-only; no state changes
- Requires only read scope

### Read Tool: Read File

```python
MAX_READ_BYTES = 512 * 1024  # 512 KiB

def read_file(path: str, offset: int = 0, max_bytes: int | None = None) -> dict:
    """
    Read contents of a file under an allowed root.
    
    Scope: read
    Idempotent: Yes
    Side effects: None
    
    Args:
        path: File path (must resolve under allowed root)
        offset: Byte offset to start reading (default 0)
        max_bytes: Maximum bytes to return (default up to MAX_READ_BYTES)
    
    Returns:
        {
            "path": str (canonical),
            "content": str (UTF-8 decoded),
            "bytes_read": int,
            "total_size": int,
            "truncated": bool
        }
    
    Raises:
        ValidationError: If path outside root or offset/max_bytes invalid
        NotFoundError: If path does not exist or is not a file
    """
    canonical = resolve_within_root(path)
    
    if not os.path.isfile(canonical):
        raise NotFoundError(f"Not a file: {path}")
    
    size = os.path.getsize(canonical)
    if offset < 0 or offset >= size:
        raise ValidationError(f"offset must be in [0, {size})")
    
    limit = min(max_bytes or MAX_READ_BYTES, MAX_READ_BYTES)
    if limit <= 0:
        raise ValidationError("max_bytes must be positive")
    
    with open(canonical, "rb") as f:
        f.seek(offset)
        data = f.read(limit)
    
    try:
        content = data.decode("utf-8")
    except UnicodeDecodeError:
        raise ValidationError("File is not valid UTF-8 (or pass max_bytes for binary range)")
    
    return {
        "path": canonical,
        "content": content,
        "bytes_read": len(data),
        "total_size": size,
        "truncated": len(data) == limit and offset + len(data) < size
    }
```

**Why this design is safe**:
- Path restricted to allowed root
- Cap on read size to avoid loading huge files into agent context
- Optional offset/limit for large files
- Read-only; requires only read scope

### Write Tool: Write File

```python
MAX_WRITE_BYTES = 256 * 1024  # 256 KiB

def write_file(path: str, content: str, create_only: bool = False) -> dict:
    """
    Write content to a file under an allowed root.
    
    Scope: write
    Idempotent: No (overwrites if exists, unless create_only=True)
    Side effects: Creates or overwrites file
    
    Args:
        path: File path (must resolve under allowed root; parent dir must exist)
        content: UTF-8 string content
        create_only: If True, fail if file already exists
    
    Returns:
        {
            "path": str (canonical),
            "bytes_written": int,
            "created": bool
        }
    
    Raises:
        ValidationError: If path outside root, content too large, or create_only and file exists
        NotFoundError: If parent directory does not exist
    """
    canonical = resolve_within_root(path)
    
    encoded = content.encode("utf-8")
    if len(encoded) > MAX_WRITE_BYTES:
        raise ValidationError(f"Content exceeds max size {MAX_WRITE_BYTES} bytes")
    
    parent = os.path.dirname(canonical)
    if parent and not os.path.isdir(parent):
        raise NotFoundError(f"Parent directory does not exist: {parent}")
    
    created = not os.path.exists(canonical)
    if create_only and not created:
        raise ValidationError(f"File already exists: {path}")
    
    with open(canonical, "wb") as f:
        f.write(encoded)
    
    return {
        "path": canonical,
        "bytes_written": len(encoded),
        "created": created
    }
```

**Why this design is safe**:
- Path restricted to allowed root; parent must exist (no arbitrary directory creation here)
- Bounded write size to limit blast radius and context size
- Optional create_only to avoid overwriting by mistake
- Requires write scope

### Destructive Tool: Delete Path

```python
def delete_path(path: str, confirm: bool = False) -> dict:
    """
    Delete a file or empty directory under an allowed root.
    
    Scope: destructive
    Idempotent: Yes (returns success if path already missing)
    Side effects: Removes file or empty directory
    
    Args:
        path: File or directory path (must resolve under allowed root)
        confirm: Must be True to execute
    
    Returns:
        {
            "path": str (canonical),
            "executed": bool,
            "type": "file" | "directory",
            "message": str (if not executed)
        }
    
    Raises:
        ValidationError: If path outside root, or directory not empty and recursive not allowed
    """
    canonical = resolve_within_root(path)
    
    if not os.path.exists(canonical):
        return {
            "path": canonical,
            "executed": False,
            "type": "file",
            "message": "Path already does not exist"
        }
    
    is_dir = os.path.isdir(canonical)
    if is_dir and os.listdir(canonical):
        raise ValidationError("Directory not empty; delete contents first or use recursive delete tool")
    
    if not confirm:
        return {
            "path": canonical,
            "executed": False,
            "type": "directory" if is_dir else "file",
            "message": "Set confirm=True to delete"
        }
    
    if is_dir:
        os.rmdir(canonical)
    else:
        os.remove(canonical)
    
    return {
        "path": canonical,
        "executed": True,
        "type": "directory" if is_dir else "file"
    }
```

**Why this design is safe**:
- Path restricted to allowed root
- confirm=True required to execute
- No recursive delete in this example to avoid mass deletion; agents must delete leaf nodes first or you expose a separate, tightly scoped recursive tool
- Idempotent when path already missing
- Requires destructive scope

## What Not to Do

Do not expose tools that accept arbitrary paths without root checks or that execute shell commands:

```python
# BAD: No root restriction
def read_file_unsafe(path: str):
    return open(path).read()

# BAD: Shell or arbitrary command
def run_command(cmd: str):
    return subprocess.run(cmd, shell=True)
```

Agents (or prompt injection) could pass paths like `/etc/passwd` or `../../secrets.env`. Always resolve and restrict to allowed roots.

## How Scopes Protect

- **Read only**: Agent can list directories and read files under allowed roots; no writes or deletes.
- **Read + write**: Agent can create/overwrite files under allowed roots; still cannot delete.
- **Destructive**: Agent can delete files or empty directories under allowed roots only when confirm=True.

## Summary

The filesystem example shows:
- **Read**: list_directory (bounded entries) and read_file (bounded size) with path resolved under allowed roots
- **Write**: write_file with path restriction and max content size
- **Destructive**: delete_path with confirm and no recursive delete by default

All operations use the same path rule: resolve to canonical path and require it to be under an allowlisted root. Never expose unbounded or unrestricted path or shell access to agents.
