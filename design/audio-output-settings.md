# Windows 音声出力 Settings UI（Phase C 詳細設計）

Audio Output **Phase C** — Windows 向け Settings 内「音声出力デバイス選択 UI」の実装前詳細設計正本。

Service / platform 抽象の正本は [audio-output.md](audio-output.md)。Localization 基盤は [localization.md](localization.md)。Visual Token は [design-system.md](design-system.md)。**Settings Shell / 共通 preference 基盤** は [settings.md](settings.md)。

関連:

- [アプリ共通 Settings 基盤](settings.md)
- [音声出力デバイス選択](audio-output.md)
- [UI Localization](localization.md)
- [Design System](design-system.md)
- [Phase 2 UI](phase2-ui.md)

## 1. 位置づけ

| Phase | 内容 | 状態 |
|-------|------|------|
| A | 共通 Model / Service 抽象 / Capability | **Implemented**（`4a34d85`） |
| B | Windows `MediaKitPlaybackService` が `AudioOutputService` も実装 | **Implemented**（`c3239f3`） |
| **C** | **Windows Audio Output Settings subsystem**（本書） | **Design Complete**（実装未着手） |

**Design Complete の範囲:** Controller / State / Composition / Ownership / Settings Section / Selection / Error / Localization / Accessibility / Testing。

**未確定（Phase C subsystem の Visual Design 完全確定ではない）:** Settings Launcher 物理配置 — **Launcher placement design pending**（§14）。fixed Bottom Control は client `68ff1b4` で **Implemented**。共通 Settings Shell / Navigation は [settings.md](settings.md) が正本。

Phase C 実装完了時点でユーザーが Settings から出力デバイスを切り替えられる。**永続化・hot unplug 自動 fallback・Android/iOS UI は含まない。**

## 2. 設計原則

- Presentation から `AudioOutputService` を **直接操作しない**。`AudioOutputController` 経由とする
- `MediaKitPlaybackService` 具象を Settings UI へ露出しない
- Audio Output エラーは **Playback Blocking Error と混同しない**
- Settings 操作で Media 画面を Error 画面へ置き換えない
- 不必要に複雑な State Machine は採用しない

## 3. Phase C スコープ

### 3.1 含む

| # | 項目 |
|---|------|
| 1 | `AudioOutputController` |
| 2 | Audio Output Settings 状態モデル（`AudioOutputState`） |
| 3 | Windows 出力デバイス一覧 UI |
| 4 | System Default 選択 |
| 5 | Specific Device 選択 |
| 6 | Device 一覧更新（Stream 反映） |
| 7 | 選択中デバイス表示 |
| 8 | Loading 状態 |
| 9 | Empty 状態 |
| 10 | Non-blocking Error |
| 11 | Application Composition（Controller wiring） |
| 12 | Controller / Service dispose ownership |
| 13 | Settings 画面を閉じた場合の扱い |
| 14 | Localization キー設計 |
| 15 | Accessibility |
| 16 | テスト方針 |

### 3.2 含まない（後続 Phase）

| Phase | 責務 |
|-------|------|
| **D** | hot unplug 検知、選択中デバイス消失時の **自動** System Default fallback、disconnect 通知 |
| **E** | Android platform output picker UI |
| **F** | iOS `AVRoutePickerView` UI |
| **G** | `AudioOutputPreference` 永続化、再起動後復元 |

Phase C は **「現在取得できている一覧と currentSelection を表示し、ユーザー操作で切り替える」** ところまで。

## 4. AudioOutputController

### 4.1 責務

| 担当 | 内容 |
|------|------|
| Capability 反映 | `AudioOutputService.capabilities` を State へ |
| 一覧監視 | `availableDevices` / `availableDevicesStream` |
| 選択監視 | `currentSelection` / `currentSelectionStream` |
| 操作委譲 | `selectSystemDefault()` / `selectDevice(deviceId)` |
| UI 通知 | `ChangeNotifier` により Settings UI へ State 通知 |
| Settings エラー | UI 操作中エラーの State 化（Non-blocking） |

**担当しない:** Service dispose、Persistence、hot unplug 自動 fallback、Media playback state の更新。

### 4.2 推奨 API（設計）

