# ChatGPT Pro Secure MCP Tunnel Auto-start Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** `collaborating-with-chatgpt-pro`の開始時に、固定Profile `g-workspace-readonly`を安全に再利用または自動起動し、ready確認後だけ既存Tunnel preflightへ進む。

**Architecture:** Personal SkillへWindows用lifecycle helperを追加し、Process identityとloopback health endpointsから`reuse`、`start`、`blocked`を決定する。Skill本文とprompt contractはBrowser navigationより前のhelper実行を所有し、既存のTask authorization、ChatGPT app、allowed root、reachability、Tool catalog検証は変更しない。

**Tech Stack:** PowerShell 7、OpenAI `tunnel-client` v0.0.10、Markdown Skill contract、既存PowerShell static validator、Python `quick_validate.py`

## Global Constraints

- 対象Profileは`g-workspace-readonly`だけに固定する。
- Executableは`%LOCALAPPDATA%\OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe`、Profileは`%APPDATA%\tunnel-client\g-workspace-readonly.yaml`を使う。
- Health base URLは`http://127.0.0.1:8080`、startup timeoutは30秒、poll intervalは500msとする。
- `Start-Process -WindowStyle Hidden`で起動し、visible consoleを作らない。
- Readyな既存Processは再利用し、既存の不健康Processまたは未知port ownerをkill、restart、duplicate-startしない。
- Timeout時に停止できるのは、同じhelper invocationが新規作成した正確なPIDだけである。
- Runtime API key、Tunnel ID、Organization ID、Workspace ID、Profile内容をcommand、JSON result、Transcript、永続logへ出さない。
- Local readyをTask authorizationまたはTunnel delivery completenessとみなさない。
- Tunnel Profile、Filesystem MCP、Windows Service、Scheduled Task、`agents/openai.yaml`、Repository `AGENTS.md`を変更しない。
- `tunnel-client`をdownloadまたはupdateしない。確認済みv0.0.10を使う。
- Personal SkillはRepository外にあるため、Skill FileをGit commitしない。編集前copyと完全diffで変更範囲を検証する。
- Repository内では本Plan以外のFileを変更しない。

---

### Task 1: RED Contract and Helper Tests

**Files:**
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/validate_secure_mcp_contract.ps1`
- Create: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/test_ensure_secure_mcp_tunnel.ps1`
- Read: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md`
- Read: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/prompt-generation-contract.md`
- Baseline copy: `%TEMP%/collaborating-with-chatgpt-pro-tunnel-autostart-20260731/`

**Interfaces:**
- Consumes: 既存`Add-Result`／`Has-All` validator APIと承認済み設計
- Produces: `TunnelLifecycleHelperPresent`、`TunnelLifecycleContract`、`TunnelLifecycleSafety` predicatesと、`Resolve-TunnelLifecycleAction`のRED test

- [ ] **Step 1: Capture exact pre-edit Personal Skill baselines**

Run:

```powershell
$skillRoot = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
$baselineRoot = Join-Path $env:TEMP 'collaborating-with-chatgpt-pro-tunnel-autostart-20260731'
if (Test-Path -LiteralPath $baselineRoot) {
    throw "Baseline path already exists: $baselineRoot"
}
New-Item -ItemType Directory -Path (Join-Path $baselineRoot 'references') -Force | Out-Null
New-Item -ItemType Directory -Path (Join-Path $baselineRoot 'scripts') -Force | Out-Null
Copy-Item -LiteralPath (Join-Path $skillRoot 'SKILL.md') -Destination (Join-Path $baselineRoot 'SKILL.md')
Copy-Item -LiteralPath (Join-Path $skillRoot 'references\prompt-generation-contract.md') -Destination (Join-Path $baselineRoot 'references\prompt-generation-contract.md')
Copy-Item -LiteralPath (Join-Path $skillRoot 'scripts\validate_secure_mcp_contract.ps1') -Destination (Join-Path $baselineRoot 'scripts\validate_secure_mcp_contract.ps1')
New-Item -ItemType File -Path (Join-Path $baselineRoot 'scripts\ensure_secure_mcp_tunnel.ps1') | Out-Null
New-Item -ItemType File -Path (Join-Path $baselineRoot 'scripts\test_ensure_secure_mcp_tunnel.ps1') | Out-Null
```

Expected: baseline directory contains three exact copies and two empty new-file baselines; no secret-bearing Profile is copied.

