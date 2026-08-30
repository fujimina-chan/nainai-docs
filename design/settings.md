# アプリ共通 Settings 基盤（詳細設計）

nainai アプリ全体の Settings Presentation Entry Point、共通 preference 基盤、および第一設定項目 **Tooltip 表示 ON / OFF** の設計正本。

関連:

- [Windows 音声出力 Settings UI（Phase C）](audio-output-settings.md)
- [音声出力デバイス選択](audio-output.md)
- [UI Localization](localization.md)
- [Design System](design-system.md)
- [Phase 2 UI](phase2-ui.md)
- [Application Composition](../architecture/media-technology.md)

## 1. 位置づけ

| 項目 | 内容 |
|------|------|
| スコープ | Settings Shell / General Section / `AppSettings` / `SettingsController` / Persistence 抽象 / Tooltip Policy |
| 第一設定項目 | `showTooltips`（Tooltip 表示 ON / OFF） |
| Audio Output との関係 | **置き換えではない**。Audio Output Phase C は [audio-output-settings.md](audio-output-settings.md) が正本 |
| 状態 | **Core Design Complete** / **Concrete Persistence Pending** / **未実装** |

**Core Design Complete の範囲:** Settings 責務 / 画面構造 / Section 構造 / Model / Controller / Persistence Interface / **async consistency（revision / serialized save）** / Tooltip 仕様 / Accessibility / Localization キー設計 / Application Composition / ownership / テスト方針 / Audio Output 境界 / Launcher Navigation contract。

**Concrete Persistence Pending:** 永続化 concrete 実装（package 選定・storage key 実装詳細）は client dependencies と全 MVP プラットフォーム確認後、実装 Phase 開始時に最終決定する（§5.3）。**Persistence Interface と async consistency 仕様（§4.4 / §12）は本設計で確定。**

**未確定（Launcher のみ）:** Settings Launcher 物理配置 — **Launcher placement design pending**（§15）。Navigation contract / Settings Shell / 「Settings を開く機能が必要」までは本設計で確定。物理配置の具体位置は [audio-output-settings.md](audio-output-settings.md) §14 と連携して決定する。

## 2. Settings 全体構造

Settings は **機能ごとに Settings 画面を乱立させない** アプリ共通 Entry Point とする。

```text
Settings（Shell）
├ General
│   └ Show Tooltips          ← 本設計（第一共通設定）
│
└ Audio
    └ Audio Output           ← Phase C（audio-output-settings.md）
```

将来追加候補（**現時点では未仕様・勝手に確定しない**）:

- Appearance
- Language（Locale 切替 — [localization.md](localization.md) と連携予定）
- Playback
- Export

### 2.1 責務分離（Common vs Feature-specific）

| 領域 | 担当 | 正本 |
|------|------|------|
| Settings Shell / Navigation / Section レイアウト | 共通 Settings | 本書 |
| `AppSettings` / 共通 preference | 共通 Settings | 本書 |
| `SettingsController` | 共通 Settings | 本書 |
| Settings Persistence 抽象（共通 preference 用） | 共通 Settings | 本書 |
| Tooltip Policy / Visual Tooltip 制御 | 共通 Settings | 本書 |
| `AudioOutputController` / `AudioOutputState` | Audio Output | [audio-output-settings.md](audio-output-settings.md) |
| Device 一覧 / Selection / hot unplug / Audio エラー | Audio Output | [audio-output.md](audio-output.md) |
| `AudioOutputPreference` 永続化（Phase G） | Audio Output | [audio-output-settings.md](audio-output-settings.md) §19 |

**禁止:** `SettingsController` が `AudioOutputController` の device selection / enumeration / Service 操作を吸収すること。`AudioOutputPreference` を `AppSettings` へ直接埋め込むこと（Phase G でも独立 Domain モデルを維持）。

## 3. AppSettings model

共通設定値を表す **immutable** model。

### 3.1 正式名称

**`AppSettings`**

### 3.2 初期フィールド（本設計で確定）

| フィールド | 型 | デフォルト | 説明 |
|------------|-----|------------|------|
| `showTooltips` | `bool` | **`true`** | Visual Tooltip 表示 ON / OFF |

**`showTooltips` デフォルト `true` を正式確定する理由:** 初回利用時の UI discoverability（Icon 説明）を維持するため。

### 3.3 拡張方針

