# ChatGPT 5.6 Pro 必須機能一覧・原回答記録

- 取得日: 2026-07-29
- 取得元: https://chatgpt.com/c/6a68d18e-9e24-83ee-9c9b-d460d9cf0bee
- 位置づけ: 外部モデル回答の非規範な原文記録。Miraikanai EngineのOwner文書、Product scope、設計判断、実装状態またはQualificationを置換しない。
- 保存状態: ブラウザ表示の回答本文を省略せず保持。

---

4m 16s考えました
C++ゲームエンジンに必要な機能一覧

前提として、2D・3D対応、独自エディタ付き、第三者に提供できる製品レベルのゲームエンジンを想定しています。
ECS、マルチスレッド、複数グラフィックスAPI対応などは実装方式であり、すべてのエンジンで必須とは限りません。

1. エンジンコア

エンジン起動・終了処理

サブシステム初期化・終了順序管理

メインループ

ゲーム更新ループ

描画ループ

固定タイムステップ

可変タイムステップ

フレームレート制御

フレームペーシング

Delta Time管理

Time Scale

一時停止

タイマー

スケジューラー

エンジン状態管理

イベントシステム

メッセージバス

コールバック管理

サービスロケーターまたは依存性注入

サブシステム間依存関係管理

設定ファイル管理

Console Variable

コマンドコンソール

UUID・GUID生成

オブジェクトハンドル

オブジェクト寿命管理

バージョン管理

データ移行処理

2. プラットフォーム抽象化

Windows対応

Linux対応

macOS対応

Android対応

iOS対応

コンソール対応用抽象化

ウィンドウ作成

ウィンドウサイズ変更

フルスクリーン

ボーダーレス

マルチモニター

DPIスケーリング

OSイベント処理

ファイルダイアログ

クリップボード

マウスカーソル制御

OS時刻取得

スリープ処理

スレッドAPI抽象化

動的ライブラリ読み込み

環境変数

プロセス起動

プラットフォーム固有パス管理

3. ファイルシステム

通常ファイルI/O

非同期ファイルI/O

Virtual File System

ディレクトリ管理

パス正規化

相対パス・絶対パス変換

ファイル監視

ファイル変更検知

パッケージファイル

Pak・Archive読み込み

圧縮

暗号化

メモリマップドファイル

ストリーミング読み込み

ユーザーデータ保存先管理

キャッシュディレクトリ管理

4. メモリ管理

RAIIベースのリソース管理

メモリアロケーター

Linear Allocator

Pool Allocator

Stack Allocator

Frame Allocator

GPUメモリアロケーター

オブジェクトプール

メモリ使用量追跡

メモリリーク検出

ダブルフリー検出

メモリアラインメント

キャッシュ効率を考慮した配置

メモリ予算管理

Out of Memory対策

DLL境界をまたぐメモリ所有権管理

5. マルチスレッド・ジョブシステム

スレッドプール

Job System

Task Graph

ジョブ依存関係

Work Stealing

Future・Promise

Fence

Atomic操作

Mutex

Read/Write Lock

Lock-free Queue

メインスレッドタスク

レンダースレッド

I/Oスレッド

アセット読み込みスレッド

物理演算スレッド

スレッドセーフなリソース管理

デッドロック検出

スレッドプロファイリング

6. 数学ライブラリ

Vector2

Vector3

Vector4

Matrix

Quaternion

Transform

Plane

Ray

Line

Triangle

Rectangle

Circle

Sphere

AABB

OBB

Frustum

色表現

補間

Lerp

Slerp

曲線

Bezier

Spline

ノイズ

乱数

座標系変換

ワールド・ローカル変換

交差判定

SIMD最適化

浮動小数点誤差対策

7. オブジェクト・ワールド管理

EntityまたはGameObject

Componentシステム

ECSまたはオブジェクト指向コンポーネントモデル

Component追加・削除

Component検索

Componentライフサイクル

Transform階層

親子関係

Scene

World

Level

Scene読み込み・保存

Scene切り替え

Additive Scene

Prefab

