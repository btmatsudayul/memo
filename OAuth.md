
- [OAuth 2.0 について](#oauth-20-%E3%81%AB%E3%81%A4%E3%81%84%E3%81%A6)
  - [概要](#%E6%A6%82%E8%A6%81)
  - [関連仕様](#%E9%96%A2%E9%80%A3%E4%BB%95%E6%A7%98)
  - [ロール](#%E3%83%AD%E3%83%BC%E3%83%AB)
  - [フロー](#%E3%83%95%E3%83%AD%E3%83%BC)
    - [プロトコルフロー](#%E3%83%97%E3%83%AD%E3%83%88%E3%82%B3%E3%83%AB%E3%83%95%E3%83%AD%E3%83%BC)
    - [認可グラント](#%E8%AA%8D%E5%8F%AF%E3%82%B0%E3%83%A9%E3%83%B3%E3%83%88)
    - [認可コードフロー詳細](#%E8%AA%8D%E5%8F%AF%E3%82%B3%E3%83%BC%E3%83%89%E3%83%95%E3%83%AD%E3%83%BC%E8%A9%B3%E7%B4%B0)
  - [OAuth 2.0の代表的な攻撃と対策](#oauth-20%E3%81%AE%E4%BB%A3%E8%A1%A8%E7%9A%84%E3%81%AA%E6%94%BB%E6%92%83%E3%81%A8%E5%AF%BE%E7%AD%96)

# OAuth 2.0 について

## 概要

OAuth 2.0とは、HTTPサービスを利用するサードパーティアプリケーションに対して、限定的なアクセスを可能とするための「認可」フレームワークのことである。

従来のサーバ・クライアント型の認証モデルでは、サードパーティアプリケーションがエンドユーザのリソース（プロフィール等）にアクセスする際、エンドユーザは自身のクレデンシャル（例：ID、パスワード）をサードパーティに対して共有する必要があった。これには、以下のような問題があった。

- サードパーティアプリケーションにてエンドユーザのアカウント情報を管理する必要がある。
- エンドユーザは、サードパーティアプリケーションに許可するアクセス可能範囲（アクセス対象のデータ、およびそれに伴う参照、更新などの操作）およびアクセス可能期間を制限することができない。
- サードパーティアプリケーションにおける情報漏えいは、エンドユーザのID、パスワードの漏洩につながる。

これに対し、OAuth 2.0では、サードパーティに対してエンドユーザのクレデンシャルを共有することなく、リソースへのアクセス権を委譲（認可）する。
OAuth 2.0フローの中で、エンドユーザにより認可され発行された情報はアクセストークンと呼ばれ、サードパーティアプリケーションはこのアクセストークンを用いることで、エンドユーザのリソースへ（許可された範囲での）アクセスが可能となる。

> **「認証」と「認可」の違いについて**
> 
> - 認証（`Authentication`）
>   - ユーザが、そのユーザ本人であることを証明すること。
> - 認可（`Authorization`）
>   - （認証済みの）ユーザに対して権限を委譲すること。
> 
> なお、OAuth 2.0は「認可」のフレームワークであり、認可処理の前段として必要になる、エンドユーザの「認証」の具体的な方法については特に定めていない。
> OAuth 2.0を利用する際は、[RFC 6819 OAuth 2.0 Threat Model and Security Considerations](https://tools.ietf.org/html/rfc6819) に定められているセキュリティ考慮事項とあわせて、適切な認証方式を検討する必要がある。

## 関連仕様

OAuth 2.0は様々な関連仕様により構成されており、これらを組み合わせることで認可を実現する。
OAuth 2.0の主な関連仕様を以下に示す。

OAuth 2.0についてより詳細を把握したい場合は、下記関連仕様の重要度が高い（私見です）順に参照することを推奨する。

**OAuth 2.0関連仕様**

| 関連仕様  | 重要度 | 説明            |
| :-------- | :----: | :-------------- |
| [RFC 6749 The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749) [(日本語訳)](http://openid-foundation-japan.github.io/rfc6749.ja.html)| 高 | OAuth 2.0のコアとなるプロトコル。アクセストークン発行までのフローを規定。
| [RFC 6750 The OAuth 2.0 Authorization Framework: Bearer Token Usage](https://tools.ietf.org/html/rfc6750) [(日本語訳)](http://openid-foundation-japan.github.io/rfc6750.ja.html)| 高 | アクセストークンを用いてリソースへのアクセスを行う際のフローを規定。
| [RFC 6819 OAuth 2.0 Threat Model and Security Considerations](https://tools.ietf.org/html/rfc6819) [(日本語訳)](http://openid-foundation-japan.github.io/rfc6819.ja.html)| 高 | OAuth 2.0の利用に際して注意すべきセキュリティ上の特徴・注意点を規定。
| [RFC 7636 Proof Key for Code Exchange by OAuth Public Clients](https://tools.ietf.org/html/rfc7636) | 中 | PKCE（ぴくしー）と呼ぶ。後述する、認可コードフローにおける認可コードの横取り攻撃への対策を規定。
| [RFC 7662 OAuth 2.0 Token Introspection](https://tools.ietf.org/html/rfc7662) | 中 | 後述する、リソースサーバがアクセストークンの妥当性を検証するために、認可サーバへ問い合わせる際のフローを規定。
| [RFC 7519 JSON Web Token (JWT)](https://tools.ietf.org/html/rfc7519) | 中 | JWT（じょっと）と呼ぶ。JSONオブジェクトを署名または暗号化しURL-safeな文字列とするための仕様を規定。OAuth 2.0ではアクセストークンをJWTとして発行することもある。また、後述するOpenID Connectでは様々な箇所でJWTを利用している。<br> **TODO：OpenID Connectとあわせて後日概要執筆予定**
| [RFC 8252 OAuth 2.0 for Native Apps](https://tools.ietf.org/html/rfc8252) | 低 | OAuth 2.0をiOS、Androidなどのネイティブアプリケーションで利用する際の、現時点でのベストプラクティスを規定。本仕様で紹介しているフローではPKCEを適用している。

また参考情報として、OAuth 2.0以外の認可の関連技術について以下に示す。

**認可の関連技術**

| 関連技術            | 説明            |
| :------------------ | :-------------- |
| OAuth 1.0           | [RFC 5849](https://tools.ietf.org/html/rfc5849) にて策定されているOAuth 2.0の元となったプロトコル。OAuth 2.0策定に伴い廃止となった。
| OpenID Connect      | OAuth 2.0をベースとした拡張されたプロトコル。OAuth 2.0では定められていなかった、エンドユーザのIDやプロフィール情報を連携（ID連携）する方法が追加されている他、よりセキュアな通信を実現するためのパラメータ等の追加、その他関連技をオプションとして規定。<br> **TODO：後日概要執筆予定** 
| Financial-grade API | OpenID Financial API WGが策定を進めている、金融分野においてAPIを提供する際の標準仕様群。JSONデータスキーマ、セキュリティおよびプライバシに関する推奨事項とプロトコルを規定。フローはOpenID Connectをベースにしている。

> **Financial-grade API（FAPI）について**
> 
> FAPIは金融分野のAPI仕様であるが、他分野であってもセキュリティへの推奨事項は参考になるため、参照することを推奨する。
> なお、FAPIが定める参照系API（例：残高照会API）および更新系API（例：口座振込API）の要件を定めている [Part1](http://openid.net/specs/openid-financial-api-part-1.html)、[Part2](http://openid.net/specs/openid-financial-api-part-2.html) は 2018/8/1現在、Implementer's Draftであるため、今後策定が進むに伴い変更が入る可能性がある。
> また、Part2で紹介されている [OAuth 2.0 Token Binding](https://tools.ietf.org/html/draft-ietf-oauth-token-binding-07) は現在策定中であるが、TLSレイヤに介入する処理が必要となる。そのため、OAuth 2.0 Token Bindingが標準化されてない現時点では実現可否はTSLの実装に依存し、Part2の要件を全てクリアするのは非常に困難である。

## ロール

OAuth 2.0では、以下の4つのロールを定義している。

**OAuth 2.0のロール**

| ロール                                  | 説明            |
| :-------------------------------------- | :-------------- |
| リソースオーナ<br>（`resource owner`）  | 保護されたリソースへのアクセスを許可するエンティティ。リソースオーナが人間の場合、エンドユーザ。
| リソースサーバ<br>（`resource server`） | 保護されたリソースをホストし、アクセストークンを用いたリソースへのリクエストに対して応答するサーバ。
| クライアント<br>（`client`）            | リソースオーナの認可を得て、リソースオーナの代理として保護されたリソースに対するリクエストを行うアプリケーション。
| 認可サーバ<br>（`authorization server`）| リソースオーナーの認証とリソースオーナーからの認可取得が成功した後、アクセストークンをクライアントに発行するサーバ。

また、クライアントは、クライアントのクレデンシャルの機密性を維持できるかどうかで以下の2つに分類される。

**クライアントタイプ**

| クライアントタイプ                      | 説明            |
| :-------------------------------------- | :-------------- |
| コンフィデンシャル<br>（`confidential`）| クレデンシャルへのアクセスが制限されたWebサーバ上に実装されたクライアントなど。
| パブリック<br>（`public`）              | iOSやAndroidなどにインストールされたネイティブアプリケーションやブラウザベース（JavaScript等）のアプリケーショなど、（改ざんなどにより）機密性を維持することができず、かつ他の手段を用いたセキュアなクライアント認証が利用できないクライアント。

> **パブリッククライアントの認証について**
> 
> OAuth 2.0におけるフローでは、クライアントがパブリッククライアントである場合、クライアント認証をクライアントを識別する目的で利用してはならず、また認証したパブリッククライアントを **信頼してはならない** としている。
> そのため、後述するインプリシットフローはブラウザベースのクライアントに最適化されたOAuth 2.0フローであるが、クライアント認証を行っておらず、セキュリティリスクの高い（ユーザビリティとのトレードオフにより、ある程度割り切った）フローとなっている。

## フロー

ここでは、OAuth 2.0の概念的なフローであるプロトコルフローと、OAuth 2.0にて定められている4つの具体的なフローの概要について説明する。
またあわせて、フローの中でも最も利用することの多い認可コードフローの詳細、ならびに利用する際に注意すべき点について補足する。

### プロトコルフロー

OAuth 2.0のプロトコルフローを以下に示す。

```
  +--------+                               +---------------+
  |        |--(A)- Authorization Request ->|   Resource    |
  |        |                               |     Owner     |
  |        |<-(B)-- Authorization Grant ---|               |
  |        |                               +---------------+
  |        |
  |        |                               +---------------+
  |        |--(C)-- Authorization Grant -->| Authorization |
  | Client |                               |     Server    |
  |        |<-(D)----- Access Token -------|               |
  |        |                               +---------------+
  |        |
  |        |                               +---------------+
  |        |--(E)----- Access Token ------>|    Resource   |
  |        |                               |     Server    |
  |        |<-(F)--- Protected Resource ---|               |
  +--------+                               +---------------+
```

* (A) クライアントはリソースオーナーに対して認可を要求する。後述する認可コードフロー、インプリシットフローでは認可サーバが仲介を行う。
* (B) クライアントは、リソースオーナーの認可を表す情報を受け取る。この値は、後述する認可グラントタイプ（認可を得るための方式）により異なる。
* (C) クライアントは認可サーバーに対して自身を認証し、認可グラントを提示することで、アクセストークンを要求する。
* (D) 認可サーバーはクライアントを認証し、認可グラントの正当性を確認する。そして認可グラントが正当であれば、アクセストークンを発行する。
* (E) クライアントはリソースサーバーの保護されたリソースへリクエストを行い、発行されたアクセストークンにより認証を行う。
* (F) リソースサーバーはアクセストークンの正当性を確認し、正当であればリクエストを受け入れる。

### 認可グラント

認可グラントとは、リソースオーナによる認可を示してクライアントがアクセストークンを取得する方式のことを指す。
OAuth 2.0では、4つのグラントタイプを定義している。

**認可グラント**

| グラントタイプ                           | 説明            |
| :--------------------------------------- | :-------------- |
| 認可コード                               | クライアントが、認可サーバが発行した認可コードを用いて、アクセストークンを発行するフロー。コンフィデンシャルクライアントのアプリケーションに最適化されている。<br>このフローでは、クライアントの認証を行う、アクセストークンをユーザエージェントを経由せずに直接クライアントに発行するなど、いくつかのセキュリティ的なメリットが存在する。
| インプリシット                           | 認可コードグラントを単純化したものでありブラウザベースのアプリケーションに最適化されたフロー。<br>認可コードを取得する代わりに直接アクセストークンを発行するなど、認可コードフローと比較して応答性・効率性を高めているという特徴があるが、クライアントの認証を行わないなど、ユーザビリティとのトレードオフでセキュリティリスクが高い。
| リソースオーナーパスワードクレデンシャル | クライアントが、リソースオーナのクレデンシャルを用いて、アクセストークンを発行するフロー。<br>クライアントが非常に高い特権を持つなどのリソースオーナとクライアントの間に高い信頼があり、かつ他のグラントタイプを選択できないケースのみに利用すべき。
| クライアントクレデンシャル               | クライアントが、自身のクレデンシャルを用いて、アクセストークンを発行するフロー。<br>認可の対象がクライアントの管理下にあるリソース（＝クライアントがリソースオーナとなる）、または予め認可サーバと調整済みのリソースに制限されているケースに利用する。

> **どのグラントタイプを利用すべきか**
> 
> 認可コードフロー以外は、クライアントの利用条件・制約やセキュリティリスクが存在する。そのため、ブラウザベースのアプリケーションでなければ、認可コードフローの利用を検討すること。
> 
> なお、[RFC 6749 The OAuth 2.0 Authorization Framework](https://tools.ietf.org/html/rfc6749) ではネイティブアプリケーションは認可コードフロー、インプリシットフローのどちらでもよいこととなっているが、後続のRFCである [RFC 8252 OAuth 2.0 for Native Apps](https://tools.ietf.org/html/rfc8252) では認可コードフローを前提としている。また、FAPIではネイティブアプリケーションは OAuth 2.0 for Native Apps の対応が必須となっているため、これから検討するのであれば、ネイティブアプリケーションであっても認可コードフローの利用を推奨する。

**TODO：後日更新予定**

### 認可コードフロー詳細

**TODO：後日更新予定**

## OAuth 2.0の代表的な攻撃と対策

**TODO：後日更新予定**

