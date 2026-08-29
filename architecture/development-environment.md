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
E:\fyna\dev\nainai\
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

## Phase 2-5 Windows 実機検証（進捗）

Phase 2-5 全体の完了を意味しない。以下は Windows 実機で実装・確認済みの項目のみ。

| 項目 | Status | 正本 |
|------|--------|------|
| Volume 整数 % 表示 | 完了 | [phase2-ui.md](../design/phase2-ui.md) §12 / [media-playback.md](../design/media-playback.md) §9 |
| Audio / Video 単一 Media フィルター | 完了 | [media-selection.md](../design/media-selection.md) §4.1 |
| Volume アイコン Mute / Unmute | 完了 | [media-playback.md](../design/media-playback.md) §9 |
| Volume 低域特性の原因調査 | 完了 | [media-playback.md](../design/media-playback.md) §9 / [media-technology.md](media-technology.md) |
| mpv Volume 逆 3 乗補正（`MediaKitVolumeMapper`） | 完了 | [media-technology.md](media-technology.md) |
| Volume 10% 付近でほぼ無音 | **Windows: Resolved / 実機確認済み**（Android / iOS は補正実装共通・実機確認未実施） | [media-playback.md](../design/media-playback.md) §9 |
| Seek Slider AXTree 更新エラー | **Resolved** | 下記 Known Issues |
| Windows 実機確認（上記含む再生操作） | 完了 | 各正本 |

未確認・未完了の Phase 2-5 項目がある場合は、本表に載せない限り完了扱いにしない。

## Known Issues（開発環境）

Product Requirement ではない。ローカル開発・ビルド環境上の制約として記録する。未解決の項目については恒久解決策は未決定。

### Windows: exFAT と plugin symlink（Resolved）

**Status:** Resolved

#### 問題

旧作業場所 `D:\fyna\dev\nainai\` の `D:` が exFAT だったため、Flutter Windows plugin が必要な build で plugin symlink 作成時に `ERROR_INVALID_FUNCTION` となることを確認した。

- Phase 2 実装コードの compile error ではない
- filesystem 環境制約である

#### 対策

正式作業場所を NTFS の `E:\fyna\dev\nainai\` へ移行した。

E: 上で次が成功済み:

- `flutter pub get`
- `flutter analyze`
- `flutter test`
- `flutter build windows`

#### 補足

- 旧 `D:` を移行検証完了まで削除しない運用は、本問題の解決とは別事項
- これを解消するために **C: へ作業コピーする運用を正式手順として採用しない**（調査時の方針。現行標準は E: NTFS）

### Android: 依存取得時の SSL / PKIX

現在の Windows 開発環境では、`media_kit_libs_android_video` 等の取得時に SSL context / PKIX 関連の問題が発生しうる。

- Phase 2-1 では一時的な truststore 変更で build 成功した例があるが、ユーザー共通 Gradle 設定は作業終了後に復旧済み
- Phase 2-2 では共通設定を変更せず、環境問題として扱った
- Product Requirement ではない
- 恒久解決策は未決定

個人のセキュリティ製品名・ユーザー名など、不要な個人環境情報は本正本に残さない。

### Windows: Seek Slider の AXTree 更新エラー（Resolved）

**Status:** Resolved（Phase 2-5 Windows 実機検証で修正・確認済み）

#### 問題

Windows 実機で Seek Slider 表示後、次のエラーが繰り返し発生した。

```text
Failed to update ui::AXTree,
error: Nodes left pending by the update: N
```

アプリ Crash ではないが、Accessibility tree 更新エラーが継続した。

#### 確認環境

| 項目 | 値 |
|------|-----|
| OS | Windows 10 Pro 10.0.19045 |
| Framework | Flutter 3.47.0 stable |
| Runtime / SDK | Dart 3.13.0 |

#### nainai での原因

Seek Slider が duration 未取得時は `onChanged = null`（disabled）だった。duration 取得後に `onChanged = callback` となり enabled へ遷移していた。

最小 Flutter Project でも disabled → enabled だけで同じ AXTree エラーを再現した。

#### nainai での対策

Seek Slider を Widget lifetime 中、常に enabled 状態に維持する。

duration 未取得中も `onChanged` は non-null とし、callback 内で `hasDuration == false` なら return する。これにより disabled → enabled 遷移を発生させない。

Slider の `max` を `1` → `duration` へ変更する挙動は維持する。最小再現 Test では、enabled 固定状態での max 変更だけでは AXTree エラーは再現しなかった。

#### 実機確認（修正後）

- Audio 選択直後: AXTree エラーなし
- Seek hover: AXTree エラーなし
- Tooltip 表示: AXTree エラーなし
- Playing 中: AXTree エラーなし
- Seek 正常
- Play / Pause / Stop 正常
- Volume 正常
- Repeat 操作正常

#### 補足

確認環境では Slider の disabled → enabled 遷移が直接再現トリガーだった。Flutter Engine 内部の upstream root cause は未特定である。Flutter 全バージョン・Windows 全環境で必ず発生するものではない。

#### 詳細・汎用手順

詳細な最小再現、Test Matrix、汎用 Workaround、Accessibility 上の注意、Flutter 更新時の再確認手順は [engineering-knowledge](https://github.com/fujimina-chan/engineering-knowledge/blob/main/troubleshooting/flutter/windows/slider-axtree-disabled-to-enabled.md) を参照。

## 関連文書

- [repository-structure.md](repository-structure.md) … リポジトリ責務と開発運用
- [system-overview.md](system-overview.md) … システム構成
- [media-technology.md](media-technology.md) … メディア技術選定・Package 挙動
- [ADR-0001 Flutter 採用](../adr/0001-use-flutter.md)
- [ADR-0002 プラットフォームロードマップ](../adr/0002-platform-roadmap.md)