Prefab Variant

Prefab Override

タグ

レイヤー

オブジェクト検索

空間インデックス

Scene Partition

World Streaming

Level of Detail管理

Large World Coordinates

Origin Rebasing

オブジェクト有効・無効切り替え

実行時生成・破棄

8. リフレクション・シリアライズ

C++ではエディタ連携や保存機能に必要となる重要領域です。

型情報登録

プロパティ情報登録

関数情報登録

Enum情報登録

C++コード生成

メタデータ

エディタ公開プロパティ

読み取り専用属性

範囲・カテゴリ属性

JSONシリアライズ

Binaryシリアライズ

YAMLなどのテキスト形式

ネットワークシリアライズ

差分シリアライズ

バージョン付きシリアライズ

後方互換データ変換

不明フィールドの処理

オブジェクト参照復元

循環参照対応

Clone・Duplicate

Undo用スナップショット

9. レンダリング基盤

Graphics Device管理

DirectX 12、Vulkan、Metalなどのバックエンド

Render Hardware Interface

Swap Chain

Command Queue

Command Buffer

Render Pass

Render Target

Framebuffer

Descriptor管理

Pipeline State管理

GPU同期

Fence

Semaphore

GPUリソース寿命管理

Buffer管理

Vertex Buffer

Index Buffer

Uniform・Constant Buffer

Structured Buffer

Texture管理

Sampler管理

Depth・Stencil

Blend State

Rasterizer State

Render Graph・Frame Graph

複数フレーム同時処理

GPUメモリ管理

Device Lost対策

解像度変更対応

10. シェーダーシステム

Vertex Shader

Pixel・Fragment Shader

Compute Shader

Geometry Shader

Mesh Shader対応用設計

シェーダーコンパイル

オフラインコンパイル

実行時コンパイル

Shader Reflection

Shader Variant

Shader Permutation

シェーダーキャッシュ

シェーダーホットリロード

エラー表示

プラットフォーム別変換

Include管理

マクロ・Define管理

Pipeline Stateキャッシュ

11. マテリアルシステム

Material

Material Instance

パラメーター管理

テクスチャパラメーター

数値・色・ベクトルパラメーター

シェーダー選択

Rendering Queue

Transparent・Opaque管理

Two-sided

Alpha Test

Blend Mode

マテリアル継承

マテリアルホットリロード

マテリアルエディタ

Shader GraphまたはNode Graph

12. 共通描画機能

Camera

Orthographic Camera

Perspective Camera

Viewport

Multiple Camera

Offscreen Rendering

Culling

Frustum Culling

Occlusion Culling

Batching

Dynamic Batching

GPU Instancing

Indirect Drawing

LOD

Render Layer

Sorting

Picking

Object ID Buffer

デバッグ描画

線・グリッド・境界描画

スクリーンショット

HDR

Gamma Correction

Color Space管理

Tone Mapping

MSAA

Temporal AAなどの拡張用設計

13. 2Dレンダリング

Sprite Renderer

Sprite Sheet

Texture Atlas

Sprite Animation

描画順序

Sorting Layer

Pixel Perfect Camera

2D Camera

Tilemap

Tile Palette

Animated Tile

Isometric Tilemap

Nine-slice

2D Lighting

2D Shadow

2D Particle

2D Skeletal Animation

パララックス背景

ビットマップフォント

Sprite Batching

ピクセルスナップ

HD-2D用2D・3D混在描画

14. 3Dレンダリング

Static Mesh

Skinned Mesh

Submesh

Mesh Import

Normal・Tangent生成

PBRマテリアル

Directional Light

Point Light

Spot Light

Area Light用拡張

Shadow Mapping

Cascaded Shadow Maps

Environment Map

Image Based Lighting

Skybox

Fog

Terrain

Decal

Reflection Probe

Light Probe

GPU Skinning

Morph Target

Post Processing

Bloom

Depth of Field

Motion Blur

Ambient Occlusion

Screen Space Reflection用拡張

Volumetric Lighting用拡張

Render Pipelineカスタマイズ

