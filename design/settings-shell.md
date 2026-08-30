# Settings Shell / Launcher / Presentation Composition（詳細設計）

Common Settings の **Launcher から Shell 内 Section まで** の Presentation 最終設計正本。

**上位契約:** [settings.md](settings.md)（Launcher 配置 / `AppSettings` / `SettingsController` / Tooltip Policy）、[settings-persistence.md](settings-persistence.md)（Concrete Persistence / Phase G user command）、[audio-output-settings.md](audio-output-settings.md)（`AudioOutputSettingsSection`）。

**委譲:** Application Composition の **constructor graph / dispose order / platform service construction** は docs lane 1 **[audio-output-composition.md](audio-output-composition.md)**（Phase H）が正本。本書は **Presentation が必要とする依存契約** のみ記載する。

関連:

- [Phase 2 UI](phase2-ui.md) §14 — Launcher 視覚参照
- [Design System](design-system.md) — Token / breakpoint
- [Audio Output Preference 永続化（Phase G）](audio-output-persistence.md)
- [Windows Audio Output Settings（Phase C）](audio-output-settings.md)
- [Android System Route Picker（Phase E）](audio-output-android.md)
- [iOS System Route Picker（Phase F）](audio-output-ios.md)
- [Windows hot unplug（Phase D）](audio-output-hot-unplug.md)

## 1. 位置づけ

| 項目 | 内容 |
|------|------|
| スコープ | Settings Launcher / Settings Shell / Section Composition / Tooltip Presentation / open-close lifecycle |
| 状態 | **Design Complete** / **Not Implemented** |
| 含まない | `SettingsRepository` concrete 実装、Application Composition 具体 graph（Phase H）、Phase D fallback ロジック本体 |

### 1.1 既存確定要素の統合

| 要素 | 正本 | 本書での扱い |
|------|------|--------------|
| App-level Settings Launcher | [settings.md](settings.md) §15 | **維持**（§3） |
| Common Settings Core | [settings.md](settings.md) §4 | **参照** — `SettingsController` / `AppSettings` |
| `showTooltips` | [settings.md](settings.md) §6 | **Shell General Section**（§9） |
| Common Settings Persistence | [settings-persistence.md](settings-persistence.md) | **load / save / failure UI**（§10–§12） |
| `AudioOutputSettingsSection` | [audio-output-settings.md](audio-output-settings.md) | **Shell Audio Section**（§14） |
| Phase G user command | [audio-output-persistence.md](audio-output-persistence.md) §5 | **Audio command 契約**（§14） |

## 2. 設計原則

| 原則 | 内容 |
|------|------|
| App-level Settings | Media 選択 / Playback 状態に **非依存** |
| Media 非破壊 | Settings 表示中も loaded media を **勝手に unload しない** |
| Controller lifetime | `SettingsController` / `AudioOutputController` は **App lifetime**。Shell open/close で **再生成しない** |
| Presentation 薄さ | Shell / Section が `AudioOutputService` を **直接 import / inject しない** |
| fake 禁止 | 未実装 Settings 項目を **増やさない** |
| Blocking 禁止 | Settings load/save 失敗を **Playback Blocking Error** にしない |

Professional Creative Media Tool（[design-system.md](design-system.md) Persona）の **現在 UI を壊さない**。

## 3. Settings Launcher（確定 — 維持）

[settings.md](settings.md) §15 / [phase2-ui.md](phase2-ui.md) §14 を正とする。

| 項目 | 内容 |
|------|------|
| 配置 | **App-level top-right utility button** |
| アイコン（第一候補） | `Icons.settings_rounded` 相当 |
| Hit target | **44 logical px 以上** |
| SafeArea | **必須** |
| 禁止 | Main application の **full-width permanent AppBar**（44–56px 縦消費）、Bottom Panel 内配置、Media Area 内部配置 |

### 3.1 Semantics / Tooltip

