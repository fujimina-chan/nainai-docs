# 音声出力デバイス選択（詳細設計）

**設定 → オーディオ → 音声出力デバイス** 機能の詳細設計正本。

| Phase | 内容 | client commit | 状態 |
|-------|------|---------------|------|
| **A** | 共通 Model / Service 抽象 / Capability | `4a34d85` | **Implemented** |
| **B** | Windows `MediaKitPlaybackService` が `AudioOutputService` も実装 | `c3239f3` | **Implemented** |

本ドキュメントは設計正本として実装状態も記録する。**Phase B 完了時点でも、ユーザーが Settings 画面から出力デバイスを選択できる状態ではない。** Settings UI・`AudioOutputController`・Application Composition wiring・hot unplug fallback・Android / iOS 実装・永続化は未実装。

関連:

- [Windows Settings UI（Phase C）](audio-output-settings.md)
- [UI Localization](localization.md)
- [メディア再生](media-playback.md)
- [Phase 2 UI](phase2-ui.md)
- [メディア技術選定](../architecture/media-technology.md)
- [ADR-0005 media_kit](../adr/0005-media-kit.md)

## 1. 概要

### 1.1 目的

ユーザーが **どこから音を出すか** を選択できるようにする。

| プラットフォーム | 方針 |
|------------------|------|
| Windows | media_kit 1.2.6 公開 API によるデバイス列挙・直接選択 |
| Android | OS 標準の出力選択 UX（Android platform output picker） |
| iOS | OS 標準の `AVRoutePickerView` によるルート選択 |

### 1.2 設計原則

**同一の物理デバイス一覧 UI をクロスプラットフォームで無理に共通化しない。**

共通化するのはユーザーの目的（出力先を選ぶ）と、ドメインモデル / Service 抽象 / Capability 判定までとする。実際の選択 UX は各 OS の能力に合わせる。

### 1.3 スコープ外

- Volume / Mute（[media-playback.md](media-playback.md) の既存設計を変更しない）
- OS 全体のシステム音量変更
- Equalizer / 空間オーディオ等の拡張
- 設定画面全体の Presentation 確定
- 設定永続化 library の選定
- Settings UI / `AudioOutputController` / Application Composition wiring（Phase C 以降）
- Android / iOS platform 実装（Phase E / F）
- hot unplug fallback（Phase D）
- 設定永続化（Phase G）

## 2. Windows 設計（確定候補）

### 2.1 採用 API