15. アセットシステム

Asset ID・GUID

Asset Registry

Asset Database

Source Asset管理

Import済みAsset管理

メタファイル

アセット依存関係

循環依存検出

画像インポーター

3Dモデルインポーター

音声インポーター

フォントインポーター

シーンインポーター

glTFなどの標準形式対応

アセット再インポート

Import Settings

非同期読み込み

バックグラウンド読み込み

参照カウント

アセットアンロード

ストリーミング

Texture Streaming

Mesh Streaming

Hot Reload

Missing Asset代替表示

アセット検証

Derived Data Cache

アセットCooking

プラットフォーム別変換

圧縮

パッケージング

16. 物理・衝突判定

自作するより、物理ライブラリを統合できる構造を持たせることが重要です。

Physics World

固定更新

Collision Shape

Box

Sphere

Capsule

Convex Mesh

Triangle Mesh

2D Collider

Rigidbody

Static Body

Kinematic Body

Dynamic Body

Broad Phase

Narrow Phase

Contact生成

Trigger

Collision Layer

Collision Mask

Material

摩擦

反発

重力

Constraint

Joint

Raycast

Shape Cast

Sweep

Overlap Query

Character Controller

Continuous Collision Detection

Sleeping

物理デバッグ描画

Transform同期

マルチスレッド物理演算

決定論対応用設計

17. アニメーション
2D

Sprite Animation Clip

Frame Event

Animation State

Blend

Flipbook

2D Bone Animation

3D

Skeleton

Bone Hierarchy

Animation Clip

Pose

Skinning

Animation State Machine

Blend Tree

Cross Fade

Animation Layer

Additive Animation

Animation Mask

Root Motion

Animation Event・Notify

Retargeting

IK

Look At

Procedural Animation

Animation Compression

GPU Animation評価用拡張

18. パーティクル・VFX

Particle System

CPU Particle

GPU Particle

Emitter

Spawn Module

Update Module

Lifetime

Velocity

Acceleration

Color over Lifetime

Size over Lifetime

Rotation

Curve

Gradient

Sprite Sheet Particle

Mesh Particle

Ribbon

Trail

Particle Collision

Depth Sorting

Pooling

Burst

Sub Emitter

Event連携

VFX Graph

Node Graph

Niagara相当システムへ拡張できるデータ駆動設計

19. オーディオ

Audio Device

Audio Clip

WAVなどのデコード

圧縮音声デコード

ストリーミング再生

Audio Source

Audio Listener

2D Audio

3D Spatial Audio

距離減衰

Doppler Effect

Volume

Pitch

Pan

Loop

Pause・Resume

Mixer

Bus

Channel

Sound Group

同時発音数制御

優先度

Fade

Reverb

Filter

Audio Effect

音声アセット管理

オーディオスレッド

20. 入力システム

Keyboard

Mouse

Gamepad

Touch

Pen

ゲームパッド振動

デバイス接続・切断

Input Action

Input Axis

Input Mapping

キーリバインド

Modifier・Chord

Dead Zone

Sensitivity

複数プレイヤー入力

UI入力

ゲーム入力とUI入力の切り替え

Input Context

入力バッファ

長押し・ダブルタップ

テキスト入力

Unicode

日本語IME

入力記録・再生

21. ランタイムUI

Canvas

Widget

Panel

Button

Text

Image

Slider

Checkbox

Scroll View

List

Tree

Text Input

レイアウト

Anchor

Flex・Grid相当

Auto Layout

DPIスケーリング

Safe Area

フォント読み込み

フォントアトラス

Unicode表示

日本語表示

Text Shaping

Rich Text

クリッピング

Mask

Focus

キーボード・ゲームパッドナビゲーション

Event Bubbling

Drag & Drop

UI Animation

UI Style・Theme

アクセシビリティ用拡張

22. ゲームプレイ基盤

C++ Gameplay API

Component API

Gameplay Module

Game Mode

Player

Controller

Pawn・Character相当

イベント・Signal

State Machine

Timer

CoroutineまたはTask

