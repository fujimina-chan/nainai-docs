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
        +--> AudioOutputController              (borrow)
        |       |
        |       +--> AudioOutputService         (Controller が borrow)
        |
        +--> AudioOutputPreferenceRepository

AudioOutputController
        X--> AudioOutputPreferenceCoordinator   (知らない)

AudioOutputPreferenceRepository
        X--> AudioOutputController              (知らない)
        X--> AudioOutputService                 (知らない)

AudioOutputService
        X--> AudioOutputPreferenceRepository    (知らない)
        X--> AudioOutputPreferenceCoordinator   (知らない)
```

Coordinator は **readiness 待機** のため `AudioOutputService` を **直接 borrow してよい**（Controller 経由で間接参照のみに **限定しない**）。ただし Service は Repository / Coordinator を **知らない**。

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
applyStartupRestore()                             // initialAvailableDevices → Controller.select*
```

restore 成功後は **ユーザー明示操作ではない** → **同じ preference を re-save しない**。

## 6. Device enumeration readiness（client / media_kit 調査 — 正式決定）

### 6.1 client `4109a13` read-only 調査

| 項目 | 結果 |
|------|------|
| `MediaKitPlaybackService.availableDevices` | `_player.state.audioDevices` の **同期 getter**（L82–86） |
| `MediaKitPlaybackService.availableDevicesStream` | `_player.stream.audioDevices.map(...)` — **media_kit stream をそのまま expose**（L90–91） |
| media_kit 初期 `PlayerState.audioDevices` | **`[AudioDevice('auto', '')]` のみ**（`media_kit` 1.2.6 `player_state.dart` L114） |
| mapper 後 | `auto` 除外 → **`availableDevices == []`** になりうる |
| 完全一覧 | mpv **`audio-device-list` property event** で **非同期** 到着（`real.dart` L1401–1433） |
| `AudioOutputController` | **subscribe-before-snapshot**（L18–37）。`isLoadingDevices` は production で **常に false**（Settings Section コメント参照） |

### 6.2 media_kit `1.2.6` stream 契約（read-only 確認）

| 確認事項 | 結果 |
|----------|------|
| `Player.stream.audioDevices` の実体 | `audioDevicesController.stream.distinct(...)`（`platform_player.dart` L95–97, L379–380） |
| controller 種別 | **`StreamController<List<AudioDevice>>.broadcast()`** — **replay なし** |
| 新規 listener への current value emit | **保証なし** — broadcast は **subscribe 以降の event のみ** 受信 |
| stream への add タイミング | mpv `MPV_EVENT_PROPERTY_CHANGE` で `audio-device-list` 受信時 **のみ**（`real.dart` L1432–1433）。Player 生成時の `[auto]` 初期 state は **stream へ push されない** |
| 初期 `audio-device-list` event | mpv property observe 後に **非同期** 到着（コメント L1388: idle-active **前** に emit されうる） |
| subscription 前 event 取り逃し | **ありうる** — Controller が constructor で先に subscribe しても、Coordinator が **別 subscription** で `.first` すると **既 emit event を受け取れない** |
| `state.audioDevices` が authoritative になる API | **なし** — 同期 getter は常に最新 state を返すが、**「enumeration 完了済み」か「未完了の `[auto]` のみ」かを区別する公式 flag / Future は media_kit に存在しない** |

**Case A 不成立:** `availableDevicesStream.first` 単独方式は **正式採用しない**。新規 listener への replay 保証が **確認できなかった** ため、`.first` は **永久待機** しうる。

### 6.3 正式結論 — 未 ready empty と authoritative empty

起動直後の **`availableDevices == []` を authoritative missing と判断してはならない**。

| 状態 | 意味 |
|------|------|
| 同期 snapshot `[]`（auto のみ）かつ readiness **未 complete** | **enumeration 未完了の一時値** |
| readiness **complete** 後の `[]` | **authoritative empty** — 本当に selectable device なし |
| readiness **complete** 後の non-empty | **authoritative snapshot** — restore 判定に使用 |

### 6.4 Phase G explicit readiness contract（正式 — Case B）

`availableDevicesStream.first` は **廃止**。Phase G 実装要件として **`AudioOutputService` に explicit readiness API** を追加する。

```dart
/// Initial device enumeration が authoritative になった時点の snapshot を返す。
///
/// - [AudioOutputCapabilities.supportsDeviceEnumeration] == true:
///   最初の authoritative enumeration 完了で **一度だけ** complete。
///   返却 list は [availableDevices] と同型（`auto` 除外済み）。
///   0 件でも complete する（authoritative empty）。
/// - supportsDeviceEnumeration == false:
///   **即 complete**（fake readiness event **禁止**）。
///
/// 複数 await は同一 Future を共有し、late awaiter は **即 complete** + キャッシュ snapshot。
Future<List<AudioOutputDevice>> get initialAvailableDevices;
```

名称は実装 Phase で `initialDeviceEnumerationReady` 等へ調整可。**意味は固定**: 「initial enumeration が authoritative になった」。

**Concrete 実装方針（`MediaKitPlaybackService`）:**

