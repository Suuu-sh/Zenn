---
title: 'CloudFrontのResponse Headers PolicyとAccess-Control-Allow-*を整理する'
emoji: '☁️'
type: 'tech'
topics: ['aws', 'cloudfront', 'cors', 'http']
published: false
---

# はじめに

CloudFront を触っていると、

- Response Headers Policy って結局何をしているのか
- `Access-Control-Allow-*` はどれが何の役割なのか
- CORS が通らないとき、CloudFront と Origin のどちらを見るべきか

が、地味に混乱しやすいです。

この記事では、

- CloudFront の **Response Headers Policy の仕組み**
- `Access-Control-Allow-*` 系ヘッダーの **役割**
- CloudFront で CORS を設定するときの **考え方とハマりどころ**

をまとめます。

なお、最初に 1 つだけ整理しておくと、
**`Allow-Access-Allow-*` ではなく `Access-Control-Allow-*`** です。

---

# まず結論

CloudFront の Response Headers Policy は、
**CloudFront がブラウザへ返すレスポンスヘッダーを後段で調整する仕組み** です。

つまり、

- Origin が返したヘッダーをそのまま返すだけではなく
- CloudFront 側でヘッダーを追加・上書き・削除できる

ということです。

特に CORS では、CloudFront からブラウザへ返すレスポンスに

- `Access-Control-Allow-Origin`
- `Access-Control-Allow-Methods`
- `Access-Control-Allow-Headers`
- `Access-Control-Allow-Credentials`
- `Access-Control-Expose-Headers`
- `Access-Control-Max-Age`

のようなヘッダーを付けて、
**「このクロスオリジンアクセスをブラウザが許可してよいか」** を伝えます。

---

# Response Header とは何か

HTTP のレスポンスには本文だけでなく、
**レスポンスヘッダー** が付きます。

たとえば次のようなものです。

- `Content-Type`
- `Cache-Control`
- `Set-Cookie`
- `Access-Control-Allow-Origin`
- `Strict-Transport-Security`

ブラウザは本文だけを見ているわけではなく、
これらのヘッダーを見て

- キャッシュしてよいか
- Cookie を保存してよいか
- JavaScript から参照を許可するか
- HTTPS を強制するか

を判断します。

CORS もその 1 つで、
**レスポンスヘッダーで許可条件を返す仕組み** です。

---

# CloudFront の Response Headers Policy はどこで効くのか

CloudFront の Response Headers Policy は、
**cache behavior に紐づけて使う設定** です。

ざっくり流れを書くとこうです。

```text
Browser
  -> CloudFront
    -> Origin

Origin response
  -> CloudFront が response headers policy を適用
  -> Browser へ返す
```

ポイントは次の通りです。

- CloudFront は **Origin から来たレスポンス** に対してヘッダーを追加・削除できる
- その変更は、**CloudFront がキャッシュから返すレスポンスにも適用** される
- Origin を改修しなくても、CloudFront 側でブラウザ向けヘッダーを調整できる

また AWS ドキュメント上でも、
Response Headers Policy は
**キャッシュから返すレスポンスにも、Origin から転送するレスポンスにも適用される** とされています。

---

# Response Headers Policy でできること

CloudFront の Response Headers Policy では、主に次のことができます。

## 1. CORS ヘッダーの付与

代表例が `Access-Control-Allow-*` 系です。

- フロントエンドから別オリジン API を呼ばせたい
- S3 配信コンテンツを別ドメインの画面から読み込みたい
- `Authorization` ヘッダー付きのクロスオリジンリクエストを許可したい

といったときに使います。

---

## 2. セキュリティヘッダーの付与

たとえば次のようなヘッダーです。

- `Strict-Transport-Security`
- `Content-Security-Policy`
- `X-Frame-Options`
- `X-Content-Type-Options`

Origin で細かく返していなくても、
CloudFront 側である程度そろえられます。

---

## 3. カスタムヘッダーの追加

独自のヘッダーも追加できます。

- `X-App-Version`
- `X-Env`
- `Cache-Control`

などです。

ただし `Cache-Control` を Response Headers Policy で追加しても、
それは **ブラウザ向けレスポンスに付く** だけで、
**CloudFront 自身のキャッシュ制御には直接効きません。**

---

## 4. 不要ヘッダーの削除

たとえば

- `X-Powered-By`
- 不要な `Vary`

などを削除できます。

ただし、CORS を動的に返している構成で `Vary: Origin` まで消すと危険なので、
ここは安易に触らないほうが安全です。

---

## 5. Server-Timing の付与

CloudFront の処理状況を `Server-Timing` ヘッダーで見られるようにもできます。
パフォーマンス調査用です。

---

# `Access-Control-Allow-*` の役割

ここが一番混乱しやすいところです。

## 1. `Access-Control-Allow-Origin`

**どの Origin からのアクセスを許可するか** を表します。

例:

```http
Access-Control-Allow-Origin: *
```

または

```http
Access-Control-Allow-Origin: https://app.example.com
```

意味は次の通りです。

- `*` : 認証情報を使わない公開アクセス向け
- 明示的な Origin : 特定サイトだけ許可したい場合

