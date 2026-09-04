# iOS 音声出力 / System Route Picker（Phase F 詳細設計）

Audio Output **Phase F** — iOS 向け **OS 標準 route selection UX** の実装前詳細設計正本。

Service / platform 抽象の正本は [audio-output.md](audio-output.md)。Settings Shell / Section 構造は [settings.md](settings.md) / [audio-output-settings.md](audio-output-settings.md)。Windows hot unplug は [audio-output-hot-unplug.md](audio-output-hot-unplug.md)（**iOS 非適用**）。Android Phase E は [audio-output-android.md](audio-output-android.md)。

関連:

- [音声出力デバイス選択](audio-output.md)
- [Windows 音声出力 Settings UI（Phase C）](audio-output-settings.md)
- [UI Localization](localization.md)
- [Phase 2 UI](phase2-ui.md)

## 1. 位置づけ

| Phase | 内容 | 状態 |
|-------|------|------|
| A/B | 共通 Model / Service 抽象 / Windows 実装 | **Implemented** |
| C | Windows Settings subsystem | **Implemented**（I-3B） |
| D | Windows hot unplug / fallback | **Implemented**（コード）/ 実機 **Pending** |
| E | Android System Route Picker | **Implemented** / 実機 **Pending** |
| **F** | **iOS platform route picker**（本書） | **Implemented**（コード + I-3B wiring）/ **iOS compile・実機 Pending** |
| G | Preference 永続化 | **Implemented**（コード） |

**Design Complete の範囲:** 方式比較と **1 方式への選定** / Capability / Service API mapping / Presentation 境界 / Permissions / Settings UI / Error / Lifecycle / Phase G 境界 / テスト方針 / Localization キー案 / 実機検証項目。

**含まない:** Preference 保存（Phase G）、Windows device list / Phase D fallback、第三者 casting プロトコルの独自実装。

### 1.1 設計原則（確定）

iOS では **Bluetooth / AirPlay 等の physical output device ID をアプリが一覧化し、Radio List で直接強制選択する** 設計を第一案に **しない**。

OS 標準 **Route Picker**（`AVRoutePickerView`）を **Presentation 層** で embed する。

Windows Phase C の Radio List UI を iOS へ **移植しない**。

Android Phase E の AndroidX API / SDK version 分岐 / Permission 設計を **そのままコピーしない**（共通化可能なのは Service 抽象・OS-managed 概念・Non-blocking error 方針のみ）。

Phase F では **偽装禁止** — fake device 列挙 / fake route picker / 常に成功する空実装 **禁止**。

### 1.2 `supportsSystemRoutePicker` の正式意味（全 platform 統一）

`AudioOutputCapabilities.supportsSystemRoutePicker` は、

**`AudioOutputService.openSystemRoutePicker()` を Application から直接呼び出せるか**

を意味する。

| 含まない意味 | 内容 |
|--------------|------|
| 「OS に route picker UI が何らかの形で存在する」 | **含めない** — embedded native control の有無は **Presentation 責務** |

iOS では embedded `AVRoutePickerView` があるが、`openSystemRoutePicker()` は **Unsupported** のため `supportsSystemRoutePicker = false` とする（§4 / §8）。

### 1.3 Domain Capability 拡張禁止（Phase F 確定）

今回、次を `AudioOutputCapabilities` へ **追加しない**:

- `supportsEmbeddedSystemRoutePicker`
- `supportsNativeRoutePickerView`
- その他 iOS Presentation 専用 bool

**理由:** iOS 固有 Presentation 要件を Domain へ漏らさない。Android Phase E 実装は既存 4 capability で並行中。

iOS Settings の route picker 有無は **Composition / Presentation**（iOS 向け native route picker component の存在）で決定する。

## 2. 方式比較と選定

### 2.1 候補比較