| 項目 | 内容 |
|------|------|
| Semantics label（正式） | ja: **設定** / en: **Settings**（ARB `settings`） |
| Visual Tooltip | **`TooltipPolicy` 対象** — OFF 時 Visual のみ非表示 |
| Semantics | **常時維持** — Tooltip OFF ≠ a11y OFF |

### 3.2 利用可能な Media 状態（確定）

Launcher は **常に利用可能**:

| 状態 |
|------|
| Unselected / Loading / Ready / Playing / Paused / Stopped / Blocking Error / Non-blocking Error |

Bottom Panel（expanded / collapsed / hidden）、**compact landscape** に **依存しない**。

### 3.3 起動契約

```text
Settings IconButton.onPressed
    → openSettings()
    → Settings Shell を表示（§4）
```

**禁止:** Gear → `AudioOutputSettingsSection` **直結**。

## 4. Settings Shell 形式 — 比較と選定

### 4.1 候補比較

| 候補 | Desktop（≥800px） | Mobile（<800px） | 判定 |
|------|-------------------|------------------|------|
| **Centered Dialog** | Main view 分断 | Desktop 幅縮小 **禁止** | **不採用** |
| **Full Route 置換（Desktop）** | Main media view 破壊 | — | **Desktop 不採用** |
| **End Side Sheet** | Media 左側維持 | 単独では Mobile UX 不足 | **Desktop 採用** |
| **Adaptive Shell** | Side Sheet | Full-screen Route | **採用（1 方式）** |

### 4.2 採用方式（確定）

**名称: Adaptive Settings Shell**

| breakpoint | Presentation |
|------------|--------------|
| **≥ 800 logical px** | **End-anchored Side Sheet** |
| **< 800 logical px** | **Full-screen Settings Route** |

Desktop Side Sheet を Mobile へ **縮小しない**。

```text
openSettings()
    ↓
SettingsShell（adaptive）
    ├─ Desktop: End Side Sheet
    └─ Mobile: Full-screen Settings Route
            ├─ GeneralSettingsSection
            └─ AudioOutputSettingsSection
```

### 4.3 Desktop Side Sheet — 寸法（確定）

breakpoint `layout-breakpoint-desktop` = **800**（[design-system.md](design-system.md)）。

**Panel 幅（正式）:**

```text
sheetWidth = min(480 logical px, viewportWidth × 0.40)
```

| viewport | sheetWidth |
|----------|------------|
| 800（最小 Desktop） | **320 logical px** |
| 1200 | 480 logical px |

追加の magic width **禁止**。上記式のみ。

### 4.4 Desktop Side Sheet — modal 挙動（確定）

Side Sheet 表示中:

| 項目 | 挙動 |
|------|------|
| Media Widget tree | **保持**（unload **しない**） |
| Playback | **継続** — pause **しない** |
| Position | **reset しない** |
| Bottom Panel state | **reset しない** |
| Scrim | 背面 Media UI への **pointer 操作を遮断** |
| 要約 | **Visible: yes / Playback: continue / Interactive behind sheet: no** |

### 4.5 Desktop close（確定）

Side Sheet を閉じる手段（**すべて** サポート）:

| 手段 |
|------|
| Sheet 内 **明示的 Close button** |
| **Escape** key |
| **Scrim click / tap** |

close 時 **`SettingsController` / `AudioOutputController` を dispose しない**。

### 4.6 Mobile Full-screen Route（確定）

| 項目 | 内容 |
|------|------|
| レイアウト | **Full-screen Settings Route** — Desktop Side Sheet 縮小 **禁止** |
| SafeArea | **必須** |
| Header | **Back** + タイトル **設定 / Settings** を表示可 |
| AppBar 例外 | Main application の permanent AppBar 禁止とは **別物** — **Settings Route 内 navigation header は許可** |
| System back | Settings Route を **close** |
| loaded media | Route close で media 自体は **破棄しない** |

## 5. Settings Sections（MVP — 確定）

```text
設定

一般
  └─ ツールチップを表示（showTooltips）

オーディオ
  └─ 音声出力（AudioOutputSettingsSection）
```

