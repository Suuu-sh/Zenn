---
title: '個人開発でSentry × Codex × GitHub Actionsによる障害対応の初動自動化をやっている話'
emoji: '🚨'
type: 'tech'
topics: ['sentry', 'codex', 'githubactions', 'devops']
published: false
---

# はじめに

個人開発している backend で本番エラーが見つかったあとに、毎回かなり似た初動作業が発生していました。

- どの issue から手を付けるか決める
- 調査用の branch を切る
- 共有用の PR を作る
- 修正に必要な材料を集める

こういう部分は、人が毎回手でやるより、自動化したほうが速くて安定します。
そこで今は、**Sentry / GitHub Actions / Codex を組み合わせて、障害対応の入り口を自動化する仕組み**を個人開発で回しています。

この記事では、個人開発で実際に運用している構成をそのまま整理します。
ポイントは、**完全自動修復を目指すのではなく、修正に入りやすい状態を自動で作る**ことです。

---

# 先に全体像

今の流れを一枚で書くとこうです。

```mermaid
flowchart LR
    A["Sentry\nprod の未解決 issue"] --> B["GitHub Actions\n毎日 09:00 JST / 手動実行"]
    B --> C["sentry_triage.py\nissue と latest event を取得"]
    C --> D["issue ごとの branch を作成\nfeature/sentry short-id"]
    C --> E["調査レポートを生成\ndocs/sentry-triage short-id.md"]
    C --> F["draft PR を作成または更新"]
    F --> G["Codex が PR とレポートを読む"]
    G --> H["最小修正を実装"]
    H --> I["CI で test / lint / build"]
    I --> J["レビューして merge"]
    J --> K["Deploy workflow で本番反映"]
```

要するに、
**Sentry が異常を見つけ、GitHub Actions が修正用の作業台を作り、その上で Codex が直しやすくする**
構成です。

---

# なぜこの形にしたか

本番エラー対応で重いのは、修正そのものよりも、修正に入るまでの段取りだったりします。

たとえば次のような作業です。

- Sentry で未解決 issue を探す
- 直近 event を開く
- branch 名を決める
- PR を作る
- 最初に何を見るべきかを文章に残す

このあたりは毎回かなり似ています。
だからこそ、自動化しやすいです。

一方で、

- この修正方針で本当に良いか
- 影響範囲はどこまでか
- deploy してよいか

のような判断は、まだ人間や Codex のレビューを挟みたいです。

なので個人開発では、
**収集・整理・着手準備までは自動化し、修正判断は人間と Codex に残す**
方針にしています。

---

# 1. Sentry 側の役割

まず、エラーの起点は Sentry です。
個人開発では backend と mobile の両方で Sentry を使っていますが、今の自動改善フローで対象にしているのは **backend の production issue** です。

backend 側では panic recovery や application error の capture を入れていて、本番で発生した異常を Sentry に集約しています。

自動化の入力条件はかなり絞っています。

- project: `go`
- environment: `prod`
- query: `is:unresolved`
- time range: `24h`
- limit: `3`

つまり、
**直近 24 時間の production 未解決 issue を最大 3 件まで拾う**
構成です。

ここを広げすぎると、古い issue やノイズが増えて、かえって運用しづらくなります。
まずは小さめに回すのがちょうどよかったです。

---

# 2. GitHub Actions が毎日 issue を拾う

Sentry の issue 取得は GitHub Actions の workflow で回しています。
backend リポジトリには `Sentry Triage` workflow を置いていて、次のタイミングで動きます。

- 毎日 09:00 JST 相当
- `workflow_dispatch` による手動実行

やっていること自体はシンプルです。

1. backend リポジトリを checkout
2. Python をセットアップ
3. `SENTRY_AUTH_TOKEN` を確認
4. `scripts/sentry_triage.py` を実行

```yaml
on:
  schedule:
    - cron: '0 0 * * *'
  workflow_dispatch:
```

cron は UTC で書くので、`0 0 * * *` は JST だと 09:00 です。

朝の時点で issue と修正用 PR の土台が揃っている状態を作れるので、かなり扱いやすいです。

---

# 3. triage スクリプトがやっていること

中核は `scripts/sentry_triage.py` です。
このスクリプトは単に issue 一覧を取るだけではなく、**Codex がそのまま作業に入れる状態まで整える**役割を持っています。

## 3-1. issue ごとに branch を切る

未解決 issue を取得したら、`shortId` を使って branch を作ります。
形式は次のようなものです。

```text
feature/sentry/<short-id>
```

issue ごとに branch が分かれるので、

- 並列で修正しやすい
- PR の意図が分かりやすい
- 追跡や revert がしやすい

という利点があります。

---

## 3-2. 最新 event から調査レポートを作る

各 issue について、最新 event を追加で引いて Markdown の調査レポートを生成します。
保存先は次の形式です。

```text
docs/sentry-triage/<short-id>.md
```

レポートには次のような情報を入れています。

- issue の title / status / level / count
- first seen / last seen
- culprit
- latest event の timestamp / release / platform / logger
- event URL
- tags
- 次に見るべきこと

つまり、PR を開いた人が
**最初にどこから見ればよいか**
を迷わないようにしています。

