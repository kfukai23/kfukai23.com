---
title: present? と any? の使い分けを読み手の気持ちになって考える
date: "2020-07-24T11:33:03.284Z"
description: "Qiita に載せた記事の再掲"
---

今後まとまった（記事っぽくできそうな）メモはここにまとめていくことにしたいので、一年くらい前に書いた唯一の Qiita 記事を投下しておく。

https://qiita.com/kfukai23/items/dd88aaebbeae1dfcd755

当時はまだ研修期間中だったこともあって、記事を仕上げるために同僚にずいぶんレビューしてもらったことを思い出した。本当にありがたかったし、「この会社に入ってよかった」と心底思った。

ソフトウェアエンジニアとして働き始めたころは割と頻繁に見てた Qiita だけど、ある時を境に全く見なくなった。view 数稼ぎとしか思えない釣りタイトル記事で溢れかえってるし、情報の質にばらつきがありすぎる。
この記事の原文を取るために10か月ぶりくらいにログインしたら、知らない人から LGTM がいくつかついていたり、view 数が 1,000 近くに伸びていて驚いた。プラットフォームとしてはまだまだ賑わっていることを実感。
それにしてもどこからこの記事に辿りついたのか...

以下本文

---

## present? と any? の使い分けを読み手の気持ちになって考える

### TL;DR
一見似た動作をするメソッド同士でも、レシーバにどんなオブジェクトを取れるかが異なる場合には、期待する範囲に近いものを選ぶことで、意図のわかりやすいコードにしよう

### intro
先日こういうコードを書いていたら、レビューでアドバイスをもらいました。

```erb
<% if @user.errors.present? %>
  <ul class="alert alert-warning">
    <% @user.errors.full_messages.each do |msg| %>
      <li><%= msg %></li>
    <% end %>
  </ul>
<% end %>

<%= form_with model: @user, url: admin_users_path, local: true do |f| %>
# （略）
```

-> レビュー：「`present?` は `any?` に置き換えても良いかも。**メソッドが実装されているオブジェクトの範囲は、より狭いものを使う癖をつけておくのがおすすめ**」

???となったので、ちょっと調べてみました。

### 検証環境
- Ruby 2.6.3

- Rails 5.2.3

### そもそも `present?` とは
#### 仕様
https://api.rubyonrails.org/v5.2.3/classes/Object.html#method-i-present-3F

#### 定義箇所
console で調べてみた：

```ruby
> @user.errors.method(:present?)
=> #<Method: ActiveModel::Errors(Object)#present?>

> @user.errors.method(:present?).owner
=> Object

> @user.errors.method(:present?).source_location
=> ["/Users/(ユーザー名)/.rbenv/versions/2.6.3/lib/ruby/gems/2.6.0/gems/activesupport-5.2.3/lib/active_support/core_ext/object/blank.rb", 26]
```
ソースコード：
https://github.com/rails/rails/blob/v5.2.3/activesupport/lib/active_support/core_ext/object/blank.rb#L26-L28

```ruby
# （略）
class Object
  # An object is blank if it's false, empty, or a whitespace string.
  # For example, +nil+, '', '   ', [], {}, and +false+ are all blank.

  # （略）

  def present?
    !blank?
  end

  # （略）
```

上記の通り、 `Object` クラスを再オープンして定義されている。
つまり、どんなクラスのインスタンスに対しても呼び出す事ができる。

#### `blank?` については以下の通り

ソースコード：
https://github.com/rails/rails/blob/v5.2.3/activesupport/lib/active_support/core_ext/object/blank.rb#L19-L21

```ruby
  def blank?
    respond_to?(:empty?) ? !!empty? : !self
  end
```
読み解いてみると：

- `respond_to?(:メソッド名)`
    - 引数に取ったメソッドをレシーバに対して呼べるかどうかチェックして真偽値を返すメソッド（リファレンス）

- `!!`
    - `nil` でなく真偽値が確実に返るようにする

- `!self`
    - `nil`, `false` は`偽`と評価される -> `!` で反転して `true` が返る
    - 上記以外の値（文字列、数値その他）は`真`と評価される -> `!` で反転して `false` が返る


### そもそも any? とは
#### 定義箇所

```ruby
> @user.errors.method(:any?)
=> #<Method: ActiveModel::Errors(Enumerable)#any?>

> @user.errors.method(:any?).source_location
=> nil # C言語で定義されているため（※注）

> @user.errors.method(:any?).owner
=> Enumerable
```
**`Enumerable` モジュール** に定義されている。

つまり、`Enumerable` モジュールをインクルードしているクラス（`Array`, `Hash`, `Enumerator`, `Range` など）のインスタンスに対して呼び出すことができる。

このケースでは、`ActiveModel::Errors` クラスが `Enumerable` モジュールをインクルードしているので、`any?` が使える。（該当箇所：https://github.com/rails/rails/blob/v5.2.3/activemodel/lib/active_model/errors.rb#L60)

※注 Rails ではなく Ruby に定義されている。

https://docs.ruby-lang.org/ja/2.6.0/method/Method/i/source_location.html
> その手続オブジェクトが ruby で定義されていない(つまりネイティブ である)場合は nil を返します。

#### 仕様
リファレンス
https://docs.ruby-lang.org/ja/2.6.0/method/Enumerable/i/any=3f.html
以下 [Ruby 2.6.0 リファレンスマニュアル instance method Enumerable#any?](https://docs.ruby-lang.org/ja/2.6.0/method/Enumerable/i/any=3f.html) から引用
> すべての要素が偽である場合に false を返します。 真である要素があれば、ただちに true を返します。

```ruby
p [1, 2, 3].any? {|v| v > 3 }   # => false
p [1, 2, 3].any? {|v| v > 1 }   # => true
p [].any? {|v| v > 0 }          # => false
p %w[ant bear cat].any?(/d/)    # => false
p [nil, true, 99].any?(Integer) # => true
p [nil, true, 99].any?          # => true
p [].any?                       # => false
```

### レビューコメントの趣旨は...
- `present?` は `Object` クラスに定義されているので、**どんなクラスのインスタンスに対しても呼び出せる**
- `any?` は `Enumerable` モジュールに定義されているので、**`Enumerable` モジュールをインクルードしているクラスのインスタンスに対してしか呼び出せない**
- つまり、メソッドが定義されているオブジェクトの影響範囲は `present?` > `any?` であるといえる

見方を変えると、

- コードの読み手は、この箇所を見たとき **`present?` が定義されていれば「レシーバがどんな型かは特に考慮していないのかな」 と推測する**し、 **`any?` なら暗に「レシーバが `Enumerable` モジュールをインクルードしたクラスのインスタンスであることを期待しているのだな」と推測する**
- 冒頭のケースでは、 `errors` が必ず `ActiveModel::Errors` クラスのオブジェクトであることがわかっている。このような場合は、どんなオブジェクトに対しても呼べる `present?`よりも `any?` を使うことで、**「errors には `Enumerable` なものが渡ってきますよ」という意図を表現したほうがわかりやすい**
