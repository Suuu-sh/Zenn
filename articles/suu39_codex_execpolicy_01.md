<!-- ---
title: 'Codexの承認コマンドをrulesに蓄積する運用を作った話'
emoji: '🛡️'
type: 'tech'
topics: ['codex', 'openai', 'cli', 'security']
published: false
---

# はじめに

Codex を使っていると、同じようなコマンドに何度も承認を求められることがあります。

たとえば、読み取り専用の確認やローカルテストの実行は安全だと分かっていても、
毎回 `approval` が出ると少しだけ作業の流れが切れます。

そこで今回は、**セッションごとに承認したコマンドを、将来のセッションでも再利用できるように rules に蓄積する**運用を作りました。

この記事では、実際にどう整理したかを共有します。

---

# 先に結論

今回の運用は、かなりシンプルです。

- `~/.codex/rules/default.rules` を正本にする
- 安全に一般化できるコマンドだけを `prefix_rule()` で追加する
- `~/.codex/AGENTS.md` には「どう運用するか」を書く
- ルールを変えたら Codex を再起動する

ポイントは、**「その場で通ったコマンド全部を追加する」のではなく、「今後も安全に使える prefix だけを追加する」**ことです。

---

# 公式ドキュメントで押さえたこと

まず前提として、Codex の rules は公式ドキュメントで以下のように案内されています。

- `.rules` ファイルを `~/.codex/rules/` 配下に置く
- 例として `~/.codex/rules/default.rules` が挙げられている
- Codex は起動時に `rules/` 配下をスキャンする
- TUI で allow を追加すると、ユーザー層の `~/.codex/rules/default.rules` に書き込まれる
- ルールを変えたら Codex を再起動する

参考:

