# Miraikanai Engine 実現可能性 Remediation 設計

> **状態: superseded historical remediation design。** Current authorityはProduct Active 14／WP75、Future 3／23／Claim52、current Control Plane plans、Critical Path、Future Inception、current remediation Reviewである。本文の17件等は当時のbaseline記録であり実装へ使わない。

- 状態: ユーザー承認済み
- 承認日: 2026-07-23
- 承認根拠: 2026-07-23監査結果に対する「公式推奨で計画書を推奨で更新して」
- 対象baseline: Git commit `a87f35fcec9df0e76cb6240234853575a1276337`
- 変更対象: Architecture正本と実装計画。Engine runtime実装、Capability activation、Release承認は対象外

## 1. Outcome

AI-native Game EngineというProduct intentを維持したまま、現在の計画を実装者とValidatorが一意に解釈できる状態へ修正する。完了時には次を満たす。

1. Product、Control Plane、実装計画が同じWork Package schemaとstate語彙を使用する。
2. Phase exitはFixture全体ではなく、Phase固有のRequirement／Target／Candidate／freshness bindingとして評価できる。
3. Phase 0から2D First Playable、AI MVP、3D First Playableへ到達するWork Package DAGが存在する。
4. AI、Editor、CLI、MCPがCookだけでなくPackage、Install、Launch、Smoke、Diagnosis、Support、Resetまで同じGateway Operationを使用する。
5. 初心者MVPとAI生成Project C++／Shaderの専門家承認を分離する。
6. ChatGPT Chat／Work remote app、ChatGPT desktop app内Codex host、Codex CLI／IDE、Claude、Cursor、cloud／local modelを、製品名の条件分岐ではなくsurface別Host、Transport、Provider、Deployment、Model、Authorityの直交Profileとして扱う。
7. D3D12 MVPはCX0 Headerで開始し、CX2／CX3のModules移行とShippingを外部Toolchain成立条件へ分離する。
8. Open World、Online、MMO、大人数network shooter、FPS、advanced physics／animation、AAA rendering、Terrain／Foliage、Console、Web、XR、全自動Asset生成、first-party local inference、runtime generationを、実装済みと誤認させない将来Capability incubationとして追跡する。

## 2. Authority

判断根拠は次の順序で適用する。

1. ユーザーが承認したProduct intentと本設計。
2. active architecture Owner文書。
3. Product Planの機械Registry。
4. 外部API、SDK、Toolchain、Platformの公式一次資料。
5. 2026-07-23監査所見。

外部公式資料は外部仕様だけを所有する。Miraikanai固有のPhase、ID、Risk、Approval、TTL、Capability maturityを「公式推奨」と表現しない。公式資料がProject判断を規定しない場合は、fail-closed、最小権限、再現可能性、移行量をProject decisionの根拠とする。

## 3. 検討した方式

### 3.1 採用: 既存正本をclean migrationする

既存logical IDとOwner境界を可能な限り維持し、schema、Phase gate、Work Package DAG、Operation closureを同一ChangeSetで修正する。既存文書を別の「v2計画書」で上書きせず、active正本を一つに保つ。

利点は、既存の43正本、Product registry、Control Plane計画、ECS Decision、D3D12計画を再利用できることである。欠点は複数文書の同時修正が必要なことだが、機械参照を二重化しないため長期的な誤差が最小になる。

### 3.2 不採用: Product Plan v2を別文書として追加する

旧正本と新正本のAuthorityが競合し、AIとValidatorがどちらを読むべきか一意にならないため採用しない。

### 3.3 不採用: Runtime実装を先行して後から文書を合わせる

ECS E1以降、D3D12、Project Native、Package／Device操作の開始Gateが未定義のままになるため採用しない。

## 4. Canonical Product schema

### 4.1 Work Package

Product PlanをWork Package正本とし、全consumerは次のFieldへ統一する。