```dart
class AudioOutputController extends ChangeNotifier {
  AudioOutputController({required AudioOutputService service});

  AudioOutputState get state;

  Future<void> selectSystemDefault();
  Future<void> selectDevice(String deviceId);

  @override
  void dispose();
}
```

`openSystemRoutePicker()` は Windows Phase C では Capability 上未サポート（`UnsupportedError`）。Controller API へは含めない。

Presentation は `AudioOutputService` 型を import しない。`AudioOutputController` と `AudioOutputState` のみ参照する。

## 5. AudioOutputState

名称第一候補: **`AudioOutputState`**（immutable）。

| フィールド | 型（概念） | 説明 |
|------------|------------|------|
| `capabilities` | `AudioOutputCapabilities` | Service から取得 |
| `availableDevices` | `List<AudioOutputDevice>` | specific device 一覧（`auto` 不含） |
| `currentSelection` | `AudioOutputPreference` | 現在の選択 |
| `isLoadingDevices` | `bool` | 初回 / 再列挙待ち |
| `isSelecting` | `bool` | 選択操作進行中 |
| `error` | `AudioOutputError?` | Settings 内 Non-blocking error（null = なし） |

**意図的に持たない:** Playback status、Media 情報、Persistence 状態、hot unplug 検知フラグ。

`error` は [audio-output.md](audio-output.md) §10.3 の `AudioOutputErrorType` 候補と整合する軽量モデルとする。`MediaError` とは型を共有しない。

## 6. Application Composition

Application Composition / dispose ownership の技術正本は [media-technology.md](../architecture/media-technology.md) §Application Composition。本節は Phase C に伴う **ownership 移行** を確定する。

### 6.0 Current（Phase B 実装済み — client main `c3239f3`）

現在の client では次の ownership  chain である（**実装済み**）。

```text
NainaiApp
  ├─ MediaKitPlaybackService()      ← 1 instance 生成
  ├─ MediaController(service)       ← MediaPlaybackService として注入
  └─ MediaScreen
```

**Current dispose（実装済み）:**

```text
NainaiApp.dispose()
    └─ MediaController.dispose()
           ├─ StreamSubscription 等を cancel
           ├─ MediaPlaybackService.dispose()   ← Controller が Service を dispose
           └─ super.dispose()
```

[media-playback.md](media-playback.md) §12 dispose および [media-technology.md](../architecture/media-technology.md) §Application Composition に記載のとおり、**現状は `MediaController` が注入された `MediaPlaybackService` の dispose まで担当** する。

`AudioOutputController` は未実装のため、共有 Service ownership の問題は Phase B 時点では顕在化しない。

### 6.1 Phase C 実装要件 — ownership 移行

Phase C では **同一 `MediaKitPlaybackService` を `MediaController` と `AudioOutputController` が共有** するため、Current の dispose  chain は **そのままでは不整合** となる。

**Phase C 以降の正式 ownership（実装要件）:**

```text
NainaiApp                          ← Composition Root / 唯一の Service owner
│
├── MediaKitPlaybackService        ← 1 instance / 1 Player
│     ├─ MediaPlaybackService として MediaController へ注入（borrow）
│     └─ AudioOutputService として AudioOutputController へ注入（borrow）
│
├── MediaController                ← Service を借りる側。dispose しない
└── AudioOutputController          ← Service を借りる側。dispose しない
```

| 時点 | Service owner | MediaController | AudioOutputController |
|------|---------------|-----------------|----------------------|
| **Current** | 実質 `MediaController`（dispose 経由） | Service を dispose する | 未実装 |
| **Phase C 以降** | **`NainaiApp`（Composition Root）** | Service を **借りる** のみ | Service を **借りる** のみ |

**Phase C implementation requirement:** `MediaController.dispose()` から **`MediaPlaybackService.dispose()` 呼び出しを除去** する。これは Audio Output Phase C の Application Composition 変更として **必須**。将来検討ではない。

### 6.2 Service 共有 — 二重 Player / 二重 Service 禁止（Phase C 確定）

`MediaKitPlaybackService` は **1 インスタンス** が `MediaPlaybackService` と `AudioOutputService` の両 interface を実装する（Phase B 実装済み）。

```text
playbackService = MediaKitPlaybackService()   // 1 instance ONLY

MediaController(playbackService)              // MediaPlaybackService として
AudioOutputController(playbackService)        // AudioOutputService として
```

