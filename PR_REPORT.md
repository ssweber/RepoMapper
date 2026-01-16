# RepoMapper: Bug Fixes and Modernization

## Executive Summary

This PR addresses a critical bug that prevented RepoMapper from generating any output when installed via `uvx`, and modernizes the codebase to use standard Python packaging practices and current type hint syntax.

## Root Cause Analysis

### Issue 1: Tuple Unpacking Bug
When running `uvx --from "git+https://github.com/..." repomapper .`, the tool would fail with:
```
map_string is None
Error: expected string or buffer
TypeError: expected string or buffer
```

**Root Cause**: In `repomap.py:200`, the code was not unpacking the tuple returned by `get_repo_map()`:
```python
# Before (broken)
map_content = repo_map.get_repo_map(...)  # Returns (str|None, FileReport)
if map_content:  # Tuple is always truthy!
    tokens = repo_map.token_count(map_content)  # TypeError: can't count tokens on tuple
```

The method `get_repo_map()` returns a tuple `(map_string, file_report)`, but the code assigned the entire tuple to `map_content`. Since a non-empty tuple is always truthy, it would attempt to count tokens on the tuple object, causing a TypeError.

### Issue 2: Missing Package Data
When installed via git with uvx, the `queries/` directory containing `.scm` tree-sitter query files was not included in the package. This caused 0 definitions and 0 references to be found (since tree-sitter couldn't parse any files without the query definitions).

**Root Cause**: The project used `py-modules` in setuptools, which doesn't have a standard way to include package data. The `queries/` directory was never being packaged.

## Solution Approach

1. **Adopt modern Python packaging standards** - Convert to `src/` layout with hatch build system
2. **Fix critical bugs** - Properly unpack tuples and include package data
3. **Modernize codebase** - Use current Python type hint syntax (PEP 604, 585)
4. **Add linting** - Configure ruff to maintain code quality

## Manual Changes (Detailed)

### 1. Package Structure Conversion

**Rationale**: The `py-modules` approach is deprecated. Modern Python projects use `src/` layout with proper package structure, which provides better isolation, testing, and packaging reliability.

**Changes**:
- Created `src/repomapper/` directory
- Moved all `.py` files from root to `src/repomapper/`
- Moved `queries/` directory to `src/repomapper/queries/`
- Created `src/repomapper/__init__.py` with version info

### 2. Build System Migration

**File**: `pyproject.toml`

**Before**:
```toml
[build-system]
requires = ["setuptools>=69.0", "wheel"]
build-backend = "setuptools.build_meta"

[tool.setuptools]
py-modules = ["repomap_server", "repomap", "repomap_class", "utils", "importance", "scm"]
```

**After**:
```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["src/repomapper"]
```

**Rationale**: Hatchling is the modern, recommended build backend. It automatically handles package data within the package directory and has simpler configuration.

### 3. Critical Bug Fix: Tuple Unpacking

**File**: `src/repomapper/repomap.py:200`

**Before**:
```python
map_content = repo_map.get_repo_map(
    chat_files=chat_files,
    other_files=other_files,
    mentioned_fnames=mentioned_fnames,
    mentioned_idents=mentioned_idents,
    force_refresh=args.force_refresh
)

if map_content:
    if args.verbose:
        tokens = repo_map.token_count(map_content)  # TypeError!
```

**After**:
```python
map_content, file_report = repo_map.get_repo_map(
    chat_files=chat_files,
    other_files=other_files,
    mentioned_fnames=mentioned_fnames,
    mentioned_idents=mentioned_idents,
    force_refresh=args.force_refresh
)

if map_content:
    if args.verbose:
        tokens = repo_map.token_count(map_content)  # Now works!
```

**Rationale**: `get_repo_map()` returns `tuple[str|None, FileReport]`. Must unpack to access the string component.

**Additional Enhancement**: Added verbose output for file report when no map is generated:
```python
else:
    tool_output("No repository map generated.")
    if args.verbose:
        tool_output(f"File report: {file_report.total_files_considered} files considered, "
                   f"{file_report.definition_matches} definitions, "
                   f"{file_report.reference_matches} references")
```

### 4. Import Path Updates

**Rationale**: Moving to `src/repomapper/` package structure requires all internal imports to be relative.

**Changed in ALL files**:
```python
# Before
from utils import count_tokens, read_text
from scm import get_scm_fname

# After
from .utils import count_tokens, read_text
from .scm import get_scm_fname
```

**Files Modified**:
- `src/repomapper/repomap.py`
- `src/repomapper/repomap_class.py`
- `src/repomapper/repomap_server.py`

### 5. Console Script Entry Points

**File**: `pyproject.toml`

**Before**:
```toml
[project.scripts]
repomapper = "repomap:main"
repomap-mcp = "repomap_server:main"
```

**After**:
```toml
[project.scripts]
repomapper = "repomapper.repomap:main"
repomap-mcp = "repomapper.repomap_server:main"
```

**Rationale**: Package name must be included in the module path now that modules are in a package.

### 6. Python Version Requirement

**File**: `pyproject.toml`

**Before**: `requires-python = ">=3.13"`
**After**: `requires-python = ">=3.11"`

**Rationale**: Modern type hint syntax (PEP 604, 585) is available in Python 3.10+, and requiring 3.13 unnecessarily limits adoption. Python 3.11 is a good baseline for new projects.

### 7. Ruff Configuration

**File**: `pyproject.toml`

**Added**:
```toml
[tool.ruff]
line-length = 100

[tool.ruff.lint]
select = [
    "E",    # pycodestyle errors
    "F",    # pyflakes
    "UP",   # pyupgrade
    "B",    # flake8-bugbear
    "I",    # isort
]
ignore = [
    "E501",   # line too long (handled by formatter)
    "E402",   # module level import not at top of file
    # ... etc
]
```

**Rationale**: Establishes baseline code quality standards. Selected rules focus on:
- Code errors (E, F)
- Modernization (UP)
- Common bugs (B)
- Import organization (I)

### 8. Manual Bug Fixes After Ruff

#### 8.1 Removed Duplicate Tag Definition

**File**: `src/repomapper/repomap_class.py:39`

**Issue**: `Tag` was imported from `utils` but then redefined locally:
```python
from .utils import Tag  # Line 20
# ...
Tag = namedtuple("Tag", ...)  # Line 39 - DUPLICATE!
```

**Fix**: Removed the duplicate definition. The import from `utils` is sufficient.

#### 8.2 Removed Unused Variable

**File**: `src/repomapper/repomap_class.py:286`

**Issue**: `input_files` was declared but never used:
```python
input_files: dict[str, dict] = {}  # Never used
```

**Fix**: Removed the variable declaration.

#### 8.3 Fixed Undefined Variable Reference

**File**: `src/repomapper/repomap_class.py:344`

**Issue**: Code path returning undefined `file_report`:
```python
if not G.nodes():
    return [], file_report  # file_report not defined in this scope!
```

**Fix**: Create proper FileReport:
```python
if not G.nodes():
    return [], FileReport(excluded, 0, 0, len(all_fnames))
```

#### 8.4 Fixed Bare Except Clause

**File**: `src/repomapper/repomap_class.py:352`

**Issue**: Bare `except:` catches all exceptions including KeyboardInterrupt:
```python
try:
    ranks = nx.pagerank(...)
except:  # BAD: catches everything
    ranks = {node: 1.0 for node in G.nodes()}
```

**Fix**: Use `except Exception:` to catch only expected errors:
```python
except Exception:  # GOOD: doesn't catch KeyboardInterrupt, etc.
    ranks = {node: 1.0 for node in G.nodes()}
```

#### 8.5 Removed Unused Important Files Filter

**File**: `src/repomapper/repomap_class.py:521-523`

**Issue**: Variable assigned but never used:
```python
important_files = filter_important_files(
    [self.get_rel_fname(f) for f in other_fnames]
)
# important_files never used after this
```

**Fix**: Removed the variable assignment and import. This appears to be dead code from a previous implementation.

#### 8.6 Removed Unused Imports

**Files**: Multiple

**Removed**:
- `src/repomapper/utils.py`: `import os` (unused)
- `src/repomapper/utils.py`: `List` from typing (unused after modernization)
- `src/repomapper/repomap.py`: Multiple unused imports after refactoring
- `src/repomapper/repomap_class.py`: `namedtuple` after removing duplicate Tag
- `src/repomapper/repomap_class.py`: `filter_important_files` after removing dead code

## Automatic Ruff Fixes (--fix)

Ruff automatically fixed **102 issues** across the codebase. These are grouped by category:

### Type Hint Modernization (PEP 604 & PEP 585)

**Changes**: 89 instances across all files

**Pattern 1 - Generic types** (PEP 585):
```python
# Before
from typing import List, Dict, Set, Tuple, Optional
def foo() -> List[str]: ...
def bar() -> Dict[str, Any]: ...

# After
def foo() -> list[str]: ...
def bar() -> dict[str, Any]: ...
```

**Pattern 2 - Optional types** (PEP 604):
```python
# Before
from typing import Optional
def foo() -> Optional[str]: ...

# After
def foo() -> str | None: ...
```

**Pattern 3 - Union types** (PEP 604):
```python
# Before
from typing import Union
def foo() -> Union[str, int]: ...

# After
def foo() -> str | int: ...
```

**Files affected**: All `.py` files in `src/repomapper/`

**Rationale**: Python 3.9+ supports built-in generic types, and Python 3.10+ supports `|` union syntax. This is now the standard way to write type hints.

### Import Organization (isort)

**Changes**: 13 instances

**Pattern**:
```python
# Before
import os
from typing import List
import sys
from pathlib import Path

# After
import os
import sys
from pathlib import Path
from typing import List
```

**Rules Applied**:
1. Standard library imports first
2. Third-party imports second
3. Local imports third
4. Alphabetically sorted within each group
5. Blank line between groups

**Files affected**: All `.py` files with imports

### Deprecated Import Removal

**Pattern**:
```python
# Before
from typing import List  # UP035: deprecated

# After
# Import removed (using list[] instead)
```

**Rationale**: `typing.List`, `typing.Dict`, etc. are deprecated in favor of built-in types.

### Import from collections.abc

**Pattern**:
```python
# Before
from typing import Callable

# After
from collections.abc import Callable
```

**Rationale**: PEP 585 moved abstract base classes to `collections.abc`. For runtime type checking, these should be imported from their proper location.

## Testing & Verification

### Build Verification
```bash
$ uv build
Successfully built dist\repomapper-0.1.1.tar.gz
Successfully built dist\repomapper-0.1.1-py3-none-any.whl
```

### Package Contents Verification
```bash
$ python -m zipfile -l dist/repomapper-0.1.1-py3-none-any.whl | grep queries
repomapper/queries/tree-sitter-language-pack/...
repomapper/queries/tree-sitter-languages/...
```
✅ Queries directory properly included

### Linting Verification
```bash
$ uv run ruff check src/
All checks passed!
```

### Functional Testing
```bash
$ uv run repomapper . --verbose
Chat files: []
Repo-map: 0.8 k-tokens
Generated map: 2720 chars, ~868 tokens
[... full repo map output ...]
```
✅ Tool generates output correctly

### Original Bug Verification
The original issue was:
```
map_string is None
(None, FileReport(excluded={}, definition_matches=0, reference_matches=0, total_files_considered=105))
```

This is now fixed:
1. ✅ Tuple properly unpacked (no longer `(None, FileReport(...))`)
2. ✅ Queries included, so definitions/references > 0
3. ✅ Map generated successfully

## Migration Path for Reviewers

To understand this PR, review in this conceptual order:

1. **Package structure changes** (`pyproject.toml`, file moves to `src/`)
2. **Critical bug fix** (`repomap.py:200` tuple unpacking)
3. **Import updates** (relative imports in all files)
4. **Manual bug fixes** (sections 8.1-8.6 above)
5. **Automatic modernization** (type hints and imports)

## Backwards Compatibility

- ✅ **API**: No changes to public API or CLI interface
- ✅ **Python version**: Relaxed to 3.11+ (more compatible)
- ✅ **Functionality**: All features work identically
- ⚠️ **Development**: Anyone with local dev environments should delete old `.venv` and rebuild

## Risk Assessment

**Low Risk Changes**:
- Type hint modernization (no runtime impact)
- Import organization (no runtime impact)
- Package structure (standard practice)

**Medium Risk Changes**:
- Build system change (mitigated by testing)
- Tuple unpacking fix (fixes existing bug)

**Testing Mitigation**:
- Functional testing on real codebase
- Build verification
- Package contents verification

## Recommendations

For future PRs of this scope, consider:
1. Commit file moves separately
2. Commit auto-fixes separately
3. Commit manual changes separately

This would create 3 smaller, easier-to-review commits.
