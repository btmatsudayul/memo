# RFC 7523：JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants 和訳
<!-- TOC -->

- [Document](#document)
- [Contents](#contents)
    - [Abstract](#abstract)
    - [1. Introduction](#1.-introduction)
    - [2. HTTP Parameter Bindings for Transporting Assertions](#2.-http-parameter-bindings-for-transporting-assertions)
        - [2.1. Using JWTs as Authorization Grants](#2.1.-using-jwts-as-authorization-grants)
        - [2.2. Using JWTs for Client Authentication](#2.2.-using-jwts-for-client-authentication)
    - [3. JWT Format and Processing Requirements](#3.-jwt-format-and-processing-requirements)
        - [3.1. Authorization Grant Processing](#3.1.-authorization-grant-processing)
        - [3.2. Client Authentication Processing](#3.2.-client-authentication-processing)
    - [4. Authorization Grant Example](#4.-authorization-grant-example)
    - [5. Interoperability Considerations](#5.-interoperability-considerations)
    - [6. Security Considerations](#6.-security-considerations)
    - [7. Privacy Considerations](#7.-privacy-considerations)

<!-- /TOC -->
## Document

[RFC 7523：JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants](https://tools.ietf.org/html/rfc7523)

## Contents

以下、関係ありそうな箇所のみ直訳。

### Abstract

  この仕様では、クライアント認証だけでなくOAuth 2.0アクセストークンをリクエストする手段としてJSON Webトークン（JWT）ベアラトークンを使用することを定義しています。

### 1.Introduction

  [RFC](https://tools.ietf.org/html/rfc7523#section-1)

  JSON Webトークン（JWT）[JWT]は、IDとセキュリティ情報をセキュリティドメイン間で共有できるJSONベースの[RFC7159]セキュリティトークンエンコーディングです。セキュリティトークンは、一般に、アイデンティティプロバイダによって発行され、セキュリティ関連の目的でトークンの主題を識別するためにその内容に依存する依拠当事者によって消費される。

  OAuth 2.0認証フレームワーク[RFC6749]は、アクセストークンを使用して認証されたHTTP要求をリソースに作成する方法を提供します。アクセストークンは、リソース所有者の（時に暗示的な）承認を伴う承認サーバ（AS）によって第三者のクライアントに発行される。 OAuthでは、認可許可は、リソース所有者の認可を表す中間資格を記述するために使用される抽象的な用語です。クライアントがアクセストークンを取得するために、認可許可が使用されます。さまざまな種類のクライアントとユーザーエクスペリエンスをサポートするために、いくつかの認可許可タイプが定義されています。 OAuthでは、追加のクライアントをサポートするため、またはOAuthと他の信頼フレームワークとの間にブリッジを提供するために、新しい拡張グラントタイプを定義することもできます。最後に、OAuthは、認証サーバと対話する際にクライアントが使用する追加の認証メカニズムの定義を可能にします。

  「Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants」[RFC7521]は、OAuth 2.0の抽象的な拡張であり、OAuth 2.0を使用してクライアント資格証明および/または認可付与としてアサーション（a.k.a.セキュリティトークン）を使用するための一般的なフレームワークを提供します。この仕様は、OAuthアサーションフレームワーク[RFC7521]に、JWTベアラトークンを使用してOAuth 2.0アクセストークンを要求し、クライアントクレデンシャルとして使用する拡張認可タイプを定義するようにプロファイルします。この仕様で定義されたJWTのフォーマットと処理ルールは、密接に関連する仕様「OAuth 2.0クライアント認証と認可グラントのセキュリティアサーションマークアップ言語（SAML）2.0プロファイル」[RFC7522]と意図的に類似していますが同一ではありません。 JWTの構造とセマンティクスがSAMLアサーションと異なるところで、違いが生じます。たとえば、JWTは、SAMLアサーションの<SubjectConfirmation>または<AuthnStatement>要素に直接相当するものはありません。

  このドキュメントでは、クライアントがJWTのセマンティクスで表現された既存の信頼関係を、認証サーバーでの直接のユーザー承認手順なしに利用したい場合に、JWTベアラトークンを使用してアクセストークンを要求する方法を定義します。また、JWTをクライアント認証メカニズムとしてどのように使用できるかを定義します。クライアント認証にセキュリティトークンを使用することは、認証トークンとしてセキュリティトークンを使用することとは直交しており、分離することができます。これらは組み合わせて使用​​することも、別々に使用することもできます。 JWTを使用するクライアント認証は、クライアントがトークンエンドポイントに対して認証するための代替方法に過ぎず、完全で意味のあるプロトコル要求を形成するために、いくつかの許可タイプとともに使用する必要があります。 JWT認可付与は、クライアントの認証または識別の有無にかかわらず使用できます。サポートされているクライアント認証のタイプと同様に、JWT認可許可とともにクライアント認証が必要かどうかは、認可サーバーの裁量によるポリシー決定です。

  JWTをクライアントが認可サーバーと交換する前またはクライアント認証に使用する前に、JWTを取得するプロセスは範囲外です。

### 2. HTTP Parameter Bindings for Transporting Assertions

  [RFC](https://tools.ietf.org/html/rfc7523#section-2)

  OAuthアサーションフレームワーク[RFC7521]は、トークンエンドポイントとの対話中にアサーション（a.k.a.セキュリティトークン）を転送するための一般的なHTTPパラメータを定義します。 このセクションでは、JWTベアラトークンで使用するためのパラメータの特定のパラメータと処理を定義します。

#### 2.1. Using JWTs as Authorization Grants

  ベアラJWTを認可許可として使用するために、クライアントはOAuthアサーションフレームワーク[RFC7521]のセクション4で定義されているようなアクセストークン要求を以下の特定のパラメータ値とエンコーディングで使用します。

  - `grant_type`の値は`urn：ietf：params：oauth：grant-type：jwt-bearer`です。

  - `assertion`パラメータの値は単一のJWTを含まなければなりません。

  要求されたスコープを示すために、OAuthアサーションフレームワーク[RFC7521]で定義されているように、 `scope`パラメータを使用することができます。

  OAuth 2.0 [RFC6749]のセクション3.2.1で説明されているように、クライアントの認証はオプションです。したがって、パラメータに依存するクライアント認証の形式が使用されている場合にのみ `client_id`が必要です。

  次の例は、認可付与としてJWTを使用したアクセストークン要求を示しています（表示用に余分な改行があります）。

```
     POST /token.oauth2 HTTP/1.1
     Host: as.example.com
     Content-Type: application/x-www-form-urlencoded

     grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
     &assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
     eyJpc3Mi[...omitted for brevity...].
     J9l-ZhwP[...omitted for brevity...]
```

#### 2.2. Using JWTs for Client Authentication

  クライアント認証にJWTベアラトークンを使用するには、クライアントは次のパラメータ値とエンコーディングを使用します。

  - `client_assertion_type`の値は` urn：ietf：params：oauth：client-assertion-type：jwt-bearer`です。

  - `client_assertion`パラメータの値は単一のJWTを含みます。 それは複数のJWTを含んではいけません。

  次の例は、アクセストークン要求で認証コード付与の提示中にJWTを使用するクライアント認証を示しています（表示目的でのみ追加の改行があります）。

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

### 3. JWT Format and Processing Requirements

  [RFC](https://tools.ietf.org/html/rfc7523#section-3)

  OAuth 2.0 [RFC6749]に記述されているようにアクセストークンレスポンスを発行するか、またはクライアント認証のためにJWTに依存するために、認証サーバは以下の基準に従ってJWTを検証しなければならない（MUST）。 追加の制限およびポリシーの適用は、許可サーバーの裁量に委ねられます。

  OAuth 2.0 [RFC6749]に記述されているようにアクセストークンレスポンスを発行するか、またはクライアント認証のためにJWTに依存するために、認証サーバは以下の基準に従ってJWTを検証しなければならない（MUST）。 追加の制限およびポリシーの適用は、許可サーバーの裁量に委ねられます。

  1. JWTは、JWTを発行したエンティティの一意の識別子を含む `iss`（発行者）クレームを含まなければなりません。 そうでない場合に指定するアプリケーションプロファイルがない場合、準拠するアプリケーションは、RFC 3986 [RFC3986]のセクション6.2.1で定義されている単純な文字列比較メソッドを使用して発行者の値を比較しなければならない。

  2. JWTには、JWTの主体であるプリンシパルを識別するサブ（主題）クレームが含まれていなければならない。 2つのケースを区別する必要があります。

      - 認可許可の場合、主体は通常、アクセストークンが要求されている認可されたアクセサー（すなわち、リソース所有者または許可された代理人）を識別するが、場合によっては、匿名の識別子または匿名を示す他の値ユーザー。

      - クライアント認証のためには、サブジェクトはOAuthクライアントの `client_id 'でなければなりません。

  3. JWTには、承認サーバーを意図したオーディエンスとして識別する値を含む「aud」（オーディエンス）クレームが含まれていなければならない。認可サーバーのトークンエンドポイントURLは、JWTの意図されたオーディエンスとして認可サーバーを識別するための `aud`要素の値として使用することができます（MAY）。認可サーバーは、意図したオーディエンスとして独自のIDを含まないJWTを拒否しなければならない（MUST）。そうでない場合に指定するアプリケーションプロファイルがない場合、準拠するアプリケーションは、RFC 3986 [RFC3986]のセクション6.2.1で定義されている単純な文字列比較メソッドを使用してオーディエンス値を比較しなければならない（MUST）。セクション5で説明したように、特定の認可サーバーのオーディエンスとして使用される正確な文字列は、認可サーバーとJWTの発行者によって帯域外で構成されなければなりません。

  4. JWTには、JWTを使用できる期間を制限する `exp`（有効期限）クレームが含まれていなければならない（MUST）。 認可サーバーは、システム間で許容されるクロックスキューを条件として、経過した有効期限を持つJWTを拒否しなければならない（MUST）。 認可サーバは、将来的に不合理に遠い `exp`クレーム値を持つJWTを拒否するかもしれないことに注意してください。

  5. JWTは、処理のためにトークンを受け入れてはならない前の時間を識別する `nbf`（not before）クレームを含むことができる（MAY）。

  6. JWTには、JWTが発行された時刻を識別する `iat`（発行済み）クレームが含まれています（MAY）。 認可サーバーは、過去に不合理に遠い `iat`クレーム値を持つJWTを拒否するかもしれないことに注意してください。

  7. JWTはトークンの一意の識別子を提供する `jti`（JWT ID）クレームを含むかもしれません。 承認サーバは、適用可能な `exp`インスタントに基づいてJWTが有効と見なされる時間の間、使用された` jti`値のセットを維持することによって、JWTが再生されないことを保証してもよい（MAY）。

  8. JWTには他の主張が含まれることがあります。

  9. JWTは、デジタル署名されているか、発行者によって適用されたメッセージ認証コード（MAC）を持っていなければならない（MUST）。 認可サーバーは、無効な署名またはMACを持つJWTを拒絶しなければならない（MUST）。

  10. 承認サーバは、JSON Webトークン（JWT）[JWT]ごとに他の点で有効でないJWTを拒絶しなければならない（MUST）。

#### 3.1. Authorization Grant Processing

  JWT認可付与は、クライアントの認証または識別の有無にかかわらず使用できます。 サポートされているクライアント認証のタイプと同様に、JWT認可許可とともにクライアント認証が必要かどうかは、認可サーバーの裁量によるポリシー決定です。 ただし、要求にクライアント信任状が存在する場合、許可サーバはそれを検証しなければならない（MUST）。

  JWTが有効でない場合、または現在の時刻がトークンの有効時間枠内にない場合、認証サーバーはOAuth 2.0 [RFC6749]で定義されているエラー応答を作成します。 `error`パラメータの値は `invalid_grant`エラーコードでなければならない（MUST）。 認可サーバーは、 `error_description`または `error_uri`パラメータを使用してJWTが無効とみなされた理由に関する追加情報を含めることができる（MAY）。

  例えば：

```
     HTTP/1.1 400 Bad Request
     Content-Type: application/json
     Cache-Control: no-store

     {
      "error":"invalid_grant",
      "error_description":"Audience validation failed"
     }
```

#### 3.2. Client Authentication Processing

  クライアントJWTが有効でない場合、認証サーバーはOAuth 2.0 [RFC6749]で定義されているエラー応答を構成します。 `error`パラメータの値は`invalid_client`エラーコードでなければなりません。 承認サーバは、 `error_description`または`error_uri`パラメータを使用してJWTが無効であると考えられた理由に関する追加情報を含めることができる（MAY）。


### 4. Authorization Grant Example

  [RFC](https://tools.ietf.org/html/rfc7523#section-4)

  次の例は、適合するJWTとアクセストークン要求の外観を示しています。

  この例では、 `https：// jwt-idp.example.com`というシステムエンティティによって発行され、署名されたJWTを示しています。 JWTの件名は電子メールアドレスで「mike @ example.com」と識別されます。 JWTの対象読者は、 `https：// jwt-rp.example.net`です。これは、認可サーバーがそれ自身を識別する識別子です。 JWTは、アクセス・トークン要求の一部として、https://authz.example.net/ token.oauth2にある許可サーバーのトークン・エンドポイントに送信されます。

  以下に、JWTのJWTクレームセットを生成するためにエンコードできるJSONオブジェクトの例を示します。

```
     {"iss":"https://jwt-idp.example.com",
      "sub":"mailto:mike@example.com",
      "aud":"https://jwt-rp.example.net",
      "nbf":1300815780,
      "exp":1300819380,
      "http://claims.example.com/member":true}
```

  JWTのヘッダーとして使用される次のJSONオブジェクトの例では、JWTが、`kid`値`16`によって識別される鍵を使用してElliptic Curve Digital Signature Algorithm（ECDSA）P-256 SHA-256で署名されていることを宣言しています。

```
     {"alg":"ES256","kid":"16"}
```

  たとえば、アクセストークン要求の一部として前の例に示されたクレームとヘッダーをJWTに提示するには、クライアントは次のHTTPS要求を行います（表示用に余分な改行が必要）。

```
     POST /token.oauth2 HTTP/1.1
     Host: authz.example.net
     Content-Type: application/x-www-form-urlencoded

     grant_type=urn%3Aietf%3Aparams%3Aoauth%3Agrant-type%3Ajwt-bearer
     &assertion=eyJhbGciOiJFUzI1NiIsImtpZCI6IjE2In0.
     eyJpc3Mi[...omitted for brevity...].
     J9l-ZhwP[...omitted for brevity...]
```

### 5. Interoperability Considerations

  [RFC](https://tools.ietf.org/html/rfc7523#section-5)

  このプロファイルの相互運用可能な展開を実現するには、識別子、キー、エンドポイントに関するシステムエンティティ間の合意が必要です。合意を必要とする具体的な事項は、発行者とオーディエンスの識別子の値、トークンエンドポイントの位置、JWT上のデジタル署名またはMACの適用と検証に使用する鍵、JWTの1回限りの使用制限、最大JWTの有効期間、およびJWTの特定の件名と請求要件が含まれます。そのような情報の交換は、この仕様の範囲外であることは明らかです。場合によっては、これらの値を制約または処方するか、またはそれらの値を交換する方法を指定する追加のプロファイルが作成されることがあります。このようなプロファイルの例には、OAuth 2.0動的クライアント登録コアプロトコル[OAUTH-DYN-REG]、OpenID Connect動的クライアント登録1.0 [OpenID.Registration]、およびOpenID Connect Discovery 1.0 [OpenID.Discovery]が含まれます。

  [JWA]の「RS256」アルゴリズムは、このプロファイルのJSON Web Signatureアルゴリズムを実装するために必須です。

### 6. Security Considerations

  [RFC](https://tools.ietf.org/html/rfc7523#section-6)

  以下の仕様に記述されているセキュリティの考慮事項は、すべてこのドキュメントに適用されます：「Assertion Framework for OAuth 2.0 Client Authentication and Authorization Grants」[RFC7521]、「The OAuth 2.0 Authorization Framework」： [RFC6749]、「JSON Web Token」、[JWT]。

  この仕様では、認可許可またはクライアント認証のいずれかに対するJWTの使用に対するリプレイ保護は要求されていません。 これは任意の機能であり、実装は独自の裁量で使用できます。

### 7. Privacy Considerations

  [RFC](https://tools.ietf.org/html/rfc7523#section-7)

  JWTにはプライバシーに敏感な情報が含まれており、そのような情報が意図しない者に漏洩するのを防ぐため、TLS（Transport Layer Security）などの暗号化されたチャネルを介してのみ送信する必要があります。 特定の情報がクライアントに開示されないようにする場合は、JWTを認可サーバーに暗号化する必要があります。

  配備は、交換を完了するために必要な情報の最小量を決定し、そのような主張のみをJWTに含める必要があります。 場合によっては、OAuthアサーションフレームワーク[RFC7521]の6.3.1節で説明されているように、 `sub`（主語）要求は匿名または匿名のユーザを表す値とすることができる。
