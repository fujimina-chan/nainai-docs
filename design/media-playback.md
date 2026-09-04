# メディア再生（機能設計）

初期 MVP / Phase 2 における、選択済み音声・動画ファイルのアプリ内再生機能を定義します。

本ドキュメントは主に **何を実現するか** を定めます。UI レイアウトは [phase2-ui.md](phase2-ui.md)、視覚ルールは [design-system.md](design-system.md)、採用技術は [media-technology.md](../architecture/media-technology.md) を参照してください。

関連:

- [MVP 要件](../requirements/mvp-requirements.md)
- [Phase 2 UI](phase2-ui.md)
- [Design System](design-system.md)
- [メディアファイル選択](media-selection.md)
- [システム概要](../architecture/system-overview.md)
- [メディア技術選定](../architecture/media-technology.md)
- [ADR-0005 media_kit](../adr/0005-media-kit.md)

## 1. 概要

[media-selection.md](media-selection.md) で選択されたローカルメディアファイルを、アプリ内で再生する。

バックエンド通信・DB・認証・クラウド同期は使用しない。

```text
選択済みローカルファイル
       ↓
読み込み（media_kit）
       ↓
アプリ内再生（音声 / 動画）
```

## 2. 前提

| 項目 | 内容 |
|------|------|
| 対象プラットフォーム（MVP） | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |
| 入力 | ファイル選択で得た参照（音声または動画） |
| 元ファイル | 再生操作によってコピー・移動・削除・上書き・編集しない |
| 採用技術 | `media_kit` 1.2.6 / `media_kit_video` 2.0.1 / `media_kit_libs_video` 1.0.7 |

- 音声・動画の両方を扱うため video 用構成を使用する
- `media_kit_libs_audio` と `media_kit_libs_video` は同時使用しない

## 3. Phase 2 の必須操作

| 操作 | 説明 |
|------|------|
| Select File / Select Another File | ファイル選択（詳細は media-selection.md） |
| Play / Pause | 同一 Primary Control を状態で切替 |
| Stop | 再生を止め、位置を先頭（00:00）へ戻す。Media はロードされたまま |
| Seek | 再生位置を任意位置へ変更する |
| Volume | アプリ内の再生音量を変更する（% 表示・Mute / Unmute 含む。下記 §9） |
| Repeat OFF / ONE | Phase 2 のリピート（下記） |

これ以外の Playback 操作を Phase 2 に追加しない（Previous / Next / Shuffle / 10 秒送り戻し / Fullscreen / Subtitle 等は対象外）。

## 4. 必須表示

| 表示 | 説明 |
|------|------|
| ファイル名 | 現在扱っているメディアのファイル名 |
| Current Time | 現在位置（Monospace。横幅が揺れないこと） |
| Total Duration | メディア全体の長さ（同上） |

### 動画の場合

- 映像をアプリ内に表示する（画面の主役）

### 音声の場合

- 空の Video Box を表示しない
- Static Audio Placeholder + File Name + Audio 種別表示
- Artist / Album / Codec 等の未取得 Metadata は表示しない
- Fake Dynamic Visualizer は禁止

レイアウトの詳細は [phase2-ui.md](phase2-ui.md)、トークンは [design-system.md](design-system.md) を参照。

## 5. 再生状態（Media State）

Phase 2-3 時点で、状態正本は **MediaController** が保持する immutable な **MediaState** とする。

`MediaPlaybackStatus`:

| 状態 | 意味 |
|------|------|
| `unselected` | メディアが選択されていない |
| `loading` | 選択後、再生準備のために読み込み中 |
| `ready` | 読み込み成功。まだ再生していない、または再生可能な待機 |
| `playing` | 再生している |
| `paused` | 一時停止。Position を保持 |
| `stopped` | 再生を止め、Position が先頭（00:00） |
| `error` | 読み込みまたは再生に失敗した（Blocking） |

- Audio / Video 種別は状態とは別に Media 情報として保持する。UI は状態から描画する
- `isPlaying` / `isPaused` / `isLoading` 等の **重複 Boolean State は追加しない**
- `loading` 中は Play / Pause / Stop / Seek を操作不可とし、UI だけでなく Controller 側でも拒否する

### MediaController（Phase 2-3・実装済み）

- `ChangeNotifier` を extends
- Constructor Injection: `MediaSelectionService` / `MediaPlaybackService`
- Package 固有型を Controller へ漏らさない
- Public API: `selectMedia` / `play` / `pause` / `stop` / `seek` / `setVolume` / `setRepeatMode` / `toggleRepeatMode` / `state` / `dispose`

### 状態遷移（概念）