---

## 3-3. 個人情報はそのまま出さない

地味ですが重要なのがここです。
triage スクリプトでは、レポート生成前に**メールアドレスや IP アドレスを redact**しています。

そのため、Sentry の event 情報をそのまま PR に貼るのではなく、共有に必要な情報だけを残しつつ、センシティブな値はマスクする構成です。

監視データを開発フローへ自動で流し込むなら、この層は最初から入れておいたほうが安全です。

---

## 3-4. draft PR を自動で作る

レポートを書いたら、その branch を push して draft PR を作ります。
同じ branch の PR が既にあれば、新規作成ではなく本文更新に寄せています。

ここで大事なのは、
**この PR は「直した」ことを示すのではなく、「この issue を直すための作業台ができた」ことを示す**
という点です。

だから draft PR にしていて、中には issue の要約と次のアクションを書いています。

この時点で、人間や Codex は

- どの issue を扱う PR なのか
- 何を見ればよいのか
- どの branch で直せばよいのか

をすぐ把握できます。

---

# 4. Codex はどこで効くのか

このフローで Codex が一番効くのは、
**調査材料が揃った issue 専用 branch に対して、最小修正を素早く入れるところ**です。

Sentry の画面だけ渡して Codex に直させようとすると、前提共有のコストが高くなります。
でも今の構成なら、最初から

- issue 単位の branch
- 調査レポート
- draft PR
- 既存の CI

が揃っています。
そのため、Codex がかなり入りやすいです。

```mermaid
sequenceDiagram
    participant S as Sentry
    participant G as GitHub Actions
    participant R as Triage Report / Draft PR
    participant C as Codex
    participant CI as CI
    participant H as Human

    S->>G: 未解決 issue が対象として残る
    G->>R: branch / report / draft PR を作成
    C->>R: issue の文脈を読む
    C->>C: 修正方針を決める
    C->>CI: 修正コミットを push
    CI-->>H: test / lint / build 結果を返す
    H->>H: 内容を確認して merge 判断
```

Codex に全部を丸投げするのではなく、
**GitHub Actions で前段を整地して、Codex が「読んで直す」に集中できるようにしている**
イメージです。

---

# 5. CI / Deploy まで含めて閉じる

修正したら終わりではなく、その後ろの流れもつないでいます。

backend には通常の CI workflow があり、PR に対して

- `go test ./... -v -race -coverprofile=coverage.out`
- `golangci-lint`
- `go build`

が走ります。

つまり、Codex が修正した内容も、**いつもの品質ゲート**にそのまま乗ります。

さらに `main` に merge されれば Deploy workflow が動き、

- migration 適用
- Fly.io への deploy

までつながります。

この構成にしておくと、Sentry から始まった issue 対応が
**検知 → 着手準備 → 修正 → 検証 → 本番反映**
まで一本の流れとして見えるようになります。

---

# この仕組みでよかったこと

実際に運用していてよかったのは、次の 4 点です。

## 1. 初動が速くなった

issue を見つけてから branch / PR / メモを用意するまでを自動化したので、修正に入り始めるまでがかなり短くなりました。

---

## 2. Codex に渡す文脈が安定した

Codex は強いですが、入力文脈がブレると修正品質も揺れます。
今は毎回ほぼ同じ形式の report と PR を用意しているので、Codex に渡す土台が安定しました。

---

## 3. issue ごとの作業が混ざりにくい

branch が issue 単位で分かれているので、複数の不具合対応が混線しにくくなりました。
これは人間にとっても、Codex にとっても重要です。

---

## 4. 完全自動にしなかったのがよかった

自動化は全部つなぎたくなりますが、修正判断まで無人化するのはまだ怖いです。

今は

- 収集
- 整理
- branch 作成
- PR 作成

までを自動化し、最後の修正判断と merge 判断は人間側に残しています。
この分担が今の規模ではちょうどよかったです。

---

# まだ残っている課題

もちろん、まだ完成形ではありません。

今の課題は次のあたりです。

- backend しか自動改善フローに乗っていない
- issue の優先度付けはまだ単純
- 修正案の自動生成までは workflow に入れていない
- 同一原因の issue を束ねる仕組みが弱い

特に最後は重要です。
1 issue = 1 branch は分かりやすい反面、根本原因が同じ issue が複数あると少し冗長になります。

今後は、

- issue のクラスタリング
- release や culprit を見た優先度調整
- Codex に渡す追加コンテキストの整備

あたりを進めると、さらに効率が上がりそうです。

---

# まとめ

個人開発でやっている自動改善システムを一言でいうと、

**Sentry が見つけた production issue を GitHub Actions が修正しやすい PR に変換し、その上で Codex が直しやすくする仕組み**
です。

大事だったのは、いきなり完全自動修復を目指すのではなく、
**人間と Codex が最短で修正に入れる状態を自動で作ること**でした。

このレイヤーだけでも、障害対応の体験はかなり変わります。

同じように

- Sentry は入っている
- GitHub Actions もある
- Codex で日常開発している

というチームなら、かなり相性がいい構成だと思います。
まずは**issue を拾って branch と draft PR を切るところまで**でも自動化してみると、効果が出やすいはずなのだ。