Data Asset

Data Table

Gameplay Tag

Command System

Factory

Object Pool

Spawn System

Scene Transition

セーブ・ロード

チェックポイント

リプレイ

チート・デバッグコマンド

ゲーム固有コードとエンジンコードの分離

23. スクリプティング

必ずしも必要ではありませんが、第三者向けエンジンでは実質的に重要です。

スクリプト言語統合

C++バインディング

エンジンAPI公開

スクリプトコンポーネント

実行時ロード

スクリプトホットリロード

エラー・スタックトレース

デバッガー

サンドボックス

メモリ制限

実行時間制限

Reflection連携

Editor Inspector連携

Visual Scripting

Node Graph

カスタムノード

非同期処理ノード

イベントノード

24. AI・ナビゲーション

ゲームジャンルに応じて必要です。

Navigation Mesh

Grid Navigation

A* Pathfinding

Hierarchical Pathfinding

Dynamic Obstacle

Navigation Link

Crowd Simulation

Steering Behavior

Behavior Tree

Blackboard

Utility AI

State Machine

AI Perception

視覚・聴覚判定

Line of Sight

AI Debugger

AI Navigation Debug Draw

Goal Oriented Action Planning用拡張

25. ネットワーク・マルチプレイヤー

マルチプレイヤー対応なら必須です。

Socket抽象化

UDP・TCPトランスポート

Client・Server

Dedicated Server

Listen Server

Connection管理

Session管理

Packet Serialization

Packet Fragmentation

Reliable Delivery

Sequence Number

Timeout

Heartbeat

RPC

State Replication

Property Replication

Entity Replication

Snapshot

Delta Compression

Interest Management

Network Relevancy

Authority管理

Client Prediction

Server Reconciliation

Interpolation

Extrapolation

Lag Compensation

Rollback

ネットワーク時刻同期

帯域制御

パケット損失・遅延シミュレーション

切断・再接続

ロビー

マッチメイキング連携

サーバー側入力検証

不正パケット対策

26. エディタ

ドッキング可能なエディタUI

Scene View

Game View

Hierarchy

Inspector

Content Browser

Project Browser

Console

Toolbar

Status Bar

Property Editor

Transform Gizmo

移動・回転・拡縮

Local・World切り替え

Grid Snap

Vertex Snap

Selection

Multi Selection

Box Selection

Copy・Paste

Duplicate

Drag & Drop

Undo・Redo

Command Pattern

Scene保存

自動保存

クラッシュ復元

Prefab Editor

Material Editor

Animation Editor

Particle Editor

UI Editor

Tilemap Editor

Physics Editor

Input Mapping Editor

Project Settings

Build Settings

Play In Editor

Pause

Frame Step

Runtime Property Editing

エディタレイアウト保存

ショートカット設定

エディタ拡張API

Plugin管理

27. デバッグ機能

Log

Log Category

Log Level

Assert

Ensure

Stack Trace

Crash Handler

Crash Dump

Symbol解決

エラー画面

Debug Draw

Collider表示

Navigation表示

Bounding Box表示

Render Pass表示

シーン統計

FPS表示

Frame Time表示

Draw Call表示

Triangle Count表示

GPUメモリ表示

メモリリーク表示

Remote Console

Debug Command

Runtime Object Inspector

リプレイによる不具合再現

28. プロファイラー

CPU Profiler

GPU Profiler

Memory Profiler

Allocation Profiler

Thread Profiler

Job Profiler

Asset Loading Profiler

Physics Profiler

Audio Profiler

Network Profiler

Frame Timeline

Scope Marker

カスタムプロファイルマーカー

Hitch検出

フレーム予算監視

CSV・JSON出力

外部プロファイラー連携

RenderDocなどのGPUデバッガー連携

29. ビルドシステム

CMakeなどのビルド構成

Debug Build

Development Build

Profile Build

Shipping Build

プラットフォーム別ビルド

コンパイラー設定管理

依存ライブラリ管理

Reflectionコード生成

Shader Build

Asset Cooking

Incremental Build