```text
unselected
  │ Select File（確定）
  ▼
loading
  ├─ 成功 ──▶ ready（自動再生なし、error clear）
  └─ 失敗 ──▶ error（Blocking。旧 Media へ rollback しない）

ready / stopped
  │ Play
  ▼
playing
  ├─ Pause ─────▶ paused ──(Play)──▶ playing
  ├─ Stop ──────▶ stopped（位置は 00:00、Media はロード維持、repeatMode 維持）
  ├─ 終端到達
  │    ├─ Repeat OFF ──▶ seek(0) → stopped（00:00）。Player.stop() は使わない
  │    └─ Repeat ONE ──▶ media_kit single に委譲。Controller は stopped にしない
  └─ 失敗 ──────▶ error（Blocking）または Non-blocking（status 維持）

いずれの状態でも（error 含む）Select Another File が可能
  │
  ▼
選択確定前は旧 State を維持
選択確定後 → selectedMedia=新 / loading / position=0 / duration=0
（volume / repeatMode は維持。失敗時も旧 Media へ rollback しない）
```

ファイル選択のキャンセルは再生状態を変更しない（[media-selection.md](media-selection.md)）。キャンセルは Error ではない。

## 6. Play / Pause

同一の Primary Control を状態によって切り替える。

| 状態 | Primary Control |
|------|-----------------|
| `ready` / `paused` / `stopped` | Play |
| `playing` | Pause |

| 操作成功時 | status | position |
|------------|--------|----------|
| `play` | `playing` | （変更しない） |
| `pause` | `paused` | 維持 |

## 7. Stop の意味

nainai における Stop は次を意味する。

1. Pause 相当に再生を止める
2. Position を **00:00** へ戻す
3. Media 自体はロードされたままとする
4. Repeat 設定は維持する

その後 Play すると先頭から再生できる。

| 操作 | 再生 | 位置 | Media | status（成功時） |
|------|------|------|-------|------------------|
| Pause | 止める | 保持する | ロード維持 | `paused` |
| Stop | 止める | 00:00 へ戻す | ロード維持 | `stopped` |

### 実装確認

nainai Stop は次で実現する。

- `pause`
- `seek(Duration.zero)`

**`Player.stop()` は使用しない。**

理由: media_kit の `stop` は Media を unload するため、nainai の Stop 仕様（Media loaded 維持）と異なる。

詳細な Package 挙動は [media-technology.md](../architecture/media-technology.md) を参照。

## 8. Seek

- ユーザーが再生位置を任意位置へ変更できる
- Phase 2 では Seek UI を提供する（レイアウトは phase2-ui.md）
- 時間精度は今回固定しない

MediaController でも `0 <= position <= duration` に clamp する（負値は zero。duration 既知なら上限を duration）。Service 側 Guard との二段防御。

将来の分割・タイムラインでは、より高精度な時間管理が必要になる。本設計のシークは、その将来要件を阻害しないこと（位置を「時間」として扱えること）を前提とする。

## 9. Volume

- Phase 2 では **アプリ側の再生音量** を変更できる
- **OS 全体の音量設定を変更する機能ではない**
- Volume Slider 近傍に現在音量を **整数パーセント**（例: `0%` / `20%` / `50%` / `100%`）で表示する。表示正本は `MediaState.volume`（`0.0`～`1.0`）。Volume 専用の表示状態は持たない
- Volume アイコンで Mute / Unmute できる（UI 配置は [phase2-ui.md](phase2-ui.md)）

| 境界 | スケール |
|------|----------|
| nainai 内部（MediaController / MediaState / UI） | `0.0`～`1.0`（MediaVolume で clamp） |
| media_kit 1.2.6 | `0`～`100` |

変換は Playback Service 境界で行い、media_kit の Volume scale を Controller / UI へ漏らさない。mpv softvol の 3 乗特性を相殺する補正の詳細は [media-technology.md](../architecture/media-technology.md) を正とする。

Media 未選択でも Session 設定として Volume 変更可能とする。

### Mute / Unmute（Phase 2-5 実装済み）

Volume アイコンを操作可能とする。

| 状態 | アイコン | click 時 |
|------|----------|----------|
| Volume > 0 | Volume | `0%` へ（Mute） |
| Volume = 0 | Volume Off | 最後の非ゼロ Volume へ復元（Unmute） |

- Mute 状態の正本は **`MediaState.volume == 0`** とする。公開された `isMuted` 等の二重状態は持たない
- 最後の非ゼロ Volume は Controller 内部の補助値として保持する
- Slider から直接 `0%` にした場合も Mute として扱い、アイコンから以前の非ゼロ Volume へ復元可能
- Mute 中に Slider を `0` より大きくすれば自然に Unmute となる

