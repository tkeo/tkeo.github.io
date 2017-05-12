---
layout: post
title: "rubyのrefinementsを使ってみたかった"
date: 2017-03-20 22:48:28 +0900
comments: true
categories: ruby
---
今やってるプロジェクトでrefinementsを使うのが適していそうな場面が出てきたので初めて使ってみることにしたがなんだか微妙な感じになったという日記。

試したバージョンは2.3.3と2.4.0で、このあたりを参考にした。

* https://docs.ruby-lang.org/en/trunk/syntax/refinements_rdoc.html
* http://qiita.com/joker1007/items/68d066a12bc763bd2cb4

同じmoduleをincludeしているクラスに対して共通のrefinementsを適用したかったが、2.3ではmoduleに対してrefineは使えないのでrefineブロック内でincludeしてみたが…

```
module C
end
class A
  include C
end
class B
  include C
end

module Foo
  def foo
    "foo"
  end

  def foobar
    foo + "bar"
  end
end

module R
  refine A do
    include Foo
  end

  refine B do
    include Foo
  end
end

using R
puts A.new.foo    # => foo
puts B.new.foo    # => foo
puts A.new.foobar # => undefined local variable or method `foo' for #<A:0x007fc71782dcb0> (NameError)
```

どうもそう単純にはいかないらしい。ぐぬぬ。

いろいろこねくり回した結果、思った通りの動きをするコードは一応何通りか書けた。

```
module C
end
class A
  include C
end
class B
  include C
end

# module_evalで無理やり突っ込むパターン
FooMethods = -> {
  def foo
    "foo"
  end

  def foobar
    foo + "bar"
  end
}

module R
  refine A do
    module_eval do
      FooMethods.call
    end
  end

  refine B do
    module_eval do
      FooMethods.call
    end
  end
end

# module_eval内でrefineしちゃうパターン
module R2
  %w(A B).each do |klass|
    module_eval <<-CODE
      refine #{klass} do
        def foo
          "foo"
        end

        def foobar
          foo + "bar"
        end
      end
    CODE
  end
end

# あきらめたパターン
module R3
  refine Object do
    def foo
      if kind_of?(C)
        "foo"
      else
        raise NoMethodError
      end
    end

    def foobar
      if kind_of?(C)
        foo + "bar"
      else
        raise NoMethodError
      end
    end
  end
end

using R
puts A.new.foo    # => foo
puts B.new.foo    # => foo
puts A.new.foobar # => foobar
```

うーん…どれもなんかやだな。
もうちょっとうまいやり方はないものか。
