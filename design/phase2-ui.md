# Phase 2 UI

Phase 2 における再生画面の UI 方針を定義します。

視覚ルール（色・余白・形状・タイポグラフィ）は [design-system.md](design-system.md)、再生操作の機能仕様は [media-playback.md](media-playback.md)、ファイル選択は [media-selection.md](media-selection.md) を正とする。

関連:

- [Design System](design-system.md)
- [メディア再生](media-playback.md)
- [メディアファイル選択](media-selection.md)
- [MVP 要件](../requirements/mvp-requirements.md)

## 1. 位置づけ

Phase 2 は、初期 MVP の再生体験に対する **UI 大枠設計** である。

- Google Stitch 出力はレイアウト・情報階層・余白・配置・色調・Desktop / Mobile 差の参考にのみ用いる
- Stitch HTML を正とせず、Flutter へそのまま移植しない
- 仕様と Stitch が矛盾した場合は **必ず nainai-docs を優先**する
- 旧仕様・未実装機能を UI へ先行表示しない

Stitch 成果物の保存場所・版管理・正本性は [design-system.md](design-system.md) の「Stitch 成果物の扱い」を正とする。

## 2. Phase 2 に存在する Playback UI 操作

| 操作 / 表示 | 備考 |
|-------------|------|
| Select File | 未選択時の主役 |
| Select Another File | メディア選択後の別ファイル選択 |
| Play / Pause | 同一 Primary Control を状態で切替 |
| Stop | 位置を 00:00 へ戻す（詳細は media-playback.md） |
| Seek | 任意位置へ移動 |
| Current Time | Monospace 時間表示 |
| Total Duration | Monospace 時間表示 |
| Volume | アプリ内再生音量 |
| Repeat OFF | 終端後は stopped（00:00） |
| Repeat ONE | 現在 Media を繰り返す。Active は Accent |

これ以外の Playback 操作を Phase 2 UI へ追加しない。

## 3. 画面は Media State から描画する

最低限の状態: `unselected` / `loading` / `ready` / `playing` / `paused` / `stopped` / `error`

Audio / Video 種別は別途 Media 情報として保持し、UI は状態と種別から描画する。

状態の意味と遷移は [media-playback.md](media-playback.md) を正とする。

## 4. Desktop Control Bar（確定）

Desktop では Media 種別と状態に応じて、次の Control 構造を採用する。

「Bottom Control Area」と「Floating Control Bar」の二者択一は解消済みである。  
**Video は Floating Control Bar、Audio は画面下部の固定 Control Panel** とする。

### 4.1 Video（Desktop）

動画領域下部に **Floating Control Bar** を配置する。

- Control Bar は動画 Content を主役とし、動画全体を大きく確保したまま操作できる構造にする
- Phase 2 では Control Bar を **常時表示**する

Phase 2 ではまだ実装しない（将来改善）:

- 一定時間後の自動非表示
- Mouse Hover のみで表示
- Cursor 移動による表示制御

Floating Control Bar に含む Phase 2 操作:

- Current Time
- Seek
- Total Duration
- Play / Pause
- Stop
- Repeat OFF / ONE
- Volume
- Select Another File

Phase 2 対象外の操作を追加しない。

### 4.2 Audio（Desktop）

画面下部に **固定 Control Panel** を配置する。

Video のように Media へ Overlay する必要はない。

Control Panel に含む操作:

- Current Time
- Seek
- Total Duration
- Play / Pause
- Stop
- Repeat OFF / ONE
- Volume
- Select Another File

### 4.3 Unselected（Desktop）

Playback Control Bar / Panel を表示しない。

Select File を主役とした Empty State のみ表示する。

大量の Disabled Control は表示しない。

### 4.4 Loading（Desktop）

Playback Control Bar / Panel は表示しない。

表示するもの:

- Loading Spinner
- File Name
- Loading 状態

内部的には Playback 操作を Controller 側で拒否する。

### 4.5 Blocking Error（Desktop）

Playback Control Bar / Panel は表示しない。

表示:

- Error 状態
- ユーザー向け簡潔な Error Message
- Select Another File

