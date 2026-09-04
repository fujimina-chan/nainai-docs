# UI Localization（機能設計）

nainai-client に導入済みの UI 文字列 Localization 基盤を定義する。

文言の正本は ARB ファイルとする。UI レイアウトは [phase2-ui.md](phase2-ui.md)、エラー区分は [media-playback.md](media-playback.md) を参照。

関連:

- [Phase 2 UI](phase2-ui.md)
- [メディア再生](media-playback.md)
- [メディアファイル選択](media-selection.md)
- [音声出力デバイス選択](audio-output.md)

## 1. 採用技術

| 項目 | 内容 |
|------|------|
| 方式 | Flutter 標準 `flutter_localizations` + **gen-l10n** + **ARB** |
| 対応 Locale | `ja` / `en` |
| ARB 正本 | `lib/l10n/app_ja.arb` / `lib/l10n/app_en.arb` |
| 生成物 | ARB から生成。生成ファイルを翻訳内容の正本に **しない** |

`pubspec.yaml` で `flutter: generate: true` を有効化している。

## 2. 現在の表示言語

| 項目 | 状態 |
|------|------|
| 標準表示 | **日本語**（`Locale('ja')` を固定） |
| システム Locale 追従 | **未実装** |
| 言語切替 UI | **未実装** |
| Locale 永続保存 | **未実装** |
| LocaleController | **未実装** |
| `SettingsController`（`AppSettings` / Tooltip 等） | **[設計確定・未実装](settings.md)** — Locale 用 Controller とは別物 |

## 3. 実装済み UI 文言（日本語）

Phase 2 Media 画面および Error UI で、少なくとも以下が Localization 経由で表示される。

| ARB キー | 日本語 |
|----------|--------|
| `selectFile` | ファイルを選択 |
| `selectAnotherFile` | 別のファイルを選択 |
| `loading` | 読み込み中… |
| `audio` | 音声 |
| `play` | 再生 |
| `pause` | 一時停止 |
| `stop` | 停止 |
| `repeatOne` | **この曲を繰り返し再生** |
| `mute` | ミュート |
| `unmute` | ミュート解除 |
| `media` | メディア |
| `collapsePlaybackControls` | しまう（Bottom Panel collapse Tooltip / Semantics） |
| `expandPlaybackControls` | 戻す（Bottom Panel expand Tooltip / Semantics） |

Error UI も日本語化済み（`selectionFailed` / `unsupportedMedia` / `loadFailed` 等。一覧は ARB 正本を参照）。

### Repeat ONE の正式日本語

Repeat ONE の Tooltip / Semantics 等に用いる日本語は **「この曲を繰り返し再生」** とする。

「1件リピート」等は **正式仕様ではない**。

## 4. OS File Picker ラベル

OS File Picker の Media フィルター表示名（`XTypeGroup.label`）も Localization 対象とする。

| Locale | ラベル |
|--------|--------|
| ja | メディア |
| en | Media |

Service 層へ `BuildContext` は注入しない。`MediaTypeGroupLabelProvider`（label provider）によって UI 層から localized 文字列を渡す設計を採用済み（[media-selection.md](media-selection.md) §4.1）。

## 5. Settings 向けキー

共通 Settings（Display / Tooltip / Audio 等）のキー設計は [settings.md](settings.md) §8。音声出力 Settings 向けキーは [audio-output-settings.md](audio-output-settings.md) §16 および [audio-output.md](audio-output.md) §11。いずれも **未実装**。Settings UI 実装 Phase で ARB へ追加する。

## 6. 検証（client）

Localization 統合後、nainai-client で次を確認済み（2026-08-30 時点）:

| コマンド | 結果 |
|----------|------|
| `flutter analyze` | No issues found |
| `flutter test` | 145 tests PASS |

client commit: `7e7c245` — `feat: UIのLocalization基盤と日本語表示を追加`

## 7. 未実装・未確定

- 日本語 / English 切替 UI
- システム言語に合わせる設定
- Locale 永続保存
- LocaleController
- `SettingsController`（[settings.md](settings.md) — Locale Controller とは別）
- Tooltip 表示 ON/OFF — [settings.md](settings.md) **Core Design Complete** / **未実装**（現在 client は Tooltip 常時有効）
