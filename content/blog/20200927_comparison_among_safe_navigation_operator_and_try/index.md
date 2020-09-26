---
title: try や try! と比べて &. をなぜ使うか
date: "2020-09-27T19:19:03.284Z"
description: ""
---

`&.` を使うのがベターな場面が多いのは知っていたけど、理由をちゃんと知っておきたかったのでまとめてみる。

### 三者の比較
- try
  - レシーバが nil でないときにメソッドを呼び出し、応答しない場合は nil を返す
- try!
  - レシーバが nil でないときにメソッドを呼び出し、応答しない場合は NoMethodError を返す
- &.
  - レシーバが nil でないときにメソッドを呼び出し、応答しない場合は NoMethodError を返す
  - 実質 `try!` と同じように使えるけど、こちらはレシーバが nil の場合はメソッドの引数を評価しないところが異なっている
    - 参考: [Safe Navigation Operator で呼ばれるメソッドの引数はレシーバが nil なら評価されない](https://qiita.com/yuya_takeyama/items/6126064f3e90eef24511)

表にしてみるとこう。

| method  | レシーバが nil の時 | レシーバが nil でないがメソッド呼び出しに応答しない時 |
| --- | --- | --- |
| try  | nil | nil |
| try! | nil | NoMethodError |
| &. | nil | NoMethodError |

一応試してみる。
ちなみに、Ruby 2.5.3, Rails 5.1.6.2 で検証した。
環境を作るのが面倒で、仕事で関わってる案件の環境を使ったので微妙に古いけど、まあ最新のと変わらないだろう

```ruby
$ bin/rails c

# レシーバが nil の場合
> hoge = nil
=> nil
> hoge.respond_to?(:hoge)
=> false
> hoge.hoge
NoMethodError: undefined method 'hoge' for nil:NilClass
from (pry):3:in '<main>'

# いずれの方式でも nil が返る
> hoge.try(:hoge)
=> nil
> hoge.try!(:hoge)
=> nil
> hoge&.(:hoge)
=> nil

# レシーバが nil でない場合
> foo = "foo"
=> "foo"
> foo.respond_to?(:foo)
=> false
> foo.foo
NoMethodError: undefined method 'foo' for "foo":String
from (pry):10:in '<main>'

# 存在しないメソッドを呼び出すと nil を返す
> foo.try(:foo)
=> nil

# こちらは NoMethodError が返る
> foo.try!(:foo)
NoMethodError: undefined method 'foo' for "foo":String
from /app/smalldesk/vendor/bundle/gems/activesupport-5.1.6.2/lib/active_support/core_ext/object/try.rb:17:in 'public_send'
> foo&.(:foo)
NoMethodError: undefined method 'call' for "foo":String
from (pry):13:in '<main>'
> foo&.foo
NoMethodError: undefined method 'foo' for "foo":String
from (pry):14:in '<main>'
```

### なぜ `&.` を使うといいのか
- まず、`try` は `NoMethodError`を握りつぶしてしまうので基本的には除外
  - レシーバが nil ではないがそのメソッドを持っていない = レシーバにくることを想定していないオブジェクトが渡ってきたということなので、例外を上げた方が気付きやすい（というか `try` だと握り潰されてしまって気づけない）
- で、`try!`と`&.`のうちなぜ後者かというと
  - 処理が高速
  - 記述量が少なく読みやすい
  - 純粋な Ruby の機能であるので Rails の実装に依存しないですむ
