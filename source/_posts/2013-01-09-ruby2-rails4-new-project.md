---
layout: post
title: Ruby2.0+Rails4で新規プロジェクトを作成したメモ
date: 2013-01-09 13:32:37 +0900
categories: 
---
## 前準備
    $ brew install openssl
    $ brew link openssl
    $ brew install readline
    $ brew link readline

プロジェクトのディレクトリ作る。

    $ mkdir hoge && cd hoge

## ruby

ruby 2.0.0-devを入れる。

    $ CONFIGURE_OPTS="--with-openssl-dir=`brew --prefix openssl` --with-readline-dir=`brew --prefix readline`" rbenv install 2.0.0-dev
    $ rbenv local 2.0.0-dev

bundlerのバージョンが1.2だと怒られるので1.3を入れる。

    $ gem install bundler -v 1.3.0.pre.4

## rails

一旦Gemfileにrailsだけ書いてbundle installする。 まだrails 4.0のgemがないのでgithubから取ってくるように指定。

    $ cat Gemfile
    source :rubygems
    gem 'rails', github: 'rails/rails'

bundle installを実行し、まずrails本体だけ入れる。

    $ bundle install

rails newしていろいろファイルを生成。Gemfileを上書きするか聞かれるので Y で上書き。

    $ bundle exec rails new . -d mysql -T --skip-bundle --edge

再度bundle install

    $ bundle install

これで完了。あとはいつもどおりDBを作って…

    $ bundle exec rake db:create

起動する。

     $ bundle exec rails s
    => Booting WEBrick
    => Rails 4.0.0.beta application starting in development on http://0.0.0.0:3000
    => Call with -d to detach
    => Ctrl-C to shutdown server
            SECURITY WARNING: No secret option provided to Rack::Session::Cookie.
            This poses a security threat. It is strongly recommended that you
            provide a secret to prevent exploits that may be possible from crafted
            cookies. This will not be supported in future versions of Rack, and
            future versions will even invalidate your existing user cookies.
    
            Called from: xxx/ruby/2.0.0/bundler/gems/rails-e63e280bed3a/actionpack/lib/action_dispatch/middleware/session/abstract_store.rb:24:in `initialize'.
    [2013-01-07 19:35:35] INFO  WEBrick 1.3.1
    [2013-01-07 19:35:35] INFO  ruby 2.0.0 (2013-01-07) [x86_64-darwin11.4.2]
    [2013-01-07 19:35:35] INFO  WEBrick::HTTPServer#start: pid=76843 port=3000

なにやらWARNING出てるけど、とりあえずwelcome画面が出た。

## 参考
* [Ruby2.0 + Rails4 なアプリを作成する - blog.katsuma.tv](http://blog.katsuma.tv/2013/01/ruby2.0_rails4.html)
* [Rails4 beta さわってみたメモ #railshackathon - 130単位](http://d.hatena.ne.jp/deeeki/20121124/rails4_1st_impression)
