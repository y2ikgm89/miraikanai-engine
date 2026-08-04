# Secure MCP Tunnel Independent Runtime Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 廃止済みPersonal Skillへ依存せず、`G:\workspace`だけを読むexact 4-Tool MCP Serverを独立runtimeとして構築し、既存`g-workspace-readonly` Secure MCP Tunnel Profileへ安全に接続して、現在のWindowsユーザーのログオン30秒後に最小権限で自動起動する。

**Architecture:** `%LOCALAPPDATA%\MCP\g-workspace-readonly`を独立したPython project兼Local Git repositoryとし、CPython 3.14.6、uv lock、専用venv、MCP Python SDK v2の`MCPServer`でread-only Serverを実装する。Server単体／stdio検証が完了するまで既存Profileを変更せず、sanitized baselineとbackupを取得後にMCP commandだけを切り替える。Tunnel ready後、current-user／interactive-token／limited-run-levelのScheduled Taskを登録し、manual start、性能guardrail、次回実ログオンの順に受入確認する。

**Tech Stack:** CPython 3.14.6、uv 0.12.1、MCP Python SDK `mcp==2.0.0`、Pillow 12.3.0、pypdf 6.14.2、python-docx 1.2.0、python-pptx 1.0.2、openpyxl 3.1.5、pytest 9.1.1、PowerShell 7／Windows PowerShell ScheduledTasks、OpenAI `tunnel-client` v0.0.10、Windows Task Scheduler

## Global Constraints

- 承認済み設計は`docs/superpowers/specs/2026-08-04-secure-mcp-tunnel-independent-runtime-design.md`である。本Planはその実装手順であり、Engine Architecture Owner、Schema、Registry、Fixture、Build、CIまたはProduct状態を変更しない。
- 廃止済み`C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro`を読込み、復元、再作成、fallbackまたはcompatibility routeとして使用しない。
- runtime rootは`%LOCALAPPDATA%\MCP\g-workspace-readonly`、allowed rootはreal path `G:\workspace`だけとする。CLI、environment、ProfileまたはTool argumentから別rootを指定できる入口を作らない。
- 公開Toolは`list_allowed_directories`、`list_directory`、`search`、`fetch`のexact 4件だけとする。write、edit、move、create、delete、shell、process、arbitrary URL、aliasまたはdeprecated Toolを公開しない。
- 全Tool annotationは`readOnlyHint=true`、`destructiveHint=false`、`idempotentHint=true`、`openWorldHint=false`の完全一致とする。
- `search`と`fetch`は明示的structured outputを返す。`fetch`はreadしたsource bytesのbyte countとlowercase SHA-256をmetadataへ含め、Imageはsignature確認済みsource bytesをMCP `ImageContent`として返す。
- direct textは2 MiB、imageは10 MiB、PDF／Officeは50 MiB、抽出textは400,000 Unicode characters、directory listingは2,000 entries、search resultは100 entriesを上限とする。超過はsilent truncationせずtyped errorにする。
- TextはUTF-8／UTF-8 BOM／UTF-16 BOMだけを扱う。ImageはPNG、JPEG、WebP、non-animated GIFだけをsignatureとparserの両方で検証する。PDFはtext-bearingだけ、OfficeはDOCX／PPTX／XLSXだけを扱う。
- legacy／macro-enabled／encrypted Office、scan-only PDF、SVG、unknown binary、size limit超過、result limit超過はfail closedにする。Formula、macro、OLE、external link、embedded executableまたはURLを実行しない。
- Path inputはroot-relative pathまたはServer発行の同じstable root-relative IDだけを受ける。absolute、UNC、device、drive-relative、NUL、`..`、reparse解決後のroot escapeを拒否し、read直前にstat、regular-file、size、containmentを再検証する。
- dependencyはexact direct versionを`pyproject.toml`へ固定し、transitive dependency、artifact URL、hashを`uv.lock`へ固定する。runtime実行時にuv、PATH Python、system package、`npx -y`またはnetwork installを使わない。
- Profile commandは専用`.venv\Scripts\python.exe -m readonly_local_files.server`のabsolute commandだけとし、Skill、Codex plugin、Codex internal runtimeまたはwrapperへ向けない。
- API key value、Tunnel ID、Organization／Workspace ID、Profile body、environment一覧またはallowed root外PathをRepository、Task XML、command line、test output、Tool errorまたはperformance recordへ残さない。
- Profile migration前にbackupを取り、変更対象をMCP commandだけに限定する。Tunnel identity、User-scope API key reference、association、health addressは維持する。unknown Profile fieldがあれば自動変換せず停止する。
- Scheduled Taskはcurrent exact user、`Interactive` logon type、`Limited` run level、Logon Trigger 30秒delay、`IgnoreNew`、restart 1分／最大3回、network required、start when available、unlimited execution、visible definitionへ固定する。管理者、SYSTEM、password保存、Machine-scope key、Service、infinite retryは使わない。
- 既存Process、foreign port ownerまたはunhealthy processを自動killしない。自分がこの手順で開始したexact Processだけを、明示したrollback時に停止できる。
- performance guardrailはready後10分idle平均CPU 1%未満、Tunnel＋Python combined working set 256 MiB未満、restart／reconnect loopなし、`G:\workspace`へのwrite 0件とする。超過時はTaskを無効化し、on-demand運用へ戻す。
- Local runtime projectは秘密を含まないLocal Git repositoryとして初期化する。`.venv`、cache、log、backup、acceptance fixture、measurement raw dataは`.gitignore`対象とし、各TaskのGREEN後に小さくcommitする。
- 実ログオンTriggerの受入は実際の次回ログオンまで完了扱いにしない。登録直後のmanual start成功とlogon自動起動実証を分離して報告する。

## File and Responsibility Map

| File／State | Responsibility |
| --- | --- |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\.python-version` | Python 3.14 series selection |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\pyproject.toml` | package metadata、exact direct dependencies、pytest settings |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\uv.lock` | complete resolved dependency graph、source artifacts、hashes |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\.gitignore` | venv、cache、logs、private evidence exclusion |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\__init__.py` | package version |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\models.py` | limits、Pydantic result models、typed errors、annotation contract |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\paths.py` | root-relative validation、real-path containment、directory reads |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\extractors.py` | signature detection、Text／Image／PDF／Office extraction、digest |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\server.py` | exact 4 Tools、search／fetch dispatch、self-test、stdio entry point |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_catalog.py` | exact catalog、annotations、schemas、forbidden capability tests |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_paths.py` | traversal、UNC／device、reparse escape、sort、listing limit tests |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_extractors.py` | supported／blocked format、digest、size／result limit tests |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_stdio.py` | official MCP Client initialize、tools/list、tools/call、stdout purity |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\ops\Test-RuntimeContract.ps1` | version、lock、venv、catalog、root、Profile、health contract checks |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\ops\Register-LogonTask.ps1` | exact least-privilege Scheduled Task registration／verification |
| `%LOCALAPPDATA%\MCP\g-workspace-readonly\ops\Measure-IdleRuntime.ps1` | bounded idle CPU／memory／I/O／network sampling and guardrail verdict |
| `%APPDATA%\tunnel-client\g-workspace-readonly.yaml` | existing Tunnel identity and settings; only MCP command migrates |
| Task `OpenAI Secure MCP Tunnel - G Workspace Readonly` | current-user logon startup and bounded recovery |
| `docs/superpowers/specs/2026-08-04-secure-mcp-tunnel-independent-runtime-design.md` | adopted design and final sanitized local-state disposition |

---

### Task 1: Capture Sanitized Baselines and Establish the Independent Project

**Files:**
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\.gitignore`
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\.python-version`
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\pyproject.toml`
- Create test: `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_catalog.py`
- Read only: `%APPDATA%\tunnel-client\g-workspace-readonly.yaml`

