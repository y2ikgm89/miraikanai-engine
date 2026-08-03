# MCP Current Protocol Baseline

- 文書ID: mirakan.decision.mcp-current-protocol-baseline
- 状態: review
- 正本範囲: initial V1のMCP supported-version setを公式current `2026-07-28`のsingletonにする判断理由
- 非正本範囲: MCP Schema／Server／SDK／Transport実装、Operation catalog、Authorization payload、Host／Provider attestation、Conformance Fixture／Receipt、実装Task、実装順序。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)、[Executable Contracts](../02-foundation/executable-contracts.md)、[AI Security／Approval](../01-governance/ai-security-approval.md)
- 外部根拠検証日: 2026-08-03
- 文書種別: Architecture Decision／MCP protocol baseline
- Decision owner document: `mirakan.arch.toolchain-dependencies`
- Decision日: 2026-08-03
- Supersedes: none

## Context

ToolchainはMCP `2025-11-25`を選択baselineとしていた一方、公式currentは`2026-07-28`であり、per-request protocol version、unsupported-version error、`server/discover`を含むlifecycleが異なる。pre-publicのinitial V1で両Revisionを同時に扱うと、legacy initializeとcurrent discovery／request negotiation、Transport carrier、Conformanceを二重に設計する必要がある。

## Decision drivers

- current公式仕様へ直接合わせ、公開前のlegacy branchを持たない。
- supported-version setとtransport別version carrierを一意にする。
- SDK fallbackをMiraikanaiのsupport／Conformanceと誤認しない。
- missing／unsupported／payload-header mismatchを副作用前にfail closedにする。

## Considered options

### A. `2025-11-25`だけを維持する

採用しない。initial V1公開前から旧lifecycleを固定し、current仕様への移行負債を作る。

### B. `2025-11-25`と`2026-07-28`を同時対応する

採用しない。initialize／discovery、version negotiation、Transport、negative Conformanceの二branchを必要とし、現在存在しないconsumerのための互換層になる。

### C. versionを自動選択またはSDKへ委譲する

採用しない。Engine-owned supported set、failure、Audit identityが不明確になる。

### D. `2026-07-28`だけをinitial baselineにする

採用する。supported setをsingletonにし、旧Revisionは未対応として拒否する。

## Decision

1. initial V1のsupported-version setはexact `[2026-07-28]`である。
2. request `_meta["io.modelcontextprotocol/protocolVersion"]`を必須にし、Streamable HTTPでは同じ値の`MCP-Protocol-Version` headerとのbyte equalityを要求する。
3. `server/discover`は同じsingletonと実際のServer identity／capabilityだけを公開する。
4. missing、unsupported、carrier mismatchまたはno-mutual-versionを副作用前に拒否する。
5. `2025-11-25` initialize、alias、fallback、dual lifecycleをinitial V1へ持たない。

## Consequences

- MCP projection、Toolchain、AI Security Profileが一つのprotocol identityへ揃う。
- 旧Clientを初期互換対象にしないclean breakになる。
- 将来別Revisionを追加する場合はCompatibility Consumer InventoryとRevision別Conformanceが必要になる。
- RepositoryにMCP Server、Schema、SDK、FixtureまたはReceiptは存在せず、本Decisionは相互運用またはActivationを主張しない。

## Canonical Owner documents

- protocol pin／dependency／materialization gate: [Toolchain／Dependencies](../02-foundation/toolchain-dependencies.md)
- MCD→MCP projectionとversion failure: [Executable Contracts](../02-foundation/executable-contracts.md)
- transport／credential／authorization／attestation: [AI Security／Approval](../01-governance/ai-security-approval.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

- [MCP 2026-07-28 specification](https://modelcontextprotocol.io/specification/2026-07-28)
- [MCP protocol versioning](https://modelcontextprotocol.io/docs/2026-07-28/learn/versioning)
