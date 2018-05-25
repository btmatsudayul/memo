# OAuth 2.0 Multiple Response Type Encoding Practices 調査メモ

## Overview

  - `response_type`
    - 認可フローを指定するためのパラメータ。
    - OpenID Connect（OAuth 2.0）では複数の定義済みレスポンスタイプをスペースで連結した Multiple-Valued Response Types という新たなレスポンスタイプが定義されている。
  - `response_mode`
    - Authorization EndpointからAuthorization Responseパラメーターを返却する際のほう穂を指定するためのパラメータ。
    - 各レスポンスタイプには、デフォルトの`response_mode`値が定義されている。

## Response Mode

|Mode      |説明 |
|:---------|:-------------------------|
|`query`   |クライアントにリダイレクトされる際、Authorization Responseは`redirect_uri`のクエリに設定される |
|`fragment`|クライアントにリダイレクトされる際、Authorization Responseは`redirect_uri`のフラグメント設定される  |

 - **memo** :
    - http://host:port?<クエリ>#<フラグメント> のこと。
    - `response_type=code`（＝OAuth 2.0認可コードグラント）の場合、レスポンスタイプのデフォルトは`query`
    - `response_type=token`（＝OAuth 2.0インプリシットグラント）の場合、レスポンスタイプのデフォルトは`fragment`

## Response Type

特殊なResponse Typeとして、以下が定義されている。

  - `none`
    - レスポンスとしてアクセス資格を返却しない（必要としない）フロー。アプリの初回インストール時のリソースアクセスなどに利用することができる。

各Response Typeにおいて返却されるパラメータは以下の通り。

|Type                      |Access Token |Token Type |Authz Code |ID Token |デフォルトresponse_mode |
|:-------------------------|:-----------:|:---------:|:---------:|:-------:|:--------------------|
|(1) `code`                |   |   | X |   |query    |
|(2) `token`               | X | X |   |   |fragment |
|(3) `code token`          | X | X | X |   |fragment |
|(4) `code id_token`       |   |   | X | X |fragment |
|(5) `id_token token`      | X | X |   | X |fragment |
|(6) `code id_token token` | X | X | X | X |fragment |


  - (5)の例

```
  GET /authorize?
    response_type=id_token%20token
    &client_id=s6BhdRkqt3
    &redirect_uri=https%3A%2F%2Fclient.example.org%2Fcb
    &state=af0ifjsldkj HTTP/1.1
  Host: server.example.com
```
```
  HTTP/1.1 302 Found
  Location: https://client.example.org/cb#
  access_token=SlAV32hkKG
  &token_type=bearer
  &id_token=eyJ0 ... NiJ9.eyJ1c ... I6IjIifX0.DeWt4Qu ... ZXso
  &expires_in=3600
  &state=af0ifjsldkj

```