- [Codex Rules](https://developers.openai.com/codex/rules)
- [AGENTS.md](https://developers.openai.com/codex/guides/agents-md)

また、ルールの中身は `prefix_rule()` で書きます。
大事なのは次の3点です。

1. **prefix は先頭一致**
2. **`justification` を書ける**
3. **複数ルールが当たると、より厳しい判定が優先される**

この3つを意識すると、かなり事故りにくくなります。

---

# どういうコマンドを allow にしたか

今回は、次のようなものだけを allow する方針にしました。

| カテゴリ | 例 | 方針 |
|---|---|---|
| 読み取り | `cat`, `ls`, `find`, `grep`, `head`, `tail`, `wc`, `pwd` | allow |
| 無害な出力 | `echo`, `printf`, `true` | allow |
| Git の確認 | `git status`, `git diff`, `git log`, `git show` | allow |
| ローカル検証 | `npm test`, `npm run test`, `pytest`, `python3 -m pytest` | allow |
| パッケージ確認 | `pip list`, `pip show`, `pip freeze` | allow |
| 破壊的操作 | `rm -rf`, `git reset --hard` | 追加しない |
| インストール | `npm install`, `pip install` | 追加しない |
| ネットワーク | `curl`, `wget` など | 追加しない |

要するに、**読み取りと検証だけを広げて、書き込み系は広げない**方針です。

---

# 実際の `prefix_rule()` の考え方

たとえば、こんな感じです。

```starlark
prefix_rule(
    pattern = ["git", "status"],
    decision = "allow",
    justification = "Read-only git status is safe for local review.",
)
```

これは「`git status` 系は安全だから、今後は承認なしで通してよい」という意味です。

一方で、同じ Git でも `git reset --hard` は別物です。
履歴や作業ツリーを書き換えるので、**allow にしません**。

同じ考え方で、`npm test` は allow しても、`npm install` は allow しません。

---

# ルールは「安全に一般化できるもの」だけ追加する

ここがいちばん大事でした。

セッション中に承認されたコマンドの中には、
**その場限りでしか安全でないもの**もあります。

たとえば:

- 特定のリポジトリだけで安全
- 特定のパスだけで安全
- 引数次第で危険になる
- シェルラッパーの中に複数コマンドが隠れる

こういうものを雑に広く allow すると、後で困ります。

なので、ルール化するときは以下を意識しました。

1. **prefix を短くしすぎない**
2. **逆に広げすぎない**
3. **`justification` に理由を書く**
4. **`match` / `not_match` で自分でも確認する**

Codex の rules は、ただのメモではなく、実際に動くポリシーです。
だからこそ、**「安全に再利用できるか」**を優先しました。

---

# `AGENTS.md` に書いたこと

`AGENTS.md` は execpolicy の本体ではありません。
でも、**運用ルールを書いておく場所**としてはかなり便利です。

今回の整理では、`~/.codex/AGENTS.md` に次のような趣旨を入れました。

- `~/.codex/rules/default.rules` を正本として扱う
- セッションで承認されたうち、一般化できるものだけ追記する
- destructive / install / network 系は入れない

これは「Codex に何をさせるか」ではなく、**自分たちの運用方針を固定する**ためのものです。

---

# 検証は `codex execpolicy check` でやる

ルールを足したら、`codex execpolicy check` で確認すると分かりやすいです。

たとえば:

```bash
codex execpolicy check --pretty --rules ~/.codex/rules/default.rules -- git status
codex execpolicy check --pretty --rules ~/.codex/rules/default.rules -- npm test
codex execpolicy check --pretty --rules ~/.codex/rules/default.rules -- python3 -m pytest
```

このあたりは `allow` になってほしいコマンドです。

逆に:

```bash
codex execpolicy check --pretty --rules ~/.codex/rules/default.rules -- git reset --hard
```

これは **allow されない** ことを確認したいコマンドです。

「通したいものが通る」だけではなく、**通してはいけないものが通らない** ことも確認するのが大事です。

---

# こうすると何がうれしいか

この運用にしてから、次のメリットがありました。

- 毎回の approval が減る
- いつも使う安全なコマンドが安定する
- ルールが増えても `default.rules` にまとまる
- `AGENTS.md` で運用意図を残せる

特に便利なのは、**「安全な行動を Codex に覚えさせる」** という感覚です。

その場の承認が、次回以降の手間削減につながります。

---

# Codex にこの運用を毎回やらせるためのプロンプト

最後に、この運用を **新しい Codex セッションでも毎回同じ方針で進めさせるためのプロンプト** を置いておきます。

新しいセッションの最初に、以下をそのまま貼る想定です。

```text
あなたは Codex として、私のローカル開発環境にある execpolicy rules を安全にメンテナンスしてください。

方針:
- 正本は /Users/yota/.codex/rules/default.rules とする
- セッション中に承認されたコマンドのうち、今後も安全に一般化できるものだけを最小限の prefix_rule として追加する
- 追加対象は read-only inspection、local verification、harmless no-op に限定する
- destructive commands、dependency installation、package publishing、network access を伴うコマンドは追加しない
- ルールを編集したら codex execpolicy check で allow / deny を検証する
- 必要なら /Users/yota/.codex/AGENTS.md も同じ方針に合わせて更新する

出力の期待値:
- 変更が必要なら、実際にファイルを更新する
- 更新後に、追加したルールの意図と検証結果を短く説明する
- 安全でない拡張は提案だけに留める
```

このプロンプトを入れておくと、Codex に
**「承認されたコマンドを、そのままではなく安全に一般化して rules に落とす」**
という作業を毎回再現させやすくなります。

---

# 注意点

この運用は便利ですが、何でも allow すると危険です。

特に次は慎重に扱ったほうがいいです。

- `rm`
- `git reset --hard`
- `git push`
- `npm install`
- `pip install`
- `curl` や `wget` のような外部アクセス
- `bash -lc` のようなシェルラッパーの広すぎる許可

Codex の rules は強力なので、**「一度通ったから全部通す」** にしないのがコツです。

---

# まとめ

Codex の承認コマンドを rules に蓄積すると、かなり快適になります。

ただし、やるべきなのは「全部を覚えさせること」ではなく、
**安全に一般化できる prefix だけを、最小限で allow すること**です。

今回の構成をもう一度まとめると、こうなります。

- `~/.codex/rules/default.rules` に execpolicy を集約する
- `prefix_rule()` は安全な prefix だけに使う
- `AGENTS.md` で運用方針を共有する
- `codex execpolicy check` で allow / deny を検証する

同じように Codex を使っている人の参考になればうれしいです。 -->
