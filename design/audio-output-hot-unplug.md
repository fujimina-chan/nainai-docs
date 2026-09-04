# Windows 音声出力 hot unplug / automatic fallback（Phase D 詳細設計）

Audio Output **Phase D** — Windows 向け **選択中 specific device 消失時の自動 System Default fallback** の実装前詳細設計正本。

Service / platform 抽象の正本は [audio-output.md](audio-output.md)。Phase C Settings UI は [audio-output-settings.md](audio-output-settings.md)。Non-blocking Banner パターンは [phase2-ui.md](phase2-ui.md) §5。

関連:

- [音声出力デバイス選択](audio-output.md)
- [Windows 音声出力 Settings UI（Phase C）](audio-output-settings.md)
- [Phase 2 UI](phase2-ui.md)
- [UI Localization](localization.md)

## 1. 位置づけ

| Phase | 内容 | 状態 |
|-------|------|------|
| A | 共通 Model / Service 抽象 / Capability | **Implemented** |
| B | Windows `MediaKitPlaybackService` が `AudioOutputService` も実装 | **Implemented** |
| C | Windows Audio Output Settings subsystem | **Implemented**（I-3B） |
| **D** | **Windows hot unplug / automatic fallback**（本書） | **Implemented**（コード `c5c979e`）/ 実機 Acceptance **Pending** |
| E | Android platform output picker UI | **Implemented** / 実機 **Pending** |
| F | iOS route picker UI | **Implemented** / compile・実機 **Pending** |
| G | `AudioOutputPreference` 永続化 | **Implemented**（コード）/ 実機 **Pending** |

**Design Complete の範囲:** reconciliation scheduler / fallback episode / retry / notification event / ownership / race protection / reconnect / Phase G 境界 / Playback invariant / テスト方針 / Localization キー案。

**含まない:** Android / iOS hot unplug（Phase E / F）、Preference 永続化（Phase G）、Settings Launcher 物理配置（別 docs レーン）。

### 1.1 Phase C との境界

| Phase C（確定済み） | Phase D（本書） |
|---------------------|-----------------|
| ユーザー **明示操作** による device 選択 | **runtime** に specific device が一覧から消失 |
| missing selected device 時 **自動 fallback しない** | **自動** `selectSystemDefault()` |
| Settings 内 `audioOutputSelectedDeviceUnavailable` 警告 | fallback 成功後 System Default selected + disconnect **通知** |
| 不一致一時状態で Service 正本を維持 | fallback 成功後 `currentSelection = systemDefault` |

Phase C 実装済み client 上に Phase D を **追加** する。Phase C の手動選択 API / UI は維持する。

## 2. 目的

ユーザーが **specific device** を選択中に USB / Bluetooth / HDMI 等で device が消失しても、**再生機能を壊さず** System Default へ fallback する。

**禁止:** Playback Blocking Error、Media Error 画面置換、Player 再生成、Media 再 open、再生位置リセット。

## 3. Reconciliation timing（確定）

### 3.1 正式仕様: scheduled microtask coalescing

Phase D の reconciliation timing は **1 回の scheduled microtask による coalescing** のみとする。

| 採用 | 禁止 |
|------|------|
| `scheduleMicrotask` による 1 回 coalescing | Timer ベースの任意 ms debounce |
| 同一 event turn 付近の stream 更新を 1 回にまとめる | magic number（50ms / 150ms 等） |
| microtask 時点の **最新 snapshot** で判定 | 古い event payload 単体を根拠に fallback |

**理由:** テストを時間依存にしない。user operation との競合は epoch / `isSelecting` で別途防ぐ。

### 3.2 Reconciliation scheduler

`availableDevicesStream` または `currentSelectionStream` 更新を受信した際、**即座に fallback しない**。

```text
stream event
    ↓
Controller が latest snapshot を更新（devices + selection）
    ↓
_scheduleReconciliation()
    ↓
reconciliationScheduled == true ?
    ├─ Yes → 追加 schedule しない（coalesce）
    └─ No  → reconciliationScheduled = true
              scheduleMicrotask(_runReconciliation)
    ↓
microtask 実行:
    reconciliationScheduled = false
    最新 snapshot で §3.3 条件評価
```

**古い event payload 単体** を fallback 根拠に **しない**。

### 3.3 Reconciliation 条件（microtask 実行時 — すべて必須）