- [ ] **Step 2: Add failing lifecycle predicates to the validator**

Add after `$adjudicationPath`:

```powershell
$lifecyclePath = Join-Path $skillRoot 'scripts\ensure_secure_mcp_tunnel.ps1'
```

After loading the existing contracts, add:

```powershell
$lifecyclePresent = Test-Path -LiteralPath $lifecyclePath
$lifecycle = if ($lifecyclePresent) {
    Get-Content -Raw -LiteralPath $lifecyclePath
}
else {
    ''
}
Add-Result 'TunnelLifecycleHelperPresent' $lifecyclePresent

$lifecycleContract = (Has-All $skill @(
    '(?is)before.{0,160}(?:open|navigate).{0,100}Browser.{0,220}`ensure_secure_mcp_tunnel\.ps1`'
    '(?is)`already-running`.{0,180}`started`'
    '(?is)(?:authorized chat fallback|authorized fallback).{0,180}`blocked`'
)) -and (Has-All $prompt @(
    '(?is)local Tunnel lifecycle.{0,500}Browser'
    '(?is)`g-workspace-readonly`'
    '(?is)`/healthz`.{0,120}`/readyz`'
))
Add-Result 'TunnelLifecycleContract' $lifecycleContract

$lifecycleSafety = $lifecyclePresent -and (Has-All $lifecycle @(
    '(?is)g-workspace-readonly'
    '(?is)Start-Process.{0,500}WindowStyle.{0,80}Hidden'
    '(?is)/healthz'
    '(?is)/readyz'
    '(?is)StartupTimeoutSeconds.{0,80}30'
    '(?is)Stop-Process.{0,300}(?:startedProcess|started\.Id)'
))
Add-Result 'TunnelLifecycleSafety' $lifecycleSafety
```

- [ ] **Step 3: Run the static validator and verify RED**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected: nonzero exit with:

```text
FAIL TunnelLifecycleHelperPresent
FAIL TunnelLifecycleContract
FAIL TunnelLifecycleSafety
```

All pre-existing predicates must remain `PASS`.

- [ ] **Step 4: Create the failing helper behavior test**

Create `test_ensure_secure_mcp_tunnel.ps1` with:

```powershell
$ErrorActionPreference = 'Stop'

$helperPath = Join-Path $PSScriptRoot 'ensure_secure_mcp_tunnel.ps1'
if (-not (Test-Path -LiteralPath $helperPath)) {
    throw "Missing helper: $helperPath"
}

. $helperPath

function Assert-Equal {
    param(
        [Parameter(Mandatory)]
        [object]$Actual,
        [Parameter(Mandatory)]
        [object]$Expected,
        [Parameter(Mandatory)]
        [string]$Name
    )

    if ($Actual -ne $Expected) {
        throw "$Name expected '$Expected' but got '$Actual'"
    }
    Write-Output "PASS $Name"
}

$cases = @(
    @{
        Name = 'ReadyExpectedProcessIsReused'
        ExpectedProcessCount = 1
        EndpointResponding = $true
        Health = $true
        Ready = $true
        Expected = 'reuse'
    }
    @{
        Name = 'StoppedTunnelIsStarted'
        ExpectedProcessCount = 0
        EndpointResponding = $false
        Health = $false
        Ready = $false
        Expected = 'start'
    }
    @{
        Name = 'UnhealthyExpectedProcessIsBlocked'
        ExpectedProcessCount = 1
        EndpointResponding = $true
        Health = $true
        Ready = $false
        Expected = 'block-existing-unhealthy'
    }
    @{
        Name = 'ForeignPortOwnerIsBlocked'
        ExpectedProcessCount = 0
        EndpointResponding = $true
        Health = $false
        Ready = $false
        Expected = 'block-port-conflict'
    }
    @{
        Name = 'DuplicateExpectedProcessesAreBlocked'
        ExpectedProcessCount = 2
        EndpointResponding = $true
        Health = $true
        Ready = $true
        Expected = 'block-existing-unhealthy'
    }
)

foreach ($case in $cases) {
    $actual = Resolve-TunnelLifecycleAction `
        -ExpectedProcessCount $case.ExpectedProcessCount `
        -EndpointResponding $case.EndpointResponding `
        -Health $case.Health `
        -Ready $case.Ready
    Assert-Equal -Actual $actual -Expected $case.Expected -Name $case.Name
}
```

