# Audio Output Application Composition（Phase H 詳細設計）

Audio Output **Phase H** — Phase A〜G で確定した Audio Output 機能を **NainaiApp / Application Composition** へ統合する方法の実装前詳細設計正本。

関連:

- [音声出力デバイス選択](audio-output.md)
- [Windows Settings UI（Phase C）](audio-output-settings.md)
- [Windows hot unplug / fallback（Phase D）](audio-output-hot-unplug.md)
- [Android System Route Picker（Phase E）](audio-output-android.md)
- [iOS System Route Picker（Phase F）](audio-output-ios.md)
- [Preference 永続化（Phase G）](audio-output-persistence.md)
- [Common Settings Concrete Persistence](settings-persistence.md)

**client 参照:** `dccf48f` — Production Composition **Implemented**（Phase H `f280a11` + I-3B Settings wiring）。詳細結果: [settings-shell.md](settings-shell.md) §19。

**本書のスコープ:** platform 別 object graph / lifecycle / startup・dispose ordering / command・state route / owner・borrower。**Settings Shell UI 設計は扱わない**（[settings-shell.md](settings-shell.md) は別レーン — §15 で Presentation 契約の整合方針のみ記載）。

## 1. 位置づけ

| Phase | 内容 | 状態 |
|-------|------|------|
| **H** | **Cross-platform Application Composition**（本書） | **Design Complete** / **Implemented**（`f280a11` + I-3B） |

## 2. Phase G 最終依存（維持）

```text
AudioOutputPreferenceCoordinator
        |
        +--> AudioOutputController              (borrow — runtime facade)
        |       |
        |       +--> AudioOutputService         (Controller が borrow)
        |
        +--> AudioOutputPreferenceRepository

禁止:
  AudioOutputController → AudioOutputPreferenceCoordinator
  AudioOutputService → AudioOutputPreferenceRepository
  AudioOutputPreferenceCoordinator → AudioOutputService   (直接 borrow 禁止)
```

Coordinator は readiness 待機を含め **Controller のみ** を Application facade として利用する（[audio-output-persistence.md](audio-output-persistence.md) §4）。

## 3. Platform Composition（正式 — Service + Presentation Mode）

### 3.1 原則

| 原則 | 内容 |
|------|------|
| 決定場所 | **`NainaiApp`（Composition Root）のみ** |
| 決定単位 | **`AudioOutputService` + Settings Presentation Mode** を **同時** に確定 |
| Player 数 | **全 platform で 1 個** |
| 禁止 | Service capability だけで Presentation UI を決定すること（§3.4） |
| 禁止 | Presentation Widget 内部での `Platform.is*` 乱用 |

**Service 選択だけでは Presentation を決められない。** 例: iOS は全 capability が `false` だが Settings には **embedded `AVRoutePickerView`** を表示する必要がある。

### 3.2 `AudioOutputPlatformComposition`（概念）

Composition Root が **1 箇所** で生成する bundle。exact Dart class 名は実装 Phase で調整可。**責務は固定**:

```dart
// 概念
class AudioOutputPlatformComposition {
  const AudioOutputPlatformComposition({
    required this.audioOutputService,
    required this.settingsPresentationMode,
  });

  final AudioOutputService audioOutputService;
  final AudioOutputSettingsPresentationMode settingsPresentationMode;
}
```

```text
AudioOutputPlatformComposition
├── audioOutputService          ← platform AudioOutputService
└── settingsPresentationMode    ← Settings UI composition 方式
```

### 3.3 Settings Presentation Mode（正式 — 3 種）

| Mode | Platform | 意味 |
|------|----------|------|
| **`deviceList`** | **Windows** | System Default + specific device **Radio List** |
| **`systemRoutePickerCommand`** | **Android** | OS-managed route + **Controller command** で System Output Switcher を開く |
| **`embeddedSystemRoutePicker`** | **iOS** | OS-managed route + **`IOSAudioRoutePickerView` / `AVRoutePickerView`** embed |

enum / sealed class 等の具体表現は実装 Phase で調整可。**3 意味は固定**。

### 3.4 Capability と Presentation Mode の分離（禁止事項）