| Section | 内容 |
|---------|------|
| **General** | `showTooltips` のみ |
| **Audio** | `AudioOutputSettingsSection` |

fake 項目 **禁止**。MVP で General / Audio のみのため **別 navigation rail / sidebar hierarchy は導入しない**（§16）。

## 6. Presentation 依存契約（Phase H 境界）

本書が定める **Presentation 側の inject 契約** のみ。具体的 constructor graph / dispose order / platform service construction は **[audio-output-composition.md](audio-output-composition.md)**（Phase H）が正本。

```text
SettingsShell
  ├─ SettingsController          ← state / setShowTooltips
  └─ AudioOutputSettingsSection
        ├─ State: AudioOutputController
        └─ User commands: AudioOutputPreferenceCoordinator
            または Coordinator を backing とする command interface
```

| 層 | 依存 |
|----|------|
| **State 表示** | `SettingsController` / `AudioOutputController` |
| **User commands（Phase G）** | `AudioOutputPreferenceCoordinator`（または同等 command abstraction） |
| **禁止** | Presentation へ `AudioOutputService` **直接 inject** |

Shell / Section が Service 型を **import しない**。

## 7. State ownership（確定）

| 対象 | lifetime |
|------|----------|
| `SettingsController` | **App** — Shell close で dispose **禁止** |
| `AudioOutputController` | **App** — 同上 |
| `AudioOutputPreferenceCoordinator` | **App**（Phase G — Phase H が graph 確定） |
| `SettingsShell` open state | **Shell のみ** |

## 8. Tooltip architecture（確定）

**採用: `TooltipPolicy` + `NainaiTooltip`**

各 Widget へ `if (showTooltips) { Tooltip(...) }` **大量直書き禁止**。

### 8.1 TooltipPolicy 責務

| 担当 | 非担当 |
|------|--------|
| Visual Tooltip **enabled / disabled** のみ | Accessibility semantics を OFF にすること |

```dart
// 概念
TooltipPolicy(enabled: settings.showTooltips)

NainaiTooltip(
  message: l10n.collapsePanel,
  child: IconButton(...),
)
```

### 8.2 Semantics 独立（必須）

| 禁止 | 内容 |
|------|------|
| Tooltip message と Semantics label の **同一 ON/OFF 制御** | `showTooltips == false` でも button semantic name / slider semantic label / Launcher semantic label **維持** |

### 8.3 Windows AXTree 保護

既存 Windows accessibility 修正を **壊さない**。

| 禁止 | 内容 |
|------|------|
| Slider identity / enabled / onChanged / Semantics wrapper の **付け替え** | TooltipPolicy 導入理由に **しない** |

`NainaiTooltip` は **既存 control identity を極力変えない薄い Presentation wrapper** とする（[settings.md](settings.md) §6.3）。

Phase 2 既存 controls は **`NainaiTooltip` へ段階移行**。

## 9. General Section — 初期化 UI（確定 — 1 方式）

### 9.1 `SettingsController.initialize()` 完了前 — Loading placeholder

**正式採用: Loading placeholder 方式**

| 項目 | 内容 |
|------|------|
| 設定行 | 「ツールチップ」行 **自体は表示** |
| Switch | **表示しない**（ON/OFF どちらも確定表示 **禁止**） |
| trailing | compact loading indicator、または design-system 準拠 loading placeholder |

**理由:** `AppSettings.defaults().showTooltips == true` であっても persisted value が `false` の可能性がある。**false 固定禁止** に加え **true 固定の一瞬表示も避ける**。

**不採用:** indeterminate Switch / disabled Switch / skeleton Switch の **候補列挙**。

Shell 自体は initialize 前も **open 可能**。

### 9.2 initialize 成功

loaded `SettingsController.settings.showTooltips` を反映した **Switch へ置換**:

| 値 | 表示 |
|----|------|
| `true` | **ON** |
| `false` | **OFF** |