- [ ] **Step 5: Run the helper test and verify RED**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1'
```

Expected: nonzero exit with `Missing helper`.

---

### Task 2: Deterministic Tunnel Lifecycle Helper

**Files:**
- Create: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/ensure_secure_mcp_tunnel.ps1`
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/test_ensure_secure_mcp_tunnel.ps1`

**Interfaces:**
- Consumes: exact executable path, exact Profile File, Profile name, loopback health base URL
- Produces:
  - `Resolve-TunnelLifecycleAction(int,bool,bool,bool) -> string`
  - `Invoke-SecureMcpTunnelEnsure(...) -> PSCustomObject`
  - compact JSON with `status`, `reason`, `process_id`, `health`, `ready`, `elapsed_ms`

- [ ] **Step 1: Add the helper parameter contract and pure state resolver**

Start `ensure_secure_mcp_tunnel.ps1` with:

```powershell
[CmdletBinding()]
param(
    [string]$ExecutablePath = (
        Join-Path $env:LOCALAPPDATA 'OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe'
    ),
    [string]$ProfileFile = (
        Join-Path $env:APPDATA 'tunnel-client\g-workspace-readonly.yaml'
    ),
    [string]$ProfileName = 'g-workspace-readonly',
    [uri]$HealthBaseUrl = 'http://127.0.0.1:8080',
    [ValidateRange(1, 60)]
    [int]$StartupTimeoutSeconds = 30,
    [ValidateRange(100, 5000)]
    [int]$PollIntervalMilliseconds = 500
)

$ErrorActionPreference = 'Stop'

function Resolve-TunnelLifecycleAction {
    param(
        [int]$ExpectedProcessCount,
        [bool]$EndpointResponding,
        [bool]$Health,
        [bool]$Ready
    )

    if ($ExpectedProcessCount -eq 1) {
        if ($Health -and $Ready) {
            return 'reuse'
        }
        return 'block-existing-unhealthy'
    }
    if ($ExpectedProcessCount -gt 1) {
        return 'block-existing-unhealthy'
    }
    if ($EndpointResponding) {
        return 'block-port-conflict'
    }
    return 'start'
}
```

- [ ] **Step 2: Add endpoint and expected-process probes**

Implement:

```powershell
function Invoke-LocalEndpointProbe {
    param([uri]$Uri)

    try {
        $response = Invoke-WebRequest `
            -Uri $Uri `
            -Method Get `
            -TimeoutSec 1 `
            -SkipHttpErrorCheck
        return [pscustomobject]@{
            Responded = $true
            Successful = [int]$response.StatusCode -ge 200 -and
                [int]$response.StatusCode -lt 300
        }
    }
    catch {
        return [pscustomobject]@{
            Responded = $false
            Successful = $false
        }
    }
}

function Get-TunnelEndpointState {
    param([uri]$BaseUrl)

    $healthProbe = Invoke-LocalEndpointProbe -Uri ([uri]::new($BaseUrl, '/healthz'))
    $readyProbe = Invoke-LocalEndpointProbe -Uri ([uri]::new($BaseUrl, '/readyz'))
    [pscustomobject]@{
        EndpointResponding = $healthProbe.Responded -or $readyProbe.Responded
        Health = $healthProbe.Successful
        Ready = $readyProbe.Successful
    }
}

