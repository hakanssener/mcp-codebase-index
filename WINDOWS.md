# Windows / Antigravity Compatibility

This fork includes two fixes to make `mcp-codebase-index` work on **Windows** with **Antigravity** (the VS Code-based AI coding IDE) and other MCP clients that use piped stdio on Windows.

## Problem

The original tool was designed for Claude Code on macOS/Linux. On Windows, two issues cause the MCP server to hang:

### 1. `pathlib.Path.glob()` follows NTFS junctions

`_discover_files()` used `Path.glob("**/*.ext")` which:
- Follows NTFS junction points (e.g., in `.agent/` or `docs/`)
- Walks the entire `.git/` objects directory before applying exclude patterns
- Can hang indefinitely on Windows with mapped drives or junction-heavy repos

### 2. `subprocess.run()` corrupts asyncio stdio pipes

`_ensure_index()` called `is_git_repo()` → `subprocess.run(["git", ...])` for git detection. On Windows, spawning subprocesses inside an async MCP server corrupts the `ProactorEventLoop`'s stdin/stdout pipe handles. The tool call completes internally but the JSON-RPC response **never gets written to stdout**.

## Fix

### File: `project_indexer.py` — `_discover_files()`

Replaced `Path.glob()` with `os.walk(followlinks=False)` + early directory pruning:

```python
def _discover_files(self) -> list[str]:
    prune_dirs = {".git", "__pycache__", "node_modules", ".venv", "venv"}
    for pattern in self.exclude_patterns:
        clean = pattern.replace("**/", "").replace("/**", "").strip("/")
        if "/" not in clean and "*" not in clean:
            prune_dirs.add(clean)

    for dirpath, dirs, filenames in os.walk(self.root_path, followlinks=False):
        dirs[:] = [d for d in dirs if d not in prune_dirs]
        # ... match files against include/exclude patterns
```

### File: `server.py` — `_ensure_index()`

Skip git subprocess calls. Load cache directly when available:

```python
def _ensure_index() -> None:
    if _indexer is not None:
        return
    _project_root = os.environ.get("PROJECT_ROOT", os.getcwd())
    cached_index = _load_cache(_project_root)
    if cached_index is not None:
        _indexer = ProjectIndexer(_project_root)
        _indexer._project_index = cached_index
        _query_fns = create_project_query_functions(cached_index)
        return
    _build_index()
```

## Trade-offs

- **Git-based incremental updates are disabled.** The `reindex` tool still works for manual re-indexing.
- **Cache must be pre-built** for instant startup. Run:
  ```bash
  python -c "
  from mcp_codebase_index.project_indexer import ProjectIndexer
  from mcp_codebase_index.query_api import create_project_query_functions
  import pickle, os
  root = os.environ.get('PROJECT_ROOT', '.')
  idx = ProjectIndexer(root)
  index = idx.index()
  with open(os.path.join(root, '.codebase-index-cache.pkl'), 'wb') as f:
      pickle.dump({'version': 1, 'index': index}, f, protocol=pickle.HIGHEST_PROTOCOL)
  print(f'Cached {index.total_files} files')
  "
  ```

## MCP Config (Antigravity / VS Code)

```json
{
  "codebase-index": {
    "command": "mcp-codebase-index",
    "args": [],
    "env": {
      "PROJECT_ROOT": "C:\\path\\to\\your\\project",
      "PYTHONUNBUFFERED": "1"
    }
  }
}
```

## Diagnostics

If the tool hangs, the root cause is almost certainly one of:
1. A `subprocess.run()` call corrupting asyncio's stdio pipes
2. File discovery walking into junctions or `.git/`

Check stderr for diagnostic messages:
```
[mcp-codebase-index] _ensure_index: starting...
[mcp-codebase-index] _ensure_index: loading cache...
[mcp-codebase-index] Cache hit, using cached index
```

If you see `checking git...` or `is_git=True`, the git subprocess path is being taken and may hang.