```text
WorkPackageRegistryV1
  entries[]:
    work_package_id
    phase_id
    owner_document_id
    target_refs[]
    fallback_ref
    provided_fixture_refs[]
    required_capability_refs[]
    requires_work_package_refs[]
    scheduling_state
    defer_reason
    reconsideration_gate_refs[]
    blocked_reason_ref
```

- `scheduling_state`のclosed値は`declared | ready | active | blocked | deferred | complete`とする。
- RequirementとPhase completionはWork Packageへ複写しない。
- `provided_fixture_refs[]`は実装寄与を表し、Work Package単独の完了Gateではない。
- 完了Receiptはappend-onlyな`WorkPackageLifecycleRecordV1`が所有する。

### 4.2 Phase Fixture binding

同じFixtureを複数Phaseで安全に再利用するため、次をProduct Planへ追加する。

```text
PhaseFixtureBindingRegistryV1
  entries[]:
    gate_id
    phase_id
    fixture_id
    evaluated_requirement_refs[]
    target_refs[]
    candidate_binding_policy_ref
    freshness_policy_ref
```

`ProductPhaseRegistryV1.exit_fixture_refs[]`は`exit_gate_refs[]`へ置換する。GateはFixtureの`requirement_refs[]`と`target_refs[]`のsubsetだけを参照でき、範囲外参照を拒否する。

`candidate_binding_policy_ref=policy.product.same-candidate.v1`はProject revision、Candidate root hash、Contract set hash、Toolchain lock、Target Profileを全Receiptで一致させる。`freshness_policy_ref`は§8の決定論的Freshness policyを参照する。

## 5. 実装DAG

既存Phase identityは維持し、orderとWork Package集合だけを修正する。

### Phase 0: Foundation

- Control Plane bootstrapとOwner approval
- C++23 CX0 build baseline
- Math／Memory baseline
- Scheduling core
- ECS E0 Contract
- ECS E1 Storage
- ECS E2 Query／Mutation

### Phase 1: Headless Authoring

- ECS E3 Cook／Load
- Headless Project State／Asset／Save baseline
- deterministic headless Authoring round-trip

### Phase 2: Editor Runtime Windows

- ECS E4 Game System binding
- Render Graph core
- D3D12 CX0 backend
- Windows Input／Audio／UI core
- Editor Runtime、Package、clean install／offline launch

### Phase 3: Manual 2D

- Gameplay core、Timer、World／Camera 2D
- Collision／Physics／Animation 2D
- Navigation path following
- ECS E5 2D subsystem integration
- Shooter core／2D fixture

### Phase 4: AI Authoring MVP-A

- AI authoring、Debug／Replay／Support operation
- ECS E6 Debug／AI
- 初心者はDefinition-firstと事前Qualification済みNative／Shader Packを使用
- 新規AI生成Native／Shaderは別のCode owner Gateを通す
- ECS E7 Windows 2D qualification

### Phase 5以降

- Phase 5 External Agentはlocal STDIO MCPとStreamable HTTPのHost conformanceを扱う。
- first-party local inferenceはMVP blockerにせずFuture incubationへ置く。外部MCP Hostがlocal modelを使う経路はPhase 5で許可する。
- Phase 6 3DはCollision／Physics／Animation／World／Cameraの3D Work PackageとECS E5／E7 3D qualificationを追加する。
- Phase 7 MobileはVulkan、Metal、Android package、Apple package、mobile lifecycleを別Work Packageにする。
- Phase 8 aggregate Work PackageはGenre／Rendering／UI providerの後に実行する。

## 6. AI Operation closure

次の14 IDは`planning.operation_family.build_device_play_debug_task`のreserved candidateとして記録する。current MCD、Owner Manifest、Trusted Service allowlist、Policy、Validator／closure、Diagnostic／Receipt、Provider／MCP Tool、alias集合はすべて空、Capability stateは`not_activated`である。`activation.build_gateway.operation_pipeline.v1`が14件の完全closureを同じContract set transactionで成立させた場合だけfamily全体をatomic activateし、部分登録、read-only先行公開、name-only aliasを許可しない。

