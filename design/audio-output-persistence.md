# Audio Output Preference 永続化（Phase G 詳細設計）

Audio Output **Phase G** — ユーザーが **明示的に選択した** 音声出力 Preference の **永続化・起動時復元** の実装前詳細設計正本。

関連（参照のみ — cross-reference 更新は Phase F 公開後の統合へ委譲）:

- [音声出力デバイス選択](audio-output.md)
- [Windows Settings UI（Phase C）](audio-output-settings.md)
- [Windows hot unplug / fallback（Phase D）](audio-output-hot-unplug.md)
- [Common Settings Concrete Persistence](settings-persistence.md)

**client 参照:** `4109a13` — `AudioOutputPreference` / `AudioOutputController` / `AudioOutputService` / `AudioOutputCapabilities` **Implemented**。Persistence **Not Implemented**。

## 1. 位置づけ

| Phase | 内容 | 状態 |
|-------|------|------|
| **G** | **Audio Output Preference 永続化・起動時復元**（本書） | **Design Complete** / **Not Implemented** |

## 2. 目的と Phase D 決定

| 項目 | 内容 |
|------|------|
| 保存対象 | ユーザー **明示選択** の Preferred Audio Output |
| Phase D 未決 | **候補 B 正式採用** — fallback 時 **runtime のみ** System Default。**persisted preference は保持**（save しない） |

## 3. 概念分離

### 3.1 Runtime — `currentSelection`

`AudioOutputService` / `AudioOutputState.currentSelection` — **現在実際に使用している** route。Phase D fallback が変更するのは **こちらのみ**。

### 3.2 Persistence truth — `desiredPreference` / `lastPersistedPreference`

| 概念 | 意味 |
|------|------|
| **`desiredPreference`** | 現在 session の **最新ユーザー明示 intent**（save 未完了でも反映） |
| **`lastPersistedPreference`** | `AudioOutputPreferenceRepository.save()` が **最後に正常完了** した snapshot（fsync 保証 **なし** — [settings-persistence.md](settings-persistence.md) §10.2 同型） |

**禁止:** `preferredSelection` 単称で「persisted intent」と「session intent」を混同すること。

### 3.3 操作マトリクス

| 操作 | `currentSelection` | `desiredPreference` | `lastPersistedPreference` |
|------|-------------------|---------------------|---------------------------|
| ユーザー Device A 選択成功 | A | A | save 成功後 A |
| Phase D automatic fallback | systemDefault | **変更なし** | **変更なし** |
| ユーザー explicit System Default | systemDefault | systemDefault | save 成功後 systemDefault |
| save(A) 失敗（route 成功後） | A | A | **以前の成功値** |
| 起動時 A 不在 | systemDefault | A（loaded） | A（loaded） |

## 4. 依存アーキテクチャ（正式 — 一方向）

**禁止:** Controller ↔ Coordinator **双方向依存**。

```text
AudioOutputPreferenceCoordinator
        |
        +--> AudioOutputController      (borrow)
        |
        +--> AudioOutputService         (borrow — readiness 等)
        |
        +--> AudioOutputPreferenceRepository

AudioOutputController
        X--> AudioOutputPreferenceCoordinator   (知らない)

AudioOutputPreferenceRepository
        X--> AudioOutputController              (知らない)
```

| コンポーネント | 担当 |
|----------------|------|
| **`AudioOutputController`** | runtime state、`AudioOutputService` 観測、Phase D reconciliation / automatic fallback、**runtime selection primitive**（`selectDevice` / `selectSystemDefault`）、`_selectGeneration` race |
| **`AudioOutputPreferenceCoordinator`** | persisted load、startup restore orchestration、**user command entry**、selection 成功後 persist、serialized save ordering、persistence error、`desiredPreference` / `lastPersistedPreference` |
| **`AudioOutputPreferenceRepository`** | storage I/O のみ |

**Controller は Persistence を知らない。** `AudioOutputPreferenceRepository.load/save` を **直接呼ばない**。

## 5. Presentation パス（正式）

### 5.1 User command path（Phase G 完成形）