- 将来の共通設定（Appearance / Language 等）は `AppSettings` へ field 追加で拡張可能
- `copyWith` / equality を採用し、部分更新とテスト容易性を確保する
- **機能固有 Preference**（例: `AudioOutputPreference`）は **別 Domain モデル** のまま維持。`AppSettings` への統合は行わない

### 3.4 責務境界（AppSettings vs AudioOutputPreference）

| モデル | 永続化 Phase | 管理 Controller | 用途 |
|--------|--------------|-----------------|------|
| `AppSettings` | 本設計（Tooltip 実装 Phase） | `SettingsController` | アプリ横断 UI  preference |
| `AudioOutputPreference` | Phase G（未設計詳細） | `AudioOutputController`（将来 Repository 経由） | 音声出力 device 選択 |

## 4. SettingsController

Presentation から Persistence 実装を **直接触らせない** Application 層 Controller。

### 4.1 責務

| 担当 | 内容 |
|------|------|
| State 保持 | 現在の `AppSettings` を保持 |
| 起動時読込 | `SettingsRepository.load()` を非同期 invoke |
| 変更 API | `setShowTooltips(bool value)` 等 |
| 通知 | `ChangeNotifier` により UI / Tooltip Policy へ通知 |
| 保存 | 変更後 `SettingsRepository.save(AppSettings)` を **直列化** して invoke（§4.4） |
| 永続化順序 | monotonic revision / generation により **latest user intent wins**（§4.4） |
| スナップショット | `lastPersistedSettings` を保持（§4.4 / §12） |
| エラー | Settings 内 Non-blocking error（Playback を止めない） |

**担当しない:** Audio Output device 操作、`MediaController` state 更新、Playback Blocking Error、Repository の concrete 実装詳細。

### 4.2 推奨 API（設計）

```dart
class SettingsController extends ChangeNotifier {
  SettingsController({required SettingsRepository repository});

  AppSettings get settings;           // 現在 UI が反映すべき desired state
  AppSettings get lastPersistedSettings; // 最後に永続化成功した snapshot（§4.4）
  SettingsError? get error;             // null = なし。Non-blocking
  bool get isLoading;                   // 起動時 initial load 進行中（§11 / §12.3）
  bool get isSaving;                    // persistence write 進行中（§12.3）

  Future<void> initialize(); // 起動時 load。§11 参照
  Future<void> setShowTooltips(bool value);

  @override
  void dispose();
}
```

外部 State Management package の新規導入を **前提にしない**。Composition Root から `ListenableBuilder` / `AnimatedBuilder` 等で `SettingsController` を注入する。

### 4.4 設定 mutation と永続化順序（確定）

SettingsController は **設定変更の永続化順序を管理する** 責務を持つ。

**正式方針: latest user intent wins**

| 原則 | 内容 |
|------|------|
| UI 即時更新 | ユーザー操作後、`settings`（desired state）を **即時** 更新し `notifyListeners` |
| 直列化 write | persistence write は Controller が **直列化** する。同一設定への write を無秩序に並列実行しない |
| 最終永続値 | 最終的な永続値は **最新ユーザー操作** と一致させる |
| stale 結果禁止 | 古い非同期 save / load 結果が新しい state を **上書きしてはならない** |

Controller 内部概念（実装 Phase）:

| 概念 | 説明 |
|------|------|
| `settings` | 現在 UI / Tooltip Policy が反映すべき **desired** state |
| `lastPersistedSettings` | 最後に **永続化成功** した snapshot |
| `_revision` | monotonic revision / generation。ユーザー mutation ごとに increment |

**ユーザー変更フロー:**

```text
revision++
settings = newSettings          // desired state 即時更新
notifyListeners()                 // UI / Tooltip 即時反映
_enqueueSerializedSave(revision, settings)
```

**Persistence（serialized）:**

```text
Controller が save を順序制御（Future chain / queue 等）
各 save 完了時、当該 revision が「最新 revision」でなければ:
  → UI state を変更しない（結果を discard）
当該 revision が最新かつ save 成功:
  → lastPersistedSettings = saved snapshot
```

実装 Phase では monotonic revision + serialized persistence、または **同等に安全な方法** を用いる。

**load 競合:** §11.4 参照。

### 4.5 SettingsController と AudioOutputController の関係

