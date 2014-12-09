---
layout: post
title: "EmacsでUnity開発をする"
date: 2014-12-10 06:00:00 +0900
categories: 
---
これは[ドリコム Advent Calendar 2014](http://www.adventar.org/calendars/518)の10日目の記事です。

9日目は[@hiracy](https://twitter.com/y05_net)さんによる[サーバが増えた時にインフラ担当者がやってきたこと](http://qiita.com/y05_net/items/c8bfb69ff624d4b711e4)です。

## 自己紹介
* [@tkeo](https://twitter.com/tkeo)
* 2007年新卒でドリコムに入社、8年目
* メインの仕事はスマホゲームのサーバサイド(Rails)で、今年からクライアント(Unity)のほうも触り始めた
* Rails歴 6年、Unity歴 8ヶ月
* いまの作業比率は サーバ:クライアント = 2:1 ぐらい

## 今回の話
Unityを始めた当初はMonoDevelopを使っていたのですが、どうにも手になじまず腹が立ったので、普段サーバ側の開発で使っているemacsを使うことにしました。

以下はいろいろハマりながらも、インターネット上の先人の力を借りてまとめた自分の設定です。

## 環境
* Mac OS X Mavericks
* emacs 24.4.1
* パッケージ管理はcask

## [csharp-mode](https://github.com/josteink/csharp-mode)
これがなくては始まらないC#用のメジャーモード。

```lisp
; cask
(depends-on "csharp-mode")
```

```lisp
; init.el
(require 'csharp-mode)
(add-hook 'csharp-mode-hook
          '(lambda ()
             (setq indent-tabs-mode nil)
             (setq c-basic-offset 4)
             (c-set-offset 'substatement-open 0)
             (flycheck-mode 1)
             (omnisharp-mode)))
```

## 自動補完

### [OmniSharpServer](https://github.com/OmniSharp/omnisharp-server)
C#のコードをパースしたりいろいろやってくれる（らしい）サーバです。
他のエディタでも使えるらしいです。

monoが必要なのでインストールして、

```text
$ brew install mono
```

あとはREADME通りにビルド。

```text
$ git clone https://github.com/nosami/OmniSharpServer.git
$ cd OmniSharpServer
$ git submodule update --init --recursive
$ xbuild
```

すると`OmniSharp/bin/Debug`に`OmniSharp.exe`ができているはず。
試しに起動してみる。

```text
$ mono OmniSharp/bin/Debug/OmniSharp.exe -s /path/to/unity-project.sln
```

するとずらずらっとLoadingほげほげと出てきて、`Solution has finished loading`で起動完了。無事起動が確認できたらCtrl+Cで止めておく（このあとemacsから起動するので）

### [omnisharp-emacs](https://github.com/OmniSharp/omnisharp-emacs)
OmniSharpServerをemacsから使うためのものです。

```lisp
; cask
(depends-on "omnisharp")
```

```lisp
; init.el
(require 'omnisharp)
(setq omnisharp-server-executable-path (expand-file-name "/path/to/OmniSharp/bin/Debug/OmniSharp.exe"))
```

csファイルを開いた時にslnファイルを選べと言われるので、選んであげるとバックグラウンドで起動してくれます。

`M-x omnisharp-auto-complete`で候補表示、`M-x omnisharp-go-to-definition`で定義にジャンプできます。
適当なキーバインドを設定しておくと便利。

## 文法チェック
[flycheck](http://www.flycheck.org/en/latest/)

omnisharp-emacsにcheckerが含まれていて、OmniSharpサーバと連携してチェックをしてくれます。

```lisp
; cask
(depends-on "flycheck")
```

```lisp
; init.el
(require 'flycheck)
(setq flycheck-check-syntax-automatically '(mode-enabled save idle-change))
(setq flycheck-idle-change-delay 2)
```

## emacsclientアプリ化
Unity上でcsファイルをダブルクリックして開きたくなったときのための設定（あまりしないけど）

Unityの設定で外部エディタに指定できるのはMacのアプリ(*.app)だけなので、emacsclientを起動するアプリを作ります。
[vim用にやっている人がいた](https://gist.github.com/mkitt/824401)のでその設定を参考にしました。

アプリを作るのにはAutomatorを使います。手順は以下のとおり。

1. アプリケーションを選択。
2. アクション＞ユーティリティの中にある「AppleScriptを実行」を右にドラッグ＆ドロップ。
3. 下記コードを貼り付けて、appを適当な場所に保存。

```applescript
on run {input, parameters}
    do shell script "/usr/local/bin/emacsclient -n " & quoted form of POSIX path of input
end run
```

![automator](/images/screenshot-2014-12-10-automator.png)

作ったappをUnity上で指定してあげると、外部エディタとしてemacsが使えるようになります。
あとはserverとして起動するのを忘れずに。

```lisp
; init.el
(require 'server)
(unless (server-running-p)
  (server-start))
```

## リファレンス検索
rubyの開発でも使っている[Dash](http://kapeli.com/dash)を使えるようにします。

### dash-at-point
```lisp
(depends-on "dash-at-point")
```

```
; init.el
(global-set-key (kbd "C-c d") 'dash-at-point)
(global-set-key (kbd "C-c e") 'dash-at-point-with-docset)
(add-to-list 'dash-at-point-mode-alist '(csharp-mode . "cs"))
```

カーソル位置の単語で検索します。
上記設定ではcsharp-modeのときはキーワード "cs" が設定されたdocsetの中から検索します。

![dash](/images/screenshot-2014-12-10-dash.png)

Dash側で画像のようにキーワードを設定しておくと、.NET FrameworkとUnity 3Dのドキュメントを同時に検索できます。

### 自前docsetを作る
doxygenを使えばDash用のdocsetを作れます。
コードを含むアセットを購入した時に（たまに）やってます。

```text
$ brew install doxygen
```

```text
$ cat doxygen.config
GENERATE_DOCSET        = YES
SEARCHENGINE           = NO
DISABLE_INDEX          = YES
GENERATE_TREEVIEW      = NO
GENERATE_LATEX         = NO
GENERATE_HTMLHELP      = YES
RECURSIVE              = YES

PROJECT_NAME           = hogehoge
OUTPUT_DIRECTORY       = output
INPUT                  = path/to/Scripts
DOCSET_BUNDLE_ID       = hogehoge
```

こんな感じの設定ファイルを作って、まずはHTMLドキュメントを生成。

```text
$ doxygen doxygen.config
```

htmlディレクトリに移動してmakeをたたくとdocsetが作られます。

```text
$ cd output/html
$ make
$ open hogehoge.docset
```

## その他
日本語を含むときはBOM付きで保存しないと化けます。`C-x RET f`で`utf-8-with-signature`に変更。

## まとめ
以上、MonoDevelopを使わずにemacsでUnity開発するための設定を紹介しました。

明日はみっきーさんです。

## 参考リンク
* [EmacsでUnity開発(c#)するための環境構築 - Qiita](http://qiita.com/fujimisakari/items/d043a2fae31ed740e290)
* [Emacs24.3(for Mac OSX)でUnity用のC#を書くために - とあるぼっちの生存報告](http://bocchies.hatenablog.com/entry/2014/05/09/041130)
* [Emacs 24.3 + omnisharp-emacsで快適C#生活 [クソゲ〜製作所]](http://decomo.info/wiki/emacs/emacs_24.3_csharp_code_completion_with_omnisharp)
* [AppleScript for use with Automator which allows opening files within Terminal Vim](https://gist.github.com/mkitt/824401)
* [NGUI のプログラムを元にオフラインでも使えるAPIリファレンスを作る方法 - 強火で進め](http://d.hatena.ne.jp/nakamura001/20121208/1354985761)