次を **すべて** 満たす場合のみ **fallback episode 候補** とする:

| # | 条件 |
|---|------|
| 1 | Controller が dispose されていない |
| 2 | `currentSelection.mode == specificDevice` |
| 3 | `currentSelection.deviceId` が non-empty |
| 4 | `availableDevices` に同一 `deviceId` が **存在しない** |
| 5 | user selection operation が優先されていない（§6） |
| 6 | 同一 disappearance について fallback **済み / 実行中** ではない（§5） |

`currentSelection == systemDefault` の場合は **何もしない**（OS routing に委ねる）。

### 3.4 ID 判定

**`AudioOutputDevice.id` のみ**（`label` は identity に使わない）。

## 4. Fallback episode（確定）

同一 disconnect に対する重複 fallback / notification を防ぐ **正式概念**。

### 4.1 識別情報（最低限）

| 属性 | 説明 |
|------|------|
| `episodeId` | monotonic。新 episode 開始ごとに increment |
| `missingDeviceId` | 消失した specific device id |
| `selectionEpoch` | episode 開始時点の selection epoch snapshot |
| `attemptCount` | 当該 episode の fallback 試行回数（§7） |
| `inFlight` | fallback async 実行中 |

### 4.2 Episode lifecycle

| イベント | 扱い |
|----------|------|
| §3.3 条件成立 + episode 未開始 | 新 episode 開始 → fallback attempt |
| 新しい specific device をユーザーが明示選択 | 現 episode **無効化** |
| missing device 状態の解消（device 再出現） | episode **終了 / 無効化**。auto restore **しない**（§11） |
| fallback 成功 | episode **完了** |
| 最大 attempt 到達かつ失敗 | episode **終了**（retry 不可） |

## 5. Fallback policy ownership（確定）

| 配置 | 判定 |
|------|------|
| **`AudioOutputController`**（Application 層） | **採用** — fallback 判断 / episode / retry / notification / epoch |
| `MediaKitPlaybackService` | **実行のみ**（`selectSystemDefault()`）。policy 埋込み **禁止** |
| 専用 Coordinator | Controller 複雑化時の内部 refactor 候補 |

Presentation は Controller 経由のみ。media_kit 型を露出しない。

## 6. User selection 優先と epoch（確定）

### 6.1 原則: latest user intent wins

ユーザーが `selectDevice(B)` / `selectSystemDefault()` を **開始** した時点で **selection epoch** を進める。

automatic fallback **開始時** にも開始 epoch を snapshot する。

fallback **完了時**:

```text
開始時 epoch != 現在 epoch
    → stale completion。state / notification / error を上書きしない
```

### 6.2 Phase C selection generation との統合

Phase C 実装時の既存 `_selectGeneration` 等を **確認** し、Phase D は **同一 selection ordering model へ統合** する。

独立した二重 epoch 機構を **乱立させない**。

### 6.3 `isSelecting` との関係（確定）

| 禁止 | 採用 |
|------|------|
| `isSelecting == true` → fallback **永久 skip** | user selection **進行中** は automatic fallback を **開始しない** |
| | user operation **完了後**、最新 snapshot で **再 reconcile**（`_scheduleReconciliation()`） |

operation 終了時に再評価可能な設計とする。

### 6.4 シナリオ

- **fallback 中に user selects B** → epoch 進行。stale fallback discard。B 維持
- **fallback 中に user selects System Default** → 同上
- **stream 順序逆転** → microtask 時点 latest snapshot で評価
- **Device A 消失 → fallback 中 → A 復帰** → reconciliation で条件 false。auto restore **禁止**

## 7. Retry policy（確定）

### 7.1 正式仕様

1 fallback episode あたり **最大 2 attempts**（初回 + automatic retry **最大 1 回**）。

| 禁止 | 内容 |
|------|------|
| 「1–2 回」の曖昧表現 | 上限は **2 attempts** で固定 |
| 固定時間 Timer retry | 500ms backoff 等 **禁止** |
| 無限 retry | **禁止** |

### 7.2 Retry trigger（確定）

初回 fallback 失敗後、**次に Audio Output state へ意味のある更新** が入り reconciliation が **再実行** された場合 **のみ** retry 可能。

retry 条件（**すべて** 必須）:

| # | 条件 |
|---|------|
| 1 | 同一 fallback episode |
| 2 | `attemptCount == 1`（初回失敗済み） |
| 3 | specific selected device が依然 `availableDevices` に **存在しない** |
| 4 | `currentSelection` が依然その specific device |
| 5 | newer user selection なし（epoch 一致） |
| 6 | Controller not disposed |

retry 後も失敗 → 当該 episode で automatic retry **終了**。

**途中の retry 可能な失敗** をユーザーへ **毎回通知しない**（§10）。

## 8. Fallback action（確定）

reconciliation + episode 開始後:

```text
AudioOutputController._executeAutomaticFallback(episode)
    ↓
AudioOutputService.selectSystemDefault()
    ↓
Windows: setAudioDevice(AudioDevice.auto())   // name == 'auto'
    ↓
成功: currentSelection → systemDefault
```

- Player 再生成 / Media 再 open **禁止**（§13）
- Phase D から **Persistence API / SettingsRepository を呼ばない**（§14）

## 9. Notification event model（確定）

### 9.1 `AudioOutputNotification`（event model）

単なる `String` field や rebuild ごとに再表示される state field は **禁止**。

```dart
// 概念モデル（設計）
class AudioOutputNotification {
  final int notificationId;       // monotonic。必須
  final AudioOutputNotificationType type;
  // type: fallbackToSystemDefault | fallbackFailed
  // message は Localization key 相当（technical details 禁止）
}
```

### 9.2 One-shot delivery 契約（確定）

Presentation は **同一 event を一度だけ提示** できる契約とする。

| 要件 | 内容 |
|------|------|
| `notificationId` | Controller 上 **monotonic**。必須 |
| Presentation | `lastHandledNotificationId` より **大きい id のみ** 表示 |
| 代替 | consume / acknowledge API（実装 Phase で A/B 選択可） |
| 禁止 | rebuild ごとに同一 Banner を再表示 |

### 9.3 Success notification

fallback **成功** 時、**1 episode につき 1 回**:

| キー | ja | en |
|------|----|----|
| `audioOutputFallbackToSystemDefault` | 選択していた音声出力デバイスが利用できなくなったため、システム既定に切り替えました | The selected audio output device became unavailable. Switched to System Default. |

Non-blocking Banner（[phase2-ui.md](phase2-ui.md) §5）。stream 更新ごとに **再表示しない**。

### 9.4 Failure notification

**最終 attempt も失敗** した時点で `fallbackFailed` を **1 回** 通知。

| キー | ja | en |
|------|----|----|
| `audioOutputFallbackFailed` | 音声出力をシステム既定に切り替えられませんでした | Could not switch audio output to System Default. |

初回失敗時点では retry 可能性が残るため、**原則ユーザー通知しない**。Controller 内部 error state には attempt 中状態を保持可。

### 9.5 persistent error との分離

| 概念 | 用途 |
|------|------|
| `AudioOutputNotification` | ephemeral one-shot UI event |
| `AudioOutputState.error` | persistent non-blocking error（最終失敗等） |

## 10. Error lifecycle（確定）

### 10.1 specific device disappeared（fallback 開始前）

- Phase C Settings: `audioOutputSelectedDeviceUnavailable`
- device unavailable 状態。automatic fallback 未実行

### 10.2 fallback 成功

| 項目 | 扱い |
|------|------|
| `currentSelection` | `systemDefault` |
| deviceUnavailable 系 error | **clear** |
| fallback failure error | **clear** |
| notification | success **1 回**（§9.3） |
| Settings UI | System Default **selected**（Controller snapshot に従う） |

### 10.3 最終 fallback 失敗

| 項目 | 扱い |
|------|------|
| `currentSelection` | **偽装しない**（System Default selected 表示にしない） |
| Settings UI | Phase C missing-device 表現を **維持可能** |
| error | fallback error を non-blocking で **保持** |
| notification | failure **1 回**（§9.4） |

Playback Blocking Error **禁止**。fallback 失敗を無かったことに **しない**。

## 11. Reconnect（確定）

fallback **成功後**:

```text
currentSelection = systemDefault
```

元 device が `availableDevices` に **復帰** しても:

- 新 fallback episode を開始 **しない**
- auto restore **禁止**
- ユーザーが Settings から **明示再選択** するまで System Default 維持

## 12. Phase C Settings UI との関係（確定）

- fallback **成功後**: `currentSelection` stream / snapshot に従い Settings UI は **System Default selected**
- Presentation が独自に selected state を **書き換えない**
- fallback **失敗時**: 存在しない specific device 選択中という Phase C missing-device 表現を **維持可能**