**禁止:**

- `MediaController` 用と `AudioOutputController` 用に **別々の `MediaKitPlaybackService` を生成** すること
- **Player を 2 個** 生成すること
- interface ごとに concrete instance を分けること

共有 concrete instance を interface 型で注入する。

### 6.3 Phase C 目標構成（Windows）

```text
NainaiApp
  ├─ playbackService = MediaKitPlaybackService()   // 1 instance
  ├─ MediaController(playbackService)
  ├─ AudioOutputController(playbackService)
  ├─ FileSelectorMediaSelectionService
  ├─ MediaScreen
  └─ SettingsScreen
        └─ AudioOutputSettingsSection   ← Phase C
```

Phase C 最小スコープとして **Audio Output Section のみ** でもよい。Settings Shell / General Section / Tooltip は [settings.md](settings.md) が正本。Phase C 実装時は Shell 未完了でも Section 単体開発可（§14）。

### 6.4 dispose ownership（Phase C 以降 — 設計確定）

| 対象 | Phase C 以降 |
|------|--------------|
| `MediaKitPlaybackService` / `Player` | **`NainaiApp`（Composition Root）が唯一 owner**。Application lifetime で **1 回だけ** dispose |
| `MediaController` | 自身の `StreamSubscription` / Listener 等のみ dispose。**共有 Service を dispose しない** |
| `AudioOutputController` | 自身の `StreamSubscription` / Listener 等のみ dispose。**共有 Service を dispose しない** |

`MediaController` と `AudioOutputController` は **同一 concrete `MediaKitPlaybackService` を共有** する。いずれの Controller からも `MediaPlaybackService.dispose()` / `AudioOutputService.dispose()` を呼んではならない。

### 6.5 dispose 順序（Phase C 以降 — `NainaiApp.dispose()`）

Controller が保持する resource を先に解放し、**最後に共有 Service を 1 回だけ** dispose する。

```text
NainaiApp.dispose()
    1. AudioOutputController.dispose()     ← Subscription / Listener のみ
    2. MediaController.dispose()           ← Subscription / Listener のみ（Service dispose しない）
    3. MediaKitPlaybackService.dispose()   ← Player / Service を 1 回だけ
```

`MediaKitPlaybackService.dispose()` を **複数箇所から呼ばない**。

## 7. Device 一覧 UI

### 7.1 データソース

Windows では `AudioOutputService.availableDevices` / `availableDevicesStream` を表示する。

**System Default は Service 一覧に含まれない。** Settings UI が一覧 **先頭** に独立行として追加する。

```text
音声出力

○ システム既定          ← UI が明示追加（System Default）
○ Speakers (Realtek …)  ← availableDevices[0]
○ Headphones (…)        ← availableDevices[1]
○ Display Audio (…)     ← availableDevices[2]
```

### 7.2 一覧更新

`availableDevicesStream` を Controller が購読し、接続デバイス変更を State へ反映する。Phase C では **一覧の再描画のみ**。消失デバイスに対する自動 selection 変更は Phase D。

### 7.3 specific device 0 件（Empty ではない）

`availableDevices.isEmpty` でも **Settings 一覧全体を Empty 画面 / Blocking Empty UI にしてはならない。** System Default は UI 側で常に先頭に表示するため、**常に 1 行以上選択可能** である。

`availableDevices` が 0 件（specific device なし）の場合:

```text
音声出力

○ システム既定          ← 選択可能（常に表示）

利用可能な個別デバイスはありません   ← 補助メッセージ（Non-blocking）
```

| 項目 | 扱い |
|------|------|
| System Default 行 | **常に表示・選択可能** |
| specific device 行 | 0 件 |
| 補助メッセージ | Section 内 informational（Semantic Warning 可） |
| Blocking Empty UI | **禁止** — 「音声出力デバイスがありません」等で Section 全体を無効化しない |

Localization: `audioOutputNoIndividualDevices`（§16）。`audioOutputNoDevices` は Blocking Empty 用途として **使用しない**。

Media 画面・Playback controls には影響しない。

### 7.4 Loading 状態