| 概念 | 責務 |
|------|------|
| **`AudioOutputCapabilities`** | **Service command 能力** — enumeration / direct select / programmatic route open / persistent ID |
| **`AudioOutputSettingsPresentationMode`** | **Settings UI composition 方式** |

**禁止:**

- `supportsSystemRoutePicker == false` だから iOS picker **非表示**
- `supportsDeviceEnumeration == false` だから Android / iOS **同一 UI**
- Presentation Mode を capability bool で **代用**
- iOS 向けに **新 Domain capability bool** を追加（[audio-output-ios.md](audio-output-ios.md) §1.3 維持）

### 3.5 Composition Root factory（概念）

`Platform.isWindows` / `Platform.isAndroid` / `Platform.isIOS` 等の platform 判定は **ここに集約**。Presentation へ platform 型を **露出しない**。

```dart
// 概念 — NainaiApp._NainaiAppState
late final MediaKitPlaybackService _playbackService;
late final AudioOutputPlatformComposition _audioOutputComposition;

@override
void initState() {
  super.initState();
  _playbackService = MediaKitPlaybackService();
  _audioOutputComposition = _createAudioOutputPlatformComposition(_playbackService);
  // audioOutputService = _audioOutputComposition.audioOutputService
  // ...
}

AudioOutputPlatformComposition _createAudioOutputPlatformComposition(
  MediaKitPlaybackService playback,
) {
  if (Platform.isAndroid) {
    return AudioOutputPlatformComposition(
      audioOutputService: AndroidAudioOutputService(),
      settingsPresentationMode: AudioOutputSettingsPresentationMode.systemRoutePickerCommand,
    );
  }
  if (Platform.isIOS) {
    return AudioOutputPlatformComposition(
      audioOutputService: IOSAudioOutputService(),
      settingsPresentationMode: AudioOutputSettingsPresentationMode.embeddedSystemRoutePicker,
    );
  }
  // Windows / desktop
  return AudioOutputPlatformComposition(
    audioOutputService: playback,
    settingsPresentationMode: AudioOutputSettingsPresentationMode.deviceList,
  );
}
```

### 3.6 Platform 別 Composition 値（正式）

| Platform | `audioOutputService` | `settingsPresentationMode` | Player owner |
|----------|---------------------|---------------------------|--------------|
| **Windows** | **同一** `MediaKitPlaybackService` | `deviceList` | `_playbackService` |
| **Linux / macOS**（将来） | **同一** `_playbackService` | `deviceList` | `_playbackService` |
| **Android** | `AndroidAudioOutputService`（別 instance） | `systemRoutePickerCommand` | `_playbackService` **のみ** |
| **iOS** | `IOSAudioOutputService`（別 instance） | `embeddedSystemRoutePicker` | `_playbackService` **のみ** |

## 4. Platform 別 Object Graph（正式）

### 4.1 Windows

```text
NainaiApp
├── SharedPreferencesAsync()
├── MediaKitPlaybackService              ← Player 1 / dual interface owner
├── AudioOutputPlatformComposition
│     ├─ audioOutputService = playback (同一)
│     └─ settingsPresentationMode = deviceList
├── MediaController(playback)
├── AudioOutputController(composition.audioOutputService)
├── SharedPreferencesAudioOutputPreferenceRepository
├── AudioOutputPreferenceCoordinator(repository, controller)
├── MediaKitVideoSurface(playback.videoController)
└── MaterialApp
      └─ AudioOutputNotificationHost(controller)
            └─ MediaScreen + Settings route
                  └─ AudioOutputSettingsSection
                        ├─ controller              (state)
                        ├─ selectionCommands       (Coordinator)
                        └─ mode = deviceList
```

### 4.2 Android

```text
NainaiApp
├── MediaKitPlaybackService              ← Player 1 / playback only
├── AndroidAudioOutputService            ← AudioOutputService（Player 非所有）
├── AudioOutputPlatformComposition
│     ├─ audioOutputService = AndroidAudioOutputService
│     └─ settingsPresentationMode = systemRoutePickerCommand
├── MediaController(playback)
├── AudioOutputController(androidAudioOutputService)
├── Coordinator + Repository
└─ Settings Section
      ├─ controller              (state)
      ├─ routePickerCommands     (Controller — SystemRoutePickerCommands)
      └─ mode = systemRoutePickerCommand
            ↓ openSystemRoutePicker
      AndroidAudioOutputService → MethodChannel → native System Output Switcher
```