```text
operation.build.request_package
operation.device.install
operation.device.launch
operation.device.reset_data
operation.play.run_smoke
operation.debug.aggregate
operation.debug.query
operation.debug.read_causality
operation.debug.read_replay_slice
operation.debug.validate_finding
operation.debug.support-bundle.generate
operation.task.status
operation.task.read_receipt
operation.task.cancel
```

Activation後も`device.install`と`device.reset_data`は対象Device identity／generation、Package Receipt、明示consent、Risk R3 Approvalを必須とする。`launch`、`smoke`、Debug queryはこれらの権限を継承せず、各Operationのallowlistを個別評価する。currentでは14候補のdispatchを`MIRAKAN-POLICY-CAPABILITY_NOT_ACTIVATED`で拒否し、Project／Task／Device状態を不変にする。

## 7. Host、Provider、Model、local inference

次の型をGovernance／Executable Contractへ追加する。

```text
AiCallerContextV1
ExternalClientSecurityProfileV1
McpSessionGrantV1
ProviderManifestV1
InferenceDeploymentProfileV1
ModelSnapshotProfileV1
```

互換判定単位は`{host profile, transport profile, provider/runtime profile, model snapshot, tool projection, authority profile}`である。

- Host: Editor、ChatGPT Chat／Work remote app、ChatGPT desktop app内Codex host、Codex CLI、Codex IDE、Claude Desktop、Claude Code、Cursor。
- Transport: local STDIO MCP、Streamable HTTP、direct Provider API。
- Deployment: cloud endpointまたはlocal process／IPC。
- Model: exact provider model ID、またはweights／quantization hash。
- Authority: query、proposal、managed source edit、build job。Approval、Commit、Activation、Signing、Releaseは常に不可。

Gemma、Kimi、Qwen、DeepSeek、Grok、GLM等をEngine分岐にしない。conformance Receiptがない組合せは`proposal_only`または`not_activated`とする。local runtimeはmodel license／provenance、RAM／VRAM、context、tool schema conformance、process／network sandbox、logging／retention、resource exhaustion、fallbackをProfileへ固定し、silent cloud fallbackを禁止する。

ChatGPT Chat／Workのcustom MCP appは現行公式範囲ではweb-onlyかつremote MCPで、private／local serverはSecure MCP Tunnel経由にする。ChatGPT desktop app内Codex hostはCodex設定を共有する別surfaceとしてlocal STDIO／Streamable HTTPを許容する。両者のHost Profile、Receipt、設定、権限を流用しない。

## 8. Approval、状態、Freshness

### 8.1 Code owner

```text
CodeOwnerAssignmentV1
  assignment_id, subject_identity_ref, path_or_module_scope_refs[],
  qualification_receipt_ref, independence_policy_ref,
  valid_from, expires_at, revoked_at

CodeOwnerApprovalV1
  assignment_ref, exact_diff_hash, source_revision,
  build_receipt_refs[], review_receipt_ref, decision, issued_at
```

初心者のGameplay intent承認はCode owner承認を代替しない。Code owner不在時はDefinition／事前Qualification済みPackへdowngradeし、新規Native／Shader生成を停止する。

### 8.2 Target readiness

`target_readiness.state`は`predicted | blocked | qualified`へ統一し、`optimization_required`は`blocked_reason_ref`とする。`not_activated`はCapability activationだけに使用する。

### 8.3 Evidence freshness

```text
TechnicalQualificationReceiptV1
  issued_at, expires_at, freshness_policy_ref,
  revocation_snapshot_ref, subject_hash, evidence_hashes[]
```

Freshnessは現在時刻だけでなく、Source、Candidate、Toolchain、Target、Device、Policy、Receipt revocationの一致から導出する。Project既定は次とする。

