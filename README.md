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
| ④ | [かみしばい「はいぷ先生の まほうのルールブック」](./docs/04_haip-kamishibai.html) | 新入社員・研修 | 子ども向けアニメ風の紙芝居(HTML アニメーション)。第 1 話は ③ の HAIP(12 ページ)、第 2 話は ⑤ の AI エージェントと UCP(8 ページ)。各ページに技術用語への「おとなのことば」対応メモつき。ブラウザで開くだけで動作 |
| ⑤ | [AI エージェントと UCP 基礎理解ドキュメント](./docs/05_ai-agent-ucp-basics.md) | 新入社員・初心者 | AI エージェントの基礎、周辺プロトコル(MCP / A2A / AP2)、UCP(Universal Commerce Protocol)の概要と仕組み、DID / VC 基盤との接点 |
| ⑥ | [仕様相談・開発相談エージェントの作り方(Claude Projects 版)](./docs/06_consultation-agent-setup.md) | ドキュメント運用担当 | ①〜⑤ をナレッジとした社内相談エージェントのセットアップ手順、そのまま使える Custom Instructions、運用・メンテナンス方法 |
| ⑦ | [HAIP 差分マトリクス(Excel)](./docs/07_haip-diff-matrix.xlsx) | 実装者 | HAIP と素の OID4VCI / OID4VP の MUST・任意「差分」比較(発行・検証・フォーマット、27項目)。サンプル・参照リンク付き |
| ⑧ | [HAIP 要件カタログ(Excel)](./docs/08_haip-requirements-catalog.xlsx) | 実装者 | HAIP の全要件を MUST/SHOULD/MAY/MUST NOT で分類(エコシステム・Issuerメタデータ・暗号・Attestation 中心)+ OID4VCI メタデータ全パラメータ。サンプル・参照リンク付き |
| ⑨ | [HAIP 仕様書 読み合わせコンパニオン(Excel)](./docs/09_haip-spec-companion.xlsx) | 仕様書を精読する人 | **仕様書の見出し順に全項目(PAR/PKCE/DPoP 等の基本フロー含む)を1シートに統合**。仕様書ページと並べてスクロールしながら読む用。HAIP / OID4VCI(素) / OID4VP(素)の3仕様レベル比較付き |
| ⑩ | [かみしばい 第3話「ひみつを まもれる おつかいさん」](./docs/10_confidential-client-fapi2-kamishibai.html) | 新入社員・研修 | **Confidential Client と FAPI2 Security Profile** を子ども向けアニメ風の紙芝居(全11ページ)で解説。金庫・ぎんこう・きっぷ等の比喩で、PAR/PKCE/iss/DPoP まで。各ページに「おとなのことば」対応メモつき |
| ⑪ | [かみしばい 第4話「FAPI2 の証明の流れ」](./docs/11_fapi2-flow-kamishibai.html) | 実装者・研修 | **PAR → PKCE → iss → DPoP の認可フローを技術的に正確に**追う紙芝居(全14ページ)。各ステップに実際に送受信する HTTP メッセージ(パラメータ)と受け手の検証チェックリストを掲載。サンプル値は RFC 7636 / RFC 9449 の例に準拠 |

## 読む順番の目安

- はじめての方: ① → ②
- 既存基盤の HAIP 対応を担当する方: ①(復習)→ ③
- 社内相談エージェントを作りたい方: ①〜⑤ を一通り把握 → ⑥
- 仕様書を精読する方: ⑨ を仕様書ページと並べて読む(全項目網羅)。差分だけ知りたいときは ⑦、レベル別カタログは ⑧

## 仕様バージョンの前提

| 仕様 | バージョン | 状態 |
|---|---|---|
| OpenID for Verifiable Presentations (OID4VP) | 1.0 | Final(2025-07) |
| OpenID for Verifiable Credential Issuance (OID4VCI) | 1.0 | Final(2025-09) |
| OpenID4VC High Assurance Interoperability Profile (HAIP) | 1.0 | Final(2025-12) |

> ⚠️ 本リポジトリのドキュメントは公開情報に基づく整理であり、実装判断の際は各仕様の Final 本文を一次情報として必ず確認してください。