変更: Switch → `setShowTooltips` → subtree 即時反映 + async persist。

Visual Tooltip ON/OFF は §8。Semantics **常時維持**。

## 10. 保存中 UI（`setShowTooltips` → save 中）

[settings-persistence.md](settings-persistence.md) serialized write 契約を **壊さない**。

| 項目 | 設計 |
|------|------|
| Switch | **disable しない** — optimistic current state 表示 |
| Shell 全体 | **Blocking しない** |
| 操作 | ON → OFF → ON 等 **rapid toggle 可能** |
| serialized save | `SettingsController` が吸収 |
| global save spinner | **表示しない**（MVP visual indicator **不要**） |

`isSaving` は Controller **内部 state / test** に存在してよい。MVP で save 専用 global indicator **必須としない**。

**理由:** 軽量 preference / serialized write 済み / rapid user intent を妨げない / UI churn 抑制。

## 11. Save failure（確定）

| 項目 | Shell 扱い |
|------|------------|
| Controller | `lastPersistedSettings` へ **revert** |
| Switch | revert 後 Controller state に **従う** |
| 表示 | Shell 内 **Non-blocking error** |
| technical details | **禁止** |
| App 全体 | **Blocking Error 禁止** |

## 12. Load failure（確定）

[settings-persistence.md](settings-persistence.md) — load 失敗時 defaults `showTooltips: **true**`。

| 項目 | Shell 扱い |
|------|------------|
| initialize | **完了扱い**（defaults 適用済み） |
| Switch | **ON** を正式表示 |
| error | **Non-blocking settings load error** |
| Shell | **利用可能** |

## 13. Close behavior（確定）

| 項目 | 内容 |
|------|------|
| 明示 Save / Apply / OK | **要求しない** — 変更ごと persist |
| Desktop dismiss | §4.5（Close / Escape / scrim） |
| Mobile dismiss | system back / header Back |
| revert | save failure 時のみ（§11） |

## 14. Audio Output Section — Presentation 依存（確定）

Widget: **`AudioOutputSettingsSection`**（[audio-output-settings.md](audio-output-settings.md)）。

### 14.1 正式 Presentation 依存

| 依存 | 用途 |
|------|------|
| **State** | `AudioOutputController` / `AudioOutputState` |
| **User commands** | `AudioOutputPreferenceCoordinator` または Coordinator backing の **command interface** |
| **禁止** | `AudioOutputService` を Presentation へ **直接 inject** |

### 14.2 Phase G user command path（Windows 等）

[audio-output-persistence.md](audio-output-persistence.md) §5.1:

```text
Presentation
    ↓ user command
AudioOutputPreferenceCoordinator.selectDevice / selectSystemDefault
    ↓
AudioOutputController
    ↓
AudioOutputService
    ↓ selection success
Coordinator.persistExplicitSelection(...)
```

Shell / Section から Controller へ **直接** `selectDevice` を呼ぶ完成形 **にしない**（Coordinator / command abstraction 経由）。

Phase H readiness / enumeration gate UI は Section + Controller state。具体 lifecycle は Phase H 正本。

### 14.3 Windows Audio UI（維持）

System Default + specific devices **Radio List**（Phase C）。

### 14.4 Android Audio UI（維持）

```text
現在の出力: システム管理
[ 音声出力先を選択 ]
```

| 項目 | 内容 |
|------|------|
| State | `AudioOutputController` |
| Command | Coordinator / command abstraction → Controller → `AndroidAudioOutputService.openSystemRoutePicker()` |
| API < 30 | action **非表示** |
| Phase G | route picker 結果から Preference device ID **作成禁止** |

### 14.5 iOS 例外境界（維持）

| 項目 | 内容 |
|------|------|
| State | `AudioOutputController` |
| Native picker | **`IOSAudioRoutePickerView`**（Platform View）— Presentation-specific |
| 性質 | Coordinator / Service による programmatic open **ではない** |
| `supportsSystemRoutePicker` | **参照しない** |