```text
Presentation
    ↓  user command
AudioOutputPreferenceCoordinator.selectDevice / selectSystemDefault
    ↓  runtime primitive
AudioOutputController.selectDevice / selectSystemDefault
    ↓
AudioOutputService
    ↓  success
Coordinator.persistExplicitSelection(...)   // serialized save
```

**Phase G 完成形** では Presentation から Controller へ **直接** user command を送らない。

`AudioOutputController.selectDevice()` / `selectSystemDefault()` は **runtime primitive** として残る。Coordinator が呼ぶ。Dart 上の private 化は必須ではないが、**設計上の正式 command entry は Coordinator**。

将来 `AudioOutputSelectionCommands` 等の小 interface で Presentation の Concrete 依存を弱めてもよい。Phase G は **責務境界確定** を優先。

### 5.2 State path（変更なし）

```text
AudioOutputController
    ↓  AudioOutputState
Presentation
```

Coordinator は **`AudioOutputState` を複製しない**。

### 5.3 Phase D fallback path（Coordinator 非経由）

```text
AudioOutputController（Phase D reconciliation）
    ↓
AudioOutputService.selectSystemDefault()
```

**Coordinator を経由しない。** 構造上 **Preference save なし** を保証。

### 5.4 Startup restore path

```text
Composition Root
    ↓
AudioOutputPreferenceCoordinator.initialize()     // Repository.load
    ↓
applyStartupRestore()                             // readiness → Controller.select*
```

restore 成功後は **ユーザー明示操作ではない** → **同じ preference を re-save しない**。

## 6. Device enumeration readiness（client 調査 — 正式決定）

### 6.1 client `4109a13` read-only 調査

| 項目 | 結果 |
|------|------|
| `MediaKitPlaybackService.availableDevices` | `_player.state.audioDevices` の **同期 getter**（L82–86） |
| media_kit 初期 `PlayerState.audioDevices` | **`[AudioDevice('auto', '')]` のみ**（`media_kit` 1.2.6 `player_state.dart` L114） |
| mapper 後 | `auto` 除外 → **`availableDevices == []`** になりうる |
| 完全一覧 | mpv **`audio-device-list` property event** で **非同期** 到着（`real.dart` L1388–1433） |
| `AudioOutputController` | **subscribe-before-snapshot**（L18–37）。`isLoadingDevices` は production で **常に false**（Settings Section コメント参照） |

### 6.2 正式結論

起動直後の **`availableDevices == []` を authoritative missing と判断してはならない**。

| 状態 | 意味 |
|------|------|
| 同期 snapshot `[]`（auto のみ） | **enumeration 未完了の一時値** になりうる |
| **first `availableDevicesStream` emission** | Phase G における **authoritative enumeration snapshot** |

### 6.3 Phase G readiness contract（正式）

| 項目 | 決定 |
|------|------|
| restore 判定前 | **`await audioOutputService.availableDevicesStream.first`**（または同等の **first authoritative snapshot** Future） |
| 根拠 | media_kit README / `player.stream.audioDevices.first` パターン |
| **禁止** | `Future.delayed(500ms)` / arbitrary debounce / 「少し待つ」 |

first emission **後**:

| 条件 | restore 判定 |
|------|-------------|
| A.id ∈ devices | **present** → `selectDevice(A.id)` 試行 |
| A.id ∉ devices | **missing at startup**（§12） |
| devices が `[]` | **authoritative empty** — specific 復元 **不可**（missing 扱い） |

## 7. Persistence 技術

| 項目 | 決定 |
|------|------|
| Package | **`shared_preferences` ^2.5.5** + **`SharedPreferencesAsync`** |
| Concrete | **`SharedPreferencesAudioOutputPreferenceRepository`** |
| Key | **`audioOutput.preference`** — **1 JSON string / 1 `setString`** |
| Payload | `version: 1`, `mode`, `deviceId`, `label` |
| Common Settings | **`SettingsRepository` へ混ぜない** |

```json
{ "version": 1, "mode": "systemDefault", "deviceId": null, "label": null }
{ "version": 1, "mode": "specificDevice", "deviceId": "<id>", "label": "..." }
```