### Volume 低域特性（Phase 2-5・Windows: Resolved / 実機確認済み）

Windows 実機で「UI 10% 付近でほぼ無音」となる現象を調査した結果、nainai 側の値渡しバグではなく、mpv softvol の 3 乗特性との組み合わせによるものと確認した。

`MediaKitVolumeMapper` による逆 3 乗補正後、Windows 実機で 0 / 1 / 5 / 10 / 20 / 30 / 50 / 75 / 100% を確認し、操作感に問題なし。**Windows 実機では本問題を Resolved として扱う。**

Android / iOS では `MediaKitVolumeMapper` による補正実装は共通だが、**実機確認は未実施**。

これは「人間の聴感を数学的に線形化する」仕様ではない。目的は **mpv 内部の 3 乗変換を相殺し、nainai のユーザー Volume と mpv 適用後の振幅比を整合させること** である。

## 10. Repeat

Phase 2 の Repeat は `off` / `one` のみ。

| 設定 | 終端到達時の挙動 |
|------|------------------|
| OFF | MediaController が `seek(Duration.zero)` → `stopped` + position zero |
| ONE | media_kit の `PlaylistMode.single` に委譲。Controller は manual seek/play せず、`stopped` にしない |

- `toggleRepeatMode`: off → one / one → off
- Repeat ONE Active は Accent Color で示す（[design-system.md](design-system.md)）
- Repeat ONE の日本語 Tooltip / Semantics: **「この曲を繰り返し再生」**（[localization.md](localization.md) §3）
- File switch / Stop 後も **アプリ起動中は保持**
- Repeat ALL は将来機能（Phase 2 では禁止）
- completed を受けて手動で `seek` → `play` する Repeat は行わない

Package 側の `PlaylistMode` 対応は [media-technology.md](../architecture/media-technology.md) を参照。

## 11. Load とファイル切り替え

### Load

`SelectedMedia.sourceUri` を再生基盤へ渡し、**自動再生しない**（open 時 `play: false`）。

| 結果 | MediaState |
|------|------------|
| 成功 | `status = ready`、`position = zero`、error clear、自動再生なし |
| 失敗 | `selectedMedia` は新 Media のまま、`status = error`、`position`/`duration` = zero。旧 Media へ rollback しない |

### ファイル切り替え

Media A 使用中に B を選択した場合:

1. **B 選択確定前** → A の State を維持
2. **B 選択確定後** → `selectedMedia = B`、`status = loading`、`position`/`duration` = zero
3. `volume` / `repeatMode` は維持
4. 元ファイル自体には変更を加えない
5. B の load 失敗時も A へ rollback しない

切替時に、旧 Media へ対して nainai Stop や `Player.stop()` を必須としない。**新 Media をそのまま open** する。

### generation 二段防御（実装済み）

| 層 | 役割 |
|----|------|
| MediaKitPlaybackService | load serialization / generation |
| MediaController | `_loadGeneration` による stale async result の state 更新抑制 |

詳細は [media-technology.md](../architecture/media-technology.md)。

## 12. Stream 購読（completed / playing / position 等）

MediaController は Playback Service の Stream を購読済み（Phase 2-3）。

- **`playing == false` は `paused` へ自動遷移しない**
- `playing == true` のとき、矛盾しない状態なら `playing` へ同期可能
- Playback status 遷移の正本は **MediaController**
- `position` / `duration` / `volume` は正規化・clamp のうえ、値が変わった場合のみ State 更新

| Repeat | completed 時 |
|--------|----------------|
| OFF | `seek(Duration.zero)` → `stopped` + position zero。`Player.stop()` は使わない |
| ONE | media_kit single に委譲。Controller は manual seek/play せず、`stopped` にしない |

### dispose

#### Current（Phase 2-3 実装済み）

基本順:

1. Controller 所有 Subscription を cancel
2. `MediaPlaybackService.dispose()`
3. `super.dispose()`

dispose 後に `notifyListeners` しない Guard を持つ。

**Current ownership:** `MediaController` が注入 `MediaPlaybackService` を dispose する（[media-technology.md](../architecture/media-technology.md) §Application Composition）。

#### Phase C 以降（Implemented）

Audio Output Phase C で `MediaKitPlaybackService` を `MediaController` と `AudioOutputController` が共有するため、Service dispose は `NainaiApp` が担当。詳細は [audio-output-settings.md](audio-output-settings.md) §6 / [audio-output-composition.md](audio-output-composition.md)。

## 13. エラー

### Blocking Error

現在の Media を再生できない場合。Media 画面を Error 表示へ切り替えてよい。

