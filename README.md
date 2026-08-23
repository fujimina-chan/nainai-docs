# nainai-docs

nainai プロジェクト全体の設計・要件・意思決定を管理する正本リポジトリです。

アプリケーションコード、バックエンドコード、インフラ実装はこのリポジトリには配置しません。

## nainai について

nainai は、端末内に保存されている音声・動画ファイルを選択し、アプリ内で再生・編集・出力するための Flutter アプリケーションです。

初期 MVP の対象は **Windows / Android / iOS** です。**Web / macOS / Linux** は初期 MVP の対象外ですが、将来対応予定です（恒久的な非対応ではありません）。

詳細は [product-overview.md](requirements/product-overview.md)、[system-overview.md](architecture/system-overview.md)、および [ADR](adr/README.md) を参照してください。

## 目的

- プロジェクト全体の要件・設計・意思決定記録（ADR）を一箇所で管理する
- 各実装リポジトリ（client / backend / infra）の開発判断の根拠を残す
- 設計変更時に、このリポジトリを正本として更新する

## ディレクトリ構成

```text
nainai-docs/
├── README.md                          # このファイル（入口）
├── requirements/                      # 要件定義
│   ├── product-overview.md            # プロダクト全体像
│   └── mvp-requirements.md            # 初期 MVP の要件
├── design/                            # 機能設計
│   ├── media-selection.md             # 端末内メディアファイル選択
│   └── media-playback.md              # 音声・動画のアプリ内再生
├── architecture/                      # アーキテクチャ
│   ├── system-overview.md             # システム構成
│   ├── media-technology.md            # メディア技術選定
│   └── repository-structure.md        # リポジトリ構成と責務
└── adr/                               # Architecture Decision Records
    ├── README.md                      # ADR の運用方針
    ├── 0001-use-flutter.md
    ├── 0002-platform-roadmap.md
    ├── 0003-application-id.md
    ├── 0004-file-selector.md
    └── 0005-media-kit.md
```

## ドキュメント一覧

| 文書 | 内容 |
|------|------|
| [product-overview.md](requirements/product-overview.md) | nainai の目的、プラットフォームロードマップ、将来の方向性 |
| [mvp-requirements.md](requirements/mvp-requirements.md) | 初期 MVP の対象機能と対象外 |
| [media-selection.md](design/media-selection.md) | 端末内メディアファイル選択の機能設計 |
| [media-playback.md](design/media-playback.md) | 音声・動画のアプリ内再生の機能設計 |
| [system-overview.md](architecture/system-overview.md) | システム構成とデータフロー |
| [media-technology.md](architecture/media-technology.md) | ファイル選択・再生の技術選定 |
| [repository-structure.md](architecture/repository-structure.md) | 4 リポジトリの責務と開発運用 |
| [adr/README.md](adr/README.md) | ADR の目的と基本形式 |
| [ADR-0001](adr/0001-use-flutter.md) | Flutter 採用 |
| [ADR-0002](adr/0002-platform-roadmap.md) | プラットフォームロードマップ |
| [ADR-0003](adr/0003-application-id.md) | Application ID |
| [ADR-0004](adr/0004-file-selector.md) | 入力ファイル選択に file_selector を採用 |
| [ADR-0005](adr/0005-media-kit.md) | 音声・動画再生に media_kit を採用 |

## 設計変更時の運用

要件・アーキテクチャ・重要な技術判断を変更する場合は、**このリポジトリ（nainai-docs）を正本として先に更新**してください。

実装リポジトリ側の変更だけでは、プロジェクト全体の合意形成が追いつかなくなる可能性があります。

## 関連リポジトリ

| リポジトリ | 役割 |
|------------|------|
| nainai-client | Flutter アプリ本体（初期: Windows / Android / iOS、将来: Web / macOS / Linux） |
| nainai-backend | 将来的なサーバー機能（MVP では未使用） |
| nainai-infra | 開発環境、ビルド、CI/CD、配布基盤 |
| nainai-docs | 要件・設計・ADR の正本（このリポジトリ） |

各リポジトリは別 Cursor ウィンドウで管理されます。詳細は [repository-structure.md](architecture/repository-structure.md) を参照してください。
