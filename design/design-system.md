# Design System

nainai の Design System を定義します。色・タイポグラフィ・余白・形状などの視覚ルールと、テーマ責務の分離方針を扱います。

画面構成・状態別レイアウトは [phase2-ui.md](phase2-ui.md) を参照してください。再生操作の機能仕様は [media-playback.md](media-playback.md) を参照してください。

関連:

- [Phase 2 UI](phase2-ui.md)
- [メディア再生](media-playback.md)
- [メディアファイル選択](media-selection.md)
- [MVP 要件](../requirements/mvp-requirements.md)
- [プロダクト概要](../requirements/product-overview.md)

## 1. アプリ名称

正式名称は **nainai** とする。

「nainai Media Player」を製品正式名称として使用しない。

nainai は将来、Player / Editor / Timeline / Playlist / Equalizer / Compressor 等へ拡張する。アプリ全体を Media Player に限定しない。

## 2. Design Persona

採用ペルソナ: **Professional Creative Media Tool**

キーワード:

- Content First
- Minimal
- Precise
- Professional
- Creative
- Media-focused
- Distraction-free

「Modern Corporate」という表現は使用しない。

## 3. Stitch 成果物の扱い

### 3.1 正本性

Google Stitch による Phase 2 UI の大枠設計は **視覚参考資料** とする。

**仕様の正本は nainai-docs である。** Stitch 成果物そのものは仕様の正本ではない。

Flutter 実装時に仕様と Stitch HTML が矛盾した場合は、**必ず nainai-docs を優先**する。

Stitch HTML をそのまま Flutter へ移植しない。HTML 内の未確定・旧仕様は採用しない。

HTML は次の参考にのみ用いる。

- レイアウト
- 情報階層
- 余白
- コンポーネント配置
- 色調
- Desktop / Mobile の違い

### 3.2 保存場所と版管理

Google Stitch 成果物の保存先:

```text
D:\fyna\dev\nainai\google_stitch
```

新しい確定版ができるたびに、既存版を上書きせず番号を増やす。

```text
D:\fyna\dev\nainai\google_stitch\
├── stitch1
├── stitch2
├── stitch3
└── ...
```

各 `stitchN` は、その時点で確定した Stitch 参考成果物の **Snapshot** として扱う。

### 3.3 現在の参照版

現時点では **`stitch1`** を Phase 2 UI 実装時の視覚参考版とする。

```text
D:\fyna\dev\nainai\google_stitch\stitch1
```

将来 `stitch2` 等が正式確定した場合は、その時点の最新確定版を参照する。

単に番号が大きいだけではなく、**確定版として採用されたもの** を最新とする。

### 3.4 Phase 2 で禁止する旧 Stitch 要素（例）

Stitch 出力に残っていても Phase 2 では採用しない。

- 製品名としての「nainai Media Player」
- Previous / Next / Shuffle / Playlist / Library / Settings
- Subtitle / Fullscreen
- 10 秒戻し / 10 秒送り
- Drag & Drop
- Dynamic Audio Visualizer（実音声と非同期の疑似表現を含む）
- Artist 等の未取得 Metadata
- Windows 独自 Window Chrome
- Mobile の Close / Minimize 等（Desktop 用 Window Control）
- Editor Navigation
- Repeat ALL

## 4. Theme Architecture

色は最低限、次の 3 責務へ分離する。

| 責務 | 役割 |
|------|------|
| Base Palette | 背景・Surface・Container などの基調色 |
| Accent Palette | Primary Action / Selected / Active / Seek Progress / Focus / Repeat ONE Active 等 |
| Semantic Palette | Error / Warning / Success 等の意味色（Accent と独立） |

将来、ユーザーが **Base Palette** と **Accent Color** を独立して選択できるようにする。

現時点ではテーマ選択 UI 自体は作らない。

Color 値を Widget へ直接散在させない。Theme / Design Token を介して使用する。