| 状況 | UI |
|------|-----|
| 初回 device 一覧未取得 | Section 内 Progress / skeleton。System Default 行は disabled または非操作可 |
| Stream 更新待ち（初回のみ） | 同上 |
| 選択操作 `isSelecting` | 操作中の Radio 行または Section 全体を一時 disable（Blocking Error にしない） |

## 8. 選択方式

- **単一選択**。Radio List / Radio Tile 相当を基本とする（[design-system.md](design-system.md) Token 準拠）
- 選択判定は **`AudioOutputDevice.id` のみ**。`label` は表示用であり identity に使わない
- `currentSelection` に応じて選択状態を反映:

| `currentSelection` | 選択行 |
|--------------------|--------|
| `systemDefault` | System Default 行 |
| `specificDevice(deviceId: id, …)` | `availableDevices` 内で `id` が一致する行 |
| `specificDevice` だが `id` が一覧に **ない** | **どの Radio も selected にしない**（§8.1） |

`label` 変更で別デバイス扱いに **しない**（Phase A/B 仕様維持）。

### 8.1 currentSelection と一覧不一致（Phase C 確定）

Phase C では hot unplug **自動 fallback を実装しない**（Phase D）。そのため次の状態が **一時的に成立しうる**:

```text
currentSelection = specificDevice(deviceId: A)
availableDevices に device A が存在しない
```

**Phase C UI 挙動（確定）:**

| 禁止 | 理由 |
|------|------|
| 別 device を勝手に selected にする | ユーザー意図の改変 |
| System Default を勝手に selected にする | 同上 |
| `selectSystemDefault()` を勝手に実行 | Service 状態の自動変更 |
| Playback Blocking Error | Settings 文脈の問題 |

**Phase C で行う:**

- Radio 一覧上は **どの行も selected に偽装しない**
- Section 内に **Non-blocking** で「現在選択中の出力デバイスが利用できません」を表示（`audioOutputSelectedDeviceUnavailable`）
- `currentSelection`（Service 正本）は Phase C では **変更しない** — 表示上 selected なし + 警告
- ユーザーが System Default または別 device を **明示選択** すれば通常どおり切替
- **自動 fallback そのものは Phase D**

## 9. System Default 選択

```text
UI tap（System Default 行）
    ↓
AudioOutputController.selectSystemDefault()
    ↓
AudioOutputService.selectSystemDefault()
    ↓
Windows: setAudioDevice(AudioDevice.auto())   // name == 'auto'
```

- UI に `auto` という media_kit 内部値を **表示しない**
- System Default 行の表示文言は Localization `systemDefault`（§15）

## 10. Specific Device 選択

```text
UI tap（device 行）
    ↓
AudioOutputController.selectDevice(device.id)
    ↓
AudioOutputService.selectDevice(deviceId)
```

- UI から渡すのは **`AudioOutputDevice.id` のみ**
- `AudioDevice` / media_kit 型を Presentation へ露出しない
- Phase B 仕様維持: unknown `deviceId` → Service が `ArgumentError`。Controller は Settings Non-blocking Error へ変換。**System Default へ勝手に fallback しない**

### 10.1 操作直前にデバイス消失

ユーザーが行をタップした時点で `deviceId` が一覧から消えていた場合:

- Phase C: **Non-blocking Error**（例: `deviceUnavailable`）
- 自動 fallback: **Phase D**

## 11. 選択操作中

デバイス変更は非同期操作。

| 禁止 | 理由 |
|------|------|
| UI 全体を Blocking Error へ | Settings 文脈の操作失敗 |
| Media 再生画面の破壊 | Playback と独立 |
| Playback Error へ変換 | [audio-output.md](audio-output.md) §10 |

**許可:** `isSelecting == true` 中、当該 Section または操作中 Radio を temporary disable。成功時 `error` clear。失敗時 `error` 更新し **直前の successful selection 表示を維持**。

## 12. Error policy

Audio Output Settings のエラーは **Settings 内 Non-blocking** とする。

| 種別 | Phase C 扱い | Media 画面 |
|------|--------------|------------|
| enumeration 失敗 | Section 内 warning + System Default のみ選択可 | 影響なし |
| selection 失敗 | Section 内 inline / banner error | 影響なし |
| 操作直前 device 消失 | Non-blocking error | 影響なし |
| hot unplug 自動 fallback | **Phase D** | — |

