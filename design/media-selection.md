# メディアファイル選択（機能設計）

初期 MVP における、端末内メディアファイルの選択機能を定義します。

本ドキュメントは主に **何を実現するか** を定めます。採用技術の詳細は [media-technology.md](../architecture/media-technology.md) を参照してください。

関連:

- [MVP 要件](../requirements/mvp-requirements.md)
- [メディア再生](media-playback.md)
- [Phase 2 UI](phase2-ui.md)
- [Design System](design-system.md)
- [システム概要](../architecture/system-overview.md)
- [メディア技術選定](../architecture/media-technology.md)
- [ADR-0004 file_selector](../adr/0004-file-selector.md)

## 1. 概要

ユーザーが端末内に保存されている音声・動画ファイルを選び、nainai がそのファイルへアクセスして利用する機能です。

**アップロードではありません。** Web への送信やクラウドへの送信は行いません。OS のファイル選択 UI を通じて、ローカルファイルへの参照を取得します。

```text
ユーザー操作
    ↓
OS のファイル選択 UI（file_selector）
    ↓
選択成功 → ファイル参照をアプリが保持
    ↓
再生機能へ渡す（media-playback.md）
```

## 2. 前提

| 項目 | 内容 |
|------|------|
| 対象プラットフォーム（MVP） | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |
| 通信 | なし（ローカル完結） |
| DB / 認証 / クラウド同期 | 使用しない |
| 採用技術（入力選択） | `file_selector` 1.1.0（publisher: flutter.dev） |

## 3. ファイル選択開始

- ユーザー操作によって OS のファイル選択 UI を開く
- **アプリ起動時に自動でファイル選択画面を開かない**
- 未選択時の主役操作は **Select File**（選択後は **Select Another File**）。画面方針は [phase2-ui.md](phase2-ui.md)
- Phase 2 では **Drag & Drop は対象外**（OS File Picker のみ）

## 4. 選択対象

| 種別 | MVP |
|------|-----|
| 音声（`MediaKind.audio`） | 対象 |
| 動画（`MediaKind.video`） | 対象 |

- OS のファイル選択 UI で音声・動画を選べる形にする
- **正式な再生対応形式一覧（official supported formats）は未確定**。再生保証でもない
- MediaKind の rough classification 用拡張子は後述。Package API 詳細は [media-technology.md](../architecture/media-technology.md)

## 5. 単一ファイル選択

- 初期 MVP では **一度に 1 ファイルのみ** 選択する
- 将来の複数ファイル選択（プレイリスト等）を阻害しない設計とする
  - 単一選択を前提にアプリ状態を固定しすぎない
  - 「現在選択中のメディア」を 1 つ扱うモデルを基本とし、将来複数へ拡張可能な余地を残す

## 6. キャンセル

ユーザーがファイル選択をキャンセルした場合:

| 方針 | 内容 |
|------|------|
| エラー扱いしない | キャンセルは正常な操作結果とする |
| 既存メディアを破棄しない | 既に選択・再生中のファイルがある場合、それを勝手に破棄しない |
| UI を異常状態にしない | キャンセル前の状態を維持する |

`file_selector` 1.1.0 では Cancel 時に `openFile()` が `null` を返す（確認範囲は当該バージョン）。

MediaController（Phase 2-3）: `MediaSelectionService.selectMedia()` が `null`（Cancel）のとき **State を一切変更しない**（`selectedMedia` / `status` / `position` / `duration` / `volume` / `repeatMode` / `error` を維持）。Cancel は Error ではない。

## 7. 選択成功後に扱う情報（SelectedMedia）

選択成功時、Domain では **`SelectedMedia`** として扱う。

| 情報 | 表現 | 説明 |
|------|------|------|
| ファイル名 | （表示用） | UI 表示 |
| ソース参照 | `Uri sourceUri` | 再生等でアクセスするための参照 |
| メディア種別 | `MediaKind`（`audio` / `video` のみ） | rough classification の結果 |

方針:

- `XFile` を Domain 全体へ流さない
- ローカル Path は `Uri.file(...)` で **file URI** へ変換して保持する
- `String path` を恒久 Domain Model へ固定しない

### MediaKind の rough classification（Selection 実装）

分類手順:

1. `audio/*` / `video/*` の MIME が利用可能なら優先
2. MIME が得られなければ extension fallback
3. どちらでも分類不可なら `unsupportedMedia`

| MediaKind | rough classification 用 extension（例） |
|-----------|----------------------------------------|
| audio | mp3, wav, m4a, aac, flac, ogg, opus, wma |
| video | mp4, mkv, mov, avi, webm, m4v, wmv, mpeg, mpg |

