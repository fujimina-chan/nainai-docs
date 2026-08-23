# メディア再生（機能設計）

初期 MVP / Phase 2 における、選択済み音声・動画ファイルのアプリ内再生機能を定義します。

本ドキュメントは主に **何を実現するか** を定めます。UI レイアウトは [phase2-ui.md](phase2-ui.md)、視覚ルールは [design-system.md](design-system.md)、採用技術は [media-technology.md](../architecture/media-technology.md) を参照してください。

関連:

- [MVP 要件](../requirements/mvp-requirements.md)
- [Phase 2 UI](phase2-ui.md)
- [Design System](design-system.md)
- [メディアファイル選択](media-selection.md)
- [システム概要](../architecture/system-overview.md)
- [メディア技術選定](../architecture/media-technology.md)
- [ADR-0005 media_kit](../adr/0005-media-kit.md)

## 1. 概要

[media-selection.md](media-selection.md) で選択されたローカルメディアファイルを、アプリ内で再生する。

バックエンド通信・DB・認証・クラウド同期は使用しない。

```text
選択済みローカルファイル
       ↓
読み込み（media_kit）
       ↓
アプリ内再生（音声 / 動画）
```

## 2. 前提

| 項目 | 内容 |
|------|------|
| 対象プラットフォーム（MVP） | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |
| 入力 | ファイル選択で得た参照（音声または動画） |
| 元ファイル | 再生操作によってコピー・移動・削除・上書き・編集しない |
| 採用技術 | `media_kit` 1.2.6 / `media_kit_video` 2.0.1 / `media_kit_libs_video` 1.0.7 |

- 音声・動画の両方を扱うため video 用構成を使用する
- `media_kit_libs_audio` と `media_kit_libs_video` は同時使用しない

## 3. Phase 2 の必須操作

| 操作 | 説明 |
|------|------|
| Select File / Select Another File | ファイル選択（詳細は media-selection.md） |
| Play / Pause | 同一 Primary Control を状態で切替 |
| Stop | 再生を止め、位置を先頭（00:00）へ戻す。Media はロードされたまま |
| Seek | 再生位置を任意位置へ変更する |
| Volume | アプリ内の再生音量を変更する |
| Repeat OFF / ONE | Phase 2 のリピート（下記） |

これ以外の Playback 操作を Phase 2 に追加しない（Previous / Next / Shuffle / 10 秒送り戻し / Fullscreen / Subtitle 等は対象外）。

## 4. 必須表示

| 表示 | 説明 |
|------|------|
| ファイル名 | 現在扱っているメディアのファイル名 |
| Current Time | 現在位置（Monospace。横幅が揺れないこと） |
| Total Duration | メディア全体の長さ（同上） |

### 動画の場合

- 映像をアプリ内に表示する（画面の主役）

### 音声の場合

- 空の Video Box を表示しない
- Static Audio Placeholder + File Name + Audio 種別表示
- Artist / Album / Codec 等の未取得 Metadata は表示しない
- Fake Dynamic Visualizer は禁止

レイアウトの詳細は [phase2-ui.md](phase2-ui.md)、トークンは [design-system.md](design-system.md) を参照。

## 5. 再生状態（Media State）

実装上の enum 名やクラス名は本ドキュメントでは固定しない。概念として少なくとも次を区別する。

| 状態 | 意味 |
|------|------|
| `unselected` | メディアが選択されていない |
| `loading` | 選択後、再生準備のために読み込み中 |
| `ready` | 読み込み成功。まだ再生していない、または再生可能な待機 |
| `playing` | 再生している |
| `paused` | 一時停止。Position を保持 |
| `stopped` | 再生を止め、Position が先頭（00:00） |
| `error` | 読み込みまたは再生に失敗した（Blocking） |

Audio / Video 種別は状態とは別に Media 情報として保持する。UI は状態から描画する。

`loading` 中は Play / Pause / Stop / Seek を操作不可とし、UI だけでなく Controller 側でも拒否する。

### 状態遷移（概念）

```text
unselected
  │ Select File
  ▼
loading
  ├─ 成功 ──▶ ready
  └─ 失敗 ──▶ error（Blocking）

ready / stopped
  │ Play
  ▼
playing
  ├─ Pause ─────▶ paused ──(Play)──▶ playing
  ├─ Stop ──────▶ stopped（位置は 00:00、Media はロード維持）
  ├─ 終端到達
  │    ├─ Repeat OFF ──▶ stopped（00:00）
  │    └─ Repeat ONE ──▶ 先頭から同じ Media を再生継続
  └─ 失敗 ──────▶ error（Blocking）または Non-blocking（維持可能な場合）

いずれの状態でも（error 含む）Select Another File が可能
  │
  ▼
現在の再生を止め、新しいメディアを loading へ
```

ファイル選択のキャンセルは再生状態を変更しない（[media-selection.md](media-selection.md)）。キャンセルは Error ではない。

## 6. Play / Pause

同一の Primary Control を状態によって切り替える。

| 状態 | Primary Control |
|------|-----------------|
| `ready` / `paused` / `stopped` | Play |
| `playing` | Pause |

Pause 時は Position を保持する。

## 7. Stop の意味

nainai における Stop は次を意味する。