- **兄弟関係**（compose されるが、一方が他方の sub-controller ではない）
- `SettingsScreen` は両 Controller を **別々に** 参照する
- Phase C では `AudioOutputController` を先行実装し、共通 Shell 未実装でも Section 単体開発可能（[audio-output-settings.md](audio-output-settings.md) §14）

## 5. SettingsRepository（Persistence 抽象）

Controller 自身へ Persistence を直書きしない。

### 5.1 レイヤ配置

| 要素 | レイヤ | 配置（概念） |
|------|--------|--------------|
| `SettingsRepository`（abstract） | Domain / Application 境界 | `lib/settings/` または `lib/domain/settings/` |
| `AppSettings` | Domain | 同上 |
| Concrete implementation | Infrastructure | `lib/infrastructure/settings/` 等 |

### 5.2 推奨 API（設計）

```dart
abstract class SettingsRepository {
  /// 永続化値を読込。未保存・破損時は default `AppSettings` 相当を返す（§11）。
  Future<AppSettings> load();

  /// 保存。失敗時は throw または Result 型で Controller が Non-blocking error へ変換。
  Future<void> save(AppSettings settings);
}
```

設定単位 API（`loadShowTooltips()` 等）への分割は **初期実装では不要**。`AppSettings` 単位の load / save で十分。

### 5.3 Concrete 技術（Pending）

| 候補 | 概要 |
|------|------|
| A. 標準 key-value package（例: `shared_preferences` 等） | Flutter エコシステムで一般的。Windows / Android / iOS 対応要確認 |
| B. 独自 file persistence（例: JSON file in app support dir） | package 依存を減らせる。path / 権限 / 同期の実装コスト |

**本設計時点の確定:**

- **`SettingsRepository` interface は確定**
- **Concrete implementation は実装 Phase 開始時に最終決定**

実装 Phase 開始時に nainai-client の `pubspec.yaml` dependencies、Windows / Android / iOS 初期 MVP 対象での動作、既存 Infrastructure 方針を確認し、採用理由を docs へ追記する。**調査不足の段階で `shared_preferences` 等を無条件確定しない。**

### 5.4 Persistence key（概念）

| 概念 | 値 |
|------|-----|
| 設定フィールド | `showTooltips` |
| 保存 key（内部） | **`settings.showTooltips`**（namespace 付き。Localization key と混同しない） |
| 保存形式 | bool（concrete 実装依存。JSON 全体保存も可） |

将来 `AppSettings` 拡張時は `settings.*` namespace を維持する。大規模 migration framework は **現段階では作らない**。形式変更が必要になった場合は Repository 内で version / migration を段階的に追加する。

## 6. Tooltip 表示 ON / OFF（正式仕様）

### 6.1 設定名（確定）

| 項目 | 値 |
|------|-----|
| Model field | `showTooltips` |
| 型 | `bool` |
| デフォルト | **`true`** |

### 6.2 挙動（確定）

| 値 | Visual Tooltip |
|----|----------------|
| `true`（ON） | 現在どおり hover / long-press 等で Visual Tooltip を **表示** |
| `false`（OFF） | nainai 管理の Visual Tooltip を **表示しない** |

- 変更は **アプリ再起動不要** で **即時反映**（§10）
- 再起動後も永続化値を restore（§11）

### 6.3 Visual Tooltip と Accessibility Semantics の分離（必須）

**Tooltip OFF はアクセシビリティ情報の無効化ではない。**

OFF でも **維持する:**

- Semantics label
- button semantics
- selected state
- enabled / disabled state
- screen reader 向け情報

**禁止:** `showTooltips == false` のとき Semantics label を null / 空にすること。

例（概念）:

```text
現在:  IconButton(tooltip: 'しまう', ...)
OFF時: Visual Tooltip 非表示。Semantics label「しまう」は維持
```

### 6.4 適用範囲

`showTooltips` が制御する対象:

- nainai 自身が Presentation 上で提供する **補助 Visual Tooltip**

対象例（将来 `TooltipPolicy` 移行対象。client 固定 Bottom Panel は `68ff1b4` で **実装済み** — Tooltip Policy 実装 Phase で適用）:

- 「しまう」「戻す」（Bottom Panel 折りたたみ）
- 「別のファイルを選択」
- Repeat / Mute
- Settings Launcher icon
- その他 nainai 管理 IconButton 説明