**重要:** 上表は **MediaKind classification 用** である。正式な再生対応形式一覧ではなく、再生保証（playback guarantee）でもない。

## 8. 元ファイルに対する非破壊方針

ファイルを選択しただけでは、元ファイルに対して次を行わない。

- コピーしない
- 移動しない
- 削除しない
- 上書きしない
- 編集しない

選択は「参照の取得」であり、元ファイルの内容や配置を変更しない。

## 9. 別ファイル選択

- 再生中・停止中を問わず、ユーザーは別のファイルを選択できる
- 別ファイル選択時の再生側の挙動は [media-playback.md](media-playback.md) の「ファイル切り替え」に従う
  - 新しいメディアを読み込む（切替時の Player 操作詳細は media-playback / media-technology）
  - 元ファイル自体には変更を加えない

## 10. 入力選択と出力先選択の分離

本機能は **入力ファイル選択（Media Selection）** のみを扱う。

将来の出力先選択（OutputLocationSelection）とは別責務とする。`file_selector` を将来のすべてのファイル入出力へ直接利用する設計にはしない。出力先技術は未選定。

詳細: [media-technology.md](../architecture/media-technology.md)

## 11. OS 別の考慮

Windows / Android / iOS では、ファイル選択・アクセスの実装詳細が異なる。

| プラットフォーム | 方針 |
|------------------|------|
| Windows | OS のファイル選択 UI を `file_selector` 経由で利用 |
| Android | サンドボックス、ストレージ権限、ファイルアクセス制約がある |
| iOS | サンドボックス、権限、ファイルアクセス制約がある |

`XTypeGroup`（extensions / mimeTypes / uniformTypeIdentifiers）の Platform 要件は、`file_selector` 1.1.0 で確認した範囲として [media-technology.md](../architecture/media-technology.md) に記録する。将来バージョンでも同一とは断定しない。

権限・アクセス制約の詳細は実装・実機検証で確定する。iOS での未検証挙動を確定仕様にしない。

## 12. Picker の二重起動防止

OS File Picker の二重起動防止は、次の **二段 Guard として実装済み**（Phase 2-3）。

| 層 | 内容 |
|----|------|
| `FileSelectorMediaSelectionService` | Service 側 transient guard |
| `MediaController` | `_isSelecting` により、Picker 実行中の 2 回目の `selectMedia()` を Service へ到達させない |

- Picker 実行中の再呼び出しは、新しい Picker を開かない
- 成功 / Cancel / Exception の全経路で Guard を解除する
- `_isSelecting` は **MediaPlaybackStatus に含めない** transient state

## 13. 必要となる UI 要素

- Select File / Select Another File
- 選択中ファイルのファイル名表示（選択後）
- キャンセル時に既存状態を維持できる画面構成

配色・余白・形状・タイポグラフィは [design-system.md](design-system.md)、画面構成は [phase2-ui.md](phase2-ui.md) を正とする。

## 14. エラー（Selection）

選択成功後の読み込み・再生失敗は主に [media-playback.md](media-playback.md) で扱う。

Selection フローで確認済みの区分:

| 結果 | 扱い |
|------|------|
| Picker Cancel | Error ではない。MediaController は State を変更しない |
| Picker 起動 / 取得失敗 | `selectionFailed`（**non-blocking**。現在 Media / status 維持、error のみ更新） |
| Audio / Video 分類不能 | `unsupportedMedia`（同上） |

方針:

- Selection Service の Package 固有 Exception は nainai 側 Exception（Controller では `MediaError`）へ変換する
- ユーザー向け文言は Presentation 層の責務
- Blocking / Non-blocking の表示方針は [media-playback.md](media-playback.md) / [phase2-ui.md](phase2-ui.md) に従う
- 技術的 Exception 文字列をそのまま表示しない
- アプリ全体をクラッシュさせない
- キャンセルはエラーに含めない

## 15. 将来機能との境界

MVP / Phase 2 のファイル選択には含めない。

- Drag & Drop
- 複数ファイル同時選択 / Playlist / Folder Playback / Device Media
- 波形 / タイムライン編集 / 区間設定 / 分割
- 歌詞 / 字幕 / 説明 / メタデータ編集
- ファイル出力 / 保存先設定（OutputLocationSelection）

本設計は、これら将来機能を阻害しないこと（単一選択モデルの拡張余地、非破壊方針、入出力責務分離）を前提とする。

## 16. 未確定事項

- **正式な再生対応形式一覧**（コーデック・コンテナ。rough classification とは別）
- 出力先選択ライブラリ / API
- iOS 実機での選択・アクセス挙動の追加検証
