# 開発環境

nainai のローカル開発環境に関するプロジェクトルールを定めます。

本文書は恒久的なアーキテクチャ意思決定（ADR）ではなく、**実機検証に基づく開発運用ルール**です。

## 対象プラットフォーム

| フェーズ | プラットフォーム |
|----------|------------------|
| 初期（MVP） | Windows / Android / iOS |
| 将来対応予定 | Web / macOS / Linux |

詳細は [ADR-0002 プラットフォームロードマップ](../adr/0002-platform-roadmap.md) を参照してください。

## Windows リポジトリ配置

nainai の Windows 上での Flutter 開発では、**リポジトリパスを ASCII 文字のみで構成する**。

### 現在の標準配置

```text
D:\fyna\dev\nainai\
├── nainai-infra
├── nainai-backend
├── nainai-client
└── nainai-docs
```

実 Git リポジトリそのものを、上記のような ASCII 実パスへ配置する。

### 理由

旧配置（日本語等の非 ASCII 文字を含むパス）では、実際の開発環境で次の問題を確認した。

- `flutter analyze` における Analysis Server の異常
- Flutter Windows ビルド / MSBuild におけるパス問題

そのため、nainai プロジェクト固有の安全な開発ルールとして、Windows 上のリポジトリパスを ASCII のみとする。

これは「Windows では日本語パスが使えない」という一般論ではない。nainai の Flutter 開発における互換性リスクを避けるためのプロジェクトルールである。

### ジャンクション

ディレクトリジャンクションは、問題調査時の一時的な検証方法として用いたことがある。

**正式な開発手順には採用しない。**

ジャンクション経由ではなく、実 Git リポジトリを ASCII パスへ直接配置する。

### パス変更時の生成キャッシュ

リポジトリパスを変更した場合、Flutter / CMake 等が旧絶対パスを含む生成キャッシュを保持していることがある。

旧パス由来のキャッシュが残った状態では、Windows ビルドが失敗する場合がある。その場合は次を実施する。

1. `flutter clean`
2. 依存関係の再取得・再生成
3. 再ビルド

**毎回必ず `flutter clean` するルールにはしない。** パス変更後など、旧絶対パスが残っている疑いがあるときに実施する。

なお、旧パス対策として使用していた `android.overridePathCheck=true` は削除済みである。ASCII 実パス配置では不要であり、削除後も Android APK ビルドは成功している。

## 検証済み項目（Windows）

Flutter **3.47.0** / Windows 環境において、標準配置（ASCII 実パス）から次が成功済みである。

| コマンド | 結果 |
|----------|------|
| `flutter pub get` | 成功 |
| `flutter analyze` | 成功 |
| `flutter test` | 成功 |
| `flutter build windows` | 成功（必要に応じて `flutter clean` 後） |
| `flutter build apk --debug` | 成功（`android.overridePathCheck=true` 削除後も成功） |
| `flutter run -d windows` | 成功（Windows GUI 正常起動、即時クラッシュなし） |

ジャンクションは使用していない。

## Known Issues（開発環境）

Product Requirement ではない。ローカル開発・ビルド環境上の制約として記録する。恒久解決策は未決定。

### Windows: exFAT と plugin symlink

現在の主要開発配置は `D:\fyna\dev\nainai\` である。

当該 `D:` が exFAT の場合、Flutter plugin が必要な Windows build で plugin symlink 作成時に `ERROR_INVALID_FUNCTION` となることを確認済み。

- Phase 2 実装コードの compile error ではない
- filesystem 環境制約である

**重要:**

- これを解消するために **C: へ作業コピーする運用を正式手順として採用しない**
- nainai の作業場所は **D: を維持**する
- Windows build 環境の正式解決策は別途決定する

### Android: 依存取得時の SSL / PKIX

現在の Windows 開発環境では、`media_kit_libs_android_video` 等の取得時に SSL context / PKIX 関連の問題が発生しうる。

- Phase 2-1 では一時的な truststore 変更で build 成功した例があるが、ユーザー共通 Gradle 設定は作業終了後に復旧済み
- Phase 2-2 では共通設定を変更せず、環境問題として扱った
- Product Requirement ではない
- 恒久解決策は未決定

個人のセキュリティ製品名・ユーザー名など、不要な個人環境情報は本正本に残さない。

## 関連文書

- [repository-structure.md](repository-structure.md) … リポジトリ責務と開発運用
- [system-overview.md](system-overview.md) … システム構成
- [media-technology.md](media-technology.md) … メディア技術選定・Package 挙動
- [ADR-0001 Flutter 採用](../adr/0001-use-flutter.md)
- [ADR-0002 プラットフォームロードマップ](../adr/0002-platform-roadmap.md)
