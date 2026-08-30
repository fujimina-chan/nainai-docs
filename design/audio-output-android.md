# Android 音声出力 / System Route Picker（Phase E 詳細設計）

Audio Output **Phase E** — Android 向け **OS 標準 route selection UX** の実装前詳細設計正本。

Service / platform 抽象の正本は [audio-output.md](audio-output.md)。Settings Shell / Section 構造は [settings.md](settings.md) / [audio-output-settings.md](audio-output-settings.md)。Windows hot unplug は [audio-output-hot-unplug.md](audio-output-hot-unplug.md)（**Android 非適用**）。

関連:

- [音声出力デバイス選択](audio-output.md)
- [Windows 音声出力 Settings UI（Phase C）](audio-output-settings.md)
- [UI Localization](localization.md)
- [Phase 2 UI](phase2-ui.md)

## 1. 位置づけ

| Phase | 内容 | 状態 |
|-------|------|------|
| A/B | 共通 Model / Service 抽象 / Windows 実装 | **Implemented** |
| C | Windows Settings subsystem | **Design Complete** / Not Implemented |
| D | Windows hot unplug / fallback | **Design Complete** / Not Implemented |
| **E** | **Android platform route picker**（本書） | **Design Complete** / **Not Implemented** |
| F | iOS route picker | 未設計詳細 / 未実装 |
| G | Preference 永続化 | 未設計詳細 / 未実装 |

**Design Complete の範囲:** 方式比較と **1 方式への選定** / Capability / Service API mapping / Native 境界 / Permissions / Settings UI / Error / Lifecycle / Phase G・D 境界 / テスト方針 / Localization キー案。

**含まない:** iOS（Phase F）、Preference 保存（Phase G）、Windows device list / Phase D fallback。

### 1.1 設計原則（確定）

Android では **物理 device ID 列挙 + アプリ内 Radio List 強制選択** を第一案に **しない**。

Bluetooth / wired / speaker / Cast 等は **OS routing** に委ね、**System Route Picker** を中心とする。

Windows Phase C の Radio List UI を Android へ **移植しない**。

Phase E では **偽装禁止** — fake route picker / 列挙 fake / 常に成功する空実装 **禁止**。

## 2. 方式比較と選定

### 2.1 候補比較

