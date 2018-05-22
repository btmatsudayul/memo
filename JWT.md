# JWT/JWS/JWE/JWK/JWA調査メモ

<!-- TOC -->

- [Overview](#overview)
- [Terminology](#terminology)
- [Data Structure](#data-structure)
    - [JWS](#jws)
    - [JWE](#jwe)
    - [JWT（Unsecured JWT）](#jwtunsecured-jwt)
- [JWT Claims](#jwt-claims)
    - [Registered Claim Names](#registered-claim-names)
- [JOSE Header](#jose-header)
- [Appendix](#appendix)
    - [JWS Exapmle HMAC-SHA2 Integrity Protection](#jws-exapmle-hmac-sha2-integrity-protection)
        - [Input](#input)
        - [Input(encoded Base64uri)](#inputencoded-base64uri)
        - [Output](#output)

<!-- /TOC -->



## Overview
 1. JWT（JSON Web Token）
    - JWTはJWS、JWEのいずれか、または両方によってエンコードされた
      JSON形式で表現された情報(JWTクレームセット)のこと。
    - JWTクレームセットは0個以上の名前/値のペア（クレーム）から構成される。
    - JWTクレームセットに適用されるエンコード情報はヘッダ（JOSE Header）に含まれ、この内容によりJWTはJWS、またはJWEとして表現される。
    - JWTはJWSやJWEの構造の中に入れ子として含めることができ、単一のJWTに対して署名や暗号化を繰り返し実施することもできる。
 2. JWS（JSON Web Signature）
    - デジタル署名まはたHMACにより保護されたコンテンツをJSONとして表現するための仕様。 
 3. JWE（JSON Web Encryption）
    - 暗号化により保護されたコンテンツをJSONとして表現するための仕様。 
 4. JWK（JSON Web Key）
    - 暗号鍵をJSONとして表現するための仕様。
 5. JWA（JSON Web Algorithms）
    - JWS、JWE、JWKで利用するアルゴリズムの規定。

|目的 |JOSE Header | JWTクレームセットの扱い |
|:----------|:----------|:----------|
|デジタル署名またはHMACを適用 |JWS用 | Base64uri変換後、 PayloadとしてJWSに設定される
|暗号化を適用 |JWE用 | 暗号化・Base64uri変換後、暗号文としてJWEに設定される 


## Terminology
| 用語 | 説明 | 備考 |
|:-----|:-----|:-----|
|JSON Web Token (JWT)|JWSまたはJWEでエンコードされたJSONオブジェクト |"じょっと"と呼ぶ |
|JWT Claims Set| JWTによって伝えられるクレームの集合であるJSONオブジェクト |JWTのボディ部のようなイメージ |
|Claim|名前と値のペアからなる情報 | |
|JOSE Header |使用する暗号方式とそのパラメータを含むJSONオブジェクト |JWT/JWS/JWEのヘッダ部。JOSE：JSON Object Signing and Encryption |
|JSON Web Signature (JWS)|デジタル署名、またはMACされたメッセージを表すデータ構造 | |
|JWS Payload |保護対象となるメッセージ |JWTの場合、JWT Claims Setがこれに該当する。JWSの仕様上はJSONである必要がない点について注意|
|JWS Signature |JWS Protected HeaderとJWS Payloadのデジタル署名| |
|JWS Protected Header | デジタル署名またはHMACによって完全性が保護されるヘッダパラメータを含むJSONオブジェクト | JWS Compact Serializationの場合、ヘッダパラメータ全体（JOSE Header）がJWS Protected Headerとなる |
|Base64url Encoding |Base64エンコーディングをURLで利用するため、一部文字(`=`)を除去したエンコーディング方式 |
|JWS Compact Serialization |URLセーフな文字列で表現したJWS | |
|JWS JSON Serialization |JSONで表現したJWS|OpenID Conncectでは利用されない |
|JSON Web Encryption (JWE) |暗号化されたメッセージを表すデータ構造 | |
|JWE Ciphertext |暗号文 |JWEの場合、JWT Claims Setを暗号化することでこれが生成される |
|JWE Protected Header |暗号化よって完全性が保護されるヘッダパラメータを含むJSONオブジェクト | JWE Compact Serializationの場合、ヘッダパラメータ全体（JOSE Header）がJWE Protected Headerとなる |
|JWE Compact Serialization |URLセーフな文字列で表現したJWE | |
|JWE JSON Serialization |JSONで表現したJWE |OpenID Conncectでは利用されない |
|Nested JWT |JWT（JWS or JWE）の中にJWTがネストして含まれているJWT |JWSのJWS PayloadとしてJWS/JWEが指定されている、またはJWEのJWE CiphertextとしてJWS/JWEが指定されている、など。OpenID ConnectではJWE Ciphertextに暗号化したJWSを設定したJWEがこれにあたる。 |
|Unsecured JWT |クレームが署名、暗号化されていないJWT |JOSE Headerが `"alg":"none"` （署名なし）のJWS |

## Data Structure

### JWS
  - JWSではデジタル署名、またはHMACを行ったコンテンツをURL-safeな文字列として表現したJWS Compact Serializationと、JSONオブジェクトとして表現したJWS JSON Serializationが存在する。
  - JWS JSON Serializationでは、コンテンツに対して複数のデジタル署名、またはHMACを行うことができる。
  - OpenID Connectでは、JWS Compact Serializationのみを利用する。
  - "alg":"HS256"（HMAC SHA-256）の場合、JWS Signatureは
    [エンコード済JWSヘッダ, ピリオド(`.`), エンコード済JWS Payloadを連結したもの]の
    ASCIIオクテット列を元に、HMAC SHA-256アルゴリズムの鍵を利用して生成したオクテット列がJWS Signatureとなる。

```
  ## JWS Compact Serialization データ構造 ##

  Base64uri( UTF-8( <JOSE Header("alg":"HS256"など)> ) )
  .
  Base64uri( <JWS Payload> )
  .
  Base64uri( <JWS Signature> )
```
```
  ## JWS JSON Serialization データ構造 ##

  {
    "payload" : "<Base64uri( <JWS Payload> )>"
    "signatures" : [
      {
        "protected" : "<Base64uri( UTF-8( <署名/HMACに用いるヘッダ> ) )>",
        "header" : "<UTF-8( <署名/HMACに用いないヘッダ> )>"
        "signature" : "<JWS Signature（1回目）>",
        ...
      },
      ...
      {
        "protected" : "<Base64uri( UTF-8( <署名/HMACに用いるヘッダ> ) )>",
        "header" : "<UTF-8( <署名/HMACに用いないヘッダ> )>"
        "signature" : "<JWS Signature（N回目）>",
        ...
      }
    ]
  }
```

### JWE
  - JWSと同様、暗号化を行ったコンテンツをURL-safeな文字列として表現したJWE Compact Serializationと、JSONオブジェクトとして表現したJWE JSON Serializationが存在する。

```
  ## JWE Compact Serialization データ構造 ##

  Base64uri( UTF-8( <JOSE Header> ) )
  .
  Base64uri( <JWE暗号鍵> )
  .
  Base64uri( <JWE初期ベクトル)
  .
  Base64uri( <JWE暗号文> )
  .
  Base64uri( <JWE認証タグ> )
```

```
  ## JWS JSON Serialization データ構造 ##
  使わなそうなので一旦省略・・・
```

### JWT（Unsecured JWT）

  
```
  ## JWT データ構造 ##
  Base64uri( UTF-8( <JOSE Header("alg":"none")> ) )
  .
  Base64uri( <JWTクレームセット>   )
  .
  <指定なし("alg":"none"のため、JWS Signatureがなし)>
```

## JWT Claims

  - **TODO : OpenID Connect内容をベースに記載内容見直し中！！！**


  - JWT クレームセット内のクレーム名は一意でなければならない（MUST）。
  - 受信者はクレーム名が重複したJWTを拒否するか、あるいは最後の重複メンバ名だけを返却するJSONパーサを利用しなければならない（MUST）。
  - 有効とみなされなければならないクレームはコンテキストに依存するため、本仕様の対象外。ただし、そのような要件がアプリケーションによって定められていない場合、理解されないクレームは無視されなければならない（MUST）。


### Registered Claim Names
  
  - "*iss*" (Issuer)
    - JWTの発行者を表す識別子。オプション。
    - *iss*の値は文字列あるいはURI値（StringOrURI）。

  - "*sub*" (Subject)
    - JWTの主題を表す識別子。オプション。
    - *sub*の値は文字列あるいはURI値（StringOrURI）。
    - *sub*は発行者のコンテキスト内でユニークでなければならない（MUST）。

  - "*aud*" (Audience)
    - JWTの利用者を表す識別子。オプション。
    - *aud*の値は文字列あるいはURI値（StringOrURI）。
    - *aud*が含まれる場合、受信者は自身の識別子が設定されていない場合は拒否しなければならない（MUST）。

  - "*exp*" (Expiration Time)
    - JWTの有効期限。オプション。
    - クレームを処理する際は、*exp*が現在時刻以前であることを確認しなければならない（MUST）。
    - 実装者はクロックスキューを考慮し、わずかな猶予（通常、数分以上となることはない）を与えてもよい（MAY）。
    - *exp*の値はエポック秒（NumericDate）でなければならない（MUST）。

  - "*nbf*" (Not Before)
    - JWTが有効となる日時。オプション。
    - クレームを処理する際は、現在時刻が*nbf*以前であってはならない（MUST）。
    - 実装者はクロックスキューを考慮し、わずかな猶予（通常、数分以上となることはない）を与えてもよい（MAY）。
    - *nbf*の値はエポック秒（NumericDate）でなければならない（MUST）。

  - "*iat*" (Issued At)
    - JWTを発行した時刻。オプション。

  - "*jtl*" (JWT ID)
    - JWTの一意な識別子。オプション。
    - アプリケーションが複数の発行者を利用する場合、異なる発行者による同じ値の衝突を防止しなければならない（MUST）。
    - *jtl*は大文字・小文字を識別する文字列。

## JOSE Header

  - "*typ*" (Type)
    - JWTのメディアタイプ。オプション。
    - このパラメータはJWT利用するアプリケーションがJWTのメディアタイプを判断するために利用することができる。
    - メッセージをコンパクトにするため、メディアタイプに他の ``/`` がない場合、``application /`` を省略することを推奨する（RECOMMENDED）。
      - 受信者は、*typ*が ``/`` を含まない場合、 ``application /`` が*typ*の前に含まれているものとして扱わなければならない（MUST）。
    - 大文字、小文字の区別はないが、従来の実装との互換性を保つため、大文字で表記することを推奨する（RECOMMENDED）。
    - 設定例：
    `
    { "typ" : "JWT" }       # JWT？これいる？
    { "typ" : "JOSE" }      # JWS Compact Serialization
    { "typ" : "JOSE+JSON" } # JWS JSON Serialization
    `

  - "*cty*" (Content Type)
    -  Payloadのコンテントタイプ。オプション。
    - 入れ子となっていないJWTの場合、このヘッダパラメータの利用は推奨されない（NOT RECOMMENDED）。
    - 入れ子となっている場合、このヘッダパラメータが存在していなければならず（MUST）、値は"JWT"でなければならない（MUST）。

  - "*alg*" (Algorithm)
    - JWSの署名に利用される暗号アルゴリズム。必須。
    - *alg*の値は文字列あるいはURI値（StringOrURI）を含む大文字、小文字を区別するASCII文字列。
    - ヘッダパラメータに存在しなければならず、実装によって理解、処理されなければならない（MUST）。
      - 補足：「処理されなければならない」という部分を誤った仕様だと言ってる人もいるようです。
        - [JSON Web Tokenライブラリの危機的な脆弱性](https://www.eisbahn.jp/yoichiro/2015/09/jwt_vulnerability.html)
        - [JOSE（JavaScriptオブジェクトへの署名と暗号化）は、絶対に避けるべき悪い標準規格である](https://postd.cc/jwt-json-web-tokens-is-bad-standard-that-everyone-should-avoid/)
    - 設定例：
    `
    { "alg":"HS256" }
    `

  - "*jku*" (JWK Set URL)
    - JSONでエンコードされた公開鍵セットのリソース。オプション。
    - 設定例：
    `
    { "jku" : "https://example.com/jwks.json" }
    `

  - "*jwk*" (JSON Web Key)
    - JWSにデジタル署名を行った秘密鍵に対応する公開鍵。オプション。

  - "*kid*" (Key ID)
    - 鍵を特定するためのヒントとなる識別子。オプション。
    - JWKを利用する場合、*kid* の値はJWK *kid* の値と一致するように指定する。

  - "*x5u*" (X.509 URL)
    - JWSにデジタル署名を行った秘密鍵に対応するX.509公開鍵証明書もしくは証明書チェーンのリソース。オプション。

  - "*x5c*" (X.509 Certificate Chain)
    - JWSにデジタル署名を行った秘密鍵に対応するX.509公開鍵証明書もしくは証明書チェーン。オプション。
    - 証明書はBase64エンコードしたDESをJSON配列として表現する。
    - 設定例：
    `
    { "x5c" : ["MIIE3jCCA8agAwIBA...==",
               "MIIE+zCCBGSgAwIBA...==",
               "MIIC5zCCAlACAQEwD...Pd"] }
    `

  - "*x5t*" (X.509 Certificate Thumbprint)
    - JWSにデジタル署名を行った秘密鍵に対応するX.509公開鍵証明書。オプション。
    - DES形式で表現し、ダイジェストをBase64uriエンコードした文字列を指定する。

  - "*crit*" (Critical)
    - 必須パラメータ指定。

  - "*enc*" (Encryption Algorithm)
  - "*zip*" (Compression Algorithm)

## Appendix

### JWS Exapmle HMAC-SHA2 Integrity Protection

#### Input
  - JOSE Header
```
   {
     "alg": "HS256",
     "kid": "018c0ae5-4d9b-471b-bfd6-eef314bc7037"
   }
```
  - Payload content
```
   "It\xe2\x80\x99s a dangerous business, Frodo, going out your "
   "door. You step onto the road, and if you don't keep your feet, "
   "there\xe2\x80\x99s no knowing where you might be swept off "
   "to."
```
  - HMAC symmetric key
```
   {
     "kty": "oct",
     "kid": "018c0ae5-4d9b-471b-bfd6-eef314bc7037",
     "use": "sig",
     "alg": "HS256",
     "k": "hJtXIZ2uSN5kbQfbtTNWbpdmhkV8FJG-Onbc6mxCcYg"
   }
```

#### Input(encoded Base64uri)
  - JOSE Header
```
   eyJhbGciOiJIUzI1NiIsImtpZCI6IjAxOGMwYWU1LTRkOWItNDcxYi1iZmQ2LW
   VlZjMxNGJjNzAzNyJ9
```
  - Payload content
```
   SXTigJlzIGEgZGFuZ2Vyb3VzIGJ1c2luZXNzLCBGcm9kbywgZ29pbmcgb3V0IH
   lvdXIgZG9vci4gWW91IHN0ZXAgb250byB0aGUgcm9hZCwgYW5kIGlmIHlvdSBk
   b24ndCBrZWVwIHlvdXIgZmVldCwgdGhlcmXigJlzIG5vIGtub3dpbmcgd2hlcm
   UgeW91IG1pZ2h0IGJlIHN3ZXB0IG9mZiB0by4
```
  - JWS Signature( produces the JWS Signature {JOSE Header + '.' + Payload content} )
```
    s0h6KThzkfBBBkLspW1h84VsJZFTsPPqMDA7g1Md7p0
```

#### Output
```
   eyJhbGciOiJIUzI1NiIsImtpZCI6IjAxOGMwYWU1LTRkOWItNDcxYi1iZmQ2LW
   VlZjMxNGJjNzAzNyJ9
   .
   SXTigJlzIGEgZGFuZ2Vyb3VzIGJ1c2luZXNzLCBGcm9kbywgZ29pbmcgb3V0IH
   lvdXIgZG9vci4gWW91IHN0ZXAgb250byB0aGUgcm9hZCwgYW5kIGlmIHlvdSBk
   b24ndCBrZWVwIHlvdXIgZmVldCwgdGhlcmXigJlzIG5vIGtub3dpbmcgd2hlcm
   UgeW91IG1pZ2h0IGJlIHN3ZXB0IG9mZiB0by4
   .
   s0h6KThzkfBBBkLspW1h84VsJZFTsPPqMDA7g1Md7p0
```