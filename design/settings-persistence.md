# Common Settings Concrete Persistence（詳細設計）

`SettingsRepository` の **Concrete 技術選定** と実装 Phase 向け詳細設計正本。

関連:

- [アプリ共通 Settings 基盤](settings.md) — `AppSettings` / `SettingsController` / Repository 抽象 / async consistency
- [Application Composition](../architecture/media-technology.md)

**上位設計:** [settings.md](settings.md) §4.4 / §5 / §11 / §12 / §14 が Persistence **契約** の正本。本書は **Concrete 方式** の正本。

## 1. 位置づけ

| 項目 | 内容 |
|------|------|
| スコープ | `SettingsRepository` concrete 実装技術、保存 key、missing / invalid / failure 挙動、テスト境界 |
| 第一保存対象 | `settings.showTooltips`（`bool`） |
| client 参照 | `4109a13` — `AppSettings` / `SettingsController` / `SettingsRepository` interface **Implemented** |
| 状態 | **Design Complete** / **Concrete Technology Selected** / **Implementation Not Implemented** |

### 1.1 Status（確定）

| 項目 | 状態 |
|------|------|
| Common Settings Persistence（本書） | **Design Complete** |
| Concrete Technology | **Selected** — `shared_preferences`（§5） |
| Concrete Repository implementation | **Not Implemented** |
| Common Settings Core（Model / Controller / interface） | client `4109a13` **Implemented** |
| Tooltip Policy / General Section UI wiring | **Not Implemented**（Visual Tooltip **Always enabled**） |

### 1.2 Android minSdk compatibility（確定 — 2026-08-30 実値確認）

| 項目 | 値 |
|------|-----|
| `shared_preferences` 2.5.5 公式 Android 下限 | **SDK 24+** |
| nainai client 正式 minSdk | **24** |
| 確認ファイル | `nainai-client` `android/app/build.gradle.kts` L22 — `minSdk = flutter.minSdkVersion`（**override なし**） |
| Flutter SDK 定義 | `C:\flutter\packages\flutter_tools\gradle\src\main\kotlin\FlutterExtension.kt` L26 — `minSdkVersion = 24`（`local.properties` → `flutter.sdk=C:\flutter`） |
| 互換性 | **採用可能**（24 ≥ 24）。**client minSdk 変更は不要** |

## 2. 保存対象・対象外

### 2.1 対象（Common Settings Persistence）

| 現在 | 将来 |
|------|------|
| `settings.showTooltips` | Common Settings に追加される **軽量 preference**（bool / string / int 等の小規模値） |

### 2.2 対象外

| データ | 扱い |
|--------|------|
| `MediaState` / playback position / selected media | Media 層。本 Repository **対象外** |
| `AudioOutputPreference` | Phase G — **別 Repository / 別 key namespace**（[settings.md](settings.md) §3.4） |
| credentials / API key / token | 秘密情報 — 将来 **別責務**（secure storage 等） |
| large binary / cache | 本方式 **不向き**（§7.1） |

## 3. Security

Common Settings は **秘密情報ではない**。`showTooltips` 等は平文 key-value で十分。

| 採用しない（過剰） | 理由 |
|-------------------|------|
| secure storage | 秘密情報ではない |
| 暗号化 DB | スコープ・complexity に見合わない |

将来 credential 保存が必要になった場合は **別 Repository / 別設計** とする。

### 3.1 Critical data ではない — durability 境界

