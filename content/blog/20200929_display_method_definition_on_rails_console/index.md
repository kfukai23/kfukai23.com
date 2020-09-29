---
title: コンソールから Rails のメソッド定義を確認する
date: "2020-09-29T19:19:03.284Z"
description: ""
---

rails console でメソッドの中身を見られることを知った。

```ruby
> Comment.method(:present?)
=> #<Method: Comment(id: integer, episode_id: integer, body: string, commented_at: string, created_at: datetime, updated_at: datetime).present?>
> Comment.method(:present?).source
=> "  def present?\n    !blank?\n  end\n"
> Comment.method(:present?).source.display
  def present?
    !blank?
  end
=> nil
```

へー Ruby の Method クラスに source なんてメソッドがあるとは、さすが気が効いてるなぁと一瞬思ったけど、こういう形で Ruby の内部実装を見る機会ってあんまり提供しているイメージもなかったので、Rails が Method クラスを拡張してるのかな？と思った。
結果、そういう訳でもなく、 `method_source` gem が提供しているメソッドだった。

```ruby
> Comment.method(:present?).method(:source)
=> #<Method: Method(MethodSource::MethodExtensions)#source>
> Comment.method(:present?).method(:source).source_location
=> ["/Users/kaitofukai/.rbenv/versions/2.6.6/lib/ruby/gems/2.6.0/gems/method_source-1.0.0/lib/method_source.rb", 109]
```

railtie が依存してる、つまり最初からインストールされていて使える。

Gemfile.lock

```
GEM
  remote: https://rubygems.org/
    ...
    railties (6.0.3.3)
      actionpack (= 6.0.3.3)
      activesupport (= 6.0.3.3)
      method_source
      rake (>= 0.8.7)
      thor (>= 0.20.3, < 2.0)
    rake (13.0.1)
    ...
```

