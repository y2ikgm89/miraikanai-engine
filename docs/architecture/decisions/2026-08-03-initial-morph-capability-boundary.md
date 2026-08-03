# Initial Morph Capability Boundary

- 文書ID: mirakan.decision.initial-morph-capability-boundary
- 状態: review
- 正本範囲: Morph Targetをinitial V1／C1／C2から除外し、将来のend-to-end Capabilityとして扱う判断理由
- 非正本範囲: Morph Source Schema、track、runtime evaluation、Rendering方式、LOD、Save／Replay、Tool、Fixture、Receipt、実装Task、実装順序。各Owner文書を参照する
- 依存: [Architecture Governance](../01-governance/architecture-governance.md)、[Asset Lifecycle](../03-authoring/asset-lifecycle.md)、[Animation](../05-simulation/animation.md)、[LOD](../06-rendering/lod.md)、[Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)、[Persistence／Save](../04-runtime/persistence-save.md)
- 外部根拠検証日: 2026-08-03
- 文書種別: Architecture Decision／initial capability scope
- Decision owner document: `mirakan.arch.asset-lifecycle`
- Decision日: 2026-08-03
- Supersedes: none

## Context

既存文書には`morph_policy`、Morph compatibility、`morph_target` vocabularyが部分的に存在したが、Source delta identity、default／animated weight、track interpolation、skinとの評価順、bounds、Runtime／Rendering、LOD／Virtual Geometry、fallback、Save／Replay、Target Qualificationが一つの契約へ閉じていなかった。部分Fieldだけを残すと、実際にはImportまたはRuntime評価できないのにMorph対応を推測できる。

glTF 2.0でもMorph Targetはbase primitive attributeに対するweighted delta、default weights、animation target `weights`、accessor count／attribute整合を横断する。外部仕様がMiraikanaiのscopeを決めるわけではないが、Import Field一つではend-to-end意味を表せないことを確認できる。

## Decision drivers

- partial support claimとsilent data lossを防ぐ。
- initial V1のSource／Artifact／Runtime／fallback意味をclosedに保つ。
- Morphのためのempty Schema、placeholder API、zero-weight fallbackを先行作成しない。
- 将来採択時にAsset、Animation、Rendering、LOD、Persistence、Target Qualificationをatomicに閉じる。

## Considered options

### A. Importだけ対応してRuntimeは非対応にする

採用しない。Source保存と実行可能性の区別が曖昧になり、対応表示、Cook、Package、fallbackでsilent lossを招く。

### B. zero weightまたはbase meshへfallbackする

採用しない。animation／expression意味を保持せず、成功として扱えない。

### C. initial V1でend-to-end Morphを定義する

採用しない。必要なOwner契約が広く、C1／C2 minimum surfaceの成立に必須ではない。未設計部分を推測で固定しない。

### D. initial V1／C1／C2から明示除外する

採用する。Source検出時にfail closedとし、Future Capabilityはend-to-end closureが揃った新versionだけで提案する。

## Decision

1. initial `MeshImportSettingsV1`からMorph policyを除く。
2. Morph delta、default weightまたはanimated weight channelを検出したSourceはunsupportedとしてImport／Cook／Packageを拒否する。
3. Animation、LOD、Virtual Geometryのinitial contractはMorph Field、empty placeholderまたはzero-weight fallbackを持たない。
4. `morph_target`等のFuture vocabularyが残る文書ではplanning-only／選択不能を明記する。
5. 将来採択はSource identityからTarget Qualificationまでを同じversioned Capability closureへ閉じる。

## Consequences

- initial V1のMorph support claimは明確にfalseとなり、部分Fieldからの過大主張を防げる。
- Morphを含むAssetは警告継続でなくfail closedになる。
- 将来Morphを採択する場合は複数Ownerの新versionとCompatibility判断が必要になる。
- RepositoryにImporter、Schema、Runtime、FixtureまたはReceiptは存在せず、本Decisionは実装を主張しない。

## Canonical Owner documents

- Source検出／Import／Artifact boundary: [Asset Lifecycle](../03-authoring/asset-lifecycle.md)
- track／evaluation／Animation binding: [Animation](../05-simulation/animation.md)
- representation selection: [LOD](../06-rendering/lod.md)
- Future geometry compatibility: [Virtualized／Continuous Geometry](../06-rendering/virtualized-continuous-geometry.md)
- durable Save／Replay inclusion boundary: [Persistence／Save](../04-runtime/persistence-save.md)

## Supersedes／Superseded by

- Supersedes: none
- Superseded by: none

## Official or primary sources

- [Khronos glTF 2.0 specification](https://registry.khronos.org/glTF/specs/2.0/glTF-2.0.html)
