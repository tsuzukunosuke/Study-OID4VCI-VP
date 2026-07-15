# ① OID4VCI / OID4VP 基礎理解ドキュメント

> **対象読者**: デジタルアイデンティティ技術の初心者〜中級者。OAuth 2.0 / OpenID Connect の基本用語(アクセストークン、認可コードなど)を聞いたことがあるレベルを想定しています。
>
> **このドキュメントのゴール**: VC(Verifiable Credential)の発行から VP(Verifiable Presentation)の検証まで、**「誰が・誰に・どんなデータを・どの順番で」渡すのか**を具体的なデータ例つきで理解できるようになること。

---

## 目次

1. [まず全体像:3 者モデルとトラストトライアングル](#1-まず全体像3-者モデルとトラストトライアングル)
2. [VC / VP / DID とは何か](#2-vc--vp--did-とは何か)
3. [クレデンシャルフォーマット(SD-JWT VC / mdoc)](#3-クレデンシャルフォーマットsd-jwt-vc--mdoc)
4. [選択的開示(Selective Disclosure)の仕組み](#4-選択的開示selective-disclosureの仕組み)
5. [OID4VCI:VC 発行プロトコル](#5-oid4vcivc-発行プロトコル)
6. [OID4VP:VP 提示・検証プロトコル](#6-oid4vpvp-提示検証プロトコル)
7. [クレデンシャルの失効・ステータス管理](#7-クレデンシャルの失効ステータス管理)
8. [用語集](#8-用語集)
9. [参考資料](#9-参考資料)

---

## 1. まず全体像:3 者モデルとトラストトライアングル

従来の ID 連携(OpenID Connect など)では、検証者(サービス)がユーザーの属性を知りたいとき、**毎回 IdP(発行者)に問い合わせ**ていました。VC のモデルでは、属性情報を**一度クレデンシャルとしてユーザーの手元(ウォレット)に発行**し、以降はユーザーが自分の意思で提示します。

```mermaid
flowchart LR
    Issuer["Issuer(発行者)<br/>例: 企業の人事部、大学、行政"]
    Holder["Holder(保持者)<br/>ユーザー本人+ウォレットアプリ"]
    Verifier["Verifier(検証者)<br/>例: 入館システム、金融機関、EC サイト"]

    Issuer -- "① VC を発行<br/>(OID4VCI)" --> Holder
    Holder -- "② VP を提示<br/>(OID4VP)" --> Verifier
    Verifier -. "③ 信頼(署名検証)<br/>直接通信は不要" .-> Issuer
```

| 登場人物 | 役割 | 具体例 |
|---|---|---|
| **Issuer(発行者)** | 属性を証明するクレデンシャル(VC)に署名して発行する | 大学(卒業証明)、企業(社員証)、行政(住民票相当) |
| **Holder(保持者)** | VC をウォレットに保管し、必要なときに VP として提示する | スマホのウォレットアプリを持つユーザー |
| **Verifier(検証者)** | 提示された VP の署名・有効性を検証してサービスを提供する | 酒類販売(年齢確認)、入館ゲート、口座開設 |

このモデルの最大のポイントは **③ の矢印が「点線」であること**です。Verifier は Issuer とリアルタイム通信をしません。VC に含まれる **Issuer の電子署名**を検証することで「この属性は確かに Issuer が発行したものだ」と確認できます。これにより:

- **プライバシー**: Issuer は「ユーザーがいつ・どこで VC を使ったか」を知り得ない(Issuer によるトラッキングの防止)
- **可用性**: Issuer のサーバーが落ちていても検証できる
- **選択的開示**: ユーザーは必要な属性だけを見せられる(例:生年月日を見せずに「20 歳以上」だけ)

そして、この 3 者間のやり取りを標準化したプロトコルが本ドキュメントの主役です。

| プロトコル | 何を標準化するか | 現行バージョン |
|---|---|---|
| **OID4VCI** (OpenID for Verifiable Credential Issuance) | Issuer → Holder への **VC 発行** | 1.0 Final(2025 年 9 月承認) |
| **OID4VP** (OpenID for Verifiable Presentations) | Holder → Verifier への **VP 提示** | 1.0 Final(2025 年 7 月承認) |
| **HAIP** (OpenID4VC High Assurance Interoperability Profile) | 上記 2 つ+フォーマットの**高保証プロファイル**(選択肢を絞り相互運用性を担保) | 1.0 Final(2025 年 12 月承認)→ [③ HAIP ドキュメント](./03_haip-migration-guide.md)参照 |

> 💡 **なぜ OAuth ベースなのか**: OID4VCI / OID4VP は、実績のある **OAuth 2.0 / OpenID Connect の仕組み(トークンエンドポイント、認可リクエスト等)を土台**に設計されています。既存の認可サーバー実装・ライブラリ・運用ノウハウを流用でき、特定のクレデンシャル形式や鍵解決方式(DID 等)に依存しない点が、他の VC 交換プロトコルと比べた強みです。

---

## 2. VC / VP / DID とは何か

### 2.1 VC(Verifiable Credential:検証可能クレデンシャル)

**「発行者のデジタル署名つき属性証明書」**です。紙の卒業証明書や免許証のデジタル版と考えてください。中身は大きく 3 つに分かれます。

```
┌─────────────────────────────────────┐
│ VC(Verifiable Credential)          │
│ ┌─────────────────────────────────┐ │
│ │ メタデータ                        │ │  ← 発行者 (iss)、発行日 (iat)、
│ │                                 │ │     有効期限 (exp)、種別 (vct)
│ ├─────────────────────────────────┤ │
│ │ クレーム(属性)                   │ │  ← 氏名、生年月日、社員番号 など
│ ├─────────────────────────────────┤ │
│ │ 発行者の署名 + Holder の公開鍵     │ │  ← 改ざん検知と「持ち主の証明」に使う
│ └─────────────────────────────────┘ │
└─────────────────────────────────────┘
```

重要なのは、VC には**Holder(持ち主)の公開鍵が埋め込まれる**ことです(`cnf` クレーム)。これにより「この VC は、対応する秘密鍵を持つ本人しか使えない」状態になります(**Key Binding / Holder Binding** と呼びます)。盗まれた VC を他人が使うことを防ぐ仕組みです。

### 2.2 VP(Verifiable Presentation:検証可能プレゼンテーション)

VC をそのまま Verifier に渡すのではなく、**「今、この検証者に対して、本人が提示している」ことの証明**を付けて渡すパッケージが VP です。SD-JWT VC の場合、具体的には:

- 開示する属性(ディスクロージャ)を選んだ VC 本体
- **KB-JWT(Key Binding JWT)**: Holder が自分の秘密鍵で「宛先の Verifier(`aud`)」「検証者が発行したチャレンジ(`nonce`)」に署名したもの

これにより Verifier は「VC が本物」かつ「提示しているのが正当な持ち主」かつ「このセッションのための提示(リプレイでない)」を同時に検証できます。

### 2.3 DID(Decentralized Identifier:分散型識別子)

`did:web:example.com` や `did:key:z6Mk...` のような形式の、**特定の中央機関に依存しない識別子**です。DID を解決(resolve)すると DID ドキュメントが得られ、そこに公開鍵が載っています。

> ⚠️ **よくある誤解**: 「OID4VCI/OID4VP を使うには DID やブロックチェーンが必須」ではありません。両仕様は**鍵解決方式に中立**で、Issuer の鍵は「HTTPS の well-known JWKS」「X.509 証明書チェーン(`x5c`)」「DID」のいずれでも運用できます。実際、後述の HAIP は **X.509 ベース**を採用しており、DID は必須ではありません。

---

## 3. クレデンシャルフォーマット(SD-JWT VC / mdoc)

OID4VCI/OID4VP は「入れ物(プロトコル)」であり、「中身(クレデンシャルの形式)」は選択できます。実務で主流の 2 つを押さえてください。

| | **IETF SD-JWT VC** | **ISO mdoc(ISO/IEC 18013-5)** |
|---|---|---|
| フォーマット識別子(OID4VCI/VP 上) | `dc+sd-jwt` | `mso_mdoc` |
| ベース技術 | JWT + 選択的開示拡張(SD-JWT) | CBOR / COSE(バイナリ) |
| 出自 | IETF OAuth WG | ISO(モバイル運転免許証 mDL の規格) |
| 選択的開示 | ディスクロージャ(ハッシュ方式) | ネームスペース単位の要素開示(同じくハッシュ方式) |
| 主な用途 | Web 系サービス全般、EUDI Wallet の PID 等 | 運転免許証、対面提示(NFC/BLE)にも対応 |

> 📝 W3C VC Data Model(JSON-LD ベース)のフォーマットもありますが、EUDI Wallet や HAIP の潮流では **SD-JWT VC と mdoc の 2 本柱**が事実上の主流です。本ドキュメントでは以降 SD-JWT VC を軸に説明します。
>
> ⚠️ フォーマット識別子は旧ドラフトの `vc+sd-jwt` から **`dc+sd-jwt` に変更**されました。旧識別子で実装している場合は要修正です(詳細は [③ HAIP ドキュメント](./03_haip-migration-guide.md))。

---

## 4. 選択的開示(Selective Disclosure)の仕組み

SD-JWT の核心は「**署名を壊さずに、属性を隠したまま渡せる**」ことです。仕組みは次の通りです。

### 4.1 発行時:属性を「ハッシュ」に置き換える

Issuer は、隠せるようにしたい属性を平文で JWT に入れる代わりに、**ソルトつきハッシュ値**だけを `_sd` 配列に入れます。

**SD-JWT のペイロード(Issuer が署名する部分)**:

```json
{
  "iss": "https://issuer.example.com",
  "iat": 1767225600,
  "exp": 1798761600,
  "vct": "https://credentials.example.com/employee_credential",
  "_sd_alg": "sha-256",
  "_sd": [
    "X9yH0Ajrdm1Oij4tWso9UzzKJvPoDxwmuEcO3XAdRC0",
    "s0BKYsLWxQQeU8tVlltM7MKsIRTrEIa1PkJmqxBBf5U"
  ],
  "cnf": {
    "jwk": { "kty": "EC", "crv": "P-256", "x": "TCAER19Zvu...", "y": "ZxjiWWbZMQ..." }
  }
}
```

- `vct`: クレデンシャルの種別(Verifiable Credential Type)
- `_sd`: 隠された属性のハッシュ値リスト
- `cnf`: Holder の公開鍵(Key Binding 用)

### 4.2 属性の実体は「ディスクロージャ」として別添え

各属性は `[ソルト, 属性名, 値]` の JSON 配列を Base64url 化した**ディスクロージャ**として、JWT の外に付属します。

```
ディスクロージャ(デコード後): ["_26bc4LT-ac6q2KI6cBW5es", "family_name", "山田"]
ディスクロージャ(実際の形):  WyJfMjZiYzRMVC1hYzZxMktJNmNCVzVlcyIsImZhbWlseV9uYW1lIiwi5bGx55SwIl0
```

このディスクロージャを SHA-256 でハッシュすると、`_sd` 配列内の値と一致します。**渡されなかった属性は、ハッシュ値から復元できない**(ソルトがあるため辞書攻撃も不可)というわけです。

### 4.3 全体の形式:`~`(チルダ)区切り

```
<Issuer 署名つき JWT>~<ディスクロージャ1>~<ディスクロージャ2>~...~<KB-JWT>
```

- **発行時**(Issuer → Holder): 全ディスクロージャつきで渡される(KB-JWT なし)
- **提示時**(Holder → Verifier): Holder が**見せたいディスクロージャだけ**を残し、末尾に KB-JWT を付ける

### 4.4 提示時:KB-JWT(Key Binding JWT)

```json
// KB-JWT ヘッダ
{ "typ": "kb+jwt", "alg": "ES256" }
// KB-JWT ペイロード
{
  "nonce": "n-0S6_WzA2Mj",
  "aud": "x509_san_dns:verifier.example.com",
  "iat": 1767312000,
  "sd_hash": "Dy-RYwZfaaoC3inJbLslgPvMp09bH-clYP_3qbRqtW4"
}
```

- `nonce`: Verifier が発行したチャレンジ(リプレイ防止)
- `aud`: 提示先の Verifier(横流し防止)
- `sd_hash`: 「JWT+選択したディスクロージャ」全体のハッシュ(すり替え防止)

Holder は `cnf` に対応する**秘密鍵**でこれに署名します。Verifier は VC 内の `cnf` 公開鍵で KB-JWT を検証し、「正当な持ち主による、このセッション向けの提示」であることを確認します。

---

## 5. OID4VCI:VC 発行プロトコル

### 5.1 全体像

OID4VCI は **「OAuth 2.0 で保護された Credential API」** です。ウォレットが OAuth クライアント、Issuer が「認可サーバー+リソースサーバー(Credential Endpoint)」に相当します。

主要な構成要素:

| 要素 | 役割 |
|---|---|
| **Credential Offer** | Issuer からウォレットへの「発行のお誘い」(QR コード等) |
| **Issuer メタデータ** | `/.well-known/openid-credential-issuer` で公開。発行可能なクレデンシャルのカタログ |
| **Authorization / Token Endpoint** | 通常の OAuth 2.0。アクセストークンを発行 |
| **Nonce Endpoint** | proof 用の使い捨て nonce(`c_nonce`)を払い出す |
| **Credential Endpoint** | アクセストークンと引き換えに VC を発行する本丸 |
| **Deferred Credential Endpoint** | 即時発行できない場合(人手の審査等)に後で取りに来るための口 |
| **Notification Endpoint** | ウォレットから「保存に成功した/ユーザーが削除した」等の通知を受ける |

### 5.2 2 つの発行フロー

| | **Authorization Code Flow** | **Pre-Authorized Code Flow** |
|---|---|---|
| 起点 | ウォレット主導でも Issuer 主導でも可 | Issuer 主導(事前に本人確認済みが前提) |
| ユーザー認証 | 発行の中で Issuer が認証(ログイン画面等) | **事前に別チャネルで完了済み**(窓口、ログイン済みポータル等) |
| 典型例 | ウォレットから「社員証がほしい」と申請 | 人事ポータルにログイン済みのユーザーに QR を表示 |
| 追加の保護 | PKCE | `tx_code`(SMS 等で別送する短い PIN) |

### 5.3 シーケンス(Pre-Authorized Code Flow の例)

```mermaid
sequenceDiagram
    autonumber
    participant U as ユーザー
    participant W as ウォレット (Holder)
    participant I as Issuer(認可サーバー + Credential Endpoint)

    Note over U,I: 【事前】ユーザーは Issuer のポータルにログイン済み(本人確認済み)
    I->>U: Credential Offer を QR コード等で提示
    U->>W: QR を読み取り
    W->>I: Issuer メタデータ取得<br/>GET /.well-known/openid-credential-issuer
    I-->>W: 発行可能なクレデンシャルのカタログ
    W->>U: 「社員証を受け取りますか?」+ tx_code 入力
    W->>I: トークンリクエスト<br/>POST /token (pre-authorized_code + tx_code)
    I-->>W: アクセストークン
    W->>I: POST /nonce
    I-->>W: c_nonce(使い捨てチャレンジ)
    Note over W: 鍵ペアを生成し、c_nonce を含む<br/>proof JWT に秘密鍵で署名
    W->>I: クレデンシャルリクエスト<br/>POST /credential (access_token + proof)
    I-->>W: VC(SD-JWT VC + 全ディスクロージャ)
    W->>I: POST /notification(保存成功を通知)
```

### 5.4 流通するデータの具体例

#### (1) Credential Offer

QR コードやディープリンクに載る URL:

```
openid-credential-offer://?credential_offer_uri=https%3A%2F%2Fissuer.example.com%2Foffers%2F1f8e02...
```

`credential_offer_uri` を GET すると得られる中身(値渡しで直接埋め込むことも可能):

```json
{
  "credential_issuer": "https://issuer.example.com",
  "credential_configuration_ids": ["EmployeeCredential"],
  "grants": {
    "urn:ietf:params:oauth:grant-type:pre-authorized_code": {
      "pre-authorized_code": "SplxlOBeZQQYbYS6WxSbIA",
      "tx_code": {
        "input_mode": "numeric",
        "length": 4,
        "description": "社内メールに届いた 4 桁のコードを入力してください"
      }
    }
  }
}
```

#### (2) Issuer メタデータ(`/.well-known/openid-credential-issuer`)

```json
{
  "credential_issuer": "https://issuer.example.com",
  "credential_endpoint": "https://issuer.example.com/credential",
  "nonce_endpoint": "https://issuer.example.com/nonce",
  "deferred_credential_endpoint": "https://issuer.example.com/deferred",
  "notification_endpoint": "https://issuer.example.com/notification",
  "credential_configurations_supported": {
    "EmployeeCredential": {
      "format": "dc+sd-jwt",
      "vct": "https://credentials.example.com/employee_credential",
      "cryptographic_binding_methods_supported": ["jwk"],
      "credential_signing_alg_values_supported": ["ES256"],
      "proof_types_supported": {
        "jwt": { "proof_signing_alg_values_supported": ["ES256"] }
      },
      "display": [
        { "name": "社員証", "locale": "ja-JP", "background_color": "#12107c" }
      ],
      "claims": [
        { "path": ["family_name"], "display": [{ "name": "姓", "locale": "ja-JP" }] },
        { "path": ["given_name"],  "display": [{ "name": "名", "locale": "ja-JP" }] },
        { "path": ["employee_id"], "display": [{ "name": "社員番号", "locale": "ja-JP" }] }
      ]
    }
  }
}
```

#### (3) トークンリクエスト/レスポンス

```http
POST /token HTTP/1.1
Host: issuer.example.com
Content-Type: application/x-www-form-urlencoded

grant_type=urn:ietf:params:oauth:grant-type:pre-authorized_code
&pre-authorized_code=SplxlOBeZQQYbYS6WxSbIA
&tx_code=1234
```

```json
{
  "access_token": "eyJhbGciOiJFUzI1NiJ9...",
  "token_type": "Bearer",
  "expires_in": 86400,
  "authorization_details": [
    {
      "type": "openid_credential",
      "credential_configuration_id": "EmployeeCredential",
      "credential_identifiers": ["EmployeeCredential-instance-1"]
    }
  ]
}
```

#### (4) proof JWT(鍵所有証明)

ウォレットは「これから発行される VC を紐づけたい鍵を、確かに自分が持っている」ことを証明する JWT を作ります。

```json
// ヘッダ
{
  "typ": "openid4vci-proof+jwt",
  "alg": "ES256",
  "jwk": { "kty": "EC", "crv": "P-256", "x": "TCAER19Zvu...", "y": "ZxjiWWbZMQ..." }
}
// ペイロード
{
  "aud": "https://issuer.example.com",
  "iat": 1767225600,
  "nonce": "tZignsnFbp"   // ← nonce エンドポイントで取得した c_nonce
}
```

> 💡 この `jwk` の公開鍵が、発行される VC の `cnf` に入ります。つまり **proof は「VC と Holder の鍵を結びつける儀式」**です。

#### (5) クレデンシャルリクエスト/レスポンス

```http
POST /credential HTTP/1.1
Host: issuer.example.com
Authorization: Bearer eyJhbGciOiJFUzI1NiJ9...
Content-Type: application/json

{
  "credential_configuration_id": "EmployeeCredential",
  "proofs": { "jwt": ["eyJ0eXAiOiJvcGVuaWQ0dmNpLXByb29mK2p3dCIs..."] }
}
```

```json
{
  "credentials": [
    { "credential": "eyJ0eXAiOiJkYytzZC1qd3QiLCJhbGciOiJFUzI1NiJ9.eyJfc2QiOlsi...~WyJfMjZi...~WyJhczFl...~" }
  ],
  "notification_id": "3fwe98js"
}
```

すぐに発行できない場合は代わりに `transaction_id` が返り、ウォレットは後から Deferred Credential Endpoint に取りに行きます。

---

## 6. OID4VP:VP 提示・検証プロトコル

### 6.1 全体像

OID4VP は **OAuth 2.0 の認可リクエストを拡張**し、「アクセストークン」の代わりに「**VP トークン**」を返してもらうプロトコルです。Verifier がクライアント、ウォレットが認可サーバーの位置づけになります。

2 つの利用形態があります:

- **Same-device**: スマホ内のブラウザ → 同じスマホのウォレットへリダイレクト
- **Cross-device**: PC のブラウザに QR コード表示 → スマホのウォレットで読み取り(応答は `direct_post` で Verifier のサーバーへ直接 POST)

### 6.2 シーケンス(Cross-device、direct_post の例)

```mermaid
sequenceDiagram
    autonumber
    participant U as ユーザー
    participant W as ウォレット (Holder)
    participant V as Verifier

    U->>V: サービス利用開始(例: 年齢確認が必要)
    V->>U: 認可リクエストを QR コードで提示<br/>(client_id + request_uri)
    U->>W: QR を読み取り
    W->>V: GET request_uri(リクエストオブジェクト取得)
    V-->>W: 署名付きリクエスト JWT<br/>(dcql_query, nonce, response_uri, ...)
    Note over W: Verifier の身元を検証(X.509 等)し、<br/>要求内容をユーザーに表示
    W->>U: 「Verifier X が『年齢が 20 歳以上』の提示を求めています」
    U->>W: 同意 + 開示する属性を選択
    Note over W: 選択したディスクロージャ + KB-JWT で VP を構成
    W->>V: POST response_uri(vp_token)※direct_post
    V-->>W: redirect_uri(完了ページ)
    Note over V: 署名・KB-JWT・nonce・ステータスを検証
```

### 6.3 流通するデータの具体例

#### (1) 認可リクエスト(QR コード / ディープリンクの中身)

```
openid4vp://?client_id=x509_san_dns%3Averifier.example.com
  &request_uri=https%3A%2F%2Fverifier.example.com%2Frequest%2F5FdW...
```

リクエスト本体はサイズが大きいため、`request_uri` 参照渡しが一般的です。取得されるのは **Verifier が署名したリクエストオブジェクト(JWT)**:

```json
// ペイロード
{
  "client_id": "x509_san_dns:verifier.example.com",
  "response_type": "vp_token",
  "response_mode": "direct_post.jwt",
  "response_uri": "https://verifier.example.com/response",
  "nonce": "n-0S6_WzA2Mj",
  "state": "eyJhb...x9zA",
  "dcql_query": {
    "credentials": [
      {
        "id": "employee",
        "format": "dc+sd-jwt",
        "meta": { "vct_values": ["https://credentials.example.com/employee_credential"] },
        "claims": [
          { "path": ["family_name"] },
          { "path": ["employee_id"] }
        ]
      }
    ]
  },
  "client_metadata": {
    "jwks": { "keys": [ { "kty": "EC", "crv": "P-256", "use": "enc", "x": "...", "y": "..." } ] },
    "encrypted_response_enc_values_supported": ["A128GCM"]
  }
}
```

注目ポイント:

- **`client_id` のプレフィックス**(`x509_san_dns:` 等)が「Verifier の身元をどう検証するか」を指定します。`x509_san_dns` なら、リクエスト JWT の `x5c` ヘッダにある X.509 証明書チェーンを検証し、証明書の SAN と DNS 名が一致することを確認します。
- **`dcql_query`(DCQL: Digital Credentials Query Language)**が「どのクレデンシャルの、どの属性がほしいか」の問い合わせ言語です。旧ドラフトの `presentation_definition`(DIF Presentation Exchange)は **1.0 で廃止され、DCQL に一本化**されました。
- **`nonce`** はこの取引のためのチャレンジで、後で KB-JWT に入って返ってきます。

#### (2) 認可レスポンス(VP トークンの提出)

ウォレットは `response_uri` へ直接 POST します(`direct_post`)。

```http
POST /response HTTP/1.1
Host: verifier.example.com
Content-Type: application/x-www-form-urlencoded

vp_token=%7B%22employee%22%3A%5B%22eyJ0eXAiOiJkYytzZC1qd3QiLCJhbGciOiJFUzI1NiJ9...~WyJfMjZi...~eyJ0eXAiOiJrYitqd3QiLCJhbGciOiJFUzI1NiJ9...%22%5D%7D
&state=eyJhb...x9zA
```

`vp_token` のデコード後(**DCQL のクエリ ID がキー**の JSON オブジェクト):

```json
{
  "employee": [
    "<Issuer 署名つき SD-JWT>~<family_name のディスクロージャ>~<employee_id のディスクロージャ>~<KB-JWT>"
  ]
}
```

`response_mode=direct_post.jwt` の場合、このレスポンス全体を Verifier の公開鍵(`client_metadata.jwks`)宛てに **JWE で暗号化**して送ります。経路上(ブラウザやプロキシのログ等)に属性情報が平文で残ることを防ぐためで、HAIP では必須です。

#### (3) Verifier の検証チェックリスト

Verifier(検証基盤)は少なくとも以下を確認します:

1. **Issuer 署名の検証**: SD-JWT の署名を Issuer の公開鍵で検証(`x5c` チェーン、または `iss` に基づく JWKS/DID 解決)
2. **Issuer の信頼判定**: 証明書チェーンがトラストアンカー(トラストリスト)に連なるか
3. **ディスクロージャの整合**: 各ディスクロージャのハッシュが `_sd` に含まれるか
4. **KB-JWT の検証**: `cnf` の公開鍵で署名検証。`nonce` が自分の発行値と一致、`aud` が自分、`sd_hash` が提示物と一致、`iat` が許容範囲内
5. **有効期限・ステータス**: `exp` と、ステータスリスト(次章)で失効していないこと
6. **クレーム内容の確認**: `vct` が期待した種別か、要求した属性が揃っているか

---

## 7. クレデンシャルの失効・ステータス管理

「退職した社員の社員証 VC」を無効化するには、**IETF Token Status List** を使います(HAIP でも採用)。

仕組み:

1. Issuer は VC 発行時に `status` クレームを埋め込む:

```json
"status": {
  "status_list": {
    "idx": 412,
    "uri": "https://issuer.example.com/statuslists/1"
  }
}
```

2. `uri` の先には、**全 VC のステータスを 1〜数ビットずつ詰めた圧縮ビット列**(に Issuer が署名した Status List Token)が公開されている
3. Verifier は検証時にこのリストを取得し、`idx` 番目のビットを見る(`0x00`=有効、`0x01`=失効 など)

> 💡 **プライバシー面の工夫**: Verifier が Issuer に「この VC は有効?」と 1 件ずつ問い合わせる方式だと、Issuer に利用状況が漏れます。ステータスリストは**大量の VC のステータスをまとめて配布**するため、どの VC を検証したのかが Issuer に分かりにくい設計です。

---

## 8. 用語集

| 用語 | 意味 |
|---|---|
| VC (Verifiable Credential) | 発行者の署名つきデジタル属性証明 |
| VP (Verifiable Presentation) | VC に持ち主の署名(KB-JWT 等)を付けて検証者へ提示するパッケージ |
| Issuer / Holder / Verifier | 発行者 / 保持者 / 検証者 |
| ウォレット | Holder が VC を保管・提示するためのアプリ(スマホアプリ等) |
| SD-JWT | 選択的開示可能な JWT(IETF) |
| SD-JWT VC | SD-JWT を VC 用途にプロファイルしたフォーマット(`dc+sd-jwt`) |
| mdoc / mDL | ISO/IEC 18013-5 のモバイル文書/モバイル運転免許証(`mso_mdoc`) |
| ディスクロージャ | SD-JWT で属性の実体を運ぶ部品。ハッシュで本体と紐づく |
| Key Binding (KB-JWT) | 提示時に Holder が秘密鍵で署名するチャレンジ応答。持ち主証明 |
| `cnf` | VC に埋め込まれる Holder の公開鍵(confirmation) |
| `vct` | SD-JWT VC のクレデンシャル種別識別子 |
| Credential Offer | Issuer からウォレットへの発行案内(QR 等) |
| Pre-Authorized Code | 事前本人確認済みユーザー向けの発行用ワンタイムコード |
| `tx_code` | Pre-Authorized Code Flow で別送される短い PIN |
| proof (proof JWT) | 発行時にウォレットが行う鍵所有証明 |
| `c_nonce` | proof に入れる Issuer 発行の使い捨てチャレンジ |
| DCQL | Verifier が要求内容を記述するクエリ言語(OID4VP 1.0 で標準) |
| `direct_post` / `direct_post.jwt` | VP を Verifier のサーバーへ直接 POST するレスポンスモード(`.jwt` は暗号化つき) |
| Status List | VC の失効管理用ビット列(IETF Token Status List) |
| DID | 分散型識別子。OID4VC では利用可能だが必須ではない |
| HAIP | 高保証・相互運用プロファイル。詳細は [③](./03_haip-migration-guide.md) |

---

## 9. 参考資料

- [OpenID for Verifiable Credential Issuance 1.0(Final)](https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0-final.html)
- [OpenID for Verifiable Presentations 1.0(Final)](https://openid.net/specs/openid-4-verifiable-presentations-1_0-final.html)
- [OpenID4VC High Assurance Interoperability Profile 1.0(Final)](https://openid.net/specs/openid4vc-high-assurance-interoperability-profile-1_0-final.html)
- [SD-JWT (IETF OAuth WG: Selective Disclosure for JWTs)](https://datatracker.ietf.org/doc/draft-ietf-oauth-selective-disclosure-jwt/)
- [SD-JWT-based Verifiable Credentials (SD-JWT VC)](https://datatracker.ietf.org/doc/draft-ietf-oauth-sd-jwt-vc/)
- [Token Status List (IETF OAuth WG)](https://datatracker.ietf.org/doc/draft-ietf-oauth-status-list/)
- [Authlete: OpenID for Verifiable Credential Issuance(日本語解説)](https://www.authlete.com/ja/developers/oid4vci/)
- [Qiita: OpenID for Verifiable Credentials 解説(川崎貴彦氏)](https://qiita.com/TakahikoKawasaki/items/e37caf50776e00e733be)
