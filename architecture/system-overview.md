# システム概要

nainai のシステム構成とデータフローを整理します。

## 基本思想

nainai は **ローカルファースト** のアプリケーションです。

端末内に保存されたメディアファイルを、OS 標準のファイル選択 UI 経由で取得し、クライアントアプリ内で直接再生します。MVP ではサーバー（バックエンド）を経由しません。

## クライアント技術（確定）

| 項目 | 内容 |
|------|------|
| Flutter project name | `nainai` |
| 実装 | nainai-client（Flutter アプリ） |
| 言語 | Dart（Flutter SDK に同梱されるものを使用） |
| 初期対象（MVP） | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |
| コードベース | 可能な限り共通の Flutter コードベース |
| プラットフォーム固有処理 | 必要に応じて各 OS 側へ分離 |

関連 ADR:

- [ADR-0001 Flutter 採用](../adr/0001-use-flutter.md)
- [ADR-0002 プラットフォームロードマップ](../adr/0002-platform-roadmap.md)

## アプリ識別子（確定）

| 項目 | 値 |
|------|-----|
| Organization ID | `com.fyna` |
| Android Application ID | `com.fyna.nainai` |
| iOS Bundle ID | `com.fyna.nainai` |

関連 ADR: [ADR-0003 Application ID](../adr/0003-application-id.md)

## MVP 時のデータフロー

```text
Local Media File
       ↓
Operating System File Picker
       ↓
MediaController（MediaState 正本）
       ↓
Local Media Playback
```

### 各要素の説明

| 要素 | 説明 |
|------|------|
| Local Media File | 端末内ストレージに保存されている音声・動画ファイル |
| Operating System File Picker | Windows / Android / iOS それぞれの OS が提供するファイル選択 UI |
| MediaController | Selection / Playback Service を束ね、immutable MediaState を状態正本とする（Phase 2-3 実装済み。詳細は [media-playback.md](../design/media-playback.md)） |
| Local Media Playback | Flutter アプリ内でのメディア再生（`media_kit` 系。詳細は [media-technology.md](media-technology.md)） |

## メディア技術（確定）

| 領域 | 技術 |
|------|------|
| 入力ファイル選択 | `file_selector` 1.1.0 |
| 音声・動画再生 | `media_kit` 1.2.6 / `media_kit_video` 2.0.1 / `media_kit_libs_video` 1.0.7 |

- 入力（MediaSelection）と出力先（OutputLocationSelection）は別責務
- 出力先選択技術は未選定
- 製品リリース前に推移依存を含めたライセンス監査を必須とする

詳細: [media-technology.md](media-technology.md)

関連 ADR: [ADR-0004](../adr/0004-file-selector.md)、[ADR-0005](../adr/0005-media-kit.md)

## UI / Design System（Phase 2）

再生 UI 大枠と Design System は次を正とする。

- [phase2-ui.md](../design/phase2-ui.md)
- [design-system.md](../design/design-system.md)
- [media-playback.md](../design/media-playback.md)

開発環境ルール・Known Issue は [development-environment.md](development-environment.md) を参照。

## MVP で使用しないコンポーネント

| コンポーネント | 状態 |
|----------------|------|
| nainai-backend | 未使用 |
| 外部 API | 未使用 |
| データベース | 未使用 |
| クラウドストレージ | 未使用 |

## 将来の拡張（方向性のみ）

将来的に以下を追加できる可能性があります。**API 構成・DB 構成は未決定** です。

### プラットフォーム拡張

初期 MVP の Windows / Android / iOS に加え、**Web / macOS / Linux** への対応を予定しています。共通 Flutter コードベースを維持し、プラットフォーム固有処理のみ個別実装します。

### バックエンド連携（方向性）

```text
Local Media File
       ↓
Operating System File Picker
       ↓
nainai-client ──→ nainai-backend（将来）
       ↓
Local Media Playback / File Output
```

想定される将来機能の例（詳細未確定）:

- ユーザーアカウント・認証
- クラウド同期
- 分割データ・歌詞・字幕等のサーバー側保存
- 複数端末間でのデータ共有

## 未確定事項

- 対応する音声・動画フォーマット
- 出力先選択ライブラリ / API
- 将来 nainai-backend を追加する際の API 設計
- データ永続化の方式（ローカル DB、ファイルベース等）
- 認証・同期の方式
- 分割処理技術 / FFmpeg 採用有無
- 状態管理ライブラリ / ルーティングライブラリ