一番大事なのは、
**Cookie や認証付きの CORS では `*` は使えない** ことです。

---

## 2. `Access-Control-Allow-Methods`

**クロスオリジンで許可する HTTP メソッド** を表します。

例:

```http
Access-Control-Allow-Methods: GET, POST, OPTIONS
```

これは主に **preflight request (`OPTIONS`) の応答** で使われます。

つまりブラウザが、

- `POST` していい？
- `PUT` していい？
- `DELETE` していい？

を事前確認するときに返す情報です。

---

## 3. `Access-Control-Allow-Headers`

**実際のリクエストで送ってよいヘッダー** を表します。

例:

```http
Access-Control-Allow-Headers: Content-Type, Authorization
```

これも主に preflight の応答で使われます。

たとえばフロントエンドが

- `Authorization`
- `Content-Type: application/json`
- 独自ヘッダー `X-Api-Key`

などを付けて送るなら、
それを許可する必要があります。

CloudFront の CORS 設定では、
**`Authorization` はワイルドカードで省略できず、明示的に列挙が必要** です。

---

## 4. `Access-Control-Allow-Credentials`

**Cookie や認証情報を含むクロスオリジンリクエストを許可するか** を表します。

使うときは次のようになります。

```http
Access-Control-Allow-Credentials: true
```

注意点はかなり重要で、

- `Access-Control-Allow-Credentials: true`
- `Access-Control-Allow-Origin: *`

の組み合わせはブラウザで通りません。

認証付き CORS をやるなら、
`Access-Control-Allow-Origin` は `*` ではなく
**具体的な Origin** を返す必要があります。

---

## 5. `Access-Control-Expose-Headers`

**JavaScript から読めるレスポンスヘッダーを追加で公開する** ためのヘッダーです。

ブラウザは CORS レスポンスで、
すべてのレスポンスヘッダーを JavaScript に見せるわけではありません。

なので、たとえばフロントエンドから

- `ETag`
- `Content-Disposition`
- `X-Request-Id`

を読みたいなら、
`Access-Control-Expose-Headers` に並べる必要があります。

---

## 6. `Access-Control-Max-Age`

**preflight の結果をブラウザが何秒キャッシュしてよいか** を表します。

例:

```http
Access-Control-Max-Age: 600
```

preflight が毎回飛ぶと無駄が増えるので、
必要に応じて設定します。

---

# simple request と preflight request

CORS を理解するときは、
**simple request と preflight request を分けて考える** と分かりやすいです。

## simple request

たとえば単純な `GET` で、
特殊なヘッダーも付けていないようなケースです。

この場合は preflight が発生せず、
レスポンスの `Access-Control-Allow-Origin` などを見てブラウザが判断します。

---

## preflight request

次のようなとき、ブラウザは本番リクエストの前に `OPTIONS` を送ります。

- `PUT` / `PATCH` / `DELETE` を使う
- `Authorization` ヘッダーを付ける
- `application/json` を送る
- 独自ヘッダーを付ける

このときサーバー側は、主に

- `Access-Control-Allow-Methods`
- `Access-Control-Allow-Headers`
- `Access-Control-Allow-Origin`
- 必要なら `Access-Control-Allow-Credentials`
- 必要なら `Access-Control-Max-Age`

を返します。

---

# CloudFront で CORS を設定するときの考え方

CloudFront で CORS を考えるときは、
**レスポンス側** と **Origin 転送側** を分けて考えると整理しやすいです。

## 1. Response Headers Policy

これは
**ブラウザへ何を返すか** の設定です。

つまり、

- `Access-Control-Allow-Origin`
- `Access-Control-Allow-Methods`
- `Access-Control-Allow-Headers`
- `Access-Control-Expose-Headers`
- `Access-Control-Allow-Credentials`
- `Access-Control-Max-Age`

を、CloudFront が viewer response にどう付けるかを決めます。

---

## 2. Origin Request Policy / Cache Policy

これは
**Origin にどのリクエスト情報を渡すか** の設定です。

ここを混同するとハマります。

たとえば Origin 側で

- `Origin` を見て許可可否を変える
- preflight の `Access-Control-Request-Method` を見て判断する
- `Access-Control-Request-Headers` を見て判断する

ような実装なら、CloudFront から Origin へ
必要なリクエストヘッダーを転送しないと動きません。

AWS には managed origin request policy として

- `CORS-CustomOrigin`
- `CORS-S3Origin`

も用意されています。

---

# CloudFront の managed policy は何を選べばいいか

まずは次の理解で十分です。

## `SimpleCORS`

- simple request 向け
- `Access-Control-Allow-Origin: *` を付ける
- preflight 前提の用途には足りない

---

## `CORS-With-Preflight`

- preflight ありの CORS 向け
- `Access-Control-Allow-Origin`
- `Access-Control-Allow-Methods`
- `Access-Control-Expose-Headers`

を managed policy で付与する

ただしこの managed policy には
`Access-Control-Allow-Headers` や `Access-Control-Allow-Credentials` は含まれません。
そのため、`Authorization` や独自ヘッダーを使う API では
**custom policy のほうが安全なことが多い** です。

---