- `technicalDetails` / Exception 文字列を通常 UI へ露出しない
- Semantic Warning 色は [design-system.md](design-system.md) 準拠

## 13. Settings UI（Visual / Layout）

### 13.1 Design System 準拠

| 項目 | 値 |
|------|-----|
| Persona | Professional Creative Media Tool |
| Background | `#0b1326`（`background` token） |
| Surface | 既存 surface token |
| Primary / Accent | `#8083FF`（`accent-primary`） |
| Touch target | **44 logical px 以上**（Mobile 基準を Desktop Settings にも適用） |
| Radius | 既存 Design System 準拠 |

**独自色を追加しない。**

### 13.2 Section 構成

Settings 内の独立 Section:

| Locale | Section title |
|--------|---------------|
| ja | 音声出力 |
| en | Audio Output |

Section 内:

1. System Default 行（常に先頭）
2. specific device 行（`availableDevices` 順）

Phase C では **Windows のみ**。Android / iOS 向け Physical Device List UI は Phase E / F。

## 14. Settings 入口（Implementation Prerequisite）

### 14.1 Phase C で確定する範囲

**Settings UI 内部** — Audio Output Section の Controller / State / Widget / Error / Localization / Accessibility / Test。

### 14.2 Settings Launcher 配置（未確定 — 2026-08-30 更新）

**前提更新:** Windows 固定 Bottom Control（Audio / Video 共通 Bottom Panel、collapse / compact landscape 等）は client **`68ff1b4` で Implemented**（[phase2-ui.md](phase2-ui.md) §4.1）。「fixed Bottom Control 統合待ち」は **解消** した。

**現 Status:** **Launcher placement design pending** — Launcher の物理配置そのものは **未設計・未実装**。

client `68ff1b4` 時点の Media 画面構造（read-only 確認）:

| 領域 | 内容 |
|------|------|
| body 上部 | Non-blocking Banner（該当時のみ） |
| body 中央 | Audio / Video Media 領域（Bottom Panel 除く残りを中央配置） |
| `bottomNavigationBar` | `DesktopMediaBottomPanel` — Seek / Transport / Volume / Select Another File（compact 時 folder icon）/ collapse toggle |
| collapse 時 | Toggle bar のみ（▲ icon） |

**Launcher 候補比較（Phase C 設計入力 — 本日時点で未確定）:**

| 候補 | 概要 | 評価メモ |
|------|------|----------|
| **A. Bottom Panel 内** | compact bar 右端または toggle bar 付近に Settings icon | 通常 Windows / narrow Windows / compact landscape expanded で controls と同階層。compact collapsed 時は Panel が toggle のみになり Settings 到達性が下がる |
| **B. Media Area 上部** | body 上部（Banner 下）に Settings 入口 | Bottom Panel 密度に依存しない。compact landscape でも一貫しうる。Media 領域との視覚的分離が必要 |
| **C. App-level 独立位置** | Window chrome 近傍・将来 AppBar 等 | OS 標準 Window Chrome 方針（[phase2-ui.md](phase2-ui.md) §10）と整合要検討。Android / iOS では別 Navigation パターンになりうる |

**判断:** 資料と client 構造のみでは **物理配置を確定しない**。Phase C 実装前に、通常 Windows / narrow Windows / compact landscape / Android / iOS の一貫性を評価して Launcher を決定する。

Launcher 未確定でも **Audio Output Section 単体** の Widget / Controller 実装とテストは可能（Modal / 暫定 route）。

| 項目 | 状態 |
|------|------|
| fixed Bottom Control | **Implemented**（`68ff1b4`） |
| Settings Launcher 配置 | **Launcher placement design pending** |
| Common Settings / Tooltip ON/OFF | **[Core Design Complete](settings.md)** / **未実装** |
| Audio Output Settings UI / Controller | Phase C **Design Complete**、実装未着手 |

共通 Settings Shell / Navigation contract の正本は [settings.md](settings.md) §15。

## 15. Settings 画面を閉じた場合

| 項目 | 扱い |
|------|------|
| 選択結果 | Service / `currentSelection` に反映済みなら **維持**（Phase G まで Persistence なし） |
| Controller | Application lifetime で保持。Settings 閉鎖で dispose しない |
| `error` | Settings 再表示時に clear するかは実装時判断可。Phase C 最小: 再表示時に clear **推奨** |
| Media 再生 | 選択成功後も Playback state は独立。Media 画面を閉じない |

