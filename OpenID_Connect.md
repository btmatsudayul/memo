# OpenID Connect調査メモ
<!-- TOC -->

- [JWT](#jwt)
    - [ID Token](#id-token)
    - [UserInfo Response](#userinfo-response)
    - [Request Object](#request-object)
    - [Aggregated Claims](#aggregated-claims)
    - [Client Authentication](#client-authentication)
    - [Summary](#summary)
- [Flow](#flow)
    - [Authentication using the Authorization Code Flow](#authentication-using-the-authorization-code-flow)
    - [Authentication using the Implicit Flow](#authentication-using-the-implicit-flow)
    - [Authentication using the Hybrid Flow](#authentication-using-the-hybrid-flow)

<!-- /TOC -->

## JWT

OpenID Connectにおける主なJWTの利用用途は以下の通り。

  - [ID Token](#id-token) 
    - Token Response時の追加パラメータ
    - [OIDC 2.ID Token](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#IDToken)
  - [UserInfo Response](#userinfo-response)
    - UserInfo EndpoitからのResponse（⇒
    - [OIDC 5.3. UserInfo Endpoint](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#UserInfo)
  - [Request Object](#request-object)
    - Authorization Request時の追加パラメータ
    - [OIDC 5.5. Requesting Claims using the "claims" Request Parameter](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#ClaimsParameter)
  - [Aggregated Claims](#aggregated-claims) ・・・
    - Claimsの種別
    - [OIDC 5.6. Claim Types](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#ClaimTypes)
  - [Client Authentication](#client-authentication)
    - Token Resquest時のクライアント認証
    - [OIDC 9. Client Authentication](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#ClientAuthentication)

### ID Token

[概要]
  - エンドユーザの認証に関するクレームを含んだセキュリティトークン。
  - OpenID Connectでは、OAuth 2.0にて定義されているリクエスト/レスポンスパラメータに加えて、各フローでID Tokenが付与される（認可コード、アクセストークンとは別情報としてID Tokenが付与される）。
  - ID Tokenは（Authorization Code Flowの場合）Authorization ServerがToken Endpointにて発行する。

  **TODO: ID TokenのClaimについて整理する。**

[利用目的]

  - ①：Clientへのユーザ情報＋独自パラメータの連携。
    - ClientがID Tokenとして返却を要求するパラメータは、Authorization Requestパラメータ`id_token`で指定することも可能。
      - [OIDC 5.5. Requesting Claims using the "claims" Request Parameter](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#ClaimsParameter)
  - ②：Token Response、またはAuthentication Responseの妥当性の検証。
    - ClientではAccess Token、ID Tokenの検証を行うことで、応答が改ざんされていないことを検証。
      - [OIDC 3.1.3.5. Token Response Validation(Authorization Code Flow)](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#TokenResponseValidation)
      - [OIDC 3.2.2.8. Authentication Response Validation(Implicit Flow)](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#ImplicitAuthResponseValidation)
      - [OIDC 3.3.2.8. Authentication Response Validation(Hybrid Flow)](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#HybridAuthResponseValidation)
    - ID TokenはAccess Token、Authorization codeとは別情報であるが、ID Tokenに含める`at_hash`、`c_hash`によってID TokenとAccess Token、Authorization codeの紐付きを検証することも可能。

[電文例]
```
  ## Token Response ##
  HTTP/1.1 200 OK
  Content-Type: application/json
  Cache-Control: no-cache, no-store
  Pragma: no-cache
  {
   "access_token":"SlAV32hkKG",
   "token_type":"Bearer",
   "expires_in":3600,
   "refresh_token":"tGzv3JOkF0XG5Qx2TlKWIA",
   "id_token":"eyJ0 ... NiJ9.eyJ1c ... I6IjIifX0.DeWt4Qu ... ZXso"
  }
```

### UserInfo Response

[概要]
  - UserInfo Endpointより返却される、エンドユーザに関するクレーム。
  - Clientは、Acccess Tokenを用いてUserInfo Endpointに要求することでUserInfo Responseを取得する。
    - OAuth 2.0でいうところの、Resource ServerからのResponseのユーザ情報版。
  - UserInfo Responseが署名、または暗号化されている場合はJWT、署名、または暗号化されていない場合はJSONにて返却する。


[利用目的]
  - UserInfo Endpointから返却する情報への署名、または暗号化。
    - 署名を行う場合はJWSを適用。
    - 暗号化を行う場合はJWSにて署名を適用したJWTにJWEを適用する。

[電文例]
```
  ## UserInfo Response (using JSON)
  HTTP/1.1 200 OK
  Content-Type: application/json

  {
   "sub": "248289761001",
   "name": "Jane Doe",
   "given_name": "Jane",
   "family_name": "Doe",
   "preferred_username": "j.doe",
   "email": "janedoe@example.com",
   "picture": "http://example.com/janedoe/me.jpg"
  }
```
```
  ## UserInfo Response (using JWT)
  HTTP/1.1 200 OK
  Content-Type: application/jwt

  eyJhbGciOiJSUzI1NiIsImtpZCI6IjEifQ.eyJzdWIiOiJhbGl....
```

### Request Object

[概要]
  - Authorization Requestに付与されるパラメータ。
  - 付与するRequest ObjectをJWT(`request`)、またはリソースへの参照(`request_uri`)として指定する。
  - Reqeust Objectは署名、暗号化なしでもよい（が、そのような利用はおそらくありえない？？）。
  - パラメータ指定は以下のとおり。なお、同時利用は不可とする。
    - `request`：
      - リクエストパラメータをクレームとしたJWT（Request Object）を指定。オプション。
      - 本パラメータ指定時は、OAuth 2.0のパラメータと併用することが可能。
        - 有効なOAuth 2.0リクエストとするために必須パラメータはリクエストシンタックスに指定する必要がある。
        - 同じパラメータをリクエストシンタックス、Request Objectの両方に指定する場合、値が一致する必要がある。
    - `request_uri`：
      - Request Object値を含むリソースへの参照。オプション。
      - Authorization Serverは本パラメータを含むリクエストを受信した場合、`request_uri`にHTTP GETリクエストを送らなければならない。
    - [OIDC 6. Passing Request Parameters as JWTs](http://openid-foundation-japan.github.io/openid-connect-core-1_0.ja.html#JWTRequests)

[利用目的]

  - ①：Authorization Requestに含めるパラメータの署名、暗号化（`request`、`request_uri`）
    - （HTTP GETメソッドを利用する場合に）URIに生値を指定したくないなど。
  - ②：パラメータの共通化（`request`、`request_uri`）
    - サービスで不変なパラメータをJWTとしてまとめたい、事前に登録しておきたい（`request_uri`）など。
  - ③：Authorization Requestサイズの抑止（`request_uri`）
    - Requestサイズが大きくなってしまう場合、実値を参照させることでURI、Formサイズを抑止したいなど。

[電文例]
```
  ## Authorization Request (using request) ##
  https://server.example.com/authorize?
     response_type=code%20id_token
     &client_id=s6BhdRkqt3
     &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
     &scope=openid
     &state=af0ifjsldkj
     &nonce=n-0S6_WzA2Mj
     &request=eyJhbGciOiJSUzI1NiIsImtpZCI6ImsyYmRjIn0.ew0KICJpc3MiOiA
     iczZCaGRSa3F0MyIsDQogImF1ZCI6ICJodHRwczovL3NlcnZlci5leGFtcGxlLmN
     vbSIsDQogInJlc3BvbnNlX3R5cGUiOiAiY29kZSBpZF90b2tlbiIsDQogImNsaWV
     udF9pZCI6ICJzNkJoZFJrcXQzIiwNCiAicmVkaXJlY3RfdXJpIjogImh0dHBzO...
```
```
  ## Authorization Request (using request_uri) ##
  https://server.example.com/authorize?
    response_type=code%20id_token
    &client_id=s6BhdRkqt3
    &request_uri=https%3A%2F%2Fclient.example.org%2Frequest.jwt
    %23GkurKxf5T0Y-mnPFCHqWOMiZi4VS138cQO_V7PZHAdM
    &state=af0ifjsldkj&nonce=n-0S6_WzA2Mj
    &scope=openid
```

### Aggregated Claims

[概要]
  - OpenID Providerから返却されるClaimの表現方式の一つ。Claimsの値がJWTにて表現される。

**Claims形式**

|Type              |説明             |
|:-----------------|:---------------|
|Normal Claims     |OpenID Providerによって直接アサートされたClaim。JSONオブジェクト形式。|
|Aggregated Claims |OpenID Provider以外のClaims ProviderによってアサートされたClaim。OpenID Providerからは値が返却される。|
|Distributed Claims|OpenID Provider以外のClaims ProviderによってアサートされたClaim。OpenID Providerからは値への参照が返却される。|

  - OpenID ProviderとしてはNormal Claimsのみ対応が必須。
  - Claims Provider：Claimを管理しているサーバ。

[利用目的]

  - ①：OpenID Proviederを通じて、Claims ProviderのClaimの返却など（OPのProxy化）
    - OpenID Provider：API-GW、Claims Provider：既存の認証サーバとなるような構成、かつOpenID ProviderではClaims Providerからの応答（JWT）をそのまま流したいなど？

```
一般的に, どのような時に Aggregated Claims と Distributed Claims の利用が適切かは, OP 次第である. 場合によっては, どの Claim Type をいつ使うべきかについて, RP と OP の間で out-of-band な方法で合意されることもある. 
```

[電文例]
```
  ## Normal Claims ##
  {
   "name": "Jane Doe",
   "given_name": "Jane",
   "family_name": "Doe",
   "email": "janedoe@example.com",
   "picture": "http://example.com/janedoe/me.jpg"
  }
```
```
  ## Aggregated Claims(_claim_names, _claim_sources) ##
  {
   "name": "Jane Doe",
   "given_name": "Jane",
   "family_name": "Doe",
   "birthdate": "0000-03-22",
   "eye_color": "blue",
   "email": "janedoe@example.com",
   "_claim_names": {
     "address": "src1",
     "phone_number": "src1"
   },
   "_claim_sources": {
     "src1": {"JWT": "jwt_header.jwt_part2.jwt_part3"}
   }
  }
```
```
  ## Distributed Claims(_claim_names, _claim_sources) ##
  {
   "name": "Jane Doe",
   "given_name": "Jane",
   "family_name": "Doe",
   "email": "janedoe@example.com",
   "birthdate": "0000-03-22",
   "eye_color": "blue",
   "_claim_names": {
     "payment_info": "src1",
     "shipping_address": "src1",
     "credit_score": "src2"
    },
   "_claim_sources": {
     "src1": {"endpoint":
                "https://bank.example.com/claim_source"},
     "src2": {"endpoint":
                "https://creditagency.example.com/claims_here",
              "access_token": "ksj3n283dke"}
   }
  }
```

### Client Authentication

[概要]
  - ClientがToken Endpointにアクセスする際、自身を認証する方式。
  - OpenID Connectでは、従来の方式（Basic認証等）に加えて、JWTの連携による認証方式の指定が可能となっている。
  - client_secret_jwt、private_key_jwtの必要パラメータについてはRFC参照。

**Client Authentication方式**

|Type               |説明             |備考  |
|:------------------|:---------------|:----|
|client_secret_basic|`client_secret`をHTTP Basic 認証スキーマを利用して自身を認証する |  |
|client_secret_post |`client_secret`をリクエストボディに含めて自身を認証する |Client Credential方式？ |
|client_secret_jwt  |HMAC SHA-256等のHMAC SHAアルゴリズムを利用し、`client_secret`を共通鍵として作成したJWTにて自身を認証する |JWT Bearer Tokenを利用 |
|private_key_jwt    |事前に交換済みの秘密鍵で署名したJWTにて自身を認証する |JWT Bearer Tokenを利用 |
|none               |認証しない         |Implicit FlowまたはPublicなClientなど |

[利用目的]

  - OpenID Conenctでは
    - Public Client：指定なし（none？他の代替手段で認証する？）
    - Confidential Client：client_secret_jwt or private_key_jwt

[電文例]
```
  POST /token HTTP/1.1
  Host: server.example.com
  Content-Type: application/x-www-form-urlencoded

  grant_type=authorization_code&
    code=i1WsRn1uB1&
    client_id=s6BhdRkqt3&
    client_assertion_type=
    urn%3Aietf%3Aparams%3Aoauth%3Aclient-assertion-type%3Ajwt-bearer&
    client_assertion=PHNhbWxwOl ... ZT
```

### Summary

| Flow |Req./Res. |From |To   |ID Token |UserInfo Res. |Req. Object |備考 |
|:-----|:---------|:----|:----|:-------:|:------------:|:----------:|:---|
| Authorization Code Flow |Authentication Request  |RP |OP |X| | |`id_token_hint`：前回発行されたID Token(OPTIONAL) |
|                         |                        |   |   | | |X|`request`<br>`request_uri`：リクエストに含めるRequest Object(OPTIONAL) |
|                         |Authentication Response |OP |OP | | | | |
|                         |Token Request           |RP |OP | | | | |
|                         |Token Response          |OP |RP |X| | |`id_token`：新規発行したID Token |
|Implicit Flow            |Authentication Request  |RP |OP |X| | |`id_token_hint`：前回発行されたID Token(OPTIONAL) |
|                         |                        |   |   | | |X|`request`<br>`request_uri`：リクエストに含めるRequest Object(OPTIONAL) |
|                         |Authentication Response |OP |RP |X| | |`id_token`：新規発行したID Token |
|Hybrid Flow              |Authentication Request  |RP |OP |X| | |`id_token_hint`：前回発行されたID Token(OPTIONAL) |
|                         |                        |   |   | | |X|`request`<br>`request_uri`：リクエストに含めるRequest Object(OPTIONAL) |
|                         |Authentication Response |OP |RP |X| | |`id_token`：`response_type`が`code id_token` `code id_token token`の場合のみ新規発行したID Token |
|                         |Token Request           |RP |OP | | | | |
|                         |Token Response          |OP |RP |X| | |`id_token`：新規発行したID Token |
|共通                      |UserInfo Request        |RP |OP | | | | |
|                         |UserInfo Response       |OP |RP | |X| | |

  - 注：Authentication Requestの `claims` パラメータには `id_token` というメンバが存在するが、
      これはID Tokenではなく返却されるID Tokenに要求するパラメータの指定。


## Flow

### Authentication using the Authorization Code Flow

[概要]
  - **TODO：なにかかく**

  1. Client は必要なパラメータを含む Authentication Request を用意する. 
  2. Client は Authorization Server にリクエストを送信する. 
  3. Authorization Server は End-User を Authenticate する. 
  4. Authorization Server は End-User の Consent/Authorization を得る. 
  5. Authorization Server は Authorization Code を添えて End-User を Client に戻す. 
  6. Client は Token Endpoint へ Authorization Code を送信する. 
  7. Client は ID Token と Access Token をレスポンスボディに含むレスポンスを受け取る. 
  8. Client は ID Token を検証し, End-User の Subject Identifier を取得する. 

[利用目的]


### Authentication using the Implicit Flow

[概要]
  - **TODO：なにかかく**

  1. Client は必要なパラメータを含む Authentication Request を用意する. 
  2. Client は Authorization Server にリクエストを送信する. 
  3. Authorization Server は End-User を Authenticate する. 
  4. Authorization Server は End-User の Consent/Authorization を得る. 
  5. Authorization Server は ID Token, および要求があれば Access Token を添えて End-User を Client に戻す. 
  6. Client は ID Token を検証し, End-User の Subject Identifier を取得する. 

[利用目的]



### Authentication using the Hybrid Flow

[概要]
  - **TODO：なにかかく**

  1. Clientは必要なリクエストパラメータを含んだ Authentication Request を構築する. 
  2. Clientは Authorization Server にリクエストを送信する. 
  3. Authorization Server は End-User を認証する. 
  4. Authorization Server は End-User の同意/認可を取得する. 
  5. Authorization Server は End-User を Authorization Code と, レスポンスタイプに応じた1つ以上の追加パラメ  ータとともにClientに戻す. 
  6. Clientは Token Endpoint で Authorization Code を使用してレスポンスを要求する. 
  7. Clientは ID Token と Access Token をレスポンスボディに含んだレスポンスを受け取る. 
  8. Clientは ID Token をトークンを検証し, End-User の Subject Identifier を取得する. 

[利用目的]