**制御対象外（仕様上強制しない）:**

- OS ネイティブ UI
- file picker
- platform route picker（Android / iOS Audio Output）
- 外部ライブラリ内部 UI

### 6.5 Tooltip 実装方式（Presentation helper）

各 Widget が `showTooltips ? localizedText : null` を **バラバラに直書きする構造を避ける**。

| コンポーネント（第一候補） | 責務 |
|---------------------------|------|
| **`TooltipPolicy`** | `SettingsController`（または注入 `bool`）から Visual Tooltip 表示可否を判定 |
| **`NainaiTooltip`** | Visual Tooltip の有無を Policy に従い制御。**Semantics は常に維持** |
| **`NainaiIconButton`**（任意） | IconButton + Tooltip + Semantics の定形 wrapper |

**名称:** 実装 Phase 開始時に nainai-client 既存 Widget 命名規則（`MediaController` / `MediaKitVideoSurface` 等）を確認し最終決定する。本設計では上記を **第一候補** とする。

**TooltipPolicy 注入:** Composition Root または Settings  subtree 直上で `SettingsController` を listen し、Policy へ `showTooltips` を渡す。MediaScreen 等 App 全体 subtree でも Policy を参照できるよう、Application lifetime scope で提供する。

## 7. Settings UI

### 7.1 General Section

| Locale | Section title |
|--------|---------------|
| ja | 一般 |
| en | General |

Tooltip 行:

| 要素 | 内容 |
|------|------|
| Control | `Switch` または `SwitchListTile` 相当 |
| Touch target | **44 logical px 以上**（[design-system.md](design-system.md) 準拠。Desktop Settings にも適用） |
| 即時反映 | Switch 操作 → `setShowTooltips` → 全 App subtree へ伝播 |

構造概念:

```text
一般

ツールチップを表示                 [ ON ]
ボタンやアイコンにマウスを
合わせたときに説明を表示します
```

Visual Token は [design-system.md](design-system.md) 準拠。Settings Shell 共通 Background / Surface / Accent を Audio Section と共有する。

### 7.2 Audio Section

| Locale | Section title |
|--------|---------------|
| ja | オーディオ |
| en | Audio |

Section 内コンテンツは [audio-output-settings.md](audio-output-settings.md) の `AudioOutputSettingsSection` を配置する。

**Shell と platform-specific contents の分離:**

- **Settings Shell**（General / Audio Section 見出し、Navigation、共通 Layout）— 本書
- **Audio Output platform-specific contents** — Phase C（Windows device list）、Phase E / F（Mobile route picker）

Windows / Android / iOS で **Settings 内容構造（General + Audio）は共有** する。Audio Output Section の **中身** のみ platform capability に応じて異なる。

### 7.3 Desktop / Mobile Presentation

- **Settings 内容（Section / 項目）は共有**
- **Shell の Presentation 形態** は adaptive とする（実装 Phase で Flutter adaptive pattern を採用）
  - Mobile: フルスクリーン Route が第一候補
  - Desktop（Windows）: フルページ Route または Dialog / 固定幅 Panel — Launcher 配置確定後に visual を調整可
- Tooltip Switch UI 自体は **Windows / Mobile 共通**

## 8. Localization

[localization.md](localization.md) の gen-l10n / ARB 基盤を使用。**日本語直書き禁止。** 本設計時点では ARB 変更なし。

### 8.1 共通 Settings キー（camelCase — 本設計で確定）

| キー | ja | en |
|------|----|----|
| `settings` | 設定 | Settings |
| `general` | 一般 | General |
| `showTooltips` | ツールチップを表示 | Show tooltips |
| `showTooltipsDescription` | ボタンやアイコンにマウスを合わせたときに説明を表示します | Show descriptions when hovering over buttons and icons |
| `audio` | オーディオ | Audio |

Audio Output 向けキー（`audioOutput` 等）は [audio-output-settings.md](audio-output-settings.md) §16 が正本。実装 Phase で ARB 統合・重複排除する。

Settings 保存失敗等の Non-blocking メッセージキー（例: `settingsSaveFailed`）は実装 Phase で追加可。本設計では必須キーとして確定しない。

### 8.2 SettingsController と LocaleController

[localization.md](localization.md) の **Locale 切替用 `LocaleController` は未実装** であり、本書の **`SettingsController`（AppSettings 管理）とは別物** とする。Language Section 追加時に compose 関係を設計する。

