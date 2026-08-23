# メディア技術選定

MVP におけるローカルファイル選択および音声・動画再生の技術選定を記録します。

機能要件（何を実現するか）は次を参照してください。

- [メディアファイル選択](../design/media-selection.md)
- [メディア再生](../design/media-playback.md)
- [Phase 2 UI](../design/phase2-ui.md)
- [Design System](../design/design-system.md)

本ドキュメントは **どの技術で実現するか** を定めます。

## 1. 対象プラットフォーム

| フェーズ | プラットフォーム |
|----------|------------------|
| 初期（MVP） | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |

選定パッケージは、将来の Web / macOS / Linux にも対応可能な構成とする。

## 2. ファイル選択技術（入力）

### 採用パッケージ

| 項目 | 内容 |
|------|------|
| パッケージ | `file_selector` |
| バージョン | `1.1.0` |
| publisher | `flutter.dev` |

関連 ADR: [ADR-0004](../adr/0004-file-selector.md)

### 用途（責務）

ユーザーが端末内に保存されている音声・動画ファイルを選択することに **限定** する。

- アップロードではない
- OS 標準のファイル選択 UI を利用する
- MVP では単一ファイル選択

### 採用理由

- Flutter 公式 publisher（`flutter.dev`）から提供されている
- OS 標準のファイル選択 UI を利用する目的に合致する
- Windows / Android / iOS に対応する
- 将来予定の Web / macOS / Linux にも対応する
- 単一ファイル選択という MVP 要件を満たす

## 3. 入力ファイル選択と出力先選択の責務分離

`file_selector` を将来のすべてのファイル入出力へ直接利用する設計にはしない。

nainai では将来、次の出力先を個別指定できる必要がある。

- 音声ファイルの出力先
- 動画ファイルの出力先
- 歌詞ファイルの出力先
- 字幕ファイルの出力先
- 説明ファイルの出力先

OS ごとにディレクトリ選択・保存先選択の制約が異なるため、次を **別責務** とする。

| 概念 | 責務 | MVP |
|------|------|-----|
| MediaSelection（入力ファイル選択） | 端末内の既存メディアを選ぶ | `file_selector` を使用 |
| OutputLocationSelection（出力先選択） | 書き出し先を選ぶ | 未実装。技術未選定 |

- 具体的なクラス名は未確定
- 将来の出力先選択技術は **今回決定しない**

## 4. メディア再生技術

### 採用パッケージ

| パッケージ | バージョン |
|------------|------------|
| `media_kit` | `1.2.6` |
| `media_kit_video` | `2.0.1` |
| `media_kit_libs_video` | `1.0.7` |

関連 ADR: [ADR-0005](../adr/0005-media-kit.md)

### 構成方針

- 音声と動画の両方を扱うため、**video 用構成**（`media_kit_libs_video`）を使用する
- `media_kit_libs_audio` と `media_kit_libs_video` を **同時使用しない**

### 責務

選択されたローカル音声・動画ファイルのアプリ内再生。

- 再生 / 一時停止 / 停止 / シーク / 音量
- 再生位置・総時間の取得
- 動画の映像描画

### 採用理由

- 音声・動画の両方を同じ再生基盤で扱える
- Windows / Android / iOS に対応する
- 将来予定の Web / macOS / Linux にも対応する
- 再生、一時停止、シーク、音量、再生位置等の MVP 要件に適合する
- 動画描画にも対応できる

## 5. 責務の整理

```text
Local Media File
       ↓
MediaSelection（file_selector）…… 入力のみ
       ↓
nainai-client
       ↓
Local Media Playback（media_kit + media_kit_video + media_kit_libs_video）

（将来）
OutputLocationSelection（技術未選定）…… 出力先のみ
```

| 領域 | 技術 | 備考 |
|------|------|------|
| 入力ファイル選択 | `file_selector` 1.1.0 | 出力先選択には使わない |
| 再生 | `media_kit` 系（上記バージョン） | audio libs との同時使用なし |
| 出力先選択 | 未選定 | MediaSelection と分離 |

## 6. ライセンス注意事項

製品化を前提とする。

- Flutter / Dart パッケージ本体のライセンスだけで、製品全体の利用可否を判断してはいけない
- 特に `media_kit` はネイティブメディアライブラリやコーデック等を利用する
- **製品リリース前に、推移依存を含めたライセンス監査を必須とする**

今回、ライセンス監査そのものは実施しない。

## 7. 未確定事項

- 対応音声形式
- 対応動画形式
- 出力先選択ライブラリ / API
- ファイル書き出し方式
- 状態管理ライブラリ
- ルーティングライブラリ
- 分割処理技術
- FFmpeg 採用有無
- backend 技術 / DB
