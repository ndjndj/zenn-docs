---
title: "ActiveRecord で指定したカラムの配列を取得する"
emoji: "🐰"
type: "tech"
topics:
  - "rails"
  - "ruby"
  - "activerecord"
published: true
published_at: "2024-03-05 18:51"
---

Rails の ActiveRecord で、特定のモデルから指定したカラムの値の配列を取得したかったので、調べたことを備忘録としてまとめておきます。

# 結論
- ただ単純に配列を得たいだけなら、.pluck メソッドが効率的
- モデルのインスタンスメソッドなどをあとで使いたいなら、.select メソッドが効率的
- 指定するカラムが複数になるなら .select の方が取り回しはしやすいかも

# 背景
下記のような User モデルから、name カラムのみの配列を取得したかった。

```ruby
create_table "users", force: :cascade do |t|
    t.string "uuid", null: false
    t.string "user_account_id", null: false
    t.string "name", null: false
    t.intger "age"
    t.datetime "created_at", null: false
    t.datetime "updated_at", null: false
end

# name カラムの配列のみを取得したい
# -> ["taro", "ndj", "john", "jiro", "hanako", "michel"]
```

# 配列をただ取得したいだけなら、pluck メソッドが効率的
今回の場合だと、name の配列のみをただ取得したいだけなら、pluck メソッドを使えば効率的に配列を取得することができます。

```ruby
# 年齢が30歳のユーザーの名前を取得
names = User.where(age: 30).pluck(:name)
# -> ["taro", "ndj", "john", "jiro", "hanako", "michel"]
```

指定したカラムの値を直接取得することができるため、メモリを節約して効率的に配列を取得することができます。

# モデルをあとで活用したいなら、select メソッドが効率的
今回の場合だと、得られた値に加えて User モデルのインスタンスメソッドを使いたい場合は select メソッドを使用すれば、ActiveRecord::Relation モデルを返り値として得られるため、取り回しがよいです。

```ruby
# 年齢が30歳のユーザーにクーポンを配布する
users = User.where(age: 30).select(:id)
users.map do |user|
    # User モデルのインスタンスメソッド
    user.deliver_coupon
end 
```

# どちらのメソッドもカラムは複数指定できる
ちなみにどちらのメソッドも複数カラム指定することができます。

```ruby
names = User.where(age: 30).pluck(:name, :age)
# -> [["john", 30], ["taro", 30], ["jiro", 30]]
users = User.where(age: 30).select(:id, :name)
```
ただし、pluck メソッドの場合は、多次元配列になってしまい、少し扱いづらい形になってしまうので、ハッシュ化などの処理を挟む必要があります。

https://osu-log.com/archives/463

また、パフォーマンス次第ですが、.select でとってきてモデルとして扱った方がいい場合もあるかもしれないですね。

# さいごに
処理全体を見通して、それぞれ適切な方を選択して使い分けできるようにしたいです。

# 参考
https://railsdoc.com/page/model_pluck
https://railsdoc.com/page/model_select
https://zenn.dev/linkedge/articles/d17ddda0c9476b
https://osu-log.com/archives/463