### 4.6 Non-blocking Error（Desktop）

現在の Video / Audio UI をそのまま維持する。

その上に Notification / Banner を表示する。

現在 Media の Control Bar / Panel を消したり、Blocking Error 画面へ切り替えたりしない。

## 5. Unselected（共通）

主役: **Select File**

- 大量の Disabled Controls を表示しない
- Navigation を表示しない
- OS File Picker を起動する
- File Picker のキャンセルは Error ではない
- Drag & Drop は Phase 2 対象外
- Playback Control Bar / Panel は表示しない（第 4.3 節）

## 6. Loading（共通）

表示:

- Spinner
- File Name
- Loading 状態

Play / Pause / Stop / Seek は操作不可とする。

UI 側だけでなく Controller 側でも操作を拒否する。

Playback Control Bar / Panel は表示しない（第 4.4 節）。

## 7. Video

Video を画面の主役とする。

### Desktop

第 4.1 節の Floating Control Bar を用いる。

### Mobile

Desktop Floating Control Bar をそのまま縮小しない。

縦方向構成を基本とする:

```text
Video Area
  ↓
File Name
  ↓
Seek / Time
  ↓
Playback Controls
  ↓
Volume
  ↓
Select Another File
```

## 8. Audio

空の Video Box を表示しない。

表示:

- Static Audio Placeholder
- File Name
- Audio（種別の明示）

### Desktop

第 4.2 節の固定 Control Panel を用いる。

### Mobile

Desktop を単純縮小しない。Video と同様の縦方向構成を基本とする:

```text
Audio Area（Static Audio Placeholder）
  ↓
File Name
  ↓
Seek / Time
  ↓
Playback Controls
  ↓
Volume
  ↓
Select Another File
```

Phase 2 では次を表示しない。

- Artist / Album
- Codec / Bitrate / Resolution
- その他未取得 Metadata

実際の音声へ同期していない Fake Dynamic Visualizer は禁止する。

## 9. Error UI

### Blocking Error

現在の Media を再生できない場合。

- 例文言: `Could not load this file.`
- 操作: Select Another File
- Playback Control Bar / Panel は表示しない
- 技術的 Exception は表示しない
- 原因が確定していない場合、`Unsupported format` / `File corrupted` などを断定しない
- 色: Semantic Error（Red）

### Non-blocking Error

現在 Media を維持可能な場合、Media 画面を Error 画面へ置き換えない。

- 現在の Video / Audio UI（Control Bar / Panel を含む）を維持する
- その上に Notification / Banner を表示する
- 色: Semantic Warning（Amber）

### Unknown Error

将来的に問題報告機能へ接続予定とする。

Phase 2 では Report ボタンを実装しない。報告方法・送信情報等は将来別途設計する。

## 10. Windows

Phase 2 では **OS 標準 Window Chrome** を使用する前提とする。

アプリ独自の Minimize / Maximize / Close を UI Component として作らない。

## 11. Android / iOS

Desktop 用 Window Control を表示しない。

Mobile 向けに Layout を最適化する（第 7・8 節）。Desktop Floating Control Bar をそのまま縮小しない。

## 12. Play / Pause の Primary Control

同一の Primary Control を状態によって切り替える。

| 状態 | 表示 |
|------|------|
| `ready` / `paused` / `stopped` | Play |
| `playing` | Pause |

Pause 時は Position を保持する（[media-playback.md](media-playback.md)）。

## 13. Repeat の見た目

- Phase 2 は OFF / ONE のみ
- Repeat ONE Active は Accent Color で示す
- Repeat ALL は将来機能であり Phase 2 UI に出さない

## 14. 将来 Mode との境界

Design System は Player / Editor / Timeline 等へ拡張可能とするが、Phase 2 UI に次を先行表示しない。

- Editor Navigation
- Playlist / Library / Settings
- Timeline / Equalizer / Compressor 等の未実装 Mode

## 15. 未確定事項

- Volume UI の具体的コントロール形状（スライダー等）
- Static Audio Placeholder の具体ビジュアル
- Banner / Notification の正確な配置・消失条件
- Desktop / Mobile の Breakpoint 値
