# リポジトリ構成

nainai プロジェクトは 4 つのリポジトリで構成されます。

## リポジトリ一覧

```text
nainai-client    … アプリ本体
nainai-backend   … サーバー機能（将来）
nainai-infra     … 開発・配布基盤
nainai-docs      … 要件・設計・ADR の正本
```

## 各リポジトリの責務

### nainai-client

| 項目 | 内容 |
|------|------|
| 役割 | Flutter アプリ本体 |
| Flutter project name | `nainai` |
| 実装 | Flutter（Dart は Flutter SDK に同梱されるものを使用） |
| 初期対象（MVP） | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |
| コードベース | 可能な限り共通の Flutter コードベース。プラットフォーム固有処理のみ各 OS 側へ分離 |
| アプリ識別子 | Organization ID `com.fyna` / Android・iOS `com.fyna.nainai` |
| 内容 | UI、ファイル選択、メディア再生、将来の編集・出力機能 |
| MVP | 端末内ファイルの選択とアプリ内再生 |

### nainai-backend

| 項目 | 内容 |
|------|------|
| 役割 | 将来的なサーバー機能 |
| 内容 | API、認証、データ永続化等（詳細未決定） |
| MVP | **未使用** |

### nainai-infra

| 項目 | 内容 |
|------|------|
| 役割 | 開発環境、ビルド、CI/CD、配布基盤 |
| 内容 | ビルドパイプライン、ストア配布設定、開発用ツール等 |
| MVP | クライアントのビルド・配布に必要な最小限の基盤 |

### nainai-docs

| 項目 | 内容 |
|------|------|
| 役割 | プロジェクト全体の要件・設計・ADR の正本 |
| 内容 | 要件定義、アーキテクチャ文書、Architecture Decision Records |
| MVP | 本リポジトリに記載のとおり |

## リポジトリ間の関係（MVP）

```text
nainai-docs ────────── 設計・要件の正本
     │
     │ （参照）
     ▼
nainai-client ◄──── nainai-infra
  （アプリ本体）      （ビルド・配布）

nainai-backend … MVP では未使用
```

## 開発運用

### 別 Cursor ウィンドウでの管理

各リポジトリは **別 Cursor ウィンドウで管理** されます。

- あるリポジトリの Cursor ウィンドウから、他リポジトリのファイルを直接編集しない
- 設計・要件の変更は nainai-docs を正本として更新する
- 実装の変更は該当する実装リポジトリ（client / backend / infra）で行う

### 変更の流れ

1. 要件や設計に変更が生じた場合 → **nainai-docs** を先に更新
2. クライアント実装に反映 → **nainai-client** で作業
3. インフラ・配布に関わる変更 → **nainai-infra** で作業
4. サーバー機能が必要になった場合 → **nainai-backend** で作業（将来）

## 未確定事項

- 各リポジトリのブランチ戦略
- リリース・バージョニングの運用
- リポジトリ間の依存関係の自動化（サブモジュール、モノレポ化等の要否）