**Interfaces:** Produces an isolated Git-backed project, exact dependency manifest, secret-free baseline facts, and the first expected RED test. It does not modify the active Profile, start Tunnel, or register a Task.

- [ ] **Step 1: Reconfirm exact local prerequisites without secrets**

Run:

```powershell
$ErrorActionPreference = 'Stop'
$runtimeRoot = Join-Path $env:LOCALAPPDATA 'MCP\g-workspace-readonly'
$profilePath = Join-Path $env:APPDATA 'tunnel-client\g-workspace-readonly.yaml'
$tunnelExe = Join-Path $env:LOCALAPPDATA `
    'OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe'

if (Test-Path -LiteralPath $runtimeRoot) {
    throw "Runtime root already exists; inspect before overwrite: $runtimeRoot"
}
if (-not (Test-Path -LiteralPath 'G:\workspace' -PathType Container)) {
    throw 'Allowed root is absent.'
}
if (-not (Test-Path -LiteralPath $profilePath -PathType Leaf)) {
    throw 'Tunnel Profile is absent.'
}
if (-not (Test-Path -LiteralPath $tunnelExe -PathType Leaf)) {
    throw 'tunnel-client is absent.'
}

python --version
uv self version
& $tunnelExe --version
if (-not [Environment]::GetEnvironmentVariable(
    'CONTROL_PLANE_API_KEY', 'User'
)) { throw 'User-scope runtime key reference is absent.' }
if ([Environment]::GetEnvironmentVariable(
    'CONTROL_PLANE_API_KEY', 'Machine'
)) { throw 'Unexpected Machine-scope runtime key exists.' }
'prerequisites-ok'
```

Expected: Python `3.14.6`、uv `0.12.1`、tunnel-client `0.0.10` and `prerequisites-ok`. API key value is never printed.

- [ ] **Step 2: Capture only sanitized Profile invariants and Process／port baseline**

Read the Profile in memory and assert, without emitting values:

```powershell
$profileText = Get-Content -LiteralPath $profilePath -Raw
if ($profileText -notmatch 'env:CONTROL_PLANE_API_KEY') {
    throw 'Profile does not use the User-scope environment reference.'
}
if ($profileText -notmatch [regex]::Escape(
    '.agents/skills/collaborating-with-chatgpt-pro'
)) { throw 'Expected obsolete MCP command was not found.' }

$tunnelProcesses = @(Get-CimInstance Win32_Process |
    Where-Object { $_.ExecutablePath -eq $tunnelExe })