[media_kit](https://pub.dev/packages/media_kit) **1.2.6** の公開 API のみを使用する。private API および libmpv への直接 FFI は使用しない。

| API | 用途 |
|-----|------|
| `Player.state.audioDevices` / `Player.stream.audioDevices` | 利用可能デバイス一覧 |
| `Player.state.audioDevice` / `Player.stream.audioDevice` | 現在選択中デバイス |
| `Player.setAudioDevice(...)` | デバイス切替 |
| `AudioDevice.auto()` | システム既定 |

`AudioDevice` の構造（media_kit 1.2.6 確認済み）:

| フィールド | 意味 |
|------------|------|
| `name` | mpv `audio-device` プロパティに渡す識別子 |
| `description` | ユーザー向け表示名 |

`setAudioDevice` は内部で mpv プロパティ `audio-device` を設定する。`audio-device-list` 変更は Stream 経由で通知される。

### 2.2 設定 UI（概念）

```text
オーディオ

音声出力デバイス

○ システム既定
○ Speakers (...)
○ Headphones (...)
○ HDMI / Display Audio (...)
```

Presentation は radio / dropdown / selectable list 等、最終 Settings UI 設計と整合する方式を採用する。特定 Widget へは固定しない。

### 2.3 システム既定

| 項目 | 値 |
|------|-----|
| 内部 API | `AudioDevice.auto()`（`name = 'auto'`） |
| 永続化 `mode` | `systemDefault` |
| 保存する deviceId | **なし**（OS 既定デバイスの具体的 ID を保存しない） |

### 2.4 特定デバイス選択

| 項目 | 値 |
|------|-----|
| 永続化 `mode` | `specificDevice` |
| `deviceId` | `AudioDevice.name` |
| 表示名 | `AudioDevice.description` |

**`description` を識別キーにしない。** 同名デバイスが複数存在しうるため、`name` を正本 ID とする。

一覧表示では `description` が空の場合、`name` をフォールバック表示に用いてよい。

### 2.5 デバイス消失時のフォールバック

USB / Bluetooth / HDMI 等で選択中デバイスが一覧から消えた場合:

```text
specificDevice
    ↓ device missing
AudioDevice.auto()
    ↓
systemDefault（永続化も更新）
```

- 無効な `deviceId` を永続的に保持し続けない
- フォールバック後、Settings UI は「システム既定」選択状態を反映する

検知方法（実装候補）:

1. 保存 `deviceId` が `audioDevices` 一覧に存在しない
2. `setAudioDevice` 失敗後、一覧再取得でも該当 ID がない

### 2.6 再生中のデバイス切替

media_kit / mpv は audio output reload により切替可能なため、**再生中のデバイス変更を許可する** 設計候補とする。

期待動作:

| 項目 | 期待 |
|------|------|
| Player 再生成 | 不要 |
| media 再 open | 不要 |
| position | 維持 |
| pause | 必須ではない |
| audio glitch | 短時間発生しうる |

**実装後に Windows 実機検証を必須とする。** 本設計時点では media_kit ソースおよび mpv 挙動に基づく設計候補であり、全 Windows 環境での保証ではない。

### 2.7 Volume との関係

出力先変更時も [media-playback.md](media-playback.md) の `MediaState.volume` / `MediaKitVolumeMapper` / Mute 設計は変更しない。ユーザーが設定したアプリ内 Volume を維持する。

### 2.8 Phase B 実装（Windows）

Phase B 実装済み（client commit `c3239f3`）。Windows では `MediaKitPlaybackService` が `MediaPlaybackService` と `AudioOutputService` の両 interface を、**同一 `_player` 1 個** で実装する。Audio Output 専用 Player は追加生成しない。

#### Capability（実装値）

| Capability | 値 |
|------------|-----|
| `supportsDeviceEnumeration` | `true` |
| `supportsDirectDeviceSelection` | `true` |
| `supportsSystemRoutePicker` | `false` |
| `supportsPersistentDeviceId` | `true` |

#### デバイス列挙

media_kit 1.2.6 公開 API:

- `Player.state.audioDevices`
- `Player.stream.audioDevices`

Domain 変換:

| media_kit | Domain |
|-----------|--------|
| `AudioDevice.name` | `AudioOutputDevice.id` |
| `AudioDevice.description` | `AudioOutputDevice.label` |

`availableDevices` / `availableDevicesStream` から **`name == 'auto'` は除外** する。System Default は独立した Preference として扱う。

#### システム既定

media_kit `AudioDevice.auto()`（`name == 'auto'`）を System Default とする。

`currentSelection` / `currentSelectionStream` では `AudioOutputPreference.systemDefault()` へ変換する。

`selectSystemDefault()` は内部で `AudioDevice.auto()` を `setAudioDevice()` へ渡す。

#### 特定デバイス選択

`selectDevice(deviceId)` の確定動作:

1. `deviceId` を trim
2. trim 後が空文字なら `ArgumentError`
3. trim 後が `'auto'` なら `ArgumentError`（System Default は `selectSystemDefault()` を使用）
4. 現在の media_kit device 一覧から specific device を検索
5. 該当する実 `AudioDevice` を `setAudioDevice()` へ渡す
6. unknown `deviceId` は `ArgumentError`
7. System Default へ **勝手に fallback しない**

例:

- `selectDevice('auto')` → `ArgumentError`
- `selectDevice('  auto  ')` → trim 後 `ArgumentError`

成功時、`currentSelection` / `currentSelectionStream` は `AudioOutputPreference.specificDevice(...)` へ変換する。

#### System Route Picker

`supportsSystemRoutePicker == false`。`openSystemRoutePicker()` 呼び出しは `UnsupportedError`。

#### Stream / dispose

追加の manual subscription は持たない。`_player.stream.audioDevices` / `_player.stream.audioDevice` を Domain へ map して公開する。既存 Player ownership / dispose 仕様は変更していない。

## 3. Android 設計

### 3.1 方針

media_kit / mpv の `AudioDevice` API を、Bluetooth / 有線 / USB / 端末スピーカー等の **物理出力先一覧 UI として使用しない。**

Android では **Android platform output picker**（OS 標準のメディア出力選択 UX）を利用する方向とする。

### 3.2 重要: 単一 API への固定禁止

具体的な実装を `MediaRouter2.showSystemOutputSwitcher()` のみに固定しない。

| 理由 | 内容 |
|------|------|
| API level 差 | 利用可能 API が端末・OS バージョンで異なる |
| AndroidX MediaRouter | 第三者アプリ向けの標準ルート選択基盤 |
| MediaSession 統合 | 通知 / BT / 自動車等との連携状況で最適方式が変わる |

設計書上は **Android platform output picker** という抽象機能として記載し、実装時に minSdk / targetSdk / MediaSession 有無を踏まえて具体 API を選定する。

### 3.3 nainai-client 現状環境（read-only 調査）

調査日: 2026-08-30
調査対象: `E:\fyna\dev\nainai\nainai-client`（変更なし）

| 項目 | 値 | 根拠 |
|------|-----|------|
| minSdk | **24** | `android/app/build.gradle.kts` → `flutter.minSdkVersion`。Flutter 3.47.0 SDK デフォルト（`FlutterExtension.kt`） |
| targetSdk | **36** | 同上 → `flutter.targetSdkVersion` |
| compileSdk | **36** | 同上 → `flutter.compileSdkVersion` |
| MediaSession | **未使用** | コードベース・Manifest に該当参照なし |
| AndroidX MediaRouter | **未使用** | 依存・import なし |
| media3 (AndroidX Media3) | **未使用** | `pubspec.yaml` / Gradle 依存に該当なし |

補足:

- Flutter **3.47.0** / Dart **3.13.0**（[development-environment.md](../architecture/development-environment.md) 記載）
- 採用メディア: `media_kit` 1.2.6 / `media_kit_video` 2.0.1
- `MainActivity` は素の `FlutterActivity`（カスタム Android コードなし）

### 3.4 UI（概念）

```text
音声出力デバイス

現在の出力:
システム管理

[出力先を変更]
```

- ボタン操作で OS 標準の出力選択 UI を表示または遷移する
- Windows のような独自 device list は表示しない
- 「現在の出力」表示は OS が提供する情報を反映する。取得不能時は「システム管理」等の抽象表示とする

### 3.5 永続化

Android では物理 device ID の永続保存を **前提にしない。**

| 保存 | 方針 |
|------|------|
| `mode` | 原則 `systemDefault`（OS 管理）のみ |
| `deviceId` / `deviceLabel` | 保存しない |

OS がルーティングを管理するため、アプリ再起動後もユーザー OS 設定に従う。

## 4. iOS 設計

### 4.1 方針

OS 標準の **`AVRoutePickerView`** を利用する方向とする。

- アプリ独自に Bluetooth / AirPlay 等の device ID 一覧を構築しない
- 安定 device ID の永続保存を前提にしない

### 4.2 UI（概念）

```text
音声出力

[AirPlay / 出力先を選択]
```

`AVRoutePickerView` を Flutter UI へ組み込む方式は **未確定**。設計候補:

| 候補 | 概要 | 備考 |
|------|------|------|
| Platform View | Flutter `UiKitView` で `AVRoutePickerView` を埋め込む | 定番。Flutter 3.47 で利用可能 |
| Native view embedding | 同上だが専用 Plugin / MethodChannel でラップ | 再利用・テスト容易 |
| カスタム Platform Channel + モーダル | ネイティブ側で picker を表示 | Platform View 不要だが UX 差異あり |

**推奨候補: Platform View（`UiKitView`）** — Apple 提供 UI をそのまま使え、Settings 行内への配置が自然。最終方式は iOS 実機検証後に確定する。

### 4.3 永続化

Android と同様、物理 device ID の永続保存を前提にしない。`mode = systemDefault`（OS 管理）を基本とする。

## 5. 共通ドメインモデル

Package 非依存のドメインモデル。**Phase A 実装済み**（`lib/features/settings/models/`）。

```dart
enum AudioOutputSelectionMode {
  systemDefault,
  specificDevice,
}
```

```dart
class AudioOutputPreference {
  // factory AudioOutputPreference.systemDefault()
  // factory AudioOutputPreference.specificDevice({ required deviceId, deviceLabel? })
}
```

### 5.1 Preference ルール（Phase A 確定）

| mode | deviceId | deviceLabel |
|------|----------|-------------|
| `systemDefault` | `null` | `null` |
| `specificDevice` | **必須**（trim 後 non-empty） | 任意（表示キャッシュ） |

- `specificDevice` 作成時、`deviceId` の前後空白は **trim して保存**
- trim 後が空文字の `deviceId` は **禁止**（`ArgumentError`）
- 復元時は `deviceId` を正本とし、一覧から label を再解決する。再解決不能ならフォールバック（§2.5）

| フィールド | Windows（Phase B） | Android / iOS |
|------------|-------------------|---------------|
| `mode` | `systemDefault` / `specificDevice` | 原則 `systemDefault` |
| `deviceId` | `AudioDevice.name` にマップ | 通常 `null` |
| `deviceLabel` | 表示用キャッシュ | 通常 `null` |

### 5.2 AudioOutputDevice（Phase A 確定）

```dart
class AudioOutputDevice {
  final String id;    // identity key
  final String label; // display metadata only
}
```

- **identity は `id` のみ**。`==` / `hashCode` も `id` 基準
- 同一 `id` で `label` が変わっても同一 device として扱う
- Windows（Phase B）では `id` は media_kit `AudioDevice.name` にマップする

## 6. Platform Capability

UI がプラットフォーム名を直接 `if` 分岐するより、Capability に基づいて表示を切り替える。**Phase A 実装済み**（`AudioOutputCapabilities`）。

```dart
class AudioOutputCapabilities {
  final bool supportsDeviceEnumeration;
  final bool supportsDirectDeviceSelection;
  final bool supportsSystemRoutePicker;
  final bool supportsPersistentDeviceId;
}
```

| Capability | Windows（Phase B 実装） | Android | iOS |
|------------|-------------------------|---------|-----|
| `supportsDeviceEnumeration` | true | false | false |
| `supportsDirectDeviceSelection` | true | false | false |
| `supportsSystemRoutePicker` | false | true | true |
| `supportsPersistentDeviceId` | true | false | false |

Service / UI は Capability を参照し、未サポート method 呼び出しを避ける。

## 7. Service 境界

### 7.1 AudioOutputService（抽象）

`MediaPlaybackService` とは **分離** する。**Phase A 実装済み**（`lib/features/settings/services/audio_output_service.dart`）。

```dart
abstract class AudioOutputService {
  AudioOutputCapabilities get capabilities;

  List<AudioOutputDevice> get availableDevices;
  Stream<List<AudioOutputDevice>> get availableDevicesStream;

  AudioOutputPreference get currentSelection;
  Stream<AudioOutputPreference> get currentSelectionStream;

  Future<void> selectSystemDefault();
  Future<void> selectDevice(String deviceId);

  /// Android / iOS。Windows では Unsupported。
  Future<void> openSystemRoutePicker();
}
```

**snapshot + stream の対称構造** を採用（`availableDevices` / `availableDevicesStream`、`currentSelection` / `currentSelectionStream`）。

全 platform がすべての method を実装できる前提にしない。未サポートは `UnsupportedError` または no-op + Capability で UI 側が非表示とする。

### 7.1.1 Phase A 依存境界

Phase A 共通層は次へ **依存しない**:

- `media_kit`
- Flutter UI（`BuildContext` 等）
- Localization
- Platform 判定（`dart:io` 等）

`MediaController` への追加もなし。Persistence も未実装。

Phase B（Windows）では `MediaKitPlaybackService` が抽象 `AudioOutputService` を同一 `_player` で実装する。Android / iOS の platform 具象実装は Phase E / F 以降。

### 7.2 MediaPlaybackService との関係 — 3 案比較

**制約: Player を二重生成してはならない。**

現状 `MediaKitPlaybackService` は単一 `Player` を service lifetime で保持する。Application 層へ raw な `media_kit.Player` を露出しない。

| 案 | 概要 | 長所 | 短所 |
|----|------|------|------|
| A | `MediaKitPlaybackService` が `MediaPlaybackService` と `AudioOutputService` の両 interface を実装 | Player 所有が 1 箇所。Application wiring が単純。package 型を Application へ漏らさない | 1 具象クラスが 2 抽象を実装（内部責務は分離してよい） |
| B | Application 層で `Player` を生成し、別 Service 2 つへ注入 | 具象 Service を物理分割できる | Application へ raw `Player` が露出。共有 wiring・dispose 境界が複雑 |
| C | 別 `MediaKitAudioOutputService` が内部 Player access を限定的共有 | B より Application 露出は少ない | 「限定的」の境界が曖昧になりやすい |

#### 推奨（Windows）: 案 A — 単一 `MediaKitPlaybackService` が両 interface を実装（Phase B 実装済み）

抽象 interface は `MediaPlaybackService` と `AudioOutputService` に分離する。Windows では 1 つの `MediaKitPlaybackService` が両方を実装する（Phase B、`c3239f3`）。

```text
MediaPlaybackService          AudioOutputService
        ▲                            ▲
        └── MediaKitPlaybackService ─┘
                       │
                    Player
```

- `Player` は `MediaKitPlaybackService` が引き続き所有する
- Application 層へ raw な `media_kit.Player` を露出しない
- Controller 側は抽象 interface のみを受け取る（[media-playback.md](media-playback.md) の package 非露出方針と整合）

案 B（Application 層 Player 注入）は第一推奨としない。Player 所有・dispose 責務が Application に分散し、package 型が Application 境界へ漏れる。

案 C は案 B の変形に過ぎ、公開境界として A を正とする。

#### Mobile（Android / iOS）

Windows と同一具象実装を強制しない。

```text
MediaController
  → MediaKitPlaybackService（MediaPlaybackService として）

AudioOutputController
  → プラットフォーム固有 AudioOutputService
```

Mobile の `AudioOutputService` 実装は media_kit `Player` 非依存となりうる（OS route picker 利用）。

## 8. Controller 設計

音声出力設定は **Media playback state ではなく Settings 領域** に属する。

**`MediaController` へ device preference を直接追加しない。**

Phase C（Windows Settings UI）の詳細設計 — `AudioOutputController` / `AudioOutputState` / UI / Error / Test — は [audio-output-settings.md](audio-output-settings.md) を正とする。

### 8.1 候補比較

| 案 | 概要 | 長所 | 短所 |
|----|------|------|------|
| `SettingsController` 配下 | 設定全体の ChangeNotifier が AudioOutput を保持 | 設定画面拡張に自然 | Settings 全体未実装 |
| 独立 `AudioOutputController` | 音声出力専用 ChangeNotifier | 単機能でテスト容易。MediaController と独立 | Settings 導入時に統合 wiring が必要 |

#### 推奨: 独立 `AudioOutputController`（Phase C で実装予定）

[audio-output-settings.md](audio-output-settings.md) §4–§6 に Phase C 確定設計あり。将来 Settings 画面実装時に `SettingsController` が compose する。最終名称は未確定（§13 参照）。

## 9. Application Composition

### 9.1 現状（Phase B — Windows Service のみ）

```text
NainaiApp
  ├─ FileSelectorMediaSelectionService
  ├─ MediaKitPlaybackService()   ← MediaPlaybackService + AudioOutputService（同一 _player）
  ├─ MediaController            ← MediaPlaybackService として注入
  └─ MediaScreen
```

- `Player` は `MediaKitPlaybackService` 内部で生成・所有する
- Windows では `MediaKitPlaybackService` が `AudioOutputService` も実装済み（Phase B）
- **`AudioOutputController` / Settings UI / Application Composition wiring は未実装** — Service API は存在するが、ユーザー向け Settings からは選択できない

### 9.2 Phase C 目標構成（Windows — 設計確定）

[audio-output-settings.md](audio-output-settings.md) §6。`AudioOutputController` wiring と Settings Screen を含む。

#### Mobile（Android / iOS — Phase E / F）

```text
NainaiApp
  ├─ MediaKitPlaybackService
  │     └─ MediaController          ← MediaPlaybackService
  ├─ PlatformAudioOutputService
  │     └─ AudioOutputController    ← AudioOutputService
  ├─ FileSelectorMediaSelectionService
  ├─ MediaScreen
  └─ SettingsScreen (future)
```

Windows と Mobile で具象 Service の組み合わせは異なってよい。抽象 interface は共通。

`MediaKit.ensureInitialized()` は既存 [main.dart](https://github.com) パターンを維持。

### 9.3 dispose ownership

**Current（Phase B 実装済み）:** `MediaController.dispose()` が注入 `MediaPlaybackService` を dispose する。詳細は [media-playback.md](media-playback.md) §12、[media-technology.md](../architecture/media-technology.md) §Application Composition。

**Phase C 以降（設計確定・未実装）:** 共有 `MediaKitPlaybackService` の唯一 owner は `NainaiApp`。`MediaController` / `AudioOutputController` は Service を dispose しない。`MediaController` からの `MediaPlaybackService.dispose()` 呼び出しを **除去** する。詳細は [audio-output-settings.md](audio-output-settings.md) §6。

**共通原則（Current / Phase C 以降）:**

**共有インスタンスを複数 Controller がそれぞれ dispose する設計は禁止する。**

| 対象 | 原則 |
|------|------|
| `Player` | `MediaKitPlaybackService` が所有。Controller から dispose しない |
| concrete service | Phase C 以降: **`NainaiApp` が 1 箇所** で dispose。Current: `MediaController` 経由 |
| Controller | 自身の StreamSubscription 等のみ dispose |

Windows では `MediaKitPlaybackService` 1 インスタンスが両 interface を提供するため、Service dispose も **1 回** で足りる。

Mobile では `MediaKitPlaybackService` と `PlatformAudioOutputService` が別インスタンスとなりうる。その場合も **各 concrete service の dispose 責務は Composition Root 1 箇所** に集約し、Controller からの二重 dispose を禁止する。

### 9.4 起動時復元手順

```text
設定ロード（将来の Persistence 層）
    ↓
AudioOutputService 初期化
    ↓
利用可能デバイス確認（Windows）
    ↓
保存 deviceId が一覧に存在?
    ├─ Yes → selectDevice(deviceId)
    └─ No  → selectSystemDefault() + 永続化更新
    ↓
MediaController / 初回 load 可能
```

#### Player 生成前後の適用タイミング

| タイミング | 内容 |
|------------|------|
| Player 生成直後 | `MediaKitPlaybackService` 構築（Windows では AudioOutput 能力も内包） |
| 初回 `open` 前（推奨） | 保存 preference を `setAudioDevice` で適用 |
| 初回 `open` 後 | デバイス消失検知・フォールバック（§2.5） |

media_kit では `audio-device` / `audio-device-list` は Player 初期化前後で取得可能（ソースコメント: playback lifecycle 外プロパティ）。**初回 media load 前の適用を推奨** するが、load 後の切替も設計上許容（§2.6）。

Android / iOS は起動時に device ID 復元を行わない。

## 10. エラー設計

Playback Blocking Error（[media-playback.md](media-playback.md) §13）へ統合 **しない。**

音声出力は Settings 操作上の問題であり、メディア再生不能（`status = error`）とは別扱いとする。

### 10.1 エラー種別

| 種別 | 例 | 推奨扱い |
|------|-----|----------|
| デバイス列挙失敗 | `audioDevices` 取得不可 | **Non-blocking**。Settings UI に警告。再生は継続 |
| デバイス切替失敗 | `setAudioDevice` 例外 | **Non-blocking**。選択 UI にインラインエラー。前回成功状態を表示 |
| 選択中デバイス消失 | 保存 ID が一覧にない | Phase D: **自動フォールバック**（§2.5）。Phase C: Non-blocking error |
| OS route picker 表示失敗 | Android / iOS picker 起動不可 | **Non-blocking**。ボタン付近にエラー表示 |

### 10.2 UI フィードバック

[media-playback.md](media-playback.md) の Non-blocking Error パターン（Notification / Banner、Semantic Warning）を **Settings 文脈** で流用する。

- Media 画面を Error 画面へ置き換えない
- 技術的 Exception 文字列をユーザーへ表示しない
- 列挙失敗時は Windows 設定で「システム既定」のみ選択可能にフォールバック

### 10.3 エラーモデル（候補）

```dart
enum AudioOutputErrorType {
  enumerationFailed,
  selectionFailed,
  routePickerFailed,
  deviceUnavailable,
}
```

`AudioOutputController` が Settings 向け error state を保持。`MediaError` とは型を共有しない。

## 11. Localization

[localization.md](localization.md) の ARB 基盤を使用する。**日本語直書きは禁止。**

### 11.1 Settings 向け翻訳キー候補（未実装）

Phase C 詳細キー案は [audio-output-settings.md](audio-output-settings.md) §16。以下は Android / iOS 含む全体向け候補。

| キー | 用途 | 例（ja 参考） |
|------|------|---------------|
| `settingsTitle` | 設定画面 | 設定 |
| `audioSettingsSectionTitle` | オーディオ節 | オーディオ |
| `audioOutputDeviceTitle` | 項目見出し | 音声出力デバイス |
| `systemDefault` | Windows 選択肢 | システム既定 |
| `currentAudioOutput` | Android 現在表示ラベル | 現在の出力 |
| `systemManaged` | Android / iOS 状態 | システム管理 |
| `changeAudioOutput` | ボタン | 出力先を変更 |
| `selectAudioRoute` | iOS ボタン | AirPlay / 出力先を選択 |
| `audioOutputEnumerationFailed` | 列挙失敗 | 出力デバイスを取得できませんでした |
| `audioOutputSelectionFailed` | 切替失敗 | 出力デバイスを変更できませんでした |
| `audioOutputDeviceUnavailable` | 消失通知 | 選択していた出力デバイスが利用できなくなりました |
| `audioRoutePickerFailed` | picker 失敗 | 出力先の選択画面を表示できませんでした |

実際の Localization 実装レーン（[localization.md](localization.md)）と最終統合時にキー名を照合する。

## 12. 実装フェーズ分割

```text
Phase A ──▶ Phase B ──▶ Phase C
              │
              └──▶ Phase D

Phase A ──▶ Phase E (Android)
Phase A ──▶ Phase F (iOS)

Phase A ──▶ Phase G (永続化)
Phase B + G ──▶ Windows 起動時復元完成
```

| Phase | 内容 | 状態 | 依存 |
|-------|------|------|------|
| **A** | 共通 Model / `AudioOutputService` 抽象 / Capability | **Implemented** | なし |
| **B** | Windows `MediaKitPlaybackService` が `AudioOutputService` も実装 | **Implemented** | A |
| **C** | Windows Settings UI + `AudioOutputController` + Composition wiring | **Design Complete**（Launcher: **Launcher placement design pending**） | A, B |
| **D** | Windows hot unplug / fallback | 未実装 | B |
| **E** | Android platform output picker | 未実装 | A |
| **F** | iOS `AVRoutePickerView` 組み込み | 未実装 | A |
| **G** | 設定永続化（`AudioOutputPreference`） | 未実装 | A |

### 並行可能範囲

| 並行 | 内容 |
|------|------|
| B + E + F | A 完了後、プラットフォーム実装は独立 |
| C + D | B 完了後、UI と fallback は部分並行可 |
| G | A 完了後いつでも開始。B と組み合わせで Windows 復元完成 |

### Phase A 検証（client）

Phase A は Localization 統合後の nainai-client 上で次を確認済み（2026-08-30 時点）:

| コマンド | 結果 |
|----------|------|
| `flutter analyze` | No issues found |
| `flutter test` | 145 tests PASS |

client commit: `4a34d85` — `feat: 音声出力デバイスの共通基盤を追加`

### Phase B 検証（client）

Phase B は client main `c3239f3` 上で次を確認済み:

| コマンド | 結果 |
|----------|------|
| `flutter analyze` | No issues found |
| `flutter test` | 165 tests PASS |

client commit: `c3239f3` — `feat: Windows音声出力デバイス切り替え基盤を実装`

## 13. 未確定事項

本設計で **勝手に確定しない** 項目:

| 項目 | 備考 |
|------|------|
| Android native 実装詳細 | MediaRouter2 / AndroidX / MediaSession 等の具体選定 |
| Android API ごとの fallback | minSdk 24 を前提に実装時決定 |
| iOS Flutter embedding 方式 | Platform View 推奨候補だが未確定 |
| 設定永続化 library | shared_preferences 等は未選定 |
| 設定画面全体デザイン | Phase 2 UI 外 |
| `AudioOutputController` / `SettingsController` 最終名称 | Settings 全体設計に依存 |
| Windows 再生中切替の実機保証 | 実装後検証必須 |

## 14. 将来機能との境界

本機能に含めない:

- 入力デバイス（マイク）選択
- 複数 Player / 同時再生
- OS システム音量制御
- 出力先ごとの Volume 記憶
- サラウンド / チャンネル設定

[media-playback.md](media-playback.md) §17 の将来機能と競合しないこと。単一 Player モデル・Volume 独立設計を維持する。
