# Study-OID4VCI-VP

OID4VCI / OID4VP と HAIP(OpenID4VC High Assurance Interoperability Profile)の学習・整理用ドキュメント集です。

## 背景と目的

OID4VCI / OID4VP 準拠の VC 発行・VP 検証基盤を、新しく確定した規格(特に HAIP 1.0)へ対応させるにあたり、

1. チームが DID / VC の基礎と仕様を正しく理解できること
2. 対外的に技術の価値を説明できること
3. HAIP への具体的な移行方針を持てること

を目的にドキュメントを整備しています。

## ドキュメント一覧

| # | ドキュメント | 対象読者 | 内容 |
|---|---|---|---|
| ① | [OID4VCI / OID4VP 基礎理解ドキュメント](./docs/01_oid4vci-vp-basics.md) | 初心者〜中級のエンジニア | 3 者モデル、VC/VP/DID の基礎、SD-JWT の選択的開示、OID4VCI 発行フロー・OID4VP 提示フローで**実際に流れるデータの具体例**、失効管理、用語集 |
| ② | [OID4VC / OID4VP 対外説明資料](./docs/02_oid4vc-external-explainer.md) | 社外・非エンジニア | スライド構成の説明資料。課題と価値、ユースケース、標準化の潮流(EUDI Wallet 等)、セキュリティ FAQ、想定問答 |
| ③ | [HAIP 対応ガイド](./docs/03_haip-migration-guide.md) | 既存基盤の開発者 | HAIP 1.0 Final の位置づけ、ドラフト実装からの**変更点ギャップ一覧**、MUST 要件チェックリスト、実装方法、移行ロードマップ |

## 読む順番の目安

- はじめての方: ① → ②
- 既存基盤の HAIP 対応を担当する方: ①(復習)→ ③

## 仕様バージョンの前提

| 仕様 | バージョン | 状態 |
|---|---|---|
| OpenID for Verifiable Presentations (OID4VP) | 1.0 | Final(2025-07) |
| OpenID for Verifiable Credential Issuance (OID4VCI) | 1.0 | Final(2025-09) |
| OpenID4VC High Assurance Interoperability Profile (HAIP) | 1.0 | Final(2025-12) |

> ⚠️ 本リポジトリのドキュメントは公開情報に基づく整理であり、実装判断の際は各仕様の Final 本文を一次情報として必ず確認してください。