## 9. Accessibility

### 9.1 General Section — Tooltip Switch

Switch は次を認識可能にする:

- label（`showTooltips` Localization）
- description（`showTooltipsDescription` — 実装時 Switch と関連付け）
- current checked state
- enabled state

### 9.2 Tooltip OFF 時の Settings / App 全体

- Settings 内 Semantics を削除しない
- Tooltip OFF でも Icon Control の Semantics label は維持（§6.3）
- **Windows:** Tab / Space 等による Keyboard 操作可能（Switch focus / toggle）
- Radio / device 行等 Audio Section の Accessibility は [audio-output-settings.md](audio-output-settings.md) §17 が正本

## 10. 設定変更の即時反映

| 要件 | 設計 |
|------|------|
| 再起動 | **不要** |
| Settings 閉鎖 | **不要**（閉じなくても反映） |
| 伝播先 | 既に表示中の MediaScreen 等 **App 全体 subtree** |
| 実装 | `SettingsController` の `notifyListeners` → `TooltipPolicy` / `ListenableBuilder` 経由で Visual Tooltip を rebuild |

`showTooltips` 変更は Playback state / Media 選択 state に **影響しない**。

## 11. 初期値・永続化・起動時 restore

### 11.1 初回状態

| 条件 | `showTooltips` |
|------|----------------|
| 保存値なし（初回起動） | **`true`** |
| 保存値あり | 保存値を使用 |
| 壊れた値 / 読込失敗 | **`true` へ fallback** + 必要に応じ Settings 内 Non-blocking error |

### 11.2 起動手順（第一候補 — flicker 最小化）

```text
1. NainaiApp 起動
2. SettingsController 生成。settings = AppSettings.defaults()  // showTooltips: true
3. runApp — 設定未読込でも default true で安全に開始
4. initialize() が非同期 load。成功時 `lastPersistedSettings = loaded`
5. load 完了後、revision guard を満たす場合のみ `settings` を loaded 値へ更新 + notifyListeners（§11.4）
```

**意図:** default が `true` のため、大多数の初回ユーザーは load 前後で Tooltip 表示が変わらない。`false` 保存ユーザーのみ load 完了後に Visual Tooltip が消える（許容。flicker は ON→OFF 方向のみ）。

アプリ全体を Settings load 完了まで **Blocking しない**。

### 11.4 load 競合（確定）

遅れて完了した load 結果が、load 開始後に行われたユーザー mutation を **上書きしてはならない**。

**推奨（第一候補）:** initial load 完了前は Settings 画面での mutation を開始させない（`isLoading == true` 中は General Section Switch を disabled 等）。App 全体の Blocking 画面は **不要**。Tooltip は default `true` で安全に開始。

**代替（revision guard）:** load 開始時の revision を記録し、load 完了時に **その revision 以降にユーザー mutation が存在する** 場合、load 結果で `settings` を上書きしない。`lastPersistedSettings` のみ load 結果で更新する等、実装 Phase でいずれかを採用。

いずれの方式でも **stale load result が desired state を巻き戻さない** こと。

### 11.5 永続化

- Tooltip ON / OFF は **再起動後も維持**
- Concrete storage は §5.3 Pending
- Audio Output Preference 永続化（Phase G）とは **別 Repository / 別 key namespace** を維持

## 12. Error policy

Settings 操作のエラーは **Settings 内 Non-blocking** とする。

| 禁止 | 理由 |
|------|------|
| Playback Blocking Error へ昇格 | Settings と Playback は独立 |
| Media 画面を Error 画面へ置換 | 同上 |
| 再生停止 | 同上 |

### 12.1 保存失敗時の UI state（確定）

`setShowTooltips` 等は **楽観的に UI を即時更新** する（§4.4）。save API 失敗時の扱いは **失敗した save の revision が最新か** で分岐する。

#### 最新 revision の save 失敗

失敗した save が **最新ユーザー操作** に対応し、それより新しい操作が **ない** 場合のみ:

| 項目 | 扱い |
|------|------|
| UI state | **`lastPersistedSettings` へ revert** |
| 通知 | Settings 内 Non-blocking error（例: 保存失敗メッセージ） |
| 方針 | **ユーザーに「保存された」と誤認させない** |

