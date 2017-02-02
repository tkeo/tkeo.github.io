---
layout: post
title: "Octopressデプロイ環境を作り直した"
date: 2017-02-02 21:44:43 +0900
comments: true
categories: octopress
---
雑なメモ。
会社からでも更新できるように環境を作った。

とりあえずcloneしてsourceブランチに切り替える。

```
git clone git@github.com:tkeo/tkeo.github.io.git blog
cd blog
git co -t origin/source
```

もろもろ必要なgemをinstallする。bundle execとか付けたくないので雑にgemsetに突っ込む。

```
rbenv gemset init blog
bundle
```

デプロイするには_deployディレクトリにmasterブランチが必要なのでworktreeを作る。

```
git worktree add _deploy master
```

これでOK。