例: Blocking Error の日本語は ARB `loadFailed`（「ファイルを読み込めませんでした。」）等。文言正本は [localization.md](localization.md)。

- Select Another File を提供する
- 技術的 Exception は表示しない
- 原因が確定していない場合、`Unsupported format` / `File corrupted` などを断定しない
- Error String から `unsupportedMedia` を推測しない
- 色: Semantic Error（Red）

### Non-blocking Error

現在 Media を維持可能な場合、Media 画面を Error 画面へ置き換えない。

- Notification / Banner を使用する
- 色: Semantic Warning（Amber）

### Error 区分と扱い（Phase 2-3）

| 区分 | 扱い |
|------|------|
| `selectionFailed` / `unsupportedMedia` | **non-blocking**。現在 Media / status を維持し `error` のみ更新 |
| `operationFailed` | 原則 **non-blocking**。status 維持、error 更新 |
| `loadFailed` / `mediaUnavailable` / `playbackFailed` 等の fatal | **blocking** として `status = error`（操作文脈を考慮） |

`MediaSelectionException` は MediaController で `MediaError` へ変換する。

media_kit では、一部失敗が open Future の throw ではなく後続 error stream として流れる場合がある。Service 単独では完全分類できない。

Phase 2-3 実装: `errorStream` から `playbackFailed` を受け、**MediaController の現在 status が `loading` なら `loadFailed` へ寄せる**。これは Controller 状態を利用した Phase 2 範囲の分類であり、media_kit の非同期 error を 100% 正確に load/playback 由来へ識別できるとは断定しない。

### Error clear（Phase 2）

- 新 Media 選択 + load 成功 → error clear
- 成功した操作 → 関連する selection / operation 系 non-blocking error を clear 可能
- blocking error → 無関係な Volume 操作成功等では消さない。正常な新 Media load 等で復旧
- 過剰な Error 履歴管理は行わない

### Unknown Error / 問題報告

将来的に問題報告機能へ接続予定とする。Phase 2 では Report ボタンを実装しない。

方針共通:

- アプリ全体をクラッシュさせない
- エラー後も別ファイル選択など通常操作へ復帰できること

## 14. UI との対応

必要 UI の配置・Desktop / Mobile 差分は [phase2-ui.md](phase2-ui.md) を正とする。

**Windows（Implemented `68ff1b4`）:** Playback Controls は `Scaffold.bottomNavigationBar` 固定 Bottom Panel。Audio / Video は Panel 上の残り領域を中央配置。narrow でも Mobile layout へ切り替えない。詳細は [phase2-ui.md](phase2-ui.md) §4.1。

Design Token・色・余白・形状は [design-system.md](design-system.md) を正とする。

## 15. OS 別の考慮

Windows / Android / iOS では、メディア再生の実装詳細が異なる可能性がある。

Android / iOS ではサンドボックスや権限・ファイルアクセス制約が再生可否に影響しうる。詳細は実装時に確定する。iOS で未検証の挙動を確定仕様にしない。

Windows では Phase 2 で OS 標準 Window Chrome を使用する（独自の Minimize / Maximize / Close は作らない）。詳細は [phase2-ui.md](phase2-ui.md)。

採用パッケージ・初期化・lifecycle は [media-technology.md](../architecture/media-technology.md) を参照。

開発環境上の Known Issue は [development-environment.md](../architecture/development-environment.md) を参照。

## 16. ライセンス注意

`media_kit` はネイティブメディアライブラリやコーデック等を利用する。パッケージ本体のライセンスだけで製品全体の利用可否を判断してはいけない。

**製品リリース前に、推移依存を含めたライセンス監査を必須とする。**（今回は監査未実施）

## 17. 将来機能との境界

Phase 2 / MVP の再生には含めない。

- Previous / Next / Shuffle
- Repeat ALL
- Playlist / Library / Folder Playback
- 波形 / Dynamic Visualizer
- タイムライン編集 / 区間設定 / 分割
- 歌詞 / 字幕 / Fullscreen
- メタデータ編集・未取得 Metadata の表示
- ファイル出力 / 保存先設定
- Equalizer / Compressor
- Drag & Drop
- Editor Navigation

本設計は、これら将来機能を阻害しないこと（時間位置の扱い、単一メディア切替モデル、非破壊方針、Repeat の拡張余地）を前提とする。

## 18. 未確定事項

- **正式な再生対応形式一覧**（rough classification とは別。Selection 側の拡張子表は再生保証ではない）
- Banner の自動消失時間・clear 専用 UI / API
- 状態管理ライブラリ / ルーティングライブラリ
- 分割処理技術 / FFmpeg 採用有無
