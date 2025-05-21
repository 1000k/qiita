---
title: Gitリポジトリの中身を、ブランチとタグも含めて別リポジトリにコピーする
tags:
  - Git
  - GitHub
  - Bitbucket
private: false
updated_at: '2020-11-16T16:29:24+09:00'
id: 2a5dcda5fd41f38fd5d5
organization_url_name: null
slide: false
ignorePublish: false
---
Bitbucket にあるソースを丸ごと GitHub Enterprise に移行することになりました。「丸ごと」というのは、 main 以外の全ての branches と tags も含めての移行です。

しかし、普通に `git clone -> git remote set-url -> git push` するだけでは main しか移行できません。

少しコマンドが複雑だったので、手順をメモしておきます。


やりたいこと
----
- Bitbucket にある Git リポジトリの中身を、 branches と tags も含めて GitHub Enterprise に移行する


コマンド
----
まずは移行元リポジトリの最新のソースを clone します。

```bash
git clone https://bitbucket.org/1000k/FOOBAR.git
cd FOOBAR/
```

この時点では main しかダウンロードできていません。

次のコマンドで、リモートの全ブランチを再帰的にダウンロードします。

```bash
git branch -r | grep -v "\->" | grep -v main | while read remote; do git branch --track "${remote#origin/}" "$remote"; done
git fetch --all
git pull --all
```

これでリポジトリの全コンテンツがダウンロードされました。

あとは移行先のリポジトリに push するだけです。

```bash
# Origin (リポジトリURL) を切り替える
git remote set-url origin https://github.com/1000k/FOOBAR.git

# すべてのブランチを push
git push --all origin
git push --tags
```

参考
----
- [branch - How to fetch all git branches - Stack Overflow](http://stackoverflow.com/questions/10312521/how-to-fetch-all-git-branches)
- [version control - Set up git to pull and push all branches - Stack Overflow](http://stackoverflow.com/questions/1914579/set-up-git-to-pull-and-push-all-branches)
