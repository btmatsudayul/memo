# Client Authentication(RFC7521, RFC7523)調査メモ

<!-- TOC -->

- [Overview](#overview)
- [Terminology](#terminology)
- [Framework](#framework)
    - [How to issue Assertion](#how-to-issue-assertion)
        - [Assertion Created by Third Party](#assertion-created-by-third-party)
        - [Self-Issued Assertion](#self-issued-assertion)
    - [Assertion Type](#assertion-type)
    - [Transporting Assertions](#transporting-assertions)
        - [Using Assertions for Client Authentication](#using-assertions-for-client-authentication)
        - [Using Assertions as Authorization Grants](#using-assertions-as-authorization-grants)
    - [Processing Rules](#processing-rules)
- [JSON Web Token (JWT) Bearer Token](#json-web-token-jwt-bearer-token)
    - [Transporting Assertions](#transporting-assertions-1)
        - [Using JWTs as Authorization Grants](#using-jwts-as-authorization-grants)
        - [Using JWTs for Client Authentication](#using-jwts-for-client-authentication)

<!-- /TOC -->

## Overview

  - [RFC 7521：Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants](https://tools.ietf.org/html/rfc7521)
    - OAuth 2.0において、Assertionを利用して認証、または認可を行うためのフレームワークに関する仕様。
    - 認証のためのAssertionと、認可のためのAssertionは個別に利用することができる。
    - RFC 7521では抽象的なメッセージフローと処理ルールのみを定義しており、実装を行う際には関連するほかの仕様を参照する必要がある。
  - [RFC 7523：JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants](https://tools.ietf.org/html/rfc7523)
    - JWTを利用してRFC 7521を実現するための具体的な利用方法を定義した仕様。
    - OpenID Connectではこちらを利用してクライアント認証を行う必要がある。

## Terminology
| 用語 | 説明 |
|:-----|:-----|
|Assertion|IDとセキュリティ情報をセキュリティドメイン間で共有できる情報パッケージ。<br>Assertionには発行者、発行日時、Assertionが有効である条件などが含まれる。 |
|Client |自身の認証、認可のためにAssertionを利用するエンティティ。 |
|Issuer |Assertionを作成、書名するエンティティ。Assertionの発行者。 |
|Relying Party |Clientから提示されたAssertionを消費するエンティティ。<br>OpenID Connectにおける`Relying Party`とは異なることに注意。 |
|JSON Web Token(JWT) Bearer Token|JWTにより表現したAssertion（Bearer Assertions）。<br>JWTにより表現した **Access Tokenではない** 。|


## Framework

### How to issue Assertion

ClientがAssertionの取得するプロセスはは、以下の２パターン存在する。

  - サードパーティがIssuerとなる
  - Client自身がIssuerとなる

#### Assertion Created by Third Party

Assertionを発行/更新/変換/検証することのできるサードパーティのエンティティ（Security Token Service:STS）を利用してAsssertionを取得するパターン。

Assertionを発行する発行するための代表的な標準仕様は[WS-Trust](https://msdn.microsoft.com/ja-jp/library/cc964071.aspx)（内容未確認）。

```
     Relying
     Party                     Client                   Token Service
       |                          |                         |
       |                          |  1) Request Assertion   |
       |                          |------------------------>|
       |                          |                         |
       |                          |  2) Assertion           |
       |                          |<------------------------|
       |    3) Assertion          |                         |
       |<-------------------------|                         |
       |                          |                         |
       |    4) OK or Failure      |                         |
       |------------------------->|                         |
       |                          |                         |
       |                          |                         |

                Figure 1: Assertion Created by Third Party
```

#### Self-Issued Assertion

AssertionをClient自身が発行するパターン。

Assertionに対して署名等を行う場合には、対称鍵または非対称鍵が必要となるが、この鍵の取得プロセスについてはRFC 7521では未定義。

サードパーティを利用しないため、cliet secretなどをAssertionに含めることができる（＝Relying PartyにおいてClientの認証にAssertionを利用できる？）。

```

     Relying
     Party                     Client
       |                          |
       |                          | 1) Create
       |                          |    Assertion
       |                          |--------------+
       |                          |              |
       |                          | 2) Assertion |
       |                          |<-------------+
       |    3) Assertion          |
       |<-------------------------|
       |                          |
       |    4) OK or Failure      |
       |------------------------->|
       |                          |
       |                          |

                      Figure 2: Self-Issued Assertion
```

### Assertion Type

Assertionの内容によってRelying Partyにおける要件が異なることから、Assertionには以下の２タイプが存在する。

  - Bearer Assertions
    - 鍵の有無にかかわらず、Bearer Assertionsの所有者すべてが利用することのできるAssertion。
    - Assertionが漏れることの無いよう、すべてのエンティティ間でセキュアな通信チャネルを利用する必要がある。
  - Holder-of-Key Assertions
    - 利用のためにキーが必要となるAssertion。
    - Assertionを発行する際にキーの識別子をバインドし、ClientはRelying Partyに対してその識別子に対応するキーを知っていることを示す必要がある。
    - **RFC7521, 7523では本Assertionの利用は対象外**。

### Transporting Assertions

認可サーバ（OpenID ConnectではOP）のToken EndpointへAssertionを送信する場合の定義について。
以下の２パターンが存在する。

  - Assertionを認証に利用する
  - Assertionを認可（認可グラント）に利用する

#### Using Assertions for Client Authentication

Assertionをクライアントの資格情報として利用するケース。
Assertionの関連情報として、以下パラメータを含む。

トークンエンドポイントでは、リクエスト中の`client_assertion_type`、`client_assertion`の存在有無をチェックすることで、
Assertionベースのクライアント認証であるか否かを判断できる。


  - client_assertion_type
    - 認可サーバにて定義されたAssertionの形式。絶対URI形式。
    - 必須。
  - client_assertion
    - クライアントの認証に利用されるAssertion。Assertionの実装方式により設定内容は異なる。
    - 必須。
  - client_id
    - クライアントID。
    - （Assertionの`sub`に同情報が設定されるため）オプション。

```
     POST /token HTTP/1.1
     Host: server.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=authorization_code&
     code=n0esc3NRze7LTCu7iYzS6a5acc3f0ogp4&
     client_assertion_type=urn%3Aietf%3Aparams%3Aoauth
     %3Aclient-assertion-type%3Asaml2-bearer&
     client_assertion=PHNhbW...[omitted for brevity]...ZT
```

#### Using Assertions as Authorization Grants

Assertionを認可グラントとして利用するケース。

  - 具体例・・・OAuth 2.0のグラントタイプを拡張して、フローを変更する、Token Requestに業務独自項目を含めるなど？

この場合、リフレッシュトークンは発行されない。
Clientは、同じAssertionを利用して新しいAssertionを要求したり、新しいAssertionが有効である場合は
期限切れのアクセストークンをリフレッシュすることができる。

  - grant_type
    - 認可サーバにて定義されたAssertionの形式。絶対URI形式。
    - 必須。
  - assertion
    - 認可に利用されるAssertion。Assertionの実装方式により設定内容は異なる。
    - 必須。
  - scope
    - 認可要求時のスコープと同じか、それ以下のスコープを指定する必要がある。
    - オプション。

```
     POST /token HTTP/1.1
     Host: server.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Asaml2-bearer&
     assertion=PHNhbWxwOl...[omitted for brevity]...ZT4
```

### Processing Rules

その他、Assertion利用時の一般的なルールについては[RFC 7521 5. Assertion Content and Processing](https://tools.ietf.org/html/rfc7521#section-5)参照。


## JSON Web Token (JWT) Bearer Token

以下、JWTをAssertionに利用する場合の補足。

各JWTに含める必要のあるパラメータ等、ルールについては[RFC 7523 3. JWT Format and Processing Requirements](https://tools.ietf.org/html/rfc7523#section-3)参照。

### Transporting Assertions

#### Using JWTs for Client Authentication

基本的なパラメータは[Using Assertions for Client Authentication](#using-assertions-for-client-authentication)参照。

RFC 7521との違いは以下。

  - client_assertion_type
    - `urn：ietf：params：oauth：client-assertion-type：jwt-bearer` 固定。
  - client_assertion
    - 単一のJWTを指定。

```
     POST /token.oauth2 HTTP/1.1
     Host: as.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=authorization_code&
     code=n0esc3NRze7LTCu7iYzS6a5acc3f0ogp4&
     client_assertion_type=urn%3Aietf%3Aparams%3Aoauth%3A
     client-assertion-type%3Ajwt-bearer&
     client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6IjIyIn0.
     eyJpc3Mi[...omitted for brevity...].
     cC4hiUPo[...omitted for brevity...]
```

#### Using JWTs as Authorization Grants

基本的なパラメータは[Using Assertions as Authorization Grants](#using-assertions-as-authorization-grants)参照。

RFC 7521との違いは以下。

  - grant_type
    - `urn：ietf：params：oauth：grant-type：jwt-bearer` 固定。
  - assertion
    - 単一のJWTを指定。

```
     POST /token.oauth2 HTTP/1.1
     Host: as.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
     &ampassertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
     eyJpc3Mi[...omitted for brevity...].
     J9l-ZhwP[...omitted for brevity...]
```


## Links

 - [JWT bearer token authorizationグラント種別](https://techinfoofmicrosofttech.osscons.jp/index.php?JWT%20bearer%20token%20authorization%E3%82%B0%E3%83%A9%E3%83%B3%E3%83%88%E7%A8%AE%E5%88%A5#c65d5829)
 - [OAuth 2.0 JWT ベアラートークンフロー](https://help.salesforce.com/articleView?id=remoteaccess_oauth_jwt_flow.htm&type=0)