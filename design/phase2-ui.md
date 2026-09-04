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

### Localization（Phase A）

| 項目 | 状態 |
|------|------|
| Localization 基盤 / 日本語標準表示 / `en` resource | 実装済み（[localization.md](localization.md)） |
| Settings 言語切替 / Locale 永続化 / システム言語自動追徰 | 未実装 |

§4.1 の Windows 固定 Bottom Panel は **実装済み**（client `68ff1b4`。[localization.md](localization.md) §3）。

## 2. Phase 2 に存在する Playback UI 操作

| 操作 / 表示 | 備考 |
|-------------|------|
| Select File | 未選択時の主役 |
| Select Another File | メディア選択後の別ファイル選択 |
| Play / Pause | 同一 Primary Control を状態で切替 |
| Stop | 位置を 00:00 へ戻す（詳細は media-playback.md） |
| Seek | 任意位置へ移動 |
| Current Time / Total Duration | Timecode（通常 `MM:SS`、1 時間以上 `HH:MM:SS`） |
| Volume | アプリ内再生音量（Slider + 整数 % 表示 + アイコン Mute / Unmute） |
| Repeat OFF | 終端後は stopped（00:00） |
| Repeat ONE | 現在 Media を繰り返す。Active は Accent |

これ以外の Playback 操作を Phase 2 UI へ追加しない。

### Phase 2 で禁止する操作（例）

- Previous / Next / Shuffle / Playlist / Settings
- Fullscreen / Subtitle / 10 秒移動
- Repeat ALL
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

## 4. Desktop Playback Controls

### 4.1 Windows（Implemented — client `68ff1b4`）

Windows では Audio / Video 共通で、Playback Controls を **`Scaffold.bottomNavigationBar` 固定 Bottom Panel**（`DesktopMediaBottomPanel`）へ配置する。

| 項目 | 仕様 |
|------|------|
| 配置 | `MediaScreen` → `Scaffold.bottomNavigationBar` |
| 対象 | Windows かつ `ready` / `playing` / `paused` / `stopped` かつ Audio / Video |
| narrow 幅 | **Mobile layout へ切り替えない**（Windows 専用 body + Bottom Panel を維持） |
| Media 領域 | Bottom Panel を除く残り領域で Audio / Video を **中央配置**（`_WindowsMediaBody`） |
| 横スクロール | **なし** |

**Collapse / Expand:**

| 項目 | 仕様 |
|------|------|
| 状態 | Panel 内 presentation-only state（`DesktopMediaBottomPanel`） |
| compact landscape | **初期 collapsed** |
| 非 landscape compact | 初期 expanded |
| collapse icon | `▼`（`keyboard_arrow_down_rounded`） |
| expand icon | `▲`（`keyboard_arrow_up_rounded`） |
| 可視ラベル | 「しまう」「戻す」等の **文字ボタンは表示しない** |
| Tooltip（ja） | しまう / 戻す |
| Tooltip（en） | Hide / Restore |
| Semantics | Tooltip 文言を label として維持（ARB: `collapsePlaybackControls` / `expandPlaybackControls`） |

**compact landscape レイアウト:**

| 条件 | Control Bar |
|------|-------------|
| compact landscape + expanded + 幅 ≥ 720px | **1 段**（`_SingleRowCompactBar`） |
| compact landscape + expanded + 幅 < 720px | **最大 2 段** fallback（`_TwoRowCompactBar`） |
| compact 時 Select Another File | **folder icon**（`Icons.folder_open_rounded`）。Tooltip / Semantics は `selectAnotherFile` |

Breakpoint 正本: [design-system.md](design-system.md)（desktop 800px、compact landscape `height ≤ 480` かつ `width > height`、single-row min width 720px）。

**Tooltip — 現在:** client `dccf48f` で `TooltipPolicy` / `NainaiTooltip` **Implemented**（I-3C）。`showTooltips` ON/OFF は visual のみ。正本 [settings.md](settings.md) §6 / [settings-shell.md](settings-shell.md) §19.3。本書では重複定義しない。

**検証（client `68ff1b4`）:** `flutter analyze` PASS / `flutter test` 214 tests PASS / Windows 実機確認 OK。

**Android / iOS:** narrow 含め **既存 Mobile layout を維持**（本変更の対象外）。

### 4.2 Non-Windows Desktop（設計 — 未統一）

macOS / Linux 将来 Desktop 向けの参考設計。Windows では §4.1 を正とする。

**Video:** 動画領域下部に Floating Control Bar（restrained glass、常時表示）。

**Audio（wide Desktop）:** 画面下部固定 Control Panel。

Floating / 固定 Panel に含む操作は第 2 節と同範囲。

Phase 2 ではまだ実装しない（将来改善）: auto-hide、Hover のみ表示、Cursor 移動制御。

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

- **Windows:** §4.1 固定 Bottom Panel。Video は Bottom Panel 上の残り領域に中央配置（`BoxFit.contain` 相当の fit）
- **Non-Windows Desktop:** §4.2 Floating Control Bar（設計）

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

- **Windows:** §4.1。Media 領域中央に Static Audio Placeholder + File Name 等
- **Non-Windows Desktop:** §4.2 固定 Control Panel（設計）

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

UI 文言は [localization.md](localization.md) の ARB 正本に従う（標準表示: 日本語）。

### Blocking Error

- 例（ja）: 「ファイルを読み込めませんでした。」（ARB `loadFailed`）
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
- Repeat ONE の日本語: **「この曲を繰り返し再生」**（[localization.md](localization.md)）
- Repeat ALL は Phase 2 UI に出さない

## 12. Volume UI（Phase 2-5 確定）