## 16. Localization

[localization.md](localization.md) の gen-l10n / ARB 基盤を使用。**日本語直書き禁止。** 実装 Phase で ARB へ追加（本設計時点では ARB 変更なし）。

### 16.1 Phase C キー候補（camelCase）

| キー | ja | en |
|------|----|----|
| `audioOutput` | 音声出力 | Audio Output |
| `systemDefault` | システム既定 | System Default |
| `audioOutputDevices` | 出力デバイス | Output Devices |
| `audioOutputLoading` | 出力デバイスを読み込み中… | Loading output devices… |
| `audioOutputNoIndividualDevices` | 利用可能な個別デバイスはありません | No individual audio devices available |
| `audioOutputSelectedDeviceUnavailable` | 現在選択中の出力デバイスが利用できません | The currently selected output device is unavailable |
| `audioOutputSelectionFailed` | 出力デバイスを変更できませんでした | Could not change the output device |
| `audioOutputEnumerationFailed` | 出力デバイスを取得できませんでした | Could not load output devices |
| `audioOutputDeviceUnavailable` | 選択した出力デバイスが利用できなくなりました | The selected output device is no longer available |

[audio-output.md](audio-output.md) §11.1 の既存候補（`settingsTitle` 等）と実装時に統合・照合する。

## 17. Accessibility

各 device 行および System Default 行は Semantics で次を認識可能にする:

| 属性 | 内容 |
|------|------|
| label | 表示名（Localization 経由） |
| selected | `currentSelection` と一致 |
| enabled | `!isSelecting` かつ error 状態を考慮 |

- Icon のみで意味を伝える Control には Tooltip + Semantics を必須とする
- Radio 選択状態は `selected` Semantics と視覚的 selected 状態を一致させる

## 18. テスト方針

Service Phase B テスト（device mapping / `selectDevice` 規則等）と **責務重複させない**。Controller / Widget は Service を mock する。

### 18.1 Controller unit tests

| ケース |
|--------|
| initial capabilities |
| initial devices / selection |
| `availableDevicesStream` 更新反映 |
| `currentSelectionStream` 更新反映 |
| `selectSystemDefault` 成功 |
| `selectDevice` 成功 |
| selection 失敗 → Non-blocking error、selection 表示維持 |
| unknown device → error（自動 fallback なし） |
| dispose 後 `notifyListeners` しない |

### 18.2 Widget tests

| ケース |
|--------|
| System Default 行表示 |
| device 一覧表示 |
| current selection 反映（system / specific） |
| System Default 選択操作 |
| specific device 選択操作 |
| loading 表示 |
| empty（specific 0 件、System Default のみ）表示 |
| currentSelection が一覧にない（selected なし + 警告） |
| non-blocking error 表示 |
| accessibility（selected / enabled Semantics） |
| 日本語表示 |
| English 表示 |

## 19. Phase 境界まとめ

| Phase | 責務 | 状態 |
|-------|------|------|
| **C** | Windows Audio Output Settings subsystem（Controller / Section / wiring / ユーザー操作切替） | **Design Complete** |
| | Settings Launcher 物理配置 | **Launcher placement design pending** |
| **D** | hot unplug、選択中デバイス消失の **自動** fallback、通知 | 未実装 |
| **E** | Android platform output picker Settings UI |
| **F** | iOS route picker Settings UI |
| **G** | Preference persistence、再起動復元 |

## 20. 未確定事項（Phase C 実装前）

| 項目 | 備考 |
|------|------|
| Settings Launcher 物理配置 | §14 — **Launcher placement design pending**（fixed Bottom Control は `68ff1b4` Implemented） |
| `SettingsController` との compose タイミング | 独立 `AudioOutputController` を Phase C で先行実装可。[settings.md](settings.md) §4.5 |
| Settings 全体 Navigation 構造 | Shell / General / Tooltip は [settings.md](settings.md)。Audio Output Section 以外の Phase C 外項目も同書 |
| Section 内 Banner 自動消失 | [phase2-ui.md](phase2-ui.md) と同様、勝手に秒数確定しない |