## 13. Playback invariant（確定）

Phase D operation によって **変更禁止**:

| 維持 | 禁止 |
|------|------|
| loaded media | Player 再生成 |
| playback position | Media reopen |
| playing / paused | `seek(0)` |
| Repeat | `stop` |
| Volume / Mute | `MediaController` state reset |

**Audio route のみ** 変更する。One Player 原則: `MediaKitPlaybackService` 1 / `Player` 1。

[audio-output.md](audio-output.md) §2.6。実装後 Windows 実機検証必須。

## 14. Phase G（Persistence）境界（確定）

Phase D が扱うのは **runtime `currentSelection` のみ**。

| 禁止 | 内容 |
|------|------|
| Phase D から `SettingsRepository` 等へ保存 | **禁止** |
| Phase G 仕様の先取り確定 | A/B は Phase G で決定 |

将来 Phase G 候補（参考のみ）:

- **A:** fallback 時 saved Preference も System Default へ
- **B:** runtime のみ System Default、preferred device は保持

[audio-output.md](audio-output.md) §2.5「永続化も更新」は **Phase G 連携目標**。Phase D 単体の必須条件 **ではない**。

## 15. State 拡張（最小）

| フィールド | 説明 |
|------------|------|
| `isFallingBack` | fallback in-flight |
| `pendingNotification` | 最新 `AudioOutputNotification?`（monotonic id 付き） |
| `error` | persistent non-blocking error |
| `_activeFallbackEpisode` | Controller 内部。State へ不必要に露出しない |

**意図的に持たない:** Playback status、Persistence 状態、reconnect auto-restore フラグ。

## 16. Application Composition

Phase C Composition を維持。Phase D は `AudioOutputController` **振る舞い拡張** が中心。`MediaController` へ fallback を注入しない。

Banner は Media subtree が `pendingNotification.notificationId` と `lastHandledNotificationId` で one-shot 表示。

## 17. テスト方針（実装 Phase 要求）

### 17.1 Reconciliation

| ケース |
|--------|
| devices event 単体では即 fallback しない |
| microtask 後の latest snapshot で判定 |
| multiple stream events が 1 reconciliation に coalesce |
| `systemDefault` 時 no-op |
| device 依然 exists 時 no-op |
| `_scheduleReconciliation` 二重呼び出しで microtask 1 回 |

### 17.2 Fallback / retry

| ケース |
|--------|
| specific device disappearance → fallback |
| success → `systemDefault` |
| first failure → retry eligibility のみ（ユーザー notification なし） |
| meaningful state update 後に最大 1 retry |
| second failure → retry 終了 |
| infinite retry なし |
| episode あたり最大 2 attempts |

### 17.3 Race

| ケース |
|--------|
| fallback 中 user selects B |
| fallback 中 user selects System Default |
| stale fallback success discard |
| stale fallback failure discard |
| stream order inversion |
| user selection 終了後 reconcile |

### 17.4 Notification

| ケース |
|--------|
| success notification 1 回 / episode |
| failure notification 1 回 / episode（最終失敗時） |
| duplicate stream で再通知しない |
| rebuild で同一 notification 再表示しない（notificationId） |

### 17.5 Reconnect / dispose / Playback

| ケース |
|--------|
| old device 復帰で auto restore しない |
| scheduled reconciliation 後 dispose → 副作用なし |
| fallback Future in-flight 中 dispose |
| dispose 後 notification / error / state 更新なし |
| position / playing-paused / Repeat / Volume-Mute 維持 |
| Media reopen なし |

## 18. Phase 境界まとめ

| Phase | 責務 |
|-------|------|
| **C** | ユーザー操作による選択 |
| **D** | runtime hot unplug / automatic fallback / disconnect 通知 |
| **E** | Android |
| **F** | iOS |
| **G** | persistence |

## 19. 状態

| 項目 | 状態 |
|------|------|
| Phase D 設計 | **Design Complete** |
| Phase D 実装 | **Implemented**（`c5c979e`） |
| `AudioOutputNotificationHost` | **Implemented**（App-level — Settings Shell 外） |
| Windows 実機 hot unplug / fallback 検証 | **Pending Acceptance**（[settings-shell.md](settings-shell.md) §19.5） |
| Phase I-3 baseline | client `dccf48f` — 526 PASS |