Volume コントロールは Slider + 整数パーセント表示 + Volume アイコンで構成する。

| 要素 | 内容 |
|------|------|
| Slider | `MediaState.volume`（`0.0`～`1.0`）を操作する |
| パーセント表示 | Slider 近傍に整数 %（例: `0%` / `50%` / `100%`）を表示する。Volume 専用の表示状態は持たない |
| Volume アイコン | Volume > 0 時は Volume アイコン、Volume = 0 時は Volume Off アイコン。click で Mute / Unmute（機能仕様は [media-playback.md](media-playback.md) §9） |

## 13. 将来 Mode との境界

Phase 2 UI に次を先行表示しない。

- Editor Navigation / Playlist / Library
- Settings Screen 本体（Display / Audio Section — **Implemented** I-3。Launcher は §14）
- Timeline / Equalizer / Compressor 等

## 14. Settings Launcher（App-level — Implemented）

正本: [settings.md](settings.md) §15。**Settings Launcher implementation:** **Implemented**（`Icons.settings_rounded`）。Phase I-3 結果: [settings-shell.md](settings-shell.md) §19。

### 14.1 配置（確定）

| 項目 | 内容 |
|------|------|
| 位置 | **App-level Top-right Utility Button**（SafeArea 内右上） |
| アイコン | `Icons.settings_rounded` 相当（第一候補） |
| 所属 | Media content **外側** の App Utility Layer。**Bottom Panel / Playback Controls の一部ではない** |

```text
Scaffold
├ body → Stack
│   ├ Main Content
│   └ Top-right App Utility Layer → Settings IconButton
└ bottomNavigationBar → DesktopMediaBottomPanel（Media 状態に応じて）
```

常設 AppBar（44〜56px 縦消費）は **採用しない**。App Utility Layer は Media を **大きく覆う常設 Overlay** に **しない**（§14.6）。

### 14.2 Bottom Panel との分離

[§4.1](#41-windows-固定-bottom-panel--implemented) の `DesktopMediaBottomPanel` 内へ Settings Launcher を **配置しない**。Settings = App level / Playback = Media level。

### 14.3 全状態でアクセス可能

| 条件 | Launcher |
|------|----------|
| Media: Unselected / Loading / Ready / Playing / Paused / Stopped / Blocking Error / Non-blocking Error | **常に利用可能** |
| Bottom Panel: expanded / collapsed / 非表示 | **常に利用可能** |

Unselected / Blocking Error でも Error UI **内部** へ Launcher を埋め込まない。

### 14.4 compact landscape

844×390 / 800×400 / 667×375 等でも右上 Launcher を維持。compact だから Bottom Panel へ移動 **しない**。§4.1 の 1 段 / 2 段 fallback と **独立**（Panel を **3 段化しない**）。

### 14.5 Mobile（Android / iOS）

App-level 右上 Settings を基本。SafeArea / system inset 必須。Windows Bottom Panel パターンを Mobile へ **持ち込まない**（§7・§8 参照）。

### 14.6 App Utility Layer と Media との重なり

| 条件 | 内容 |
|------|------|
| 視覚領域 | Launcher は **必要最小限** |
| Hit target | **44 logical px 以上** |
| 配置 | SafeArea 内右上、**既存 Spacing Token 内** |
| Overlay | Media content を **大きく覆う常設 Overlay にしない** |
| Video | **広い範囲を覆わない** |
| 競合 | Playback Controls、Non-blocking Error Banner 等と **競合しない** |
| compact landscape | Media 表示領域を **著しく奪わない** |
| 禁止 | **44〜56px 全幅 AppBar**、**大きな常設背景 Panel** |

### 14.7 Utility Button surface

Video 上に Launcher が位置する場合でも視認できるよう、既存 Theme の **surface token** による **小さい utility surface**（IconButton + 必要最小限 background）を **許可**。**独自色は禁止**。

### 14.8 Media 中央配置との関係

client `68ff1b4` で確定した、Audio / Video を **Bottom Panel 除く領域で中央配置** する基本仕様を、Launcher 追加のために **不用意に変更しない**。Launcher は App Utility であり Media レイアウト再設計の理由に **しない**。重なり問題時は utility inset / minimal padding で **局所調整**。**Media 全体を大きく下へ押し下げる設計は禁止**。

### 14.9 視覚・A11y / Tooltip

- SafeArea + Spacing Token、hit target **44 logical px 以上**
- **現在（client `dccf48f`）:** `showTooltips` で Visual Tooltip ON/OFF（`TooltipPolicy` / `NainaiTooltip` — I-3C）
- Tooltip / Semantics label は 設定 / Settings（ARB `settings`）。`showTooltips == false` 時は Visual Tooltip のみ非表示、Semantics **維持**（[settings.md](settings.md) §6.3）

### 14.10 Navigation

Activate → **`openSettings()`** → Common Settings Shell（Display + Audio）。**Gear → `AudioOutputSettingsSection` 直結禁止**。

## 15. Phase 2 baseline / Pending Acceptance

| 項目 | 状態 |
|------|------|
| Automated baseline（client `dccf48f`） | **526 tests PASS** / analyze PASS / analyze --no-pub PASS / diff-check PASS |
| Settings Shell / Tooltip / Audio Output wiring | **Implemented**（I-3 — [settings-shell.md](settings-shell.md) §19） |
| 実機 Acceptance（Windows / Android / iOS / Bluetooth / iOS compile） | **Pending**（[settings-shell.md](settings-shell.md) §19.5） |

## 16. 未確定事項

- Static Audio Placeholder の具体ビジュアル
- Banner の自動消失時間・clear 専用 UI / API
- 正式 Font Asset 導入後の見た目調整
