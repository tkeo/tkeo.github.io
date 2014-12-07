---
layout: post
title: Rails4 で MiniTest + Spork + Guard
date: 2013-01-17 14:33:15 +0900
categories: 
---
正式版が出る頃には変わっている可能性が高いが、現時点でのメモを残しておく。

## minitest-rails

https://github.com/blowmage/minitest-rails

Gemfile に追加。

    gem "minitest-rails"

必要なファイルを生成。

    $ rails generate mini_test:install

test/minitest_helper.rb が生成される。

    ENV["RAILS_ENV"] = "test"
    require File.expand_path('../../config/environment', __FILE__)
    
    require "minitest/autorun"
    require "minitest/rails"
    
    # Add `gem "minitest/rails/capybara"` to the test group of your Gemfile
    # and uncomment the following if you want Capybara feature tests
    # require "minitest/rails/capybara"
    
    # Uncomment if you want awesome colorful output
    # require "minitest/pride"
    
    class ActiveSupport::TestCase
      # Setup all fixtures in test/fixtures/*.(yml|csv) for all tests in alphabetical order.
      fixtures :all
    
      # Add more helper methods to be used by all tests here...
    end

各testファイルでこれを読み込むために下の一行が必要。

    require 'minitest_helper'

動作確認。

    $ ruby -Itest test/foo/bar_test.rb


## spork-minitest

https://github.com/semaperepelitsa/spork-minitest

Gemfile に追加。

    gem "spork", "~> 1.0rc"
    gem "spork-minitest", "~> 1.0.0.beta1"

下記コマンドを実行し、minitest_helper に spork 向けの設定を追加する。

    $ spork minitest --bootstrap

test/minitest_helper.rb に追記されるので適宜編集する。

    $LOAD_PATH << "test"
    require 'rubygems'
    require 'spork'
    #uncomment the following line to use spork with the debugger
    #require 'spork/ext/ruby-debug'
    
    Spork.prefork do
      # Loading more in this block will cause your tests to run faster. However,
      # if you change any configuration or code from libraries loaded here, you'll
      # need to restart spork for it take effect.
    
      ENV["RAILS_ENV"] = "test"
      require File.expand_path('../../config/environment', __FILE__)
    
      require "minitest/autorun"
      require "minitest/rails"
      require 'simplecov'
      require 'simplecov-rcov'
    
      SimpleCov.formatter = SimpleCov::Formatter::RcovFormatter
      SimpleCov.start 'rails'
    
      # Add `gem "minitest/rails/capybara"` to the test group of your Gemfile
      # and uncomment the following if you want Capybara feature tests
      # require "minitest/rails/capybara"
    
      # Uncomment if you want awesome colorful output
      # require "minitest/pride"
    
      class ActiveSupport::TestCase
        # Setup all fixtures in test/fixtures/*.(yml|csv) for all tests in alphabetical order.
        fixtures :all
    
        # Add more helper methods to be used by all tests here...
      end
    end
    
    Spork.each_run do
      # This code will be run each time you run your specs.

    end
    
    # --- Instructions ---
    # Sort the contents of this file into a Spork.prefork and a Spork.each_run
    # block.
    #
    # The Spork.prefork block is run only once when the spork server is started.
    # You typically want to place most of your (slow) initializer code in here, in
    # particular, require'ing any 3rd-party gems that you don't normally modify
    # during development.
    #
    # The Spork.each_run block is run each time you run your specs.  In case you
    # need to load files that tend to change during development, require them here.
    # With Rails, your application modules are loaded automatically, so sometimes
    # this block can remain empty.
    #
    # Note: You can modify files loaded *from* the Spork.each_run block without
    # restarting the spork server.  However, this file itself will not be reloaded,
    # so if you change any of the code inside the each_run block, you still need to
    # restart the server.  In general, if you have non-trivial code in this file,
    # it's advisable to move it into a separate file so you can easily edit it
    # without restarting spork.  (For example, with RSpec, you could move
    # non-trivial code into a file spec/support/my_helper.rb, making sure that the
    # spec/support/* files are require'd from inside the each_run block.)
    #
    # Any code that is left outside the two blocks will be run during preforking
    # *and* during each_run -- that's probably not what you want.
    #
    # These instructions should self-destruct in 10 seconds.  If they don't, feel
    # free to delete them.

sporkの動作確認をする。

    $ spork

sporkを立ち上げた状態のまま、別窓を開いて testdrb でテスト実行してみる。

    $ testdrb test/**/*.rb


## guard-minitest

https://github.com/guard/guard-minitest

本家のものを使いたいところではあるが、sporkと組み合わせた時に動かないので一旦他の人が作ったブランチを使わせてもらう。
(ref. https://github.com/guard/guard-minitest/pull/41)

sporkを使わないのであれば本家のものでもおそらく問題ない。

Gemfile に追加。

    gem "guard-minitest", github: "itrcdevs/guard-minitest", branch: "fix_for_spork-minitest"

設定ファイルを生成する。すでにファイルが存在する場合は内容が追記される。

    $ guard init minitest

生成された Guardfile を編集する。

    # More info at https://github.com/guard/guard#readme
    
    guard 'minitest' do
      # with Minitest
      watch(%r|^test/(.*)_test\.rb|)
      watch(%r|^lib/(.*)([^/]+)\.rb|)     { |m| "test/#{m[1]}test_#{m[2]}.rb" }
      watch(%r|^test/test_helper\.rb|)    { "test" }
    
      # Rails
      watch(%r|^app/controllers/(.*)\.rb|) { |m| "test/controllers/#{m[1]}_test.rb" }
      watch(%r|^app/helpers/(.*)\.rb|)     { |m| "test/helpers/#{m[1]}_test.rb" }
      watch(%r|^app/models/(.*)\.rb|)      { |m| "test/models/#{m[1]}_test.rb" }
    end

guard の動作確認。

    $ guard

適当にファイルをtouchしてみるとテストが走るはず。
何も起こらない場合はちゃんと監視対象に入っているか記述を見なおしてみる。


## guard-spork

Gemfile に追加。

    gem "guard-spork"

Guardfile に設定を追加する。

    $ guard init spork

Guardfile に追加された spork の箇所の編集と、minitest で drb を使うように変更する。

    guard 'spork', minitest: true, minitest_env: { 'RAILS_ENV' => 'test' }, bundler: true do
      watch('config/application.rb')
      watch('config/environment.rb')
      watch('config/environments/test.rb')
      watch(%r{^config/initializers/.+\.rb$})
      watch('Gemfile')
      watch('Gemfile.lock')
      watch('test/minitest_helper.rb') { 'test' }
    end

    guard 'minitest', drb: true do
      # (省略)
    end

実行。

    $ guard

これだけで spork が自動的に起動する。
先と同じようにファイルを編集したときに動けばOK

## その他

* factory_firl_rails を入れると mintiest 用のgeneratorがちゃんと動かない (追記：githubのmasterは修正されてた)
* まだプロジェクトの規模が小さいのでsporkの恩恵を感じられない…
* spork-rails が必要かと思ってrails4ブランチも含めて試してみたが要らなかった（入れると逆にダメになる）
