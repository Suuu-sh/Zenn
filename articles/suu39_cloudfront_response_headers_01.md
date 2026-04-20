---
title: 'CloudFrontのResponse HeadersとCORSを簡単に整理する'
emoji: '☁️'
type: 'tech'
topics: ['aws', 'cloudfront', 'cors', 'http']
published: false
---

# はじめに

CloudFront の CORS 設定は、
`Access-Control-Allow-*` が多くて分かりにくいです。

自分も、実際に CORS エラーが出て
CloudFront の設定を変更する機会がありました。
そのときに整理した内容を、この記事にまとめています。

この記事では、

- Response Headers Policy は何をするのか
- `Access-Control-Allow-*` は何を意味するのか
- どこでハマりやすいのか

だけを、できるだけ簡単に整理します。

---

# 先に結論

CloudFront の **Response Headers Policy** は、
**CloudFront がブラウザに返すレスポンスヘッダーを調整する仕組み** です。

つまり、CloudFront 側で

- ヘッダーを追加する
- ヘッダーを上書きする
- ヘッダーを削除する

ができます。

CORS では、この仕組みで `Access-Control-Allow-*` を返します。

---

# Response Headers Policy の役割

CloudFront にはいろいろな policy がありますが、
まずは次の 2 つを分けると分かりやすいです。

## Response Headers Policy

**ブラウザに何を返すか** を決める設定です。

たとえば、

- `Access-Control-Allow-Origin`
- `Access-Control-Allow-Methods`
- `Strict-Transport-Security`

のようなレスポンスヘッダーを付けられます。

---

## Origin Request Policy / Cache Policy

こちらは
**Origin に何を渡すか** や **CloudFront のキャッシュ** の設定です。

つまり、

- ブラウザに返すヘッダー
- Origin に渡すリクエストヘッダー

は別物です。

CORS でハマるときは、ここを混同していることが多いです。

---

# `Access-Control-Allow-*` の意味

## `Access-Control-Allow-Origin`

**どの Origin からのアクセスを許可するか** です。

```http
Access-Control-Allow-Origin: *
```

または

```http
Access-Control-Allow-Origin: https://app.example.com
```

- `*` は広く公開したいとき
- 特定の Origin は限定公開したいとき

ただし、**認証付き CORS では `*` は使えません。**

---

## `Access-Control-Allow-Methods`

**許可する HTTP メソッド** です。

```http
Access-Control-Allow-Methods: GET, POST, OPTIONS
```

主に preflight (`OPTIONS`) への応答で使います。

---

## `Access-Control-Allow-Headers`

**実際のリクエストで送ってよいヘッダー** です。

```http
Access-Control-Allow-Headers: Content-Type, Authorization
```

たとえば `Authorization` や独自ヘッダーを送るなら必要です。

---

## `Access-Control-Allow-Credentials`

**Cookie や認証情報を許可するか** です。

```http
Access-Control-Allow-Credentials: true
```

これを使うときは、
`Access-Control-Allow-Origin: *` にはできません。

---

## `Access-Control-Expose-Headers`

**JavaScript から読めるレスポンスヘッダーを増やす** ためのものです。

たとえば `ETag` や `X-Request-Id` をフロントから読みたいときに使います。

---

## `Access-Control-Max-Age`

**preflight の結果を何秒キャッシュするか** です。

```http
Access-Control-Max-Age: 600
```

---

# preflight って何か

ブラウザは、いきなり本番リクエストを送らずに、
先に `OPTIONS` で確認することがあります。
これが **preflight** です。

たとえば次のときに起こりやすいです。

- `Authorization` ヘッダーを付ける
- `PUT` / `DELETE` を使う
- `application/json` を送る

このときサーバーは、

- そのメソッドでいいか
- そのヘッダーでいいか
- その Origin を許可するか

を `Access-Control-Allow-*` で返します。

---

# CloudFront で見るべきポイント

CloudFront で CORS を見るときは、次の 2 点です。

## 1. ブラウザに返すヘッダーは正しいか

これは **Response Headers Policy** 側です。

---

## 2. Origin に必要なヘッダーを渡しているか

Origin 側で CORS 判定するなら、CloudFront から Origin に

- `Origin`
- `Access-Control-Request-Method`
- `Access-Control-Request-Headers`

を渡す必要があります。

これは **Origin Request Policy** 側の話です。

---

# よくあるハマりどころ

## 1. `*` と credentials を一緒に使う

これはブラウザで失敗します。

- `Access-Control-Allow-Credentials: true`
- `Access-Control-Allow-Origin: *`

は一緒に使えません。

---

## 2. `Authorization` を許可していない

認証付き API なのに
`Access-Control-Allow-Headers` に `Authorization` がないと失敗しやすいです。

---

## 3. `OPTIONS` を見ていない

`GET` は通るのに `POST` だけ失敗するなら、
preflight の応答不足を疑うと切り分けしやすいです。

---

## 4. Response Headers Policy だけ見ている

Origin 側で CORS を判断する構成なら、
Origin Request Policy も見ないと解決しません。

---

# まとめ

CloudFront の Response Headers Policy は、
**CloudFront がブラウザへ返すレスポンスヘッダーを制御する設定** です。

CORS では、特に次だけ押さえるとかなり分かりやすくなります。

- `Access-Control-Allow-Origin` : どの Origin を許可するか
- `Access-Control-Allow-Methods` : どの HTTP メソッドを許可するか
- `Access-Control-Allow-Headers` : どのリクエストヘッダーを許可するか
- `Access-Control-Allow-Credentials` : Cookie / 認証を許可するか

そして CloudFront では、

- **Response Headers Policy = ブラウザへ返す設定**
- **Origin Request Policy = Origin に渡す設定**

と分けて考えるのがポイントです。

---

### 参照したインターネットソース

- Add or remove HTTP headers in CloudFront responses with a policy  
  https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/modifying-response-headers.html

- Understand response headers policies  
  https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/understanding-response-headers-policies.html

- Use managed response headers policies  
  https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-response-headers-policies.html

- Use managed origin request policies  
  https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-managed-origin-request-policies.html

- Cross-Origin Resource Sharing (CORS)  
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS
