# Read-only Local File Search Index Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use
> superpowers:subagent-driven-development (recommended) or
> superpowers:executing-plans to implement this plan task-by-task. Steps use
> checkbox (`- [ ]`) syntax for tracking.

**Goal:** Replace the timeout-prone full-root and content-scanning `search`
implementation with a bounded process-local metadata index while preserving
the official OpenAI `search(query)` and `fetch(id)` compatibility shapes.

**Architecture:** Add a focused `search_index.py` module that eagerly builds an
immutable filename/path index, refreshes it after a five-second TTL, and swaps
only complete replacement indexes. `server.py` owns one `ReadonlyRuntime`
containing the existing `PathPolicy` and the new `SearchIndexCache`; `search`
queries metadata only, while `fetch` remains the sole content reader and
extractor.

**Tech Stack:** Python 3.14, official MCP Python SDK `mcp==2.0.0`, Python
standard-library `os.scandir`／`threading.Lock`／`time.monotonic`, pytest 9.1.1,
PowerShell lifecycle tests, Secure MCP Tunnel, Codex in-app Browser.

## Global Constraints

- Source design:
  `docs/superpowers/specs/2026-07-31-chatgpt-pro-readonly-local-files-design.md`.
- Preserve the exact public catalog:
  `list_allowed_directories`, `list_directory`, `search`, `fetch`.
- Preserve official `search` input as exactly one string property, `query`.
- Preserve official `fetch` input as exactly one string property, `id`.
- Keep explicit output schemas and identical JSON in `structuredContent` and
  the first text content item.
- Keep `url: ""` for local files; do not invent a public or `file://` URL.
- Keep all four Tool annotations exactly read-only, non-destructive,
  idempotent, and closed-world.
- Do not add write Tools, aliases, a fifth maintenance Tool, browser upload,
  attachment, paste, Project Source, shell execution, network access,
  persistent index files, SQLite, OS search services, or new dependencies.
- Search file content must never be opened, decoded, hashed, parsed, or
  extracted. `fetch` remains the only content-reading operation.
- Search index TTL is exactly 5 seconds. Build limits are exactly 5 seconds,
  50,000 directories, and 100,000 searchable files.
- Search result limit remains 100; 101 observed matches return
  `result-too-large` rather than partial results.
- A failed refresh returns `search-index-unavailable`; it must not return a
  stale or partially built index.
- Search discovery excludes the exact case-insensitive directory names in the
  approved design. Direct `list_directory` and `fetch` access remain governed
  by the existing Task Contract and path policy.
- Keep Python `>=3.14,<3.15` and all dependency versions exact.
- This is a breaking internal package change: bump the package and MCP server
  version from `1.0.0` to `2.0.0`; do not keep the old `execute_tool` signature
  or content-scan implementation.
- The Personal Skill directory is not Git-backed. Take a fresh task-local
  baseline before editing, but do not claim Personal Skill commits.
- Repository commits contain only repository evidence and planning documents.
- Browser acceptance uses the exact Project
  `AIネイティブC++ゲームエンジンプロジェクト`, memory
  `プロジェクトのみ`, response performance `非常に高い`, and app
  `G Workspace Readonly`.
- Treat the precise index algorithm, limits, exclusions, and response mode as
  Miraikanai Project decisions, not universal OpenAI recommendations.

## File Map

| Path | Responsibility |
| --- | --- |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\search_index.py` | Immutable index, bounded builder, TTL refresh, deterministic ranking |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\models.py` | Shared searchable extensions, exclusions, and exact limits |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\extractors.py` | Consume the shared text-extension allowlist without changing extraction behavior |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\server.py` | `ReadonlyRuntime`, Tool dispatch, compatibility payloads, MCP version |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\__init__.py` | Package version `2.0.0` |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\pyproject.toml` | Distribution version `2.0.0`; dependencies unchanged |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests\test_search_index.py` | Index behavior, exclusion, limits, refresh, no-content-read regressions |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests\test_catalog.py` | Runtime dispatch and official compatibility result regression |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests\test_stdio.py` | Real fixed-root stdio and `known.md` deadline regression |
| `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\ensure_secure_mcp_tunnel.ps1` | Existing controlled lifecycle; no production edit expected |
| `docs/superpowers/specs/2026-07-31-chatgpt-pro-readonly-local-files-design.md` | Post-implementation evidence and final Browser disposition |
| `docs/superpowers/plans/2026-07-31-readonly-local-file-search-index.md` | Execution checkboxes and exact verification evidence |

---

### Task 1: Freeze the Regression and Index Contract

**Files:**

- Create:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests\test_search_index.py`
- Modify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests\test_stdio.py`
- Read:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\server.py`

**Interfaces:**

- Consumes: existing `PathPolicy`, `ReadonlyFileError`, and fixed production
  root `ALLOWED_ROOT`.
- Produces the wished-for API:
  `SearchIndexCache(policy, *, clock=monotonic, refresh_seconds=5.0,
  build_seconds=5.0, directory_limit=50_000, file_limit=100_000)` and
  `SearchIndexCache.search(query, result_limit=100)`.

- [x] **Step 1: Take a fresh non-repository baseline**

Use a task-specific temporary directory and preserve the exact Personal Skill
state before editing:

```powershell
$source = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
$stamp = Get-Date -Format 'yyyyMMdd-HHmmss'
$baseline = Join-Path $env:TEMP (
    'readonly-search-index-' + $stamp
)
New-Item -ItemType Directory -Path $baseline | Out-Null
Copy-Item -LiteralPath $source -Destination $baseline -Recurse
Resolve-Path -LiteralPath $baseline
```

Expected: one absolute `%TEMP%` path is printed. Record it in the execution
evidence without copying secrets or Tunnel configuration into the repository.

- [x] **Step 2: Write the failing index behavior tests**

Create `tests/test_search_index.py` with real filesystem fixtures and
hand-derived expected values:

```python
from pathlib import Path

import pytest

from readonly_local_files.models import ReadonlyFileError
from readonly_local_files.paths import PathPolicy
from readonly_local_files import search_index
from readonly_local_files.search_index import SearchIndexCache


class FakeClock:
    def __init__(self) -> None:
        self.value = 100.0

    def __call__(self) -> float:
        return self.value

    def advance(self, seconds: float) -> None:
        self.value += seconds


class StepClock:
    def __init__(self, step: float) -> None:
        self.value = 0.0
        self.step = step

    def __call__(self) -> float:
        self.value += self.step
        return self.value


def test_search_ranks_exact_filename_before_path_match(
    tmp_path: Path,
) -> None:
    (tmp_path / "Known.md").write_text("do not read", encoding="utf-8")
    nested = tmp_path / "known"
    nested.mkdir()
    (nested / "Other.md").write_text("do not read", encoding="utf-8")

    cache = SearchIndexCache(PathPolicy(tmp_path))

    matches = cache.search("known")

    assert [entry.relative_id for entry in matches] == [
        "Known.md",
        "known/Other.md",
    ]


def test_search_requires_all_casefolded_query_tokens(
    tmp_path: Path,
) -> None:
    target = tmp_path / "Plans" / "Quarterly Report.md"
    target.parent.mkdir()
    target.write_text("do not read", encoding="utf-8")
    (tmp_path / "Quarterly.md").write_text("do not read", encoding="utf-8")

    cache = SearchIndexCache(PathPolicy(tmp_path))

    assert [
        entry.relative_id
        for entry in cache.search("QUARTERLY report")
    ] == ["Plans/Quarterly Report.md"]


def test_search_never_opens_artifact_content(
    tmp_path: Path,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    (tmp_path / "Known.md").write_text(
        "CONTENT-MUST-NOT-BE-READ",
        encoding="utf-8",
    )

    def fail_open(*args: object, **kwargs: object) -> None:
        raise AssertionError("search opened artifact content")

    monkeypatch.setattr(Path, "open", fail_open)

    cache = SearchIndexCache(PathPolicy(tmp_path))

    assert [entry.relative_id for entry in cache.search("known.md")] == [
        "Known.md"
    ]
    assert cache.search("CONTENT-MUST-NOT-BE-READ") == ()


@pytest.mark.parametrize(
    "excluded",
    [
        ".git",
        ".hg",
        ".svn",
        ".venv",
        "venv",
        "node_modules",
        "__pycache__",
        ".mypy_cache",
        ".pytest_cache",
        ".ruff_cache",
        ".cache",
        "build",
        "dist",
        "out",
        "target",
        "bin",
        "obj",
        "coverage",
    ],
)
def test_search_excludes_non_source_directory_names(
    tmp_path: Path,
    excluded: str,
) -> None:
    hidden = tmp_path / excluded
    hidden.mkdir()
    (hidden / "Known.md").write_text("hidden", encoding="utf-8")

    cache = SearchIndexCache(PathPolicy(tmp_path))

    assert cache.search("known.md") == ()


def test_search_ignores_unsupported_extensions(tmp_path: Path) -> None:
    (tmp_path / "Known.exe").write_bytes(b"MZ")
    (tmp_path / "Known.md").write_text("visible", encoding="utf-8")

    cache = SearchIndexCache(PathPolicy(tmp_path))

    assert [entry.relative_id for entry in cache.search("known")] == [
        "Known.md"
    ]


def test_search_refreshes_only_after_five_seconds(
    tmp_path: Path,
) -> None:
    clock = FakeClock()
    (tmp_path / "First.md").write_text("first", encoding="utf-8")
    cache = SearchIndexCache(PathPolicy(tmp_path), clock=clock)
    (tmp_path / "Second.md").write_text("second", encoding="utf-8")

    clock.advance(4.999)
    assert cache.search("second.md") == ()

    clock.advance(0.001)
    assert [entry.relative_id for entry in cache.search("second.md")] == [
        "Second.md"
    ]


def test_failed_refresh_never_returns_stale_results(
    tmp_path: Path,
) -> None:
    clock = FakeClock()
    (tmp_path / "First.md").write_text("first", encoding="utf-8")
    cache = SearchIndexCache(
        PathPolicy(tmp_path),
        clock=clock,
        file_limit=1,
    )
    (tmp_path / "Second.md").write_text("second", encoding="utf-8")
    clock.advance(5.0)

    with pytest.raises(ReadonlyFileError) as caught:
        cache.search("first.md")

    assert caught.value.code == "search-index-unavailable"
    assert caught.value.details == {"limit": 1, "observed": 2}


def test_search_index_enforces_directory_limit(tmp_path: Path) -> None:
    (tmp_path / "A").mkdir()
    (tmp_path / "B").mkdir()
    cache = SearchIndexCache(
        PathPolicy(tmp_path),
        directory_limit=2,
    )

    with pytest.raises(ReadonlyFileError) as caught:
        cache.search("anything")

    assert caught.value.code == "search-index-unavailable"
    assert caught.value.details == {"limit": 2, "observed": 3}


def test_search_index_enforces_build_deadline(tmp_path: Path) -> None:
    (tmp_path / "Known.md").write_text("known", encoding="utf-8")
    cache = SearchIndexCache(
        PathPolicy(tmp_path),
        clock=StepClock(0.3),
        build_seconds=0.5,
    )

    with pytest.raises(ReadonlyFileError) as caught:
        cache.search("known")

    assert caught.value.code == "search-index-unavailable"
    assert caught.value.details["limit_ms"] == 500
    assert caught.value.details["observed_ms"] >= 500


def test_search_index_redacts_scan_errors(
    tmp_path: Path,
    monkeypatch: pytest.MonkeyPatch,
) -> None:
    def fail_scan(path: object) -> object:
        raise PermissionError(r"C:\secret\must-not-leak")

    monkeypatch.setattr(search_index.os, "scandir", fail_scan)
    cache = SearchIndexCache(PathPolicy(tmp_path))

    with pytest.raises(ReadonlyFileError) as caught:
        cache.search("known")

    assert caught.value.code == "search-index-unavailable"
    assert caught.value.details == {"reason": "io-error"}
    assert "secret" not in caught.value.message.casefold()


def test_search_rejects_101_matches_without_partial_results(
    tmp_path: Path,
) -> None:
    for index in range(101):
        (tmp_path / f"Known-{index:03}.md").write_text(
            "do not read",
            encoding="utf-8",
        )

    cache = SearchIndexCache(PathPolicy(tmp_path))

    with pytest.raises(ReadonlyFileError) as caught:
        cache.search("known")

    assert caught.value.code == "result-too-large"
    assert caught.value.details == {"limit": 100, "observed": 101}
```