`deviceId` = identity。`label` = display cache のみ — **restore 条件に不使用**。UI は `availableDevices` の最新 label を優先。

## 8. Repository boundary

```dart
abstract interface class AudioOutputPreferenceRepository {
  Future<AudioOutputPreference?> load();
  Future<void> save(AudioOutputPreference preference);
  Future<void> clear();
}
```

| `load()` → null | **保存 preference なし** |
| invalid（malformed current version） | **適用しない**。§13 |
| `version` > supported | **適用しない**。payload **削除 / 上書き禁止** |

## 9. Coordinator 内部 model

### 9.1 起動時 `load()` 成功

| `load()` 結果 | `desiredPreference` | `lastPersistedPreference` |
|---------------|-------------------|---------------------------|
| **null**（未保存） | **null** | **null** |
| valid preference | loaded | loaded |

**null = no persisted preference** — implicit systemDefault へ **内部変換しない**（restore は skip、runtime = Service 既定）。

### 9.2 Persistence-specific UI state

Coordinator が **`preferredDeviceUnavailable`**（名称は実装 Phase で確定可）を保持可:

- saved specific A が **authoritative snapshot 上 missing**
- runtime を specific A の **fake 状態** に **しない**
- Phase C `currentSelection` missing（§8.1）と **別概念**

## 10. 起動時 restore sequence（正式）

```text
1. AudioOutputService / AudioOutputController 準備（既存 Composition）
2. AudioOutputPreferenceCoordinator.initialize()
       → Repository.load()
       → desiredPreference / lastPersistedPreference 初期化
3. supportsPersistentDeviceId == false → restore skip（Android / iOS）
4. await availableDevicesStream.first   // §6.3 authoritative snapshot
5. saved systemDefault → Controller.selectSystemDefault()
6. saved specific A かつ A ∈ devices → Controller.selectDevice(A.id)
7. saved specific A かつ A ∉ devices → §12 missing
8. select 失敗 → §11 restore failure
```

**restore はユーザー明示操作ではない** — **re-save 禁止**。

## 11. Restore failure

Device A ∈ devices だが `Controller.selectDevice(A.id)` 失敗:

| 項目 | 正式 |
|------|------|
| runtime | **Service / Controller の実際 `currentSelection` を正** とする。Presentation だけ default に **書き換え禁止** |
| 必要時 | `Controller.selectSystemDefault()` を **明示試行** し safe default へ（Service 実装に合わせる） |
| `desiredPreference` / `lastPersistedPreference` | **A 保持** |
| UI | **Non-blocking restore error** |
| Playback Blocking Error | **禁止** |

## 12. Missing device at startup

authoritative snapshot 上 **A ∉ devices**:

| 項目 | 正式 |
|------|------|
| runtime | System Default（Service 正本） |
| `desiredPreference` / `lastPersistedPreference` | **A 保持** |
| reconnect（同一 session） | **auto restore 禁止**（Phase D） |
| UI | `preferredDeviceUnavailable` 等 **Non-blocking**（Phase C missing 表示と **区別**） |

## 13. invalid / unknown payload

### 13.1 malformed current-version payload

JSON 破損 / unknown mode / specific で deviceId 欠落 等:

| 項目 | 正式 |
|------|------|
| 適用 | **しない** |
| runtime | System Default（restore skip） |
| repair write | **禁止** |
| UI | **Non-blocking persistence error**（第一候補 **確定**） |

### 13.2 unknown future version

`version > supported`:

| 項目 | 正式 |
|------|------|
| 適用 | **しない** |
| payload | **削除 / 上書き禁止** |
| corruption 同一扱い | **必須ではない** |
| UI error | **過剰通知回避** — 実装 Phase で **silent skip または soft warning** を選択可 |

## 14. User operation と persist timing

```text
Coordinator.select*()
  → Controller.select*() 成功
  → desiredPreference = preference
  → serialized save(preference)

Controller.select*() 失敗
  → save 禁止

Phase D fallback
  → Coordinator 非経由 → save 禁止
```

## 15. Save failure / ordering

Coordinator が **serialized persistence writes** を管理。Repository は **1 load / 1 save** のみ。

