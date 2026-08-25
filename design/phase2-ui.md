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

Layout breakpoint（800 logical pixels）等の Token は [design-system.md](design-system.md) を正とする。

## 2. Phase 2 に存在する Playback UI 操作

| 操作 / 表示 | 備考 |
|-------------|------|
| Select File | 未選択時の主役 |
| Select Another File | メディア選択後の別ファイル選択 |
| Play / Pause | 同一 Primary Control を状態で切替 |
| Stop | 位置を 00:00 へ戻す（詳細は media-playback.md） |
| Seek | 任意位置へ移動 |
| Current Time / Total Duration | Timecode（通常 `MM:SS`、1 時間以上 `HH:MM:SS`） |
| Volume | アプリ内再生音量 |
| Repeat OFF | 終端後は stopped（00:00） |
| Repeat ONE | 現在 Media を繰り返す。Active は Accent |

これ以外の Playback 操作を Phase 2 UI へ追加しない。

### Phase 2 で禁止する操作（例）

- Previous / Next / Shuffle / Playlist / Settings
- Fullscreen / Subtitle / 10 秒移動
- Repeat ALL
- Mute（Phase 2 必須ではない）
- Fake metadata / Fake waveform / Dynamic Visualizer

## 3. 画面は Media State から描画する

最低限の状態: `unselected` / `loading` / `ready` / `playing` / `paused` / `stopped` / `error`

| 状態 | UI |
|------|-----|
| `unselected` | Empty State + Select File。Playback controls なし |
| `loading` | Progress + File Name。Playback controls なし |
| `ready` / `playing` / `paused` / `stopped` | Media + controls |
| Blocking `error` | Error + Select Another File。Playback controls なし |
| Non-blocking error | 現在 Media UI を維持 + Banner / notification |

Audio / Video 種別は別途 Media 情報として保持し、UI は状態と種別から描画する。

状態の意味と遷移は [media-playback.md](media-playback.md) を正とする。

## 4. Desktop Control Bar（確定）

Desktop では Media 種別と状態に応じて、次の Control 構造を採用する。

**Video は Floating Control Bar、Audio は画面下部の固定 Control Panel** とする。

### 4.1 Video（Desktop）

動画領域下部に **Floating Control Bar** を配置する。

- Control Bar は動画 Content を主役とし、動画全体を大きく確保したまま操作できる構造にする
- Phase 2 では Control Bar を **常時表示**する（auto-hide なし）
- restrained glass 表現を用いる

Phase 2 ではまだ実装しない（将来改善）:

- 一定時間後の自動非表示
- Mouse Hover のみで表示
- Cursor 移動による表示制御

Floating Control Bar に含む Phase 2 操作:

- Current Time / Seek / Total Duration
- Play / Pause / Stop
- Repeat OFF / ONE
- Volume
- Select Another File

Phase 2 対象外の操作を追加しない。

### 4.2 Audio（Desktop）

画面下部に **固定 Control Panel** を配置する。

Video のように Media へ Overlay する必要はない。

Fake waveform / visualizer は置かない。

Control Panel に含む操作は Video Floating Control Bar と同範囲（第 2 節）。

### 4.3 Unselected / Loading / Blocking Error（Desktop）

Playback Control Bar / Panel を表示しない。

| 状態 | 表示 |
|------|------|
| Unselected | Select File を主役とした Empty State |
| Loading | Loading Progress + File Name |
| Blocking Error | Error + Select Another File |

大量の Disabled Control は表示しない。

### 4.4 Non-blocking Error（Desktop）

現在の Video / Audio UI をそのまま維持する。

その上に Notification / Banner を表示する。

現在 Media の Control Bar / Panel を消したり、Blocking Error 画面へ切り替えたりしない。

## 5. Banner（Non-blocking）

Phase 2 現在:

- Controller の `error` が存在する間表示する **state-driven banner**
- 自動消失時間は **未定**（3 秒 / 5 秒等を勝手に確定しない）
- clear 専用 UI / API も未確定

色は Semantic Warning（[design-system.md](design-system.md)）。

## 6. Unselected / Loading（共通）

Unselected:

- 主役は **Select File**
- Navigation なし、Playback controls なし
- OS File Picker。Cancel は Error ではない
- Drag & Drop は Phase 2 対象外

Loading:

- Progress + File Name
- Play / Pause / Stop / Seek は操作不可（UI + Controller）
- Playback controls なし

## 7. Video

Video を画面の主役とする。

### Video Fit

内容全体を確認可能な fit を基本とする。Phase 2 実装では **`BoxFit.contain`**。装飾目的の crop を行わない。

### Desktop

第 4.1 節の Floating Control Bar（常時表示・restrained glass）を用いる。

### Mobile

Desktop Floating Control Bar をそのまま縮小しない。縦方向構成:

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
- File Name（ユーザーから確認可能であること。Media Area 内に出す場合、同名の二重表示は必須ではない。「Audio では filename 不要」とはしない）
- Audio（種別の明示）

### Desktop

第 4.2 節の固定 Control Panel を用いる。

### Mobile

Desktop を単純縮小しない。縦方向構成:

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

- Artist / Album / Codec / Bitrate / Resolution
- その他未取得 Metadata
- Fake Dynamic Visualizer

## 9. Error UI

### Blocking Error

- 例文言: `Could not load this file.`
- Select Another File
- Playback controls なし
- 技術的 Exception は表示しない
- 原因未確定時に `Unsupported format` / `File corrupted` を断定しない
- 色: Semantic Error

### Non-blocking Error

- 現在 Media UI を維持 + Banner（第 5 節）
- 色: Semantic Warning

### Unknown Error

将来の問題報告機能へ接続予定。Phase 2 では Report ボタンを実装しない。

## 10. Windows / Mobile Chrome

- Windows: OS 標準 Window Chrome。独自 Minimize / Maximize / Close を作らない
- Android / iOS: Desktop 用 Window Control を出さない。Mobile 向け縦構成（第 7・8 節）

## 11. Play / Pause / Repeat の見た目

| 状態 | Primary Control |
|------|-----------------|
| `ready` / `paused` / `stopped` | Play |
| `playing` | Pause |

- Repeat ONE Active は Accent（`accent-primary`）
- Repeat ALL は Phase 2 UI に出さない

## 12. 将来 Mode との境界

Phase 2 UI に次を先行表示しない。

- Editor Navigation / Playlist / Library / Settings
- Timeline / Equalizer / Compressor 等

## 13. 未確定事項

- Volume UI の具体的コントロール形状（スライダー等）の細部
- Static Audio Placeholder の具体ビジュアル
- Banner の自動消失時間・clear 専用 UI / API
- 正式 Font Asset 導入後の見た目調整