#### 古い revision の save 失敗

**より新しい revision のユーザー操作が既に存在する** 場合、古い save 失敗で **現在の UI state を revert してはならない**。error 表示も最新 revision の結果を上書きしない（discard または最新操作に紐づく error のみ）。

**競合例（禁止パターン）:**

```text
rev 1: true → false, save(false) 開始
rev 2: false → true, save(true) 成功 → lastPersistedSettings = true
rev 1: save(false) 遅延完了 → false を永続化   ← 禁止
rev 1: save(false) 失敗 → rev 2 の true を false へ revert   ← 禁止
```

**許可:** rev 1 の遅延完了は revision guard により discard。rev 1 の失敗も rev 2 より古いため UI 変更なし。

session 中のみ変更を維持し再起動で失われる、は **採用しない**（persist 失敗と成功の区別が曖昧になるため）。

### 12.2 読込失敗

- `AppSettings.defaults()`（`showTooltips: true`）で開始
- Non-blocking error を Settings 表示時または初回 load 失敗時に提示可
- Playback には影響しない

### 12.3 isLoading / isSaving（最小状態）

単一 bool を増やすことが目的ではない。UI が次を正しく表現するための **最小状態**:

| 状態 | 用途 |
|------|------|
| `isLoading` | 起動時 initial restore 中。Settings mutation 開始抑制（§11.4） |
| `isSaving` | persistence write 進行中（任意: Switch 二重操作抑制等） |
| `error` | 最新 relevant な save / load 失敗 |

保存中・読込中でも **アプリ全体を Blocking しない**。Media 再生は継続。

## 13. Application Composition

Application Composition の技術正本は [media-technology.md](../architecture/media-technology.md) §Application Composition。本節は **Settings 基盤追加** を定義する。

### 13.1 目標構成（Settings 実装 Phase — 設計確定・未実装）

```text
NainaiApp                          ← Composition Root
│
├── SettingsRepository             ← Infrastructure（Composition Root が生成）
├── SettingsController             ← AppSettings 正本
│
├── MediaKitPlaybackService        ← 1 instance（Phase C 以降 ownership は NainaiApp）
├── MediaController
├── AudioOutputController          ← Phase C（Audio Output 専用。SettingsController と独立）
│
├── FileSelectorMediaSelectionService
├── MediaScreen
└── SettingsScreen（または Settings Route）
      ├─ GeneralSettingsSection    ← showTooltips
      └─ AudioOutputSettingsSection ← Phase C
```

**`build()` ごとに `SettingsController` / `SettingsRepository` を生成しない。** Application lifetime で 1 組。

### 13.2 Phase C との compose 順序

Audio Output Phase C は共通 Shell 未実装でも Section 単体を Modal / 暫定 Route で開発可能（[audio-output-settings.md](audio-output-settings.md) §14）。

推奨実装順（docs 上の推奨。client レーン判断は別）:

1. `AppSettings` + `SettingsRepository` interface + `SettingsController` + Tooltip Policy
2. General Section（showTooltips）+ Settings Shell 最小構成
3. Audio Output Section を Shell へ統合（Phase C）
4. Settings Launcher physical placement 確定（Presentation 最終設計 — Launcher placement design pending）

### 13.3 TooltipPolicy の App 全体提供

`SettingsController` を MediaScreen subtree から参照できるよう、Composition Root で listen scope を App ルート近傍に置く（`ListenableBuilder` の ancestor 注入等）。Settings 画面を開いていなくても `showTooltips` 変更が Media 画面 Icon に即反映されること。

## 14. ownership / dispose

| 対象 | Owner | dispose |
|------|-------|---------|
| `SettingsRepository` concrete | **Composition Root（`NainaiApp`）** | 実装依存。stateless / `SharedPreferences` wrapper 等で **dispose 不要** なら明記 |
| `SettingsController` | **Composition Root** | `NainaiApp.dispose()` で dispose |
| `AudioOutputController` | Composition Root | [audio-output-settings.md](audio-output-settings.md) §6.4 |
| `MediaKitPlaybackService` | Composition Root（Phase C 以降） | 同上 |

**`SettingsController` が `SettingsRepository` を保持するが、Repository を dispose する責務は Composition Root** とする（Controller は borrow のみ。Service 共有 ownership パターンと整合）。

### 14.1 dispose 順序（Settings 追加後 — 設計）