$portOwners = @(Get-NetTCPConnection -LocalPort 8080 -State Listen `
    -ErrorAction SilentlyContinue)
[pscustomobject]@{
    ProfileExists = $true
    UsesUserKeyReference = $true
    ObsoleteCommandConfirmed = $true
    TunnelProcessCount = $tunnelProcesses.Count
    HealthPortOwnerCount = $portOwners.Count
}
```

Expected: obsolete command confirmed; Tunnel Process and port owner are both 0 before migration. If not, stop and diagnose rather than terminate anything.

- [ ] **Step 3: Create the isolated project metadata**

Create directories and initialize local Git, then create `.gitignore`:

```gitignore
.venv/
.pytest_cache/
__pycache__/
*.py[cod]
*.log
.coverage
htmlcov/
.private/
measurements/raw/
```

Create `.python-version`:

```text
3.14
```

Create `pyproject.toml`:

```toml
[build-system]
requires = ["hatchling==1.31.0"]
build-backend = "hatchling.build"

[project]
name = "readonly-local-files"
version = "1.0.0"
description = "Least-privilege read-only local file MCP server"
requires-python = ">=3.14,<3.15"
dependencies = [
  "mcp==2.0.0",
  "openpyxl==3.1.5",
  "Pillow==12.3.0",
  "pypdf==6.14.2",
  "python-docx==1.2.0",
  "python-pptx==1.0.2",
]

[dependency-groups]
dev = ["pytest==9.1.1"]

[tool.hatch.build.targets.wheel]
packages = ["src/readonly_local_files"]

[tool.pytest.ini_options]
addopts = "-ra --strict-markers"
testpaths = ["tests"]
```

Run:

```powershell
New-Item -ItemType Directory -Path $runtimeRoot | Out-Null
New-Item -ItemType Directory -Path `
    (Join-Path $runtimeRoot 'src\readonly_local_files'), `
    (Join-Path $runtimeRoot 'tests'), `
    (Join-Path $runtimeRoot 'ops') | Out-Null
git -C $runtimeRoot init
uv lock --directory $runtimeRoot --python 3.14.6
uv sync --directory $runtimeRoot --locked --python 3.14.6
```

Expected: `.venv\Scripts\python.exe` is CPython 3.14.6, `uv.lock` exists, and no system Python package changes.

- [ ] **Step 4: Write the first missing-catalog test**

Create `tests\test_catalog.py`:

```python
import asyncio

from mcp import Client

from readonly_local_files.server import create_server

EXPECTED_TOOLS = (
    "list_allowed_directories",
    "list_directory",
    "search",
    "fetch",
)


def test_catalog_is_exact_read_only_surface(tmp_path) -> None:
    async def scenario() -> None:
        async with Client(create_server(tmp_path), raise_exceptions=True) as client:
            listed = await client.list_tools()
        assert tuple(tool.name for tool in listed.tools) == EXPECTED_TOOLS
        for tool in listed.tools:
            assert tool.annotations.read_only_hint is True
            assert tool.annotations.destructive_hint is False
            assert tool.annotations.idempotent_hint is True
            assert tool.annotations.open_world_hint is False

    asyncio.run(scenario())
```

- [ ] **Step 5: Run the test and verify RED**

Run:

```powershell
uv run --directory $runtimeRoot python -m pytest tests\test_catalog.py -q
```

Expected: collection fails only because `readonly_local_files.server` does not exist.

- [ ] **Step 6: Commit the project baseline**

After reviewing that neither Profile nor secret was copied into the project:

```powershell
git -C $runtimeRoot add .gitignore .python-version pyproject.toml uv.lock tests
git -C $runtimeRoot diff --cached --check
git -C $runtimeRoot commit -m 'chore: initialize independent readonly MCP runtime'
```

Expected: one root commit containing only metadata, lock, and the RED test.

---

### Task 2: Implement Models and the Exact MCP Tool Catalog

**Files:**
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\__init__.py`
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\models.py`
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\server.py`
- Modify test: `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_catalog.py`

**Interfaces:** Produces immutable limits, Pydantic result models, `ReadonlyFileError`, exact `ToolAnnotations`, `create_server(root: Path) -> MCPServer`, and four registered Tool definitions with temporary typed `not-ready` behavior.

- [ ] **Step 1: Extend catalog tests for schemas and forbidden capabilities**

Add assertions that every Tool has object input/output schemas, that only `search` and `fetch` accept their documented arguments, and that none of these fragments occur in any Tool name: `write`, `edit`, `move`, `create`, `delete`, `shell`, `process`, `http`, `url`.

Expected schema contracts:

```python
EXPECTED_INPUT_KEYS = {
    "list_allowed_directories": set(),
    "list_directory": {"path"},
    "search": {"query"},
    "fetch": {"id"},
}
```

- [ ] **Step 2: Define limits, typed errors, and structured models**

Create `__init__.py`:

```python
__version__ = "1.0.0"
```

In `models.py`, define these public contracts:

```python
from dataclasses import dataclass, field
from pathlib import Path
from typing import Any, Final, Literal

from mcp.types import ToolAnnotations
from pydantic import BaseModel, ConfigDict, Field

ALLOWED_ROOT: Final = Path(r"G:\workspace")
TOOL_NAMES: Final = (
    "list_allowed_directories",
    "list_directory",
    "search",
    "fetch",
)
DIRECT_TEXT_LIMIT: Final = 2 * 1024 * 1024
IMAGE_LIMIT: Final = 10 * 1024 * 1024
DOCUMENT_LIMIT: Final = 50 * 1024 * 1024
RESULT_TEXT_LIMIT: Final = 400_000
DIRECTORY_ENTRY_LIMIT: Final = 2_000
SEARCH_RESULT_LIMIT: Final = 100

READ_ONLY_ANNOTATIONS: Final = ToolAnnotations(
    read_only_hint=True,
    destructive_hint=False,
    idempotent_hint=True,
    open_world_hint=False,
)


@dataclass(frozen=True)
class ReadonlyFileError(Exception):
    code: str
    message: str
    details: dict[str, int | str] = field(default_factory=dict)


class StrictModel(BaseModel):
    model_config = ConfigDict(extra="forbid")


class AllowedDirectory(StrictModel):
    id: str
    path: str


class AllowedDirectoriesResult(StrictModel):
    directories: list[AllowedDirectory]


class DirectoryEntry(StrictModel):
    id: str
    name: str
    type: Literal["file", "directory"]
    size_bytes: int | None


class DirectoryResult(StrictModel):
    path: str
    entries: list[DirectoryEntry]


class SearchHit(StrictModel):
    id: str
    title: str
    url: str = ""
    metadata: dict[str, Any]


class SearchResult(StrictModel):
    results: list[SearchHit]


class FetchMetadata(StrictModel):
    source_bytes: int = Field(ge=0)
    source_sha256: str = Field(pattern=r"^[0-9a-f]{64}$")
    detected_mime: str
    extractor: str
    extraction_status: Literal["complete"] = "complete"
    format: dict[str, Any] = Field(default_factory=dict)


class FetchResult(StrictModel):
    id: str
    title: str
    text: str
    url: str = ""
    metadata: FetchMetadata
```

Also define internal immutable `ResolvedFile` and `ExtractedArtifact` dataclasses. `ExtractedArtifact` contains text, MIME, extractor, source byte count, SHA-256, format metadata, and optional original image bytes／MIME.

- [ ] **Step 3: Create an exact `MCPServer` catalog**

In `server.py`, use only the official v2 high-level API:

```python
from pathlib import Path
from typing import Annotated

from mcp.server import MCPServer
from mcp.types import CallToolResult, TextContent

from .models import (
    ALLOWED_ROOT,
    AllowedDirectoriesResult,
    DirectoryResult,
    FetchResult,
    READ_ONLY_ANNOTATIONS,
    SearchResult,
)


def create_server(root: Path = ALLOWED_ROOT) -> MCPServer:
    mcp = MCPServer("G Workspace Readonly")

    @mcp.tool(
        name="list_allowed_directories",
        annotations=READ_ONLY_ANNOTATIONS,
    )
    def list_allowed_directories(
    ) -> Annotated[CallToolResult, AllowedDirectoriesResult]:
        raise RuntimeError("not-ready")

    @mcp.tool(name="list_directory", annotations=READ_ONLY_ANNOTATIONS)
    def list_directory(
        path: str,
    ) -> Annotated[CallToolResult, DirectoryResult]:
        raise RuntimeError("not-ready")

    @mcp.tool(name="search", annotations=READ_ONLY_ANNOTATIONS)
    def search(query: str) -> Annotated[CallToolResult, SearchResult]:
        raise RuntimeError("not-ready")

    @mcp.tool(name="fetch", annotations=READ_ONLY_ANNOTATIONS)
    def fetch(id: str) -> Annotated[CallToolResult, FetchResult]:
        raise RuntimeError("not-ready")

    return mcp
```

Use Pydantic return models to generate explicit output schema. `fetch` uses `Annotated[CallToolResult, FetchResult]` because it may return both structured metadata and MCP `ImageContent`.

- [ ] **Step 4: Run catalog tests and verify GREEN**

Run:

```powershell
uv run --directory $runtimeRoot python -m pytest tests\test_catalog.py -q
uv lock --directory $runtimeRoot --check
uv sync --directory $runtimeRoot --check
```

Expected: exact four names, exact annotation values, output schemas present, forbidden capability count 0.

- [ ] **Step 5: Commit models and catalog**

```powershell
git -C $runtimeRoot add src tests\test_catalog.py
git -C $runtimeRoot diff --cached --check
git -C $runtimeRoot commit -m 'feat: define exact readonly MCP catalog'
```

---

### Task 3: Implement Windows Path Containment and Directory Reads

**Files:**
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\paths.py`
- Create test: `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_paths.py`
- Modify: `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\server.py`

**Interfaces:** Produces `PathPolicy`, `resolve_file`, `resolve_directory`, stable forward-slash IDs, deterministic non-recursive listing, and read-time containment verification.

- [ ] **Step 1: Write traversal, type, case, and reparse RED tests**

Parameterize rejection of:

```python
INVALID_PATHS = (
    r"C:\Windows\win.ini",
    r"C:Windows\win.ini",
    r"\\server\share\file.txt",
    r"\\?\C:\Windows\win.ini",
    r"\\.\PhysicalDrive0",
    "../outside.txt",
    "safe/../../outside.txt",
    "nul\x00suffix",
)
```

Add tests for a valid mixed-separator relative file, Windows case-insensitive containment, missing file, directory passed as file, file passed as directory, sorted non-recursive listing, 2,001-entry failure, and symlink／junction escape. Symlink creation may skip only when Windows denies creation; add a separate monkeypatched resolved-path escape test that never skips.

- [ ] **Step 2: Verify RED**

```powershell
uv run --directory $runtimeRoot python -m pytest tests\test_paths.py -q
```

Expected: collection fails only because `readonly_local_files.paths` is absent.

- [ ] **Step 3: Implement `PathPolicy`**

Use this core shape in `paths.py`:

```python
import os
from pathlib import Path, PureWindowsPath

from .models import (
    DIRECTORY_ENTRY_LIMIT,
    DirectoryEntry,
    ReadonlyFileError,
    ResolvedFile,
)


class PathPolicy:
    def __init__(self, root: Path) -> None:
        self.root = root.resolve(strict=True)

    def _parts(self, value: str, *, allow_dot: bool = False) -> tuple[str, ...]:
        if "\x00" in value or (not value and not allow_dot):
            raise ReadonlyFileError("path-outside-root", "Invalid relative path.")
        candidate = PureWindowsPath(value.replace("/", "\\"))
        if (
            candidate.is_absolute()
            or candidate.drive
            or candidate.root
            or any(part == ".." for part in candidate.parts)
        ):
            raise ReadonlyFileError("path-outside-root", "Invalid relative path.")
        return tuple(part for part in candidate.parts if part not in ("", "."))

    def _is_contained(self, target: Path) -> bool:
        root = os.path.normcase(str(self.root))
        resolved = os.path.normcase(str(target))
        try:
            return os.path.commonpath((root, resolved)) == root
        except ValueError:
            return False
```

Complete resolution so it joins only validated components, calls `resolve(strict=True)`, rejects resolved escape, verifies file／directory type, calls `stat()` immediately before returning, and emits generic errors without echoing untrusted absolute values. `revalidate_for_read` repeats resolve、containment、regular-file、size immediately before opening.

Directory listing uses `Path.iterdir()` without recursion, rejects reparse targets that resolve outside root, sorts by `(name.casefold(), name)`, gives stable forward-slash `id`, and raises `directory-too-large` on the 2,001st entry.

- [ ] **Step 4: Wire the first two Tools**

Add a deterministic structured-result helper shared by all Tools, then replace the first two temporary bodies:

```python
policy = PathPolicy(root)

def _structured_result(payload: StrictModel) -> CallToolResult:
    structured = payload.model_dump(mode="json")
    encoded = json.dumps(
        structured,
        ensure_ascii=False,
        sort_keys=True,
        separators=(",", ":"),
    )
    return CallToolResult(
        content=[TextContent(type="text", text=encoded)],
        structured_content=structured,
    )


def list_allowed_directories(
) -> Annotated[CallToolResult, AllowedDirectoriesResult]:
    payload = AllowedDirectoriesResult(
        directories=[AllowedDirectory(id="workspace", path="G:/workspace")]
    )
    return _structured_result(payload)


def list_directory(
    path: str,
) -> Annotated[CallToolResult, DirectoryResult]:
    payload = DirectoryResult(
        path=path,
        entries=policy.list_directory(path),
    )
    return _structured_result(payload)
```

The public path string is fixed and not derived from arbitrary input.

- [ ] **Step 5: Verify GREEN and commit**

```powershell
uv run --directory $runtimeRoot python -m pytest `
    tests\test_paths.py tests\test_catalog.py -q
git -C $runtimeRoot add src tests
git -C $runtimeRoot diff --cached --check
git -C $runtimeRoot commit -m 'feat: enforce workspace path containment'
```

Expected: all cases pass; only an actual Windows privilege denial may skip the live symlink case.

---

### Task 4: Implement Strict Text and Image Extraction

**Files:**
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\extractors.py`
- Create test: `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_extractors.py`

**Interfaces:** Produces `extract_file(policy: PathPolicy, relative_id: str) -> ExtractedArtifact`, signature-first media detection, exact source bytes／digest, and typed fail-closed errors.

- [ ] **Step 1: Write Text／Image RED tests**

Create fixtures and assertions for:

- UTF-8、UTF-8 BOM、UTF-16 LE BOM、UTF-16 BE BOM exact decoding;
- NUL-containing／invalid UTF-8／unknown-extension binary rejection;
- PNG、JPEG、WebP、single-frame GIF original-byte return and MIME;
- animated GIF、SVG、extension-signature mismatch rejection;
- 2 MiB + 1 text and 10 MiB + 1 recognized image rejection;
- 400,001-character extracted text rejection;
- source byte count and `sha256(source.read_bytes()).hexdigest()` equality;
- source changed between resolve and open returning `source-changed`.

Use one helper:

```python
def assert_error(
    policy: PathPolicy,
    relative_id: str,
    expected: str,
) -> None:
    with pytest.raises(ReadonlyFileError) as caught:
        extract_file(policy, relative_id)
    assert caught.value.code == expected
```

- [ ] **Step 2: Verify RED**

```powershell
uv run --directory $runtimeRoot python -m pytest `
    tests\test_extractors.py -q
```

Expected: collection fails only because `extractors.py` is absent.

- [ ] **Step 3: Implement verified bounded reads**

`extractors.py` must lazy-import heavy parsers inside their specific functions. The shared read helper follows this shape:

```python
from hashlib import sha256
import os
import stat


def _read_source(
    policy: PathPolicy,
    relative_id: str,
    limit: int,
) -> tuple[ResolvedFile, bytes]:
    resolved = policy.resolve_file(relative_id)
    checked = policy.revalidate_for_read(resolved)
    with checked.path.open("rb") as stream:
        opened = os.fstat(stream.fileno())
        if not stat.S_ISREG(opened.st_mode):
            raise ReadonlyFileError("path-not-file", "Target is not a file.")
        if opened.st_size > limit:
            raise ReadonlyFileError(
                "file-too-large",
                "Source exceeds the byte limit.",
                {"limit": limit, "observed": opened.st_size},
            )
        data = stream.read(limit + 1)
        if len(data) != opened.st_size or stream.read(1):
            raise ReadonlyFileError("source-changed", "Source changed during read.")
    return checked, data
```

Before returning, repeat containment against the opened file's final resolved path where the platform API permits. Never follow a resolved path outside the root.

- [ ] **Step 4: Implement signature-first Text／Image dispatch**

Detection rules:

```python
TEXT_EXTENSIONS = frozenset({
    ".c", ".cc", ".cfg", ".cmake", ".cpp", ".csv", ".h", ".hpp",
    ".ini", ".json", ".log", ".md", ".ps1", ".py", ".rst",
    ".toml", ".tsv", ".txt", ".xml", ".yaml", ".yml",
})
IMAGE_SIGNATURES = (
    (b"\x89PNG\r\n\x1a\n", "image/png"),
    (b"\xff\xd8\xff", "image/jpeg"),
    (b"GIF87a", "image/gif"),
    (b"GIF89a", "image/gif"),
)
```

Recognize WebP only as `RIFF`、4-byte size field、`WEBP`. Reject SVG before generic text. Text decoding permits UTF-8／BOM variants only and converts `UnicodeDecodeError` to `unsupported-media-type`. For images, use lazy `PIL.Image.open(BytesIO(data))`, `verify()`, exact format, and `n_frames == 1` for GIF. Do not transcode; retain verified original bytes.

Return `ExtractedArtifact` with lowercase SHA-256, exact length, `status="complete"`, and no absolute source path.

- [ ] **Step 5: Verify GREEN and commit**

```powershell
uv run --directory $runtimeRoot python -m pytest `
    tests\test_extractors.py tests\test_paths.py -q
git -C $runtimeRoot add src tests
git -C $runtimeRoot diff --cached --check
git -C $runtimeRoot commit -m 'feat: add bounded text and image extraction'
```

---

### Task 5: Implement PDF and Modern Office Extraction

**Files:**
- Modify: `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\extractors.py`
- Modify test: `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_extractors.py`

**Interfaces:** Extends `extract_file` for text PDF、DOCX、PPTX、XLSX and produces ordered text plus non-executing format metadata.

- [ ] **Step 1: Add PDF／Office RED fixtures**

Generate in test temp directories:

- one PDF page containing a unique text marker;
- one encrypted PDF and one blank／scan-only PDF;
- DOCX paragraphs plus a 2-cell table;
- PPTX with two slides in known order;
- XLSX with two sheets, scalar cells, and one formula cell;
- OOXML packages containing `vbaProject.bin`, `externalLinks/`, or `embeddings/`;
- `.docm`、`.pptm`、`.xlsm`、`.doc`、`.ppt`、`.xls`;
- one valid-looking PDF and one OOXML package larger than 50 MiB.

Expected typed errors:

| Case | Code |
| --- | --- |
| encrypted PDF／encrypted OOXML | `encrypted-file` |
| blank textless PDF | `scan-only-pdf` |
| macro or legacy Office | `unsupported-office-format` |
| malformed parser input | `extractor-failed` |
| document above limit | `file-too-large` |

- [ ] **Step 2: Verify focused RED**

```powershell
uv run --directory $runtimeRoot python -m pytest `
    tests\test_extractors.py `
    -k 'pdf or docx or pptx or xlsx or office' -q
```

Expected: new cases fail because document dispatch is absent.

- [ ] **Step 3: Implement PDF extraction**

Lazy-import `pypdf.PdfReader`; read from `BytesIO(data)` with strict parsing. Reject `reader.is_encrypted`. Extract pages in order and join with exactly two newlines. If all page text is blank, return `scan-only-pdf`. Record only `{"pages": <count>}`. Convert parser exceptions to generic `extractor-failed` without exception repr or source content.

- [ ] **Step 4: Inspect OOXML package before parser use**

Require both ZIP signature and allowlisted extension. Inspect case-folded ZIP entry names before opening with Office libraries:

```python
MACRO_ENTRIES = frozenset({
    "word/vbaproject.bin",
    "ppt/vbaproject.bin",
    "xl/vbaproject.bin",
})
```

Reject macro entry or macro-enabled content type. Do not open entries under `externalLinks/` or `embeddings/`; only record their counts. Treat the OLE compound signature with modern extensions as `encrypted-file`, and with legacy extensions as `unsupported-office-format`. Never invoke Office COM, LibreOffice, link resolution, formula calculation, or external network access.

- [ ] **Step 5: Extract ordered DOCX／PPTX／XLSX text**

- DOCX: paragraphs in document order, then table row／cell order.
- PPTX: slide order, shape order, text-frame paragraph order.
- XLSX: workbook sheet order and row／column order using `read_only=True`. Load `data_only=False` for formulas and separately `data_only=True` for already-cached values. Report formula string and cached value but never evaluate it.
- Enforce the 400,000-character result limit after joining.
- Record only bounded counts: pages、slides、sheets、formula cells、external links、embedded objects、macro-present false.

- [ ] **Step 6: Verify all extractors and commit**

```powershell
uv run --directory $runtimeRoot python -m pytest tests\test_extractors.py -q
git -C $runtimeRoot add src tests
git -C $runtimeRoot diff --cached --check
git -C $runtimeRoot commit -m 'feat: add safe document extraction'
```

Expected: every allowlisted format passes and every blocked class returns the exact typed error.

---

### Task 6: Implement Search, Fetch, Errors, Self-test, and Stdio

**Files:**
- Modify: `%LOCALAPPDATA%\MCP\g-workspace-readonly\src\readonly_local_files\server.py`
- Modify test: `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_catalog.py`
- Create test: `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_stdio.py`

**Interfaces:** Completes all four Tools, deterministic search, structured fetch with optional `ImageContent`, sanitized typed errors, `--self-test`, and production stdio entry point.

- [ ] **Step 1: Write in-memory Tool behavior RED tests**

Using `async with Client(create_server(tmp_path), raise_exceptions=True)`, test:

- root listing and deterministic directory output;
- empty search query returns `results=[]`;
- filename and extractable text matches return stable relative IDs;
- 101st match returns `result-too-large`, not partial results;
- unsupported／encrypted／scan-only files are skipped by search;
- text fetch structured content has exact text／byte count／SHA-256;
- image fetch has first `TextContent` carrying the JSON-equivalent structured result and exactly one `ImageContent`;
- malformed extra／missing／wrong-type arguments return a typed generic failure without traceback;
- `write_file` is unknown／method-not-found;
- Tool error has no source content, secret, or external absolute path.

- [ ] **Step 2: Verify RED**

```powershell
uv run --directory $runtimeRoot python -m pytest tests\test_catalog.py -q
```

Expected: missing behavior produces focused failures while catalog tests remain GREEN.

- [ ] **Step 3: Implement deterministic search**

Search rules are exact:

1. `query.strip().casefold()`; empty query returns no results.
2. Walk root deterministically without following directory symlinks or junctions.
3. Resolve and contain every visited directory; prune any failure.
4. Match normalized relative filename first.
5. For supported non-image sources within their limit, extract and search case-folded text.
6. Skip typed unsupported、encrypted、scan-only、oversized sources; propagate path policy or internal integrity errors.
7. Emit `SearchHit(id=<forward-slash relative path>, title=<basename>, url="", metadata={"match": "name"|"content"})`.
8. Raise `result-too-large` on the 101st match; never return a partial first 100.

- [ ] **Step 4: Implement fetch and direct MCP result**

Build a `FetchResult`, serialize it deterministically, and use:

```python
import base64
import json
from typing import Annotated

from mcp.types import CallToolResult, ImageContent, TextContent


def _call_result(
    payload: StrictModel,
    artifact: ExtractedArtifact,
) -> Annotated[CallToolResult, FetchResult]:
    structured = payload.model_dump(mode="json")
    content = [
        TextContent(
            type="text",
            text=json.dumps(
                structured,
                ensure_ascii=False,
                sort_keys=True,
                separators=(",", ":"),
            ),
        )
    ]
    if artifact.image_data is not None:
        content.append(
            ImageContent(
                type="image",
                data=base64.b64encode(artifact.image_data).decode("ascii"),
                mime_type=artifact.image_mime,
            )
        )
    return CallToolResult(content=content, structured_content=structured)
```

Before final implementation, verify the installed v2 field spellings with a focused unit test. Use the Python fields `structured_content` and `mime_type`; assert wire serialization emits the MCP wire aliases `structuredContent` and `mimeType`.

- [ ] **Step 5: Convert expected file failures to sanitized Tool errors**

Tool handlers catch only `ReadonlyFileError`; unexpected exceptions propagate to MCP diagnostics and tests, preventing silent corruption. Expected failures return `CallToolResult(isError=True)` with JSON:

```json
{"error":{"code":"<typed-code>","details":{},"message":"<generic message>"}}
```

Never echo the supplied path, query, source text, parser repr, API key or Profile data.

- [ ] **Step 6: Write subprocess stdio RED tests**

Use the official process client:

```python
from mcp import ClientSession, StdioServerParameters
from mcp.client.stdio import stdio_client


async def scenario(python_executable: str) -> None:
    parameters = StdioServerParameters(
        command=python_executable,
        args=["-m", "readonly_local_files.server"],
    )
    async with stdio_client(parameters) as streams:
        read_stream, write_stream = streams
        async with ClientSession(read_stream, write_stream) as session:
            await session.initialize()
            listed = await session.list_tools()
            assert [tool.name for tool in listed.tools] == list(EXPECTED_TOOLS)
```

Create one GUID fixture directory directly under `G:\workspace`, record its exact resolved path, and remove only that verified fixture in test cleanup. Exercise all four Tools, unknown Tool failure, and source digest. Confirm no CLI or environment root override works. Confirm stdout carries protocol frames only and diagnostics go to stderr.

- [ ] **Step 7: Implement CLI and self-test**

Normal entry point calls only `create_server(ALLOWED_ROOT).run()` and writes no banner to stdout. `--self-test` validates root, catalog, annotations, installed interpreter, and sibling `uv.lock`, then emits exactly one sanitized JSON line:

```json
{"allowed_root":"G:\\workspace","annotations_exact":true,"catalog":["list_allowed_directories","list_directory","search","fetch"],"forbidden_tools":[],"python":"3.14.6","uv_lock_sha256":"<64 lowercase hex>"}
```

Missing root, lock, dependency, catalog mismatch or Python mismatch exits non-zero.

- [ ] **Step 8: Verify full Server GREEN and commit**

```powershell
$venvPython = Join-Path $runtimeRoot '.venv\Scripts\python.exe'
uv lock --directory $runtimeRoot --check
uv sync --directory $runtimeRoot --locked
uv sync --directory $runtimeRoot --check
& $venvPython -m pytest (Join-Path $runtimeRoot 'tests') -q
& $venvPython -m readonly_local_files.server --self-test
git -C $runtimeRoot status --short
git -C $runtimeRoot diff --check
git -C $runtimeRoot add src tests uv.lock
git -C $runtimeRoot commit -m 'feat: complete readonly MCP server'
```

Expected: all tests pass, self-test emits one line, lock is current, and runtime Git is clean after commit.

---

### Task 7: Add Reproducible Runtime, Task, and Measurement Operations

**Files:**
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\ops\Test-RuntimeContract.ps1`
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\ops\Register-LogonTask.ps1`
- Create: `%LOCALAPPDATA%\MCP\g-workspace-readonly\ops\Measure-IdleRuntime.ps1`
- Create tests: `%LOCALAPPDATA%\MCP\g-workspace-readonly\tests\test_ops_contract.py`

**Interfaces:** Produces idempotent, secret-free checks and exact Task construction. Scripts return structured objects or sanitized JSON and never mutate the Profile unless the dedicated migration Task is being executed.

- [ ] **Step 1: Write static／mocked operation RED tests**

Tests load the PowerShell files as text and, where practical, invoke them with injected test paths. Require:

- all runtime／Tunnel executable references are absolute;
- no Skill path、`npx`、`pip install`、Machine key、SYSTEM、Highest、password or API key value appears;
- runtime contract checks Python 3.14.6、uv lock current、venv sync、self-test catalog／root／annotations、Profile command equality;
- Task contract includes current user、Interactive、Limited、30 seconds、IgnoreNew、restart 1 minute／3、network required、start when available、execution time limit zero／unlimited;
- measurement samples only exact tunnel-client child set and never writes to `G:\workspace`;
- unexpected Process owner、port owner、Profile field or Task field returns failure rather than repair／kill.

- [ ] **Step 2: Implement `Test-RuntimeContract.ps1`**

The script accepts `-IncludeProfile`, `-IncludeTunnelHealth`, and `-AsJson`. It validates:

```powershell
$ExpectedRuntimeRoot = Join-Path $env:LOCALAPPDATA 'MCP\g-workspace-readonly'
$ExpectedPython = Join-Path $ExpectedRuntimeRoot '.venv\Scripts\python.exe'
$ExpectedModule = 'readonly_local_files.server'
$ExpectedRoot = 'G:\workspace'
$ExpectedTools = @(
    'list_allowed_directories',
    'list_directory',
    'search',
    'fetch'
)
```

It runs `uv lock --check`, `uv sync --check`, exact venv Python version, package import, and `--self-test`; parses one JSON line; compares exact catalog、root、annotation and lock hash. With `-IncludeProfile`, it verifies the Profile has one stdio command and exact normalized absolute command. With `-IncludeTunnelHealth`, it requires loopback `/healthz` and `/readyz` HTTP 200 and exact one expected Tunnel Process. The returned object contains booleans and generic reason codes only.

- [ ] **Step 3: Implement exact Scheduled Task registration**

`Register-LogonTask.ps1` accepts `-WhatIfContract`, `-Register`, and `-Verify`; only `-Register` writes Task Scheduler state. Core construction:

```powershell
$TaskName = 'OpenAI Secure MCP Tunnel - G Workspace Readonly'
$TunnelExe = Join-Path $env:LOCALAPPDATA `
    'OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe'
$CurrentUser = [System.Security.Principal.WindowsIdentity]::GetCurrent().Name
$Action = New-ScheduledTaskAction `
    -Execute $TunnelExe `
    -Argument 'run --profile g-workspace-readonly' `
    -WorkingDirectory (Split-Path -Parent $TunnelExe)
$Trigger = New-ScheduledTaskTrigger -AtLogOn -User $CurrentUser
$Trigger.Delay = 'PT30S'
$Principal = New-ScheduledTaskPrincipal `
    -UserId $CurrentUser `
    -LogonType Interactive `
    -RunLevel Limited
$Settings = New-ScheduledTaskSettingsSet `
    -MultipleInstances IgnoreNew `
    -RestartInterval (New-TimeSpan -Minutes 1) `
    -RestartCount 3 `
    -ExecutionTimeLimit ([TimeSpan]::Zero) `
    -RunOnlyIfNetworkAvailable `
    -StartWhenAvailable `
    -AllowStartIfOnBatteries `
    -DontStopIfGoingOnBatteries
```

Before registration, assert runtime contract and Profile contract. If the Task already exists, export a sanitized contract and stop unless every field already matches; never overwrite a divergent Task silently. Register with `Register-ScheduledTask -TaskName $TaskName -Action $Action -Trigger $Trigger -Principal $Principal -Settings $Settings` and no `-Password`.

`-Verify` inspects action、argument、working directory、principal SID／user、logon type、run level、trigger user／delay、multiple instances、restart、network、start-when-available、execution limit、hidden false. It also exports Task XML in memory and rejects occurrences of API key names with inline values, Tunnel ID, Profile body, Skill path, password logon, or encoded command.

- [ ] **Step 4: Implement bounded idle measurement**

`Measure-IdleRuntime.ps1` accepts `-DurationMinutes 10 -SampleSeconds 5`. It resolves exact Tunnel Process and its direct Python child by executable path, then samples Process CPU deltas、working set、read／write transfer counts and network adapter byte deltas. It records aggregate sanitized fields only:

```text
duration_seconds
sample_count
average_cpu_percent
max_combined_working_set_bytes
disk_read_delta_bytes
disk_write_delta_bytes
network_delta_bytes
process_count_stable
workspace_write_count
guardrail_passed
```

Workspace write count is measured by a before／after snapshot of file path、length、last-write ticks under `G:\workspace`; snapshots stay in ignored `.private` storage and are removed after aggregate comparison. A difference fails the guardrail and reports count only, not content.

- [ ] **Step 5: Verify operation tests and commit**

```powershell
uv run --directory $runtimeRoot python -m pytest `
    tests\test_ops_contract.py -q
& (Join-Path $runtimeRoot 'ops\Test-RuntimeContract.ps1') -AsJson
& (Join-Path $runtimeRoot 'ops\Register-LogonTask.ps1') `
    -WhatIfContract
git -C $runtimeRoot add ops tests
git -C $runtimeRoot diff --cached --check
git -C $runtimeRoot commit -m 'feat: add secure runtime operations'
```

Expected: pre-Profile runtime checks pass; Profile and health checks remain explicitly not requested; Task Scheduler state is unchanged.

---

### Task 8: Migrate Only the Active Profile MCP Command

**Files／State:**
- Backup: `%APPDATA%\tunnel-client\.private-backups\g-workspace-readonly-<timestamp>.yaml`
- Modify: `%APPDATA%\tunnel-client\g-workspace-readonly.yaml`
- Read only: independent runtime, current Process list, TCP 8080 owner

**Interfaces:** Replaces one YAML scalar only. Produces a validated Profile pointing to the dedicated venv while retaining a private rollback source.

- [ ] **Step 1: Re-run all pre-migration gates**

Require:

```powershell
uv run --directory $runtimeRoot python -m pytest tests -q
& (Join-Path $runtimeRoot '.venv\Scripts\python.exe') `
    -m readonly_local_files.server --self-test
& (Join-Path $runtimeRoot 'ops\Test-RuntimeContract.ps1') -AsJson
```

Also require exact Tunnel Process count 0 and TCP 8080 listener count 0. If either is non-zero, stop without kill.

- [ ] **Step 2: Validate the current Profile shape and take a private backup**

Accept only this key shape: top-level `config_version`、`control_plane`、`health`、`admin_ui`、`log`、`mcp`; one `mcp.commands` item; item keys `channel` and `command`; `api_key` exactly references `env:CONTROL_PLANE_API_KEY`; health listener is loopback. Reject unknown or duplicate command fields.

Create the backup directory with current-user-only ACL, copy the exact Profile once, and store its path only in the local working transcript. Do not add it to either Git repository.

- [ ] **Step 3: Replace exactly one command line using `apply_patch`**

The only intended YAML content change is:

```diff
-      command: "C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/mcp-server/.venv/Scripts/python.exe -m readonly_local_files.server"
+      command: "C:/Users/y2ikg/AppData/Local/MCP/g-workspace-readonly/.venv/Scripts/python.exe -m readonly_local_files.server"
```

Use `apply_patch` after confirming the exact old line. Do not rewrite／reformat the Profile and do not change channel、Tunnel ID、API key reference、base URL、health、admin UI or log fields.

- [ ] **Step 4: Verify sanitized semantic diff**

Compare backup and current Profile in memory after replacing Tunnel ID and all values except the MCP command with fixed redaction tokens. Require exactly one semantic difference: `mcp.commands[0].command`. Verify old Skill path count 0 and new exact command count 1.

- [ ] **Step 5: Run official `doctor`**

```powershell
$tunnelExe = Join-Path $env:LOCALAPPDATA `
    'OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe'
& $tunnelExe doctor --profile g-workspace-readonly --explain --json
if ($LASTEXITCODE -ne 0) { throw 'Tunnel doctor failed.' }
```

Parse JSON and report only check names、pass／fail counts and generic remediation. Expected: failure count 0, including MCP command executable.

- [ ] **Step 6: Roll back on any migration failure**

If semantic diff or doctor fails, use `apply_patch` to restore the exact command from the backup, rerun doctor, and leave the backup retained. Do not start Tunnel or register Task.

---

### Task 9: Start and Validate the Tunnel Before Persistence

**State:**
- Start: exact `tunnel-client.exe run --profile g-workspace-readonly`
- Observe: loopback health／ready, Process tree, MCP stderr diagnostics
- Do not yet register Scheduled Task

**Interfaces:** Produces one healthy Tunnel Process using the migrated Profile and one on-demand runtime acceptance result.

- [ ] **Step 1: Start one hidden exact Process and retain its PID**

```powershell
$tunnelDir = Split-Path -Parent $tunnelExe
$started = Start-Process `
    -FilePath $tunnelExe `
    -ArgumentList @('run', '--profile', 'g-workspace-readonly') `
    -WorkingDirectory $tunnelDir `
    -WindowStyle Hidden `
    -PassThru
```

This is an operator-started acceptance Process, not yet the Scheduled Task instance.

- [ ] **Step 2: Wait boundedly for health and ready**

Poll `http://127.0.0.1:8080/healthz` and `/readyz` for at most 90 seconds with one-second interval. Require HTTP 200 for both and Process still alive. On timeout, capture sanitized doctor and Process count, stop only `$started` if its PID and executable still match, then roll back Profile command.

- [ ] **Step 3: Verify exact Process topology and second-start behavior**

Require exactly one Tunnel executable Process and only expected dedicated-venv Python MCP child processes. Issue a second start attempt with the same Profile and confirm it exits／refuses without increasing steady-state Tunnel Process count. Do not kill a duplicate if identity is ambiguous; fail and stop persistence work.

- [ ] **Step 4: Verify the tunneled runtime contract locally**

Run `Test-RuntimeContract.ps1 -IncludeProfile -IncludeTunnelHealth -AsJson`. Re-run direct stdio client tests separately so a healthy Tunnel does not substitute for MCP catalog verification. Expected: exact root、catalog、annotations、lock、Profile、health and ready all true.

- [ ] **Step 5: Retain the Profile backup through final acceptance**

Do not delete the backup. Record only its existence and SHA-256 in private ignored state; never put the Profile content or hash-to-identity mapping in the repository.

---

### Task 10: Register, Manually Exercise, and Measure the Logon Task

**State:**
- Create: Task `OpenAI Secure MCP Tunnel - G Workspace Readonly`
- Modify: current-user Task Scheduler state only
- Observe: manual start, idle metrics, network recovery

**Interfaces:** Converts the verified on-demand route into bounded current-user persistence, then disables it automatically if the local performance／safety guardrail fails.

- [ ] **Step 1: Stop only the operator-started acceptance Process**

Confirm PID and executable still equal `$started`, then request graceful termination and wait up to 15 seconds. Do not target by broad process name. Require health port free and Tunnel Process count 0 before Task registration.

- [ ] **Step 2: Register and verify the exact Task**

```powershell
$register = Join-Path $runtimeRoot 'ops\Register-LogonTask.ps1'
& $register -Register
if ($LASTEXITCODE -ne 0) { throw 'Task registration failed.' }
& $register -Verify
if ($LASTEXITCODE -ne 0) { throw 'Task contract verification failed.' }
```

Expected: one current-user Task, limited privilege, no stored password or secret, and exact contract equality. If verification fails, disable the newly registered Task and stop.

- [ ] **Step 3: Manual Task start acceptance**

Use `Start-ScheduledTask -TaskName $TaskName`, then apply the same 90-second health／ready gate. Require exact one Tunnel Process, expected child Python, Task `LastTaskResult` compatible with a long-running action, and no restart loop. A second `Start-ScheduledTask` must not increase Process count because `IgnoreNew` is active.

- [ ] **Step 4: Measure 10-minute idle performance**

```powershell
& (Join-Path $runtimeRoot 'ops\Measure-IdleRuntime.ps1') `
    -DurationMinutes 10 `
    -SampleSeconds 5
```

Expected: average CPU <1%、max combined working set <256 MiB、stable process count、workspace write count 0. This is a Local guardrail, not an OpenAI／Python guarantee.

- [ ] **Step 5: Measure representative fetch and network recovery**

Create one GUID-named test fixture below `G:\workspace` with known text, call it through the official local stdio client, verify digest, and remove only that verified fixture. Then disconnect only the Tunnel's effective network path if a safe reversible adapter-level test is available; otherwise skip and report. During recovery, require no unbounded Process growth、CPU loop or log growth. Never disable the user's broader network without explicit confirmation.

- [ ] **Step 6: Enforce guardrail disposition**

If any performance、write、restart or Process guardrail fails:

```powershell
Disable-ScheduledTask -TaskName $TaskName
```

Keep the healthy on-demand Profile if Tunnel／MCP correctness passed; revert Profile only when correctness or security failed. Report the disabled state and evidence without claiming auto-start completion.

- [ ] **Step 7: Record next-login acceptance as pending**

At this point, manual Task start may be verified. Mark actual logon trigger acceptance `pending-next-login`. At the next real login, verify start time is at least 30 seconds after user logon, Task instance count 1, health／ready 200, exact runtime contract, and no restart storm. Only then mark logon auto-start fully accepted.

---

### Task 11: Final Verification, Evidence Update, and Handoff

**Files:**
- Modify: `docs/superpowers/specs/2026-08-04-secure-mcp-tunnel-independent-runtime-design.md`
- Read: independent runtime Git status／log、Profile contract、Task contract
- Do not add: Profile backup、secret、raw performance snapshots、runtime source copy

**Interfaces:** Produces a truthful repository record of sanitized local materialization and a concise operator handoff.

- [ ] **Step 1: Run complete current verification**

Runtime:

```powershell
uv lock --directory $runtimeRoot --check
uv sync --directory $runtimeRoot --check
& (Join-Path $runtimeRoot '.venv\Scripts\python.exe') -m pytest `
    (Join-Path $runtimeRoot 'tests') -q
& (Join-Path $runtimeRoot 'ops\Test-RuntimeContract.ps1') `
    -IncludeProfile -IncludeTunnelHealth -AsJson
& (Join-Path $runtimeRoot 'ops\Register-LogonTask.ps1') -Verify
git -C $runtimeRoot status --short
git -C $runtimeRoot log -5 --oneline
```

Tunnel:

- sanitized doctor failure count 0;
- health and ready HTTP 200;
- exact one Tunnel Process and expected Python child set;
- second start does not create a duplicate;
- Task definition contains no secret and matches exact contract.

Performance:

- report aggregate CPU／working-set／I/O／network values;
- report workspace write count 0;
- report network-recovery test as passed or explicitly not run;
- report logon trigger as `pending-next-login` until real login.

- [ ] **Step 2: Update only the local-state lines in the approved design**

Change `Local runtime実装状態` from `absent and unchanged` to a truthful sanitized result such as:

```text
- Local runtime実装状態: materialized and locally verified on 2026-08-04; real-logon trigger acceptance pending
```

Add a short non-normative evidence subsection with runtime root、Python／uv／Tunnel versions、exact tool count、doctor／health／ready、Task manual run、performance guardrail and remaining next-login check. Do not add Profile body、Tunnel ID、API key、Task XML or raw measurements.

- [ ] **Step 3: Verify and commit repository evidence**

```powershell
git diff --check
git status --short
git diff --stat
git diff -- `
    docs/superpowers/specs/2026-08-04-secure-mcp-tunnel-independent-runtime-design.md
git add `
    docs/superpowers/specs/2026-08-04-secure-mcp-tunnel-independent-runtime-design.md
git diff --cached --check
git commit -m 'docs: record independent secure MCP runtime verification'
```

Expected: only the approved design's local-state evidence changes; no Architecture Owner、index、Engine implementation or secret is added.

- [ ] **Step 4: Perform final secret／placeholder scans**

Run against both tracked trees:

```powershell
git grep -n -E 'CONTROL_PLANE_API_KEY=|tunnel_[A-Za-z0-9_-]+|TO[D]O|T[B]D|PLACEH[O]LDER'
git -C $runtimeRoot grep -n -E `
    'CONTROL_PLANE_API_KEY=|tunnel_[A-Za-z0-9_-]+|TO[D]O|T[B]D|PLACEH[O]LDER|collaborating-with-chatgpt-pro'
```

Expected: no secret assignment、Tunnel identity、placeholder or obsolete Skill reference. The literal environment variable name may appear only as a reference contract, never with a value.

- [ ] **Step 5: Final handoff**

Report:

- independent runtime location and clean local Git commit IDs;
- Python 3.14.6、uv 0.12.1、MCP 2.0.0、Tunnel 0.0.10;
- exact four Tools and `G:\workspace` containment;
- Profile command-only migration and retained private rollback backup;
- Task manual-start result and measured idle guardrail;
- real-login trigger acceptance as the only expected pending item, if not yet performed;
- Task disable command for immediate on-demand fallback:

```powershell
Disable-ScheduledTask `
    -TaskName 'OpenAI Secure MCP Tunnel - G Workspace Readonly'
```

Do not claim Engine implementation、Architecture materialization、ChatGPT workspace authorization or next-login acceptance from Local ready evidence alone.

---

## Plan Self-Review Checklist

| Design acceptance | Covered by |
| --- | --- |
| 1. Skill／Codex lifecycle independence | Tasks 1、6、8、final obsolete-path scan |
| 2. Python 3.14、uv lock、dedicated venv、absolute command | Tasks 1、6、7、8 |
| 3. Exact four read-only Tools and no write／shell | Tasks 2、6、11 |
| 4. `G:\workspace` real-path containment | Tasks 3、4、6 |
| 5. Command-only Profile migration | Task 8 |
| 6. Current-user delayed least-privilege startup | Tasks 7、10 |
| 7. No secret in Task／command／log／Repository | Global constraints、Tasks 7、8、11 |
| 8. Doctor、health、ready、stdio、manual Task、performance | Tasks 6、9、10、11 |
| 9. Real-logon verification remains explicit | Tasks 10、11 |
| 10. No Engine／Architecture／Product claim upgrade | Global constraints、Task 11 |

- [ ] Every design acceptance item 1–10 maps to at least one Task and verification step.
- [ ] File map contains every planned created／modified file and no abolished Skill path.
- [ ] Public Tool set is exact four everywhere; annotations and output contracts are consistent.
- [ ] Limits, allowed／blocked media, path rules, digest rules, and fail-closed behavior are consistent across tests and implementation steps.
- [ ] Profile migration changes one scalar only, has a rollback source, and never prints identity／secret values.
- [ ] Scheduled Task contract is current-user、interactive、limited、30-second delay、single-instance、bounded restart and passwordless everywhere.
- [ ] Performance thresholds are identified as Local guardrails, not official guarantees.
- [ ] Manual Task acceptance and actual real-login acceptance are never conflated.
- [ ] Every production-code Task begins with RED, reaches GREEN, and includes a focused commit.
- [ ] Incomplete-marker tokens、ellipsis placeholders and invented future artifacts are absent.
- [ ] Repository final checks are `git diff --check`、`git status --short`、`git diff --stat` plus full changed-file diff review.