Native `AndroidAudioRoutePickerAdapter` / Channel handler 登録は **FlutterEngine / MainActivity lifecycle** に属する（§10.4）。Dart `AndroidAudioOutputService.dispose()` で handler 解除する **架空設計は禁止**。

### 4.3 iOS

```text
NainaiApp
├── MediaKitPlaybackService              ← Player 1 / playback only
├── IOSAudioOutputService                ← capabilities / no-op commands（Platform View 非所有）
├── AudioOutputPlatformComposition
│     ├─ audioOutputService = IOSAudioOutputService
│     └─ settingsPresentationMode = embeddedSystemRoutePicker
├── MediaController(playback)
├── AudioOutputController(iosAudioOutputService)
├── Coordinator + Repository（restore skip）
└─ Settings Section
      ├─ controller              (state)
      ├─ mode = embeddedSystemRoutePicker
      └─ IOSAudioRoutePickerView (Presentation — UiKitView → AVRoutePickerView)
```

| 項目 | 正式 |
|------|------|
| `SelectionCommands` | **使用しない** — direct device select なし |
| `SystemRoutePickerCommands` | **使用しない** — programmatic open なし |
| `openSystemRoutePicker()` | **Unsupported**（Phase F 維持） |
| capability 全 false | **embedded picker は Presentation Mode で決定** — capability 非表示 **禁止** |

## 5. Command Contracts（正式）

### 5.1 Persist-eligible — `AudioOutputSelectionCommands`

Presentation が Concrete Coordinator へ直接強く依存しない **小 interface** を **正式採用**:

```dart
abstract interface class AudioOutputSelectionCommands {
  Future<void> selectSystemDefault();
  Future<void> selectDevice(String deviceId);
}
```

`AudioOutputPreferenceCoordinator` が実装。exact naming は実装 Phase で調整可。

**Windows 正式経路:**

```text
Presentation
    ↓
AudioOutputSelectionCommands
    ↓
AudioOutputPreferenceCoordinator
    ↓
AudioOutputController
    ↓
AudioOutputService
    ↓ success
Coordinator → PreferenceRepository.save()
```

**Android / iOS:** `AudioOutputSelectionCommands` を Settings Section へ **注入しない**（persist-eligible user select なし）。

### 5.2 Runtime-only — `SystemRoutePickerCommands`

OS-managed route picker。**persist 対象外**（Phase E — fake device ID 禁止）。

```dart
abstract interface class SystemRoutePickerCommands {
  Future<void> openSystemRoutePicker();
}
```

**提供元:** `AudioOutputController` が contract を満たす、または Controller への **薄い Application adapter**。新 Service wrapper **乱立禁止**。

**Android 正式経路:**

```text
Presentation
    ↓
SystemRoutePickerCommands
    ↓
AudioOutputController
    ↓
AndroidAudioOutputService
    ↓ MethodChannel
native System Output Switcher
```

**Coordinator 非経由。** route picker 結果から Preference save **禁止**。

### 5.3 iOS — command なし

```text
Presentation
    ↓
IOSAudioRoutePickerView
    ↓ UiKitView
AVRoutePickerView
    ↓ user tap
System route popover (OS)
```

`SelectionCommands` / `SystemRoutePickerCommands` / `openSystemRoutePicker()` — **いずれも使用しない**。

### 5.4 State path（全 platform）

```text
AudioOutputController
    ↓  AudioOutputState
Presentation
```

Runtime state を Coordinator へ **複製しない**。

Persistence state / error（`preferredDeviceUnavailable`、save/load/restore error 等）は **Coordinator 正本** — Settings Section が別途 listen。

## 6. Settings Shell との関係（参照 — 本 branch では変更しない）

別 branch `work/settings-shell-design` @ `f618ac3` では:

- **state:** `AudioOutputController`
- **commands:** Coordinator / command abstraction

Phase H はこれを **具体化**:

| Platform | state | commands | Presentation |
|----------|-------|----------|----------------|
| **Windows** | Controller | `AudioOutputSelectionCommands` | `deviceList` |
| **Android** | Controller | `SystemRoutePickerCommands` | `systemRoutePickerCommand` |
| **iOS** | Controller | — | `embeddedSystemRoutePicker` + `IOSAudioRoutePickerView` |

**[settings-shell.md](settings-shell.md) の文言修正は統合工程で別途。** 本 branch では **触れない**。

## 7. Controller / Coordinator Lifetime

### 7.1 AudioOutputController

| 項目 | 正式 |
|------|------|
| 生成 | `NainaiApp.initState` — **Application lifetime で 1 個** |
| 禁止 | `build()` ごとの再生成 |
| inject | `composition.audioOutputService` |
| readiness | `initialAvailableDevices` — Service **pass-through** |
| `SystemRoutePickerCommands` | Android のみ — Controller が提供 |
| Persistence | **非依存** |

### 7.2 AudioOutputPreferenceCoordinator

| 項目 | 正式 |
|------|------|
| 生成 | App lifetime **1 個** |
| コンストラクタ | `(repository, controller)` — Service **渡さない** |
| `initialize()` | **1 回** — non-blocking（§8） |
| `AudioOutputSelectionCommands` | **実装** |

### 7.3 initialAvailableDevices

Coordinator は **`controller.initialAvailableDevices` のみ**（Phase G-1 整合）。Service 直接 borrow **禁止**。

| Platform | 挙動 |
|----------|------|
| **Windows** | startup restore 時 await |
| **Android / iOS** | 即 `[]` complete — `supportsPersistentDeviceId == false` で restore skip |

## 8. Startup Ordering（正式）

### 8.1 `main()` — 変更なし

```dart
void main() {
  WidgetsFlutterBinding.ensureInitialized();
  MediaKit.ensureInitialized();
  runApp(const NainaiApp());
}
```

### 8.2 `NainaiApp.initState` — 正式生成順序

```text
1.  SharedPreferencesAsync()
2.  MediaKitPlaybackService()
3.  AudioOutputPlatformComposition
       ├─ audioOutputService
       └─ settingsPresentationMode
4.  MediaController(playbackService)
5.  AudioOutputController(composition.audioOutputService)
6.  SharedPreferencesAudioOutputPreferenceRepository(prefs)
7.  AudioOutputPreferenceCoordinator(repository, controller)
8.  MediaKitVideoSurface(playback.videoController)
9.  unawaited(coordinator.initialize())
```

### 8.3 `unawaited(initialize())` — error 契約（正式）

| 項目 | 正式 |
|------|------|
| `unawaited` の意味 | UI startup を **await しない** のみ — **「エラーを無視する」意味ではない** |
| operational failure | Coordinator **内部** で state/error へ変換 — **Non-blocking** |
| Future から throw | 想定可能な運用 failure を **そのまま throw しない** |
| Unhandled Future error | 想定 operational failure が **漏れない** 契約 |

**initialize 内で内部処理する failure（例）:**

- Preference read failure
- invalid persisted payload
- initial device readiness failure
- restore device missing
- restore selection failure
- dispose / stale restore

**区別:** Assertion / programming bug 等の **unexpected programmer error** まで握り潰す設計 **禁止**。

### 8.4 Startup と Playback 独立性

Media selection / main UI / first frame は restore 完了を **待たない**。restore 失敗でも **Blocking Error 禁止**。

## 9. Preference Repository — Platform Policy（維持）

**全 platform:** `SharedPreferencesAudioOutputPreferenceRepository` **1 種類**。Coordinator が capability で skip。

| 不採用 | 理由 |
|--------|------|
| NoOp Repository | 無意味な fake 乱立 |
| Fake platform Repository | 同上 |

| Platform | specific restore/save |
|----------|----------------------|
| **Windows** | **あり** |
| **Android / iOS** | **なし** — `supportsPersistentDeviceId == false`。route 結果から fake device ID **禁止** |

Repository / `SharedPreferencesAsync` — **dispose 不要**。

## 10. Dispose Ordering（正式）

### 10.1 `NainaiApp.dispose()` — 完成形

```text
NainaiApp.dispose()
    1. AudioOutputPreferenceCoordinator.dispose()
    2. AudioOutputController.dispose()
    3. MediaController.dispose()
    4. MediaKitPlaybackService.dispose()    ← Player 1 回だけ
    5. super.dispose()
```