```text
NainaiApp.dispose()
    1. AudioOutputController.dispose()
    2. MediaController.dispose()
    3. SettingsController.dispose()
    4. MediaKitPlaybackService.dispose()
    5. SettingsRepository — dispose 要否は concrete 実装に従う
```

## 15. Settings Launcher と Navigation contract

### 15.1 Pending（物理配置）

| 項目 | 状態 |
|------|------|
| Settings Launcher **アイコン / ボタンの最終配置** | **Launcher placement design pending** |
| 固定 Bottom Panel（client `68ff1b4`） | **実装済み**。Launcher 配置の前提 UI として利用可能 |
| 物理配置の確定 | Presentation 側の最終設計へ委譲。本 branch では **具体位置を独断確定しない** |

### 15.2 確定（Navigation contract）

Settings 画面自体の開き方は **本設計で確定** する。

| 概念 | 内容 |
|------|------|
| Entry API | **`openSettings(BuildContext context)`** または Composition Root 経由の **`SettingsNavigator.open()`** |
| Route | **`SettingsRoute`**（名前付き Route。実装 Phase で `go_router` 等の有無は client 方針に従う） |
| 内容 | `SettingsScreen` — General + Audio Section |
| Launcher 未確定の開発 | Modal / 暫定 AppBar action / debug entry で Section 開発を **妨げない** |

Launcher の物理配置だけ Pending。Navigation contract は Phase C / Settings 実装の prerequisite として利用可能。

## 16. Audio Output Phase C との接続境界

| 共通 Settings（本書） | Audio Output Phase C |
|----------------------|----------------------|
| Settings Shell / Section 見出し | `AudioOutputSettingsSection` Widget |
| `SettingsController` / `AppSettings` | `AudioOutputController` / `AudioOutputState` |
| Tooltip Policy | device enumeration / selection |
| General Section | Windows device list UI |
| Settings Persistence（共通） | Phase G `AudioOutputPreference` Persistence（別） |
| Settings 内 Non-blocking error（共通 save/load） | Audio Output 操作 Non-blocking error |

**参照:** [audio-output-settings.md](audio-output-settings.md) — Phase C 正本。変更は「共通 Settings Shell 参照追加」程度に留める。

## 17. テスト方針（実装 Phase 要求）

Service / Repository integration の細部は実装 Phase で確定。最低限以下を要求する。

### 17.1 Model（`AppSettings`）

| ケース |
|--------|
| default `showTooltips == true` |
| equality / `copyWith`（採用方式に応じる） |

### 17.2 SettingsController

| ケース |
|--------|
| initial default（`showTooltips: true`） |
| persisted `true` restore |
| persisted `false` restore |
| toggle `true` → `false` |
| toggle `false` → `true` |
| save request が Repository へ渡る |
| 連続 toggle で serialized save。最終永続値 = 最新操作 |
| 遅延した古い save 完了が新しい state を上書きしない（revision guard） |
| 最新 revision save failure → `lastPersistedSettings` へ revert + error |
| 古い revision save failure → 現在 UI state を revert しない |
| load failure → default `true` + error 可 |
| stale load 完了が load 後 mutation を上書きしない |
| dispose 後 `notifyListeners` しない |

### 17.3 Tooltip Policy / Widget

| ケース |
|--------|
| ON → Visual Tooltip あり |
| OFF → Visual Tooltip なし |
| OFF でも Semantics label あり |
| locale 変更時の label |
| runtime 切替即時反映 |

### 17.4 Settings Widget（General Section）

| ケース |
|--------|
| Switch 表示 |
| initial ON |
| OFF 操作 |
| ON 操作 |
| 日本語表示 |
| English 表示 |
| keyboard accessibility |
| semantics |

Audio Output Section の Widget test は [audio-output-settings.md](audio-output-settings.md) §18 が正本。責務重複させない。

## 18. 状態まとめ

| 項目 | 状態 |
|------|------|
| Common Settings Core | **Core Design Complete** |
| Tooltip ON / OFF | **Requirement / Design Complete** / **Not Implemented** |
| Concrete Persistence | **Pending** implementation-time technology selection |
| Settings Launcher physical placement | **Launcher placement design pending** |
| Audio Output Phase C | **Design Complete**（[audio-output-settings.md](audio-output-settings.md)）/ **未実装** |