function Get-ExpectedTunnelProcesses {
    param(
        [string]$ExpectedExecutablePath,
        [string]$ExpectedProfileName
    )

    $resolvedExecutable = [IO.Path]::GetFullPath($ExpectedExecutablePath)
    $profilePattern = '(?i)(?:^|\s)--profile(?:=|\s+)"?' +
        [regex]::Escape($ExpectedProfileName) + '"?(?:\s|$)'
    @(
        Get-CimInstance Win32_Process -Filter "Name = 'tunnel-client.exe'" |
            Where-Object {
                $_.ExecutablePath -and
                [IO.Path]::GetFullPath($_.ExecutablePath) -eq $resolvedExecutable -and
                $_.CommandLine -match '(?i)(?:^|\s)run(?:\s|$)' -and
                $_.CommandLine -match $profilePattern
            }
    )
}
```

- [ ] **Step 3: Add machine-readable result construction**

Implement:

```powershell
function New-LifecycleResult {
    param(
        [string]$Status,
        [string]$Reason,
        [Nullable[int]]$ProcessId,
        [bool]$Health,
        [bool]$Ready,
        [long]$ElapsedMilliseconds
    )

    [pscustomobject][ordered]@{
        status = $Status
        reason = $Reason
        process_id = if ($null -eq $ProcessId) {
            'not-applicable'
        }
        else {
            $ProcessId.Value
        }
        health = $Health
        ready = $Ready
        elapsed_ms = $ElapsedMilliseconds
    }
}
```

- [ ] **Step 4: Implement idempotent ensure behavior**

Implement `Invoke-SecureMcpTunnelEnsure` with this exact decision order:

```powershell
function Invoke-SecureMcpTunnelEnsure {
    param(
        [string]$ExpectedExecutablePath,
        [string]$ExpectedProfileFile,
        [string]$ExpectedProfileName,
        [uri]$ExpectedHealthBaseUrl,
        [int]$TimeoutSeconds,
        [int]$PollMilliseconds
    )

    $stopwatch = [Diagnostics.Stopwatch]::StartNew()
    if (
        -not (Test-Path -LiteralPath $ExpectedExecutablePath -PathType Leaf) -or
        -not (Test-Path -LiteralPath $ExpectedProfileFile -PathType Leaf)
    ) {
        return New-LifecycleResult `
            -Status 'blocked' `
            -Reason 'missing-runtime' `
            -ProcessId $null `
            -Health $false `
            -Ready $false `
            -ElapsedMilliseconds $stopwatch.ElapsedMilliseconds
    }

    $expectedProcesses = @(Get-ExpectedTunnelProcesses `
        -ExpectedExecutablePath $ExpectedExecutablePath `
        -ExpectedProfileName $ExpectedProfileName)
    $endpointState = Get-TunnelEndpointState -BaseUrl $ExpectedHealthBaseUrl
    $action = Resolve-TunnelLifecycleAction `
        -ExpectedProcessCount $expectedProcesses.Count `
        -EndpointResponding $endpointState.EndpointResponding `
        -Health $endpointState.Health `
        -Ready $endpointState.Ready

    if ($action -eq 'reuse') {
        return New-LifecycleResult `
            -Status 'already-running' `
            -Reason 'ready' `
            -ProcessId ([int]$expectedProcesses[0].ProcessId) `
            -Health $true `
            -Ready $true `
            -ElapsedMilliseconds $stopwatch.ElapsedMilliseconds
    }
    if ($action -eq 'block-existing-unhealthy') {
        $processId = if ($expectedProcesses.Count -eq 1) {
            [int]$expectedProcesses[0].ProcessId
        }
        else {
            $null
        }
        return New-LifecycleResult `
            -Status 'blocked' `
            -Reason 'existing-unhealthy' `
            -ProcessId $processId `
            -Health $endpointState.Health `
            -Ready $endpointState.Ready `
            -ElapsedMilliseconds $stopwatch.ElapsedMilliseconds
    }
    if ($action -eq 'block-port-conflict') {
        return New-LifecycleResult `
            -Status 'blocked' `
            -Reason 'port-conflict' `
            -ProcessId $null `
            -Health $endpointState.Health `
            -Ready $endpointState.Ready `
            -ElapsedMilliseconds $stopwatch.ElapsedMilliseconds
    }

    try {
        $startedProcess = Start-Process `
            -FilePath $ExpectedExecutablePath `
            -ArgumentList @('run', '--profile', $ExpectedProfileName) `
            -WindowStyle Hidden `
            -PassThru
    }
    catch {
        return New-LifecycleResult `
            -Status 'blocked' `
            -Reason 'start-failed' `
            -ProcessId $null `
            -Health $false `
            -Ready $false `
            -ElapsedMilliseconds $stopwatch.ElapsedMilliseconds
    }

    while ($stopwatch.Elapsed.TotalSeconds -lt $TimeoutSeconds) {
        if ($startedProcess.HasExited) {
            return New-LifecycleResult `
                -Status 'blocked' `
                -Reason 'start-failed' `
                -ProcessId ([int]$startedProcess.Id) `
                -Health $false `
                -Ready $false `
                -ElapsedMilliseconds $stopwatch.ElapsedMilliseconds
        }

        $endpointState = Get-TunnelEndpointState -BaseUrl $ExpectedHealthBaseUrl
        if ($endpointState.Health -and $endpointState.Ready) {
            return New-LifecycleResult `
                -Status 'started' `
                -Reason 'ready' `
                -ProcessId ([int]$startedProcess.Id) `
                -Health $true `
                -Ready $true `
                -ElapsedMilliseconds $stopwatch.ElapsedMilliseconds
        }
        Start-Sleep -Milliseconds $PollMilliseconds
    }

    if (-not $startedProcess.HasExited) {
        Stop-Process -Id $startedProcess.Id
    }
    return New-LifecycleResult `
        -Status 'blocked' `
        -Reason 'ready-timeout' `
        -ProcessId ([int]$startedProcess.Id) `
        -Health $endpointState.Health `
        -Ready $endpointState.Ready `
        -ElapsedMilliseconds $stopwatch.ElapsedMilliseconds
}
```

- [ ] **Step 5: Add the script entry point without affecting dot-sourced tests**

Append:

```powershell
if ($MyInvocation.InvocationName -ne '.') {
    $result = Invoke-SecureMcpTunnelEnsure `
        -ExpectedExecutablePath $ExecutablePath `
        -ExpectedProfileFile $ProfileFile `
        -ExpectedProfileName $ProfileName `
        -ExpectedHealthBaseUrl $HealthBaseUrl `
        -TimeoutSeconds $StartupTimeoutSeconds `
        -PollMilliseconds $PollIntervalMilliseconds
    $result | ConvertTo-Json -Compress
    if ($result.status -eq 'blocked') {
        exit 1
    }
}
```

- [ ] **Step 6: Run helper unit tests and verify GREEN**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1'
```