**以上のみ。** Windows では step 4 が `AudioOutputService` 兼ねる。**二重 dispose 禁止**。

### 10.2 dispose 不要（正式）

| Object | 理由 |
|--------|------|
| `AndroidAudioOutputService` | Player / subscription / persistent resource **非所有**（Phase E-1 確定） |
| `IOSAudioOutputService` | Player / Platform View **非所有**（Phase F）。将来 resource 追加時のみ設計再レビュー |
| `SharedPreferencesAudioOutputPreferenceRepository` | storage I/O wrapper のみ |
| `SharedPreferencesAsync` | dispose 対象 **にしない** |
| `IOSAudioRoutePickerView` / `UiKitView` | **Widget / Platform View lifecycle** が所有 — `NainaiApp` が `AVRoutePickerView` を dispose **しない** |

**禁止:** `AndroidAudioOutputService.dispose()` / native handler 登録解除を Dart dispose で行う **架空契約**。

### 10.3 Borrower 契約

| Borrower | 借りる対象 | dispose 時 |
|----------|-----------|------------|
| `AudioOutputController` | `AudioOutputService` | Service を dispose **しない** |
| `MediaController` | `MediaPlaybackService` | `MediaKitPlaybackService` を dispose **しない** |
| `AudioOutputPreferenceCoordinator` | Controller / Repository | 両者を dispose **しない** |

### 10.4 Native Android handler lifecycle

`AudioOutputChannelHandler` 等の native 登録は **FlutterEngine / MainActivity lifecycle** に属する。Dart `AndroidAudioOutputService` の dispose とは **独立**。

## 11. Phase D Notification Boundary

```text
AudioOutputController → pendingNotification
    → AudioOutputNotificationHost (App-level)
```

Coordinator **非経由**。Persistence error との **混同禁止**。

## 12. Settings UI Injection — Platform 別（正式）

**共通禁止:** `AudioOutputService` / `MediaKitPlaybackService` を Settings Shell / Section へ **直接 inject しない**。

### 12.1 Windows — `deviceList`

```dart
AudioOutputSettingsSection(
  controller: audioOutputController,
  selectionCommands: audioOutputCoordinator,   // AudioOutputSelectionCommands
  presentationMode: AudioOutputSettingsPresentationMode.deviceList,
);
```

### 12.2 Android — `systemRoutePickerCommand`

```dart
AudioOutputSettingsSection(
  controller: audioOutputController,
  routePickerCommands: audioOutputController,    // SystemRoutePickerCommands
  presentationMode: AudioOutputSettingsPresentationMode.systemRoutePickerCommand,
);
```

`selectionCommands` — **注入しない**。

### 12.3 iOS — `embeddedSystemRoutePicker`

```dart
AudioOutputSettingsSection(
  controller: audioOutputController,
  presentationMode: AudioOutputSettingsPresentationMode.embeddedSystemRoutePicker,
  routePicker: const IOSAudioRoutePickerView(),  // Presentation composition
);
```

`selectionCommands` / `routePickerCommands` — **注入しない**。

### 12.4 Composition Root → Settings 受け渡し（概念）

```text
NainaiApp
  └─ SettingsRoute(
        audioOutputController: _audioOutputController,
        audioOutputComposition: _audioOutputComposition,
        audioOutputCoordinator: _audioOutputPreferenceCoordinator,  // Windows のみ commands 用途
     )
```

Settings Section は `presentationMode` に応じて **必要な dependency のみ** 使用。mode と capability の **二重判定で UI を決めない** — mode を **正** とする。

## 13. Owner / Borrower / Disposer 表