### 15.1 latest save failure

user selects B、save(B) 失敗（最新 intent）:

| 項目 | 正式 |
|------|------|
| runtime | **B 維持**（route rollback **禁止**） |
| `desiredPreference` | **B 維持** |
| `lastPersistedPreference` | **最後の成功値** |
| UI | Non-blocking save error |

Common Settings と **異なり** runtime を `lastPersistedPreference` へ **rollback しない**。

### 15.2 stale save success

A save 遅延成功、既に desired = B:

| 項目 | 正式 |
|------|------|
| `lastPersistedPreference` | 更新してよい（事実として A が disk に残った） |
| `desiredPreference` | **B のまま** — **A へ戻さない** |
| runtime | **変更しない** |

### 15.3 stale save failure

A save 失敗、既に desired = B:

- desired / runtime **変更しない**
- newer error を **上書きしない**

### 15.4 `initialize()` / restore race

| ケース | 方針 |
|--------|------|
| load / restore 中の user selection | **user 最優先** |
| stale restore completion | route **上書き禁止** |
| generation | Coordinator 側 **`_restoreGeneration`**（Persistence / startup intent **専用**）。Controller `_selectGeneration` と **役割重複させない** |

## 16. Platform 境界

| Platform | Persistence |
|----------|-------------|
| **Windows** | specific save / restore **あり** |
| **Android**（Phase E） | **なし** — OS route / fake device ID **禁止** |
| **iOS**（Phase F） | **なし** — AVRoute / AirPlay 等から ID **禁止** |

## 17. Phase D 統合

| 契約 | 内容 |
|------|------|
| fallback completion | **`Repository.save()` 呼ばない** |
| reconnect | **auto restore 禁止** |
| automatic restore 入口 | **startup のみ**（+ user explicit via Coordinator） |

## 18. Ownership / Composition

```text
NainaiApp
├── SharedPreferencesAsync()
├── AudioOutputService
├── AudioOutputController(service)
├── AudioOutputPreferenceRepository
└── AudioOutputPreferenceCoordinator(repository, service, controller)
```

| 対象 | dispose |
|------|---------|
| Repository | **不要** |
| Coordinator | Composition Root |
| Controller | Repository **dispose しない** |

## 19. テスト方針

### 19.1 Architecture

| ケース |
|--------|
| Controller は Coordinator **非依存** |
| Coordinator は Controller + Repository + Service **依存** |
| fallback が Coordinator **非経由** |

### 19.2 Initial readiness

| ケース |
|--------|
| 同期 `[]` だけでは missing **判定しない** |
| `availableDevicesStream.first` 後に restore 判定 |
| authoritative empty → specific missing |
| authoritative に device present → restore |
| **arbitrary delay なし** |

### 19.3 Persistence truth

| ケース |
|--------|
| `desiredPreference` / `lastPersistedPreference` |
| stale save success / failure |
| latest save failure — **runtime rollback なし** |

### 19.4 Startup / race / platform

§18 旧ケース + restore 中 user select / stale restore / missing / restore failure / Windows restore / Android・iOS **no persistence**。

## 20. 公式 Source

| Source | 記録 |
|--------|------|
| nainai-client `4109a13` `media_kit_playback_service.dart` | 同期 `availableDevices` |
| nainai-client `4109a13` `audio_output_controller.dart` | subscribe-before-snapshot |
| `media_kit` 1.2.6 `real.dart` L1388–1433 | `audio-device-list` 非同期 |
| [settings-persistence.md](settings-persistence.md) | storage / durability |
| [audio-output-hot-unplug.md](audio-output-hot-unplug.md) | Phase D 境界 |

## 21. 状態まとめ

| 項目 | 状態 |
|------|------|
| Phase G design | **Design Complete** |
| Phase G implementation | **Not Implemented** |
| Phase D → Preference | **B — runtime only fallback, persist 保持** |
| Dependency | **Coordinator → Controller + Repository（一方向）** |
| Readiness | **`availableDevicesStream.first`** |
| Truth model | **`desiredPreference` / `lastPersistedPreference`** |