Each test catches a concrete break: wrong rank, OR-token matching, accidental
content reads, missing exclusions, unsupported discovery, stale refresh,
stale fallback, or partial result truncation.

- [x] **Step 3: Add the failing production-root stdio deadline test**

In `test_production_stdio_exposes_only_fixed_root_read_tools`, retain the
existing generated fixture but change the empty search call to:

```python
initialization = await session.initialize()
assert initialization.server_info.version == "2.0.0"

searched = await asyncio.wait_for(
    session.call_tool(
        "search",
        {"query": target.name},
    ),
    timeout=10,
)
assert searched.is_error is False
assert [
    result["id"]
    for result in searched.structured_content["results"]
] == [f"{relative_directory}/{target.name}"]
```

The generated UUID ensures the result is unique within `G:\workspace`. The
ten-second local test deadline is stricter than the observed Browser timeout
and must not be implemented as a production timeout increase.

- [x] **Step 4: Run RED and confirm both failures are relevant**

Run:

```powershell
$server = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server'
& "$server\.venv\Scripts\python.exe" -m pytest `
    "$server\tests\test_search_index.py" -q
```

Expected: collection fails because
`readonly_local_files.search_index` does not exist.

Then run:

```powershell
& "$server\.venv\Scripts\python.exe" -m pytest `
    "$server\tests\test_stdio.py::test_production_stdio_exposes_only_fixed_root_read_tools" `
    -q
```

Expected: FAIL because the initialized server reports `1.0.0`, or because the
old `search` exceeds ten seconds. Record the observed failure; do not edit the
test to match the old implementation.

---

### Task 2: Implement the Bounded Metadata Index

**Files:**

- Create:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\search_index.py`
- Modify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\models.py`
- Modify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\extractors.py`
- Test:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests\test_search_index.py`

**Interfaces:**

- Consumes: `PathPolicy.root`, `PathPolicy._contained`,
  `ReadonlyFileError`, and the shared extension allowlists.
- Produces:
  `SearchIndexEntry`, immutable `SearchIndex`, and
  `SearchIndexCache.search(query, result_limit=100)`.

- [x] **Step 1: Centralize exact index constants**

Move `TEXT_EXTENSIONS` from `extractors.py` to `models.py`, import it back into
`extractors.py`, and add:

```python
SEARCH_INDEX_REFRESH_SECONDS: Final = 5.0
SEARCH_INDEX_BUILD_SECONDS: Final = 5.0
SEARCH_INDEX_DIRECTORY_LIMIT: Final = 50_000
SEARCH_INDEX_FILE_LIMIT: Final = 100_000

SEARCH_INDEX_EXCLUDED_DIRECTORIES: Final = frozenset(
    {
        ".git",
        ".hg",
        ".svn",
        ".venv",
        "venv",
        "node_modules",
        "__pycache__",
        ".mypy_cache",
        ".pytest_cache",
        ".ruff_cache",
        ".cache",
        "build",
        "dist",
        "out",
        "target",
        "bin",
        "obj",
        "coverage",
    }
)

SEARCHABLE_EXTENSIONS: Final = frozenset(
    TEXT_EXTENSIONS
    | {
        ".png",
        ".jpg",
        ".jpeg",
        ".webp",
        ".gif",
        ".pdf",
        ".docx",
        ".pptx",
        ".xlsx",
    }
)
```

Do not add legacy or macro-enabled Office extensions.

- [x] **Step 2: Implement immutable entries and deterministic search**

Create `search_index.py` with these public structures:

```python
from dataclasses import dataclass
import os
from pathlib import Path
from threading import Lock
from time import monotonic
from typing import Callable

from .models import (
    SEARCHABLE_EXTENSIONS,
    SEARCH_INDEX_BUILD_SECONDS,
    SEARCH_INDEX_DIRECTORY_LIMIT,
    SEARCH_INDEX_EXCLUDED_DIRECTORIES,
    SEARCH_INDEX_FILE_LIMIT,
    SEARCH_INDEX_REFRESH_SECONDS,
    SEARCH_RESULT_LIMIT,
    ReadonlyFileError,
)
from .paths import PathPolicy


Clock = Callable[[], float]


@dataclass(frozen=True)
class SearchIndexEntry:
    relative_id: str
    title: str
    source_bytes: int
    search_key: str
    title_key: str
    stem_key: str