| 候補 | 概要 | 長所 | 短所 | 判定 |
|------|------|------|------|------|
| **A. AVRoutePickerView（AVKit）** | UIKit `UIView` サブクラス。view hierarchy へ配置しユーザーが tap | Apple 公式。AirPlay / 近傍 receiver / ローカル出力を system UI で統合 | programmatic route sheet open の public API **なし** | **採用（Phase F 唯一の route picker UI）** |
| **B. MPVolumeView route button** | `showsRouteButton` 等 | 古い iOS で実績 | iOS 13 以降 route button 用途 **deprecated**。WWDC19: **`AVRoutePickerView` を使用** | **不採用**（§12） |
| **C. AVAudioSession route APIs** | `currentRoute` / route change notification 等 | session 設定・route **観測** | **ユーザー向け output 選択 UI ではない** | **不採用**（§11） |
| **D. AVInputPickerInteraction** | input device picker | input 選択に公式 | **output route picker ではない** | **不採用** |
| **E. 第三者 Flutter plugin のみ** | pub.dev plugin 単体 | 導入が早い場合あり | 保守性・media_kit 整合未保証 | **不採用** |
| **F. Platform Channel（route open 用）** | `openSystemRoutePicker` を Channel で native open | Android と API 対称に見える | iOS に formal open API なし。hidden view / synthetic tap が必要になりやすい | **不採用** |
| **G. Flutter Platform View（UiKitView）** | Settings 内へ `AVRoutePickerView` embed | Apple 推奨（view hierarchy へ追加） | Platform View 実装・a11y が必要 | **採用（Presentation 統合）** |
| **H. media_kit device API** | Windows 同様 | API 共通化 | iOS 物理出力 UX にならない | **不採用** |

### 2.2 Phase F 採用方式（確定 — 1 方式）

**名称: iOS System Route Picker（Presentation Platform View + IOSAudioOutputService）**

```text
AudioOutputService（Domain）
  IOSAudioOutputService
    ├─ capabilities（§4 — 4 値すべて false 側）
    ├─ currentSelection = systemDefault
    └─ command APIs（openSystemRoutePicker は Unsupported）

Presentation（iOS Settings Audio Section）
  IOSAudioRoutePickerView（Dart Widget）
    └─ UiKitView
          └─ IOSAudioRoutePickerPlatformView（Swift）
                └─ AVRoutePickerView
                      ↓ user tap
                  System route popover

MediaKitPlaybackService（Player 1 個 — route policy を含まない）
```

**選定理由:**