| 候補 | 概要 | 長所 | 短所 | 判定 |
|------|------|------|------|------|
| **A. AndroidX System Output Switcher** | `SystemOutputSwitcherDialogController.showDialog(context)` | 公式 System Output Switcher UX。AndroidX が API 差を吸収 | AndroidX 依存追加。API 30 未満は利用不可 | **採用（Phase E 唯一の native backend）** |
| **A″. Framework MediaRouter2 直接** | アプリ側で `if (API >= 34) MediaRouter2.showSystemOutputSwitcher()` | Framework API を直接利用 | OS version 分岐を nainai が再実装。AndroidX と二重管理 | **不採用**（AndroidX に委任） |
| **A′. MediaRouteChooserDialogFragment** | `MediaRouteChooserDialogFragment` + `MediaRouteSelector` | minSdk 19。Cast / remote route chooser として成熟 | **remote route chooser** であり local System Output Switcher の fallback にならない。phone speaker / connected BT / local system output を OS Output Switcher と同義で保証しない | **不採用**（§18） |
| **B. 第三者 Flutter plugin のみ** | pub.dev 上の route / cast plugin 単体依存 | 実装が早い場合あり | nainai 再生基盤（media_kit）・Lifecycle との整合未保証。メンテ依存 | **不採用**（単体採用しない） |
| **C. Platform Channel + Native adapter** | Flutter `AndroidAudioOutputService` ↔ Kotlin adapter | Domain 非露出。1 Service API に統合 | Native コード追加 | **採用（Flutter ↔ Android 境界）** |
| **D. media_kit `audioDevices` / `setAudioDevice`** | 既存 Player API を Android でも使用 | Windows と API 共通化しやすい | Android では **物理出力先ではなく** `audiotrack` / `opensles` 等の low-level backend が返る（[media-kit#1238](https://github.com/media-kit/media-kit/issues/1238)）。OS route UX にならない | **不採用** |

### 2.2 Phase E 採用方式（確定 — 1 方式）

**名称: Android System Route Picker（Platform Channel + AndroidX SystemOutputSwitcherDialogController）**

```text
Settings UI
    ↓
AudioOutputController.openSystemRoutePicker()
    ↓
AndroidAudioOutputService（Dart）
    ↓ MethodChannel
AndroidAudioRoutePickerAdapter（Kotlin）
    ↓
SystemOutputSwitcherDialogController.showDialog(context)
    （AndroidX 内部で API 30–33 / 34+ を処理）
```

**選定理由:**

| 項目 | 内容 |
|------|------|
| UX | OS 標準 System Output Switcher。アプリ内 device Radio List **禁止** |
| API 吸収 | AndroidX が R / S / T / U+ の差分を担当。**nainai は OS version 分岐を再実装しない** |
| media_kit | 再生は既存 `Player` 維持。**Audio Output 制御を media_kit Android device API に委ねない** |
| 保守 | 公式 Android / AndroidX API を正とする |
| fake 禁止 | 列挙 fake / Chooser で System Switcher を **偽装しない** |

### 2.3 AndroidX dependency（Phase E 設計値）

| 項目 | 値 |
|------|-----|
| Artifact | `androidx.mediarouter:mediarouter` |
| 採用予定 version | **`1.8.1`**（2026-08 時点の公式 stable 候補） |
| 不採用 | `1.9.0-alpha01` 等 alpha 系 |

**実装 Phase 開始時** に [AndroidX mediarouter release notes](https://developer.android.com/jetpack/androidx/releases/mediarouter) で **stable version を再確認** し、pin する。

nainai-client 現状は AndroidX MediaRouter **未使用**（[audio-output.md](audio-output.md) §3.3）。

### 2.4 参照一次資料

| 資料 | 用途 |
|------|------|
| [SystemOutputSwitcherDialogController](https://developer.android.com/reference/androidx/mediarouter/app/SystemOutputSwitcherDialogController) | Phase E **primary native API** |
| [MediaRouter2.showSystemOutputSwitcher()](https://developer.android.com/reference/android/media/MediaRouter2#showSystemOutputSwitcher()) | AndroidX 内部実装の参照（**アプリ側直接呼び出しは Phase E 第一設計にしない**） |
| [AndroidX mediarouter release notes](https://developer.android.com/jetpack/androidx/releases/mediarouter) | dependency version / `SystemOutputSwitcherDialogController` 追加履歴 |
| [Media routing overview](https://developer.android.com/media/routing/mediarouter) | Media routing / Output Switcher 概念 |
| [media_kit AudioDevice issue #1238](https://github.com/media-kit/media-kit/issues/1238) | Android media_kit device API 非採用根拠 |

## 3. OS API マトリクス（Phase E 確定）

nainai Native adapter は **OS version ごとの分岐を再実装しない**。以下は **AndroidX `SystemOutputSwitcherDialogController` の責務境界** を示す設計メモである。

| OS / API | System Output Switcher | nainai 扱い |
|----------|------------------------|-------------|
| **API < 30**（Android 9 以下） | AndroidX 経由でも **利用不可** | `supportsSystemRoutePicker = false`。picker action **非表示**（§7.2 / §17） |
| **API 30–33**（Android 11–13） | `SystemOutputSwitcherDialogController.showDialog(context)` — AndroidX が R / S / T 向け処理 | `supportsSystemRoutePicker = true`（runtime） |
| **API 34+**（Android 14+） | 同上。AndroidX 内部で `MediaRouter2.showSystemOutputSwitcher()` 等を利用 | `supportsSystemRoutePicker = true`（runtime） |

**禁止（Phase E）:**

- アプリ側第一設計として `if (API >= 34) MediaRouter2.showSystemOutputSwitcher()` を直接分岐
- API 24–33 向け `MediaRouteChooserDialogFragment` fallback
- `Intent` / SystemUI package / Chooser Fragment 等の **nainai 独自再実装**

## 4. nainai-client minSdk と Capability（Phase E 確定）

### 4.1 minSdk 実値（read-only 調査）

調査日: 2026-08-30
調査対象: `E:\fyna\dev\nainai\nainai-client`（**変更なし**）

| 項目 | 値 | 根拠 |
|------|-----|------|
| minSdk | **24** | `android/app/build.gradle.kts` → `minSdk = flutter.minSdkVersion` |
| Flutter default | **24** | `C:\flutter\packages\flutter_tools\gradle\src\main\kotlin\FlutterExtension.kt` → `minSdkVersion: Int = 24` |

nainai は **API 24–29 端末も minSdk 上はサポート対象** である。System Output Switcher は **API 30 未満では利用不可** のため、Capability は OS version を runtime 考慮する（§4.2）。

### 4.2 Capability（Phase E 確定）

Windows 値を **コピーしない**。

| Capability | Android（Phase E） | 根拠 |
|------------|-------------------|------|
| `supportsDeviceEnumeration` | **`false`** | アプリが physical device 一覧を提供しない |
| `supportsDirectDeviceSelection` | **`false`** | `selectDevice(deviceId)` 非サポート |
| `supportsPersistentDeviceId` | **`false`** | OS routing。stable app-side ID を保持しない |
| `supportsSystemRoutePicker` | **runtime: API >= 30 → `true` / API < 30 → `false`** | System Output Switcher は API 30+ のみ。minSdk 24 でも **Android 全体で常時 true は禁止** |

**実装 Phase:** `AndroidAudioOutputService.capabilities` は **起動時または初回参照時** に `Build.VERSION.SDK_INT` を評価し、上記 runtime 値を返す。

`AudioOutputController` / Settings UI は Capability 参照。`supportsSystemRoutePicker == false` では `openSystemRoutePicker()` を **呼ばない**（action 非表示）。

## 5. AudioOutputService API mapping（Android — 確定）

具象: **`AndroidAudioOutputService`**（`AudioOutputService` 実装）。**`MediaKitPlaybackService` に Android route policy を埋め込まない。**

### 5.1 Application Composition（Android — Phase E 確定）

Windows では `MediaKitPlaybackService` が `MediaPlaybackService` + `AudioOutputService` の両 interface を **同一 class** で実装する（Phase B 実装済み）。

Android Phase E では **分離** する:

```text
NainaiApp（Composition Root）
  ├─ MediaKitPlaybackService        ← MediaPlaybackService のみ（既存 Player 1 個）
  └─ AndroidAudioOutputService      ← AudioOutputService（Platform Channel）
        ↓
  AudioOutputController(service: androidAudioOutputService)
```

| 原則 | 内容 |
|------|------|
| Player 数 | **1 個**。`AndroidAudioOutputService` が **新しい Player を所有してはならない** |
| Composition | `NainaiApp` が platform に応じた `AudioOutputService` を inject |
| MediaKit 分離 | route picker / capability / empty device list は `AndroidAudioOutputService` の責務 |

### 5.2 Service API

| API | Android 挙動 |
|-----|--------------|
| `capabilities` | §4.2（runtime `supportsSystemRoutePicker`） |
| `availableDevices` | **常に空リスト** `[]` |
| `availableDevicesStream` | **常に空** `[]` を emit（初期 + 変更なし）。fake device **禁止** |
| `currentSelection` | **常に** `AudioOutputPreference.systemDefault()` |
| `currentSelectionStream` | 同上を emit。picker 後も **specificDevice へ更新しない** |
| `selectSystemDefault()` | **no-op 成功**（既に systemDefault 概念）。`UnsupportedError` **禁止** |
| `selectDevice(deviceId)` | **`UnsupportedError`** |
| `openSystemRoutePicker()` | Native adapter 経由（§6）。`supportsSystemRoutePicker == false` では `UnsupportedError` |

### 5.3 `currentSelection = systemDefault` の意味（Phase E 確定）

これは **「端末 speaker を使っている」** という意味 **ではない**。

| 意味 | 内容 |
|------|------|
| Application 上の正 | **Audio route is managed by the operating system.** |
| OS routing | Bluetooth / wired / Cast 等へ OS が route していても **`specificDevice` へ偽装しない** |
| 表示 | UI は **「システム管理」**（または意味的に同等な文言）。「システム既定」と「現在 speaker」を **混同させない** |

**禁止:** picker 選択結果を `specificDevice(deviceId: …)` へ **偽装** しない。取得不能な current route 名を **捏造** しない。

## 6. Native adapter（Platform Channel 境界）

### 6.1 責務分割

| 層 | 責務 |
|----|------|
| **Flutter / Dart — `AndroidAudioOutputService`** | capabilities / empty device list / system-managed selection / route picker invocation / error translation |
| **Native — `AndroidAudioRoutePickerAdapter`** | `SystemOutputSwitcherDialogController.showDialog(context)` / foreground・lifecycle 確認 / duplicate open guard |

Domain / Presentation へ `MediaRoute2Info` / Android 型を **露出しない**。

### 6.2 Flutter 側

| 項目 | 内容 |
|------|------|
| 実装 | `AndroidAudioOutputService` |
| Channel 例 | `com.fyna.nainai/audio_output`（実装 Phase で final 化） |
| Method | `openSystemRoutePicker` → `Future<void>` |
| 成功条件 | Native が picker **表示に成功**（§6.4） |

### 6.3 Android 側（Kotlin）

| 項目 | 内容 |
|------|------|
| 配置 | `android/app/src/main/kotlin/.../audio/`（概念） |
| クラス | `AndroidAudioRoutePickerAdapter` |
| 呼び出し | **`SystemOutputSwitcherDialogController.showDialog(context)`** |
| Context | **Activity context**（foreground）。background から picker **禁止** |
| Fragment | Chooser Fragment **不使用**。`FlutterActivity` 基底のまま Phase E 第一設計とする |

```kotlin
// 概念 — API 分岐は AndroidX 内部。nainai は直接 MediaRouter2 を呼ばない。
SystemOutputSwitcherDialogController.showDialog(context)
```

### 6.4 `showDialog` 戻り値（Phase E 確定）

`SystemOutputSwitcherDialogController.showDialog(context)` は **`boolean`** を返す。

| 戻り値 | 意味 | nainai 扱い |
|--------|------|-------------|
| **`true`** | picker **表示に成功** | `openSystemRoutePicker()` 成功 |
| **`false`** | picker が表示されなかった（foreground 条件等で起動不可） | **成功扱い禁止**。Non-blocking route-picker error 候補（§8） |

**`false` とユーザー dismiss の境界:**

| 事象 | error? |
|------|--------|
| `showDialog == false` | **Yes** — 表示自体ができなかった |
| `showDialog == true` の後、ユーザーが route を変更せず dismiss | **No** — open API の成功条件は **「picker が表示されたか」** であり **「route が選択されたか」** ではない |

Phase E では OS routing をアプリが追跡しないため、picker 表示成功後の dismiss / cancel を **selection failure として扱わない**。

Playback Blocking Error に **しない**。

### 6.5 open guard（Lifecycle — 確定）

| ガード | 内容 |
|--------|------|
| `_pickerOpen` | Service / adapter 内。二重 `openSystemRoutePicker` **禁止** |
| foreground | Activity `RESUMED` 相当でのみ invoke |
| app background | invoke しない → `showDialog` 相当 `false` + Non-blocking error 可 |
| stale result | picker 完了後に Controller `currentSelection` を **書き換えない**（§5.3） |

Settings 閉鎖 / rotation / `Activity` pause 中の二重 picker を **作らない**。

## 7. Android Settings UI（確定）

Settings Shell（General + Audio Section）は [settings.md](settings.md) と共有。Section **中身** のみ Android 固有。

### 7.1 レイアウト概念（supported — API >= 30）

```text
音声出力

現在の出力:
システム管理

[ 音声出力先を選択 ]
```

| 要素 | 内容 |
|------|------|
| Section title | Localization `audioOutput` |
| 現在の出力 | **`systemManaged`（システム管理）** — 「端末 speaker 使用中」ではない（§5.3） |
| 具体 device 名 | Phase E では **表示しない**（偽 metadata 禁止） |
| 操作 | 単一 Button → `openSystemRoutePicker()` |
| Radio List | **表示しない** |

### 7.2 Capability 駆動（supported / unsupported）

| `supportsSystemRoutePicker` | UI |
|-----------------------------|-----|
| **`true`**（API >= 30） | §7.1 の Button を **表示** |
| **`false`**（API < 30） | route picker action を **非表示**（第一候補）。必要なら短い説明。過剰な error **禁止** |

device 一覧行は **常に非表示**。

### 7.3 Accessibility

Button に Localization label / Semantics。picker は OS UI の a11y に委ねる。

## 8. Error policy（確定）

| 状況 | 扱い |
|------|------|
| `showDialog == false` / picker 起動失敗 | Settings 内 **Non-blocking error**（`routePickerFailed` 等） |
| `showDialog == true` 後のユーザー cancel / dismiss | **error ではない** |
| permission denial（将来） | Non-blocking。technical details 非表示 |
| Playback Blocking Error | **禁止** |
| Media Error 画面 | **禁止** |

`AudioOutputController` が Settings 向け error state を保持。`MediaError` とは型を共有しない（Phase C 同様）。

## 9. Permissions（Phase E 確定）

Phase E では **`BLUETOOTH_CONNECT` / `BLUETOOTH_SCAN` を System Output Switcher を開くためだけに nainai が追加しない**。

| Permission | Phase E |
|------------|---------|
| `BLUETOOTH_CONNECT` | **追加しない** |
| `BLUETOOTH_SCAN` | **追加しない** |
| `MODIFY_AUDIO_ROUTING` | **不要**（一般アプリ非対象） |

根拠: nainai 自身が Bluetooth device **enumeration を行わない**。picker は OS UI に委任。

**実装 Phase:** API 30 / 31 / 33 / 34+ **実機** で picker 起動・BT 出力切替を検証する。追加 permission が必要と判明した場合、実装で **勝手に Manifest へ追加せず** docs を再レビューする。

## 10. API < 30（minSdk 24 との関係）

nainai minSdk **24** のため、**API 29 以下も正式サポート対象** である。これらでは System Output Switcher は **利用不可**。

| 禁止 | 内容 |
|------|------|
| fake route picker | 見かけだけ picker がある UI |
| `MediaRouteChooserDialogFragment` で代替したことにする | System Switcher の偽装 |
| Bluetooth Settings を System Output Picker と呼ぶ | 誤解を招く |
| direct routing の勝手な実装 | OS 管理方針と矛盾 |

Settings UI は `supportsSystemRoutePicker == false` に従い、route picker action を **非表示または disabled**（第一候補: **非表示**）。

## 11. Phase D / Phase G 境界

### 11.1 Phase D（hot unplug fallback）

Windows Phase D（[audio-output-hot-unplug.md](audio-output-hot-unplug.md)）を Android へ **適用しない**。

| 項目 | Android |
|------|---------|
| automatic fallback | **OS routing** が担う |
| `selectSystemDefault()` 自動 invoke | Phase E **実装しない** |
| Phase D episode / retry | **対象外** |

### 11.2 Phase G（Persistence）

Phase E から **Preference を保存しない**。

| 禁止 | 内容 |
|------|------|
| `SettingsRepository` / 任意 storage への write | Phase E **禁止** |
| specific device ID の保存 | **禁止** |
| picker 後の `deviceId` 永続化 | **禁止** |

Phase G で `AudioOutputPreference` を保存する場合、Android は原則 **`systemDefault` のみ** を想定（[audio-output.md](audio-output.md) §3.5）。Phase G 設計時に確定。

## 12. Playback との関係

| 項目 | 内容 |
|------|------|
| Player | 既存 **1 Player** 維持（One Player 原則） |
| route 変更 | OS / mpv 内部 routing。Phase E から Media 再 open **要求しない** |
| Volume / Mute | [media-playback.md](media-playback.md) 既存設計を変更しない |

実装後 **Android 実機** で再生中 picker 操作を検証する。

## 13. Localization（キー案 — 未実装）

[localization.md](localization.md) gen-l10n / ARB。本設計時点では ARB 変更なし。

| キー | ja | en |
|------|----|----|
| `audioOutput` | 音声出力 | Audio Output |
| `systemManaged` | システム管理 | System managed |
| `audioOutputSelectRoute` | 音声出力先を選択 | Select audio output |
| `audioOutputRoutePickerFailed` | 音声出力の選択画面を表示できませんでした | Could not show the audio output picker |
| `audioOutputSystemRouteUnavailable` | この端末では音声出力の選択に対応していません | Audio output selection is not available on this device |

Phase C Windows キー（`systemDefault` / device 一覧等）と **役割を混同しない**。

## 14. テスト方針（実装 Phase 要求）

### 14.1 Dart — `AndroidAudioOutputService` / Controller

| ケース |
|--------|
| capabilities 4 値（API >= 30 mock / API < 30 mock） |
| `supportsSystemRoutePicker` runtime（SDK_INT 30 境界） |
| `availableDevices` 常に空 |
| `selectDevice` → UnsupportedError |
| `selectSystemDefault` → no-op 成功 |
| `openSystemRoutePicker` → channel invoke |
| Native `showDialog == true` → 成功、error なし |
| Native `showDialog == false` → non-blocking error |
| picker 表示成功後 dismiss → error なし |
| duplicate open → 二重 invoke 防止 |
| `currentSelection` 常に systemDefault |
| API < 30: `openSystemRoutePicker` → UnsupportedError（action 非表示と整合） |

### 14.2 Widget — Android Settings Section

| ケース |
|--------|
| API >= 30: Button 表示（Radio 一覧なし） |
| API < 30: Button **非表示** |
| `systemManaged` 表示 |
| picker 操作 |
| ja / en |
| accessibility / semantics |

### 14.3 Native（unit / instrumentation — 推奨）

| ケース |
|--------|
| `SystemOutputSwitcherDialogController.showDialog` invoke（API 30+） |
| foreground 時 invoke → `true` |
| background 時 invoke → `false` + error 経路 |
| Activity lifecycle（pause / resume） |
| duplicate open guard |

Platform boundary は **Native instrumentation** で担保する設計とする。Dart test のみでは不十分。

**削除（Phase E 不採用）:** `MediaRouteChooserDialogFragment` fallback テスト、`showSystemOutputSwitcher` 直接分岐テスト。

## 15. MediaRouteChooserDialogFragment（不採用記録）

Phase E の採用方式から **外す**。

| 項目 | 内容 |
|------|------|
| 性質 | **remote route chooser** |
| 不採用理由 | local **System Output Switcher** の fallback として phone speaker / connected Bluetooth audio / local system output を OS Output Switcher と同義で **保証しない** |
| Phase E 方針 | 偽装禁止。API 30 未満は **unsupported** と明示 |

## 16. Phase 境界まとめ

| Phase | Android 責務 |
|-------|--------------|
| **C** | —（Windows のみ） |
| **D** | —（Windows のみ） |
| **E** | System Route Picker / `AndroidAudioOutputService` |
| **F** | —（iOS） |
| **G** | persistence（Phase E では未着手） |

## 17. 状態

| 項目 | 状態 |
|------|------|
| Phase E 設計 | **Design Complete** |
| Phase E 実装 | **Not Implemented** |
| Concrete Android route technology | **Selected** — `SystemOutputSwitcherDialogController` + `androidx.mediarouter:mediarouter:1.8.1`（実装開始時 stable 再確認） |
| Android 実機 route picker 検証 | 実装 Phase で必須 |