[shared_preferences README](https://pub.dev/packages/shared_preferences) の公式注意:

> Data may be persisted to disk asynchronously, and **there is no guarantee that writes will be persisted to disk after returning**, so this plugin **must not be used for storing critical data**.

| 区分 | 内容 |
|------|------|
| **Common Settings 対象** | UI preference / convenience settings / **再生成可能な軽量設定** のみ |
| **禁止対象** | credential / token / entitlement / 決済情報 / 唯一の重要データ / loss 不可 business data |
| **save 成功の意味** | `SharedPreferencesAsync` の write **Future が例外なく正常完了**したこと（§10.1） |
| **save 成功が意味しないこと** | OS / plugin が **物理 storage へ fsync 済み**であることの保証 |

`SettingsController.lastPersistedSettings` は **Repository.save() 正常完了時点の snapshot** であり、fsync 以上の durability を **持たない**（§10.2）。

## 4. Repository boundary（維持）

client `4109a13` の interface を **変更しない**。

```dart
abstract interface class SettingsRepository {
  Future<AppSettings> load();
  Future<void> save(AppSettings settings);
}
```

| 原則 | 内容 |
|------|------|
| 露出禁止 | `SharedPreferences` / platform 型を `SettingsController` へ **露出しない** |
| 変換 | Concrete 実装内で key ↔ `AppSettings` を変換 |
| 例外 | storage **read / write failure** は `save` / 一部 `load` で **throw**（§9） |
| ordering | revision / serialized save / latest-wins は **`SettingsController` 専任**（§11） |

## 5. 候補比較と正式決定

### 5.1 候補 A — `shared_preferences`（Flutter 公式 plugin）

| 項目 | 内容 |
|------|------|
| 概要 | 小規模 key-value の platform-native 永続化抽象 |
| pub.dev | [shared_preferences](https://pub.dev/packages/shared_preferences) — publisher: **flutter.dev** |
| 確認 version（2026-08-30） | **2.5.5**（`shared_preferences: ^2.5.5`） |
| package 要件（pub.dev） | Flutter **>= 3.38** / Dart **^3.10** |
| nainai 互換 | Flutter **3.47.0** / Dart **3.13.0** — **適合** |
| 対応 platform（公式） | Android **SDK 24+** / iOS 13.0+ / **Windows Any** / macOS 10.15+ / Linux / Web |
| 型 | `int` / `double` / `bool` / `String` / `List<String>` |
| API | Legacy `SharedPreferences`（将来 deprecated 方向）、**`SharedPreferencesAsync`**（新規推奨）、`SharedPreferencesWithCache` |

**公式 platform storage（SharedPreferencesAsync — 採用 API）:**

| Platform | 保存先（正式） |
|----------|----------------|
| **Windows** | roaming **AppData** directory |
| **Android** | **DataStore Preferences**（`SharedPreferencesAsync` **default backend**） |
| **iOS** | **NSUserDefaults** |

Android SharedPreferences backend へ **意図的に切り替えない**（DataStore default を維持）。

アプリコードが Windows path / Android native API / NSUserDefaults を **直接分岐しない**。

**`SharedPreferencesAsync` 採用理由（Legacy `SharedPreferences` 不採用）:**

| 理由 | 内容 |
|------|------|
| 新規実装向け | pub.dev が新規コードに **Async / WithCache 推奨** |
| cache なし | local cache なし — host platform storage から **毎回 async read** |
| coherence | multi-isolate / 外部 native 変更時の cache stale リスクを **避けやすい** |

**公式注意（要約）:**

- 非同期 persist。**critical data 用途は不可**（§3.1）
- write 後の disk persistence **保証なし**（§3.1）
- Legacy `SharedPreferences` は deprecated 方向 — **nainai 新規 Concrete では使用しない**

### 5.2 候補 B — 独自 JSON / file persistence

| 項目 | 内容 |
|------|------|
| 概要 | app support directory 等へ JSON ファイル read/write |
| 追加依存 | 通常 `path_provider` 等 |
| メリット | schema 自由度、ファイル単位 atomic write 設計可能 |
| デメリット | path / 権限 / locking / corruption 処理を **自前実装**、platform 差分テストコスト、MVP 1 bool に過剰 |

### 5.3 評価（MVP: Windows / Android / iOS）

| 評価軸 | A. `shared_preferences` | B. JSON file |
|--------|-------------------------|--------------|
| Windows / Android / iOS | **公式対応** | 可能だが自前 |
| Flutter 3.47 / Dart 3.13 | **適合**（client SDK `^3.13.0`） | 適合 |
| bool 等 small preference | **第一用途** | 過剰 |
| 非同期 API | **あり**（Async API） | 自前 |
| testability | mock initial values / fake Repository | temp dir + file IO |
| maintenance | **flutter.dev 公式** | 自前 |
| migration | plugin + key namespace で十分 | 自前 schema |
| corrupted / invalid | 型不一致 → null 扱い（§9.2） | parse 失敗処理自前 |
| portable path 管理 | **不要** | **必要** |
| dependency 量 | **1 package**（transitive あり） | `path_provider` + 自前コード |
| security | 非秘密データ向け | 同左 |
| backup / platform behavior | OS / plugin 定義に従う | 自前 |
| large data | **不向き**（公式も非推奨） | 可能だが対象外 |

### 5.4 正式決定（1方式）

**`shared_preferences` を Common Settings Concrete Persistence として正式採用する。**

| 項目 | 決定 |
|------|------|
| Package | **`shared_preferences`** |
| Version 候補 | **`^2.5.5`**（実装 Phase 開始時に pub.dev で最新 compatible を確認） |
| Dart API | **`SharedPreferencesAsync`**（新規コード推奨 API — pub.dev 2.3.0+） |
| Concrete class | **`SharedPreferencesSettingsRepository`** |
| 配置（client 第一候補） | `lib/features/settings/repositories/shared_preferences_settings_repository.dart` |

**不採用:** 独自 JSON file persistence（候補 B）— MVP 要件に対しコスト対効果が低い。

## 6. Concrete Repository 設計

### 6.1 クラス

```dart
class SharedPreferencesSettingsRepository implements SettingsRepository {
  SharedPreferencesSettingsRepository({
    required SharedPreferencesAsync preferences,
  }) : _preferences = preferences;

  final SharedPreferencesAsync _preferences;

  @override
  Future<AppSettings> load() async { /* §9 */ }

  @override
  Future<void> save(AppSettings settings) async { /* §10 */ }
}
```

**命名:** `SharedPreferencesSettingsRepository` を正式クラス名とする。

**非採用:** `SharedPreferencesSettingsRepository.create()` 等の **不要な async factory**。`SharedPreferencesAsync()` は **同期生成可能** — Composition Root で生成し **constructor injection** する（§6.2）。

### 6.2 生成（Composition Root）

```text
NainaiApp（Composition Root）
  1. final prefs = SharedPreferencesAsync()
  2. final repository = SharedPreferencesSettingsRepository(preferences: prefs)
  3. SettingsController(repository: repository)
  4. controller.initialize()
```

| 方針 | 内容 |
|------|------|
| injection | `SharedPreferencesAsync` を **constructor 引数** で注入（testability 優先） |
| 禁止 | Repository 内部での hidden `SharedPreferencesAsync()` 生成（テスト差し替え困難） |
| lifetime | `build()` ごとに Repository / Controller を **生成しない**（[settings.md](settings.md) §13） |

### 6.3 dispose

| 対象 | dispose |
|------|---------|
| `SharedPreferencesSettingsRepository` | **不要**（stateless wrapper。`SharedPreferencesAsync` を borrow） |
| Owner | **Composition Root** が生成。Controller は borrow のみ（client `4109a13` テストで確認済み） |

## 7. Key namespace と schema

### 7.1 保存 key（確定）

| AppSettings field | Storage key | 型 | 備考 |
|-------------------|-------------|-----|------|
| `showTooltips` | **`settings.showTooltips`** | `bool` | Localization ARB `showTooltips` と **無関係** |

- namespace **`settings.*`** を Common Settings 専用とする
- `AudioOutputPreference`（Phase G）は **`audioOutput.*`** 等 **別 namespace** — collision 禁止
- Legacy `SharedPreferences` 使用時、plugin が内部で `flutter.` prefix を付与する場合がある（[pub.dev SharedPreferences — prefix / migration](https://pub.dev/packages/shared_preferences)）。**アプリが参照する論理 key は `settings.showTooltips` で統一** し、Concrete 実装が plugin 詳細を吸収する

### 7.2 Schema 方針

| 項目 | 決定 |
|------|------|
| 現在 | **per-field key-value** — persisted field は `showTooltips` **1 個のみ** |
| 保存 | `save(AppSettings)` ↔ **`setBool('settings.showTooltips', …)` 1 回**（1 対 1） |
| `schemaVersion` | Phase 初期では **導入しない** |
| 2 個目追加前 | **multi-key atomicity / migration / schema** を **Persistence design 再レビュー必須**（§12） |

### 7.3 不向きデータ

`shared_preferences` は **large data / binary / 高頻度大量 write** に不向き。Common Settings の軽量 preference のみ対象とする。

## 8. missing value

| 条件 | Repository.load() | 返却 |
|------|-------------------|------|
| key 未存在 | **成功**（throw しない） | `AppSettings.defaults()` — `showTooltips: true` |

初回起動は **エラー扱いしない**（[settings.md](settings.md) §11.1）。

## 9. invalid value と read failure

### 9.1 invalid / 型不一致（field 単位 fallback — 確定）

| 条件 | 挙動 |
|------|------|
| `settings.showTooltips` が bool として読めない（型不一致、破損） | 当該 field を **`true`（default）** へ fallback。**load() は成功** |
| 未知の future key が store に存在 | **無視**。既知 field のみ `AppSettings` へ map |
| **repair write** | load 中に invalid を **自動上書き保存しない**。default として **解釈のみ**。次回正常 save で修復 |

**理由:** 単一 bool preference の corruption を app 全体 load failure にしない。read 時 repair write は **副作用を避ける**。

`SharedPreferencesAsync.getBool` は key 不存在または非 bool 値で **`null`** を返す — いずれも **missing と同様に default へ**。

### 9.2 storage read failure（throw — 確定）

| 条件 | 挙動 |
|------|------|
| platform storage 自体が read 不可 | **`load()` は throw** |
| 例 | 初期化失敗、想定外 platform exception |

Repository 内で **握り潰さない**。client `4109a13` `SettingsController` が catch し:

- `settings` / `lastPersistedSettings` は **`AppSettings.defaults()` 維持**
- `SettingsErrorType.loadFailed` を設定

（`settings_controller_test.dart` — load failure keeps default true）

### 9.3 load() 返却値と Controller の整合

| load 結果 | Controller |
|-----------|------------|
| 成功 + defaults（missing） | 正常 restore。error なし |
| 成功 + 保存値 | 正常 restore |
| throw | default 維持 + **loadFailed** |

## 10. save 成功契約と write failure

### 10.1 `Repository.save()` 成功の正式意味

| 項目 | 内容 |
|------|------|
| **成功** | `SharedPreferencesAsync.setBool(...)` 等の write **Future が例外なく正常完了** |
| **失敗** | write Future が throw → **`save()` も throw**（握り潰さない） |

shared_preferences の durability 保証 **以上** の意味（fsync 保証等）を **持たせない**（§3.1）。

### 10.2 `lastPersistedSettings` の設計上の意味

client `4109a13` `SettingsController.lastPersistedSettings`:

| 正式 | **意味しない** |
|------|----------------|
| **`SettingsRepository.save(AppSettings)` が正常完了した最後の snapshot** | OS が物理 storage へ **fsync 済み**であることの保証 |

field 名は **変更不要**。意味は [settings.md](settings.md) §4.4 と整合。

### 10.3 write failure と Controller

Controller（client `4109a13` 実装済み）が管理:

- serialized save queue
- latest revision wins
- 最新 revision save 失敗 → `lastPersistedSettings` へ revert + **saveFailed**
- 古い revision 失敗 → UI revert **しない**

Repository で **queue / revision / latest-wins を二重実装しない**（§11）。

## 11. Write ordering 責務

| 層 | 責務 |
|----|------|
| **SettingsController** | monotonic revision、serialized save、stale discard、revert、error |
| **SharedPreferencesSettingsRepository** | 1 回の `save(AppSettings)` を **正しく永続化** |

## 12. Atomicity と multi-key 境界

### 12.1 現在（1 persisted field — partial write 問題なし）

| 項目 | 内容 |
|------|------|
| persisted field | **`showTooltips` 1 個** |
| save 操作 | `save(AppSettings)` → **`setBool` 1 回**（1 対 1） |
| partial multi-key write | **発生しない** |

### 12.2 将来（2 個目 persisted field 追加前 — 再レビュー必須）

`SharedPreferencesSettingsRepository` が将来、

```text
settings.showTooltips
settings.xxx
settings.yyy
```

を **複数の独立 write** として保存する場合、`AppSettings` snapshot 全体の **transactional atomicity は保証できない**。

**正式ルール:** **2 個目の persisted Common Setting を追加する前に、Persistence design を再レビューする。** 無レビューでの複数 key sequential write 拡張は **禁止**。

### 12.3 将来レビュー時の選択肢（今回は未決定）

| 候補 | 概要 |
|------|------|
| **A** | 複数 key の **partial-write semantics** を正式許容 |
| **B** | `AppSettings` snapshot を **1 serialized payload**（例: 単一 JSON string key）として保存 |
| **C** | transaction 性を持つ **別 storage** へ移行 |

**今回:** A/B/C は **選ばない**。1 bool のみのため JSON / schema system の **先行導入禁止**。

### 12.4 plugin 制約（参考）

`shared_preferences` は multi-key **transaction を保証しない**。途中 failure 時は `save()` throw → Controller revert。過剰な transaction system は **導入しない**。

## 13. Application startup（将来 wiring）

```text
1. Composition Root:
     prefs = SharedPreferencesAsync()
     repository = SharedPreferencesSettingsRepository(preferences: prefs)
2. SettingsController(repository: repository)
3. runApp — settings = AppSettings.defaults()   // flicker 最小化
4. controller.initialize() — 非同期 load
5. load 完了 — revision guard 下で settings 更新
```

`initialize()` 完了前の mutation は Controller 側 **no-op**（client `4109a13` 実装済み — `isLoading` 中 `setShowTooltips` 無効）。

## 14. Cross-platform（MVP）

| Platform | MVP | 保存（plugin 抽象） | アプリコード |
|----------|-----|---------------------|--------------|
| **Windows** | 対象 | roaming AppData | path 分岐 **なし** |
| **Android** | 対象 | DataStore Preferences（Async default） | native API 分岐 **なし** |
| **iOS** | 対象 | NSUserDefaults | 分岐 **なし** |
| Web / macOS / Linux | 初期 MVP 外 | plugin 対応あり | 今回実装追加 **なし**。採用方式が将来致命障害にならないことを確認済み |

### 14.1 Backup / platform behavior

- Windows roaming AppData — OS / 企業 policy により roam される場合あり。`showTooltips` は **非秘密** のため許容
- iOS NSUserDefaults — app backup 対象になりうる。秘密情報ではないため許容
- Android — DataStore / SharedPreferences の通常 app-private storage

## 15. pubspec への影響（docs 記載のみ）

実装 Phase で client `pubspec.yaml` に追加予定:

```yaml
dependencies:
  shared_preferences: ^2.5.5
```

**今回 docs のみ — client pubspec は変更しない。** Flutter / Dart upgrade も提案しない。

## 16. Testing 設計

### 16.1 境界

| 種別 | 用途 |
|------|------|
| **`FakeSettingsRepository`**（既存） | `SettingsController` unit test — **継続使用** |
| **`SharedPreferencesSettingsRepository` test** | Concrete 実装の unit test |
| **integration / widget test** | 必要時のみ real storage（test binding + mock initial values） |

Controller test は **Fake** のまま。Repository test で storage 境界を検証する。

### 16.2 Repository unit tests（最低限）

| ケース |
|--------|
| no saved value → `AppSettings.defaults()` |
| saved `true` → restore |
| saved `false` → restore |
| save → load round trip |
| invalid / 型不一致 → field default `true`、load 成功 |
| invalid read → **repair write しない**（store 内容不変） |
| unknown extra keys → 既知 field 正常読込 |
| read failure（mock throw）→ throw |
| write failure → throw |
| `save(AppSettings(false))` → `settings.showTooltips == false`（1 field / 1 write） |

### 16.3 Version / platform compatibility

| ケース |
|--------|
| package 採用が nainai Android **minSdk 24** と矛盾しない（§1.2） |

### 16.4 Snapshot / atomicity 注記（設計テスト方針）

| 現在 | 将来 |
|------|------|
| 1 persisted field — `save` = 1 `setBool` | **2 field 目追加前** に atomicity 設計レビュー必須（§12.2） |

### 16.5 Controller integration（既存 + 追加）

client `4109a13` で **既に Implemented**:

- initialize / save / restart simulation（fake stored）
- save failure revert
- load failure default + loadFailed
- rapid changes / stale save

Concrete 追加後:

- Composition Root wiring smoke（optional integration test）
- `SharedPreferences.setMockInitialValues` 等を用いた end-to-end restore（optional）

## 17. 実装 Phase checklist

1. `pubspec.yaml` — `shared_preferences: ^2.5.5`
2. `SharedPreferencesSettingsRepository` 実装（constructor injection）
3. Composition Root — `SharedPreferencesAsync()` → Repository → `SettingsController`
4. `initialize()` 起動時 invoke
5. Repository unit tests（§16）
6. General Section / TooltipPolicy wiring（別 Phase — 本書範囲外）

## 18. 状態まとめ

| 項目 | 状態 |
|------|------|
| Common Settings Core | client `4109a13` **Implemented** |
| Common Settings Persistence design | **Design Complete** |
| Concrete Technology | **Selected** — `shared_preferences` + `SharedPreferencesAsync` |
| Concrete Repository implementation | **Not Implemented** |
| Planned dependency | `shared_preferences: ^2.5.5` |
| Planned class | `SharedPreferencesSettingsRepository` |
| Primary key | `settings.showTooltips` |
| Android minSdk compatibility | **24 ≥ 24 — OK**（§1.2） |
| AudioOutputPreference persistence | Phase G — **別設計**（本書対象外） |

## 19. 公式 Source（2026-08-30 確認）

| Source | 記録事項 |
|--------|----------|
| [shared_preferences — pub.dev](https://pub.dev/packages/shared_preferences) | version **2.5.5** / publisher flutter.dev / Flutter **>= 3.38** / Dart **^3.10** |
| pub.dev platform table | Android **SDK 24+** / iOS 13.0+ / Windows Any 等 |
| pub.dev README — API guidance | **`SharedPreferencesAsync` / `SharedPreferencesWithCache` 推奨**（Legacy deprecated 方向） |
| pub.dev README — critical data | write 後 disk persistence **保証なし** / critical data **不可** |
| pub.dev README — storage location | Windows AppData / Android DataStore default / iOS NSUserDefaults |
| pub.dev README — Android backend | `SharedPreferencesAsync` default = **DataStore Preferences** |
| nainai-client `android/app/build.gradle.kts` | `minSdk = flutter.minSdkVersion` → **24**（FlutterExtension.kt） |