@dataclass(frozen=True)
class SearchIndex:
    entries: tuple[SearchIndexEntry, ...]
    built_at: float

    def search(
        self,
        query: str,
        result_limit: int = SEARCH_RESULT_LIMIT,
    ) -> tuple[SearchIndexEntry, ...]:
        needle = query.strip().casefold()
        if not needle:
            return ()
        tokens = tuple(needle.split())
        ranked: list[tuple[int, str, str, SearchIndexEntry]] = []
        for entry in self.entries:
            if not all(token in entry.search_key for token in tokens):
                continue
            if needle == entry.title_key:
                rank = 0
            elif needle == entry.stem_key:
                rank = 1
            elif all(token in entry.title_key for token in tokens):
                rank = 2
            else:
                rank = 3
            ranked.append(
                (
                    rank,
                    entry.relative_id.casefold(),
                    entry.relative_id,
                    entry,
                )
            )
            if len(ranked) > result_limit:
                raise ReadonlyFileError(
                    "result-too-large",
                    "The search result exceeds the allowed match limit.",
                    {"limit": result_limit, "observed": len(ranked)},
                )
        ranked.sort(key=lambda item: item[:3])
        return tuple(item[3] for item in ranked)


@dataclass(frozen=True)
class SearchIndexState:
    index: SearchIndex | None
    error: ReadonlyFileError | None
    checked_at: float
```

The expected values come from filenames and relative paths only. Do not call
`extract_file`, `Path.open`, `read_bytes`, or `read_text`.

- [x] **Step 3: Implement the complete-or-error builder**

Add the private function with this exact interface:

```python
def _build_index(
    policy: PathPolicy,
    *,
    clock: Clock,
    build_seconds: float,
    directory_limit: int,
    file_limit: int,
) -> SearchIndex:
```

Its implementation must:

1. Starts from `policy.root`.
2. Uses `os.scandir` and deterministic
   `(entry.name.casefold(), entry.name)` sorting.
3. Checks the injected monotonic clock before each directory and entry.
4. Counts the root as directory 1.
5. Skips symlinks, junctions, excluded directory names, and unsupported
   extensions.
6. Resolves every retained directory and rejects any target for which
   `policy._contained(resolved)` is false.
7. Uses `entry.stat(follow_symlinks=False).st_size` for searchable files.
8. Converts IDs with
   `os.path.relpath(entry.path, policy.root).replace("\\", "/")`.
9. Raises `search-index-unavailable` on timeout, I/O error, containment error,
   or either count limit.
10. Returns only after the complete sorted tuple has been constructed.

Use this exact error shape for count limits:

```python
raise ReadonlyFileError(
    "search-index-unavailable",
    "The search index could not be built within its fixed limits.",
    {"limit": file_limit, "observed": searchable_file_count},
)
```

For the time limit use integer milliseconds without paths or exception text:

```python
raise ReadonlyFileError(
    "search-index-unavailable",
    "The search index could not be built within its fixed limits.",
    {
        "limit_ms": int(build_seconds * 1000),
        "observed_ms": int((clock() - started_at) * 1000),
    },
)
```

Translate every `OSError` from `scandir`, entry classification, resolution, or
stat to this path-free error:

```python
raise ReadonlyFileError(
    "search-index-unavailable",
    "The search index could not be built within its fixed limits.",
    {"reason": "io-error"},
) from None
```

- [x] **Step 4: Implement eager build and complete refresh**

Add `SearchIndexCache`:

```python
class SearchIndexCache:
    def __init__(
        self,
        policy: PathPolicy,
        *,
        clock: Clock = monotonic,
        refresh_seconds: float = SEARCH_INDEX_REFRESH_SECONDS,
        build_seconds: float = SEARCH_INDEX_BUILD_SECONDS,
        directory_limit: int = SEARCH_INDEX_DIRECTORY_LIMIT,
        file_limit: int = SEARCH_INDEX_FILE_LIMIT,
    ) -> None:
        self._policy = policy
        self._clock = clock
        self._refresh_seconds = refresh_seconds
        self._build_seconds = build_seconds
        self._directory_limit = directory_limit
        self._file_limit = file_limit
        self._refresh_lock = Lock()
        self._state = self._build_state()

    def _build(self) -> SearchIndex:
        return _build_index(
            self._policy,
            clock=self._clock,
            build_seconds=self._build_seconds,
            directory_limit=self._directory_limit,
            file_limit=self._file_limit,
        )

    def _build_state(self) -> SearchIndexState:
        try:
            index = self._build()
        except ReadonlyFileError as error:
            return SearchIndexState(
                index=None,
                error=error,
                checked_at=self._clock(),
            )
        return SearchIndexState(
            index=index,
            error=None,
            checked_at=index.built_at,
        )

    @staticmethod
    def _unwrap(state: SearchIndexState) -> SearchIndex:
        if state.error is not None:
            raise state.error
        if state.index is None:
            raise AssertionError("Search index state is invalid.")
        return state.index

    def _current(self) -> SearchIndex:
        current = self._state
        if self._clock() - current.checked_at < self._refresh_seconds:
            return self._unwrap(current)
        with self._refresh_lock:
            current = self._state
            if self._clock() - current.checked_at < self._refresh_seconds:
                return self._unwrap(current)
            replacement = self._build_state()
            self._state = replacement
            return self._unwrap(replacement)

    def search(
        self,
        query: str,
        result_limit: int = SEARCH_RESULT_LIMIT,
    ) -> tuple[SearchIndexEntry, ...]:
        return self._current().search(query, result_limit)
```

The constructor eagerly attempts the first build but stores a typed failure
state so Tool catalog discovery can still complete. `_unwrap` exposes that
failure through `execute_tool`, where it becomes a typed Tool error. A failed
refresh atomically replaces the old success state with a failure state; it
must not return `current`. After five seconds, the cache may attempt one new
complete rebuild.

- [x] **Step 5: Run GREEN for the index unit**

Run:

```powershell
& "$server\.venv\Scripts\python.exe" -m pytest `
    "$server\tests\test_search_index.py" -q
```

Expected: all index tests pass with no warnings.

- [x] **Step 6: Run the existing extractor suite**

