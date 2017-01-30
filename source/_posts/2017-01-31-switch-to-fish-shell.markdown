---
layout: post
title: "fish shellに移行した"
date: 2017-01-31 00:15:30 +0900
comments: true
categories: fish
---
学生の頃にzshを使い始めてかれこれ10年以上になるが、2ヶ月ほど前にfishに移行した。
ようやく慣れてきたのでメモを残しておく。

使用しているfishのバージョンは2.4.0で、プラグインなどは入れてない。
[oh-my-fish](https://github.com/oh-my-fish/oh-my-fish)を使うと便利っぽいけど、困ったときに入れたらいいかなと思ってまだ困ってないので使ってない。

## config

とりあえず`~/.zshrc`をベースに`~/.config/fish/config.fish`を作った。
`alias`とか`export`とかは大体そのままコピペで動くが、なぜかPATHのexportだけはうまくいかなかったので`set`を使うように変更した。

他は`if`を`end`で閉じるとか、`&&`のかわりに`; and`を使うとか、`$( )`は`( )`にするとか、気をつけないといけないポイントはいくつかあるけどそのへんは適当に。

## 補完

サーバーにログインするときはsshrcを使っていて、sshコマンドと同じようにタブでホスト名を補完してほしい。そういうときの設定は以下。

```
complete -c sshrc -w ssh
```

[complete - edit command specific tab-completions](https://fishshell.com/docs/current/commands.html#complete)

## 一時的な環境変数

例えばzshの場合だとこんな感じのコマンドを実行することがあるけど

```
RAILS_ENV=test rake db:reset
```

fishだとこれではダメで、`env`をつける必要がある。

```
env RAILS_ENV=test rake db:reset
```

[How do I set an environment variable for just one command?](https://fishshell.com/docs/current/faq.html#faq-single-env)

## ブレース展開

`{ }`の中身が展開されるため、展開を抑制したいときはエスケープが必要。

```
git stash drop stash@\{1\}
```

とか、

```
rails g model foo bar:references\{polymorphic\}
```

とか。ちょっと面倒。gitの引数の場合はタブ補完でバックスラッシュをちゃんとつけてくれる。

## プロセス置換

コマンド実行結果をファイルのように扱うやつ。zshだとこんな感じ。

```
diff <(foo) <(bar)
```

fishの場合は`psub`を使う。

```
diff (foo | psub) (bar | psub)
```

[psub - perform process substitution](https://fishshell.com/docs/current/commands.html#psub)

## 履歴

zshのshare_history相当のものが無いため最初は戸惑ったが、普段使っているコマンドのパターンはそれほど多くなかったので履歴が育てば不要だった。

それでもtmuxの別のwindow/paneで少し前に打ったコマンドがほしくなることはたまにあって、そういうときは`history merge`を実行して履歴を再読込している。

自動でmergeするのを試してみたけどコマンド実行のたびに微妙に待たされてストレスたまるので手動運用のほうがよいと思う。

[history - Show and manipulate command history](https://fishshell.com/docs/current/commands.html#history)
