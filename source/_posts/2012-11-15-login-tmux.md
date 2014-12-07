---
layout: post
title: ログイン直後にtmuxを起動する
date: 2012-11-15 14:15:22 +0900
categories: 
---
SHLVLっていう環境変数があることを知ったので、.zshrcに以下を追記してログイン直後にtmuxを起動するようにしてみた。

```
if [ $SHLVL = 1 ]; then
  tmux attach || tmux new
fi
```