Run:

```powershell
& "$server\.venv\Scripts\python.exe" -m pytest `
    "$server\tests\test_extractors.py" -q
```

Expected: all existing extractor tests pass, proving that moving
`TEXT_EXTENSIONS` did not change `fetch` behavior.

---

### Task 3: Replace the Old Search Runtime Without Compatibility Code

**Files:**

- Modify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\server.py`
- Modify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\src\readonly_local_files\__init__.py`
- Modify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\pyproject.toml`
- Modify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests\test_catalog.py`
- Modify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server\tests\test_stdio.py`

**Interfaces:**

- Consumes: `SearchIndexCache.search()` from Task 2.
- Produces:
  `ReadonlyRuntime(policy: PathPolicy, search_index: SearchIndexCache)`,
  `create_runtime(root)`, and the breaking
  `execute_tool(runtime, name, arguments)` signature.

- [x] **Step 1: Update catalog tests for the new runtime contract**

In `test_catalog.py`, import `create_runtime` and change every direct call that
passes a `PathPolicy` as the first argument so it constructs one
`ReadonlyRuntime` with `create_runtime(tmp_path)` and passes that runtime to
`execute_tool`.

Change the compatibility test so search discovers by metadata and fetch proves
content reading:

```python
def test_search_and_fetch_use_compatibility_shapes(tmp_path: Path) -> None:
    target = tmp_path / "Known.md"
    target.write_text("unique local evidence", encoding="utf-8")
    runtime = create_runtime(tmp_path)

    search = execute_tool(
        runtime,
        "search",
        {"query": "known.md"},
    )
    assert search.structured_content["results"] == [
        {
            "id": "Known.md",
            "title": "Known.md",
            "url": "",
            "metadata": {
                "source_bytes": len("unique local evidence".encode()),
                "extraction_status": "path-index-match",
            },
        }
    ]
    assert isinstance(search.content[0], TextContent)
    assert json.loads(search.content[0].text) == search.structured_content

    fetched = execute_tool(runtime, "fetch", {"id": "Known.md"})
    assert fetched.structured_content["text"] == "unique local evidence"
    assert json.loads(fetched.content[0].text) == fetched.structured_content
```

Change the 101-match fixture names to `Known-000.md` through
`Known-100.md` and query `known`. Keep the existing exact typed-error
assertions.

- [x] **Step 2: Run RED for the breaking runtime API**

Run:

```powershell
& "$server\.venv\Scripts\python.exe" -m pytest `
    "$server\tests\test_catalog.py" -q
```

Expected: FAIL because `create_runtime` is absent and `execute_tool` still
accepts `PathPolicy`.

- [x] **Step 3: Add `ReadonlyRuntime` and route search to the index**

In `server.py`, delete `_walk_root_files` and the complete two-pass
`search_payload(policy, query)` implementation. Remove now-unused `os`.

Add:

```python
from dataclasses import dataclass

from .search_index import SearchIndexCache


@dataclass(frozen=True)
class ReadonlyRuntime:
    policy: PathPolicy
    search_index: SearchIndexCache


def create_runtime(root: Path = ALLOWED_ROOT) -> ReadonlyRuntime:
    policy = PathPolicy(root)
    return ReadonlyRuntime(
        policy=policy,
        search_index=SearchIndexCache(policy),
    )


def search_payload(
    search_index: SearchIndexCache,
    query: str,
) -> dict[str, Any]:
    return {
        "results": [
            {
                "id": entry.relative_id,
                "title": entry.title,
                "url": "",
                "metadata": {
                    "source_bytes": entry.source_bytes,
                    "extraction_status": "path-index-match",
                },
            }
            for entry in search_index.search(query)
        ]
    }
```

Change `execute_tool` to:

```python
def execute_tool(
    runtime: ReadonlyRuntime,
    name: str,
    arguments: dict[str, Any],
) -> CallToolResult:
```

Use `runtime.policy` for the three path operations and
`runtime.search_index` for `search`. Do not keep an overload, optional cache,
or compatibility wrapper for the old signature.

Change `create_server` to construct exactly one runtime:

```python
def create_server(root: Path = ALLOWED_ROOT) -> Server:
    runtime = create_runtime(root)
    catalog = build_tool_catalog()
