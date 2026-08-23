# ADR-0003: Application ID

## ステータス

承認済（Accepted）

## コンテキスト

Android および iOS への配布・ビルドのため、Organization ID とアプリケーション識別子を正式に決定する必要があった。

## 検討した選択肢

- Android と iOS で異なる識別子体系を用いる
- Android と iOS で同一の識別子体系を用いる

## 決定

以下の識別子を正式採用する。

| 項目 | 値 |
|------|-----|
| Organization ID | `com.fyna` |
| Android Application ID | `com.fyna.nainai` |
| iOS Bundle ID | `com.fyna.nainai` |

Android と iOS で同じ識別子体系（`com.fyna.nainai`）を使用する。

Flutter project name は `nainai` とする（[ADR-0001](0001-use-flutter.md)）。

## 結果

- Android / iOS で一貫したアプリ識別が可能になる
- ストア登録・ビルド設定・証明書管理の基準が明確になる
- Web / macOS / Linux 向けの識別子詳細は、将来対応時に必要に応じて別途決定する
