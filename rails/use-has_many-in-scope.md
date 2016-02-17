# Use has_many in scope.

scope内でhas_manyを使おうとしたら"Undefined method"として怒られたので、
色々とまとめてみた。

## 失敗パターン

まず、ダメだったパターンから。

```user_controller.rb
# ユーザーが作成したコンテンツの購入数を取得しようとしてる
user.bought_contents(user.id).length
```

```user.rb
# providerはコンテンツの作成者
# Userのidをcontentsテーブルのprovider_idと紐付けている
has_many: sell_contents, class_name: :Content, foreign_key: :provider_id

# 上記のsell_contentsを更に派生させて、
# 購入したコンテンツをまとめているpurchasesテーブルとinner joinし、
# provider_idがUserのidとして合致するものを探そうとしている
scope :bought_contents, ->(provider_id) { sell_contents.joins(:purchases).where(provider_id: provider_id) }
```

## 成功パターン1

で、色々と試行錯誤して正答を３パターン発見しました。
まずは一つ目。

これは単純にscopeでやっていることをそのままcontrollerに書きました。

```user_controller.rb
user.sell_contents.joins(:purchases).where(provider_id: user.id).length
```

これだと動くには動きましたが、正直なんのこっちゃ分かりません。
最悪の可読性です。

という訳で新たなパターンを模索しました。

## 成功パターン２

次はhas_manyのsell_contentsを派生させました。
sell_contents内にクラスメソッドを作成するというやり方です。

```user_controller.rb
user.sell_contents.bought(user.id).length
```

```user.rb
has_many: sell_contents, class_name: :Content, foreign_key: :provider_id do
  def bought(provider_id)
    joins(:purchases).where(provider_id: provider_id)
  end
end
```

成功パターン１よりかは可読性が大幅アップしましたが、これでもまだ冗長的な書き方のような気がします。
もっと改良しましょう。

参考：[Scope with join on :has_many :through association](http://stackoverflow.com/questions/5856838/scope-with-join-on-has-many-through-association)

## 成功パターン３

新しくhas_manyとしてbought_contentsを作成させる形で１行に収まる書き方を発見しました。

```user_controller.rb
user.bought_contents(user.id).length
```

```user.rb
has_many: bought_contents, ->(provider_id) { joins(:purchases).where(provider_id: provider_id) }, class_name: :Content, foreign_key: :provider_id
```

sell_contentsの書き方に、scopeをくっつけたような書き方ですね。
これが可読性や冗長性を考えた時に一番ベターな書き方のような気がします。

## 何故失敗したのか？

では何故scopeでhas_manyが呼べなかったのか？

それは __has_many__ がインスタンスメソッドだからです。

scopeはクラスメソッドであるため、以下のように書き直せます。

```user.rb
def self.bought_contents(provider_id) do
  sell_contents.joins(:purchases).where(provider_id: provider_id)
end
```

上述のようにhas_manyはインスタンスメソッドであるので、
この場合クラスメソッド内からインスタンスメソッドを呼ぶため怒られます。


以上、考えてみればhas_manyにscopeオプションもある時点で気付けよというお話なのですが、
何故失敗したのか？も含めて良い勉強になりました。
