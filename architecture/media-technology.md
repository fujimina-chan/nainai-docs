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

機能仕様（SelectedMedia / MediaKind / Error）は [media-selection.md](../design/media-selection.md) を正とする。

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

### 実装確認（file_selector 1.1.0）

確認済み API:

- `openFile()`
- Cancel 時は `null`

`XTypeGroup` で用いるフィルタ:

- `extensions`
- `mimeTypes`
- `uniformTypeIdentifiers`

Platform 確認事項（**当該バージョンで確認した範囲**。将来バージョンでも同一とは断定しない）:

| プラットフォーム | 確認事項 |
|------------------|----------|
| Windows | `extensions` が必要 |
| Android | `extensions` または `mimeTypes` |
| iOS | UTI（`uniformTypeIdentifiers`）指定が必要 |

Web source 実装詳細は本フェーズの対象外。

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

機能仕様（Stop / Repeat / Volume / Error）は [media-playback.md](../design/media-playback.md) を正とする。

### 構成方針

- 音声と動画の両方を扱うため、**video 用構成**（`media_kit_libs_video`）を使用する
- `media_kit_libs_audio` と `media_kit_libs_video` を **同時使用しない**
- `media_kit_libs_audio_video` 等を重複導入しない

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

### 初期化（実装確認）

Player / VideoController 利用前に、次の順で初期化する。

```text
WidgetsFlutterBinding.ensureInitialized()
  ↓
MediaKit.ensureInitialized()
  ↓
runApp(...)
```

### Player / VideoController lifecycle（実装確認）

Playback Service（MediaKitPlaybackService）は Service lifetime で次を所有する。

- `Player` 1 個
- `VideoController` 1 個

ファイルごとに再生成しない。

`media_kit_video` 2.0.1 では、VideoController に独立した dispose API がないことを確認済み。Player.dispose 側の lifecycle を使用する。

**Ownership:** VideoController の ownership は `MediaKitPlaybackService`。Presentation は dispose しない。

### VideoController Presentation 境界（Phase 2-4 確定）

```text
MediaKitPlaybackService
    owns VideoController
        ↓
Application Composition Root
        ↓
MediaKitVideoSurface
        ↓
media_kit Video Widget
```

方針:

- `MediaController` / `MediaState` / `MediaPlaybackService`（abstract）には VideoController や media_kit 型を公開しない
- `MediaKitPlaybackService` concrete class のみに read-only `VideoController` getter を設ける
- MediaScreen は Concrete Playback Service へ依存しない
- Video Surface を Composition Root から注入する
- Presentation は Native VideoController を Widget Test で必須にしない（Fake Surface 注入で状態 UI を検証可能）。テスト実装の細部は本文書の対象外

### Application Composition（Phase 2-4）

`NainaiApp` は StatefulWidget。Application lifetime で次を 1 組だけ生成する（Widget build ごとには生成しない）。

- `FileSelectorMediaSelectionService`
- `MediaKitPlaybackService`
- `MediaController`

`NainaiApp.dispose` では `MediaController.dispose()` を呼ぶ。

MediaController が MediaPlaybackService の dispose まで担当するため、Playback Service を Application 側から二重 dispose しない。

### Load（実装確認）

```text
SelectedMedia.sourceUri
  ↓
Media(...)
  ↓
Player.open(..., play: false)
```

自動再生しない。

ファイル切替時、旧 Media への `seek(0)` / `stop` を必須とせず、新 Media をそのまま `open` する（製品仕様は [media-playback.md](../design/media-playback.md)）。

### Player.open の concurrency（実装確認）

media_kit 1.2.6 の Native Player 実装確認:

- 同一 Player への operation は内部 lock で直列化される
- cancel API はない

nainai 側 Service では、さらに次で保護する。

- load generation
- Future chain による serialization

例:

```text
A open 中
  ↓
B 要求
  ↓
C 要求
  ↓
A 完了
  ↓
B が open 前に stale なら skip
  ↓
C open
```

最後に要求された Media が最終 Media になるよう保護する。

加えて MediaController 側の `_loadGeneration` により、stale な async 結果での State 更新を抑制する（**Service + Controller 二段防御・実装済み**）。製品側の状態遷移は [media-playback.md](../design/media-playback.md) を参照。

### Stop（Package 対応）

nainai Stop は `pause` + `seek(Duration.zero)`。**`Player.stop()` は使わない**（Media unload のため）。詳細は [media-playback.md](../design/media-playback.md)。

### Volume scale（実装確認）

| 境界 | スケール |
|------|----------|
| nainai 内部 | `0.0`～`1.0` |
| media_kit 1.2.6 | `0`～`100` |

変換は Playback Service 境界で行う。scale を UI / MediaState へ漏らさない。

### Repeat（Package 対応）

| nainai | media_kit 1.2.6 |
|--------|-----------------|
| Repeat OFF | `PlaylistMode.none` |
| Repeat ONE | `PlaylistMode.single` |

Repeat ONE は media_kit の single repeat を利用する。manual な completed → seek → play による Repeat は行わない。Repeat ALL は Phase 2 対象外。

### Error stream の注意（実装確認）

一部の open / decoder / file 系失敗は、open Future の throw ではなく後続 error stream として流れる場合がある。Service 単独での完全分類はできない。

Phase 2-3: MediaController は `errorStream` の `playbackFailed` 受信時、現在 status が `loading` なら `loadFailed` へ寄せる（Phase 2 範囲の分類。100% の由来識別は断定しない）。製品側の Error 区分は [media-playback.md](../design/media-playback.md) を参照。

## 5. 責務の整理

```text
Local Media File
       ↓
MediaSelection（file_selector）…… 入力のみ
       ↓
MediaController（MediaState 正本。Selection / Playback Service を Injection）
       ↓
Local Media Playback（media_kit + media_kit_video + media_kit_libs_video）

（将来）
OutputLocationSelection（技術未選定）…… 出力先のみ
```

| 領域 | 技術 | 備考 |
|------|------|------|
| 入力ファイル選択 | `file_selector` 1.1.0 | 出力先選択には使わない |
| 状態・操作境界 | MediaController | Package 型を漏らさない。詳細は media-playback.md |
| 再生 | `media_kit` 系（上記バージョン） | audio libs との同時使用なし |
| 出力先選択 | 未選定 | MediaSelection と分離 |

## 6. ライセンス注意事項

製品化を前提とする。

- Flutter / Dart パッケージ本体のライセンスだけで、製品全体の利用可否を判断してはいけない
- 特に `media_kit` はネイティブメディアライブラリやコーデック等を利用する
- **製品リリース前に、推移依存を含めたライセンス監査を必須とする**

今回、ライセンス監査そのものは実施しない。

## 7. 未確定事項

- **正式な再生対応形式一覧**（Selection の rough classification 用拡張子とは別。再生保証ではない）
- 出力先選択ライブラリ / API
- ファイル書き出し方式
- 状態管理ライブラリ
- ルーティングライブラリ
- 分割処理技術
- FFmpeg 採用有無
- backend 技術 / DB
- Windows build 環境制約の恒久解決策（[development-environment.md](development-environment.md)）
- Android 依存取得時の SSL 問題の恒久解決策（同上）