## `CORS-with-preflight-and-SecurityHeadersPolicy`

- preflight あり CORS
- さらにセキュリティヘッダーもまとめて入れたい

ときのセットです。

---

# `Origin override` がかなり重要

CloudFront の Response Headers Policy には
`Origin override` という考え方があります。

これは、
**Origin が返したヘッダーと CloudFront policy のヘッダーがぶつかったとき、どちらを優先するか**
です。

CORS ではここが特に重要です。

- `true` : CloudFront policy 側を優先する
- `false` : Origin 側を優先する

そして CORS では `false` のときの挙動が少し強めで、
**Origin が CORS ヘッダーを返している場合、CloudFront policy 側の CORS ヘッダーは使われません。**

つまり、

- Origin でも中途半端に CORS を返している
- CloudFront でも CORS を足そうとしている

という二重管理にすると、
想定どおりにならないことがあります。

個人的には、

- CORS を Origin で管理するのか
- CORS を CloudFront で管理するのか

を、できるだけ **どちらかに寄せる** のが安全です。

---

# よくあるハマりどころ

## 1. `Access-Control-Allow-Origin: *` と認証を一緒に使っている

これは定番です。

- Cookie あり
- `fetch(..., { credentials: 'include' })`
- Authorization 付き

のような構成では、
`Access-Control-Allow-Origin: *` ではブラウザに拒否されます。

---

## 2. preflight を許可していない

`GET` は通るのに、
`POST` や `DELETE` だけ失敗するケースです。

この場合はだいたい

- `OPTIONS` を許可していない
- `Access-Control-Allow-Methods` が足りない
- `Access-Control-Allow-Headers` に `Authorization` や `Content-Type` がない

のどれかです。

---

## 3. Origin に必要なヘッダーを転送していない

Origin 側で CORS 判定しているのに、CloudFront から

- `Origin`
- `Access-Control-Request-Method`
- `Access-Control-Request-Headers`

を渡していないと、Origin 側で正しく判定できません。

Response Headers Policy だけ見ていても解決しないパターンです。

---

## 4. `Vary: Origin` を軽く扱ってしまう

特定 Origin ごとに `Access-Control-Allow-Origin` を返し分けるなら、
`Vary: Origin` も重要です。

動的に Origin を返し分けているのに `Vary: Origin` がないと、
キャッシュやブラウザの扱いが不正確になる可能性があります。

---

## 5. CloudFront のキャッシュ設定と混同する

Response Headers Policy で `Cache-Control` を追加しても、
それは **viewer response 用** です。

CloudFront 自身のキャッシュキーや TTL を決めるのは別の設定です。
ここを混ぜると、
「ヘッダーは付いたのに CloudFront のキャッシュ挙動は変わらない」
となります。

---

# どう使い分けると分かりやすいか

実務ではざっくり次の分け方が分かりやすいです。

## パターン1: 公開アセット配信

- S3 + CloudFront
- 認証なし
- 別ドメインから画像や静的ファイルを読ませたい

この場合は `Access-Control-Allow-Origin: *` ベースでも整理しやすいです。
まずは managed policy の `SimpleCORS` が候補になります。

---

## パターン2: API を跨いで呼ぶ

- フロントエンド `https://app.example.com`
- API `https://api.example.com`
- `Authorization` あり
- `application/json` あり

この場合は preflight も発生しやすく、
`*` ではなく **明示的な Origin** が必要になりやすいです。

このパターンは managed policy をそのまま使うより、
Origin 側も含めて custom policy を設計したほうが安全です。

---

# まとめ

CloudFront の Response Headers Policy は、
**CloudFront がブラウザへ返すレスポンスヘッダーを制御する仕組み** です。

特に CORS では、
`Access-Control-Allow-*` 系ヘッダーの役割を切り分けると理解しやすくなります。

- `Access-Control-Allow-Origin` : どの Origin を許可するか
- `Access-Control-Allow-Methods` : どの HTTP メソッドを許可するか
- `Access-Control-Allow-Headers` : どのリクエストヘッダーを許可するか
- `Access-Control-Allow-Credentials` : Cookie / 認証情報を許可するか
- `Access-Control-Expose-Headers` : JavaScript に見せるレスポンスヘッダーは何か
- `Access-Control-Max-Age` : preflight 結果を何秒キャッシュするか

そして CloudFront では、

- **Response Headers Policy = ブラウザへ返す内容**
- **Origin Request Policy / Cache Policy = Origin へ渡す内容とキャッシュの考え方**

を分けて考えるのが大事です。

CORS が崩れたときは、
**viewer response と origin request を別レイヤーで見る** と、かなり切り分けしやすくなります。

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

- Control origin requests with a policy  
  https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/controlling-origin-requests.html

- Cache content based on request headers  
  https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/header-caching.html

- Cross-Origin Resource Sharing (CORS)  
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Guides/CORS

- Access-Control-Allow-Origin  
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Origin

- Access-Control-Expose-Headers  
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Expose-Headers

- Access-Control-Allow-Credentials  
  https://developer.mozilla.org/en-US/docs/Web/HTTP/Reference/Headers/Access-Control-Allow-Credentials