### 4.1 Base Palette

背景・Surface・Container などの基調色。

Phase 2 初期プリセット: **Midnight Navy**

現在の Dark Navy 系デザインは、永久固定のブランドカラーではなく **初期 Preset** として扱う。

### 4.2 Accent Palette

次に使用する。

- Primary Action
- Selected
- Active
- Seek Progress
- Focus
- Repeat ONE Active

初期 Accent: **Lavender 系**

将来、Purple / Blue / Cyan / Green / Orange などへ変更可能にする。

### 4.3 Semantic Palette

Accent とは独立させる。

最低限:

- Error
- Warning
- Success

ユーザーが Accent を変更しても、Error / Warning の意味色が変化してはいけない。

Phase 2 のエラー表示色:

| 種別 | 色の方針 |
|------|----------|
| Blocking Error | Red（Semantic Error） |
| Non-blocking Error | Amber（Semantic Warning） |

## 5. 初期 Midnight Navy Design Token

現在の基準値（Base）:

| Token | 値 |
|-------|-----|
| `background` | `#0b1326` |
| `surface` | `#0b1326` |
| `surface-container-lowest` | `#060e20` |
| `surface-container-low` | `#131b2e` |
| `surface-container` | `#171f33` |
| `surface-container-high` | `#222a3d` |
| `surface-container-highest` | `#2d3449` |
| `on-surface` | `#dbe2fd` 相当 |
| `on-surface-variant` | `#c7c5d0` 相当 |

初期 Accent は Lavender 系とする（具体値は Theme / Design Token で管理する）。

`#020617` など、旧 Stitch 成果物だけに存在する別背景色を、新たな正式 Token として採用しない。

## 6. Typography

| 用途 | 方針 |
|------|------|
| 通常 UI | Inter |
| 時間表示（Current Time / Total Duration / Timecode） | Geist Mono 相当の Monospace |

時間表示の対象は次を含む。いずれも Geist Mono 相当の Monospace とする。

- Current Time
- Total Duration
- 将来の Timecode

時間表示は、数字が変化しても横幅が揺れないこと。

Flutter 実装時に、Web 用 Google Fonts コードをそのままコピーしない。フォント導入方法は Flutter 実装段階で別途決定する。

## 7. Spacing

**4px baseline** を基本とする。

| Token | 値（px） |
|-------|----------|
| `xs` | 4 |
| `sm` | 8 |
| `md` | 16 |
| `lg` | 24 |
| `xl` | 40 |

具体値は Design Token として管理し、Widget 内へマジックナンバーを散在させない。

Mobile の操作領域は最低 **44×44** を基準とする。

## 8. Shape

| 対象 | 方針 |
|------|------|
| Standard Button / Input | 4px |
| Media Container | 8px |
| Card / Overlay | 最大 12px 程度 |
| Seek Thumb | Circle |
| Play / Pause 等の Circular Icon Button | Circle |

通常 CTA を無条件で Pill 型にしない。

## 9. マジックナンバー方針

実装時に、次を意味なく Widget 内へ直接散在させない。

- Color
- Spacing
- Radius
- Animation Duration
- Control Size
- Breakpoint
- Playback 関連閾値

責務ごとの Theme / Design Token / Constants へ集約する。

ただし、定数を 1 ファイルへ無秩序に全部集めない。

## 10. 将来拡張

Design System は将来的に次へ拡張可能にする。

- Player Mode
- Editor Mode
- Timeline
- Playlist
- Folder Playback
- Device Media
- Equalizer
- Compressor

ただし Phase 2 UI へ未実装機能を先行表示しない。

## 11. 未確定事項

- 初期 Lavender Accent の正式な Color Token 一覧（色相・段階）
- フォントの Flutter への導入方法・ライセンス確認手順
- テーマ選択 UI（将来）
- Base / Accent の追加 Preset 一覧
- Animation Duration / Breakpoint の具体 Token 値