Parallel Build

Unity Build対応

Precompiled Header

Build Cache

Cross Compile

Headless Build

Dedicated Server Build

Editor Build

Runtime Build

Plugin Build

インストーラー生成

パッケージ署名

自動アップデート

差分パッチ

30. テスト・品質保証

Unit Test

Integration Test

Engine Test

Rendering Test

Golden Image Test

Physics Test

Serialization Test

Asset Import Test

Network Test

Performance Test

Stress Test

Memory Leak Test

Determinism Test

Crash Recovery Test

Headless Test

CI

Sanitizer

AddressSanitizer

UndefinedBehaviorSanitizer

ThreadSanitizer

Static Analysis

Warning as Error

コードフォーマット

API互換性テスト

アセット破損検証

31. プラグイン・拡張機構

Engine Module

Game Module

Editor Module

Runtime Plugin

Editor Plugin

Plugin Manifest

依存関係管理

Pluginロード順序

動的ロード

静的リンク

API Version

ABI境界管理

Plugin設定

Plugin Asset

カスタムComponent

カスタムImporter

カスタムEditor Window

カスタムInspector

カスタムRender Pass

カスタムBuild Step

32. セーブ・永続化

Save Game

User Settings

Graphics Settings

Input Settings

セーブスロット

オートセーブ

非同期セーブ

セーブデータ圧縮

セーブデータ暗号化

チェックサム

破損検出

バージョン移行

プラットフォーム別保存先

クラウドセーブ連携用抽象化

33. ローカライズ

文字列テーブル

言語切り替え

地域設定

複数形

日付・時間表記

数値・通貨表記

右から左への文章

フォントフォールバック

翻訳アセット

音声差し替え

画像差し替え

ローカライズ不足検出

擬似ローカライズ

34. 製品化に必要な機能

プロジェクト作成

プロジェクトテンプレート

サンプルゲーム

APIドキュメント

チュートリアル

エンジン更新

プロジェクト移行ツール

バージョン互換性管理

Plugin SDK

クラッシュレポート

匿名テレメトリーの同意管理

ライセンス管理

サードパーティライセンス表示

エンジン設定のエクスポート

プロジェクト診断

依存関係診断

壊れたアセットの修復

コマンドラインツール

Headless Editor

自動Import・Cook・Build

35. セキュリティ

ネットワーク入力検証

ファイルパス検証

Path Traversal対策

不正アセット対策

バッファ境界検証

整数オーバーフロー対策

スクリプトサンドボックス

Pak整合性検証

署名付きパッケージ

機密情報を実行ファイルへ埋め込まない設計

クラッシュレポートの個人情報除去

サーバー権威型ゲーム設計

リソース消費上限

非信頼データのデシリアライズ制限

最小限成立するゲームエンジン

最初のプレイ可能なエンジンに絞るなら、最低限以下です。

ウィンドウ・プラットフォーム処理

メインループ・時間管理

キーボード・マウス・ゲームパッド入力

数学ライブラリ

Entity・Component・Transform

Scene読み込み・保存

2Dまたは3Dレンダラー

Shader・Texture・Mesh管理

Asset Manager

Collision・Physics

Audio

UI・Text

Gameplay用C++ API

シリアライズ

ログ・Assert・デバッグ描画

ビルド・アセットパッケージング

製品レベルで特に欠かせないもの

単にゲームが動くだけでなく、他の開発者が使えるエンジン製品にする場合は、次が重要です。

独自エディタ

Reflectionとコード生成

Asset DatabaseとImport Pipeline

Prefab

Undo・Redo

Play In Editor

非同期アセット読み込み

Hot Reload

Job System

CPU・GPU・メモリプロファイラー

Crash Dump

Asset Cooking

Plugin System

API・データのバージョン移行

自動テスト

サンプルプロジェクト

ドキュメント

実装順としては、基盤 → レンダラー → ワールド → アセット → シリアライズ → 入力・物理・音声 → エディタ → アニメーション・UI・VFX → プロファイラー・パッケージング → AI・ネットワークが安定します。