## 15. Phase D notification（確定）

| 項目 | 所有者 |
|------|--------|
| **`AudioOutputNotificationHost`** | **App-level** — Settings Shell **外** |
| Shell open/close | notification host **再生成しない** |
| Shell 内 | 設定文脈 error のみ |

[audio-output-hot-unplug.md](audio-output-hot-unplug.md)。

## 16. Navigation（MVP — 確定）

| 項目 | 内容 |
|------|------|
| 構造 | Desktop Side Sheet / Mobile Route 内部は **単純 vertical layout** |
| 構成 | Settings title + General section + Audio section |
| 禁止 | General / Audio のみのため **別 navigation rail / 過剰 sidebar** |

## 17. Accessibility（Shell 全体）

| 項目 | 内容 |
|------|------|
| Launcher | Semantics **設定 / Settings** — 常時（§3.1） |
| Side Sheet | focus trap / Escape（§4.5） |
| Mobile Route | semantic page title |
| iOS Platform View | native a11y 優先、Flutter 重複 label **禁止** |
| `showTooltips == false` | Semantics **維持**（§8.2） |

## 18. テスト方針（実装 Phase 要求）

### 18.1 Launcher

全 Media 状態、Bottom Panel 状態、compact landscape、SafeArea、Semantics 常時。

### 18.2 Settings Shell

Desktop Side Sheet / Mobile Route、open-close、Controller 再生成なし。

### 18.3 Initialization

| ケース |
|--------|
| initialize pending — **Switch 自体が表示されない** |
| loading placeholder 表示 |
| persisted `false` 読込後 — **OFF Switch** |
| load failure 後 — **ON Switch** + Non-blocking error |

### 18.4 Saving

| ケース |
|--------|
| `isSaving` 中も Switch **操作可能** |
| rapid toggle 可能 |
| global blocking **なし** |
| save failure — revert + Non-blocking error |

### 18.5 Desktop Side Sheet

| ケース |
|--------|
| `sheetWidth = min(480, viewportWidth × 0.40)` |
| Media **mounted 維持** |
| playback **継続** |
| scrim が back interaction **遮断** |
| Escape / scrim / explicit Close |
| close 後 Controller dispose **なし** |

### 18.6 Audio Output command boundary

| ケース |
|--------|
| Shell / Section が `AudioOutputService` **直接依存しない** |
| state は `AudioOutputController` |
| user command は Coordinator / command abstraction |
| iOS — native Platform View、`supportsSystemRoutePicker` **非参照** |
| Phase D notification host が Shell **外** |

### 18.7 Tooltip

`TooltipPolicy` + `NainaiTooltip`、Semantics 維持、Widget 直書き if なし。

## 19. 状態

| 項目 | 状態 |
|------|------|
| Settings Shell / Launcher Presentation | **Design Complete** |
| Settings Shell 実装 | **Not Implemented** |
| Settings Launcher 実装 | **Not Implemented**（配置 Design Complete — [settings.md](settings.md) §15） |
| Common Settings Core | client `4109a13` **Implemented** |
| Common Settings Persistence | **Design Complete** / Concrete **Selected** / **Not Implemented** |
| Tooltip Policy / `NainaiTooltip` | **Not Implemented** |

## 20. 参照一次資料（docs 内）

| 資料 | 用途 |
|------|------|
| [settings.md](settings.md) | Launcher / Controller / Tooltip 契約 |
| [settings-persistence.md](settings-persistence.md) | load / save / Coordinator command |
| [audio-output-persistence.md](audio-output-persistence.md) | Phase G user command path |
| [design-system.md](design-system.md) | breakpoint 800 |
| [audio-output-composition.md](audio-output-composition.md) | Phase H — Composition graph（本書は参照のみ） |
| [audio-output-settings.md](audio-output-settings.md) | Windows Section |
| [audio-output-android.md](audio-output-android.md) | Android Section |
| [audio-output-ios.md](audio-output-ios.md) | iOS Section |