| 項目 | 方針 |
|------|------|
| authoritative 信号 | 最初の mpv `audio-device-list` property change を処理した時点 |
| complete 値 | その時点の mapped `availableDevices`（`auto` 除外） |
| late awaiter | 既 complete なら **即** キャッシュ snapshot を返す — stream event 取り逃し **不影响** |
| **禁止** | `Future.delayed` / `Timer` / timeout 後 empty 扱い / `availableDevicesStream.first` への依存 |

Coordinator startup restore は **`await audioOutputService.initialAvailableDevices`** を使用する。**stream timing 単独には依存しない**。

authoritative snapshot **取得後**:

| 条件 | restore 判定 |
|------|-------------|
| A.id ∈ devices | **present** → `selectDevice(A.id)` 試行 |
| A.id ∉ devices | **missing at startup**（§12） |
| devices が `[]` | **authoritative empty** — specific 復元 **不可**（missing 扱い） |

### 6.5 Android / iOS

| 条件 | 方針 |
|------|------|
| `supportsDeviceEnumeration == false` または `supportsPersistentDeviceId == false` | **readiness 待機不要** — `initialAvailableDevices` は即 complete。specific-device restore **自体を行わない** |
| fake readiness | **禁止** — OS-managed route を enumeration 完了と **偽装しない** |

### 6.6 Enumeration failure

initial enumeration が **完了しない / 失敗** する platform 契約が必要な場合:

| 項目 | 正式 |
|------|------|
| startup restore | **適用しない** |
| runtime | **current route 維持** — Presentation だけ default に **書き換え禁止** |
| saved Preference | **削除しない** |
| UI | **Non-blocking restore / enumeration error**（Playback Blocking Error **禁止**） |
| timeout 偽装 | **禁止** |

### 6.7 Readiness 待機中の race / dispose

| ケース | 方針 |
|--------|------|
| readiness 待機中に user selection | **user intent 最優先** — `_restoreGeneration` で stale restore 禁止（§15.4） |
| readiness Future 完了後 | generation を再確認 — 待機中に user select 済みなら **restore を開始しない** |
| readiness 待機中に Coordinator dispose | Future 完了後も restore selection / state update / error notification **禁止** |

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

### 7.1 `AudioOutputService` readiness API（Phase G 追加）

§6.4 の `initialAvailableDevices` を `AudioOutputService` abstract interface に追加する。Controller は **readiness Future を expose しない** — Coordinator が Service を直接 await する。

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
3. capabilities 確認
4. supportsPersistentDeviceId == false → restore skip（Android / iOS）
5. await audioOutputService.initialAvailableDevices   // §6.4 authoritative snapshot
6. authoritative snapshot 取得
7. saved systemDefault → Controller.selectSystemDefault()
8. saved specific A かつ A ∈ devices → Controller.selectDevice(A.id)
9. saved specific A かつ A ∉ devices → §12 missing
10. select 失敗 → §11 restore failure
11. enumeration failure → §6.6
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
| load / restore / **readiness 待機** 中の user selection | **user 最優先** |
| stale restore completion | route **上書き禁止** |
| generation | Coordinator 側 **`_restoreGeneration`**（Persistence / startup intent **専用**）。Controller `_selectGeneration` と **役割重複させない** |

readiness 待機中 dispose — §6.7。

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
| readiness **未 complete** の `[]` ≠ authoritative empty |
| **`initialAvailableDevices` complete 後** に restore 判定 |
| authoritative non-empty → restore |
| authoritative empty → specific missing |
| stream event を **subscribe 前に取り逃しても** `.first` 永久待機 **しない**（explicit API が complete する） |
| **arbitrary delay / timeout なし** |
| readiness 待機中 user selection → user 優先 |
| readiness 待機中 dispose → post-complete restore **禁止** |
| enumeration failure → restore skip、preference 保持 |

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
| nainai-client `4109a13` `media_kit_playback_service.dart` | 同期 `availableDevices` / stream passthrough |
| nainai-client `4109a13` `audio_output_controller.dart` | subscribe-before-snapshot |
| nainai-client `4109a13` `audio_output_service.dart` | Phase G で `initialAvailableDevices` 追加 |
| `media_kit` 1.2.6 `platform_player.dart` L379–380 | `audioDevicesController` = **broadcast、replay なし** |
| `media_kit` 1.2.6 `real.dart` L1401–1433 | `audio-device-list` 非同期 property event |
| [settings-persistence.md](settings-persistence.md) | storage / durability |
| [audio-output-hot-unplug.md](audio-output-hot-unplug.md) | Phase D 境界 |

## 21. 状態まとめ

| 項目 | 状態 |
|------|------|
| Phase G design | **Design Complete** |
| Phase G implementation | **Not Implemented** |
| Phase D → Preference | **B — runtime only fallback, persist 保持** |
| Dependency | **Coordinator → Controller + Repository（一方向）** |
| Readiness | **`AudioOutputService.initialAvailableDevices`**（`availableDevicesStream.first` **廃止**） |
| Truth model | **`desiredPreference` / `lastPersistedPreference`** |