```

The Tool handler must pass `runtime` to `execute_tool`.

- [x] **Step 4: Update the public description and version**

Change the `search` Tool description to:

```python
"Search indexed root-relative local filenames and paths."
```

Change all three active versions to `2.0.0`:

```toml
# pyproject.toml
version = "2.0.0"
```

```python
# readonly_local_files/__init__.py
__version__ = "2.0.0"
```

```python
# create_server()
return Server(
    "G Workspace Readonly",
    version="2.0.0",
    on_list_tools=list_tools,
    on_call_tool=call_tool,
)
```

Dependencies and `requirements.lock` must remain byte-identical.

- [x] **Step 5: Refresh editable distribution metadata without dependencies**

Run:

```powershell
Push-Location $server
try {
    & '.\.venv\Scripts\python.exe' -m pip install `
        --no-deps --editable .
}
finally {
    Pop-Location
}
```

Expected: editable package `readonly-local-files==2.0.0` is installed without
resolving or changing dependencies.

- [x] **Step 6: Run GREEN for catalog and stdio regressions**

Run:

```powershell
& "$server\.venv\Scripts\python.exe" -m pytest `
    "$server\tests\test_catalog.py" `
    "$server\tests\test_stdio.py" -q
```

Expected: both files pass. The production-root stdio test must find its UUID
fixture within ten seconds and report server version `2.0.0`.

- [x] **Step 7: Prove the lock did not change**

Run:

```powershell
Get-FileHash -Algorithm SHA256 `
    "$server\requirements.lock" |
    Select-Object -ExpandProperty Hash
```

Expected lowercase-normalized digest:

```text
e5397d790776b5332fc84457d4344a698fb0634530805cebfd042b291a19a474
```

If the digest differs, stop. Do not update the helper fingerprint unless a
dependency change was separately approved.

---

### Task 4: Complete Local Verification

**Files:**

- Verify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\mcp-server`
- Verify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts`
- Verify:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro`

**Interfaces:**

- Consumes: the complete Personal Skill implementation.
- Produces: fresh evidence that code, lifecycle, static contract, Skill
  metadata, and exact runtime catalog are valid before replacing the live
  process.

- [x] **Step 1: Run the complete Server suite**

```powershell
& "$server\.venv\Scripts\python.exe" -m pytest "$server\tests" -q
```

Expected: all tests pass with zero failures and zero warnings. Record the exact
test count.

- [x] **Step 2: Run the original symptom locally**

Use the retained acceptance fixture and measure a direct indexed search:

```powershell
$python = "$server\.venv\Scripts\python.exe"
$code = @'
import json
import time
from pathlib import Path
from readonly_local_files.server import create_runtime, execute_tool

started = time.perf_counter()
result = execute_tool(
    create_runtime(Path(r"G:\workspace")),
    "search",
    {"query": "known.md"},
)
print(json.dumps({
    "elapsed_ms": round((time.perf_counter() - started) * 1000),
    "is_error": bool(result.is_error),
    "result_count": len(result.structured_content["results"]),
}, separators=(",", ":")))
'@
& $python -c $code
```

Expected: `is_error` is false, `result_count` is at least 1, and
`elapsed_ms` is below 5,000. Do not print file content.

- [x] **Step 3: Run lifecycle helper regression tests**

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1'
```

Expected: all lifecycle tests pass. Record the exact total.

- [x] **Step 4: Run the static Skill/runtime contract validator**

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: `TOTAL_FAILURES=0`.

- [x] **Step 5: Run package and Skill validators**

```powershell
& "$server\.venv\Scripts\python.exe" -m pip check
& "$server\.venv\Scripts\python.exe" `
    'C:\Users\y2ikg\.codex\skills\.system\skill-creator\scripts\quick_validate.py' `
    'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
```

Expected: no broken requirements and `Skill is valid!`.

- [x] **Step 6: Run the exact self-test**

```powershell
& "$server\.venv\Scripts\python.exe" `
    -m readonly_local_files.server --self-test
```

Expected: one JSON line with exact four-Tool catalog, exact allowed root,
no forbidden Tools, exact annotations, and the unchanged lock digest.

---

### Task 5: Replace the Live Tunnel Process Safely

**Files:**

- Reuse without expected modification:
  `C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\ensure_secure_mcp_tunnel.ps1`
- Inspect without exposing:
  `%APPDATA%\tunnel-client\g-workspace-readonly.yaml`

**Interfaces:**

- Consumes: verified package `2.0.0` and existing exact-profile lifecycle
  predicates.
- Produces: one newly started Tunnel process running the updated local MCP
  package, with health, ready, Profile, lock, catalog, and root checks true.

- [x] **Step 1: Confirm exactly one live target before stopping**

Dot-source the helper so its tested exact-process predicate is reused:

```powershell
$helper = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\ensure_secure_mcp_tunnel.ps1'
. $helper
$tunnelExe = Join-Path $env:LOCALAPPDATA `
    'OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe'
$targets = @(
    Get-ExpectedTunnelProcesses `
        -ExpectedExecutablePath $tunnelExe `
        -ExpectedProfileName 'g-workspace-readonly' `
        -TimeoutMilliseconds 5000
)
if ($targets.Count -ne 1) {
    throw "Expected exactly one verified Tunnel process."
}
```

Do not print `CommandLine`, environment variables, Profile content, Tunnel ID,
API key, or process ID.

- [x] **Step 2: Stop only the verified process**

```powershell
$targetProcess = Get-Process -Id $targets[0].ProcessId -ErrorAction Stop
$stopped = Stop-ExactTunnelProcess `
    -Process $targetProcess `
    -TimeoutMilliseconds 5000
if (-not $stopped) {
    throw "Verified Tunnel process did not stop cleanly."
}
```

This stop is authorized only for the exact executable and exact
`g-workspace-readonly` Profile confirmed in Step 1.

- [x] **Step 3: Start through the production helper**

```powershell
$raw = & $helper
$state = $raw | ConvertFrom-Json
[pscustomobject]@{
    status = $state.status
    reason = $state.reason
    health = $state.health
    ready = $state.ready
    profile_verified = $state.profile_verified
    lock_verified = $state.lock_verified
    catalog_verified = $state.catalog_verified
    allowed_root_verified = $state.allowed_root_verified
} | ConvertTo-Json -Compress
```

Expected: `status=started`, `reason=ready`, and all six verification booleans
true.

- [x] **Step 4: Verify reuse after the fresh start**

Run the same helper again and project only the safe fields.

Expected: `status=already-running`, `reason=ready`, and all six booleans true.

- [x] **Step 5: Verify no legacy runtime remains**

Inspect processes using exact executable and module-name predicates. Exclude
the current PowerShell command line from any diagnostic count.

Expected: one exact Tunnel process, one current
`readonly_local_files.server` child as applicable, and zero
`@modelcontextprotocol/server-filesystem`／`npx -y` legacy processes.

---

### Task 6: Browser Acceptance Without Upload

**Task status:** `incomplete`／`blocked`. Final responseはT6-R1からT6-R4の
successと一致metadataを報告し、`TimeoutError`は表示されなかったが、completed
Tool cardを永続的に可視確認できた件数は0／4だった。従ってStep 4とformal
acceptanceは未完了であり、成功、accepted、closedとは扱わない。

**Files:**

- No Browser-uploaded files.
- Retained local fixture:
  `G:\workspace\.chatgpt-pro-readonly-mcp-acceptance-2c76ebe0-6eeb-4c67-8d13-44f2c7f12438`

**Interfaces:**

- Consumes: live verified Tunnel, exact four-Tool app, and local fixture IDs.
- Produces: visible Browser evidence that `search(query="known.md")` and the
  subsequent `fetch` of the returned exact ID complete through Secure MCP
  Tunnel.

- [x] **Step 1: Refresh and verify the app catalog**

Use only the Codex in-app Browser. Open the exact app management page, invoke
the app refresh action, and visibly verify these four Actions only:

```text
fetch
list_allowed_directories
list_directory
search
```

Verify all are read Actions and no write／edit／move／create／delete／shell
Action exists.

- [x] **Step 2: Create a fresh exact-Project chat**

Visibly verify:

```text
Project: AIネイティブC++ゲームエンジンプロジェクト
Memory: プロジェクトのみ
Response performance: 非常に高い
App: G Workspace Readonly
```

Do not reuse the failed search chat because it may retain old Tool state.

- [x] **Step 3: Send a metadata-only acceptance prompt**

The prompt must name Tool calls and root-relative IDs but contain no Local
Artifact content:

```text
G Workspace Readonlyだけを使って次を順に実行してください。
1. list_allowed_directories
2. search(query="known.md")
3. search結果から
   .chatgpt-pro-readonly-mcp-acceptance-2c76ebe0-6eeb-4c67-8d13-44f2c7f12438/known.md
   と一致するidを選ぶ
4. fetch(id=<一致したid>)

添付、upload、貼り付け、Project Source、別app、write系Toolは使用しないでください。
最後に各Toolの成功／失敗、fetchしたid、source_bytes、
source_sha256、extraction_statusだけを報告してください。
```

- [ ] **Step 4: Inspect the actual Tool cards**

Require visible completion for `list_allowed_directories`, `search`, and
`fetch`. `search` must not show `TimeoutError`. The returned `id` must equal
the expected root-relative fixture ID.

If ChatGPT stops before the requested sequence, records a Tool failure, or
omits the terminal report, mark Browser acceptance `incomplete`; do not send a
follow-up in the same Skill run and do not fall back to attachment or paste.

- [x] **Step 5: Finalize Browser tabs**

After all Browser inspection, call the Browser tab-finalization operation as
the last Browser operation. Keep only the app-management evidence tab and the
new acceptance chat needed for user inspection.

---

### Task 7: Record Evidence and Close the Change

**Files:**

- Modify:
  `docs/superpowers/specs/2026-07-31-chatgpt-pro-readonly-local-files-design.md`
- Modify:
  `docs/superpowers/plans/2026-07-31-readonly-local-file-search-index.md`
- Modify only after a fully closed consultation if applicable:
  `docs/reviews/README.md`

**Interfaces:**

- Consumes: fresh local, Tunnel, Browser, Git, and retained-fixture evidence.
- Produces: one repository evidence commit and an honest disposition.

- [x] **Step 1: Record exact execution evidence**

Update this plan’s checkboxes and append one dated execution block containing:

- Server test count and exit code.
- Lifecycle test count and exit code.
- Static validator result.
- Skill validator result.
- Lock SHA-256 verification.
- Local `known.md` search elapsed milliseconds and result count.
- Tunnel safe status fields.
- Browser exact Tool sequence and terminal disposition.
- Confirmation that upload, attachment, paste, Project Source, alternate app,
  and write Tool usage were all zero.

Do not record API keys, Tunnel ID, Profile contents, process IDs, Local file
content, or full Browser transcripts.

- [x] **Step 2: Update the approved design with actual outcome**

Change future-tense statements only where runtime evidence now exists.
Distinguish:

- `official-spec`: official single-query `search` and `fetch` compatibility.
- `project-decision`: metadata-only index, exclusions, limits, `非常に高い`.
- `measured`: local elapsed time, test counts, Tunnel state, Browser outcome.

Do not call a Browser-incomplete run accepted or complete.

- [x] **Step 3: Run repository verification**

From `G:\workspace\development\GameEngine\miraikanai-engine`:

```powershell
git diff --check
git status --short
git diff --stat
git diff -- `
    docs/superpowers/specs/2026-07-31-chatgpt-pro-readonly-local-files-design.md `
    docs/superpowers/plans/2026-07-31-readonly-local-file-search-index.md
```

Expected: no whitespace errors and only the intended evidence documents
changed.

- [x] **Step 4: Commit repository evidence**

```powershell
git add -- `
    docs/superpowers/specs/2026-07-31-chatgpt-pro-readonly-local-files-design.md `
    docs/superpowers/plans/2026-07-31-readonly-local-file-search-index.md
git diff --cached --check
git commit -m "docs: record bounded local search verification"
```

Do not add Personal Skill files or temporary baselines to the repository.

- [x] **Step 5: Apply retention rules**

If Browser acceptance is complete and all accepted findings are closed, update
the compact review summary as required and delete only the exact verified
temporary fixture and baseline paths.

If Browser acceptance is incomplete, retain the fixture and baseline, record
why, and report the remaining Browser orchestration risk. Never recursively
delete a computed or unverified path.

## Execution Evidence — 2026-07-31

### Evidence classification

| Class | Recorded outcome |
| --- | --- |
| `official-spec` | OpenAI互換のsingle-property `search(query)`／`fetch(id)` shape、明示的output schema、`structuredContent`と先頭text contentの同値性を維持した。 |
| `project-decision` | metadata-only process-local index、exact exclusions、5秒TTL、5秒／50,000 directory／100,000 searchable file limit、100 result limit、response performance `非常に高い`を採用した。OpenAIの一般推奨とは扱わない。 |
| `measured` | 下記のfresh Local／Tunnel／Browser evidence。Browser formal acceptanceは`incomplete`／`blocked`であり、成功、accepted、closedとは扱わない。 |

### Local and package verification

| Check | Fresh result |
| --- | --- |
| Complete Server suite | exit code `0`; `128 passed in 13.08s` |
| Lifecycle suite | independent child exit code `0`; `35/35 PASS` |
| Static Skill/runtime validator | exit code `0`; `TOTAL_FAILURES=0` |
| Skill validator | exit code `0`; `Skill is valid!` |
| Package consistency | exit code `0`; `pip check` reported no broken requirements |
| Exact self-test | exit code `0`; exact 4 Tool catalog、forbidden Tool 0、annotations exact |
| Lock SHA-256 | `e5397d790776b5332fc84457d4344a698fb0634530805cebfd042b291a19a474` |
| Fresh direct `search(query="known.md")` | 3 fresh processes: `2689ms`、`2762ms`、`2810ms`; all `is_error=false`; all `result_count=1` |
| Direct `list_directory`／`fetch` | `is_error=false`; exact entry 1; expected ID; source bytes `34`; expected SHA-256; `extraction_status=complete` |

Task 1のfresh non-repository baseline、Task 1からTask 3のRED／GREEN、atomic
refresh fix rounds、Windows親HANDLE相対・reparse非追従のsearch／list／fetch、
breaking version `2.0.0`、dependency不変の詳細は各Task reportに記録した。
Personal SkillはGit管理外であり、Personal Skill commitは存在しない。

### Tunnel verification

最終コード反映時のcontrolled replacementではexact targetだけを停止し、fresh
start後のreuseを確認した。続く2回のpreflightは
`status=already-running`、`health=true`、`ready=true`で、
`profile_verified`、`lock_verified`、`catalog_verified`、
`allowed_root_verified`はすべてtrueだった。exact Tunnel processとcurrent runtimeは
各1件、legacy Node／NPX runtimeは0件だった。
Process ID、Tunnel ID、Profile内容、command line、環境変数、secretは記録していない。

### Browser disposition

- exact Project `AIネイティブC++ゲームエンジンプロジェクト`、memory
  `プロジェクトのみ`、response performance `非常に高い`、app
  `G Workspace Readonly`を送信直前に可視確認した。
- app管理画面では`list_allowed_directories`、`list_directory`、`search`、`fetch`の
  exact 4 read Actionsだけを確認した。
- metadata-only promptは1回だけ送信した。最終応答はT6-R1
  `list_allowed_directories`、T6-R2 `search(query="known.md")`、T6-R3 fixture parentの
  `list_directory`、T6-R4 expected IDの`fetch`をすべてsuccessと報告した。
- 最終応答のroot、catalog、Artifact ID、source bytes、source SHA-256、extraction
  status、terminal markerはLocal manifestと一致し、`TimeoutError`、typed Tool
  error、unexpected／write Toolは表示されなかった。
- ただし、応答完了後にindependently visibleなcompleted Tool cardは0／4で、exact
  per-turn Tool-card equalityは成立していない。最終応答の自己申告はactual Tool
  card evidenceの代替にしない。Task 6 reportのhonest incomplete／blocked disposition
  はreview済みだが、Browser acceptanceの成功としては承認されていない。
- Browser attachment、upload、paste、Project Source、alternate app、write Toolの
  使用はいずれも0回だった。同じSkill runでのfollow-upやfallbackは行っていない。

### Retention disposition

Browser formal acceptanceが`incomplete`のため、次のexact evidenceは削除せず保持する。

- fixture:
  `G:\workspace\.chatgpt-pro-readonly-mcp-acceptance-2c76ebe0-6eeb-4c67-8d13-44f2c7f12438`
- Personal Skill baseline:
  `C:\Users\y2ikg\AppData\Local\Temp\readonly-search-index-20260731-175500`
- transient Browser transcript:
  `C:\Users\y2ikg\AppData\Local\Temp\miraikanai-task6-browser-acceptance-root-20260731.md`

File内容とfull transcriptはrepositoryへ記録していない。Consultation closureが未成立の
ため`docs/reviews/README.md`は更新せず、fixture／baseline／transcriptを削除して
いない。残るriskは、Browser UIが4件のcompleted Tool cardを永続表示せず、actual
per-call status／targetのformal reconciliationを完了できないことである。

## Final Verification Matrix

| Requirement | Evidence |
| --- | --- |
| Official `search(query)` shape unchanged | Catalog unit test and app management |
| Exact four read-only Tools | Catalog, self-test, helper, Browser |
| Search never reads content | no-open regression and deleted content scan |
| Filename/path discovery | Index unit and stdio tests |
| Deterministic rank | Literal ordered unit expectations |
| Exclusions exact | Parameterized unit test |
| Five-second refresh | Fake-clock unit test |
| No stale/partial fallback | Refresh-failure unit test |
| 100-result limit | 101-match typed-error test |
| Root-scale timeout fixed | Ten-second stdio test and 3 measured fresh direct calls |
| Path traversal is race-resistant | parent-HANDLE-relative, reparse-nofollow tests and review |
| `fetch` owns stable content read | same-HANDLE byte-read tests, extractor suite, compatibility test |
| `list_directory` never truncates silently | raw-entry limit and reparse/device tests |
| Package breaking version | stdio initialization reports `2.0.0` |
| Lock and dependencies unchanged | SHA-256 and `pip check` |
| Tunnel runs new code | Controlled restart and safe readiness fields |
| No upload path used | Browser inspection and execution evidence |
| Original Browser symptom disposition | Local direct searchは2689–2810msで3回成功し、Browser最終応答に`TimeoutError`はなかった。ただしcompleted Tool card 0／4のためformalには未解決 |
