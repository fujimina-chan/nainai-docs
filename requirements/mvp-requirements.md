# MVP 要件

初期 MVP（Minimum Viable Product）の対象機能と対象外を定義します。

## MVP の目的

端末内の音声・動画ファイルを選択し、アプリ内で基本的な再生操作ができることを確認する。

## 対象プラットフォーム（MVP）

- Windows
- Android
- iOS

Web / macOS / Linux は **初期 MVP の対象外** です。恒久的な非対応ではなく、**将来対応予定** です。詳細は [product-overview.md](product-overview.md) および [ADR-0002](../adr/0002-platform-roadmap.md) を参照してください。

## クライアント実装

MVP のクライアントは **Flutter** で実装します。

- Flutter project name: `nainai`
- 対象（MVP）: Windows / Android / iOS
- 将来対応予定: Web / macOS / Linux（MVP スコープ外）
- Dart は Flutter SDK に同梱されるものを使用
- 可能な限り共通の Flutter コードベースを使用
- プラットフォーム固有処理のみ、必要に応じて各 OS 側へ分離

関連 ADR: [ADR-0001 Flutter 採用](../adr/0001-use-flutter.md)

## アプリ識別子（確定）

| 項目 | 値 |
|------|-----|
| Organization ID | `com.fyna` |
| Android Application ID | `com.fyna.nainai` |
| iOS Bundle ID | `com.fyna.nainai` |

関連 ADR: [ADR-0003 Application ID](../adr/0003-application-id.md)

機能の詳細設計・技術選定は次を参照してください。

- [メディアファイル選択](../design/media-selection.md)
- [メディア再生](../design/media-playback.md)
- [Phase 2 UI](../design/phase2-ui.md)
- [Design System](../design/design-system.md)
- [メディア技術選定](../architecture/media-technology.md)

## メディア関連技術（確定）

| 領域 | 技術 |
|------|------|
| 入力ファイル選択 | `file_selector` 1.1.0（publisher: flutter.dev） |
| 音声・動画再生 | `media_kit` 1.2.6 / `media_kit_video` 2.0.1 / `media_kit_libs_video` 1.0.7 |

- 入力ファイル選択と出力先選択は別責務（出力先技術は未選定）
- `media_kit_libs_audio` と `media_kit_libs_video` は同時使用しない

関連 ADR: [ADR-0004](../adr/0004-file-selector.md)、[ADR-0005](../adr/0005-media-kit.md)

## 対象機能

### ファイル選択

| 機能 | 説明 |
|------|------|
| 音声ファイル選択 | OS 標準のファイル選択 UI を通じて、端末内の音声ファイルを選ぶ |
| 動画ファイル選択 | OS 標準のファイル選択 UI を通じて、端末内の動画ファイルを選ぶ |
| 選択ファイル情報表示 | 選択したファイルの情報（ファイル名等）をアプリ内に表示する |
| 別ファイルへの変更 | 再生中または停止中に、別のファイルを選択して切り替える |

### 再生操作

| 機能 | 説明 |
|------|------|
| 音声再生 | 選択した音声ファイルをアプリ内で再生する |
| 動画再生 | 選択した動画ファイルをアプリ内で再生する |
| 再生 | 一時停止中または停止状態から再生を開始する |
| 一時停止 | 再生中のメディアを一時停止する |
| 停止 | 再生を停止し、再生位置を先頭へ戻す |
| シーク | 再生位置を任意の時点へ移動する |
| 現在時間 | 再生中の現在位置（経過時間）を表示する |
| 総時間 | メディアファイルの総再生時間を表示する |
| 音量 | 再生音量を調整する |
| Repeat OFF / ONE | 終端時の繰り返し。ALL は将来機能 |

UI 大枠・Design System は [phase2-ui.md](../design/phase2-ui.md) / [design-system.md](../design/design-system.md) を正とする。

## 対象外（MVP / Phase 2 では実装しない）

以下の機能は MVP / Phase 2 のスコープ外です。

| 機能 | 備考 |
|------|------|
| Previous / Next / Shuffle | Phase 2 禁止 |
| Repeat ALL | 将来機能 |
| Playlist / Library / Settings | 将来機能 |
| Drag & Drop | Phase 2 対象外（OS File Picker のみ） |
| Fullscreen / Subtitle / 10 秒送り戻し | Phase 2 禁止 |
| Dynamic Audio Visualizer | 実音声非同期の疑似表現を含む。禁止 |
| 分割 | 将来機能 |
| 波形編集 | 将来機能 |
| 歌詞編集 | 将来機能 |
| 字幕編集 | 将来機能 |
| ファイル出力 | 将来機能 |
| Equalizer / Compressor / Editor / Timeline | 将来機能。Phase 2 UI に先行表示しない |
| テーマ選択 UI | 将来。Theme Architecture のみ先行定義 |
| 問題報告（Report）ボタン | 将来設計 |
| 独自 Window Chrome | Phase 2 では OS 標準を使用 |
| DB | 永続化レイヤーは MVP では不要 |
| ログイン | 認証機能は不要 |
| クラウド同期 | ローカルファイルのみ |
| バックエンド通信 | nainai-backend は MVP では未使用 |
| Web / macOS / Linux 向け実装 | 初期 MVP の対象外（将来対応予定） |

## 未確定事項

以下は MVP 実装時に決定が必要な項目です。本ドキュメントでは確定しません。

- **正式な再生対応形式一覧**（コーデック・コンテナ）。Selection の MediaKind rough classification 用拡張子は [media-selection.md](../design/media-selection.md) を参照。再生保証ではない
- 出力先選択ライブラリ / API（ファイル出力は MVP 対象外）
- 音量調整の粒度（スライダー範囲等）
- Banner / Notification の消失条件
- 状態管理ライブラリ / ルーティングライブラリ
- 分割処理技術 / FFmpeg 採用有無
- backend 技術 / DB
- フォントの Flutter 導入方法
- 初期 Lavender Accent の詳細 Token 一覧