| policy | max age | 追加invalidator |
|---|---:|---|
| `policy.evidence.contract-ci.v1` | 604800秒 | Contract／Toolchain／generator変更 |
| `policy.evidence.target-device.v1` | 259200秒 | OS／driver／device generation／package変更 |
| `policy.evidence.release.v1` | 86400秒 | Candidate／signing policy／store declaration変更 |

このTTLはMiraikanaiのfail-closedな初期運用値であり、外部Vendor推奨ではない。変更にはR4 Product Decisionと既存Receiptの失効が必要である。

## 9. Toolchainと公式根拠

- MCPは現行安定版`2025-11-25`を固定する。draft／release candidateをProduction baselineへ自動採用しない。
- OpenAI direct Providerの既定explicit modelは`gpt-5.6-sol`を維持する。model familyの役割を一つへ潰さず、ModelSnapshot ProfileとEvalで変更する。
- Control PlaneのDraft 2020-12 validatorはAjv `8.20.0`、MIT、npm integrity `sha512-Thbli+OlOj+iMPYFBVBfJ3OmCAnaSyNn4M1vz9T6Gka5Jt9ba/HIR56joy65tY6kx/FCF5VXNB819Y7/GUrBGA==`を採用し、`ajv/dist/2020`、`strict=true`、`allErrors=false`、local `$ref` allowlistだけを使用する。
- CMake 4.4の`import std`はexperimental opt-inでNinja／Ninja Multi-Config限定であるため、MVP D3D12はCX0 self-contained Headerとする。
- CX3 Shippingは正式`/std:c++23`、experimental token不要のCMake、全Target Receiptが揃うまで`not_activated`とする。

公式Evidence:

- OpenAI MCP: <https://learn.chatgpt.com/docs/extend/mcp>
- OpenAI GPT-5.6 migration: <https://developers.openai.com/api/docs/guides/upgrading-to-gpt-5p6-sol>
- MCP stable specification: <https://modelcontextprotocol.io/specification/2025-11-25>
- MCP Authorization: <https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization>
- Ajv Draft 2020-12: <https://ajv.js.org/json-schema.html#draft-2020-12>
- CMake C++ Modules: <https://cmake.org/cmake/help/v4.4/manual/cmake-cxxmodules.7.html>
- MSVC 14.51 C++23 status: <https://devblogs.microsoft.com/cppblog/c23-support-in-msvc-build-tools-14-51/>

## 10. Future Capability incubation

`FutureCapabilityIncubationRegistryV1`は次のFieldを持つ。

```text
future_capability_id, owner_document_id, planning_state,
prerequisite_capability_refs[], required_decision_kinds[],
candidate_target_kinds[], qualification_fixture_kinds[],
activation_trigger, excluded_current_product_claims[]
```

初期entryは次の17件とする。

1. offline large-world／continuous streaming
2. headless dedicated server／session transport／replication
3. small cooperative multiplayer
4. rollback／competitive networking
5. large-session network shooter
6. persistence／live service／moderation／operations
7. MMO／distributed world authority
8. first-person shooter Profile
9. vehicle／ragdoll／crowd／motion warping
10. production Terrain／Foliage／GI
11. AAA photoreal rendering
12. Console target program
13. Web target program
14. XR target program
15. commercial asset generation／license／quality qualification
16. first-party local inference
17. runtime structured-data generation

全entryの初期stateは`planning_only`であり、active Product Capability、Phase exit、Shipping labelへ含めない。

## 11. Completion Gate

本更新は次をすべて満たした時だけ完了とする。

1. Work Package schema名の競合が0件。
2. 全Phase exit gateがPhase固有Requirement／Targetへ束縛される。
3. Work Package dependency cycle、後続Phase dependency、missing owner、missing targetが0件。
4. Phase 0からPhase 4までの必須Capabilityにowner Work Packageが存在する。
5. AI E2Eに必要なOperationが全件Registryへ登録される。
6. D3D12計画にCX0で作成する`.ixx`が0件。
7. 全local Markdown link、anchor、document ID、Product logical IDが解決する。
8. active文書を`approved`へ自動変更せず、Capabilityをactivateしない。