| Object | Creator | Owner | Borrowers | Disposer |
|--------|---------|-------|-----------|----------|
| `SharedPreferencesAsync` | `NainaiApp` | `NainaiApp` | Repositories | — |
| `MediaKitPlaybackService` | `NainaiApp` | `NainaiApp` | `MediaController`；Windows では `AudioOutputController` | `NainaiApp` **1 回** |
| `AndroidAudioOutputService` | `NainaiApp` | `NainaiApp` | `AudioOutputController` | — |
| `IOSAudioOutputService` | `NainaiApp` | `NainaiApp` | `AudioOutputController` | — |
| `AudioOutputPlatformComposition` | `NainaiApp` | `NainaiApp` | Settings wiring | —（値 object） |
| `MediaController` | `NainaiApp` | `NainaiApp` | `MediaScreen` | `NainaiApp` |
| `AudioOutputController` | `NainaiApp` | `NainaiApp` | NotificationHost、Settings、Coordinator | `NainaiApp` |
| `SharedPreferencesAudioOutputPreferenceRepository` | `NainaiApp` | `NainaiApp` | Coordinator | — |
| `AudioOutputPreferenceCoordinator` | `NainaiApp` | `NainaiApp` | Settings（Windows commands） | `NainaiApp` |
| `IOSAudioRoutePickerView` | Settings Presentation | Flutter element tree | — | Widget lifecycle |

## 14. テスト方針（Composition）

### 14.1 Platform Presentation Mode

| ケース |
|--------|
| Windows → `deviceList` |
| Android → `systemRoutePickerCommand` |
| iOS → `embeddedSystemRoutePicker` |
| iOS capability 全 false でも **embedded picker mode** — capability 非表示 **禁止** |

### 14.2 Commands

| ケース |
|--------|
| Windows selection → `AudioOutputSelectionCommands` → Coordinator |
| Android route picker → `SystemRoutePickerCommands` → Controller |
| Android route picker — Preference save **なし** |
| iOS — programmatic open **なし** |

### 14.3 Dispose

| ケース |
|--------|
| `AndroidAudioOutputService.dispose()` **要求しない** |
| `IOSAudioOutputService.dispose()` **要求しない** |
| Windows — `MediaKitPlaybackService` **1 回** dispose |
| Android / iOS — `MediaKitPlaybackService` **1 回** dispose |
| Repository / `SharedPreferencesAsync` dispose **なし** |
| Platform View lifecycle — Widget 側 |

### 14.4 Initialize

| ケース |
|--------|
| operational restore failure が **Unhandled Future error にならない** |
| UI startup は initialize 完了を **待たない** |
| `initialize()` **1 回** |

### 14.5 Platform graph / lifecycle

§14 旧ケース — one Player、no rebuild re-create、Controller が Service dispose しない、Phase D notification 分離。

## 15. 公式 Source

| Source | 記録 |
|--------|------|
| nainai-client `4109a13` `nainai_app.dart` | 現 Composition / dispose |
| nainai-client `4109a13` `main.dart` | startup |
| [audio-output-android.md](audio-output-android.md) §5.1 | Android 分離 |
| [audio-output-ios.md](audio-output-ios.md) §1.3 / §6.1 | capability 拡張禁止 / iOS 分離 |
| [audio-output-persistence.md](audio-output-persistence.md) | Phase G dependency / readiness |
| [audio-output-hot-unplug.md](audio-output-hot-unplug.md) §16 | NotificationHost |
| `work/settings-shell-design` @ `f618ac3` | Settings Shell Presentation 契約（統合待ち） |

## 16. 状態まとめ

| 項目 | 状態 |
|------|------|
| Phase H design | **Design Complete** |
| Phase H implementation | **Implemented**（`f280a11`） |
| Platform composition | **`AudioOutputPlatformComposition` — Service + Presentation Mode** |
| Windows mode | `deviceList` |
| Android mode | `systemRoutePickerCommand` |
| iOS mode | `embeddedSystemRoutePicker` |
| Selection commands | `AudioOutputSelectionCommands` → Coordinator（**Windows のみ**） |
| Route commands | `SystemRoutePickerCommands` → Controller（**Android のみ**） |
| Repository | 単一 Concrete + capability skip |
| Coordinator | App lifetime / `unawaited(initialize())` + internal error handling |
| Dispose | Coordinator → Controllers → `MediaKitPlaybackService` **のみ** |
| Settings Shell 整合 | **Implemented**（I-3B — [settings-shell.md](settings-shell.md) §19.2） |
| Phase I-3 baseline | client `dccf48f` — 526 PASS |
| 実機 Acceptance / iOS compile | **Pending**（[settings-shell.md](settings-shell.md) §19.5） |