| 項目 | 内容 |
|------|------|
| UX | Apple 公式 route picker。Windows Radio List **禁止** |
| 公式 | [AVRoutePickerView](https://developer.apple.com/documentation/avkit/avroutepickerview): view hierarchy へ配置する system route picker |
| Service 契約 | `openSystemRoutePicker()` 非対応 → `supportsSystemRoutePicker = false` |
| Presentation | embedded picker は **Service capability ではなく Presentation 責務** |
| fake 禁止 | hidden view / synthetic tap / private API / subview 探索 **禁止** |

### 2.3 参照一次資料

| 資料 | 用途 |
|------|------|
| [AVRoutePickerView](https://developer.apple.com/documentation/avkit/avroutepickerview) | view hierarchy へ配置する system route picker（Phase F primary UI） |
| [AVRoutePickerView.prioritizesVideoDevices](https://developer.apple.com/documentation/avkit/avroutepickerview/prioritizesvideodevices) | default **`false`** — Phase F 正式値 |
| [AVRoutePickerViewDelegate](https://developer.apple.com/documentation/avkit/avroutepickerviewdelegate) | `willBegin` / `didEnd` — lifecycle のみ |
| [MPVolumeView](https://developer.apple.com/documentation/mediaplayer/mpvolumeview) | volume UI。route button **非採用** |
| [AVAudioSession.currentRoute](https://developer.apple.com/documentation/avfaudio/avaudiosession/currentroute) | 観測のみ（picker 代替ではない） |
| [Reaching the Big Screen with AirPlay 2 (WWDC19)](https://developer.apple.com/videos/play/wwdc2019/501/) | `AVRoutePickerView` 採用 / MPVolumeView route button deprecated |
| [Tune up your AirPlay audio experience (WWDC23)](https://developer.apple.com/videos/play/wwdc2023/10238/) | AirPlay picker 統合 |
| [Support local network privacy (WWDC20)](https://developer.apple.com/videos/play/wwdc2020/10110/) | Local Network permission 境界 |
| [media_kit AudioDevice issue #1238](https://github.com/media-kit/media-kit/issues/1238) | media_kit device API 非採用根拠 |

## 3. nainai-client iOS 環境（read-only 調査）

調査日: 2026-08-30
調査対象: `E:\fyna\dev\nainai\nainai-client`（**変更なし**）

| 項目 | 値 | 根拠 |
|------|-----|------|
| iOS Deployment Target | **15.0** | `ios/Runner.xcodeproj/project.pbxproj` → `IPHONEOS_DEPLOYMENT_TARGET = 15.0` |
| `AVRoutePickerView` 要件 | **iOS 11.0+** | [AVRoutePickerView](https://developer.apple.com/documentation/avkit/avroutepickerview) |
| 互換性 | Deployment 15.0 ≥ 11.0 | nainai サポート iOS 全端末で `AVRoutePickerView` 利用可 |
| カスタム iOS Native コード | **なし**（Phase F 起点） | `AppDelegate.swift` のみ |
| Info.plist 追加 permission | **なし** | `ios/Runner/Info.plist` |

## 4. Capability（Phase F 確定）

`IOSAudioOutputService.capabilities` の正式値:

| Capability | iOS（Phase F） | 根拠 |
|------------|---------------|------|
| `supportsDeviceEnumeration` | **`false`** | physical device 一覧を Domain へ提供しない |
| `supportsDirectDeviceSelection` | **`false`** | `selectDevice(deviceId)` 非サポート |
| `supportsSystemRoutePicker` | **`false`** | `openSystemRoutePicker()` 非サポート（§1.2 / §5.3） |
| `supportsPersistentDeviceId` | **`false`** | OS routing。stable app-side ID を保持しない |

**禁止:** embedded `AVRoutePickerView` の存在を理由に `supportsSystemRoutePicker = true` と **しない**。

`AudioOutputController` は `supportsSystemRoutePicker == false` により **`openSystemRoutePicker()` を呼ばない**。iOS Settings の Platform View 表示は **Presentation が担当**（§6 / §7）。

## 5. 全 platform 比較 — Service 契約（Phase F 確定）

| Platform | `supportsSystemRoutePicker` | `openSystemRoutePicker()` | Settings UI |
|----------|----------------------------|---------------------------|-------------|
| **Windows** | `false` | `UnsupportedError` | specific device **Radio List** |
| **Android API >= 30** | `true`（runtime） | **Supported** — Controller → Service → native System Output Switcher | Flutter Button → `openSystemRoutePicker()` |
| **Android API < 30** | `false`（runtime） | `UnsupportedError` | route picker action **非表示** |
| **iOS** | **`false`** | **`UnsupportedError`** | embedded native **`AVRoutePickerView`**（Platform View） |

Android の runtime capability 設計は [audio-output-android.md](audio-output-android.md) を変更 **しない**。

## 6. AudioOutputService API mapping（iOS — 確定）

具象: **`IOSAudioOutputService`**。**`AVRoutePickerView` の表示責務を Service へ押し込まない。**

### 6.1 Application Composition（iOS — Phase F 確定）

```text
NainaiApp（Composition Root）
  ├─ MediaKitPlaybackService        ← MediaPlaybackService のみ（Player 1 個）
  └─ IOSAudioOutputService          ← AudioOutputService（§6.2 のみ）
        ↓
  AudioOutputController(service: iosAudioOutputService)

Settings Presentation（iOS）
  └─ IOSAudioRoutePickerView        ← Platform View（Service 外）
```

| 原則 | 内容 |
|------|------|
| Player 数 | **1 個**。`IOSAudioOutputService` が **Player を所有しない** |
| 分離 | route picker **Presentation** と `MediaKitPlaybackService` を **分離** |
| Composition | platform 向け `AudioOutputService` を inject。iOS route picker UI は Presentation |

### 6.2 IOSAudioOutputService 責務（確定 — ここまで）

| 責務 | 内容 |
|------|------|
| `capabilities` | §4 固定値 |
| `availableDevices` / stream | 常に `[]` |
| `currentSelection` / stream | 常に `systemDefault()` |
| `selectSystemDefault()` | no-op 成功 |
| `selectDevice()` | `UnsupportedError` |
| `openSystemRoutePicker()` | `UnsupportedError` |

**含まない:** `AVRoutePickerView` 表示 / delegate / Platform View factory。

### 6.3 `openSystemRoutePicker()` が Unsupported である理由（確定）

[AVRoutePickerView](https://developer.apple.com/documentation/avkit/avroutepickerview) は **view hierarchy へ配置し、ユーザー自身が操作する control**。

| 禁止 | 内容 |
|------|------|
| hidden `AVRoutePickerView` | **禁止** |
| synthetic tap | **禁止** |
| private API | **禁止** |
| UIKit 内部 button 探索 | **禁止** |
| fake success / no-op open | **禁止** |

Settings UI は **`openSystemRoutePicker()` を呼ばない**。Flutter 独自 Button で open command を **発行しない**（§7）。

### 6.4 `currentSelection = systemDefault` の意味（Phase F 確定）

**「iPhone speaker を使っている」** という意味 **ではない**。

| 意味 | 内容 |
|------|------|
| Application 上の正 | **OS-managed route** — Audio route is managed by the operating system |
| OS routing | Bluetooth / AirPlay / speaker 等へ OS が route していても **`specificDevice` へ偽装しない** |
| 表示 | **「システム管理」**。「システム既定」と「現在 speaker」を **混同させない** |

Phase F では `AVAudioSession.currentRoute` を Settings 表示へ **直接バインドしない**。

## 7. Presentation 境界 — Platform View（Phase F 確定）

Route picker availability は **iOS Presentation capability**（Composition 上、iOS native route picker component を載せる）として扱う。

### 7.1 名称（Swift — 確定）

Service command adapter では **ない** 名称を用いる:

| クラス（概念） | 責務 |
|---------------|------|
| **`IOSAudioRoutePickerPlatformViewFactory`** | `FlutterPlatformViewFactory` — viewType 登録 |
| **`IOSAudioRoutePickerPlatformView`** | `AVRoutePickerView` 生成・embed・delegate |
| **`IOSAudioRoutePickerView`**（Dart Widget） | `UiKitView` wrapper |

**不採用名称:** `IOSAudioRoutePickerAdapter`（Service から picker を open する adapter に誤読されるため）。

### 7.2 Platform View 実装

| 項目 | 内容 |
|------|------|
| Flutter | `UiKitView(viewType: …)` |
| Native | `IOSAudioRoutePickerPlatformViewFactory` → `AVRoutePickerView` |
| Framework | **AVKit** |
| Language | **Swift** |
| MethodChannel（route open 用） | **作らない** — embedded picker のみで成立 |

```swift
let routePickerView = AVRoutePickerView()
routePickerView.delegate = coordinator
routePickerView.prioritizesVideoDevices = false
```

| 設定 | Phase F 値 | 根拠 |
|------|-----------|------|
| `prioritizesVideoDevices` | **`false`** | Settings「音声出力」は audio 優先。Apple API default も false。system routes の **fake filter 禁止** |

Platform View registration に必要な Flutter native boundary **のみ** 追加する。将来 `AVAudioSession` route observation が必要になった場合は **別 Phase で設計**。

### 7.3 AVRoutePickerViewDelegate（Phase F 確定）

| Callback | 用途 | 解釈禁止 |
|----------|------|----------|
| `routePickerViewWillBeginPresentingRoutes` | lifecycle / duplicate presentation 把握 | — |
| `routePickerViewDidEndPresentingRoutes` | present 終了 | **「route changed successfully」と解釈しない**。変更せず dismiss も **正常** |

`currentSelection` / Service state は **更新しない**。

Domain / Presentation へ `AVAudioSessionPortDescription` / UIKit 型を **露出しない**。

## 8. iOS Settings UI（確定）

### 8.1 レイアウト概念

```text
音声出力

現在の出力:
システム管理

[ native AVRoutePickerView ]

（必要なら補助文: 音声出力先を選択）
```

| 要素 | 内容 |
|------|------|
| Section title | `audioOutput` |
| 現在の出力 | `systemManaged` — OS-managed（§6.4） |
| Route picker | **`IOSAudioRoutePickerView` / Platform View** |
| 補助文 | `audioOutputSelectRoute` を **説明テキスト** として可 |
| Flutter open Button | **禁止** — `openSystemRoutePicker()` を呼ばない |
| Radio List | **禁止** |

### 8.2 表示条件

Platform View 行の表示は **`supportsSystemRoutePicker` では決めない**。

| 条件 | 内容 |
|------|------|
| iOS Settings Audio Section | Composition / Presentation 上、**iOS 向け native route picker component を載せる** |
| device 一覧行 | **常に非表示** |

### 8.3 Accessibility（Phase F 確定）

| 原則 | 内容 |
|------|------|
| Native a11y | `AVRoutePickerView` の OS 提供 accessibility を **壊さない** |
| 二重読み上げ防止 | Flutter 補助説明を付ける場合、**同一 control に重複 label を付けない** — VoiceOver 二重読み上げ **禁止** |
| 実機確認 | VoiceOver は **実装 Phase 必須** |

## 9. Error policy（Phase F 確定）

ユーザーが `AVRoutePickerView` を操作して route picker を開く処理は **native control 自身の責務**。

| 状況 | 扱い |
|------|------|
| Platform View **生成**失敗 | Settings 内 **Non-blocking error**（必要時） |
| native view **attach** 失敗 | 同上 |
| unexpected native integration failure | 同上 |
| ユーザー dismiss（route 未変更） | **error ではない** |
| route 変更 | Service error **ではない**。`currentSelection` も **更新しない** |
| `openSystemRoutePicker()` Service error | **iOS Phase F には存在しない**（呼び出し自体 Unsupported） |
| Playback Blocking Error | **禁止** |

## 10. AirPlay / 音声出力スコープ（Phase F 確定）

`AVRoutePickerView` が提示する AirPlay route を **許容** する。

| 項目 | 方針 |
|------|------|
| 範囲 | `AVRoutePickerView` が提供する **system route UX の範囲のみ** |
| 独自 casting | **実装しない** |
| `prioritizesVideoDevices` | **`false`**（§7.2）— video device 優先 sort **しない** |
| route fake filter | system routes を nainai 側で **偽装フィルタしない** |

## 11. Permissions / Info.plist（Phase F 確定）

route picker 表示のみを目的に、Phase F 設計上 **新規追加しない**:

| Key | Phase F |
|-----|---------|
| `NSBluetoothAlwaysUsageDescription` | **追加しない** |
| `NSLocalNetworkUsageDescription` | **追加しない** |
| `NSMicrophoneUsageDescription` | **追加しない** |

実装 Phase で実機確認。必要と判明したら Manifest 追加前に docs 再レビュー。

## 12. AVAudioSession / MPVolumeView 境界

### 12.1 AVAudioSession

Phase F route picker UI に **使用しない**。session 設定 / route **観測** は再生基盤または将来 Phase。

### 12.2 MPVolumeView（不採用）

WWDC19: route button には **`AVRoutePickerView`**。MPVolumeView は **volume UI のみ**。

## 13. Lifecycle（Phase F 確定）

| 項目 | 設計 |
|------|------|
| duplicate presentation | delegate + Platform View 内 guard。Service state は不変 |
| Settings 閉鎖 | Platform View dispose |
| background / foreground | popover は OS 管理 |
| dispose | stale delegate callback で Service / Controller を **書き換えない** |

## 14. Phase G / 共通化境界

### 14.1 共通化してよい

- `AudioOutputService` interface
- OS-managed selection（`systemDefault`）
- Settings Section 概念 UX
- Non-blocking error / Playback Blocking 禁止

### 14.2 共通化してはいけない

- AndroidX / Android SDK runtime capability
- Android `openSystemRoutePicker` MethodChannel
- iOS Presentation Platform View 実装詳細
- Permission セット

Phase F から Preference **保存しない**（Phase G）。iOS route API は stable persistent output device ID を **保証しない**。

## 15. Localization（キー案 — 未実装）

| キー | ja | en |
|------|----|----|
| `audioOutput` | 音声出力 | Audio Output |
| `systemManaged` | システム管理 | System managed |
| `audioOutputSelectRoute` | 音声出力先を選択 | Select audio output |
| `audioOutputRoutePickerFailed` | 音声出力の選択画面を表示できませんでした | Could not show the audio output picker |

## 16. テスト方針（実装 Phase 要求）

### 16.1 `IOSAudioOutputService`

| ケース |
|--------|
| `supportsDeviceEnumeration` → `false` |
| `supportsDirectDeviceSelection` → `false` |
| `supportsSystemRoutePicker` → **`false`** |
| `supportsPersistentDeviceId` → `false` |
| `availableDevices` 常に空 |
| `currentSelection` 常に `systemDefault` |
| `selectSystemDefault` → no-op 成功 |
| `selectDevice` → UnsupportedError |
| `openSystemRoutePicker` → **UnsupportedError** |

### 16.2 Widget / Platform View

| ケース |
|--------|
| iOS Settings に `AVRoutePickerView` component 表示 |
| Flutter **open command Button が存在しない** |
| `systemManaged` 表示 |
| native Platform View factory |
| `prioritizesVideoDevices == false` |
| delegate `willBegin` / `didEnd` |
| dismiss → error なし |
| attach failure → non-blocking error |
| accessibility（二重 label なし） |
| ja / en |

### 16.3 実機（必須）

| ケース |
|--------|
| iPhone speaker |
| Bluetooth audio |
| AirPlay（利用可能環境） |
| open / dismiss |
| route 変更 |
| playback 継続 |
| VoiceOver |

Simulator のみでは BT / AirPlay 完全検証 **不可**。

## 17. Phase 境界まとめ

| Phase | iOS 責務 |
|-------|----------|
| **F** | `IOSAudioOutputService` + Presentation Platform View（`AVRoutePickerView`） |
| **G** | persistence（Phase F では未着手） |

## 18. 状態

| 項目 | 状態 |
|------|------|
| Phase F 設計 | **Design Complete** |
| Phase F 実装 | **Implemented**（Native `25414c3` 等 + Settings wiring I-3B） |
| Settings Presentation Mode | `embeddedSystemRoutePicker`（I-3B） |
| Concrete iOS route technology | **Selected** — `AVRoutePickerView` + `UiKitView` |
| iOS native compile / `flutter build ios` | **Known / Pending verification**（Windows host 未検証 — [settings-shell.md](settings-shell.md) §19.2） |
| iOS 実機 route picker / Bluetooth / AirPods / car Bluetooth | **Pending Acceptance**（§19.5 同） |
| Phase I-3 baseline | client `dccf48f` — 526 PASS |
