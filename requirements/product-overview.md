# プロダクト概要

## 目的

nainai は、ユーザーが端末内に保存されている音声・動画ファイルを選択し、アプリ内で再生・編集・管理するための Flutter アプリケーションです。

正式名称は **nainai** とする。「nainai Media Player」を製品正式名称として使用しない。将来の Player / Editor / Timeline / Playlist / Equalizer / Compressor 等への拡張を前提とし、アプリ全体を Media Player に限定しない。

各 OS 上で許可された保存先へファイルを書き出すことも将来の機能として想定しています（Web ブラウザの「ダウンロード」方式ではありません）。

## プラットフォームロードマップ

| フェーズ | プラットフォーム |
|----------|------------------|
| 初期（MVP） | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |

- Web / macOS / Linux は **初期 MVP の対象外** です。
- これらは恒久的な非対応ではなく、**将来対応予定** です。

関連 ADR: [ADR-0002 プラットフォームロードマップ](../adr/0002-platform-roadmap.md)

## クライアント技術

クライアントアプリケーションは **Flutter** を採用します。

| 項目 | 内容 |
|------|------|
| Flutter project name | `nainai` |
| フレームワーク | Flutter |
| 言語 | Dart（Flutter SDK に同梱されるものを使用） |
| 初期対象 | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |
| コードベース | 可能な限り共通の Flutter コードベースを使用 |
| プラットフォーム固有処理 | 必要に応じて各 OS 側へ分離 |

関連 ADR: [ADR-0001 Flutter 採用](../adr/0001-use-flutter.md)

## アプリ識別子

| 項目 | 値 |
|------|-----|
| Organization ID | `com.fyna` |
| Android Application ID | `com.fyna.nainai` |
| iOS Bundle ID | `com.fyna.nainai` |

Android と iOS で同じ識別子体系（`com.fyna.nainai`）を使用します。

関連 ADR: [ADR-0003 Application ID](../adr/0003-application-id.md)

## 扱うデータ

- 音声ファイル
- 動画ファイル

いずれも **端末内に保存されているファイル** を対象とします。ユーザーが OS 標準のファイル選択 UI を通じてファイルを選び、アプリ内で利用します。

## 基本の利用フロー（MVP）

```text
端末内ファイルを選択
       ↓
アプリ内で再生
```

## 将来の機能（方向性のみ）

以下は将来追加を想定している機能です。**詳細仕様は未確定** です。

| 機能 | 概要 |
|------|------|
| Editor / Timeline | 編集 Mode・再生位置や区間を視覚的に扱う |
| Playlist / Folder Playback | 複数メディアの管理・フォルダ再生 |
| Equalizer / Compressor | 音声処理 |
| Repeat ALL | 複数トラック等の繰り返し（Phase 2 は OFF / ONE のみ） |
| 区間指定 | ファイル内の特定区間を指定する |
| 分割 | ファイルを複数の区間・セグメントに分ける |
| 歌詞 | 分割データごとに歌詞を関連付ける |
| 字幕 | 分割データごとに字幕を関連付ける |
| 説明 | 分割データごとに説明テキストを関連付ける |
| メタデータ | ファイルや分割データに付随する情報を管理する |
| 出力 | 編集結果をファイルとして書き出す |
| 出力先設定 | ファイル種別ごとに出力先を指定する |
| テーマ選択 | Base Palette / Accent の独立選択（Architecture は先行定義） |
| UI 言語 | **Localization 基盤実装済み**（[localization.md](../design/localization.md)）。標準表示は **日本語**。`ja` / `en` ARB 対応。システム Locale 追従・言語切替 UI・Locale 永続保存は **未実装** |
| アプリ共通 Settings | **[Core Design Complete](../design/settings.md)**（Tooltip 表示 ON/OFF 等）。Persistence concrete・実装は **未着手** |
| 音声出力デバイス | Phase A/B Windows Service **実装済み**（client `c3239f3`）。Phase C Settings subsystem **[設計確定・実装未着手](../design/audio-output-settings.md)**（Launcher: **Launcher placement design pending**）。Settings からのユーザー選択・永続化・Android/iOS は **未実装** |
| Windows 再生 UI | 固定 Bottom Panel（Audio / Video 共通）**実装済み**（client `68ff1b4`、[phase2-ui.md](../design/phase2-ui.md) §4.1） |
| 問題報告 | Unknown Error からの報告（Phase 2 では未実装） |

## 設計方針

### 元ファイルを不用意に変更しない

- ユーザーが選択した **元ファイルは原則として上書き・破壊しない** 方針とします。
- 編集・出力の結果は、別ファイルとして保存する想定です（具体的な保存方式・命名規則は未確定）。

### ローカルファースト

- 初期 MVP では端末内のファイル操作とアプリ内再生に限定します。
- クラウド同期、ログイン、バックエンド通信は MVP の対象外です。

## UI / Design System（Phase 2）

Phase 2 の再生 UI 大枠と Design System は次を正とする。

- [phase2-ui.md](../design/phase2-ui.md)
- [design-system.md](../design/design-system.md)
- [media-playback.md](../design/media-playback.md)

Design Persona は **Professional Creative Media Tool**。Theme は Base / Accent / Semantic を分離する。

## 未確定事項

以下は現時点で確定していません。

- 対応する音声・動画フォーマットの一覧
- 出力先選択ライブラリ / API・ファイル書き出し方式
- 分割・歌詞・字幕・説明のデータモデル
- 字幕形式
- 出力ファイルの形式と保存先の指定方法
- タイムラインの実装方式 / 分割処理技術 / FFmpeg 採用有無
- 状態管理ライブラリ / ルーティングライブラリ
- backend 技術 / API / DB / 認証 / クラウド同期
- 各プラットフォーム間でのデータ共有方法
- フォントの Flutter 正式 Font Asset 導入（Phase 2 は fallback。Token は [design-system.md](../design/design-system.md)）

入力ファイル選択・再生の技術選定は [media-technology.md](../architecture/media-technology.md) を参照。