Expected: five `PASS` lines and exit code `0`. This test must not start or stop
a real Process.

---

### Task 3: Skill Lifecycle Contract

**Files:**
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/SKILL.md`
- Modify: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/references/prompt-generation-contract.md`
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/validate_secure_mcp_contract.ps1`

**Interfaces:**
- Consumes: helper JSON status and existing Task Contract
- Produces: Browser-before lifecycle gate and unchanged Tunnel authorization/completeness boundary

- [ ] **Step 1: Add the helper to required capabilities**

In `SKILL.md` under `## Required capabilities`, add:

```markdown
- **REQUIRED LOCAL LIFECYCLE:** Before opening or navigating Browser, run the
  bundled `scripts/ensure_secure_mcp_tunnel.ps1` by its resolved absolute path.
  This personal runtime is pinned to `g-workspace-readonly`; do not substitute
  another profile, executable, port, service, or scheduled task.
```

- [ ] **Step 2: Add a workflow phase before route resolution**

After `### 1. Establish the Task Contract`, add and renumber later phases:

```markdown
### 2. Ensure the local Tunnel lifecycle

Before opening or navigating Browser, run the bundled
`scripts/ensure_secure_mcp_tunnel.ps1` by its resolved absolute path. Accept
only `already-running` or `started` with both `health: true` and `ready: true`.
The helper reuses one exact ready process and starts
`g-workspace-readonly` only when no expected process or endpoint is present.

Treat a helper `blocked` result as unavailable local Tunnel coverage. Never
kill, restart, or duplicate an existing unhealthy process, change the health
port, choose another profile, or expose Profile contents. Select only a Task
Contract-authorized chat delivery fallback; when no such fallback preserves
required coverage, stop as `blocked`.

Local lifecycle success proves only that the client completed a successful
control-plane poll. It does not prove Task authorization, ChatGPT app
availability, allowed root, target reachability, Tool coverage, or delivery
completeness. Verify all of those separately before relying on Tunnel
Evidence.
```

Renumber the existing phases:

```text
Resolve the route -> 3
Acquire local evidence -> 4
Run preflight -> 5
Send one complete prompt -> 6
Complete, capture, and adjudicate -> 7
Stop on evidence -> 8
```

- [ ] **Step 3: Add the prompt-contract lifecycle preflight**

In `prompt-generation-contract.md`, insert immediately before
`### Secure MCP Tunnel preflight`:

```markdown
### Local Tunnel lifecycle preflight

Before opening or navigating Browser, run the bundled
`scripts/ensure_secure_mcp_tunnel.ps1` with its fixed
`g-workspace-readonly` profile. Continue to Browser preflight only for
`already-running` or `started` with successful `/healthz` and `/readyz`.
Do not infer readiness from a process name alone.

On `missing-runtime`, `existing-unhealthy`, `port-conflict`, `start-failed`,
or `ready-timeout`, do not kill or restart a pre-existing process, start a
duplicate, change the port, select another profile, or expose Profile
contents. Treat Tunnel delivery as unavailable and select only an authorized
chat fallback; otherwise stop as `blocked`.

Local lifecycle success is not Tunnel completeness. It never replaces visible
app availability, target-covering allowed root, read-only reachability,
required Tool catalog, Task Contract authorization, or final scope coverage.
```

- [ ] **Step 4: Run the contract validator and verify GREEN**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
```

Expected:

```text
PASS TunnelLifecycleHelperPresent
PASS TunnelLifecycleContract
PASS TunnelLifecycleSafety
TOTAL_FAILURES=0
```

All prior predicates remain `PASS`.

- [ ] **Step 5: Verify the Skill metadata remains unchanged**

Run:

```powershell
Get-Content -Raw 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\agents\openai.yaml'
```

Expected:

```yaml
interface:
  display_name: "Collaborate with ChatGPT Pro"
  short_description: "Consult ChatGPT Pro with a task-specific prompt"
  default_prompt: "Use $collaborating-with-chatgpt-pro to consult ChatGPT Pro for this request and adjudicate the result."

policy:
  allow_implicit_invocation: false
```

---

### Task 4: Live Runtime and Final Verification

**Files:**
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/ensure_secure_mcp_tunnel.ps1`
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/test_ensure_secure_mcp_tunnel.ps1`
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/scripts/validate_secure_mcp_contract.ps1`
- Test: `C:/Users/y2ikg/.agents/skills/collaborating-with-chatgpt-pro/`
- Compare: `%TEMP%/collaborating-with-chatgpt-pro-tunnel-autostart-20260731/`

**Interfaces:**
- Consumes: complete candidate Personal Skill and stopped or ready local Tunnel state
- Produces: live `started`／`already-running` evidence, static validation, exact Personal Skill diff, final bounded risk report

- [ ] **Step 1: Record the pre-run expected-process count**

Run:

```powershell
$exePath = 'C:\Users\y2ikg\AppData\Local\OpenAI\secure-mcp-tunnel\bin\tunnel-client.exe'
$before = @(
    Get-CimInstance Win32_Process -Filter "Name = 'tunnel-client.exe'" |
        Where-Object {
            $_.ExecutablePath -eq $exePath -and
            $_.CommandLine -match '(?i)(?:^|\s)run(?:\s|$)' -and
            $_.CommandLine -match '(?i)--profile(?:=|\s+)"?g-workspace-readonly"?(?:\s|$)'
        }
)
"EXPECTED_PROCESS_COUNT_BEFORE=$($before.Count)"
```

Expected for the currently observed state: `EXPECTED_PROCESS_COUNT_BEFORE=0`.
If it is already `1`, continue with the reuse path and report that live
stopped-to-started coverage was not reproduced.

- [ ] **Step 2: Run the helper and validate the first JSON result**

Run:

```powershell
$helper = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\ensure_secure_mcp_tunnel.ps1'
$firstRaw = & $helper
$first = $firstRaw | ConvertFrom-Json
$firstRaw
if ($first.status -notin @('started', 'already-running')) {
    throw "Lifecycle helper failed: $($first.reason)"
}
if (-not $first.health -or -not $first.ready) {
    throw 'Lifecycle helper returned success without health and ready'
}
```

Expected: exit `0`, `status` is `started` when the initial count was `0`, and
the result contains only the six approved fields.

- [ ] **Step 3: Verify live health and readiness**

Run:

```powershell
$health = Invoke-WebRequest -Uri 'http://127.0.0.1:8080/healthz' -TimeoutSec 2
$ready = Invoke-WebRequest -Uri 'http://127.0.0.1:8080/readyz' -TimeoutSec 2
"HEALTH_STATUS=$([int]$health.StatusCode)"
"READY_STATUS=$([int]$ready.StatusCode)"
```

Expected:

```text
HEALTH_STATUS=200
READY_STATUS=200
```

- [ ] **Step 4: Run the helper again and verify idempotence**

Run:

```powershell
$secondRaw = & $helper
$second = $secondRaw | ConvertFrom-Json
$secondRaw
if ($second.status -ne 'already-running') {
    throw "Expected already-running, got $($second.status)"
}
if ($second.process_id -ne $first.process_id) {
    throw 'Second invocation did not reuse the same process'
}
$after = @(
    Get-CimInstance Win32_Process -Filter "Name = 'tunnel-client.exe'" |
        Where-Object {
            $_.ExecutablePath -eq $exePath -and
            $_.CommandLine -match '(?i)(?:^|\s)run(?:\s|$)' -and
            $_.CommandLine -match '(?i)--profile(?:=|\s+)"?g-workspace-readonly"?(?:\s|$)'
        }
)
"EXPECTED_PROCESS_COUNT_AFTER=$($after.Count)"
```

Expected: same PID and `EXPECTED_PROCESS_COUNT_AFTER=1`.

- [ ] **Step 5: Validate the helper output schema and secret boundary**

Run:

```powershell
$approvedFields = @('status', 'reason', 'process_id', 'health', 'ready', 'elapsed_ms')
$actualFields = @($second.PSObject.Properties.Name | Sort-Object)
$unexpectedFields = @($actualFields | Where-Object { $_ -notin $approvedFields })
$missingFields = @($approvedFields | Where-Object { $_ -notin $actualFields })
if ($unexpectedFields.Count -ne 0) {
    throw "Unexpected result fields: $($unexpectedFields -join ', ')"
}
if ($missingFields.Count -ne 0) {
    throw "Missing result fields: $($missingFields -join ', ')"
}
if ($secondRaw -match '(?i)api[_-]?key|tunnel[_-]?id|organization[_-]?id|workspace[_-]?id|sk-[A-Za-z0-9_-]+') {
    throw 'Secret or identity material found in helper output'
}
'PASS LifecycleOutputBoundary'
```

Expected: `PASS LifecycleOutputBoundary`.

- [ ] **Step 6: Run all Skill validation**

Run:

```powershell
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\test_ensure_secure_mcp_tunnel.ps1'
& 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro\scripts\validate_secure_mcp_contract.ps1'
python 'C:\Users\y2ikg\.codex\skills\.system\skill-creator\scripts\quick_validate.py' `
    'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
```

Expected: helper tests pass, `TOTAL_FAILURES=0`, and `quick_validate.py` exits
`0` with a valid Skill result.

- [ ] **Step 7: Inspect every Personal Skill diff**

Run:

```powershell
$baselineRoot = Join-Path $env:TEMP 'collaborating-with-chatgpt-pro-tunnel-autostart-20260731'
$skillRoot = 'C:\Users\y2ikg\.agents\skills\collaborating-with-chatgpt-pro'
git diff --no-index -- (Join-Path $baselineRoot 'SKILL.md') (Join-Path $skillRoot 'SKILL.md')
git diff --no-index -- (Join-Path $baselineRoot 'references\prompt-generation-contract.md') (Join-Path $skillRoot 'references\prompt-generation-contract.md')
git diff --no-index -- (Join-Path $baselineRoot 'scripts\validate_secure_mcp_contract.ps1') (Join-Path $skillRoot 'scripts\validate_secure_mcp_contract.ps1')
git diff --no-index -- (Join-Path $baselineRoot 'scripts\ensure_secure_mcp_tunnel.ps1') (Join-Path $skillRoot 'scripts\ensure_secure_mcp_tunnel.ps1')
git diff --no-index -- (Join-Path $baselineRoot 'scripts\test_ensure_secure_mcp_tunnel.ps1') (Join-Path $skillRoot 'scripts\test_ensure_secure_mcp_tunnel.ps1')
```

Expected: only the approved lifecycle helper, test, validator predicates,
Skill workflow phase, prompt preflight, and phase renumbering are present.
`git diff --no-index` returns `1` when a difference exists; inspect output
rather than treating that code as failure.

- [ ] **Step 8: Run Repository hygiene checks**

Run:

```powershell
git diff --check
git status --short
git diff --stat
git log -2 --oneline
```

Expected: no uncommitted Repository changes. The latest committed documents are
the implementation plan and design; no Architecture Owner or `AGENTS.md`
change exists.

- [ ] **Step 9: Report completion**

Report:

```text
Personal Skill files changed
RED and GREEN results
helper unit-test count
first and second live lifecycle status
health and ready HTTP status
process count and PID reuse result
static validator and quick_validate result
Repository checks
Tunnel left running state
remaining risk: Browser app/scope/Tool completeness remains a separate runtime preflight
```