1. Pause 相当に再生を止める
2. Position を **00:00** へ戻す
3. Media 自体はロードされたままとする

その後 Play すると先頭から再生できる。

| 操作 | 再生 | 位置 | Media |
|------|------|------|-------|
| Pause | 止める | 保持する | ロード維持 |
| Stop | 止める | 00:00 へ戻す | ロード維持 |

### media_kit の `Player.stop()` との差異

`media_kit` の `Player.stop()` と、nainai の Stop は意味が異なる場合がある。

- nainai の Stop は「再生停止 + 位置を先頭へ戻し + Media はロード維持」
- media_kit API の stop がリソース解放や open 状態の破棄を伴う場合、nainai Stop の意味にそのまま対応させない

実装では、nainai の Stop 仕様を満たすよう Controller 側で意味を定義する（必要なら pause + seek(0) 等で表現する）。

## 8. Seek

- ユーザーが再生位置を任意位置へ変更できる
- Phase 2 では Seek UI を提供する（レイアウトは phase2-ui.md）
- 時間精度は今回固定しない

将来の分割・タイムラインでは、より高精度な時間管理が必要になる。本設計のシークは、その将来要件を阻害しないこと（位置を「時間」として扱えること）を前提とする。

## 9. Volume

- Phase 2 では **アプリ側の再生音量** を変更できる
- **OS 全体の音量設定を変更する機能ではない**
- ミュート専用ボタンは Phase 2 の必須操作に含めない

## 10. Repeat

Phase 2 の Repeat は次のみ。

| 設定 | 終端到達時の挙動 |
|------|------------------|
| OFF | `stopped` へ遷移し、Position を 00:00 とする |
| ONE | 現在 Media を繰り返す |

- Repeat ONE Active は Accent Color で示す（[design-system.md](design-system.md)）
- Repeat 設定はファイル切替後も、**アプリ起動中は保持**する
- Repeat ALL は将来機能（Phase 2 では禁止）

## 11. ファイル切り替え

別ファイルが選択された場合:

1. 現在のメディア再生を止める
2. 新しいメディアを読み込む（`loading`）
3. 元ファイル自体には変更を加えない

切り替え前の再生位置・一時停止状態は引き継がない。

Repeat 設定はアプリ起動中は保持する（第 10 節）。

## 12. エラー

### Blocking Error

現在の Media を再生できない場合。Media 画面を Error 表示へ切り替えてよい。

例文言: `Could not load this file.`

- Select Another File を提供する
- 技術的 Exception は表示しない
- 原因が確定していない場合、`Unsupported format` / `File corrupted` などを断定しない
- 色: Semantic Error（Red）

想定要因の例（ユーザーへ断定文言としては出さない）:

- ファイルを開けない / 存在しない
- OS からアクセスを拒否された
- メディアとして読み込めない
- 再生処理に失敗した

### Non-blocking Error

現在 Media を維持可能な場合、Media 画面を Error 画面へ置き換えない。

- Notification / Banner を使用する
- 色: Semantic Warning（Amber）

### Unknown Error / 問題報告

将来的に問題報告機能へ接続予定とする。Phase 2 では Report ボタンを実装しない。

方針共通:

- アプリ全体をクラッシュさせない
- エラー後も別ファイル選択など通常操作へ復帰できること

## 13. UI との対応

必要 UI の配置・Desktop / Mobile 差分は [phase2-ui.md](phase2-ui.md) を正とする。

Design Token・色・余白・形状は [design-system.md](design-system.md) を正とする。

## 14. OS 別の考慮

Windows / Android / iOS では、メディア再生の実装詳細が異なる可能性がある。

Android / iOS ではサンドボックスや権限・ファイルアクセス制約が再生可否に影響しうる。詳細は実装時に確定する。

Windows では Phase 2 で OS 標準 Window Chrome を使用する（独自の Minimize / Maximize / Close は作らない）。詳細は [phase2-ui.md](phase2-ui.md)。

採用パッケージは [media-technology.md](../architecture/media-technology.md) を参照。

## 15. ライセンス注意

`media_kit` はネイティブメディアライブラリやコーデック等を利用する。パッケージ本体のライセンスだけで製品全体の利用可否を判断してはいけない。

**製品リリース前に、推移依存を含めたライセンス監査を必須とする。**（今回は監査未実施）

## 16. 将来機能との境界

Phase 2 / MVP の再生には含めない。

- Previous / Next / Shuffle
- Repeat ALL
- Playlist / Library / Folder Playback
- 波形 / Dynamic Visualizer
- タイムライン編集 / 区間設定 / 分割
- 歌詞 / 字幕 / Fullscreen
- メタデータ編集・未取得 Metadata の表示
- ファイル出力 / 保存先設定
- Equalizer / Compressor
- Drag & Drop
- Editor Navigation

本設計は、これら将来機能を阻害しないこと（時間位置の扱い、単一メディア切替モデル、非破壊方針、Repeat の拡張余地）を前提とする。

## 17. 未確定事項

- 対応メディア形式
- 再生状態の実装表現（enum / クラス等）
- Seek / Volume の時間精度・粒度
- Banner / Notification の消失条件
- 状態管理ライブラリ / ルーティングライブラリ
- 分割処理技術 / FFmpeg 採用有無
